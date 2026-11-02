<!--
English translation of the project's root CLAUDE.md — the instructions the AI agent (Claude Code) reads at the start
of every session. The original is in Ukrainian. Server addresses and other deployment details are redacted.
-->

# Cipher Duel

Android game "Bulls and Cows": a code of unique digits (4–10, 0 allowed). Modes: against the computer
(easy / medium / hard), two players on one phone, online on two phones. Rock–paper–scissors decides who goes first.
The original brief is in `requirements.txt`.

## Structure
- `mobile/` — Android and iOS (Kotlin Multiplatform, Compose Multiplatform), see `mobile/CLAUDE.md`;
  iOS is built in Codemagic (`codemagic.yaml`) because this machine is Linux without a Mac
- `server/` — online-mode API (PHP 8.5, Symfony 7.4), see `server/CLAUDE.md`
- `realtime/` — WebSocket server (Node.js 22, `ws`), only delivers states; see `realtime/CLAUDE.md`
- `shared/rules-test-vectors.json` — rule scenarios that both the Kotlin and the PHP tests run. If you change the rules, update them.

## What already exists
Games against the computer and for two players, online (rooms by code, turn timer, sockets), accounts (guest + Google),
statistics, history, achievements, friends and invitations (links `/f/…`, `/r/…`), emotes in online games, 5 ready-made
avatars, choosing which digit position to type, and pinning guess digits with a long press. The iOS port compiles and runs
in the Codemagic simulator, but there is no signed build yet.
The plan of next steps is `NEXT_STEPS.md` (update it when a step is done). What is deployed to production is in Claude's
memory (`MEMORY.md`).

## Agreements
- The user writes in Ukrainian; answer in Ukrainian. The UI is English and Ukrainian, every string in both `strings.xml`.
- Comments only in critical places, in Ukrainian.
- Agree the stack and the plan with the user before large changes.
- Before a task that is better done by several agents, tell the user and wait for consent.
- Production secrets live only on the server; do not read or copy the server SSH key.
- Do not pass the user's e-mail to third-party services.
- Git: private repository, branch `main`; the author is the GitHub noreply address. Commits end with a
  `Co-Authored-By: Claude …` line.

## Checks before finishing a task
```
cd mobile && ./gradlew :core:jvmTest :app:testDebugUnitTest verifyPaparazziDebug assembleRelease :shared:compileKotlinIosArm64
cd server && php bin/phpunit
cd realtime && npm test
```
If you changed how screens look — run `./gradlew recordPaparazziDebug` and review the PNGs in `mobile/app/src/test/snapshots/images/`.
