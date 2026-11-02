<!--
English translation of server/CLAUDE.md — the agent's context file for the API server. The original is in Ukrainian.
The production section (server address, SSH access, secret locations) is summarised without those details.
-->

# Cipher Duel — server

PHP 8.5, Symfony 7.4 LTS, Doctrine ORM 3 / DBAL 4, MySQL 8.4. The protocol is described in `API.md` — keep it up to date.

## Code
- `src/Game/` — rules without a database: `Code`, `Feedback`, `Duel::replay`, `Seat`, `RpsChoice`, `GameException` (error codes → HTTP statuses).
- `src/Entity/Room` — all room logic: `addPlayer`, `setSecret`, `playRps`, `guess`, `leave`, `expireIfDue`, `pendingSeats`.
  Every state change increments `version`. Timer: `deadlineAt`; `turnSeconds` is chosen by the room creator (0 — no timers).
- `src/Service/RoomService` — transactions: `act()` locks the room row (`FOR UPDATE`; skipped on SQLite, otherwise DBAL 4 throws)
  and, if time is up, finishes the game instead of performing the action. After commit every change goes to `RoomPublisher::publish()`.
- `src/Service/RealtimePublisher` — POSTs the state for each player to the socket server (`REALTIME_URL`); errors are only logged.
  In tests it's replaced by `tests/Support/RecordingPublisher` (see `when@test` in `config/services.yaml`).
- `src/Controller/Internal/` — `/internal/*` for the socket server: connect / disconnect / expire / reset, protected by a shared secret.
  Presence (`Player::online`) is changed only by the socket server.
- `src/Service/RoomView` — the state from one player's point of view; the opponent's code only in `result`.
- Accounts: `Entity/User` (nickname, `google_sub`, statistics) + `UserSession` (a token hash per device), `AccountService`.
  Symfony Security, firewall `api` with `access_token` → `Security/SessionTokenHandler`; room actions (`room`) and `/internal` are
  outside it because they have their own tokens. Creating/joining a room — with a session; `Player::user` links a player to an account.
  Statistics are updated in `Room::finish()` only for games where the duel started. Limits — `config/packages/rate_limiter.yaml`.
- Google: `POST /api/auth/google` → `AccountService::google()` (sign in / new account / link / switch).
  The ID token is verified by `Security/GoogleJwtVerifier` (Google's keys are cached, `aud` is one of the configured client IDs);
  in tests — `tests/Support/FakeGoogleVerifier` (token `google:{sub}`).
- Game history: `Entity/HistoryEntry` (the app's record JSON, a unique `client_id` per account), `Service/HistoryService`
  (up to 500 latest), `GET/POST/DELETE /api/me/history`.
- Achievements: the app computes the rules; the server only stores JSON in `User::achievements` (counters add up,
  unlock time — the earliest), `GET/POST /api/me/achievements`.
- The app's E2E tests create accounts and hit the limit of 10/hour per IP: reset it by clearing the rate-limiter cache pool on the server.
- Friends: `Entity/Friendship` (a pair ordered by id, `requester`, `acceptedAt`), `Service/FriendService`, `/api/me/friends`.
  `User::friendCode` — the code of the `/f/{code}` link (friendship without confirmation), `User::online` — an open `/ws/me` socket.
  Game invitations — `Room::invitedUser` (`POST /api/rooms` with `friendId`, `…/decline`). After every change
  `FriendService::notify()` sends a fresh list to online users via `RealtimeFriendsPublisher` (`/publish-users` in Node);
  in tests — `tests/Support/RecordingFriendsPublisher`.
- Emotes (`src/Game/Emote.php`) and avatars (`src/Game/Avatar.php`) are enums whose ids match the app (checked by `EmotesTest` /
  `AvatarsTest` in mobile). Avatar: `User::avatar` (random for a new account), `PATCH /api/me {nickname?, avatar?}`
  (`AccountService::updateProfile`), `User::person()` — `{id, nickname, avatar}` in friend lists; in `RoomView` — `you/opponent.avatar`.
  Tests comparing exact arrays set the avatar explicitly via PATCH.
- `src/Controller/LinkController` — pages for the `/f/{code}`, `/r/{code}` links with a `cipherduel://…` button;
  `public/.well-known/assetlinks.json` — Android App Links (SHA-256 of the release and debug keys; add the Play key with Google Play App Signing).
- `src/Controller/Api/` — `/api/rooms…`, `/api/health`. `ApiExceptionSubscriber` turns errors into JSON.
- `src/Command/CleanupRoomsCommand` (`app:rooms:cleanup`) — nightly cleanup; `ExpireRoomsCommand` (`app:rooms:expire`) — every minute,
  a fallback path for the timer. Cron in `deploy/cron`.
- Long polling was removed: `GET /api/rooms/{code}` responds immediately.

## Development
```
docker compose up -d --wait        # local MySQL
php bin/console doctrine:migrations:migrate
php bin/phpunit                    # SQLite, no Docker needed
```
A new column → a migration in `migrations/` and `doctrine:schema:validate` against MySQL in Docker.

## Production (summary; addresses, access and secret locations redacted)
- A single VPS (Ubuntu): Nginx + PHP-FPM + MySQL, Let's Encrypt; the agent connects with a dedicated SSH key it may use but never read.
- `./deploy.sh`: tests (PHP + Node) → provisioning (nginx, cron, systemd, secret) → rsync of the API and `../realtime` →
  composer → migrations and cache → restart of the realtime service → health checks of both.
- Nginx: `/api` and `/ws` are public (the latter proxied to Node); `/internal/` returns 404 from outside and is reachable
  only by the Node service through a separate localhost-only server block.
- Secrets live only on the server and must never be printed.
- Pitfalls: Ubuntu's `nginx.conf` already sets `server_tokens` (don't duplicate it); `pkill -f` with a pattern that also
  appears in the ssh command itself kills that command.
- After manual checks on production, delete test rooms.
