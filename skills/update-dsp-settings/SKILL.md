---
name: update-dsp-settings
description: Apply a DSP configuration to a Music Assistant player — either by loading a previously saved preset from disk or by adjusting specific EQ bands/filters in-place. Use when the user wants to switch a player to a saved preset ("apply night-mode to living room") or tweak bands directly ("boost 60Hz by 2dB on bedroom").
---

# Update DSP Settings

Two modes: **apply preset** or **ad-hoc edit**.

## Mode A: Apply a saved preset

1. Load config per the reference skill. Resolve the target player.

2. List presets in `$DATA_ROOT/data/dsp-presets/<player-slug>/`. If the user named one, load it. If not, show the list and ask.

3. PUT the preset's `dsp` block to `$BASE/api/players/<player_id>/dsp` with the Bearer token.

4. Verify by GETting the DSP config back and confirming it matches. Report success with the preset name applied.

## Mode B: Ad-hoc edit

1. GET the current DSP config.

2. Apply the user's requested change to the in-memory object:
   - Band adjustment: locate the band by frequency (nearest match) and set `gain` / `q`.
   - Add/remove band: mutate the `bands` array.
   - Toggle enabled: flip the `enabled` flag.
   - Filter changes: mutate the `filters` array.

3. PUT the modified config back.

4. Offer to save the result as a new preset (delegate to the `save-dsp-settings` skill).

## Safety

- Before any PUT, echo a one-line summary of what will change (e.g. `living-room: band@60Hz gain -3 -> -5, 4 other bands unchanged`) and require confirmation unless the user's prompt was unambiguous.
- Keep a one-shot rollback: stash the pre-change DSP to `$DATA_ROOT/state/last-dsp-<player-slug>.json` before the PUT so the user can say "revert" and you can re-PUT it.
