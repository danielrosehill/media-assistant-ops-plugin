---
name: list-players
description: Enumerate all players known to the Music Assistant server along with their current state (playing/paused/idle), volume, and currently-playing track if any. Use when the user asks what players are available, which are active, or needs a player_id to target with another command.
---

# List Players

## Steps

1. Load config per the reference skill.

2. GET `$BASE/api/players` with the Bearer token.

3. Render a compact table with columns:
   - Name
   - Player ID (needed for other skills)
   - State (playing / paused / idle / off)
   - Volume (%)
   - Now playing (track — artist, if state is `playing`)
   - Provider / group (if relevant)

4. If the result is empty, tell the user the server sees no players and suggest checking the Music Assistant provider configuration.
