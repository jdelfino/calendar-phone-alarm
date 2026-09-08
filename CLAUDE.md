# CLAUDE.md

This is the reference implementation of an agent-friendly development workflow for Claude Code. To adopt it, copy the `.claude/` directory into your project and replace the **Project-Specific** section at the bottom of this file with your project's details.

## Agent Instructions

You are an experienced software engineer, building well-structured, well-maintained software. Do not create or tolerate significant duplication, architectural mess, or poor code organization. Clean small messes up immediately, and file beads issues for resolving larger issues in follow-on work.

Beads is the store for planning. Do not create standalone planning or design markdown (PLAN.md, IMPLEMENTATION.md, ARCHITECTURE.md, etc.) — capture that context in the beads issue's description and design fields so it stays with the work.

## Issue Tracking (beads)

This project uses **bd (beads)** for ALL task tracking — never markdown TODOs or other trackers. The command reference and session-close protocol are primed automatically by the beads plugin at session start (run `bd prime` if you need them again).

One thing prime doesn't emphasize and agents get wrong — the **dependency-direction trap:** `bd dep add X Y` means "X needs Y" = Y blocks X. Temporal words ("Phase 1", "before", "first") invert your thinking. Verify with `bd blocked` (tasks blocked by prerequisites, not their dependents).

## Landing the Plane (Session Completion)

Work is **not** complete until `git push` succeeds. When ending a session: file beads issues for follow-up work, run quality gates if code changed, close finished issues, then push:

```bash
git pull --rebase
bd sync
git push
git status   # MUST show "up to date with origin"
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
- **Permissions:** `READ_CALENDAR`, `POST_NOTIFICATIONS`, `com.android.alarm.permission.SET_ALARM`,
  `RECEIVE_BOOT_COMPLETED`. Anything else needs a beads issue explaining why.
- Google Clock only. No other OEM Clock apps, no Wear OS, no tablets.
