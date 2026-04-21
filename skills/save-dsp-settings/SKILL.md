---
name: save-dsp-settings
description: Snapshot a Music Assistant player's current DSP (EQ bands + filters) configuration and persist it as a named preset under $CLAUDE_USER_DATA. Use when the user has dialed in an EQ on a player and wants to capture it so it can be restored later, shared across players, or version-controlled.
---

# Save DSP Settings

## Steps

1. Load config per the reference skill.

2. Resolve the target player (by name or ID — see the control-player skill for the resolution pattern).

3. GET `$BASE/api/players/<player_id>/dsp` to fetch the current DSP config.

4. Ask the user for a preset name (e.g. `living-room-night`, `podcast-voice`). Slugify it (lowercase, dashes).

5. Write to:

   ```
   $CLAUDE_USER_DATA_ROOT/media-assistant-ops/data/dsp-presets/<player-slug>/<preset-name>.json
   ```

   with contents matching the preset schema in the reference skill:

   ```json
   {
     "name": "<preset-name>",
     "player_id": "<player_id>",
     "player_name": "<human name>",
     "saved_at": "<ISO 8601>",
     "dsp": { ... raw DSP block from the API ... }
   }
   ```

   `mkdir -p` the parent directory first.

6. If a preset with the same name already exists for this player, show the user a diff summary (count of band changes, enabled/disabled toggles) and confirm before overwriting.

7. Report the absolute path of the saved preset so the user can locate it on disk.
