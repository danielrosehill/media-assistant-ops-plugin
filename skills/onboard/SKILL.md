---
name: onboard
description: First-run setup for the media-assistant-ops plugin. Capture the user's local Music Assistant deployment URL + API key, snapshot the current speaker/player roster for future context, vet the overall setup (providers, connectivity, auth), and summarise the library (track/album/artist counts, active music providers). Use when the plugin is freshly installed, when `config.json` is missing, or when the user wants to re-baseline against a new server.
---

# Onboard — Music Assistant

End-to-end first-run: capture credentials, verify the server responds, snapshot the environment, and produce a context file the other skills can rely on.

## Phase 1: Capture credentials

1. Resolve the data root and ensure directories exist:

   ```bash
   DATA_ROOT="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/media-assistant-ops"
   mkdir -p "$DATA_ROOT/data" "$DATA_ROOT/state"
   ```

2. If `$DATA_ROOT/config.json` exists, display the current values (redact API key to last 4 chars) and confirm the user wants to overwrite.

3. Ask the user for:
   - **Local deployment URL** — full base URL (e.g. `http://10.0.0.50:8095`). Accept an IP alone and assume `http://` + default port `8095`.
   - **WAN URL / hostname** (optional).
   - **API key** (required) — long-lived token from Music Assistant → Settings → Security.

4. Write `$DATA_ROOT/config.json` with `lan_url`, `wan_url`, `api_key`. `chmod 600` the file.

   Never echo the full API key back — only the last 4 characters for confirmation.

## Phase 2: Vet the setup

Run these checks against the LAN URL (fall back to WAN if LAN is unreachable and a WAN URL was provided). Report each as ✓ / ✗ with a short note.

1. **Reachability** — GET `/api/info` with the Bearer token. Capture server version, Python version, OS.
2. **Auth** — confirm the API key was accepted (200 vs 401/403).
3. **Provider health** — GET `/api/providers`. For each provider, report `name`, `type` (music / player / metadata / plugin), and `available` status. Flag any provider in an error state.
4. **Port / firewall** — if GET `/api/info` fails, attempt a raw TCP connect to the port and report whether the service is up vs whether it's an auth problem.

Produce a short **Setup Vetting Report** the user can read at a glance.

## Phase 3: Snapshot the speaker/player roster

1. GET `/api/players` and persist the full response to:

   ```
   $DATA_ROOT/data/players-snapshot.json
   ```

2. Write a human-readable summary to `$DATA_ROOT/data/players-snapshot.md` with one row per player:

   | Name | Player ID | Type | Provider | Default volume | Group membership |

   This file is the future-context artefact — other skills can grep it to resolve names → IDs and it gives you (Claude) a cheap way to know what hardware exists without a live call.

3. If any player is part of a sync group, note the group and which player is the leader.

## Phase 4: Library summary

1. Hit the library summary endpoints (if the running MA version exposes them) and persist raw JSON to `$DATA_ROOT/data/library-snapshot.json`:
   - GET `/api/library/tracks?limit=1` — read total count from pagination meta
   - GET `/api/library/albums?limit=1`
   - GET `/api/library/artists?limit=1`
   - GET `/api/library/playlists?limit=1`
   - GET `/api/library/radios?limit=1`

   If a given endpoint 404s on this MA version, skip it and note "not supported on this server version" in the report — don't fail the whole onboarding.

2. List active **music providers** (from Phase 2's provider fetch, filtered to `type=music`) with track count per provider if the API exposes it.

3. Write a human-readable `$DATA_ROOT/data/library-snapshot.md`:

   ```
   # Music Assistant Library — snapshot <ISO date>
   - Tracks: N
   - Albums: N
   - Artists: N
   - Playlists: N
   - Radios: N

   ## Music providers
   - spotify (connected) — 12,345 tracks
   - plex (connected) — 4,567 tracks
   - filesystem (connected) — 890 tracks
   ```

## Phase 5: Final report to the user

Print a single consolidated summary covering:

- ✓/✗ for each vetting check
- Server version + address used
- Player count and any notable states (offline, grouped, DSP-capable)
- Library totals and active music providers
- Paths of the snapshot files written under `$DATA_ROOT/data/`
- Next suggested commands: `/media-assistant-ops:status`, `/media-assistant-ops:list-players`

## Do not

- Do not write anything to `~/.claude/` or inside the plugin install directory.
- Do not echo the full API key after capture.
- Do not block onboarding if the library endpoints are unsupported — degrade gracefully with a note.
- Do not overwrite an existing `config.json` without explicit user confirmation.
