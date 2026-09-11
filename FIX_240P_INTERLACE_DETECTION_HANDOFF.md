# 240p / Interlace Detection and Scanline Stability Handoff

## Repository state

- Repository: `niklasva/gbsc-pro`
- Branch: `fix-240p-interlace-detection`
- Branch point: `origin/main` at `c6a2fb0`
- The branch was clean when this document was created.
- No firmware fix has been implemented yet.

## User-reported behavior

Hardware and input details:

- GBSC-Pro receiving RGBS from a 1CHIP Super Famicom.
- The console uses an RGC 75-ohm CSYNC cable.
- Motion Adaptive deinterlacing is preferred because other connected sources genuinely use 480i.

Symptoms:

1. With scanlines enabled on the 240p SFC signal, they occasionally disappear for a fraction of a second.
2. The picture sometimes freezes for approximately the same short interval.
3. Scanlines sometimes cannot be enabled at all, including with a Mega Drive.
4. Selecting Bob deinterlacing makes the scanlines remain enabled.

Bob is useful as a diagnostic result, but it is not an acceptable permanent workaround because Motion Adaptive must continue to work for real 480i inputs.

## Current diagnosis

The strongest explanation is that the firmware is reacting to transient `VPERIOD_IF` measurements as though the source changed between progressive 240p/288p and interlaced 480i/576i.

The Bob result confirms that scanline availability is coupled to the Adaptive/interlace classification path. It does not by itself prove that every brief freeze has the same cause, but if the freezes also disappear under Bob, that is strong evidence that the deinterlacer transition is responsible.

The 75-ohm CSYNC cable addresses sync amplitude/termination compatibility. It does not eliminate the known NTSC SNES short-line timing irregularity (often called SNES sync jitter), so a correct cable can still expose overly sensitive firmware classification.

Relevant background:

- SNES timing, including the short NTSC non-interlaced scanline: <https://snes.nesdev.org/wiki/Timing>
- NES/SNES de-jitter project: <https://github.com/marqs85/snes_dejitter>
- Existing 1CHIP SNES/GBS-Control jumpiness report: <https://github.com/ramapcsx2/gbs-control/issues/543>

## Implicated code

### 1. Raw scanline availability check

`areScanLinesAllowed()` is in `gbs-control/gbs-control.ino` around line 6051.

It reads `GBS::VPERIOD_IF` directly and considers only these exact values progressive:

- NTSC-like: `521`, `523`, `525`
- PAL-like: `623`, `625`, `627`

When Adaptive is selected (`deintMode == 0`), every other value is treated as scanline-incompatible. This includes recognized even/interlaced values, genuinely unknown values, and transient/out-of-range measurements. There is no hysteresis in this helper.

### 2. Automatic deinterlacer switching

The automatic detection block is in `gbs-control/gbs-control.ino` around lines 6890-6960.

It recognizes:

- Interlaced: `522`, `524`, `526`, `622`, `624`, `626`
- Progressive: `521`, `523`, `525`, `623`, `625`, `627`

It switches after only two qualifying polls. The sync watcher normally runs about every 20 ms, so two readings provide very little noise immunity.

It also resets both classification counters whenever the exact raw `VPERIOD_IF` value changes. Classification should instead be based on whether the reading is progressive, interlaced, or unknown. A change between two values in the same class should not necessarily destroy accumulated confidence.

When the detector decides the signal is interlaced while Adaptive is selected, it explicitly:

1. Disables scanlines.
2. Calls `enableMotionAdaptDeinterlace()`.

### 3. The visible freeze

`enableMotionAdaptDeinterlace()` is in `gbs-control/gbs-control.ino` around line 6149.

It calls `freezeVideo()`, reconfigures the deinterlacer, calls `unfreezeVideo()`, and then delays for 60 ms. A false progressive-to-interlaced classification therefore naturally produces a short visible freeze.

The general sync-loss path also calls `freezeVideo()`. Hardware testing or temporary diagnostics should distinguish deinterlacer transitions from genuine `noSyncCounter` events before claiming that every freeze is fixed.

### 4. Immediate scanline removal bypasses the detector filter

The SD scanline block is around lines 6997-7007 of `gbs-control/gbs-control.ino`.

This statement disables active scanlines from a single raw availability result:

```cpp
if (rto->scanlinesEnabled && !areScanLinesAllowed()) {
    disableScanlines();
}
```

Consequently, even improving the two-reading Motion Adaptive detector is insufficient by itself. This direct raw check must be removed or changed to use a stable, debounced classification.

### 5. Menu input can be silently ignored

`gbs-control/pro/menu/handlers/menu-color.cpp`, around lines 238-251, sends scanline commands only when `areScanLinesAllowed()` returns true at the instant the remote button is handled.

A transient bad measurement can therefore make the button appear unresponsive. The desired scanline preference should be recordable independently of whether scanlines can be applied at that instant.

### 6. Related RGBHV regression to verify

The scaled RGBHV scanline block around line 7485 also calls `areScanLinesAllowed()`. However, the helper only accepts `videoStandardInput` values `1` and `2`; RGBHV mode `14` therefore always returns false.

This appears to make that scanline path unreachable or immediately self-disabling. It was not the primary reported problem, but the fix should avoid retaining this inconsistency.

### 7. Difference from upstream GBS-Control

The `areScanLinesAllowed()` helper was introduced in this codebase by commit `3654f05` (`feat: improve OSD menu handling and scanlines availability logic`). Current upstream GBS-Control does not have this helper or the same immediate raw scanline shutdown:

<https://raw.githubusercontent.com/ramapcsx2/gbs-control/master/gbs-control.ino>

Upstream still contains the basic automatic progressive/interlaced detector and disables scanlines when Motion Adaptive actually activates. Restoring upstream behavior is useful as a reference, though the two-poll detector should still be made more robust for the reported SFC behavior.

## Required behavior

The completed fix should satisfy all of the following:

1. A stable 240p/288p source may use scanlines with Adaptive selected.
2. Isolated or unknown `VPERIOD_IF` readings do not remove scanlines.
3. A genuine 480i/576i source still activates Motion Adaptive automatically.
4. A genuine transition between progressive and interlaced video still works.
5. Scanlines are disabled when stable interlaced video is processed with Motion Adaptive.
6. Bob behavior remains unchanged, including scanline support where currently intended.
7. The scanline preference can be changed while scanlines are temporarily unavailable; it should take effect when the source becomes compatible.
8. Bypass and non-SD modes retain their intended restrictions.
9. RGBHV scanline behavior is explicitly checked and is not accidentally made worse.

## Recommended implementation direction

Use one shared, debounced SD scan classification instead of having UI and runtime code repeatedly interpret a raw register value.

A small three-state classification is sufficient:

```text
Unknown
Progressive
Interlaced
```

Suggested approach:

1. Add a helper that maps one `VPERIOD_IF` reading to one of the three classes.
2. Track consecutive classifications, not consecutive identical raw values.
3. Unknown readings should normally preserve the last stable classification rather than immediately changing behavior.
4. Require several consistent readings before changing the stable classification. Six polls is a reasonable initial value: approximately 120 ms at the current watcher interval. Tune from hardware observations if necessary.
5. Switch Motion Adaptive only when the debounced state becomes Interlaced.
6. Switch it off only when the debounced state becomes Progressive.
7. Enable/disable scanlines from that same stable state and the selected deinterlacer mode.
8. Do not use a fresh raw `VPERIOD_IF` read to remove active scanlines.
9. Let the menu toggle `uopt->wantScanlines` even when scanlines cannot currently be rendered. The OSD may still display an unavailable state, but it should not silently discard the user's preference change.

Keep the implementation localized. Avoid duplicating classification state between `areScanLinesAllowed()`, the sync watcher, and the menu.

## Instrumentation recommended before finalizing thresholds

If hardware access is available, temporarily log only state changes or suspicious readings to avoid overwhelming the ESP8266:

- Raw `VPERIOD_IF`
- Classified sample: progressive/interlaced/unknown
- Debounce counters and stable classification
- `motionAdaptiveDeinterlaceActive`
- `scanlinesEnabled` and `wantScanlines`
- `noSyncCounter`
- Calls to `enableMotionAdaptDeinterlace()` and `disableMotionAdaptDeinterlace()`

This will show whether the SFC produces isolated even readings, unknown counts, actual sync-loss events, or some combination.

## Validation matrix

At minimum, manually check:

| Source | Input mode | Deinterlacer | Expected result |
| --- | --- | --- | --- |
| 1CHIP SFC | 240p | Adaptive | Scanlines stay enabled; no false Adaptive transition or transition freeze |
| Mega Drive | 240p | Adaptive | Scanline control responds consistently and scanlines remain stable |
| Known interlaced source | 480i | Adaptive | Motion Adaptive activates after the debounce interval; scanlines remain off |
| Known interlaced source | 480i | Bob | Bob remains active and scanline behavior matches existing intended behavior |
| Mode-switching source/game | 240p to 480i and back | Adaptive | Both transitions occur reliably with only the intended short debounce delay |
| Noisy or disconnected source | Unknown/no sync | Adaptive | No false scanline/deinterlacer state oscillation; normal no-signal recovery still works |

Also verify PAL equivalents (`288p`/`576i`) if suitable hardware is available.

## Out-of-scope unless evidence points there

- Replacing Motion Adaptive with Bob globally.
- Assuming the RGC cable is defective merely because it cannot remove SNES timing jitter.
- Large sync-watcher or frame-time-lock rewrites.
- Changing unrelated AV/S-Video processing.

## Concise task prompt for the next Codex conversation

> Read `FIX_240P_INTERLACE_DETECTION_HANDOFF.md`, inspect the referenced code, and implement the smallest robust fix on branch `fix-240p-interlace-detection`. Preserve automatic Motion Adaptive deinterlacing for genuine 480i/576i, but prevent transient or unknown `VPERIOD_IF` readings from disabling scanlines or causing false mode transitions. Keep the scanline preference independent from temporary availability, verify the related RGBHV path, and summarize the resulting behavior and any hardware validation still needed.
