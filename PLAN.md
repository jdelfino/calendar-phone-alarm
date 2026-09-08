# Meeting Alarm for Android — Plan

Goal: turn meetings from Google Calendar into real, loud, lock-screen alarms on the phone.
Control lives in the app: a list of upcoming meetings with an alarm toggle per meeting, and a
morning notification showing the day's schedule so alarms can be reviewed before the day starts.

## 1. Constraints found in research (Sept 2026)

| Topic | Finding | Consequence |
|---|---|---|
| Google Calendar UI extension | Workspace add-ons can add a sidebar when an event is opened on desktop web, but **"Calendar add-ons aren't supported on mobile clients."** | Not pursued. All control is in our app. |
| Calendar data on the phone | Google Calendar's sync adapter mirrors every calendar on the account into Android's `CalendarContract` provider (events, instances, reminders, attendees, self attendee status). | `READ_CALENDAR` is all the integration needed. No OAuth, no server, no REST API, works offline, updates within minutes of an edit anywhere. |
| System Clock alarms via intents | `AlarmClock.ACTION_SET_ALARM` (hour, minutes, message, ringtone, vibrate, `EXTRA_SKIP_UI`) creates an alarm in the default Clock app; `ACTION_DISMISS_ALARM` removes one by label, time, next, or all. Needs only the normal permission `com.android.alarm.permission.SET_ALARM`. **No date extra**: an alarm is time-of-day only, i.e. the next occurrence within 24 h. If a dismiss matches 2+ alarms the Clock app is supposed to show a picker. | Primary approach. The app never rings anything itself; it creates today's alarms in Clock and dismisses them when a meeting is toggled off, moved, or cancelled. Alarms must be created same-day (morning job + mid-day sync). Labels must be unique. |
| Exact alarms (not needed) | Android 14+: `SCHEDULE_EXACT_ALARM` denied by default, user grants it once in Settings. `USE_EXACT_ALARM` is auto-granted but Play allows it only for alarm-clock/calendar apps (declaration required). `setAlarmClock()` is never deferred by Doze. | Not needed for the Clock-intent approach. The morning job uses an inexact `setAndAllowWhileIdle` window (minutes of drift is fine hours before a meeting). |
| Full-screen intent (not needed) | Android 14+: enabled by default on install; Play revokes it for non-alarm/calling apps. Check with `NotificationManager.canUseFullScreenIntent()`, prompt via `ACTION_MANAGE_APP_USE_FULL_SCREEN_INTENT`. | Not needed for the Clock-intent approach. |
| Android 16 | No new alarm/notification/FSI restrictions for target 36. | Nothing extra to plan for. |
| Notification actions | A notification can carry at most 3 action buttons; per-item toggles inside one notification are not possible. | Digest notification lists the schedule and deep-links to the Today screen for per-meeting toggles; bulk actions only on the notification itself. |
| Prior art | Play apps such as "Calendar Alarm Clock Reminder" ring for every event; Tasker "Calendar Entry" can set a system alarm N minutes before events. | Per-meeting opt-in/out plus the morning review is the differentiator; Tasker is a zero-code fallback. |

## 2. Product design

### Screens
- **Today / Agenda** (home): next 7 days grouped by day. Each row: time, title, calendar color,
  attendee count, join-link icon, and an **alarm switch**. Header shows "Next alarm: 9:55 for Standup".
  Rows that don't meet the criteria are shown dimmed with the switch off (can still be turned on).
- **Meeting detail** (tap row): lead time override for this meeting, "this instance / all in series"
  choice for recurring meetings, join link, attendees, open in Google Calendar.
- **Settings**: criteria (min other attendees, calendars included, ignore all-day, ignore declined,
  ignore "free"), default lead time, default state for qualifying meetings (armed / not armed),
  digest time and on/off, permission checklist. (Sound, volume, snooze are Clock's settings, not ours.)

### Alarm state model
- Every meeting *instance* resolves to armed/not armed as: **user override if present, else the
  criteria-based default.** Overrides are stored in Room keyed by (event id, instance start) so they
  survive calendar re-syncs and app restarts. Series-wide overrides are keyed by event id.
- Default: **off**. The criteria (≥1 other attendee, not declined, not all-day, not marked free) decide
  which meetings are *eligible* and which days get a digest; only an explicit toggle (per instance or
  whole series) arms an alarm. The morning digest is where alarms get turned on.
- If a meeting is cancelled, moved, or declined after being armed, the alarm follows it (re-validated
  on every sync and again right before ringing).

### Morning digest notification
- Posted at a configurable time (e.g. 07:30) **only if the day has at least one meeting meeting the
  criteria**. Silent days produce nothing.
- Content (`InboxStyle`, expandable): one line per meeting, e.g. `🔔 09:55  Standup (4)` /
  `🔕 13:00  1:1 with Sam (2)` / `—  15:00  Focus block`. Collapsed line: "3 meetings today, 2 alarms".
- Actions: **Open** (Today screen, where each row has its switch), **Alarm all**, **No alarms today**.
- Stays until dismissed or end of day; re-posted (updated, not re-alerted) when the day's meetings change.
- Optional later: evening-before digest when tomorrow's first meeting starts before digest time;
  a quiet "New meeting at 15:00 — alarm armed" notification when something is added mid-day.

### The alarm itself: delegated to the system Clock app
- Toggle on (or criteria default) → `ACTION_SET_ALARM` with `EXTRA_SKIP_UI=true`, hour/minute = start minus
  lead time, `EXTRA_MESSAGE` = the meeting title plus a short unique token, e.g. `"Standup ⏰a3f9"`.
- Toggle off, meeting moved, cancelled, or declined → `ACTION_DISMISS_ALARM` with `ALARM_SEARCH_MODE_LABEL`
  and `EXTRA_MESSAGE` = the token; moved meetings then get a fresh alarm.
- **Cleanup is automatic in AOSP/Google Clock** (verified in the DeskClock source, `HandleApiCalls.kt` and
  `AlarmStateManager.kt`): a non-repeating alarm created with `EXTRA_SKIP_UI=true` is flagged
  `deleteAfterUse`, so it is **deleted** from the alarm list (not just disabled) both when it fires and is
  dismissed and when our app dismisses it early via `ACTION_DISMISS_ALARM`. Nothing accumulates.
- Label matching in the dismiss handler is a **substring** match, and 2+ matches open a picker activity,
  so the unique token is what keeps dismiss silent; never rely on the title alone.
- Dismiss only considers *enabled* alarms with an upcoming instance; an alarm the user manually switched
  off in Clock is left alone (that is fine: it will not ring).
- Because the intent has no date, alarms are only ever created for meetings starting within the next 24 h.
  A single sync routine (desired vs. ledger → set/dismiss the difference) runs on every trigger: periodic
  worker, digest time, boot, timezone change, calendar reminder broadcast, user toggle.
- Ringing, snooze, DND exemption, lock-screen UI, status-bar icon, and volume all come from Clock for free.
- Scope: **Google Clock only** (Pixel / AOSP-derived). Other OEM Clock apps are out of scope.
- Limitation accepted for now: no Join button on the alarm screen (the Clock alarm shows only the label).

## 3. Architecture

```
CalendarContract (Instances, Events, Reminders, Attendees)
        │  query next 7 days
        ▼
CalendarRepository ──► AlarmResolver (criteria + Room overrides) ──► desired alarms (next 24 h)
        ▲                                                              │ diff vs. alarms we created (Room)
        │ refresh triggers                                             ▼
  - morning job (inexact setAndAllowWhileIdle)                ClockBridge
  - WorkManager periodic (15 min)                              ├─ ACTION_SET_ALARM   (skip UI, unique label)
  - CalendarContract.ACTION_EVENT_REMINDER broadcast           └─ ACTION_DISMISS_ALARM (search by label)
    (provider-fired; verify in spike 3)
  - BOOT_COMPLETED, TIME/TIMEZONE_CHANGED                     DigestReceiver → post digest if any qualifying meeting
  - ContentObserver while app is visible
  - user toggles in app / digest actions
```

Notes:
- The app keeps a ledger in Room of every alarm it created (label, event id, instance start). Each sync
  diffs desired vs. ledger and issues set/dismiss intents only for the difference, so Clock is never spammed.
- Toggling a switch runs the diff immediately.
- Onboarding checklist: calendar permission, notifications, battery-optimisation exemption for the app
  (so the morning job and syncs run), and a one-time check that the default Clock app handled a test
  set + dismiss without showing UI.

Permissions: `READ_CALENDAR`, `POST_NOTIFICATIONS`, `com.android.alarm.permission.SET_ALARM`,
`RECEIVE_BOOT_COMPLETED`. 

Stack: Kotlin, Jetpack Compose, Room, WorkManager; no DI framework (hand-written AppGraph); minSdk 33, target latest.
Distribution: sideload / Play internal testing first.

## 4. Phases

**Phase 0 — spikes (1–2 evenings, de-risk before building UI)**
1. Query the provider on your phone and confirm attendee counts, self attendee status (declined),
   and availability (free/busy) are populated for Google invites.
2. Throwaway app on your actual phone: `ACTION_SET_ALARM` with skip-UI, then `ACTION_DISMISS_ALARM` by
   label token. Confirm: no UI appears, the alarm rings at the right time, early dismiss *deletes* it
   from the Clock list, and a fired-then-dismissed alarm also disappears.
3. Manifest receiver for `android.intent.action.EVENT_REMINDER`; confirm it fires (free second trigger path).

**Phase 1 — MVP**: criteria + overrides resolver, agenda screen with switches, alarm ledger + Clock bridge,
morning job, onboarding permissions.

**Phase 2 — digest**: morning notification with bulk actions and deep link, mid-day change handling,
reboot/timezone handling.

**Phase 3 — polish**: series vs instance overrides, per-meeting lead time, join-link extraction shown in the app/digest.


## 5. Alternatives considered

- **Marking events inside Google Calendar** (sentinel reminder, event color, web add-on): works
  cross-client but is a hidden convention and the web add-on is desktop-only. Rejected in favour of
  explicit in-app control.
- **Calendar REST API + extended properties**: adds OAuth, polling, and a second copy of data the
  phone already has. Not needed.
- **In-app ringer** (`setAlarmClock` + foreground service + full-screen activity): would give a Join
  button on the alarm screen, but costs three extra permissions and a second alarm implementation.
  Not planned; Google Clock does the job.
- **Tasker**: Calendar Entry profile → set alarm. Fastest path to something usable today, but no
  per-meeting toggles or digest.
- **Off-the-shelf Play apps**: ring for all events; no per-meeting review flow.

## Sources
- Calendar add-ons (web only): https://developers.google.com/workspace/add-ons/calendar
- Android 14 exact alarms: https://developer.android.com/about/versions/14/changes/schedule-exact-alarms
- Schedule alarms / setAlarmClock: https://developer.android.com/develop/background-work/services/alarms
- Play exact-alarm policy: https://support.google.com/googleplay/android-developer/answer/13161072
- Full-screen intent limits: https://source.android.com/docs/core/permissions/fsi-limits
- Play FSI / FGS requirements: https://support.google.com/googleplay/android-developer/answer/13392821
- Android 16 behavior changes: https://developer.android.com/about/versions/16/behavior-changes-16
- CalendarContract: https://developer.android.com/reference/android/provider/CalendarContract
- AlarmClock intents: https://developer.android.com/reference/android/provider/AlarmClock
- AOSP DeskClock intent handling (deleteAfterUse, dismiss by label): https://android.googlesource.com/platform/packages/apps/DeskClock/+/master/src/com/android/deskclock/HandleApiCalls.kt
- AOSP DeskClock parent-alarm cleanup: https://android.googlesource.com/platform/packages/apps/DeskClock/+/master/src/com/android/deskclock/alarms/AlarmStateManager.kt
- Common intents (set alarm): https://developer.android.com/guide/components/intents-common
- Tasker calendar alarms: https://www.xda-developers.com/tasker-pro-calendar-based-alarm/
- Prior art: https://play.google.com/store/apps/details?id=com.app_by_LZ.calendar_alarm_clock , https://play.google.com/store/apps/details?id=sk.mildev84.reminder
