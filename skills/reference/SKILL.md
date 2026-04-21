---
name: reference
description: Shared reference for the Music Assistant local API — base URL, auth, WebSocket command shape, and the data-storage layout this plugin uses. Load when another skill in this plugin needs to know how to talk to the server or where to read/write files.
---

# Music Assistant — Reference

Music Assistant (MA) runs as a Home Assistant add-on or standalone container. It exposes a local HTTP + WebSocket API on port **8095** by default.

## Server location

- **LAN base URL**: `http://<LAN_IP>:8095`
- **WebSocket**: `ws://<LAN_IP>:8095/ws`
- **WAN base URL** (optional): provided during onboarding if the server is reachable from outside the LAN.

Prefer the LAN address when both are available — lower latency and no NAT.

## Authentication

MA uses an **API key** (long-lived access token) passed as a Bearer token:

```
Authorization: Bearer <API_KEY>
```

For WebSocket, the first message after connect is an auth frame:

```json
{"type": "auth", "token": "<API_KEY>"}
```

## Useful HTTP endpoints

| Verb | Path | Purpose |
|---|---|---|
| GET | `/api/info` | Server version, uptime, provider list |
| GET | `/api/players` | List all players |
| GET | `/api/players/<player_id>` | Single player detail |
| POST | `/api/players/<player_id>/command/<cmd>` | Commands: `play`, `pause`, `stop`, `next`, `previous`, `volume_set`, `power_toggle` |
| GET | `/api/players/<player_id>/dsp` | Get DSP config (EQ bands + filters) |
| PUT | `/api/players/<player_id>/dsp` | Replace DSP config |

Note: MA's API surface evolves. If an endpoint 404s, fall back to the WebSocket command bus (`{"type": "command", "command": "...", "args": {...}}`) and consult the server's OpenAPI at `http://<LAN_IP>:8095/docs`.

## Data-storage layout

Resolve the plugin data root:

```bash
DATA_ROOT="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/media-assistant-ops"
```

Layout:

```
$DATA_ROOT/
├── config.json             # { "lan_ip", "wan_ip", "api_key", "port" }
└── data/
    └── dsp-presets/
        └── <player-slug>/
            └── <preset-name>.json
```

**Never** write plugin data under `~/.claude/` or inside the plugin install directory — those are overwritten on plugin updates.

## Config schema

```json
{
  "lan_ip": "10.0.0.x",
  "wan_ip": null,
  "port": 8095,
  "api_key": "ma_xxx..."
}
```

## DSP preset schema

```json
{
  "name": "living-room-night",
  "player_id": "player_abc",
  "saved_at": "2026-04-21T20:00:00+03:00",
  "dsp": {
    "enabled": true,
    "bands": [ {"freq": 60, "gain": -3.0, "q": 1.0}, ... ],
    "filters": [ ... ]
  }
}
```

## Helper: curl pattern

```bash
curl -sS -H "Authorization: Bearer $API_KEY" \
  "http://$LAN_IP:8095/api/players"
```

## Helper: reading config in a skill

```bash
DATA_ROOT="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/media-assistant-ops"
CONFIG="$DATA_ROOT/config.json"
[ -f "$CONFIG" ] || { echo "Run /media-assistant-ops:onboard first"; exit 1; }
LAN_IP=$(jq -r '.lan_ip' "$CONFIG")
API_KEY=$(jq -r '.api_key' "$CONFIG")
PORT=$(jq -r '.port // 8095' "$CONFIG")
BASE="http://$LAN_IP:$PORT"
```
