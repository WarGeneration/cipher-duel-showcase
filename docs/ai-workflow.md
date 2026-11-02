# Building Cipher Duel with an AI agent

This document explains how I built Cipher Duel together with Claude Code, an AI coding agent: how the work was set up,
what the agent was allowed to do, how its output was verified, and where human decisions mattered.
The examples are real; the git history is in the private source repository and I can show it on request.

## Roles

| I did | The agent did |
|---|---|
| Product idea, rules, game modes, visual style | Proposed the stack and architecture for each stage |
| Approved or changed every plan before work started | Wrote the code: app, API, realtime server, tests |
| Product and UX decisions on the agent's proposals | Set up and deployed the server, with my approval for each deploy |
| Tested every build on a real phone, reported bugs and UX issues | Kept documentation and context files up to date |
| Own all external accounts: VPS, domain, Google Cloud, Codemagic, Apple | Ran the full check suite before reporting a task as done |

The plan for upcoming work ([`NEXT_STEPS.md`](../context/NEXT_STEPS.md), translated) marks each step with who does it:
**[me]** (accounts, payments, keys) or **[Claude]** (code and configuration).

## 1. From a brief to a plan

The whole project started from this brief (`requirements.txt`, translated):

> I want to create my own Android app: a Bulls and Cows-style game called "Cipher Duel".
> For a start, one mode — against the computer; later we'll add two players on one phone and two players on different phones.
> All digits in the code must be unique. The code length must be configurable (4 to 10).
> When a player or the computer guesses a digit but not its position, the icon for that count should be a closed lock
> with a key; when the position is also right — an open lock with a key.
> The style should be a bit mysterious, dark, maybe purple tones.
> **Before we start, tell me which technologies and programming language you want to use; when I agree, we'll begin.**

The last line set the pattern for the whole project: the agent proposes, I approve, then it builds. The same rule is
written into the project instructions ("agree the stack and plan with the user before large changes").

## 2. Context engineering

An agent starts every session with no memory of previous ones. Instead of re-explaining the project each time,
the knowledge lives in files the agent reads automatically and is required to keep current:

- **[`CLAUDE.md`](../context/CLAUDE.md)** — project-wide rules: language, conventions, the required check command, git and
  security rules, where the plan lives.
- **[`mobile/CLAUDE.md`](../context/mobile-CLAUDE.md), [`server/CLAUDE.md`](../context/server-CLAUDE.md),
  [`realtime/CLAUDE.md`](../context/realtime-CLAUDE.md)** — architecture of each component, where things live, platform
  quirks and known pitfalls (for example, why a certain library behaves differently on SQLite in tests).
- **Persistent memory** (outside the repository) — facts that git can't tell: what is deployed to production versus
  only committed, server access details, decisions and preferences from earlier sessions.
- **`server/API.md`** — the protocol between app and server, the contract both sides are tested against.

Updating these files is part of the definition of done for every feature, so they don't rot.

## 3. Guardrails

| Rule | Why |
|---|---|
| Production secrets live only on the server; never in the repo, never pasted into chat | An agent's transcript and the repository are not secret stores |
| The agent never reads or copies the server's SSH private key | Least privilege; access is used, not handled |
| My e-mail is not sent to third-party services | Personal data stays local |
| No deploy, commit or push without an explicit "yes" from me | Outward-facing actions stay under human control |
| Ask before splitting a task across several parallel agents | Cost and coordination stay predictable |

## 4. Verification

The agent's statement "done" means nothing on its own; a set of checks does. Before finishing any task it must run:

```
cd mobile && ./gradlew :core:jvmTest :app:testDebugUnitTest verifyPaparazziDebug assembleRelease :shared:compileKotlinIosArm64
cd server && php bin/phpunit
cd realtime && npm test
```

Layers of verification and what each one catches:

- **Unit tests** for rules, AI, input logic, view models and timers (with a fake clock).
- **Shared rule vectors** (`shared/rules-test-vectors.json`): the game rules are
  implemented twice, in Kotlin (app) and PHP (server). Both test suites run the same scenarios, so a change in one
  implementation that isn't mirrored in the other fails the build. An excerpt:

  ```json
  "evaluate": [
    { "secret": "1234", "guess": "4321", "unlocked": 0, "locked": 4 },
    { "secret": "5049", "guess": "5094", "unlocked": 2, "locked": 2 },
    { "secret": "4586390", "guess": "5142386", "unlocked": 1, "locked": 4 }
  ]
  ```
- **Screenshot tests** (Paparazzi): every screen is rendered to PNG from a gallery of test states. When the UI changes,
  the agent re-records the images and reviews them before finishing.
- **End-to-end test:** two app view models play a whole online game against the real server.
- **Real device:** I play each build on my phone. This is the last layer and it found bugs the others missed (see 5.3).

## 5. Examples

### 5.1 A feature from request to commit: pinning digits

**My request:** long-press a cell to lock a digit you are sure of; locked digits glow green and carry over to the next
turn; long-press again to unlock. "If there are better ways to implement it, tell me before starting."

**The agent's proposal before writing code** added things I hadn't specified:
- a pinned cell can't be overwritten, selected or erased — otherwise the lock protects nothing;
- in the two-players-on-one-phone mode each player has their own pins;
- a small pin icon in the cell, so the state is visible without relying on colour alone;
- haptic feedback and a TalkBack action, because long-press is otherwise invisible to screen-reader users.

It also asked me two questions: how players will discover a hidden gesture, and whether pins should survive closing
the app in online games.

**My decisions:** a one-time hint that disappears for good after the first use; pins live only while the game is open.

**During implementation** the agent changed one thing and explained why: the game already uses lock icons to show
guess feedback, so a lock on the cell would be confusing — it used a pin icon and the word "pin" instead.
On reviewing the screenshot, it noticed the hint wrapped to two lines and shortened the text.

**Before reporting done:** new unit tests for the input logic and for per-player pins, a new screenshot,
the full check suite. The commit happened only after I said yes.

### 5.2 Working without an emulator or a Mac

The development machine is a Linux server: no hardware virtualisation (so no Android emulator) and no Mac (so no
local iOS builds). Instead of treating that as a blocker:

- the agent checks UI through screenshot tests and reads the rendered images itself;
- the same screen gallery is used by the iOS simulator in the cloud (Codemagic), which launches the app on each screen
  and takes screenshots, so both platforms are checked against the same states;
- the iOS code is cross-compiled on Linux as part of every check, so iOS breakage is caught immediately.

### 5.3 A bug the tests missed

After the screen gallery was introduced, opening a local game crashed on my phone. The cause: the Android activity
didn't register a back-navigation dispatcher that the game screen needs — and the screenshot gallery supplied its own
dispatcher, which hid the bug from every test. The fix came with a new test that launches the **real** activity under
Robolectric and walks from the menu into a game and back (commit `70ea77e`). Lesson recorded in the context file:
the gallery can hide integration problems.

### 5.4 A product decision the agent couldn't make: the computer's difficulty

The agent can make the computer play perfectly — that's the hard level. The question was how a *weaker* opponent
should behave so that it feels human rather than random. I chose a model where easy and medium remember only the last
one or two hints, and their memory grows as the game goes on (a player "getting into the game"), plus a rule that the
computer can't win by a lucky guess before a certain turn. I preferred this over tuning fixed turn ranges.

## 6. Time spent

About **10–14 hours** of active sessions with the agent, spread over **8 days** between 7 October and 2 November 2026,
with breaks of several days in between. I sent about **70 prompts** in that time.

How it was measured: from the timestamps in the Claude Code session logs. Consecutive events less than 15 minutes apart
count as one continuous stretch of work (10 hours in total); with a 30-minute threshold, which also covers long builds
and short checks on the phone, it is 13.5 hours.

Not included: my own time outside the sessions — testing builds on the phone, setting up the server, Google Cloud,
Codemagic and Apple accounts.

| Day | Main result |
|---|---|
| 7–8 Oct | Game against the computer, two players on one phone, online mode with a server on a VPS |
| 10 Oct | WebSocket server, accounts (guest + Google Sign-In), statistics, achievements |
| 11 Oct | Port to iOS (Kotlin Multiplatform), cloud iOS builds, screenshot gallery for both platforms |
| 13–14 Oct | Friends and invitations, emotes, avatars, a crash fix with a regression test |
| 21 Oct | Documentation, plan for the next stages |
| 2 Nov | Digit pinning; this write-up |

## 7. What I learned

- **Specify outcomes and constraints, let the agent propose the design — then decide.** Its proposals often covered
  edge cases I hadn't thought of; the product calls were still mine.
- **Tests are the interface to the agent's work.** Shared test vectors and screenshots let me trust changes across
  three codebases without reading every line.
- **Context files are infrastructure.** Keeping them current is what makes a new session productive from the first minute.
- **Automated checks are not enough on their own.** Real-device testing found what the test setup hid.
- **Next time:** start version control from the first hour. The first ~25,000 lines landed as a single initial commit,
  so the early history isn't visible.
- **Model can review its own work:** when you are working with some feature the model can be too focused on one task and
not checking other potential places of code or design. So after big task it is good to ask the model to review our current
work and check it for bugs, improvements etc.
