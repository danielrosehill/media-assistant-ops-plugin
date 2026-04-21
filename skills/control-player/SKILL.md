---
name: control-player
description: Send a playback command to a Music Assistant player — play, pause, stop, next, previous, or set volume. Use when the user asks to control a specific player by name (e.g. "pause the living room", "skip track on kitchen", "set bedroom to 40%").
---

# Control Player

## Supported commands

| Command | Endpoint | Args |
|---|---|---|
| play | POST `/api/players/<id>/command/play` | — |
| pause | POST `/api/players/<id>/command/pause` | — |
| stop | POST `/api/players/<id>/command/stop` | — |
| next | POST `/api/players/<id>/command/next` | — |
| previous | POST `/api/players/<id>/command/previous` | — |
| volume | POST `/api/players/<id>/command/volume_set` | `{"volume": 0-100}` |
| power | POST `/api/players/<id>/command/power_toggle` | — |

## Steps

1. Load config per the reference skill.

2. Resolve the player. If the user gave a name (not an ID), GET `/api/players`, match case-insensitively on `name`, and use the `player_id`. If multiple match, ask which one.

3. POST the command with the Bearer token. For `volume`, include the JSON body `{"volume": <int>}`.

4. Report the new state (re-fetch the single player and print name + state + volume).

5. If the server returns 404 on the endpoint, fall back to the WebSocket command bus pattern described in the reference skill.
