---
name: status
description: Verify the plugin can reach the configured Music Assistant server. Hit /api/info with the stored API key and report server version, uptime, and player count. Use when the user asks whether the server is reachable, suspects a connectivity issue, or wants a quick health check before running other commands.
---

# Status — Music Assistant Connectivity Check

## Steps

1. Load config per the reference skill. If missing, tell the user to run `/media-assistant-ops:onboard`.

2. GET `$BASE/api/info` with the Bearer token. Use a 5-second timeout.

3. If the LAN request fails and a `wan_ip` is configured, retry against the WAN URL and note in the report that the LAN address is unreachable.

4. Report:
   - Server version
   - Uptime (if available)
   - Number of players (GET `/api/players` and count)
   - Which address was used (LAN or WAN)

5. On failure, print the HTTP status code or curl error and suggest: verify server is running, check firewall on port 8095, regenerate API key if 401/403.
