<!--
English translation of realtime/CLAUDE.md — the agent's context file for the WebSocket server. The original is in Ukrainian.
Production user names and file locations are redacted.
-->

# Cipher Duel — socket server

Node.js 22, ES modules, a single dependency `ws`. There are no game rules here: PHP is the single source of truth.

- `src/server.js` — `createRealtimeServer({ php, secret, … })`:
  - `/ws?code=…` + `Authorization: Bearer` — the socket is registered immediately under the key sha256(token), then PHP checks the token;
  - `/ws/me` + `Authorization: Bearer {session token}` — friends and invitations; the key `u{id}` comes from PHP (`php.userConnect`),
    `POST /publish-users` broadcasts the lists, disconnection — `php.userDisconnect` after the same 3 s;
  - `POST /publish` (from PHP, with the secret) — sends each player their own state and sets a timer for `deadline` → `php.expire`;
  - `POST /publish-emote` (from PHP, with the secret) — an emote `{type: 'emote'}` to one player's sockets (the sender's opponent);
  - a disconnection is reported to PHP after 3 s (reconnecting without going "offline"); ping every 25 s.
- `src/php.js` — client for Symfony's `/internal/*`. `src/main.js` — startup, `php.reset()` on start.
- Tests: `npm test` (`node:test`, PHP replaced by a fake). Protocol — `server/API.md`, the Realtime section.
- Production: a systemd service under its own unprivileged user; deployed together with the API by `server/deploy.sh`.
