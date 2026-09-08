# CLAUDE.md

## Agent Instructions

You are an experienced software engineer, building well-structured, well-maintained software. Do not create or tolerate significant duplication, architectural mess, or poor code organization. Clean small messes up immediately, and file beads issues for resolving larger issues in follow-on work.

Beads is the store for planning. Do not create standalone planning or design markdown (PLAN.md, IMPLEMENTATION.md, ARCHITECTURE.md, etc.) — capture that context in the beads issue's description and design fields so it stays with the work.

## Issue Tracking (beads)

All task tracking lives in **bd (beads)**; the managed *Beads Issue Tracker* block at the end of this file
(written by `bd setup claude`, verified with `bd setup claude --check`) is the authoritative usage guide,
and `bd prime` is injected at session start by the beads plugin.

**This repository opts in to the Team-maintainer profile** described in that block: agents may close
beads, run quality gates, commit, and push as part of session close, unless a current instruction says
not to.

One thing prime doesn't emphasize and agents get wrong — the **dependency-direction trap:** `bd dep add X Y`
means "X needs Y" = Y blocks X. Temporal words ("Phase 1", "before", "first") invert your thinking.
Verify with `bd blocked` (tasks blocked by prerequisites, not their dependents).

## Landing the Plane (Session Completion)

Work is **not** complete until `git push` succeeds. When ending a session: file beads issues for
follow-up work, run quality gates if code changed, close finished issues, then push both code and
issue data:

```bash
git pull --rebase
git push
bd dolt push        # issue data lives under refs/dolt/data on origin
git status          # MUST show "up to date with origin"
```

Never stop before pushing — that strands work locally. If the push fails, resolve and retry until it succeeds.

---

# Project-Specific

## Project Overview

Calendar Phone Alarm is a personal Android app that makes sure I do not miss meetings. It reads the
phone's calendar provider (Google Calendar syncs into it), shows upcoming meetings with an alarm toggle
each, and creates or removes matching alarms in the **Google Clock app** through the public
`AlarmClock.ACTION_SET_ALARM` / `ACTION_DISMISS_ALARM` intents. A morning digest notification lists the
day's meetings so alarms can be reviewed. The app never rings anything itself.

Repo: https://github.com/jdelfino/calendar-phone-alarm. Single developer, sideloaded, one phone
(Pixel / Google Clock). Not published to Play.

The original design write-up is `PLAN.md`. It predates beads; do not extend it. Planning context goes
into beads issues.

## Stack

- Kotlin, single Gradle module `app`, Kotlin DSL build files, version catalog (`gradle/libs.versions.toml`).
- Jetpack Compose (Material 3) for UI. Room for persistence. WorkManager for background sync.
  kotlinx.coroutines / Flow throughout.
- **No dependency-injection framework.** A hand-written `AppGraph` (created in `Application.onCreate`)
  builds and exposes the database, repositories, and use cases. Constructor injection everywhere so tests
  can pass fakes.
- minSdk 33, targetSdk = compileSdk = latest stable. Android 13+ only, so `POST_NOTIFICATIONS` is a
  runtime permission everywhere and no API-level branching is needed.

## Commands

```bash
./gradlew testDebugUnitTest   # JVM unit tests (plain JUnit + Robolectric)
./gradlew lint                # Android lint
./gradlew assembleDebug       # Debug APK -> app/build/outputs/apk/debug/app-debug.apk
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

## Quality Gates

| Area   | Tests                         | Lint             | Build                    |
| ------ | ----------------------------- | ---------------- | ------------------------ |
| app    | `./gradlew testDebugUnitTest` | `./gradlew lint` | `./gradlew assembleDebug` |

All gates run on the JVM with no device. UI/instrumented tests are **not** required. CI (GitHub Actions)
runs the same three commands on every push and pull request.

## Testing Conventions

- Business logic (criteria, alarm resolution, ledger diffing, label tokens, digest text, time windows)
  is plain Kotlin with no Android imports and is tested with JUnit.
- Code that must touch Android classes (Room DAOs, `CalendarContract` cursor mapping, `Intent`
  construction, notification building) is tested with Robolectric.
- Android framework calls sit behind small interfaces (`CalendarSource`, `ClockAlarms`, `Clock`,
  `Notifier`) so use cases are tested with in-memory fakes, not mocks.
- Every time-dependent computation takes a `Clock`/`Instant` parameter; tests never depend on wall time.

## Product Rules (do not change without a beads issue)

- **Alarm delivery is Google Clock's job.** No in-app ringer, no `SCHEDULE_EXACT_ALARM`, no full-screen
  intent, no foreground service. `AlarmManager` is used only for the inexact daily digest trigger.
- **Default is off.** A meeting gets an alarm only if the user toggled it on (per instance or for the
  whole series). The criteria (has at least one other attendee, not declined, not all-day, not marked
  free) decide which meetings are *eligible* and which days get a digest, not which meetings are armed.
- **Clock alarms are time-of-day only.** Only create alarms for meetings starting within the next 24 h.
  Every trigger (periodic sync, digest time, boot, timezone change, calendar reminder broadcast, user
  toggle) runs the same sync: desired alarms vs. ledger, then set/dismiss the difference.
- **Ledger.** Every alarm the app creates is recorded in Room (event id, instance start, label token).
  Labels are `<meeting title> ⏰<token>`; dismiss searches by the token alone because Clock's label match
  is a substring match and two or more matches open a picker.
- **Calendar access is read-only** (`READ_CALENDAR`). Never write to the calendar provider.
- **Background launches.** Clock's intents are handled by an Activity, and Android 10+ blocks
  `startActivity` from the background. The app asks for `SYSTEM_ALERT_WINDOW` ("Display over other apps"),
  the documented exemption. When the app is neither visible nor overlay-permitted, a sync must not send
  Clock intents: it leaves the ledger alone and posts an "alarm changes waiting" notification instead.
  Notification actions that lead to Clock intents must be activity PendingIntents, never receivers.
- **Permissions:** `READ_CALENDAR`, `POST_NOTIFICATIONS`, `com.android.alarm.permission.SET_ALARM`,
  `SYSTEM_ALERT_WINDOW`, `RECEIVE_BOOT_COMPLETED`. Anything else needs a beads issue explaining why.
- Google Clock only. No other OEM Clock apps, no Wear OS, no tablets.


<!-- BEGIN BEADS INTEGRATION v:1 profile:minimal hash:970c3bf2 -->
## Beads Issue Tracker

This project uses **bd (beads)** for issue tracking. Run `bd prime` to see full workflow context and commands.

### Quick Reference

```bash
bd ready              # Find available work
bd show <id>          # View issue details
bd update <id> --claim  # Claim work
bd close <id>         # Complete work
```

### Rules

- Use `bd` for ALL task tracking — do NOT use TodoWrite, TaskCreate, or markdown TODO lists
- Run `bd prime` for detailed command reference and session close protocol
- Use `bd remember` for persistent knowledge — do NOT use MEMORY.md files

**Architecture in one line:** issues live in a local Dolt DB; sync uses `refs/dolt/data` on your git remote; `.beads/issues.jsonl` is a passive export. See https://github.com/gastownhall/beads/blob/main/docs/SYNC_CONCEPTS.md for details and anti-patterns.

## Agent Context Profiles

The managed Beads block is task-tracking guidance, not permission to override repository, user, or orchestrator instructions.

- **Conservative (default)**: Use `bd` for task tracking. Do not run git commits, git pushes, or Dolt remote sync unless explicitly asked. At handoff, report changed files, validation, and suggested next commands.
- **Minimal**: Keep tool instruction files as pointers to `bd prime`; use the same conservative git policy unless active instructions say otherwise.
- **Team-maintainer**: Only when the repository explicitly opts in, agents may close beads, run quality gates, commit, and push as part of session close. A current "do not commit" or "do not push" instruction still wins.

## Session Completion

This protocol applies when ending a Beads implementation workflow. It is subordinate to explicit user, repository, and orchestrator instructions.

1. **File issues for remaining work** - Create beads for anything that needs follow-up
2. **Run quality gates** (if code changed) - Tests, linters, builds
3. **Update issue status** - Close finished work, update in-progress items
4. **Handle git/sync by active profile**:
   ```bash
   # Conservative/minimal/default: report status and proposed commands; wait for approval.
   git status

   # Team-maintainer opt-in only, unless current instructions forbid it:
   git pull --rebase
   bd dolt push
   git push
   git status
   ```
5. **Hand off** - Summarize changes, validation, issue status, and any blocked sync/commit/push step

**Critical rules:**
- Explicit user or orchestrator instructions override this Beads block.
- Do not commit or push without clear authority from the active profile or the current user request.
- If a required sync or push is blocked, stop and report the exact command and error.
<!-- END BEADS INTEGRATION -->
