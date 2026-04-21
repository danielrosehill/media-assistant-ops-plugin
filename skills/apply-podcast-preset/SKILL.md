---
name: apply-podcast-preset
description: Apply a recommended parametric-EQ DSP preset tuned for spoken-word / podcast listening to a Music Assistant player. Tames sub-bass rumble, lifts vocal presence (1–4 kHz), and softens sibilance (~6–8 kHz). Use when the user wants a one-shot "make this speaker good for podcasts" action — typically a kitchen, bathroom, or office speaker — without hand-rolling EQ bands.
---

# Apply Podcast DSP Preset

Push a curated parametric-EQ preset to a Music Assistant player, optimised for podcasts and audiobook content.

## The preset

Six peaking/shelf bands plus a small preamp trim. Tuned for full-range speakers; a separate variant is provided for small/desktop speakers.

### Variant: `podcast-default`

| # | Type | Freq (Hz) | Gain (dB) | Q | Purpose |
|---|---|---|---|---|---|
| 1 | high_pass | 70 | — | 0.7 | Roll off sub-bass rumble / room noise |
| 2 | peaking | 200 | -3.0 | 1.0 | Cut "muddy" lower-mids |
| 3 | peaking | 500 | -1.5 | 1.2 | Reduce boxy midrange |
| 4 | peaking | 2500 | +3.0 | 1.0 | Lift vocal intelligibility |
| 5 | peaking | 4000 | +2.0 | 1.4 | Add presence / clarity |
| 6 | peaking | 7000 | -2.5 | 3.0 | Tame sibilance / harshness |

Preamp: `-2.0 dB` (compensate for the +3 dB peaks so output stays clean).

### Variant: `podcast-small-speaker`

For desktop / smart-speaker grade hardware that lacks low-end. Drops the high-pass and the 200 Hz cut (already missing on the speaker).

| # | Type | Freq (Hz) | Gain (dB) | Q |
|---|---|---|---|---|
| 1 | peaking | 2500 | +3.5 | 1.0 |
| 2 | peaking | 4000 | +2.5 | 1.4 |
| 3 | peaking | 7000 | -2.0 | 3.0 |

Preamp: `-2.0 dB`.

## Steps

1. Load config per the `reference` skill. Resolve the target player (by name or ID).

2. Ask which variant to apply if the user didn't specify. Default to `podcast-default`.

3. Stash the player's current DSP first (so the user can revert):

   ```bash
   $DATA_ROOT/state/last-dsp-<player-slug>.json
   ```

   Use the `config/players/get` command (or the matching REST endpoint) to fetch the current `player_config`, extract its `dsp` block, and save the whole player_config payload alongside.

4. Build the DSP payload. Authoritative shape (per Music Assistant docs — see `reference` skill):

   ```json
   {
     "enabled": true,
     "preamp_gain_db": -2.0,
     "per_channel": false,
     "bands": [
       {"type": "high_pass", "frequency": 70, "q": 0.7, "enabled": true},
       {"type": "peaking",   "frequency": 200,  "gain_db": -3.0, "q": 1.0, "enabled": true},
       {"type": "peaking",   "frequency": 500,  "gain_db": -1.5, "q": 1.2, "enabled": true},
       {"type": "peaking",   "frequency": 2500, "gain_db":  3.0, "q": 1.0, "enabled": true},
       {"type": "peaking",   "frequency": 4000, "gain_db":  2.0, "q": 1.4, "enabled": true},
       {"type": "peaking",   "frequency": 7000, "gain_db": -2.5, "q": 3.0, "enabled": true}
     ]
   }
   ```

   Supported filter `type` values (verbatim from MA docs): `peaking` (a.k.a. Peak/Bell), `high_shelf`, `low_shelf`, `high_pass`, `low_pass`, `notch`, `all_pass`. **Not** supported: Modal, BP, LSC x dB, HS x dB, AP, LS with dB slope, HS with dB slope.

5. Apply via the `config/players/save` command (WebSocket) or the equivalent `PUT /api/config/players/<player_id>` endpoint, merging the new `dsp` block into the existing `player_config`.

6. Verify by re-fetching the player_config and confirming the DSP block matches what was sent. Print a one-line confirmation: `living-room: applied podcast-default (6 bands, preamp -2 dB)`.

7. Tell the user how to revert: `Run /media-assistant-ops:update-dsp-settings and choose "revert last change" — or restore from $DATA_ROOT/state/last-dsp-<player-slug>.json.`

8. Offer to also save this as a named DSP preset for the player (delegate to `save-dsp-settings`) so it shows up in their preset library.

## Adapting for other content

If the user asks for variations:

- **More bass for a TV speaker doing podcasts**: drop the high_pass to 50 Hz, add a `low_shelf` at 120 Hz +2 dB.
- **Harsh-sounding speaker (cheap tweeter)**: deepen band 6 to -4 dB and add a `peaking` at 9 kHz, -2 dB, Q 2.0.
- **Mono in-ceiling speaker**: keep `per_channel: false`. For stereo desk monitors, set `per_channel: true` and apply identical bands to both channels by default.

## Do not

- Do not push gains above +6 dB on any band — combined with program material it will clip.
- Do not assume the player supports DSP. Music Assistant DSP only applies to *Music Assistant–native* player flows (snapcast / built-in player). Hardware players that handle their own decoding (e.g. some Sonos modes) bypass MA's DSP. If the player_config lacks a `dsp` key, tell the user this player doesn't support MA-side DSP and suggest applying EQ at the device.
