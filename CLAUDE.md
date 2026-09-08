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

Meeting Alarm: an Android app that reads the phone's calendar provider (Google Calendar syncs into it),
decides which meetings deserve a real alarm (criteria + per-meeting toggles), and creates/removes those
alarms in the Google Clock app via `AlarmClock.ACTION_SET_ALARM` / `ACTION_DISMISS_ALARM`. A morning
digest notification lists the day's meetings so alarms can be reviewed. The app never rings anything
itself. Design and phases: `PLAN.md` (written before beads was adopted; migrate into beads epics via
`/plan` rather than extending it).

Stack: Kotlin, Jetpack Compose, Room, WorkManager, Hilt. minSdk 31, target latest stable.
Google Clock only; no other OEM Clock apps, no Wear OS.

## Commands

```bash
./gradlew assembleDebug            # Build debug APK
./gradlew testDebugUnitTest        # Unit tests
./gradlew lint                     # Android lint
./gradlew connectedDebugAndroidTest  # Instrumented tests (device/emulator attached)
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

## Quality Gates

| Area            | Tests                                 | Lint              | Typecheck |
| --------------- | ------------------------------------- | ----------------- | --------- |
| App (JVM)       | `./gradlew testDebugUnitTest`         | `./gradlew lint`  | —         |
| Instrumented    | `./gradlew connectedDebugAndroidTest` | —                 | —         |

Instrumented tests require an attached device or emulator; skip them in review passes when none is
available and say so.

## Development Guidelines

- Alarm delivery is delegated to Google Clock. Do not add an in-app ringer, exact-alarm, full-screen-intent,
  or foreground-service code; if a need arises, file a beads issue and discuss first.
- Clock alarms are time-of-day only (no date). Only create alarms for meetings starting within 24 h.
- Every alarm the app creates is recorded in the Room ledger; Clock intents are only sent for the diff
  between desired and ledger. Labels carry a short unique token; dismiss searches by that token only
  (Clock's label match is a substring match and 2+ matches open a picker).
- Keep calendar access read-only (`READ_CALENDAR`); never write to the calendar provider.
- Business logic (criteria resolver, diffing, label tokens, digest content) lives in plain Kotlin with
  unit tests; Android framework calls sit behind thin interfaces so they can be faked.
- Permissions: `READ_CALENDAR`, `POST_NOTIFICATIONS`, `com.android.alarm.permission.SET_ALARM`,
  `RECEIVE_BOOT_COMPLETED`. Adding any other permission needs a beads issue explaining why.


<!-- BEGIN BEADS INTEGRATION v:1 profile:minimal hash:ca08a54f -->
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

## Session Completion

**When ending a work session**, you MUST complete ALL steps below. Work is NOT complete until `git push` succeeds.

**MANDATORY WORKFLOW:**

1. **File issues for remaining work** - Create issues for anything that needs follow-up
2. **Run quality gates** (if code changed) - Tests, linters, builds
3. **Update issue status** - Close finished work, update in-progress items
4. **PUSH TO REMOTE** - This is MANDATORY:
   ```bash
   git pull --rebase
   bd dolt push
   git push
   git status  # MUST show "up to date with origin"
   ```
5. **Clean up** - Clear stashes, prune remote branches
6. **Verify** - All changes committed AND pushed
7. **Hand off** - Provide context for next session

**CRITICAL RULES:**
- Work is NOT complete until `git push` succeeds
- NEVER stop before pushing - that leaves work stranded locally
- NEVER say "ready to push when you are" - YOU must push
- If push fails, resolve and retry until it succeeds
<!-- END BEADS INTEGRATION -->
