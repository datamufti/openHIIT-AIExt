# Changing OpenHIIT safely

How to add to or change upstream OpenHIIT without breaking saved data, other installs or upstream merges. This is a living document. The frozen picture of the code it starts from is [openhiit-1.7.0-baseline-2026-10-11.html](openhiit-1.7.0-baseline-2026-10-11.html). File and line references are in the research note [Where plugins could hook into OpenHIIT](https://github.com/datamufti/openHIIT-AIExt/blob/research/openhiit-seams/aiext/research/openhiit-seams.md).

## Contracts to keep

Each of these already has something depending on it. Change it only in the way shown.

| Contract | Who relies on it | Backwards-compatible change | Merge risk |
|---|---|---|---|
| Database schema (`core1.db` v8) | Every saved Timer on the device. Upstream's next migration bumps the same version number. | Leave core tables and `databaseVersion` alone. New data goes in plugin-owned tables or a separate database file with its own version. A core column that is truly needed is nullable with a default. | High: two forks bumping one integer conflict on every merge. |
| Export JSON | Files people have already exported. Stock OpenHIIT importing our files, and us importing theirs. | Add optional keys only, ideally one `extensions` object. Never rename or remove a key. Stock OpenHIIT drops unknown keys on import but still loads the Timer. | Low |
| Interval names as type | Colours, work detection and sound rules on the run screen. The `name` column in `IntervalTable`. | Add an explicit kind (work, rest, break…) beside the name. When it is missing, fall back to name matching, so old rows and exports behave as before. | Medium: touches several run-screen files. |
| Timer engine (`background_hiit_timer`) and `TimerState` | The run screen. Published separately on pub.dev by the upstream author. | Wrap it instead of forking: derive events by comparing snapshots, and expose controls through one interface of ours. If it must be forked, keep every `TimerState` key and only add new ones. | High if forked, low if wrapped. |
| SharedPreferences keys | Saved theme, volume and what's-new state. The engine reads `pause` and `volume` directly. | Never reuse or rename a key. Prefix new keys with `aiext.<plugin>.`. | Low |
| Screens and flows | The 8 integration tests, which tap through the real UI. People's habits. | Plugin UI goes in slots that are empty when no plugin is on. With every plugin off, the app looks and tests exactly like upstream. | Medium: `home.dart` is upstream's most-edited screen. |
| Provider methods | Every feature screen. The unmerged upstream `refactor` branch removes `IntervalProvider`. | Keep method signatures. Add a change stream beside them. Build nothing on `IntervalProvider`. | Medium |

## Patterns

- **Expand, then contract.** Add the new form beside the old one, support both, move readers over, and remove the old form only once nothing writes it. Example: an explicit Interval kind next to the name string.
- **Optional with a default.** Every new field is optional, and its absence means "behave as before". `TimerType.fromJson` already works this way with `?? default`.
- **Wrap the engine.** One class of ours sits between the app and `background_hiit_timer`. Today it compares snapshots; later it could hide a forked engine without the screens noticing.
- **Default off.** New behaviour sits behind a plugin toggle that starts off, so upstream's integration tests double as the regression suite for "no plugins on".
- **Own your storage.** Plugins keep data in tables or files they own, keyed by Timer id, so upstream schema changes never collide with ours.
- **One-line hooks.** Where core code must change, keep it to one line that calls a registry. Small edits merge cleanly.

## Where additions can land

The smallest core change that opens each part of the app to plugins.

| Area | Smallest hook | Core change |
|---|---|---|
| Interval generation | A list of transforms applied at the end of `generateIntervalsFromTimer`. Covers save, start and preview. | 1 line plus a registry |
| App wiring | A plugin host provider added to the `MultiProvider` in `main.dart`. | 1 line |
| Running Timer, events | Compare each `TimerState` with the previous one in `RunTimerState` and publish typed events (Interval started, paused, resumed, completed). | About 30 lines in `run_timer/workout.dart` |
| Running Timer, control | A facade over the existing controller and `togglePause` (pause, resume, skip, restart). Must go through `togglePause` so the screen's paused flag stays in sync. | Small, same file |
| Run screen UI | An overlay slot in the run screen's `Stack`. | 2 lines |
| Completion | A completed event fired once per run, plus a slot in the completion panel. | Small |
| Editor | Extra tabs in the editor, and a "Timer saved" callback. | Small, in `edit_timer.dart` |
| Settings | Plugin sections spread into the settings list. | 1 line |
| Home | Plugin actions in the bottom bar and nav rail. Extract the editor's start code into a `startTimer()` function that plugins can call. | Small, in `home.dart` (watch merges) |
| Export | Round-trip an `extensions` map in `TimerType.toJson` / `fromJson`. | Small |

## Limits that need more than a hook

- The engine runs in a background service, so plugins cannot run code inside the tick loop (for example a custom sound cue) without changing the package.
- Intervals are copied into the engine at start. Changing them mid-run means restarting the service with a new list.
- Leaving the run screen stops the service. Nothing can run a Timer without that screen open.
- Elapsed time is counted in 100 ms ticks, not by a clock. Plugins that log durations should timestamp events themselves.
