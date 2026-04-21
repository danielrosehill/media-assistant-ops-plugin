# Media Assistant Ops

A Claude Code plugin for interfacing with a [Music Assistant](https://www.music-assistant.io/) server via its local API. Control players, inspect the library, and save or update per-player DSP settings from inside Claude Code.

## Skills

- `onboard` — capture the server's LAN IP, optional WAN IP, and API key; write them to the plugin's config file.
- `status` — verify connectivity to the configured Music Assistant server and report version + player count.
- `list-players` — enumerate players known to the server with current state and volume.
- `control-player` — play / pause / stop / skip / set volume on a named player.
- `save-dsp-settings` — snapshot a player's current DSP (EQ + filters) configuration to disk as a named preset.
- `update-dsp-settings` — apply a saved DSP preset (or ad-hoc values) to a player.
- `reference` — shared reference on the Music Assistant API shape, auth, and endpoint patterns.

## Installation

```bash
claude plugins install media-assistant-ops@danielrosehill
```

After installing, run the `onboard` skill once to capture the server address and API key.

## Data storage

The plugin stores its config and saved DSP presets under:

```
${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/media-assistant-ops/
├── config.json        # server IPs + API key
└── data/dsp-presets/  # saved DSP presets per player
```

Nothing is written to `~/.claude/` or the plugin install directory.

## License

MIT
