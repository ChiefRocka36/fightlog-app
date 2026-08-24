# Fight Log — Project Notes

This file captures *why* things are the way they are — decisions made in conversation
that aren't obvious from reading the code alone. If you're picking this project back up
in a new session, read this first.

## Architecture

- **Single file, no build step.** Everything lives in `index.html`: React 18.3.1 +
  ReactDOM 18.3.1 + Babel Standalone 7.24.7, loaded via pinned CDN `<script>` tags and
  transformed in-browser (`<script type="text/babel">`). No bundler, no npm install.
  This was a deliberate choice to keep the app trivially deployable to GitHub Pages and
  editable from anywhere.
- **CDN versions are pinned on purpose.** An early deploy used unpinned `unpkg.com`
  URLs and broke with a blank screen because `unpkg` resolved to a newer, incompatible
  Babel version at request time. Never unpin these.
- **PWA via `manifest.json` + `sw.js`.** The service worker is cache-first. `sw.js` has
  a `CACHE` version string (`"fight-log-vN"`) — **bump it on every deploy**, or clients
  keep serving the old cached `index.html` indefinitely.
- **Deployed via GitHub Pages** from `https://github.com/ChiefRocka36/fightlog-app.git`
  (branch `main`) at `https://chiefrocka36.github.io/fightlog-app/`.
- **Persistence** is `window.storage`, a thin wrapper around `localStorage` (per-key,
  prefixed `fl:`). Everything goes through a single `persist(key, value, setter)`
  helper in `App`. Data is local-only — the "Backup" section in Stats is the only way
  to move data between devices (export/import JSON).

## Discipline naming

`DISCIPLINES` keeps the internal key `"conditioning"` even though the UI label was
changed to **"Endurance"**. This is intentional backward compatibility — old saved
sessions have `discipline: "conditioning"` and renaming the key would silently orphan
them. If you ever add a true rename, you'd need a one-time migration, not a find/replace.

## RPE was removed entirely

RPE tracking existed early on and was deliberately ripped out at the user's request —
don't re-add "RPE" fields, labels, or references. (Watch for stale copy — e.g. the old
`Empty` state text on the Log tab used to say "discipline, minutes, RPE" and had to be
fixed after the fact.)

## Strength sessions have a Duration field again

Duration was removed from Strength sessions once, then explicitly restored, because the
Stats minutes-per-week chart sums `session.duration` across *all* disciplines with no
per-discipline special-casing — without a duration value, Strength sessions were
invisible in the weekly-minutes chart. If Duration is ever removed from a discipline
again, the chart needs an explicit exception, or that discipline vanishes from the chart.

## Goals: progress is computed, never stored

Weekly/Monthly/Life goal progress is **not** a counter that gets incremented when you
log a session. It's derived live every render by filtering the `sessions` array by date
range + discipline (`goalWeekProgress`, `goalMonthProgress`, `goalAllTimeProgress`).
This was a deliberate design choice: it avoids any sync bug between "goal state" and
"session state," and it naturally handles multiple sessions logged on the same day
(each session counts individually, not a single done/not-done boolean per day). Don't
add a stored/incremented progress field — recompute from `sessions` instead.

## S&C program generator — design intent

The generator asks a short questionnaire (Goal / Sessions-per-week / Equipment /
Familiarity for Strength; Sessions / Equipment / Familiarity for Endurance) rather than
shuffling random exercises. The exercise selections, rep/set schemes, and substitution
rules were dictated exactly by the user (a sports scientist) and encode real
programming logic, e.g.:

- Exercises requiring high skill/coordination (e.g. **Power Clean**) only appear for
  **Advanced** familiarity; anything below that substitutes a simpler equivalent (e.g.
  **Kettlebell Swing**).
- No isolated single-joint arm work (curls, extensions) except in Home-Gym-equipment
  workouts, where equipment limitations make compound-only programming impractical.

If you touch `buildStrengthWeek` / `buildEnduranceWeek` / the `SC_*` constants, treat
the existing substitution rules as intentional programming decisions, not arbitrary
defaults — check with the user before changing them.

**"Build your own"** was moved above the generator (it used to be below) because manual
plan-building is the more frequently used path for this user.

## Exercise identity: library + autocomplete + type inference

Three related requests turned out to be one mechanism:

1. "LAST TIME" reference values (shown when logging a Strength exercise) are matched by
   **exact, case-insensitive exercise name**. A typo (e.g. "Bench Pres") silently
   creates what the app treats as a brand-new exercise with no history — no reference
   value, and it fragments PR/history tracking.
2. Bodyweight exercises (Pull-Up, Dip, Push-Up, Muscle-Up, ...) shouldn't show a bare
   "Weight" field implying you must enter your bodyweight — it should read **"Added
   Weight"** and be optional.
3. Hold exercises (Plank, Wall Sit, Back Extension Hold, ...) should log **Time**
   instead of **Reps** in the second input.

All three are solved by:

- `EXERCISE_LIBRARY` — a curated lexicon of common strength exercises with a `type`
  (`"weighted" | "bodyweight" | "hold"`).
- `inferExerciseType(name)` — exact library match first, then regex fallback for
  hold/bodyweight patterns, else defaults to `"weighted"`. Drives field labels,
  placeholders, and input mode (`inputMode="text"` for hold, so times like "45s" can be
  typed) everywhere an exercise's sets are logged or displayed.
- `collectKnownExerciseNames(sessions, plans)` — merges the library with every
  exercise name the user has ever actually logged or saved in a plan (case-insensitive
  dedupe), so **custom/unusual exercise names the user already uses are suggested too**,
  not just the built-in lexicon.
- `ExerciseNameInput` — a reusable autocomplete text input (matches-as-you-type,
  click-to-select) built on top of `collectKnownExerciseNames`. It's used everywhere an
  exercise name is freely typed: session logging (`ExerciseLogEditor`), the S&C
  "Build your own" manual plan builder, and the Personal Records form (`PRForm`).

`exerciseNames` is computed once in `App` via
`useMemo(() => collectKnownExerciseNames(sessions, plans), [sessions, plans])` and
threaded down through props. If you add a new place where an exercise name is typed
freely, wire it through `ExerciseNameInput` + `exerciseNames` rather than a plain
`Input` — that's the whole point of this mechanism.

## Editing sessions and renaming plans

- `SessionForm` is reused for both creating and editing: pass an `editSession` prop and
  it initializes all fields from that session instead of defaults, changes its header to
  "Edit session," and reuses the original `id`/`createdAt` on save so the caller can
  tell an update from a new session. `App.updateSession` also keeps the auto-created
  Calendar event (linked via `sourceSessionId`) in sync with the session's date/
  discipline.
- `PlanRow` supports inline renaming via a pencil icon. It's deliberately **not** a
  `<input>` nested inside the row's toggle `<button>` — clicking inside a nested input
  would bubble up and toggle the row open/closed. Instead the row header is a plain
  `<div>` with separate rename (pencil) and expand (chevron) buttons.

## If you're picking this up in a new conversation

The conversation history (design rationale, rejected alternatives, exact wording of
requests) does **not** persist between sessions — only this file and the committed code
do. When something about the code looks arbitrary, check here before assuming it's
accidental.
