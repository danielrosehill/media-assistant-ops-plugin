---
name: reference
description: Shared reference for the Music Assistant local API — base URL, auth, the WebSocket command bus, the DSP/EQ schema, and the data-storage layout this plugin uses. Load when another skill in this plugin needs to know how to talk to the server, what DSP shape to send, or where to read/write files.
---

# Music Assistant — Reference

Music Assistant (MA) runs as a Home Assistant add-on or standalone container. It exposes a local HTTP + WebSocket API on port **8095** by default.

## Authoritative docs (always re-check)

- **API overview**: https://www.music-assistant.io/api/
- **DSP Parametric EQ**: https://www.music-assistant.io/dsp/parametriceq/
- **DSP Tone Controls**: https://www.music-assistant.io/dsp/tonecontrols/
- **Live, version-specific OpenAPI on the running server**: `http://<LAN_IP>:8095/api-docs`

The MA API surface evolves between releases. When a call 4xx/404s, fetch `/api-docs` from the user's running server first — it's always the source of truth for that install.

## Server location

- **LAN base URL**: `http://<LAN_IP>:8095`
- **WebSocket**: `ws://<LAN_IP>:8095/ws`
- **WAN base URL** (optional): provided during onboarding if the server is reachable from outside the LAN.

Prefer the LAN address when both are available.

## Authentication

Long-lived **API key** as a Bearer token:

```
Authorization: Bearer <API_KEY>
```

For WebSocket, the first frame after connect is an auth message:

```json
{"type": "auth", "token": "<API_KEY>"}
```

## Command bus pattern

MA's primary API is a JSON-RPC-style command bus available over both HTTP (`POST /api`) and WebSocket. Each call looks like:

```json
{"command": "<command_name>", "args": { ... }, "message_id": "<uuid>"}
```

Verbatim commands worth knowing (subject to version):

- `config/players/get` — fetch a player_config by player_id
- `config/players/save` — write a player_config (this is how DSP is updated)
- `players/cmd/play`, `players/cmd/pause`, `players/cmd/stop`, `players/cmd/next`, `players/cmd/previous`
- `players/cmd/volume_set` (args: `{ "player_id": "...", "volume_level": 0-100 }`)
- `players/cmd/power` (args: `{ "player_id": "...", "powered": true|false }`)
- `players/all` — list players
- `music/library/...` — library queries

If you're unsure of the exact command name on the user's MA version, hit `http://<LAN_IP>:8095/api-docs` and grep for the keyword.

## DSP schema

Per the official DSP docs, the parametric EQ supports:

- **Filter types** (use these exact strings): `peaking` (Peak / Bell), `high_shelf`, `low_shelf`, `high_pass`, `low_pass`, `notch`, `all_pass`.
- **Unsupported** (don't generate): Modal, BP, LSC x dB, HS x dB, AP, LS with dB slope, HS with dB slope.
- **Bands**: unlimited.
- **Per-channel control**: optional (left/right independent or linked).
- **Preamp gain**: a global pre-EQ gain stage for compensation.

The DSP block lives **inside the player_config** under a `dsp` key. Canonical shape this plugin emits:

```json
{
  "dsp": {
    "enabled": true,
    "preamp_gain_db": -2.0,
    "per_channel": false,
    "bands": [
      {"type": "peaking",   "frequency": 2500, "gain_db": 3.0, "q": 1.0, "enabled": true},
      {"type": "high_pass", "frequency": 70,                   "q": 0.7, "enabled": true}
    ]
  }
}
```

Notes:
- `gain_db` is omitted for filters where it doesn't apply (`high_pass`, `low_pass`, `notch`, `all_pass`).
- `q` is meaningful for `peaking` and the pass/notch filters.
- `frequency` is integer Hz.
- The exact JSON keys used by the running MA may differ slightly between versions (e.g. `gain` vs `gain_db`, `freq` vs `frequency`) — when a save fails, fetch `/api-docs` and adapt.

DSP only applies to MA-native player flows (snapcast / built-in players). Hardware players that handle their own decoding may not expose a `dsp` key in their player_config.

Tone Controls (separate, simpler bass/treble) live alongside DSP per the docs at https://www.music-assistant.io/dsp/tonecontrols/.

## Data-storage layout (this plugin)

```bash
DATA_ROOT="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/media-assistant-ops"
```

```
$DATA_ROOT/
├── config.json                       # { lan_url, wan_url, api_key, port }
├── data/
│   ├── players-snapshot.json         # raw players/all response from onboarding
│   ├── players-snapshot.md           # human-readable roster
│   ├── library-snapshot.json
│   ├── library-snapshot.md
│   └── dsp-presets/
│       └── <player-slug>/
│           └── <preset-name>.json
└── state/
    └── last-dsp-<player-slug>.json   # rollback snapshot before any DSP change
```

**Never** write plugin data under `~/.claude/` or inside the plugin install directory.

## Helper: load config in a skill

```bash
DATA_ROOT="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/media-assistant-ops"
CONFIG="$DATA_ROOT/config.json"
[ -f "$CONFIG" ] || { echo "Run /media-assistant-ops:onboard first"; exit 1; }
LAN_URL=$(jq -r '.lan_url' "$CONFIG")
API_KEY=$(jq -r '.api_key' "$CONFIG")
```

## Helper: command-bus call

```bash
curl -sS -X POST "$LAN_URL/api" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"command":"config/players/get","args":{"player_id":"'"$PLAYER_ID"'"}}'
```
