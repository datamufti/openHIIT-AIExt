# Where plugins could hook into OpenHIIT

Research for issue #4. Question: where are the seams a plugin system would hook into (Timer creation and editing, the running-Timer screen, Timer completion, data model and database, settings, navigation, state management)? For each seam: what exists today, the smallest core change that exposes a hook, and how risky that change is for merging upstream.

Sources: this repo at `origin/main` (`d8b8d82`, which equals `upstream/main` `f95cf50`, 2026-08-08, plus fork docs), and the published source of `background_hiit_timer` 1.2.3 (the version pinned in `pubspec.lock:83`), downloaded from pub.dev. Package paths are written `bht:lib/...`. Terms follow `aiext/GLOSSARY.md`: **Timer** is the saved definition (code: `TimerType`), **Interval** is one timed segment (code: `IntervalType`).

## Short answer

The app has no extension points today. It has three useful chokepoints. Only one of them is awkward.

1. **Interval generation** is one pure function, `generateIntervalsFromTimer` (`lib/core/utils/interval_calculation.dart:7-119`). Both save and start go through it. A transform hook here is cheap.
2. **Navigation and providers** are wired in one 91-line `main.dart` (`lib/main.dart:64-89`). It has a global `navigatorKey` (`lib/shared/globals.dart:5`) and plain `MaterialPageRoute` pushes. A plugin registry can sit next to the existing `MultiProvider` without changing any feature code.
3. **The running Timer is the hard part.** The tick loop, Interval advancing and sounds all run inside the third-party `background_hiit_timer` package, in a background-service isolate (`bht:lib/background_timer.dart:200-317`). The app only sees a `TimerState` map, emitted every 100 ms through `FlutterBackgroundService().on('update')`. There are no events for Interval change, completion or pause. The app has to infer them by diffing the snapshots. The smallest hook is to do that diff once, in `RunTimerState`'s `build` callback (`lib/features/run_timer/workout.dart:137`), and publish a typed event stream.

Upstream churn is low. Since the 2025-10-30 refactor (#238), 21 upstream commits have touched `lib/`. The last one was on 2026-05-25. Since then upstream has only merged dependency and CI bumps. Risks to watch: an unmerged upstream branch `refactor` deletes `IntervalProvider` and rewrites `TimerProvider`, `home.dart` and `main.dart`. `home.dart` is the most-edited feature file.

## Seams

### 1. Timer creation and editing (`lib/features/edit_timer/`)

**Today**
- Edit state lives in `TimerCreationProvider` (`lib/core/providers/timer_creation_provider/timer_creation_provider.dart:5-130`). It has one setter per field (`setTimerName`, `setTimerTimeSettingPart`, `setTimerSoundSettingPart`, ...). Each setter `copyWith`s an immutable-ish `TimerType` and calls `notifyListeners`.
- The editor screen `EditTimer` has three hard-coded tabs: `GeneralTab`, `SoundTab` and `EditorTab` (`lib/features/edit_timer/ui/edit_timer.dart:274-297`, tab labels `:419-433`). It also has a hard-coded popup menu with copy and delete (`:308-410`).
- Submit is `_handleSubmit` (`edit_timer.dart:183-239`). It calls `generateIntervalsFromTimer`, rejects an empty result (`:194-202`), and then either starts or saves:
  - start pushes `RunTimer` with the freshly generated Intervals (`:209-214`)
  - save calls `TimerProvider.updateTimer` or `pushTimer` (`:228-233`)
- The data model is `TimerType` (`lib/core/models/timer_type.dart:7-181`). It holds a flat set of fields: `activeIntervals`, `activities: List<String>`, `showMinutes`, `color`. It also holds two child objects, `TimerTimeSettings` (getReady, work, rest, break, warmup, cooldown, restarts) and `TimerSoundSettings` (work, rest, halfway, end, countdown, break sound). `toJson` and `fromJson` (`:134-175`) are the import/export format. They drop unknown keys.
- The legacy `Workout` model (`lib/core/models/workout_type.dart`) is only used to migrate old `WorkoutTable` rows, through `deprecated_workout_repository.dart` and `timer_provider/migrations/migration_1.dart`. `TimerProvider.loadTimers` calls that migration (`timer_provider.dart:40-54`). It is dead weight for plugins. Ignore it.
- Interval calculation (`interval_calculation.dart:7-119`) is a fixed template: Get Ready, then Warmup (plus a Rest after it), then `restarts+1` rounds of `activeIntervals` × (Work, plus Rest except after the last), with a Break or Rest between rounds, then Cooldown. The last Interval gets the end sound. Interval identity is the **name string**: work Intervals take the `activities[i]` name (`:71-73`), and everything else uses literal names. Colors and "is this a work Interval" are decided by matching those names (`interval_calculation.dart:127,148-162`, `run_timer/widgets/functions/functions.dart:19-25`, `run_timer/workout.dart:109-126`).

**Minimal hook**
- (a) Add an `IntervalTransform` list, `List<IntervalType> Function(TimerType, List<IntervalType>)`, applied at the end of `generateIntervalsFromTimer`, just before `return intervals;` (`:118`). That is one line of core change plus a registry. It covers "AI generates or reshapes a Timer's Intervals", because save (`timer_provider.dart:146`), start (`edit_timer.dart:190`) and display (`interval_calculation.dart:124`) all call it.
- (b) Add a widget slot for extra editor tabs: a `List<PluginTab>` read in `_buildTabView` and in the `TabBar` (`edit_timer.dart:274`, `:419`). The `TabController` length is set in `initState`, so it must count the plugin tabs too.
- (c) Add a "Timer saved" callback after `updateTimer` or `pushTimer` (`edit_timer.dart:228-233`), or inside `TimerProvider` (see seam 6).
- Plugin data on a Timer: `TimerType` has no extension field. Don't add a column to `TimerTable`. Keep plugin data in a plugin-owned table keyed by `timerId` (see seam 4).

**Upstream churn**: medium. `edit_timer.dart` has 6 commits since 2025-10-30, last 2026-04-25. The whole `edit_timer/` folder has 8, last 2026-05-25. `interval_calculation.dart` has 3, last 2026-04-25 (#334, the break sound). `timer_type.dart` has 1 (only the refactor).

### 2. The running Timer (`lib/features/run_timer/`)

**Today**
- `RunTimer` (`lib/features/run_timer/workout.dart:20-198`) takes `TimerType timer` and `List<IntervalType> intervals` (`:21-22`). It creates `CountdownController(autoStart: true)` (`:35`), turns the wakelock on in `initState` (`:49`) and off in `dispose` (`:68`), and turns it off or on again when paused or resumed (`:106`).
- `RunTimer` wraps everything in the package's `Countdown` widget. Countdown's `build` callback gets a `TimerState` snapshot (`:134-192`). That callback also does list bookkeeping as a side effect: it moves tiles between `intervalTiles` and `removedTiles` until they match `currentInterval` (`:138-149`). It then picks the portrait or landscape view (`:151-173`) and paints the background from `timerState.status`, which is the Interval name (`:178`, `:109-126`).

How the package works:
- **Process model.** `Countdown._initializeService` writes the Intervals into its own sqflite DB, `background_service_temp_timer.db` (`bht:lib/background_timer.dart:146-149`; `bht:lib/utils/database.dart:8-9`). It then configures and starts a `flutter_background_service` (`bht:lib/background_timer.dart:154-172`). On Android that is a foreground service of type `mediaPlayback`, notification id 888 (`:155-164`). `onStart` is a `vm:entry-point` that runs in the service isolate. It re-reads the Intervals from that DB (`:200-221`).
- **Timing source.** `Timer.periodic` every 100 ms (`bht:lib/utils/constants.dart:2`, `bht:lib/background_timer.dart:271`). Each tick decrements `currentMicroSeconds` by 100,000 (`:278`). This is a tick counter, not wall-clock time, so pauses or late ticks stretch the workout.
- **Advancing.** When the counter hits 0 and this is not the last Interval, it calls `advanceToNextInterval` (`:305-308`; `bht:lib/models/timer_state.dart:69-78`). When the counter is ≤ 0, `status = "End"` (`:275-276`).
- **Sounds.** These are played in the service isolate at exact counter values:
  - countdown sound at 3.5, 2.5 and 1.5 s
  - halfway sound at half the Interval
  - the next Interval's start sound, or this one's end sound, at 0.7 s
  - a "blank" sound every whole second (`:283-312`)

  Files are `packages/background_hiit_timer/assets/audio/<name>.mp3` (`:267-268`, `bht:lib/utils/utils.dart:10-11`), and the audio context is `mixWithOthers` (`:229-231`). Volume is read from SharedPreferences `volume` (0-100) on every play (`bht:lib/utils/utils.dart:10`). The app's sound IDs are the keys of `soundOptions` (`lib/features/edit_timer/ui/widgets/tabs/sound_tab/constants.dart:1-18`).
- **Output.** Every tick, the service sends `service.invoke('update', timerState.toMap())` (`:315`). `Countdown.build` is a `StreamBuilder` on `FlutterBackgroundService().on('update')` (`:127-139`). `TimerState` holds only `paused`, `status` (an Interval name, or `"End"`), `currentInterval`, `currentMicroSeconds`, `intervalMicroSeconds`, `volume` and `changeVolume` (`bht:lib/models/timer_state.dart:7-13`). It has no Timer id, no elapsed total and no event kind.
- **Control.** `CountdownController` (`bht:lib/background_timer_controller.dart`) exposes `start`, `stop`, `pause`, `resume`, `restart`, `skipNext` and `skipPrevious`.
  - Restart and skip are `FlutterBackgroundService().invoke(...)` messages (`bht:lib/background_timer.dart:92-105`).
  - Pause and resume write a SharedPreferences bool `pause` (`:78-90`). The service polls it every tick with `preferences.reload()` (`:272-273`).
  - Stop invokes `stopService` (`:119-123`), and so does `Countdown.dispose` (`:69-76`).
  - The app wires restart and skip in `portrait_timer_view.dart:103-110` (and its landscape twin) and pause in `workout.dart:100-107`. The app keeps its own `_paused` flag (`workout.dart:37`) and does not read `timerState.paused`.

**Minimal hook (events out)**: add a `RunTimerEvents` broadcast `Stream` owned by `RunTimerState`. In the `build` callback (`workout.dart:137`), compare the incoming `TimerState` with the previous one and emit:
- `TimerStarted`
- `IntervalStarted(index, IntervalType)` when `currentInterval` changes
- `Tick(remaining)`
- `Paused` and `Resumed`, from `togglePause`
- `TimerCompleted` on the first `status == 'End'`

Pass `widget.timer` and `widget.intervals`, so subscribers get the real `IntervalType` rather than a name. That is about 30 lines in one file and needs no package change.

**Minimal hook (control in)**: expose a small facade over the existing `CountdownController` (`_controller`, `workout.dart:35`) plus `togglePause` (`:100`): `pause`, `resume`, `skipNext`, `skipPrevious`, `restart`. Publish it through the same registry, or an `InheritedWidget` around the `Scaffold`.

**Minimal hook (UI)**: the screen is a `Stack` of the coloured view plus `TimerComplete` (`workout.dart:175-190`). Adding a `List<Widget Function(BuildContext, TimerState)> overlays` to that `Stack` is a two-line slot.

**Upstream churn**: low. `workout.dart` has 2 commits since the refactor, last 2026-01-29. With `--follow` it has 86 commits since 2023, because it is the oldest file in the app. The whole `run_timer/` folder has 5, last 2026-05-09.

### 3. Timer completion

**Today**
- There is no completion callback. The package declares `Countdown.onFinished` (`bht:lib/background_timer.dart:21,28`) but never calls it. A grep finds only the declaration.
- Completion is `TimerComplete` being made visible when `timerState.status == 'End'` (`workout.dart:184-189`). The service keeps ticking after End and re-emits `End` every 100 ms (`bht:lib/background_timer.dart:271-276,315`).
- `TimerComplete.build` calls `controller.play()` on every rebuild (`lib/features/run_timer/widgets/timer_complete.dart:51`). It offers "Back" (`Navigator.pop`, `:101-112`) and "Restart" (`:116-125`).
- Nothing is persisted. There is no workout history or log table (see `lib/core/db/tables.dart`).

**Minimal hook**: the `TimerCompleted` event from seam 2, emitted once per run (latch it, reset on restart). Optionally add a widget slot inside `TimerComplete`'s `Column` (`timer_complete.dart:80-126`) for plugin content such as an AI summary. History, if wanted, belongs in a plugin-owned table.

**Upstream churn**: low. `timer_complete.dart` has 2 commits, last 2026-01-04, a wording change (#291).

### 4. Data model and database (`lib/core/db/`)

**Today**
- One sqflite DB, `core1.db`, `databaseVersion = 8` (`lib/core/db/database.dart:9,12`). It is a singleton `DatabaseManager`. On Windows and Linux it uses FFI with an **in-memory** DB (`:25-30,41-43`), so desktop builds don't persist.
- `onCreate` creates `IntervalTable`, `TimerTable`, `TimeSettingsTable` and `SoundSettingsTable` (`database.dart:48-55`; schemas at `tables.dart:33-87`). The legacy `WorkoutTable` is defined at `tables.dart:7-31` and is only read for the migration.
- Migrations are a version-keyed map, `migrations` (`lib/core/db/migrations/migration_index.dart:6-15`), run by `runMigrations` (`migration_runner.dart:6-25`). The runner also re-runs `CREATE TABLE IF NOT EXISTS` for the 4 tables on every upgrade (`:19-24`). Versions 5-7 are no-ops, and 8 adds `breakSound` (`migration_8.dart:4-12`).
- There are also data-level migrations in `TimerProvider.loadTimers` (warmup and break-sound fixes, `timer_provider.dart:60-74`).
- Repositories, one per table: `lib/core/db/repositories/*.dart`.
- Import/export is a JSON array of `TimerType.toJson()`:
  - export: `file_util.dart:28,56`
  - import: `import_export_util.dart:47-72`, then `saveImportedTimers` with a copy-or-skip dialog on id clash (`:74-110`)
  - entry points: `lib/features/import_export_timers/utils/functions.dart:8-30`
  - Intervals are not exported. They are regenerated on `pushTimer` (`timer_provider.dart:103`).

**Minimal hook**: don't touch `databaseVersion` or the core tables. Merging upstream's next migration would conflict on the integer and the map. Two options:
- Each plugin opens its own sqflite file. This needs no core change.
- Expose `DatabaseManager().database` and let a plugin create `plugin_<id>_*` tables with `IF NOT EXISTS` and its own version row.

For export, the smallest change is an optional `extensions` map in `TimerType.toJson()` and `fromJson()` (`timer_type.dart:134-175`), filled from a registry. Plugin data then round-trips without a schema change. `fromJson` already tolerates missing keys.

**Upstream churn**: low, but conflicts are severe when they happen. `lib/core/db/` has 4 commits since the refactor, last 2026-04-25 (#334 added migration 8 and bumped the version). Any fork edit to `database.dart:9`, `migration_index.dart` or `tables.dart` will conflict on every upstream schema change.

### 5. Settings (`lib/features/settings/`)

**Today**
- `SettingsPage` (`lib/features/settings/ui/settings.dart:10-162`) is a static `SettingsList` with two hard-coded sections: Appearance (theme and accent colour, `:56-91`) and About (`:92-120`).
- Preferences are scattered `SharedPreferences` keys:
  - theme: `ThemeProvider`, `theme_provider.dart:30-47`
  - `volume` and `changeVolume`: `workout.dart:84-98`, `control_bar.dart:43-51`
  - `pause`: owned by the package
  - what's-new: `whats_new_service.dart`
- It is reached from the home app bar (`lib/features/home/ui/widgets/app_bar.dart:22-24`) and the nav rail (`home.dart:320-333`). The page was added upstream on 2026-05-02 (#335).

**Minimal hook**: insert `...pluginRegistry.settingsSections(context)` into `sections:` (`settings.dart:55`). That is one line. Plugins own their prefs under a key prefix.

**Upstream churn**: low for now (2 commits, last 2026-05-25). The page is new, though, so upstream may add sections. Keep the fork's change to the one spread line.

### 6. State management (providers) and app wiring

**Today**
- `main()` builds a `ThemeProvider` first (`lib/main.dart:48-56`). `WorkoutTimer` then wraps `MaterialApp` in a `MultiProvider` with `IntervalProvider`, `TimerProvider` (a `ChangeNotifierProxyProvider` that is handed the `IntervalProvider`) and `TimerCreationProvider` (`main.dart:64-75`).
- `TimerProvider` (`lib/core/providers/timer_provider/timer_provider.dart:17-173`) owns the in-memory list and does all CRUD: `pushTimer` `:96`, `pushTimerCopy` `:107`, `updateTimer` `:118`, `deleteTimer` `:162`, `updateTimerOrder` `:85`. Each one writes to the repositories and then calls `notifyListeners`.
- `IntervalProvider` is a thin cache over `IntervalRepository` (`interval_provider.dart:5-33`). Nothing in the run path reads it: `RunTimer` gets its Intervals passed in.

**Minimal hook**:
1. Add a `PluginHost` (registry) provider to the `MultiProvider` list (`main.dart:65`). Plugins register in `main()` before `runApp`.
2. For Timer lifecycle events, add a `Stream<TimerChange>` (created, updated, deleted) to `TimerProvider` and emit at the four CRUD methods. Alternatively, a plugin can `addListener` and diff the `timers` list with no core change, at the cost of losing intent.

**Upstream churn**: medium, with a known pending change. `lib/core/providers/` has 6 commits, last 2026-05-25. `timer_provider.dart` has 4. Upstream branch `refactor` ("decouple timerprovider from intervalprovider", 2026-05-11, not merged into `upstream/main`) deletes `interval_provider.dart` and edits `timer_provider.dart`, `main.dart` and `home.dart` (`git diff --stat upstream/main...upstream/refactor`: 8 files, +71/-85). Hooks should not depend on `IntervalProvider`.

### 7. Navigation and routing

**Today**
- No named routes. `MaterialApp(home: ListTimersPage, navigatorKey: navigatorKey)` (`main.dart:78-86`; `globals.dart:5`). Every screen is an imperative `Navigator.push(MaterialPageRoute(...))`:
  - home to editor: `home.dart:184-190`, `:242-250`, `:280-284`
  - editor to run: `edit_timer.dart:209-214`
  - home to settings: `app_bar.dart:22-24`, `home.dart:327-331`
- The only way to start a Timer is from the editor's Start button. Home never constructs `RunTimer`; a grep finds the only call at `edit_timer.dart:210`.
- Home builds its bottom bar (`home.dart:198-256`) and nav rail (`:261-339`) from hard-coded `NavBarIconButton`s.

**Minimal hook**:
- Plugins can push their own pages with the existing global `navigatorKey`. This needs no core change.
- For an entry point on home, add a widget slot (`List<NavAction>`) to the bottom bar and nav rail children.
- For "start this Timer" from a plugin, pull the push at `edit_timer.dart:209-214` into a function `startTimer(context, TimerType)` that a plugin can call.

**Upstream churn**: `home.dart` is the hottest feature file, with 11 commits since the refactor (last 2026-05-02). It is also in the pending `refactor` branch (+/-53 lines). Keep edits there to slot insertions.

## Constraints

- **SDK.** `pubspec.yaml` declares `sdk: '>=3.3.0 <4.0.0'`. The resolved lockfile needs `dart >=3.11.0`, `flutter >=3.38.4` (`pubspec.lock`, `sdks:`). CI uses Flutter `channel: stable` (`.github/workflows/analyze.yaml:33`, `integration_tests.yaml:40`). No Flutter SDK is installed in this dev environment (`which flutter` finds nothing), so nothing here was run.
- **State and dependencies.** `provider` ^6.1.2 is the only state library (`pubspec.yaml`). The timer engine is `background_hiit_timer` ^1.2.3, written by the upstream author (`a-mabe/background_timer`; see the commented git dependency in `pubspec.yaml`). It pulls in `flutter_background_service`, `flutter_local_notifications`, `sqflite` and `openhiit_audioplayers`. Android `min_sdk_android: 34` is set for the launcher icons (`pubspec.yaml`).
- **Tests.** There is no `test/` directory, so no unit or widget tests exist. `integration_test/` has 8 end-to-end `testWidgets` flows: `load_test`, `simple_test`, `advanced_test`, `edit_timer_test`, `customize_test`, `interval_adjust_test`, `minutes_view_test` and `restart_rounds_test`. Shared helpers are in `integration_test/utils/`. They run with `flutter drive` on an API-36 emulator in CI (`integration_tests.yaml:76,92`). None of them asserts on `TimerState` or completion events. `generateIntervalsFromTimer` is pure, so it is the easiest place to start unit tests.

## What makes the event and control APIs hard

1. **Engine in another isolate and package.** Ticks, advancing and sounds happen in the service isolate (`bht:lib/background_timer.dart:200-317`). A plugin cannot run code there, for example a custom cue, without forking or changing the package. The only channel back is the untyped `'update'` map.
2. **Snapshots, not events.** You have to infer Interval changes, completion and pause by diffing `TimerState`. `End` repeats every 100 ms, and `onFinished` is never fired. Diffing in `RunTimerState` works, but only while the UI is alive.
3. **Events stop when the UI is gone.** The stream is consumed by a `StreamBuilder` (`bht:lib/background_timer.dart:127-139`), so with the app backgrounded and the widget not rebuilding, a UI-side event bus may lag. Leaving the run screen disposes `Countdown`, which stops the service (`:69-76`), so there is no Timer running "headless" without the screen.
4. **Inconsistent control paths.** Pause goes through SharedPreferences polling (`:78-90,272-273`), skip and restart through service messages, and the app tracks pause separately (`workout.dart:37,100-107`). A control API has to go through `RunTimerState.togglePause`, not `CountdownController.pause` directly, or the UI flag goes out of sync.
5. **Only one run at a time, with fixed Intervals.** The package clears and rewrites its temp DB on every start (`bht:lib/background_timer.dart:146-149`). You can't change Intervals mid-run (for example, AI extends a rest) without restarting the service with a new list.
6. **Names as identity.** Interval kinds are strings. A work Interval named after an activity like "Rest" would be miscoloured and miscounted (`functions.dart:19-30`, `interval_calculation.dart:127`). Events should carry the `IntervalType` and its index, not the name.
7. **Tick-counted timing.** Elapsed time is `ticks × 100 ms`, not a clock (`bht:lib/background_timer.dart:271-278`). Plugins that log durations should timestamp events themselves.

## Summary table

| Seam | Today | Minimal hook | Upstream churn risk |
|---|---|---|---|
| Timer editing (`edit_timer/`, `TimerCreationProvider`) | Hard-coded 3 tabs and popup menu; `_handleSubmit` saves or starts (`edit_timer.dart:183-239`) | Extra-tab slot in `_buildTabView`/`TabBar`; "Timer saved" callback | Medium: 8 commits since 2025-10-30, last 2026-05-25 |
| Interval calculation | One pure fixed-template function (`interval_calculation.dart:7-119`), used by save, start and display | Registry of `IntervalTransform`s applied before `return` (1 line) | Low-medium: 3 commits, last 2026-04-25 |
| Data model (`TimerType`, legacy `Workout`) | Flat fields plus time and sound settings; `Workout` only for migration | Optional `extensions` map in `toJson`/`fromJson`; plugin data elsewhere | Low: 1 commit (refactor) |
| Running Timer (`run_timer/workout.dart` + `background_hiit_timer`) | Service-isolate 100 ms tick, sounds in service, `TimerState` snapshots via `on('update')`, wakelock in `RunTimerState` | Diff snapshots in `build` callback and emit a typed `RunTimerEvents` stream; control facade over `_controller` + `togglePause`; overlay slot in `Stack` | Low: 2 commits, last 2026-01-29 (package changes need a fork: high) |
| Timer completion | `status=='End'` shows `TimerComplete`; `onFinished` never fired; no history | Latched `TimerCompleted` event; slot in `TimerComplete` column | Low: 2 commits, last 2026-01-04 |
| Database (`core1.db` v8, sqflite) | 4 tables + legacy; version-keyed migration map; in-memory on desktop | Plugin-owned DB file or `plugin_*` tables; never bump core version | Low frequency, high conflict if `database.dart:9` / `migration_index.dart` edited |
| Import/export | JSON array of `TimerType.toJson()`; Intervals regenerated | Round-trip `extensions` map | Low: 3 commits, last 2026-01-29 |
| Settings | Static `SettingsList`, 2 sections; ad-hoc SharedPreferences keys | Spread plugin sections into `sections:` (1 line) | Low now (2 commits), page is new (2026-05-02) |
| State / app wiring (`main.dart`, providers) | `MultiProvider` of Interval/Timer/TimerCreation; `TimerProvider` does CRUD | `PluginHost` provider; `Stream<TimerChange>` from CRUD methods | Medium: unmerged upstream `refactor` removes `IntervalProvider`, edits `TimerProvider`/`main.dart` |
| Navigation | Imperative `MaterialPageRoute`; global `navigatorKey`; start only from editor | Use `navigatorKey` (no change); nav-action slot in home bar/rail; extract `startTimer()` | Medium-high for `home.dart` (11 commits, last 2026-05-02, also in `refactor`) |
