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

- [Changelog — 2026-11: reproducible runs, candidates, per-schedule hours, room equipment](#changelog--2026-11-reproducible-runs-candidates-per-schedule-hours-room-equipment)
- [Changelog — 2026-10 (b): the generator becomes a search (API)](#changelog--2026-10-b-the-generator-becomes-a-search-api)
- [Changelog — 2026-10: the `.sd-*` editor, overview, live check, quality](#changelog--2026-10-the-sd--editor-overview-live-check-quality)
- [Changelog — 2026-09 (b): per-role timetables, co-teaching, multiple schedules](#changelog--2026-09-b-per-role-timetables-co-teaching-multiple-schedules)
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

## Changelog — 2026-11: reproducible runs, candidates, per-schedule hours, room equipment

> Recorded **2026-11-07** against API `95164a4` + frontend `b6ca8ff` (both on `dev`, **not
> deployed**). Where this changes §3, §4, §6, §9 or §10, **this block wins.** Bug ids refer to
> `Frontistirio-bugs-brain-repo/scheduling-bugs.md` §10.

### The thing to know first: the search is now reproducible *on request*

`solveTimetable`'s own comment said "the same input gives the same timetable". It was not true. The
construction phase is seeded (`mulberry32(opts.seed)`), but the simulated-annealing phase reads
`Date.now()` for **both** its stopping point and its cooling schedule:

```js
const progress = Math.max(it / maxIter, (Date.now() - t0) / timeLimitMs);
```

so a busier machine does fewer iterations, cools on a different curve, and lands somewhere else.

That is harmless for "build me the week" and fatal for anything that shows a result and then
produces it again — which is exactly what the new generation preview does. A new option:

```js
solveTimetable(ctx, lessons, grid, { seed, deterministic: true })
// progress = it / maxIter — no clock in the loop
```

With `deterministic: true` the run depends **only** on `(seed, maxIterations)`. The iteration budget
is derived inside `planGeneration` rather than passed in:

```js
const deterministicIterationCap = (lessonCount) => Math.min(120000, 5000 + 1500 * lessonCount);
```

so the preview and the later "apply this one" compute the *same* cap from the same data without any
caller having to remember it. `stats` now carries `deterministic` and `seed`, and the generation
report passes both to the client.

The time-boxed path is **unchanged** for an ordinary generate: it gets more search done per second,
and nobody is promising to reproduce it.

### `GET /schedule/preview-generate` — four candidates, nothing written

| | |
|---|---|
| Query | `schedule_id`, `mode` (`fresh` \| `fill`) |
| Seeds | `[20261025, 7, 991, 20260612]` — fixed, so two previews of the same data agree |
| Returns | `score_before`, `penalties_before`, `lessons_now`, `locked_now`, `lessons_to_place`, and `candidates[]` sorted best-first (ties: fewer unplaced, then lower seed, so the ORDER is stable too) |
| Each candidate | `seed`, `kept`, `placed`, `unscheduled`, `grid`, `score`, `penalties`, `class_idle_hours`, `teacher_idle_hours`, `room_changes`, `unplaced_lessons[]` (capped at 20), `search{}` |
| Writes | none — it is a `GET` for that reason |

`score_before` is the same in every candidate (it scores what is on the timetable now), so a
candidate below it is a run worth **not** applying. That is the point of a preview, and without it a
score of 78 means nothing.

Applying one is `POST /schedule/generate` with `seed`. The controller turns a finite `seed` into
`{ seed, deterministic: true }`; without a seed nothing changes.

### `POST /schedule/rebuild-grid` — SCH-06's missing action

The saved `time_slots` **are** the grid, deliberately: otherwise a settings change would slide
finished lessons off their cells. `staleGridIssues` (2026-11-06) told the admin the grid had drifted;
this is the action.

- Each entry is kept **iff** the fresh grid has a slot run starting and ending exactly where it does
  (`sitsOnGrid`). Nothing is moved to a nearby slot: a lesson placed at 17:20 does not quietly become
  17:00, because a timetable that moved by itself is worse than one that says what it dropped.
- A dropped entry is simply removed from `entries`. It is **not** written to `unscheduled` — that
  list is the generator's "could not place, here is why". The palette is computed by subtracting
  placed entries from the period's obligations, so a dropped lesson reappears there on its own.
- `confirm: false` (the default) is a **dry run**: nothing is written, the report says what would
  happen. Widening the teaching window restarts the slot phase at the new earliest hour, which can
  push *every* lesson off the grid — the admin has to see that number first.
- A **locked** entry that would not survive refuses the whole action (`locked_entries_off_grid`,
  with the entries named). A lock promises the lesson stays put, including through a "from scratch"
  generate; breaking it quietly would make locking worthless.
- `no_grid_available` when the period has no availability at all: refusing beats replacing a working
  grid with an empty one.

### `Schedule.working_hours` + `withScheduleWindow` — ALG-06

Availability lives on the teachers and the rooms, **per teaching period**. Right for the normal week,
wrong for a holiday one: the generator read the same input whichever timetable it filled, so with a
deterministic search a Christmas schedule came out as a byte-identical copy of the normal one. "Copy
from another schedule" was the only way to start one.

```js
// Schedule
working_hours: [{ dayOfWeek, timeRanges: [{ startTime, endTime }] }]   // [] = the period's hours
```

```js
withScheduleWindow(ctx, schedule)
//  → teacherAvail / roomAvail intersected with the window
//  → a day the window does not name becomes unavailable to EVERYONE
//  → scheduleWindow set, so a caller can tell a clipped ctx from a plain one
//  → an empty window returns ctx untouched
```

Because `buildSlotGrid` derives the grid from exactly those two maps, the week's hours, its slot
phase **and** its days all follow from one edit. It is applied at every site that resolves a specific
schedule: `generateForSchedule`, `previewGeneration`, `rebuildGrid`, `getManualData`,
`getFullScheduleData`, `placeEntry`, `moveEntry`, `removeEntry` and `copyTimetable`.

`copyTimetable` is where it earns its keep: copying the normal week into a narrower one judges the
flags under the **target's** hours, so the lessons that no longer fit light up as problems to move.

**It narrows, it does not override.** No teacher is ever placed outside their own availability — the
window can only remove hours.

`normalizeWorkingHours` drops an unreadable time, an inverted range and a day outside 1–7. A
malformed window would not narrow anything, it would empty the week with nothing on screen to
explain it. `[]` on update **clears** the window (hence the check is for an array, not a non-empty
one); the field being absent means "leave it alone".

### `Course.required_equipment` — ALG-07

`ClassRoom.equipment` (`computer`, `projector`, `whiteboard`) has existed for a long time and
**nothing ever read it**, because no course said what it needed. The other half:

```js
// Course
required_equipment: { computer: Boolean, projector: Boolean, whiteboard: Boolean }  // all false = any room
```

Three shared helpers (`equipmentOf`, `equipmentNeeded`, `roomMeets`) are exported from
`scheduleSolver` so the generator's rule and the live check are literally the same lines:

- `it.roomList` filters on capacity **and** equipment. The two are tracked separately so the
  explanation can say which emptied the list: `no_room` (the φροντιστήριο has no big enough room) and
  `no_room_equipment` (rooms fit, none has the kit) are different problems fixed in different places.
  The latter short-circuits `explain` — no day will ever work, so it is said once.
- `scheduleValidate` adds `room_missing_equipment`. A hand placement into such a room **succeeds**
  and is flagged, like every other admin override. The flag is derived, so dropping the requirement
  clears it on the next read.
- A room the store has deleted is not judged: the same rule capacity and availability follow, so a
  stale id never invents a conflict.
- The requirement is normalised server-side to the three known booleans. A requirement no room can
  declare is one no room can satisfy, and it would make every lesson of that course unplaceable with
  no way to fix it.

### `POST /schedule/export-full-schedule` — UX-11

`helpers/createFullSchedulePdf.js`. One **landscape** page per teaching day; rows are the grid's
teaching hours, columns are αίθουσες \| τμήματα \| καθηγητές (`group_by`, default `room`).

- Columns are **paged, not shrunk**: 14 rooms get two sheets rather than 14 unreadable slivers.
  A column page that is empty for a day is skipped.
- A lesson with no value for the chosen dimension gets a `—` column instead of being dropped: a
  printed timetable missing a lesson that is actually happening is the worst outcome.
- A co-taught lesson prints under **every** τμήμα attending it.
- Two lessons in one cell print **both**, marked: hiding a double booking on the noticeboard is how
  two τμήματα end up at the same door.
- `columnNames` lets the caller name rooms/classes the store has but no lesson uses — an empty room
  column is information.

### Frontend

| | |
|---|---|
| `schedule.service.ts` | `previewGeneration`, `rebuildGrid(id, confirm)`, `exportFullSchedule(id, groupBy)`, `generateSchedule(id, mode, seed?)`. The seed is omitted from the body when absent, so an ordinary run is not silently made deterministic. |
| Generate menu | third option «Προεπισκόπηση & εναλλακτικές» → radio alert, one line per candidate. |
| Problems panel | «Η τελευταία αυτόματη δημιουργία» (score before → after, unplaced, search work, chosen variant), and `grid`-type issues in their own section with the rebuild action. |
| Page menu | «Εκτύπωση συνολικού προγράμματος», «Ξαναχτίσε το πλέγμα ωρών». |
| History tab (UX-10) | every action of the sitting with its time; undo on the newest un-undone line only (the inverses are only sound LIFO), and a bulk action seals what is under it. |
| Room pill (UX-09) | narrows the board to one αίθουσα; kept per device, **not** suspended while a lesson is held. |
| Meta modal | «Ώρες αυτού του προγράμματος»: a switch plus one row per working day. |
| Course form | «Εξοπλισμός που χρειάζεται», the same three chips the room form declares. |
| `schedule-grid.util.ts` | `roomMeetsCourse` + `roomEquipment`/`courseEquipment` in the availability index; `buildAvailabilityIndex` takes a 5th `courses` argument (defaulted, so existing callers are unaffected). |

### Gotchas worth keeping

- **A determinism test can pass for the wrong reason.** The first one used
  `maxIterations: 40000, timeLimitMs: 2000`; 40k iterations finish in ~50ms, so the clock never bit
  and both code paths behaved identically — a mutation with the clock put back **passed**. It needs
  `maxIterations: 200000, timeLimitMs: 5`, where a clock-reading run stops after a few thousand and
  at a count that varies.
- **`signals:audit` is right about getters.** Three times in this pass a getter that built a fresh
  array (`equipmentChips`, `undoableId`, `dayOptionsFor`/`hoursError`) ran per change-detection cycle
  and made `*ngFor` rebuild. All became fields refreshed on change.
- The `grid_stale` issue row was rendered by the shared `<a [routerLink]="['/course', course_id]">`,
  and it has no course — a dead link since 2026-11-06. `grid`-type issues now have their own section.
- The frontend's cross-repo contract check used to read the API's **default** branch, so any commit
  calling a route still only on the API's `dev` failed CI — and `release-git.js` refuses to release
  without green CI, which is a deadlock. It now reads the **same branch name**, falling back to the
  default.

---

## Changelog — 2026-10 (b): the generator becomes a search (API)

> Recorded **2026-10-25** against API `7ec738b` + working tree (**uncommitted, not deployed** at the
> time of writing), on top of Q.8. Where it changes §6, §9–§11 or any changelog below, **this block
> wins**: placement is no longer "one greedy pass, first free slot". Why it was needed: bugs brain
> repo `scheduling-bugs.md` §2 (ALG-01…09). What it achieved, with numbers: same file, §7.

### R.1 Where the code is

- **`controllers/handlers/scheduleSolver.js`** (new, pure, no DB): `solveTimetable(ctx, lessons, grid,
  opts)`, `remainingLessons`, `scheduleQuality` (server port of the editor's score),
  `clashingEntries`, `workingDays`, `WEIGHTS`. It cannot require `schedule.js` (which requires it), so
  it carries its own copies of the small time/name helpers.
- **`schedule.js`:** `placeLessons(ctx, lessons, grid, opts)` is now a thin wrapper. It calls the solver,
  then appends `ctx._coteachIssues` / `ctx._hourIssues` as before. New: `planGeneration(ctx, periodId,
  current, mode, opts)` (pure: what to keep, which grid, run, score before/after),
  `generateForSchedule(…, { mode })`, `setEntryLocked`, `diagnose(store, period, { scheduleId })`.
  `buildLessons` is unchanged. Its sort (hours ↓, students ↓) is now only the tie-break order.

### R.2 Occupancy model

Slot-indexed typed arrays instead of interval lists. Teachers (`t:`), **τμήματα (`c:`)** and students
(`s:`) share one entity space: `busy[(ent·9 + day)·N + slot]` holds a count, so a lesson can be lifted
off again. Rooms are a separate array. A τμήμα being an entity is how Q.8's SCH-02 rule lives on: one
`peopleFree` check covers teacher, τμήμα and students. Per lesson, the **static candidates** are
precomputed: (day, start) where every slot is teaching time for the teacher (availability minus
breaks) and no student is blocked. Dynamic checks: people free; room open, free and big enough; C11
(`cc[day]`, sessions of that course per τμήμα-day); C13 (`classHours`). **Fixed entries** are mapped
onto the grid by time overlap, so an entry off the grid, or one with a `"NaN:NaN"` end
(`entryInterval` repairs it as Q.8 does), still blocks what it covers.

### R.3 Phase 1: construction, most-constrained first (ALG-01)

DSatur: at every step, the lesson with the **fewest feasible (day, start, room) triples** goes next
(ties: `buildLessons` order). Counts are kept per lesson per day. After a placement only **that
day** is recounted, and only for lessons that share a person (teacher/τμήμα/student) or could use that
room. A lesson whose count hits 0 is set aside. Its value is the feasible triple with the **lowest
objective** (R.6), evaluated by apply → score → unapply. Strict `<`, so ties keep the earliest day,
slot and smallest room. That is why a lone lesson still lands at the first hour.

### R.4 Phase 2: repair (one-lesson ejection)

For each lesson set aside: first a direct try (something may have opened), then every candidate
(day, start, room) where **exactly one movable** lesson is in the way, through a shared person or
by holding that room. Lift it, place the new one, re-place the lifted one at its best feasible value;
undo if that fails. At most 400 tries per lesson. Fixed entries are never lifted. Runs after
construction and again after the local search.

### R.5 Phase 3: simulated annealing (ALG-08, partly)

Moves: **relocate** (random static candidate, keep the room if free; 55%), **swap** the times of two
lessons (35%; three times in four the partner shares a τμήμα or teacher, where swaps close gaps),
**change room** (10%).
Each move is applied only if every hard constraint holds; otherwise it is undone and skipped. Accept
if Δ ≤ 0 or with e^(−Δ/T). T0 = the mean worsening of 60 random moves, falling geometrically to
T0/1000 by progress = max(iterations/cap, time/limit). Limits: `timeLimitMs` 2000,
`maxIterations` 5000 + 2000·lessons. The **best** state is kept (strict improvement) and restored.
**Seeded PRNG** (`mulberry32`): the same input gives the same timetable, unless the time limit
(not the iteration cap) ends the run.

### R.6 The objective is the editor's score (ALG-02/03/04/07)

The solver minimises the editor's `scheduleQuality` penalties (Q.3), with the same weights and the
same measures (idle **slot indices** per τμήμα-day / teacher-day, cv of hours over the working days,
(τμήμα, course) rows with consecutive days, room changes per τμήμα-day). It is **uncapped**, so there is
still a gradient past a cap, and normalised by the live τμήμα-day / teacher-day counts as the score is.
Totals are maintained incrementally: `begin(it, day)` subtracts the contributions of what a lesson
touches, `end` adds back the recomputed ones. Two tie-breaks are **not** in the score:
`teacher_days` 0.05 per teacher-day (the old generator packed teachers onto few days; now balance wins
unless near a tie) and `room_fit` 0.01 × spare seats / largest room (best-fit keeps big rooms free).
`scheduleQuality` in the solver is a **port** (caps and rounding included), used for the report and the
tests. Its conflicts are only the pairwise ones (`clashingEntries`: teacher/room/τμήμα/student overlap,
C11, C13). Availability and capacity are not re-checked there. Conflicts cannot occur in generated
output.

### R.7 Fill, fresh and locked (ALG-05)

- **`entry.locked`** (new field, default false). `POST /schedule/lock-entry` `{ schedule_id, entryId,
  locked: boolean }` → `setEntryLocked` (store-user, `// NEW ROUTE`). UI since Q.10 (inspector toggle).
- **`mode: "fresh"`** (default): keep the **locked** entries, place everything else around them on a
  rebuilt grid. If a locked entry would not sit on the rebuilt grid (`sitsOnGrid`: start on a slot,
  end on its last slot's end), the schedule's **own** grid is kept instead, so the lock still holds
  and the lesson stays drawable. Unlocked entries, hand-placed ones included, are replaced.
- **`mode: "fill"`**: keep **every** entry and the schedule's own grid; place only
  `remainingLessons(buildLessons(…), kept)`. That is the same multiset subtraction per obligation
  (`co:` group|course, `cl:` class|course, `pr:` student|course) and length as the palette's
  `remainingHours`, so fill places exactly the chips the editor shows.
- Kept entries are written back as plain objects with their `_id`, flags and labels. `slot_minutes`
  and `time_slots` are only rewritten when the grid was rebuilt.
- The controller refuses an unknown `mode` (400 `invalid_mode`), so a typo cannot wipe a hand-made
  timetable. The response adds **`report`** (from `schedule.$locals.report`): `mode, kept, placed,
  unscheduled, grid: "rebuilt"|"kept", score_before, score_after, penalties_after, search: {
  repaired, iterations, improvements, ms }`.

### R.8 Unscheduled reasons

Same ladder as §0.8 plus Q.8's `class_busy` (severity 3, preferred over `students_conflict` when the
τμήμα itself is in the way), and the same `details[]` kinds. Both are now computed against the
**final** timetable, not at the moment the lesson failed, so every lesson a detail names is really
on the saved timetable.

### R.9 The `legacy` strategy (the baseline)

`opts.strategy: "legacy"` replays the old pass inside the solver: static order, and the value is the
lexicographic minimum of (same course on an adjacent day, teacher not yet teaching that day, day,
start, room rank). That is exactly the old day sort plus earliest slot plus best-fit room. No repair, no
search. The class rule of Q.8 applies to it too. It exists for the tests and for
`scripts/diagnoseSchedule.js`, which now prints **legacy / construct / search** side by side (placed,
unscheduled, score, penalties, idle hours, ms), the saved timetable's score (`--schedule <id>` picks one
other than the default), and a **fill** dry run on it.

### R.10 Verification (2026-10-25)

- `controllers/handlers/__tests__/scheduleSolver.test.js`, 12 specs: the 2-afternoon teacher (legacy
  leaves both lessons out as `class_busy`, search places them), repair (the ιδιαίτερο behind a τμήμα),
  τμήμα gaps (legacy > 0, search 0, score up), front-loading (legacy Mon 8h / Thu–Fri 0, search within
  2h), fill (2h placed by hand subtracted, 1h not, C11 respected around it), fresh keeps locked +
  fill keeps all on the own grid, locked off the rebuilt grid keeps the own grid, SA with a locked
  entry and a blocked student (no hard violation, locked untouched), determinism, the score
  arithmetic against a hand-computed 50, 200 lessons in < 1 s. Q.8's 6 specs pass unchanged on the
  new solver. `npm test` **331/331**; `node --check` on every touched file.
- `diagnoseSchedule.js` on the dev DB (read-only), store Κιζιρίδη, 30 lessons: legacy 22 placed,
  score 58, 10 idle τμήμα-hours → construct 24, 90, 0 → search 24, 91, 0 (68 ms). Fill on the saved
  4-entry timetable: keeps 4, adds 20, score 90 → 91. The 6 left are `teacher_no_availability` (data).
- Synthetic: 400 lessons, construct ~100 ms, search to the iteration cap in ~1 s, score 97 → 100.

### R.11 Gotchas and open points

- **Balance counts days nothing can use.** The score's (and so the solver's) working days are the days
  some teacher **or** some room works. On dev, Saturday has a teacher but no room, so its load is
  always 0 and «balance» never clears. Suggest: teacher **and** room, in the frontend `days` and in
  `workingDays` together, or the two disagree.
- **Teacher days** are not in the score (see R.6). A product decision decides whether they enter both.
- No late-hour penalty, no room equipment, one solution per run (no compare).
- The generate request now runs up to ~2 s longer on a big store (still far under API Gateway's 29 s).
- UI still to do: «Συμπλήρωση υπολοίπων» → `generate` with `mode: "fill"`, a lock toggle per lesson,
  `ScheduleEntry.locked` in `schedule.model.ts`, showing `report`, and a fresh-regenerate confirm
  that says locked lessons stay. i18n for `invalid_mode`, `lock_entry_success`,
  `unlock_entry_success`, `lock_entry_fail`.

---

## Changelog — 2026-10: the `.sd-*` editor, overview, live check, quality

> Recorded **2026-10-25**, against frontend `0f1fbfa` + working tree (uncommitted at the time of
> writing) and API `7ec738b`. **Frontend only — no API change in either pass.** Where it changes
> §8 or the changelogs below, **this block wins**. The weakness analysis that motivated the second
> pass (algorithm + UI, with the open bugs SCH-01…08) lives in the **bugs brain repo →
> `scheduling-bugs.md`**.

### Q.1 2026-10-01 — `/schedules/:id` rebuilt as the `.sd-*` editor

`SchedulingComponent` was rebuilt for usability: header with name, «Κανονικό», «Σε ισχύ σήμερα»
(computed from the `listSchedules` siblings) and the meta modal (`schedules/schedule-meta-modal/`,
also used by the `/schedules` list for create + edit). A Εβδομάδα|Ημέρα toolbar, 4 **faceted**
filters that **dim** (do not hide) lessons, «Χρώμα ανά» τάξη/μάθημα/καθηγητή (golden-angle hues,
`--hue` + `hsl()`, no `color-mix()` because old Android WebViews lack it), a docked side panel
(drawer below 1200px) with «Τοποθέτηση» (palette) and «Προβλήματα». Place/move by drag **or**
tap-chip-then-tap-cell, an empty-cell "what fits here" popover, a lesson inspector popover, and
an «Αναίρεση» toast after every write. Prefs in `localStorage['scheduleEditor:prefs']`.

Non-obvious decisions (still true):
- `schedule-grid.util.ts` **mirrors the API's `evaluatePlacement`** client-side, so the grid shows
  where a lesson fits before the drop. An unknown teacher/room counts as "not checked", never as
  "unavailable". Room overlap and grid overflow are **blocked** in the UI, because two lessons
  cannot be drawn in one room cell (the server would only flag `room_busy`).
- `schedule.unscheduled` is a generation-time snapshot the server never updates →
  `matchUnscheduled` attaches reasons to the remaining palette chips as a multiset.
- The dragged lesson's DOM node must survive the re-render, or `dragend` is lost: the same element
  stays as a faint ghost (cells tracked by key). Firefox needs `dataTransfer.setData`.
- Styles are **global** (`src/theme/schedule-editor.scss`, imported in `global.scss`): the page
  broke the 32 kB `anyComponentStyle` budget, and Ionic moves inline popovers into `ion-app`.

### Q.2 2026-10-25 — live validation (`validateSchedule`)

The server stamps `has_conflict`/`conflict_reasons` on the ONE entry placed or moved, at that
moment: the lesson it landed on stays clean (asymmetric), the flag outlives the conflict once
either side moves (stale), and data changes underneath (availability, capacity, rosters) never
surface (bugs repo SCH-03). The editor now **recomputes every conflict on every rebuild**,
client-side, in `validateSchedule(entries, grid, avail)`:
- pairwise per day, sorted by start: `teacher_busy`, `room_busy`, **`class_busy`** (shared τμήμα:
  catches an **empty τμήμα** double-booked, which the generator misses, see SCH-02) or else
  `student_busy`. **Both** entries get the reason;
- per entry against today's data: `no_room_capacity`, `teacher_unavailable`, `room_unavailable`,
  `student_unavailable` (slot by slot, so a break inside a lesson needs no availability);
- the τμήμα rules hand placement skips (SCH-04): **`class_course_twice`** and **`class_over_limit`**
  (`MAX_CLASS_HOURS_PER_DAY = 6`, mirrored from the generator).
- Students are read **live** (`liveStudentIds`): a class lesson uses today's roster
  (`rosterByClass`), a co-taught one adds its stored set (explicit ιδιαίτερα members are on no
  roster), a private one its `student_id`. Until the period is known (`avail.loaded`), the stamped
  server reasons are unioned in.
- The result drives `EntryVM.hasConflict/reasons`, the «συγκρούσεις» figure and the panel. The
  stored flag is no longer what the UI shows. The API is unchanged (follow-up: the same check
  server-side in `getManualData`).

### Q.3 2026-10-25 — quality score (`scheduleQuality`)

The first measure of whether a timetable is *good*, not only conflict-free. It gives 0–100 = 100
minus capped penalties, normalised per τμήμα-day / teacher-day so a big store is not marked down
for being big:

| Penalty | Max | Measure |
|---|---|---|
| conflicts | 40 | 8 × entries with ≥1 live reason |
| class_gaps | 25 | 25 × idle teaching hours per τμήμα-day (`idleSlots` over slot indices, so breaks never count) |
| teacher_gaps | 15 | 10 × idle hours per teacher-day |
| balance | 10 | 20 × coefficient of variation of hours over the working days |
| spread | 5 | 1 per (τμήμα, course) with sessions on consecutive days (the generator's soft C12) |
| rooms | 5 | 5 × room changes per τμήμα-day |

Bands: ≥85 «Πολύ καλό», ≥65 «Καλό», else «Θέλει δουλειά». Each penalty carries its rows (which
τμήμα/teacher, how many hours on how many days). Clicking a row sets that filter. Shown as a ring
in the header and as the top card of the panel tab, renamed «Προβλήματα» → **«Έλεγχος»** (internal
key still `problems`), with a bar chart of hours per day. The weights are a product judgement, not
measured against any data. Tune them in `scheduleQuality`.

### Q.4 2026-10-25 — «Επισκόπηση» (overview) view

A third view next to Εβδομάδα|Ημέρα. The room grid (days × rooms = 30 columns at 6 days × 5 rooms;
1.5 days visible at 1600px) answers "where is there space?". The overview answers "what does Β1
have on Tuesday?":
- rows = **τμήματα** (grouped by τάξη, then an «Ιδιαίτερα» group per student) | **καθηγητές** |
  **αίθουσες** (`pivot`, saved in prefs); columns = days (they share the width,
  `min-width: calc(head + days × 124px)`); each cell lists the day's lessons in order with
  **«κενό Nω»** markers between them (not for rooms). Row header: hours/week, days (teachers),
  gaps, and **«+N»** = palette chips still to place for that row (opens the palette, filtered).
  Clicking a row's name toggles that filter. Rows with nothing matching an active filter are
  **hidden** here (unlike the grid's dimming: a row is a perspective, not a drop target), with a
  «N γραμμές κρύβονται» note.
- **Placing from the overview:** while a lesson is picked up, the rows it can go in (its τμήμα /
  its private student / its teacher; every room in room pivot) list each day's **free times**. The
  admin picks only the time. **`bestRoomDrop`** picks the room: the preferred one (a move keeps its
  room; a placement prefers the τμήμα's most-used room), else the smallest that fits cleanly, else
  the warn with the fewest reasons. Only **clean** times are listed. A day with none shows one
  «N με σύγκρουση…» button that unfolds the override times (`unfolded` set, cleared on every new
  pick-up). Commits go through the same `commitAt` → `placeEntry`/`moveEntry` as the grid.
- The moving lesson stays drawn (dashed, `is-moving`) so `dragend` is not lost, the same rule as
  the grid's ghost.

### Q.5 2026-10-25 — smaller changes

- **Header:** the 4 stat tiles became one KPI row inside the header (lessons + progress, to place,
  conflicts, quality ring, issues only when >0). The «Επεξεργασία στοιχείων» button became a pencil
  next to the title. This saves ~110px before the grid.
- **«Συμπαγής προβολή»** (was «Μόνο αίθουσες με μαθήματα», same pref key `compactRooms`) now also
  **trims the empty hours** at both ends of the day (`computeShownSlots`). The range covers every
  lesson of the shown days, dimmed or not, so no lesson is cut. It is suspended while a lesson is
  picked up. On dev the first screen was 10:00–16:45, all empty; it is now 17:20–20:30.
- **Separate view per form factor:** `view` (wide) and `viewNarrow` in prefs, so choosing the week
  on a desktop no longer opens the 30-column week on a phone.
- **Lazy routes:** `/schedules` and `/schedules/:id` use `loadComponent`, so the editor (the
  biggest single component) left the initial bundle, which sits ~70 kB under its 5.25 MB budget.

### Q.6 Verification (2026-10-25)

- `schedule-grid.util.spec.ts` (new, 15 specs) covers `entrySlots`, `liveStudentIds`,
  `validateSchedule` (both sides, empty τμήμα, touching, student in two places,
  availability/capacity/blocked, τμήμα rules), `idleSlots`, `scheduleQuality`, `bestRoomDrop`.
  Full `test:ci` **1678/1678**; typecheck, eslint, `signals:audit` (component stays at 0 calls /
  0 getters) and `i18n:keys` are clean.
- Run against the **deployed dev API** with the real store data (4 lessons, 26 chips, 5 rooms)
  through a throwaway Playwright probe that **refused any write request**. Checked: week + compact
  (16 → 4 rows, back to 16 while a chip is picked up), overview by τμήμα and by teacher, a chip
  picked up (clean times on the teacher's 2 available days, the other 4 folded), the «Έλεγχος» tab,
  dark mode, phone at 390px (no horizontal page scroll). **Not exercised:** an actual place/move
  from the overview, which writes to the shared dev DB. That path is the same `commitAt` the grid
  uses.

### Q.7 File map

- `src/app/scheduling/schedule-grid.util.ts`: + `AvailabilityIndex.loaded`, `GridIndex`,
  `entrySlots`, `liveStudentIds`, `validateSchedule`, `MAX_CLASS_HOURS_PER_DAY`, `idleSlots`,
  `scheduleQuality` (+ `QualityReport/Penalty/Row`), `bestRoomDrop`, `RoomOption`.
- `src/app/scheduling/schedule-grid.util.spec.ts`: new.
- `src/app/scheduling/scheduling.component.{ts,html}`: live reasons on `EntryVM`, `buildQuality`,
  `buildOverview` (+ `OvRowVM/OvCellVM/OvItemVM/OvTargetVM`), `computeShownSlots`, `allRooms`,
  `setPivot`, `focusRow`, `openRowPalette`, `unfoldCell`, `commitTarget`, `focusQualityRow`, KPI
  header, «Έλεγχος» tab.
- `src/theme/schedule-editor.scss`: `.sd-kpi*`, `.sd-ring`, `.sd-ov*`, `.sd-quality`, `.sd-load*`,
  `.sd-q*`. The `.sd-stat*` / `.sd-edit` rules were removed.
- `src/app/app-routing.module.ts`: both schedule routes are lazy.
- i18n `schedule.page.{view_overview*, pivot*, compact*, check, quality*, day_load, q_*,
  live_check_hint, gap_n, row_*, show_only_row, ov_*}`, `schedule.reasons.{class_busy,
  class_course_twice, class_over_limit}`. `compact_rooms*` were removed.
- `docs/flows/F05-build-timetable.md` (§1, §3, the overview subsection) and
  `docs/releases/unreleased.md` (one entry).

### Q.8 2026-10-25 — API: whole hours only (SCH-01) and the τμήμα's own occupancy (SCH-02)

The first change to `Frontistirio-API/controllers/handlers/schedule.js` in this block (uncommitted at
the time of writing). Details and the reasoning are in the bugs brain repo `scheduling-bugs.md` §6.
- **SCH-01:** a session length must be a whole number of slots. `buildLessons` (`pushSession`) reports
  a fractional one as unscheduled **`fractional_hours`**. The palette lists it as the issue
  **`fractional_hours`** instead of a chip that no cell could take. `evaluatePlacement` refuses it
  with the fatal **`invalid_hours`** (409). Entries already saved with `endTime: "NaN:NaN"` are counted
  by `buildOccupancy(entries, excludeId, D)` as `start + ceil(hours)·D`. The editor's live check
  flags them **`invalid_hours`**. The course form still steps by 0.5, which is an open decision.
- **SCH-02:** `placeLessons` keeps `classBusy[classId|day]` (reason **`class_busy`**, severity 3,
  detail `class_busy` naming the blocking course). `buildOccupancy` also returns `klass`.
  `evaluatePlacement` takes `class_ids` and flags `class_busy`; `placeEntry`/`moveEntry` pass them.
  This supersedes C7's "a τμήμα is busy only through its students".
- New `exports._internals` (`buildLessons`, `buildSlotGrid`, `placeLessons`, `buildOccupancy`,
  `evaluatePlacement`, `buildManualPalette`) and the API's first schedule tests,
  `controllers/handlers/__tests__/scheduleGenerator.test.js` (6 specs on a hand-built context, all
  failing on HEAD). `npm test` 319/319. Frontend `evaluateDrop` now says `class_busy` too, plus i18n
  for the new keys (frontend `test:ci` 1679/1679). **Not deployed.**

### Q.9 2026-10-27 — visual pass: editor chrome, grid, palette, and the `/schedules` list

Frontend only, uncommitted at the time of writing. Verified with a read-only Playwright probe on the
dev API (light/dark/390px, write routes aborted; no console errors), typecheck, eslint, `signals:audit`
(both components at 0 calls / 0 getters; the list dropped its one `rangeLabel()` call), `i18n:keys`,
`test:ci` 1679/1679.
- **Toolbar → one wrapping row:** views · **filter pills** · display controls · panel toggle. The four
  full-width selects became `ion-select.sd-fpill`. The facet name is the select `label`
  (`labelPlacement="start"`), the choice follows, the placeholder is «Όλες/Όλα/Όλοι»
  (`schedule.page.all_f/n/m`), and the ✕ is `ClearableSelectDirective` (its `ionChange` re-filters).
  The options popover is `size: 'auto'` (`pillPopover`), so names are not cut to the pill's width.
  The chip row, `filterChips`, `clearFilter()` and `filtersOpen` were removed. Below 992px the
  pills get a row of their own and scroll sideways.
- **Header figures → one divided strip** (number over label, tone in icon/number). On a phone it
  becomes a grid: lessons across the top, the rest under it.
- **Grid:** slot rules recede to `--sd-line`. Every other day is banded (`--sd-band` as a gradient over
  the opaque background, so the sticky headers stay opaque). Today's day header is underlined. A red
  **current-time line** runs across today's empty cells, with a dot at the day edge and the time in
  red. It is drawn only when this schedule is in effect today, and redrawn every 2 min (`clock`,
  paused while a lesson is picked up). Cards are a touch roomier and show the course without its
  τάξη prefix (`EntryVM.courseShort`).
- **Palette:** each session is a calm full-width row (neutral surface, hue bar, course without
  prefix, teacher, hours). **Identical sessions fold into one row with «×N»**
  (`PaletteRowVM.items`); a drag or tap takes the first. On dev that is 26 → 15 rows.
- **`/schedules` list rebuilt:**
  - a **«Πότε ισχύει τι»** strip of the period (admin's `active_period` override, else the default
    period), month ticks, each alternative's ranges as hue segments over the default, a «Σήμερα»
    flag;
  - a card grid where each card carries a status (**σε ισχύ σήμερα / προσεχώς / σήμερα ισχύει
    άλλο / χωρίς ημερομηνίες / έληξε**, sorted that way after the default), range chips with
    day counts, lessons + «δεν χώρεσαν», «Ενημερώθηκε πριν από …» (`Intl.RelativeTimeFormat`,
    `numeric: 'always'` because «auto» gave «προηγούμενος μήνας»);
  - a ⋯ RowMenu (open / edit / delete) on each card, the whole card a stretched `routerLink`, and a
    dashed «Νέο πρόγραμμα» card that says what an alternative is for.
- Docs: F05 (pills, ×N) and F13 (list statuses, the strip) updated; one bullet added to the release
  note.

### Q.10 2026-10-27 — fill / fresh choice and lesson locking in the editor (UI for R.7)

Frontend only, uncommitted at the time of writing. **Deploy the API first**: the old API ignores
`mode`, so a «Συμπλήρωση υπολοίπων» sent to it runs a full rebuild and undoes every hand placement;
`/lock-entry` does not exist there at all.
- `ScheduleService.generateSchedule(id, mode = 'fresh')` now ALWAYS sends `mode` (also `'fresh'`,
  explicitly); `lockEntry(id, entryId, locked)` → `POST /schedule/lock-entry`. `ScheduleResponse.report`
  typed as `GenerateReport` (`mode/kept/placed/unscheduled/grid/score_before/score_after`).
  `ScheduleEntry.locked`.
- **The generate button** keeps one label («Αυτόματη δημιουργία») with a chevron once the timetable
  has lessons. Click → an `ion-popover` (`genMenu`, `.sd-gen*`) with two described options:
  **Συμπλήρωση υπολοίπων** (disabled with «Δεν λείπει κανένα μάθημα» when `stats.remaining` is 0)
  and **Από την αρχή** (its hint names the locked count), plus a tip pointing at locking. An empty
  timetable (and the two empty-state buttons, which pass no event) still generates «fresh» directly.
- **The confirm moved**: the old «there are manual placements» alert used to fire on ANY manual entry;
  now only «Από την αρχή» with `stats.loose > 0` (manual AND not locked) asks, and the message names
  both counts. «Fill» never asks — it moves nothing.
- **Toasts** read `res.report`: fill → «Μπήκαν ακόμα N μαθήματα — τα υπόλοιπα έμειναν στη θέση
  τους» (+ «M δεν χώρεσαν»), or «Δεν βρέθηκε θέση για κανένα από τα M…»; fresh keeps the old
  generated_all/partial text.
- **Locking**: inspector gets a full-width toggle (`.sd-btn.is-wide`, `.is-on` when locked,
  `aria-pressed`) «Κλείδωμα στη θέση του» / «Ξεκλείδωμα», a «Κλειδωμένο — μένει εδώ…» fact line, a
  `lock-closed` flag on the grid card (before the hand flag) and a legend entry. `EntryVM.isLocked`,
  `stats.locked`, `stats.loose`. Locked lessons can still be moved/removed by hand (the API allows it).
- Verified: typecheck, eslint, `signals:audit` (editor still 0 calls), `i18n:keys`, `api:contract`,
  `plan:contract`, `test:ci` 1681/1681 (3 new service-contract cases: fresh-by-default, fill,
  lock-entry). **Not** driven in a logged-in browser: the probe session had expired; the popover,
  inspector toggle, card flag and legend were checked in a static harness (real compiled
  `global.scss` + `variables.scss`, light + dark), which caught the lock fact line squeezed into half
  of the 2-column facts grid (now `grid-column: 1 / -1`).
- Docs: F05 §5 rewritten (fill vs fresh table, locking), §2/§3/§4 + «Γιατί δεν βλέπω» adjusted;
  two release-note bullets.

---

## Changelog — 2026-09 (b): per-role timetables, co-teaching, multiple schedules

> Recorded **2026-09-14**, verified against `Frontistirio-API` (`controllers/handlers/schedule.js`,
> `controllers/schedule.js`, `routes/schedule.js`, `models/schedule.js`, `models/course.js`,
> `controllers/handlers/course.js`, `controllers/calendar.js`) and the `Frontistirio` frontend.
> Three features built on top of the manual-scheduling work below. All shipped in the API
> commits `ac77227` (09-04) + `28969a7` (09-05) and the frontend commits `c0fb29e` (09-04) +
> `47d6a2d` (09-05); OTA release 1.5.12. Where they change older sections, **this block wins**.
>
> | When | Feature | Summary |
> |---|---|---|
> | 2026-08-28 → 09-14 | **Per-role timetables** (P.1–P.2) | read-only slices of the timetable for student / teacher / parent + PDF export + home "today" widget |
> | 2026-09-05 | **Co-teaching / συνδιδασκαλία** (P.3–P.7) | several τμήματα and/or ιδιαίτερα taught together as ONE placed lesson |
> | 2026-09-05 | **Multiple schedules** (P.8–P.11) | one default timetable + date-ranged alternatives (Χριστούγεννα, Πάσχα…) per store+period |

### P.1 Per-role read-only timetables

The admin grid (`/schedules/:id`) is the only editor. Every other role gets a **slice** of
the timetable that is **active today** (see P.8), read-only:

| Role | Endpoint(s) | Handler | Scope rule |
|---|---|---|---|
| store-user (for one student) | `GET /schedule/get-student-schedule?student_id` + `POST /export-student-schedule` | `getStudentSchedule` | `entryBelongsToStudent` |
| student (self) | `GET /schedule/get-my-schedule` + `POST /export-my-schedule` | `getStudentSchedule` | student resolved from the login (`studentsHandler.getStudentIdByUserId`) — never a client id |
| teacher (self) | `GET /schedule/get-my-teacher-schedule` + `POST /export-my-teacher-schedule` | `getTeacherSchedule` | `entryBelongsToTeacher` (`teacher_id` match); teacher from `teacherHandler.getTeacherByUserId` + store check |
| parent (children) | `GET /schedule/get-my-children-schedule` | `getChildrenSchedule` | union over the kids (`getParentIdsByUserId` → `getStudentsByParentIds`); an entry two kids share is kept once |

- All slice handlers return the **same grid definition** as the store schedule
  (`{ entries, time_slots, slot_minutes, generatedAt, student|teacher }`), so the frontend renders
  an identical weekly grid. A student/teacher is never double-booked → **no classroom
  sub-columns** (rows = slots, columns = days).
- **PDF:** `helpers/createStudentSchedulePdf.js` (pdfkit, landscape A4, class lessons blue,
  ιδιαίτερα amber, DejaVuSans). Uploaded to bucket **`logeion-test-cycle-reports`** under
  `student-schedules/<group>/<store>/…` → 60 s signed URL. The **teacher export reuses the
  student PDF** by passing `{ ...data, student: data.teacher }` — `getTeacherSchedule`
  normalises the teacher's `first_name/last_name` to `firstName/lastName` for exactly this.
  Break gaps show as time jumps in the PDF (no break bands, unlike the web grid).
- The parent has **no PDF and no dedicated page** — only the home widget.

**Frontend**

- `students/student-schedule/` (`app-student-schedule`, standalone, imported in `app.module.ts`
  because `StudentDetailsComponent` is module-declared): inputs `student` (admin → student-details
  **SCHEDULE** tab) or `self` (student → `/my-schedule`, RoleGuard `['student']`, page
  `student/my-schedule/`). **UI rework 2026-09-14:** defaults to a **per-day agenda** with a
  Day|Week toggle persisted in `localStorage['studentSchedule.view']`; weekday picker (Sun only if
  used, count per day, today dot, default = today or next day with lessons); lesson cards with the
  course name minus its τάξη prefix (`withoutPrefix` in `shared/list-card.util.ts`), a course hue
  bar, teacher/room/co-teaching, gap rows (Κενό/Διάλειμμα), now/next/done status refreshed every
  60 s; the week grid is trimmed to the first..last used slot; admin-only link to `/schedules`.
  i18n `schedule.student_view.*`.
- `teacher/teacher-schedule/` (`app-teacher-schedule`, route **`/my-teaching-schedule`**, RoleGuard
  `['teacher']`): same grid, only the teacher's lessons; view filters limited to **τάξη + τμήμα**
  (a teacher can't pick other teachers/courses); PDF export. i18n `teacher.{my_schedule,
  no_teacher_schedule}`.
- `today-schedule/` (`app-today-schedule`, standalone, `@Input() userRole`): the home **"Σημερινά
  μαθήματα"** widget — calls the role's endpoint and keeps entries whose `dayOfWeek` = today
  (JS `getDay()` 0=Sun → 7). Student + teacher get it at the top of their home dashboard main
  column; the parent gets it in the home aside; store-user never (has the full grid).
  "Full schedule" link → `/my-schedule` / `/my-teaching-schedule` (none for parents).
- `services/schedule.service.ts`: `getStudentSchedule`, `exportStudentSchedule`, `getMySchedule`,
  `exportMySchedule`, `getMyTeacherSchedule`, `exportMyTeacherSchedule`, `getMyChildrenSchedule`.

### P.2 Belongs-to predicates (one rule for grids AND calendar)

Extracted as pure, exported functions in `handlers/schedule.js`, so the timetable pages and the
calendar's lesson feed (see the **calendar brain repo**) can never disagree about who sees what:

```js
entryBelongsToStudent(e, sid, classId)   // student_ids ∋ sid
                                         // || (private && student_id == sid)
                                         // || (class && (class_id == classId || class_ids ∋ classId))
entryBelongsToTeacher(e, tid)            // teacher_id == tid
entryBelongsToAnyChild(e, idSet, classIdSet)  // same as student, over sets
getStudentScope(store, period, studentId)     // → { sid, classId }   (period_class lookup)
getChildrenScope(store, period, studentIds)   // → { idSet, classIdSet }
```

The `class_id`/`class_ids` clause is the **safety net** for a student enrolled into a τμήμα
*after* the timetable was generated (their id isn't in `student_ids` yet). `class_ids` makes it
co-teaching-aware (a co-taught entry has `class_id: null`).

### P.3 Co-teaching (συνδιδασκαλία) — concept & data model

An admin declares, **per course and per period**, that several τμήματα and/or ιδιαίτερα students
are taught **together as one physical lesson** (e.g. Β1 + Β2 attend Έκθεση jointly → 7 students,
one teacher, one room, one placement). Members not in any group schedule independently as before.

```js
// models/course.js
period_coteaching: [{
  period: ObjectId → TeachingPeriod,
  groups: [{ _id /* auto — the key schedule entries reference */,
             label: String,            // optional, e.g. "Β Έκθεση κοινό"
             class_ids:   [ObjectId → ClassModel],
             student_ids: [ObjectId → Student] }]
}]

// models/schedule.js — scheduleEntrySchema additions
class_ids:        [ObjectId]   // every τμήμα attending; normal class lesson → [class_id]; private → []
coteaching:       Boolean      // true → class_id is null
coteach_group_id: ObjectId     // → Course.period_coteaching[].groups[]._id
```

- A τμήμα / μαθητής belongs to **at most one group per course** (deduped on save, first wins);
  empty groups are dropped.
- **Teacher is auto-derived** (user decision, no teacher picker): `resolveGroupTeacher` finds the
  teachers whose `Teacher.period_courses[period].courses[course]` includes ANY of the group's
  τμήματα/students. Exactly 1 → that teacher; 0 → `no_teacher`; >1 →
  **`coteaching_teacher_conflict`** (never silently placed).

### P.4 Co-teaching in the generator

- `gatherContext` builds three maps from the courses' `period_coteaching`:
  `coteachGroups[courseId] = [{ group_id, label, class_ids[], student_ids[] }]`,
  `classToGroup[courseId][classId]`, `studentToGroup[courseId][studentId]`.
- Helpers: `coteachLabel(ctx, group)` → `"Β1 + Β2 + Παπαδόπουλος Γ."` (falls back to `label`),
  `groupStudentIds(ctx, group)` → union of every member τμήμα's roster + explicit students,
  `resolveGroupTeacher(ctx, periodId, courseId, group)` → `{ teacher, count }`.
- **`buildLessons` is now two decoupled passes:** (1) one combined lesson per (course, group) ×
  `sessions_per_week` (`type:"class"`, `coteaching:true`, `classId:null`, `class_ids`, union
  `studentIds`); unbuildable groups go to `ctx._coteachIssues`; (2) the normal teacher-assignment
  lessons, **skipping** any τμήμα/student that is a member of a group for that course.
- **`placeLessons`** applies the class rules (same course twice/day, `MAX_CLASS_HOURS_PER_DAY = 6`,
  adjacency penalty) to **every** τμήμα of the lesson via `lessonClasses(lesson)`; writes
  `class_ids / coteaching / coteach_group_id`; appends `ctx._coteachIssues` to `unscheduled`
  (reason `no_teacher` or `coteaching_teacher_conflict`).
- A combined lesson is counted **once** (`lessons_total` 2, not 4) — capacity checks use the union.

### P.5 Co-teaching in manual scheduling

- `buildManualPalette` skips grouped τμήματα/students in the τμήμα and ιδιαίτερα palettes and adds
  **`coteachPalette[]`** — one chip set per (course, group):
  `{ course_id, course_name, coteach_group_id, label, teacher_id, teacher_name, grade_name,
  student_count, sessions:[{hours}] }`. Remaining sessions are subtracted by key
  `` `${coteach_group_id}|${course_id}` ``. Group problems land in `issues[]` with
  `type:"coteaching"` + `kind` `no_teacher | coteaching_teacher_conflict | no_frequency | no_hours`.
- `getManualData` returns `coteachPalette` alongside `classPalette` / `privatePalette`.
- **`placeEntry`** with `type:"class"` + `payload.coteach_group_id` re-resolves the group **server-side**
  from the Course (τμήματα, union students, label) — the client never supplies the student set —
  and writes `class_id:null, class_ids, coteaching:true, coteach_group_id`. `moveEntry` is unchanged
  (it already works off the entry's `student_ids`).
- Frontend `scheduling.component`: a **"Συνδιδασκαλίες"** palette accordion (`onCoteachDragStart`),
  `git-merge-outline` icon on co-taught cells, the τμήμα view filter matches `class_ids`, and
  class-filter options register every `class_id` using names from `selectAllClasses` (a co-taught
  entry only carries the combined label).

### P.6 Changing a group after lessons are placed — stable ids + stale flag

A `Schedule` is a **snapshot**. Editing a co-teaching group does **not** rewrite placed entries —
the admin re-generates or removes/re-places (user decision). Two supports:

1. **Stable group `_id`s.** `updateCourseCoteaching` keeps an incoming group `_id` (the frontend
   carries `CoteachGroup._id` through load → save). Before this fix every course-details save —
   even an unrelated one — minted new ids, orphaning placed co-taught entries and re-offering the
   group in the palette (duplicate risk).
2. **Stale flag (display only).** `getManualData` returns `schedule.toObject()` passed through
   `markCoteachStale(ctx, entries)`: a co-taught entry gets `coteach_stale:true` when its group is
   gone or its `class_ids` / student union no longer equal the group's. The grid shows an amber
   dashed `.entry.stale` + chip `schedule.manual.coteach_outdated` ("Εκτός ημ/νίας", hint
   `…_hint`). Nothing is auto-reconciled.

### P.7 Co-teaching persistence & the bootstrap-projection trap

- **Save:** `controllers/course.js` `upsertCourseDetails` → if `Array.isArray(body.coteaching)` →
  `courseHandler.updateCourseCoteaching(courseId, periodId, groups, session, updatedBy)`:
  `$push` an empty period entry if missing, then `$set "period_coteaching.$[pt].groups"` with an
  `arrayFilters` on the period (full replace; `[]` clears). **No new route.**
- **UI:** `course-details.component` — a "Συνδιδασκαλία" card in the ASSIGNMENTS tab: add/remove
  groups, multi-select of the τμήματα + ιδιαίτερα **already assigned to the course** (an option is
  disabled when it belongs to another group), optional label; `coteachGroups` takes part in the
  backup/diff/save of the page. i18n `courses.coteaching.*`, `schedule.manual.coteach_title`,
  `schedule.{reasons,issues}.coteaching_teacher_conflict`.
- ⚠️ **Trap (fixed 09-05):** the NgRx store hydrates courses from `GET /bootstrap/getSingle` →
  `fetchStorePeriodCourses` in `handlers/course.js`, an **aggregation with an explicit `$project`**
  — not from `get-store-courses`. Its projection omitted `period_coteaching`, so groups vanished on
  reload although they were in the DB. Fixed by projecting `period_coteaching` (filtered to the
  period). **Any new per-period Course field must be added to that `$project` too.**

### P.8 Multiple schedules — model & the active-per-date rule

A store+period now holds **many** timetables: exactly one **default** ("Κανονικό πρόγραμμα") plus
alternatives that are in effect during **date ranges**.

```js
// models/schedule.js — scheduleSchema additions
name:        String
is_default:  Boolean
date_ranges: [{ start: Date, end: Date }]   // non-default only; end inclusive, compared date-only (UTC)

// indexes (the old unique {store_id, period_id} was dropped)
{ store_id:1, period_id:1, is_default:1 }                               // list queries
{ store_id:1, period_id:1 } unique, partialFilterExpression {is_default:true},
                             name "one_default_per_store_period"        // one default
```

(Two distinct key patterns on purpose, so Mongoose doesn't warn about duplicate indexes.)

**Rule** — `resolveActiveSchedule(store, period, date = now)`: the non-default schedule whose
`date_ranges` cover `date` (UTC `YYYY-MM-DD` compare via `dayKey` / `dateInRange`), else the default,
else the first. `activeScheduleForList(schedules, date)` is the same rule over a preloaded array
(used by the calendar's per-day loop). Non-default ranges **may not overlap** — each other or any
other non-default schedule (`validateRanges` → `invalid_range` / `overlapping_range`).

**Migration:** `scripts/migrateSchedules.js` (idempotent) marks each store+period's existing
schedule as the default; `loadSchedules` also **lazily back-fills** (oldest → default,
name "Κανονικό πρόγραμμα") if it finds none.

### P.9 Multiple schedules — CRUD and schedule-targeted editing

| Handler | Behaviour |
|---|---|
| `listSchedules` | summaries `{ _id, name, is_default, date_ranges, entry_count, unscheduled_count, generatedAt, updatedAt }`, default first then by name |
| `createSchedule` | the **first** schedule is always the default; making a new default first unsets the old one; ranges validated for non-defaults; starts empty |
| `updateScheduleMeta` | rename; `is_default:true` → unset the other default first, clear own ranges; ranges only for non-defaults |
| `deleteSchedule` | refuses to delete the only schedule (`cannot_delete_only_default`); deleting the default **promotes another**; **hard** `deleteOne` |
| `getScheduleById` / `findScheduleForStore` | verifies the id belongs to the caller's store **and** period and isn't deleted (IDOR guard) |
| `generateForSchedule(store, period, scheduleId, user)` | runs the (store-wide) generator and replaces **only that doc's** `entries/unscheduled/time_slots`, keeping its meta; no id → the default (created if missing). `generateForStore` kept as a back-compat alias |

`getManualData`, `placeEntry`, `removeEntry`, `moveEntry`, `getStoreSchedule` all take a
`scheduleId` → `findScheduleForStore` instead of `findOne({store, period})`. The controllers read
`schedule_id` from the query/body; `place-entry`, `remove-entry`, `move-entry`, `update`, `delete`
**require** it (`generate` / `get-store-schedule` fall back to the default).

### P.10 Multiple schedules — frontend

- Routing: **`/schedules`** → new standalone `SchedulesComponent` (list); **`/schedules/:id`** → the
  existing `SchedulingComponent` (the grid editor); **`/scheduling` redirects to `/schedules`**. All
  RoleGuard `['store-user']`. Menu + home card repointed (`menu.management.schedules`).
- `SchedulesComponent`: cards (name, «Κανονικό» badge, ranges, entry/unscheduled counts), inline
  create/edit editor (name + default toggle + `<input type=date>` range rows), delete with confirm,
  arrow **`routerLink`** to details, `listNav.setList('schedules')` for prev/next.
- `SchedulingComponent`: reads `scheduleId` from `route.snapshot` (the Ionic route strategy
  recreates the page on param change, so prev/next reloads correctly) and threads it into every
  service call; breadcrumb bar (Αρχική → Προγράμματα → name) + `<app-detail-nav navKey="schedules"
  routeBase="/schedules">`.
- `schedule.model.ts`: `Schedule.{name,is_default,date_ranges}`, `ScheduleSummary`,
  `ScheduleDateRange`, `ScheduleEntry.{class_ids,coteaching,coteach_group_id,coteach_stale}`,
  `PaletteCoteach`, `ManualData.coteachPalette`, `PlaceEntryPayload.coteach_group_id`.
  i18n `schedule.list.*`.

### P.11 Consumers of the active schedule

- **Per-role slices** (P.1) call `resolveActiveSchedule(atDate || now)`. The controllers never pass
  `atDate`, so `/my-schedule`, `/my-teaching-schedule`, the student-details tab and the home widget
  always show the schedule **in effect today** — during Christmas they show the Christmas timetable.
- **Calendar** (`controllers/calendar.js` `resolveLessonEventsForUser`): loads all schedules once,
  pre-buckets each schedule's in-scope entries by weekday, then for every day in the visible range
  picks `activeScheduleForList(schedules, day)` — so a month straddling a holiday range shows the
  normal lessons before/after it and the alternative ones inside it. store-user excluded. Details in
  the **calendar brain repo**.

### P.12 Backend surface added (all `authMiddleware`, tagged `// NEW ROUTE`)

| Method | Path | Role | Controller |
|---|---|---|---|
| GET | `/schedule/list` | store-user | `listSchedules` |
| POST | `/schedule/create` | store-user | `createSchedule` (`{ name, is_default, date_ranges }`) |
| POST | `/schedule/update` | store-user | `updateSchedule` (`{ schedule_id, name?, date_ranges?, is_default? }`) |
| POST | `/schedule/delete` | store-user | `deleteSchedule` (`{ schedule_id }`) |
| GET | `/schedule/by-id` | store-user | `getScheduleById` (`?schedule_id`) |
| GET / POST | `/schedule/get-student-schedule` / `/export-student-schedule` | store-user | one student's slice / PDF |
| GET / POST | `/schedule/get-my-schedule` / `/export-my-schedule` | student | own slice / PDF |
| GET / POST | `/schedule/get-my-teacher-schedule` / `/export-my-teacher-schedule` | teacher | own slice / PDF |
| GET | `/schedule/get-my-children-schedule` | parent | union of children |

Plus `schedule_id` threaded through `generate`, `get-store-schedule`, `manual-data`, `place-entry`,
`remove-entry`, `move-entry`. All new routes were wired on API Gateway (see the aws-infrastructure
brain repo's route sync log).

### P.13 Gotchas & open issues

- ⚠️ **`deleteSchedule` on the default — suspected E11000 (not reproduced).** It promotes
  `others[0]` to `is_default:true` **before** deleting the current default, so for a moment two
  defaults exist and the partial-unique index `one_default_per_store_period` should reject the
  `updateOne`. `createSchedule` / `updateScheduleMeta` do it in the safe order (unset first). Fix:
  delete first (or unset the current flag), then promote.
- `placeEntry` with an unknown/removed `coteach_group_id` does not reject — it places an entry with
  no τμήματα/students (it will then show as stale). Low risk: the id comes from the palette.
- Editing a co-teaching group never updates placed entries (by design, P.6) — the stale chip is the
  only signal; per-role views and the calendar keep showing the old combined lesson until the admin
  re-generates / re-places.
- Role views are always "today's active schedule"; there is no way for a student to preview next
  week's holiday timetable on `/my-schedule` (the calendar page does show it per date).
- The calendar expands lessons for **every** date in the visible range — it does not clip to the
  teaching period's `date_from/date_to` nor skip public holidays unless an alternative schedule
  covers them.
- Deleting a schedule is a hard delete (no recycle bin), although reads filter `isDeleted`.
- Parents: home widget only — no page, no PDF.

### P.14 File map (additions)

- **Backend:** `models/schedule.js` (entry `class_ids/coteaching/coteach_group_id`; schedule
  `name/is_default/date_ranges` + indexes); `models/course.js` (`period_coteaching`);
  `controllers/handlers/schedule.js` (co-teaching helpers, `markCoteachStale`, two-pass
  `buildLessons`, `lessonClasses`, `coteachPalette`, multi-schedule block `dayKey…activeScheduleForList`,
  belongs-to predicates + scopes, CRUD, `generateForSchedule`, `getTeacherSchedule`,
  `getChildrenSchedule`, `atDate` on slices); `controllers/schedule.js` (CRUD + my-teacher /
  my-children + `schedule_id` threading); `routes/schedule.js`; `controllers/handlers/course.js`
  (`updateCourseCoteaching`, `period_coteaching` in the bootstrap `$project`);
  `controllers/course.js` (`body.coteaching`); `controllers/calendar.js` (lesson feed);
  `helpers/createStudentSchedulePdf.js`; `scripts/migrateSchedules.js`.
- **Frontend:** `schedules/schedules.component.*` (new); `scheduling/scheduling.component.*`
  (scheduleId, co-teaching palette, stale chip, breadcrumbs/detail-nav);
  `teacher/teacher-schedule/*` (new); `today-schedule/*` (new); `students/student-schedule/*`;
  `student/my-schedule/*`; `courses/course-details/*` (co-teaching card);
  `models/schedule.model.ts`; `services/schedule.service.ts`; `app-routing.module.ts`;
  `home/home.page.html`; `app.component.html` (menu); `i18n/{el,en}.json`.

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

> **UPDATED 2026-10-25 — see [changelog R](#changelog--2026-10-b-the-generator-becomes-a-search-api).**
> 6.1–6.4 (helpers, context, lessons, grid) still hold. **6.5 is history**: placement lives in
> `scheduleSolver.js` and is a search (most-constrained first, cost-based choice, repair, simulated
> annealing), not a greedy pass. The old pass survives only as `strategy: "legacy"`.

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
