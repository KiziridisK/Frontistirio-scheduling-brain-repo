# Frontistirio — Lesson Scheduling Brain

> **Full-stack reference** for the automatic lesson-scheduling (timetable) subsystem.
> Documents *exactly* how the generator works: every input field it reads, the data
> model it writes, the greedy placement algorithm step by step, the drag-and-drop
> move re-validation, the frontend grid, and — most importantly — **where to change
> behaviour** for future parametrization.
>
> Added June 2026. Verify line numbers against current code before relying on them;
> field paths and function names are the stable contract.

---

## Table of contents

- [Changelog — 2026-09: manual scheduling (hand placement)](#changelog--2026-09-manual-scheduling-hand-placement)
0. [Changelog — 2026-08 rework](#0-changelog--2026-08-rework)
1. [What it does (concept)](#1-what-it-does-concept)
2. [End-to-end flow](#2-end-to-end-flow)
3. [Inputs — the data the algorithm consumes](#3-inputs--the-data-the-algorithm-consumes)
4. [Output data model — `Schedule`](#4-output-data-model--schedule)
5. [Backend API (routes + controller)](#5-backend-api-routes--controller)
6. [The algorithm — `controllers/handlers/schedule.js`](#6-the-algorithm--controllershandlersschedulejs)
7. [Drag-and-drop move re-validation](#7-drag-and-drop-move-re-validation)
8. [Frontend (page, grid, DnD)](#8-frontend-page-grid-dnd)
9. [Constraints reference](#9-constraints-reference)
10. [Parametrization & extension points](#10-parametrization--extension-points)
11. [Known limitations & failure modes](#11-known-limitations--failure-modes)
12. [Data prerequisites & troubleshooting](#12-data-prerequisites--troubleshooting)
13. [File map](#13-file-map)

---

## Changelog — 2026-09: manual scheduling (hand placement)

> **A whole new mode was added alongside auto-generate: the admin can now build
> the timetable BY HAND**, dragging lessons from a palette onto the grid, with a
> collected "missing data" list telling them exactly what to fix. This is the
> authoritative description of that feature; where it changes earlier behaviour
> the relevant section carries an **`UPDATED 2026-09`** note. Implemented in code
> (pending the user's API deploy + web build/OTA at the time of writing).

### N.1 Concept — τμημα-centric palette (vs the teacher-centric generator)

The auto-generator builds its work list from **`Teacher.period_courses`**
(teacher-centric): a course a teacher isn't assigned to simply never appears, so
a class course with no teacher vanishes silently. The manual palette instead
reads **`ClassModel.period_courses[period].courses[]`** (τμημα-centric) — the
source of truth for *what a τμήμα must be taught* — so a course with no teacher
or no frequency/hours is **surfaced as an issue** instead of disappearing.

- **Palette** = what still needs placing, grouped **τάξη (`Grade`) → τμήμα
  (`ClassModel`) → course**. Frequency/hours come from `Course.period_schedule`;
  a course with `sessions_per_week = 2` yields **2 draggable chips**, each
  showing the course name + that session's `hours_per_session[i]` duration.
- **Private lessons (ιδιαίτερα)** are gathered separately, from
  `Teacher.period_courses[].student_ids`, grouped **by student**.
- **"Remaining" is a multiset subtraction:** for each obligation, the required
  per-session hours (`hours_per_session`) minus the durations of entries already
  on the timetable — so `[2,1]` with a 2h lesson already placed leaves one `[1]`
  chip. Placing (auto **or** manual) consumes chips; the palette shows only
  what's left. `remainingHours(required, placed)` in the handler.

### N.2 Missing-data issues (the "ελλιπή δεδομένα" list)

A τμήμα course that **cannot** be placed is put in `issues[]` (not the palette),
each with a description + a link to the course to fix it:

| kind | meaning |
|---|---|
| `no_teacher` | course assigned to the τμήμα but no teacher teaches it (no `Teacher.period_courses` names the course **with that class in `class_ids`**) |
| `no_frequency` | `Course.period_schedule[period].sessions_per_week ≤ 0` (or missing) |
| `no_hours` | `hours_per_session` missing / shorter than `sessions_per_week` / has a non-positive value |

Private obligations get `no_frequency` / `no_hours` too (never `no_teacher` — a
private lesson has a teacher by construction). Resolved by `findClassTeacher()`
+ `courseSessions()` inside `buildManualPalette()`.

### N.3 Placement is an admin OVERRIDE (supersedes §7's strict rejection)

Both hand placement **and** the existing drag-to-move now **override** hard
constraints instead of rejecting: a lesson dropped on a conflicting cell is
**still placed**, but flagged. Only a **structurally impossible** target is
rejected (fatal): `invalid_slot` (no such start slot / block overflows the grid)
or `room_not_found`. Everything else — teacher/room/student unavailable or
busy, capacity, on-break — is a **conflict**, collected and returned as
`warnings[]`, and stamped on the entry. New `ScheduleEntry` fields
(`models/schedule.js`):

- **`manual: Boolean`** — placed/moved by hand (vs the generator). Used so
  re-generate can warn before discarding hand-made work.
- **`has_conflict: Boolean`** + **`conflict_reasons: [String]`** — the entry sits
  on a hard-constraint violation; the reasons are `schedule.reasons.*` i18n keys.

The frontend renders a conflict entry with a red outline + a "Σύγκρουση" badge
(reasons in the `title`), and a warning toast on drop. **`moveEntry` no longer
returns 409 on conflict** — it returns 200 with `warnings[]`; only fatal targets
return `success:false`.

### N.4 Shared placement core

`moveEntry` was rewritten to share one validator with the new `placeEntry`:

- **`evaluatePlacement(ctx, grid, occupancy, params)`** — the single source of
  placement rules. Returns `{ fatal }` (impossible) or `{ fatal:null, conflicts[],
  startTime, endTime }`. Half-open overlap + break checks as everywhere else.
- **`buildOccupancy(entries, excludeId)`** — teacher/room/student interval lists
  keyed `id|day`, built from the placed entries (excluding the one being moved).
- **`gridFromTimeSlots(time_slots, D)`** — wraps a **persisted** `time_slots`
  array in the same `{numSlots, slotStart, slotEnd, time_slots}` interface
  `buildSlotGrid` returns. **The persisted grid is now canonical:** once a
  schedule exists, place/move map onto *its* `time_slots` (not a recomputed grid)
  so a drop lands on exactly the slots the frontend rendered. Only when no
  schedule exists yet does `placeEntry` compute a fresh `buildSlotGrid` and
  persist it (creating the `Schedule` on first manual use — no auto-generate
  required to start hand-building).

### N.5 Auto-generate now confirms before wiping manual work

`generate` still fully regenerates (replaces `entries`), but the frontend now
**asks first** (`AlertController`) whenever `schedule.entries` contains any
`manual` entry — "these hand placements will be lost, continue?" — and only
regenerates on confirm. (The user chose confirm-and-warn over a preserve/lock
merge.)

### N.6 Grid shows all WORKING days (fix: "only Monday shows")

`buildGrid` derived day columns **only from days that had entries**, so a
timetable whose lessons all fell on one day (or a fresh one being hand-built)
showed a single column and you couldn't drop onto other days. `getManualData`
now returns **`days[]`** = every day some teacher **or** room has availability
this period (fallback Mon–Sat); the grid uses **those ∪ entry days**. So every
working day is a drop target regardless of what's already placed. *(If the grid
still shows one day after this, the cause is data: availability configured for
only that day in the period — availability is per-period.)*

### N.7 Backend surface (new)

`gatherContext` now also returns the **`classes`** array (τμήματα) for the
τμημα-centric palette. New handler exports + routes (all `authMiddleware` +
`isStoreUser`, tagged `// NEW ROUTE`):

| Method | Path | Handler | Purpose |
|---|---|---|---|
| GET  | `/schedule/manual-data` | `getManualData` | grid `time_slots` + `days[]` + `classPalette` + `privatePalette` + `issues` + current `schedule`, in one call |
| POST | `/schedule/place-entry` | `placeEntry` | hand-place one dragged session (override); creates the schedule + persists the grid on first use; re-resolves student set/labels server-side (never trusts the client) |
| POST | `/schedule/remove-entry` | `removeEntry` | remove one entry (return it to the palette) |

`placeEntry`/`moveEntry` return `{ success, message, warnings, schedule }` and
HTTP **200 even on a conflict override** (409 only on a fatal target).

### N.8 Frontend

- **Palette panel** (collapsible, `paletteOpen`) — an `ion-accordion-group`
  (`[multiple]`): one accordion per τάξη (τμήματα + chips inside), plus an
  "Ιδιαίτερα" accordion. Chips are native HTML5 `draggable`; `draggedPalette`
  branches `onDrop` between **placeEntry** (palette chip) and **moveEntry**
  (existing entry). Count badges per τάξη/τμήμα/panel.
- **Issues + Unscheduled** are now **collapsible cards BELOW the grid**
  (`issuesOpen` / `unscheduledOpen`, `.collapsible-card`/`.collapsible-header`).
  The issues card links each row to `/course/:id` to fix the data.
- **Entry block** gained a conflict outline + "Σύγκρουση" badge and a **remove
  (×)** button (kept visible, not hover-only, so it's tappable on touch).
- Data loads via **`refreshManual()`** (one `getManualData` call) instead of the
  old `getStoreSchedule`; `buildGrid` reads `this.timeSlots` + `availableDays`.
- **`RouterModule` added** to the standalone component's imports (needed for the
  issue `routerLink`s — see the standalone-routerlink gotcha).
- New i18n: `schedule.manual.*` (palette/issues/regen-confirm/conflict labels),
  `schedule.issues.{no_teacher,no_frequency,no_hours}`, and `messages.*`
  (`place_entry_success/conflict/fail`, `remove_entry_success/fail`,
  `move_entry_conflict`, `invalid_slot`).

### N.9 File map (additions)

- **Backend:** `models/schedule.js` (+ entry `manual`/`has_conflict`/
  `conflict_reasons`); `controllers/handlers/schedule.js` (+ `gridFromTimeSlots`,
  `buildOccupancy`, `evaluatePlacement`, `findClassTeacher`, `remainingHours`,
  `buildManualPalette`, `getManualData`, `placeEntry`, `removeEntry`; `moveEntry`
  rewritten to override; `gatherContext` returns `classes`); `controllers/
  schedule.js` (+ `getManualData`/`placeEntry`/`removeEntry`); `routes/
  schedule.js` (+ 3 routes).
- **Frontend:** `models/schedule.model.ts` (+ `PaletteSession/Course/Class/Grade`,
  `PalettePrivate`, `ScheduleIssue`, `ManualData`, `PlaceEntryPayload`, entry
  flags); `services/schedule.service.ts` (+ `getManualData`/`placeEntry`/
  `removeEntry`, `ManualDataResponse`, `warnings?`); `scheduling.component.{ts,
  html,scss}` (palette + collapsible cards + conflict/remove); `i18n/{el,en}.json`.

---

## 0. Changelog — 2026-08 rework

> **A "generate placed 0/35" debugging session on 2026-08-25/27 turned into a substantial
> rework of the algorithm, the move logic, the frontend grid and the failure reporting.**
> Sections 6–11 below still describe the *original* June design; where a section is now
> outdated it carries an **`UPDATED 2026-08`** note pointing back here. This §0 is the
> authoritative description of current behaviour. All changes are implemented in code
> (pending the user's API deploy + web build/OTA at the time of writing).

### 0.1 New diagnostic tooling

- **`scripts/diagnoseSchedule.js <store_id> [--full]`** (Frontistirio-API) — a **read-only
  dry-run**: runs the exact gather→build→place pipeline without persisting, and prints a
  summary (teachers/rooms with availability, slot window, placed/unscheduled, reason
  counts), per-teacher/room availability days, and per-lesson failure reasons. **The
  fastest way to debug a bad generation.** Backed by a new `exports.diagnose(store_id,
  periodId)` in the handler.
- `generateForStore` now `console.log`s a compact `buildDiagnostics()` summary on every run.

### 0.2 Root causes found (and the two that were data, not code)

1. **Missing availability data** — generation resolves the **store default period**; it
   silently places nothing if teacher/room `period_availability` isn't set *for that
   period*, even though lessons still get *built* from `period_courses`. `buildLessons`
   succeeding ≠ availability exists. (No code bug — the reasons now say this explicitly.)
2. **Placement rigidity** — see 0.3.
3. **Break/grid misalignment** — see 0.4.

### 0.3 `placeLessons` — frontier ➜ **scan-forward** (supersedes §6.5)

The per-teacher-day **frontier** was removed. The old code committed to ONE start slot
and abandoned the whole day if no room/student was free there (e.g. teacher free 10:30 but
rooms open 11:00 ⇒ *all* his lessons `no_room`). Now, for each candidate day it **scans
every start index** `for (fIdx = 0; fIdx + hours <= numSlots; fIdx++)` and places at the
**earliest** slot where the whole block is teaching-time + teacher-free + students-free +
a room fits. A `startedDays` set only *biases day preference* (prefer days the teacher
already teaches) — it no longer hard-gates placement. Net effect on the live store: 23→3
then →0 unscheduled. **"No gaps in a teacher's day" (old C9) is therefore now a soft
preference, not a structural guarantee.**

### 0.4 `buildSlotGrid` — uniform grid ➜ **break-aware** (supersedes §6.4)

The old grid was uniform (`slotStart(i) = minStart + i*D`); odd-length breaks (5/10/15′)
each **killed a whole D-slot AND knocked the phase out of alignment**, so continuous
multi-hour lessons couldn't fit between breaks. The grid is now built by **walking
minStart→maxEnd in D steps, jumping past any break and restarting the phase at the
break's end** (using the UNION of `breaksByDay` across days — a simplification, fine when
weekday breaks are identical). Slots are therefore **non-uniform** (variable startTimes,
e.g. …16:30–17:15, 17:20–18:05…) and follow the store's real bell schedule. `numSlots`
comes from the built array; `slotStart(i)`/`slotEnd(i)` read from arrays, not arithmetic.
The data-driven frontend renders whatever `time_slots` it's given.

### 0.5 Breaks **split** multi-hour lessons — they no longer block them (supersedes C8/C10)

Per the user: *a break must not prevent a 2h/3h lesson; it just interrupts it* (2h =
1h·break·1h; 3h = 2h·break·1h). The **contiguity check was removed** from `placeLessons`
(and from `explainLesson`). A block is now `hours` **consecutive teaching-slot indices** —
a break may sit between two of them. Consequently an entry's `endTime = slotEnd(startIdx +
hours - 1)` **carries the break gap**, so wall-clock span can exceed `hours × D`
(e.g. a 2h lesson stored as `16:30–18:05`). On the live store 23/38 entries legitimately
span a break. **Nothing is ever scheduled *during* a break** — the slots themselves are
break-free; only the gap *between* two of a lesson's slots may be a break.

### 0.6 Class scheduling rules (new hard/soft constraints — C11–C13)

Added to `placeLessons` (apply to `type:"class"` only; private lessons exempt):

| # | Rule | Type | Mechanism |
|---|---|---|---|
| C11 | A class must **not have the same course twice on one day** | hard | `classDayCourses["<classId>\|<day>"]` (Set of course_id); day skipped if hit → reason `class_course_repeat` |
| C12 | Ideally **not the same course on two consecutive days** | soft | `adjacencyPenalty(day)` sorts non-adjacent days first in `availDays` |
| C13 | A class gets **at most `MAX_CLASS_HOURS_PER_DAY = 6`** teaching hours/day | hard | `classDayHours["<classId>\|<day>"]`; day skipped if the block would exceed it → reason `class_daily_limit` |

`MAX_CLASS_HOURS_PER_DAY` is a **hardcoded constant** (not yet a `StoreSettings` field).
`moveEntry` does **not** enforce C11–C13 (manual moves stay an admin override).

### 0.7 `moveEntry` rewritten (supersedes §7 steps 3 & the table)

The old move used linear math (`endMin = startMin + hours*D`) and rejected any block
overlapping a break (`on_break`). It now **builds the same `buildSlotGrid(ctx)`**, finds
the target start slot by matching `time_slots[i].startTime` (by minutes), takes `hours`
consecutive slots (**break-spanning allowed**), validates each slot (teacher avail +
free, room avail + free, students free), and computes the **break-aware** `endTime =
slotEnd(startIdx+hours-1)`. `on_break` is gone (slots are break-free by construction); a
slot-not-found returns `move_entry_fail`, an overflow past day end returns
`teacher_unavailable`.

### 0.8 Failure reporting — specific reasons + per-lesson **`details[]`**

- `unscheduled[].reason` is no longer always `no_valid_slot`. It is the **binding
  constraint**, chosen by a severity ladder in `placeLessons`:
  `teacher_no_availability`(0) < `no_teaching_slot`(1) < `class_course_repeat`/`class_daily_limit`(2)
  < `students_conflict`(3) < `no_room`(4).
- **New:** each unscheduled lesson carries **`details[]`** — a per-day, structured
  explanation from `explainLesson()` that names the concrete blocker: *which student is
  busy in which lesson at what time*, or unavailable, or which class rule fired, or no
  room. Computed against turn-time occupancy, so every referenced lesson is one already on
  the timetable. Persisted in a new `unscheduledSchema.details` subdoc
  (`{ day, kind, student, course, startTime, endTime, hours, limit }`). Rendered on the
  frontend via `schedule.details.<kind>` ngx-translate templates (params) under the reason
  line; `component.detailText(d)` translates the day + interpolates.
- New i18n reason keys: `teacher_no_availability`, `no_teaching_slot`, `class_course_repeat`,
  `class_daily_limit`, `students_conflict`, `no_room` (the old move-path keys stay).

### 0.9 Frontend grid changes (supersedes parts of §8)

- **Break visualization** — `buildGrid` now composes a `rows[]` of `{kind:'slot'|'break'}`;
  a **break band** row (full-width, striped, "Διάλειμμα" + time) is inserted wherever two
  consecutive `time_slots` have a time gap. Break time is derived client-side from the
  gap (no backend change).
- **Break-spanning lessons render as multiple segments** — `buildCellMap` splits each
  entry into **contiguous-slot runs**; each run is a `{state:'start', rowspan, continuation}`
  cell, so a lesson crossing a break shows as two blocks with the break band between them
  (a "↓ συνέχεια" label + dashed top on continuation segments). Needs a `slotByIndex` map
  + `slotContiguous()`. `GridCell` gained `rowspan` and `continuation` (rowspan is the run
  length, no longer `duration_hours`).
- **View filters** — a filter-card bar (`.filter-line`/`.filter-select`) above the grid
  shows only a **τάξη** (`grade_name`), **τμήμα** (`class_id`, options cascade by grade),
  **καθηγητής** (`teacher_id`) or **μάθημα** (`course_id`). Client-side only: options are
  derived from the full entry set (stable grid), an `entryVisible()` predicate feeds
  `buildCellMap()` (split out of `buildGrid`). Needed `FormsModule` in the standalone
  component's imports for `[(ngModel)]`. New key `general.select.teacher`.
- **Drag onto an overlapping slot now works** — dragging a lesson excludes it from the
  cell map (`draggingId`, restored on drop/`dragend`) so its own covered slots become
  valid drop targets; you can shift a multi-hour lesson onto a time it currently covers.
  Drops now surface the backend's specific success/error toast instead of silently
  no-op-ing.
- **i18n cache-busting** — the `TranslateHttpLoader` suffix is now `` `.json?v=${APP_VERSION}` ``
  (`app.module.ts`), so a new release fetches fresh translations instead of a stale
  browser-cached copy (the loader had no cache-busting; a symptom was raw
  `schedule.details.*` keys showing after a translation change). Tied to the app version —
  bump the version for translation-only changes.
- New i18n: `schedule.break`, `schedule.continued`, `schedule.details.*`.

---

## 1. What it does (concept)

A store-user opens **`/scheduling`** and presses **Generate**. The backend collects
every constraint that has been configured across the app and produces a **weekly
timetable** — a set of placed lessons, each with a *day, start/end time, classroom,
teacher, course and target* (a class or a single private student). Lessons that cannot
be placed without breaking a constraint are returned separately as `unscheduled[]`
with a reason.

The timetable is rendered as a **grid** (rows = teaching-hour time slots, columns =
days, each day split into sub-columns = classrooms). The admin can **drag-and-drop**
any lesson to another cell; the move is re-validated server-side and rejected with a
specific reason if it would break a constraint.

**The unit of work ("a lesson to place") is derived from teacher assignments**, not
from courses or classes directly. See [§3](#3-inputs--the-data-the-algorithm-consumes).

Hard goals the generator honours:

- Place lessons only when the **teacher is available** and not already teaching.
- Ensure the chosen **classroom fits** the number of students and is available/free.
- Ensure **no student** (in a class or a private lesson) is placed during a time they
  are unavailable, or during another of their lessons.
- Never place a lesson **on a break**.
- **No gaps in a teacher's day** — on any day a teacher comes in, their teaching hours
  are contiguous (enforced structurally; see [§6.5](#65-placelessons--the-greedy-placement)).

---

## 2. End-to-end flow

```
Admin clicks "Generate" (/scheduling page)
        │  POST /schedule/generate
        ▼
controllers/schedule.js  → resolveContext() (user, store, default period)
        ▼
handlers/schedule.js → generateForStore(store_id, periodId, user_id)
        ├─ gatherContext()      // load + index all source data for the period
        ├─ buildSlotGrid()      // compute the teaching-hour slot grid
        ├─ buildLessons()       // expand teacher assignments into individual sessions
        ├─ placeLessons()       // greedy placement → { entries, unscheduled }
        └─ Schedule.findOneAndUpdate(upsert)   // one doc per (store, period)
        ▼
Response { success, schedule } → frontend sets schedule, builds the grid
        ▼
Admin drags a lesson → POST /schedule/move-entry → moveEntry() re-validates → saves
```

There is **one `Schedule` document per (store_id, period_id)**. Re-generating
overwrites `entries`, `unscheduled`, `time_slots`, `slot_minutes` (manual DnD edits are
lost on regenerate — see [§11](#11-known-limitations--failure-modes)).

---

## 3. Inputs — the data the algorithm consumes

All inputs are scoped to the store's **default `TeachingPeriod`** (resolved via
`teachinPeriodHandler.getStoreDefaultPeriod`). Every per-period array is read with
`findPeriodEntry(arr, periodId)` = `arr.find(e => String(e.period) === String(periodId))`.

### 3.1 Teacher assignments — the source of "lessons" (`models/teacher.js`)

```js
Teacher.period_courses[] = {
  period,                       // → TeachingPeriod
  courses: [{
    course_id,                  // → Course
    class_ids: [→ ClassModel],  // each id = one GROUP lesson obligation
    student_ids: [→ Student],   // each id = one PRIVATE lesson obligation
  }]
}
Teacher.period_availability[] = {
  period,
  workdays: [{ dayOfWeek (1-6), timeRanges: [{ startTime "HH:mm", endTime "HH:mm" }] }]
}
```

> **Critical:** if `Teacher.period_courses` is empty for the active period, the
> generator produces **no lessons**. Teacher↔course↔class/student assignments are
> created elsewhere (the `/courses/upsert-course-details` flow). This is the most
> common cause of an empty schedule.

### 3.2 Course frequency & duration (`models/course.js`)

```js
Course.period_schedule[] = {
  period,
  sessions_per_week,       // how many times/week  → how many sessions to place
  uniform_hours,           // (informational here)
  hours_per_session: [N],  // teaching hours of each session, e.g. [2, 1]
  weekly_hours,
}
Course.grade_id            // → Grade (used only for the cell label)
```

For a teaching obligation, `sessions_per_week` sessions are created; session `i` lasts
`hours_per_session[i]` teaching hours (min 1, defaults to 1 if missing). If
`sessions_per_week <= 0` or no `period_schedule` entry → that obligation is skipped
silently (no `unscheduled` record).

### 3.3 Classroom capacity & availability (`models/classroom.js`)

```js
ClassRoom.capacity                       // students it can hold
ClassRoom.period_availability[] = {
  period,
  workdays: [{ dayOfWeek (1-6), timeRanges: [{ startTime, endTime }] }]
}
```

### 3.4 Student class membership & unavailability (`models/students.js`)

```js
Student.period_class[] = { period, classes: → ClassModel }   // single class per period
Student.period_unavailability[] = {
  period,
  blockedDays: [{ dayOfWeek (1-7), timeRanges: [{ startTime, endTime }] }]
}
```

Group lesson students = all students whose `period_class.classes === class_id` for the
period (built into `studentsByClass`). The **group size** = that count → drives the
room-capacity check. A private lesson's student set = `[student_id]`.

### 3.5 Store-wide settings (`models/store-settings.js`)

```js
StoreSettings.course_hour_duration       // minutes per teaching hour (slot length D)
StoreSettings.break_times[] = {          // store-wide, NOT period-scoped
  dayOfWeek (1-7), timeRanges: [{ startTime, endTime }]
}
```

`course_hour_duration` defines the slot length `D` (default **60** if missing/≤0).
`break_times` are windows during which no lesson may be placed (any day 1-7).

---

## 4. Output data model — `Schedule`

`models/schedule.js` — one document per `{ store_id, period_id }`
(indexed `{ store_id: 1, period_id: 1 }`).

```js
Schedule {
  store_id, period_id,
  entries:      [ScheduleEntry],     // placed lessons
  unscheduled:  [UnscheduledLesson], // could-not-place, with reason
  slot_minutes: Number,              // = D used for this generation
  time_slots:   [{ index, startTime, endTime }],   // the row grid the entries sit on
  generatedAt, createdBy, updatedBy, isDeleted, deletedAt, timestamps
}

ScheduleEntry (subdoc, has _id) {
  course_id, teacher_id, classroom_id,
  type: "class" | "private",
  class_id,            // set when type==="class", else null
  student_id,          // set when type==="private", else null
  student_ids: [→ Student],   // everyone attending (capacity / conflict re-checks on move)
  student_count: Number,
  dayOfWeek: 1-7,
  startTime: "HH:mm", endTime: "HH:mm",
  duration_hours: Number,
  // Denormalized labels so the grid renders without extra lookups:
  course_name, teacher_name, classroom_name, target_name, grade_name,
  // Manual-scheduling flags (2026-09 — see the top changelog):
  manual,           // placed/moved by hand vs the auto generator
  has_conflict,     // sits on a hard-constraint violation (admin override)
  conflict_reasons, // [i18n reason keys] of the broken constraints
}

UnscheduledLesson (subdoc, no _id) {
  course_id, teacher_id, class_id, student_id, type, duration_hours,
  course_name, teacher_name, target_name,
  reason,  // i18n key under schedule.reasons.* — the BINDING constraint (see §0.8);
           // was "no_valid_slot" pre-2026-08
  details  // NEW (§0.8): per-day structured blockers, [{day,kind,student,course,
           //            startTime,endTime,hours,limit}]
}
```

> **Why denormalized labels?** The grid must render a cell without resolving 5 foreign
> keys per entry. `entry._id` is the handle drag-and-drop uses to move a lesson.

---

## 5. Backend API (routes + controller)

`routes/schedule.js` (mounted in `index.js` as `app.use("/schedule", scheduleRoutes)`),
all behind `authMiddleware` + `isStoreUser`:

> **UPDATED 2026-09 — three more store routes for manual scheduling:
> `GET /manual-data`, `POST /place-entry`, `POST /remove-entry` (see the top
> changelog §N.7). `move-entry` now returns 200 even on a conflict override.**

| Method | Path | Controller | Purpose |
|---|---|---|---|
| POST | `/schedule/generate` | `generateSchedule` | Run the algorithm, upsert + return the `Schedule` |
| GET  | `/schedule/get-store-schedule` | `getStoreSchedule` | Return the current `Schedule` (or `null`) |
| POST | `/schedule/move-entry` | `moveEntry` | Re-validate & persist a drag-and-drop move |

`controllers/schedule.js` is thin. `resolveContext(req)` extracts `user_id`,
`store_id` and the **default period id**; if there is no default period it throws
`default_period_not_found` (surfaced to the toast). `moveEntry` returns HTTP **409** on
a constraint violation (frontend reads `err.error.message` as the reason key).

---

## 6. The algorithm — `controllers/handlers/schedule.js`

This file is the heart of the system. It is a **deterministic greedy heuristic**, not
an optimal constraint solver. Sections below follow the code top to bottom.

### 6.1 Time helpers

All times are `"HH:mm"` strings; internally everything is **minutes since midnight**.

```js
toMin("16:30") === 990
toHHMM(990)    === "16:30"
overlaps(s1,e1,s2,e2) = s1 < e2 && s2 < e1   // HALF-OPEN: touching (e1===s2) is allowed
within(s,e, ranges)   = some range fully contains [s,e)
intersectsAny(s,e, ranges) = [s,e) overlaps any range
findPeriodEntry(arr, periodId) = arr.find(e => String(e.period)===String(periodId))
buildDayRanges(days, key) = { dayOfWeek: [{start,end}(min)] }  // from workdays/blockedDays
```

> **Touching is allowed** everywhere: a lesson ending 18:00 and another starting 18:00
> do **not** conflict. This is the contract for *all* overlap checks (teacher, room,
> student, break).

### 6.2 `gatherContext(store_id, periodId)`

Loads (with `.lean()`) and indexes everything for the store:

- `Teacher`, `Student`, `Course`, `ClassModel`, `ClassRoom`, `Grade`, `StoreSettings`
  (all `{ store_id, isDeleted:false }`; settings via `findOne`).
- `*ById` maps for O(1) lookup.
- `teacherAvail[teacherId] = { day: [ranges] }` — from the period's `workdays`.
- `roomAvail[roomId]       = { day: [ranges] }`.
- `studentBlocked[studentId] = { day: [ranges] }` — from `blockedDays`.
- `studentsByClass[classId] = [studentId]` — from each student's `period_class`.
- `breaksByDay = { day: [ranges] }` — from `StoreSettings.break_times`.
- `D = course_hour_duration > 0 ? course_hour_duration : 60`.

`gatherContext` is shared by **both** generation and move re-validation, so they apply
identical rules.

### 6.3 `buildLessons(ctx, periodId)` — expand assignments into sessions

Iterate every teacher's `period_courses[period].courses`. For each `{course_id,
class_ids, student_ids}`:

- Look up the `Course` and its `period_schedule`. If `sessions_per_week <= 0`, skip.
- For each **class_id** → one obligation with `studentIds = studentsByClass[classId]`,
  `studentCount = studentIds.length`, `type:"class"`.
- For each **student_id** → one obligation with `studentIds=[id]`, `studentCount:1`,
  `type:"private"`.
- Each obligation is expanded into `sessions_per_week` **session objects**, session `i`
  carrying `hours = max(1, hours_per_session[i] || 1)`.

Each lesson object also carries `course`, `gradeName`, `targetName` (class name or
`"First Last"`), `teacher`, `sessionIndex`.

**Ordering (hardest-first):**

```js
lessons.sort((a,b) => b.hours - a.hours || b.studentCount - a.studentCount);
```

Longer lessons and bigger groups are placed first (they are the hardest to fit). → see
[§10](#10-parametrization--extension-points) to change priority.

### 6.4 `buildSlotGrid(ctx)` — the teaching-hour grid

> **UPDATED 2026-08 — see [§0.4](#04-buildslotgrid--uniform-grid--break-aware-supersedes-64).**
> The grid is no longer the uniform `minStart + i*D` described below; it is **break-aware**
> (the phase restarts after each break, so slots follow the store's bell schedule and are
> non-uniform). The alignment caveat below is largely resolved by that change. The
> `minStart`/`maxEnd`/`D`/`numSlots`/fallback inputs are still accurate.

```
minStart = min over ALL teacher & room availability range starts
maxEnd   = max over ALL teacher & room availability range ends
(fallback if none: 08:00–22:00)

D        = ctx.D
numSlots = floor((maxEnd - minStart) / D)
slotStart(i) = minStart + i*D          slotEnd(i) = slotStart(i) + D
time_slots[i] = { index:i, startTime:toHHMM(slotStart(i)), endTime:toHHMM(slotEnd(i)) }
```

The grid is **global** (shared by all days/teachers/rooms). A lesson of `H` teaching
hours occupies `H` consecutive slot indices. `time_slots` is persisted so the frontend
renders exactly the rows the placement used.

> **Alignment caveat:** slots are anchored at `minStart` stepping by `D`. Availability
> that does not align to that grid loses the partial slot, because `within()` requires
> the whole slot `[slotStart, slotEnd)` to sit inside an availability range. Example:
> grid anchored at 08:00 with D=60 → slots …16:00,17:00…; a teacher available
> 16:10–20:00 cannot use the 16:00 slot (not fully inside) and effectively starts 17:00.
> Keep availability and `course_hour_duration` on consistent boundaries.

### 6.5 `placeLessons` — the greedy placement

> **UPDATED 2026-08 — see [§0.3](#03-placelessons--frontier--scan-forward-supersedes-65),
> [§0.5](#05-breaks-split-multi-hour-lessons--they-no-longer-block-them-supersedes-c8c10),
> [§0.6](#06-class-scheduling-rules-new-hardsoft-constraints--c11c13).** The **frontier is
> gone** (replaced by a scan of every start slot); the **contiguity/"cannot straddle a
> break" check is gone** (lessons split across breaks); and **class rules C11–C13** were
> added. The occupancy model, `isTeachingSlot`, `findRoom`, `studentsFree` and best-fit
> room selection below are still accurate.

**Occupancy** is tracked as interval lists keyed `"<id>|<day>"`:
`teacherBusy`, `roomBusy`, `studentBusy`. A conflict is `intersectsAny(s, e, list)`.

**Frontier** `frontier["<teacherId>|<day>"]` = the next slot index to fill for that
teacher on that day. This is the mechanism that enforces **no gaps**: lessons are only
ever placed *at the frontier*, i.e. immediately after the teacher's last lesson that
day (skipping breaks/non-availability), so a teacher's day is always a contiguous run.

Helpers:

- `isTeachingSlot(tid, day, i)` = slot `i` is `within` the teacher's availability for
  `day` **and** does **not** intersect a break that day.
- `firstTeachingSlot(tid, day)` = lowest slot index that is a teaching slot.
- `roomsByCapacity` = rooms sorted ascending by capacity (**best-fit**).
- `findRoom(day, startIdx, hours, count)` = first room with `capacity >= count` whose
  availability covers every slot of the block **and** is free (not in `roomBusy`).
- `studentsFree(studentIds, day, startIdx, hours)` = no student is `blocked` or `busy`
  on any slot of the block.

**Main loop** — for each lesson (in hardest-first order):

1. `availDays` = teacher's days that have any availability, **sorted to prefer days the
   teacher has already started** (`frontier` defined), then by day number. This packs
   existing days before opening new ones.
2. For each candidate day:
   - `fIdx = frontier[key] ?? firstTeachingSlot`. Advance `fIdx` past leading
     non-teaching slots (breaks / gaps between availability ranges).
   - If `fIdx + hours > numSlots` → skip day.
   - **Block check:** every slot `[fIdx … fIdx+hours-1]` must be a teaching slot and the
     teacher must be free. (A break or end-of-availability inside the block fails it —
     a lesson cannot straddle a break.)
   - **Students check:** `studentsFree(...)`.
   - **Room check:** `findRoom(...)`; if none → skip day.
   - **Place:** push intervals into `teacherBusy`/`roomBusy`/`studentBusy`, build the
     `ScheduleEntry` (with denormalized labels), advance `frontier[key]` past the block
     (and any trailing break/non-teaching slots).
   - `break` out of the day loop (placed).
3. If no day worked → push to `unscheduled` with `reason: "no_valid_slot"`.

> **Why this yields no teacher gaps but more `unscheduled`:** because a lesson is only
> tried at the frontier, if it conflicts there (e.g. a student is busy) it is **not**
> slipped into a later slot that day — doing so would leave an idle teaching hour. It is
> tried on another day instead, or left unscheduled. This is the deliberate trade-off
> that satisfies "no gaps in a teacher's day".

### 6.6 `generateForStore(store_id, periodId, user_id)`

Runs 6.2→6.5, then upserts:

```js
Schedule.findOneAndUpdate(
  { store_id, period_id },
  { $set: { entries, unscheduled, slot_minutes:D, time_slots, generatedAt, updatedBy },
    $setOnInsert: { createdBy } },
  { new: true, upsert: true }
)
```

### 6.7 `getStoreSchedule(store_id, periodId)`

`Schedule.findOne({ store_id, period_id })` — returns the doc or `null`.

---

## 7. Drag-and-drop move re-validation

> **UPDATED 2026-09 — `moveEntry` was rewritten again to share the
> `evaluatePlacement` validator with the new `placeEntry`, and is now an admin
> OVERRIDE: a move onto a conflicting cell is APPLIED and flagged
> (`has_conflict` + `conflict_reasons`, reasons returned as `warnings[]`),
> NOT rejected. Only a fatal target (`invalid_slot` / `room_not_found`) fails.
> It also maps onto the persisted (canonical) grid. The strict "first failure
> returns its reason key, HTTP 409" table below is therefore superseded — see
> the top changelog §N.3–N.4.**

> **UPDATED 2026-08 — see [§0.7](#07-moveentry-rewritten-supersedes-7-steps-3--the-table).**
> `moveEntry` now reuses `buildSlotGrid` + slot-index matching (break-aware end time,
> break-spanning allowed); step 3's linear `endMin = startMin + hours*D` and the `on_break`
> row are obsolete. The other checks in the table still hold.

`moveEntry(store_id, periodId, { entryId, dayOfWeek, startTime, classroom_id }, user_id)`:

1. Load the `Schedule`; find `entry = schedule.entries.id(entryId)` (→ `entry_not_found`).
2. `gatherContext` again (same rules as generation).
3. Compute the target block: `startMin = toMin(startTime)`,
   `endMin = startMin + entry.duration_hours * D` (duration is preserved; only
   day/start/room change).
4. Build occupancy from **all other entries on that day** (excluding the moved entry):
   `teacherIntervals`, `roomIntervals`, and per-student `studentIntervals`.
5. Checks (first failure returns its reason key, HTTP 409):

   | Check | Reason key |
   |---|---|
   | teacher availability covers block | `teacher_unavailable` |
   | teacher free (other entries) | `teacher_busy` |
   | not on a break | `on_break` |
   | room exists | `room_not_found` |
   | `room.capacity >= entry.student_count` | `no_room_capacity` |
   | room availability covers block | `room_unavailable` |
   | room free (other entries) | `room_busy` |
   | each student not unavailable | `student_unavailable` |
   | each student free (other entries) | `student_busy` |

6. On success: update `entry.dayOfWeek/startTime/endTime/classroom_id/classroom_name`,
   set `updatedBy`, `schedule.save()`, return `{ success:true, schedule }`.

> Move does **not** re-check the "no gaps" rule — manual moves are intentional admin
> overrides. It only enforces the hard conflict constraints.

---

## 8. Frontend (page, grid, DnD)

### 8.1 Model & service

- `models/schedule.model.ts` — `Schedule`, `ScheduleEntry`, `ScheduleTimeSlot`,
  `UnscheduledLesson` (mirror of the backend doc).
- `services/schedule.service.ts` — `generateSchedule()`, `getStoreSchedule()`,
  `moveEntry(payload)`; `MoveEntryPayload = { entryId, dayOfWeek, startTime, classroom_id? }`.

### 8.2 Page component — `scheduling/scheduling.component.ts` (standalone)

State: `schedule`, `loading`, `generating`, `days[]`, `classrooms[]`, `timeSlots[]`,
`draggedEntry`. Classrooms come from NgRx `selectClassRooms`.

- `ngOnInit` → subscribe classrooms, `loadSchedule()`.
- `generate()` → `POST /generate`, set schedule, `buildGrid()`, toast.
- **`buildGrid()`** computes the render structures:
  - `slotIndexByStart["HH:mm"] = index` (from `time_slots`).
  - `days` = distinct `entry.dayOfWeek` sorted, **fallback `[1..6]`** if empty.
  - `classrooms` = store rooms **∪** rooms referenced by entries (so a deleted room
    still gets a column, labelled from `entry.classroom_name`), sorted by name.
  - `cellMap["<slotIndex>|<day>|<roomId>"]`:
    - `{state:'start', entry}` at the entry's first slot,
    - `{state:'covered'}` for the remaining `duration_hours-1` slots,
    - everything else → `{state:'empty'}` via `getCell()`.
- `getCell(slotIndex, day, roomId)` → the cell (default empty).

### 8.3 Grid rendering — `scheduling.component.html`

A single `<table class="schedule-grid">` inside a horizontally-scrollable wrapper:

- **thead row 1:** a `rowspan=2` time header + one `<th colspan="#classrooms">` per day.
- **thead row 2:** the classroom names repeated under each day.
- **tbody:** one `<tr>` per `time_slot`; first cell = `startTime–endTime`; then for each
  day × classroom render `getCell(...)`:
  - `state==='covered'` → **render no `<td>`** (the `rowspan` from the start cell above
    covers it — this is how multi-hour lessons span rows).
  - `state==='start'` → `<td [attr.rowspan]="duration_hours">` containing a
    `draggable="true"` `.entry` block (target name, course · grade, teacher).
  - `state==='empty'` → a drop target.

### 8.4 Drag-and-drop (native HTML5 — **no `@angular/cdk`**, it is not installed)

- `.entry` is `draggable="true"`; `(dragstart)="onDragStart(entry)"` stores
  `draggedEntry`.
- Empty cells: `(dragover)="allowDrop($event)"` (calls `preventDefault` to permit drop)
  and `(drop)="onDrop(slot, day, roomId)"`. (Occupied cells are not drop targets in v1.)
- `onDrop` → no-op if dropped on the same place; else `moveEntry({entryId, dayOfWeek,
  startTime: slot.startTime, classroom_id})`. On success replace `schedule` and rebuild
  the grid; on 409 show a toast with the reason (`err.error.message`).

The unscheduled list renders below the grid in a warning `ion-card`, each item showing
target/course/teacher and `schedule.reasons.<reason>`.

### 8.5 Routing & menu

- Route `/scheduling` in `app-routing.module.ts` (`RoleGuard`, `roles: ['store-user']`).
- Menu item under **Management** (`app.component.html`), icon `calendar-number-outline`,
  label `menu.management.scheduling`.

### 8.6 i18n

- `schedule.*` — `title, generate, regenerate, generating, no_schedule, time, private,
  unscheduled_title, reasons.{no_valid_slot, teacher_unavailable, teacher_busy,
  on_break, room_not_found, no_room_capacity, room_unavailable, room_busy,
  student_unavailable, student_busy}` (the pre-existing `schedule.check_schedule` was
  kept).
- `menu.management.scheduling`.
- `messages.*` — toast keys: `generate_schedule_success/fail`, `get_schedule_fail`,
  `move_entry_success/fail`, `default_period_not_found`, `schedule_not_found`,
  `entry_not_found`, and the move reason keys (`teacher_busy`, `on_break`, …).
- `days.sunday` was added earlier (days 1-7 supported).

---

## 9. Constraints reference

| # | Constraint | Type | Enforced where |
|---|---|---|---|
| C1 | Teacher available for the whole block | hard | `isTeachingSlot` / move `within(teacherAvail)` |
| C2 | Teacher not double-booked | hard | `teacherBusy` / move `teacherIntervals` |
| C3 | Classroom available for the whole block | hard | `findRoom` `within(roomAvail)` / move |
| C4 | Classroom not double-booked | hard | `roomBusy` / move `roomIntervals` |
| C5 | Classroom capacity ≥ group size | hard | `findRoom` / move `no_room_capacity` |
| C6 | No student during their unavailability | hard | `studentsFree` / move `studentBlocked` |
| C7 | No student double-booked | hard | `studentBusy` / move `studentIntervals` |
| C8 | No lesson **during** a break | hard | slots are break-free by construction (§0.4) |
| C9 | ~~No gaps in a teacher's day~~ | ~~structural~~ → **soft** | now only a `startedDays` day-preference bias (§0.3) |
| C10 | ~~A lesson cannot straddle a break~~ | **removed** | breaks now **split** a lesson (§0.5) |
| C11 | Class: no same course twice on one day | hard | `classDayCourses` (§0.6) |
| C12 | Class: ideally not same course on consecutive days | soft | `adjacencyPenalty` (§0.6) |
| C13 | Class: ≤ 6 teaching hours/day | hard | `classDayHours` (§0.6) |

"Touching" boundaries never conflict (half-open intervals, [§6.1](#61-time-helpers)).
**C8–C13 updated 2026-08 — see [§0.5](#05-breaks-split-multi-hour-lessons--they-no-longer-block-them-supersedes-c8c10)/[§0.6](#06-class-scheduling-rules-new-hardsoft-constraints--c11c13).**

---

## 10. Parametrization & extension points

This is the section to read before changing scheduling behaviour. Each row says
**what to change** and **where**.

| Goal | Where / how |
|---|---|
| **Teaching-hour length** | `StoreSettings.course_hour_duration` (Settings → Course settings). It is `D` in `buildSlotGrid` / move. |
| **Daily window** (force fixed start/end instead of deriving from availability) | `buildSlotGrid`: replace the `minStart`/`maxEnd` scan with constants (or read new settings fields). |
| **Which days are scheduled** | Currently = days a teacher has availability (`availDays` in `placeLessons`). Add days by giving teachers availability on them; teacher `workdays.dayOfWeek` is capped 1-6 in the model — raise to 7 to allow Sundays. |
| **Lesson priority (which lessons win the good slots)** | `buildLessons` final `lessons.sort(...)`. Currently hours desc → group size desc. Add tie-breakers (e.g. teachers with least availability first). |
| **Spread vs. pack days** | `placeLessons` `availDays.sort(...)`. Currently prefers already-started days (packs). Reverse to spread lessons across more days. |
| ~~**Relax "no gaps"**~~ **DONE (§0.3)** | `placeLessons` already scans all start indices (frontier removed). |
| **Room selection policy** | `roomsByCapacity` (best-fit, smallest sufficient) + `findRoom`. Change the sort for worst-fit, or add a preferred/"sticky" room per course/class, or keep a class in one room all week. |
| **Capacity rule** | `findRoom`: `room.capacity >= count`. Add slack, or per-course room requirements (projector etc. — `ClassRoom.equipment`). |
| **Break source** | `breaksByDay` from `StoreSettings.break_times`. To make breaks period-scoped, move them into a period array and read via `findPeriodEntry`. |
| **Conflict semantics (touching)** | `overlaps()` is half-open. Switch to `<=`/`>=` to forbid back-to-back. |
| ~~**Allow a lesson to span a break**~~ **DONE (§0.5)** | contiguity check removed; a lesson now splits across breaks and the frontend renders it as segments. |
| **Make a hard constraint soft** (e.g. allow capacity overflow with a warning) | move the check out of the `continue`/reason path and record a soft flag on the entry instead. |
| **Persist manual edits across regenerate** | regenerate overwrites `entries`. Add a "locked" flag per entry and have `placeLessons` seed occupancy from locked entries before placing the rest. |
| **Better solver** | replace `placeLessons` with a backtracking/CSP or ILP solver; keep `gatherContext` + the `Schedule` output contract so the frontend is unaffected. |

---

## 11. Known limitations & failure modes

> **UPDATED 2026-08.** Several items below were fixed: "no gaps pushes to unscheduled"
> (frontier gone, §0.3), "multi-hour lesson cannot cross a break" (now splits, §0.5),
> "slot-grid alignment" (break-aware, §0.4), "move targets empty cells only" (drag frees
> the lesson's own slots, §0.9), and "`unscheduled.reason` is coarse" (specific reasons +
> `details[]`, §0.8). **Still true:** greedy/order-dependent (and `buildLessons` still
> orders hardest-first by hours→group size, **not** by teacher-availability scarcity, so a
> very constrained teacher can still be edged out once a class fills up — a
> most-constrained-first ordering would help); regenerate replaces the schedule (manual
> DnD lost); no real-time sync.

- **Greedy & order-dependent.** Result depends on lesson ordering; not guaranteed
  optimal or to place everything even if a perfect timetable exists.
- **"No gaps" pushes hard cases to `unscheduled`** rather than creating gaps
  ([§6.5](#65-placelessons--the-greedy-placement)). Expect `unscheduled[]` to be
  non-empty on tight inputs; the list tells the admin what to relax.
- **A multi-hour lesson cannot cross a break** or a between-ranges availability gap.
- **Slot-grid alignment** (off-grid availability loses partial slots — [§6.4](#64-buildslotgrid--the-teaching-hour-grid)).
- **Regenerate replaces the schedule** — manual DnD adjustments are lost. (See the
  "locked entries" idea in [§10](#10-parametrization--extension-points).)
- **No real-time sync.** Unlike most entities, `Schedule` is **not** socket-wired; the
  frontend updates only from the `generate`/`move` responses. A second admin won't see
  changes live.
- **Move targets empty cells only** in v1 (no swap). Backend would accept any valid
  target; the UI just doesn't offer occupied cells as drop zones.
- **`unscheduled.reason` is coarse** — always `no_valid_slot` from the generator (the
  per-constraint reasons exist only on the move path).

---

## 12. Data prerequisites & troubleshooting

For a non-empty, correct schedule the store needs, **for the default period**:

1. A **default `TeachingPeriod`** (else `default_period_not_found`).
2. **Teacher `period_courses`** populated (teacher↔course↔class/student assignments).
   *Empty assignments → empty schedule.*
3. **Course `period_schedule`** with `sessions_per_week > 0` and `hours_per_session`.
4. **Teacher `period_availability`** (workdays + time ranges).
5. **Classroom `period_availability`** and sensible `capacity`.
6. **Students assigned to classes** (`period_class`) so group sizes/capacity are right;
   student `period_unavailability` as needed.
7. **`StoreSettings.course_hour_duration`** set (else defaults to 60); optional
   `break_times`.

Troubleshooting symptoms:

| Symptom | Likely cause |
|---|---|
| Schedule empty after generate | No teacher `period_courses`, or all courses have `sessions_per_week=0` |
| Everything `unscheduled` | No teacher/room availability for the period, or no rooms with enough capacity |
| Lessons clump too early in the day | Expected — frontier packs from availability start (compactness) |
| A teacher's available window seems shifted later | Off-grid availability vs. `course_hour_duration` ([§6.4](#64-buildslotgrid--the-teaching-hour-grid)) |
| Move rejected | Toast shows the exact reason key (`teacher_busy`, `no_room_capacity`, …) |

---

## 13. File map

> **UPDATED 2026-09 — manual-scheduling additions (new handler functions,
> endpoints, entry flags, palette model + component) are listed in the top
> changelog §N.9.**

**Backend (`Frontistirio-API`)**
- `models/schedule.js` — `Schedule` + `ScheduleEntry` + `UnscheduledLesson` (+ `details[]` subdoc, §0.8).
- `controllers/handlers/schedule.js` — `gatherContext`, `buildLessons`, `buildSlotGrid`
  (break-aware), `placeLessons` (scan-forward + class rules + `explainLesson`),
  `buildDiagnostics`, `generateForStore`, `getStoreSchedule`, `diagnose`, `moveEntry`.
- `scripts/diagnoseSchedule.js` — read-only diagnostic CLI (§0.1).
- `controllers/schedule.js` — `resolveContext`, `generateSchedule`, `getStoreSchedule`,
  `moveEntry`.
- `routes/schedule.js` — mounted at `/schedule` in `index.js`.

**Frontend (`Frontistirio`)**
- `src/app/models/schedule.model.ts` (+ `UnscheduledDetail`)
- `src/app/services/schedule.service.ts`
- `src/app/scheduling/scheduling.component.{ts,html,scss}` — grid + break bands +
  multi-segment split + view filters + drag-frees-own-slots (§0.9)
- `src/app/app.module.ts` — `HttpLoaderFactory` now cache-busts i18n with `?v=${APP_VERSION}` (§0.9)
- route in `src/app/app-routing.module.ts`, menu in `src/app/app.component.html`
- i18n in `src/assets/i18n/{el,en}.json` (`schedule.*` incl. `break`, `continued`,
  `reasons.*`, `details.*`; `general.select.teacher`; `menu.management.scheduling`; `messages.*`)

**Inputs (read-only, documented in [§3](#3-inputs--the-data-the-algorithm-consumes))**
- `models/teacher.js` (`period_courses`, `period_availability`)
- `models/course.js` (`period_schedule`, `grade_id`)
- `models/classroom.js` (`capacity`, `period_availability`)
- `models/students.js` (`period_class`, `period_unavailability`)
- `models/store-settings.js` (`course_hour_duration`, `break_times`)
