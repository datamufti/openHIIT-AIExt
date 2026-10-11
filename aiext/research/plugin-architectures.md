# Plugin architectures for a Flutter app

Research for [issue #3](https://github.com/datamufti/openHIIT-AIExt/issues/3). Sources checked 2026-10-11.

## Answer

A Flutter release build cannot load new *compiled* Dart code at runtime. Release Dart is AOT-compiled into the app binary on both Android and iOS, and Flutter has no built-in code push. The only runtime paths are:

- an **interpreter** bundled in the app (JS, Dart bytecode, Lua, WebAssembly);
- **declarative UI** data (Remote Flutter Widgets);
- on Android only, **separate plugin APKs** that the app talks to over intents.

Nothing that arrives at runtime can bring native code. That matters for on-device AI, because inference runtimes are native libraries.

So the practical base is **compiled-in Dart plugins behind a host interface**. Give them an Obsidian-shaped contract: a manifest, an `onload`/`onunload` lifecycle, per-plugin enable/disable and settings, and a minimum-app-version field. Keep the host API narrow and serializable, so that an interpreted (JS or Dart-bytecode) tier for community plugins can be added later without redesign.

## 1. Can a Flutter release build load new code at runtime?

### The Dart side

- Release code is AOT-compiled to native machine code on both Android and iOS ([Flutter FAQ](https://docs.flutter.dev/resources/faq)). The Dart VM and hot reload exist only in debug mode ([Flutter FAQ](https://docs.flutter.dev/resources/faq)).
- "Flutter doesn't provide built-in code push functionality." The FAQ points to third-party Shorebird ([Flutter FAQ](https://docs.flutter.dev/resources/faq)).

### Deferred components (Play Feature Delivery)

- Deferred components use Dart `deferred` imports. They work on Android and web only, and on Android they rely on Play Feature Delivery and an AAB uploaded to Play ([Flutter docs: deferred components](https://docs.flutter.dev/perf/deferred-components)).
- They are split out of the *same* compile. "Flutter doesn't dispatch partial updates without re-uploading a new AAB," so new or changed code needs a full release ([Flutter docs](https://docs.flutter.dev/perf/deferred-components)).
- Distributing without Play needs a custom `DeferredComponentManager`, your own packaging tooling and a file server. This is "only recommended for advanced developers" ([Flutter wiki: Deferred Components](https://github.com/flutter/flutter/wiki/Deferred-Components)).
- Loading units are only known after `gen_snapshot` compiles the app ([Flutter wiki](https://github.com/flutter/flutter/wiki/Deferred-Components)).
- **Verdict:** this is lazy loading of first-party code, not a plugin mechanism. It also does not fit sideloaded APKs.

### Shorebird code push

- On iOS, Shorebird runs changed Dart code in a custom interpreter ([Shorebird FAQ](https://docs.shorebird.dev/code-push/faq/)).
- It works with any distribution method, sideloading included ([Shorebird FAQ](https://docs.shorebird.dev/code-push/faq/)).
- "Changes which involve native code cannot be patched" ([Shorebird FAQ](https://docs.shorebird.dev/code-push/faq/)).
- The free Hobby tier supports a single developer ([Shorebird FAQ](https://docs.shorebird.dev/code-push/faq/)).
- **Verdict:** it ships patches to the whole app from one publisher. It is not a way for third parties to add plugins.

### Interpreters inside the app

| Option | Package | Notes |
|---|---|---|
| JavaScript | [`flutter_js`](https://pub.dev/packages/flutter_js) 0.8.7 | QuickJS on Android, JavaScriptCore on iOS. Unverified uploader. |
| JavaScript (fork) | [`flutter_qjs`](https://github.com/kodjodevf/mangayomi/blob/main/pubspec.yaml) | Used by Mangayomi (see section 2) |
| Dart bytecode | [`dart_eval`](https://pub.dev/packages/dart_eval) 0.8.5 / `flutter_eval` | See the notes below |
| Lua 5.3 | [`lua_dardo`](https://pub.dev/packages/lua_dardo) 0.0.5 | Pure-Dart VM. Last published about 3 years ago, so effectively unmaintained. |
| WebAssembly | [`wasm_run_flutter`](https://pub.dev/packages/wasm_run_flutter) 0.2.0+1 | Wasmi on iOS and on most Android ABIs, Wasmtime on Android arm64 and desktop |

Notes on `dart_eval`:

- It is about 10 to 50 times slower than AOT Dart.
- Sandboxed by default: no filesystem, network or process access unless granted with `runtime.grant`.
- Generators, mixins, isolates and other features are still missing.
- Its own docs call Apple's stance on it "unclear".

Notes on `wasm_run_flutter`: it binds Rust runtimes through `flutter_rust_bridge` ([wasm_run docs](https://pub.dev/documentation/wasm_run/latest/)), and plugin authors would need a toolchain that targets Wasm.

### Server-driven UI

- [`rfw`](https://pub.dev/packages/rfw) (Remote Flutter Widgets) is published by flutter.dev. It renders declarative widget trees loaded at runtime from widgets the host provides.
- It has no logic of its own. Events go to `onEvent`, and the package suggests pairing it with `wasm_run_flutter` when logic is needed.
- It warns that moving a whole app onto it "won't work".
- It fits plugin-contributed UI cards, not plugin behaviour.

### Store rules

- **Apple App Store, guideline 2.5.2:** apps may not "download, install, or execute code which introduces or changes features or functionality of the app" ([App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)).
- **Apple guideline 4.7:** this now explicitly allows "HTML5 and JavaScript mini apps … and plug-ins", under conditions ([guidelines](https://developer.apple.com/app-store/review/guidelines/)):
  - **4.7.2:** no exposing native platform APIs to that software without Apple's permission.
  - **4.7.3:** per-instance user consent before sharing data or permissions with it.
  - **4.7.4:** an index of the offered software.
- **Google Play:** apps may not download dex, JAR or `.so` files from outside Play. The rule "does not apply to code that runs in a virtual machine or an interpreter" with indirect API access, such as JS in a webview ([Play Device and Network Abuse policy](https://support.google.com/googleplay/android-developer/answer/9888379)).
- **Sideloaded GitHub APKs:** no store policy applies. Android 14+ still requires any dynamically loaded DEX, JAR or APK file to be read-only, or it throws, and Google recommends avoiding dynamic code loading ([Android 14 behavior changes](https://developer.android.com/about/versions/14/behavior-changes-14#safer-dynamic-code-loading)). Flutter cannot load a second AOT Dart program from such a file anyway; the only route is the custom deferred-components path above.

### Separate plugin apps (Android only)

- Tasker plugins are ordinary Android apps. Tasker discovers them by intent filter, configures them through an `EditActivity`, and runs them through a broadcast receiver ([Tasker plugin docs](https://tasker.joaoapps.com/plugins.html)).
- On Android 11+, the host must declare the plugins' intent action in `<queries>` to see them ([Android package visibility](https://developer.android.com/training/package-visibility/declaring)).
- Plugins can be written in any language and ship as their own APK with their own Android permissions.
- Limits: Android only, no Flutter UI inside the host, and data passed as bundles of primitives.

## 2. How plugin ecosystems work

### Obsidian (required)

- **API surface:** a `Plugin` class with `onload()` and `onunload()`. Plugins must release their resources in `onunload` ([Anatomy of a plugin](https://docs.obsidian.md/Plugins/Getting+started/Anatomy+of+a+plugin)).
- **Settings:**
  - a `PluginSettingTab` registered with `addSettingTab` ([Settings](https://docs.obsidian.md/Plugins/User+interface/Settings));
  - values persisted with `loadData()` and `saveData()`;
  - since 1.13.0, rows can be declared with `getSettingDefinitions()` and the host saves them.
- **Manifest** ([Manifest reference](https://docs.obsidian.md/Reference/Manifest)):
  - `id` (immutable), `name`, `version` (semver), `minAppVersion`, `description`, `author`;
  - `isDesktopOnly` for plugins that use Node or Electron.
- **Versioning:** `versions.json` maps each plugin version to its `minAppVersion`, so an older app gets the newest compatible release ([Versions](https://docs.obsidian.md/Reference/Versions)).
- **Discovery and install:**
  - Plugins are GitHub repos. Each GitHub release attaches `main.js`, `manifest.json` and an optional `styles.css`, tagged with the version.
  - Authors register through the community directory, which reads the repo's manifest and runs automated review.
  - The app downloads the release assets ([Submit your plugin](https://docs.obsidian.md/Plugins/Releasing/Submit+your+plugin)).
  - The `obsidianmd/obsidian-releases` repo still hosts the `community-plugins.json` index ([obsidian-releases](https://github.com/obsidianmd/obsidian-releases)).
- **Enable/disable and permissions:**
  - Restricted mode is on by default and blocks all third-party code. Installed plugins stay in place but are ignored ([Plugin security](https://obsidian.md/help/plugin-security)).
  - There is **no sandbox**: plugins "inherit Obsidian's access levels" because Obsidian "cannot reliably restrict plugins".
  - Trust comes from automated scanning of every version, plus manual review of popular or flagged plugins ([Plugin security](https://obsidian.md/help/plugin-security)).
- **Community vs first-party:** core plugins ship in the app. Community plugins are opt-in, behind restricted mode.

### VS Code

- **Manifest** ([Extension manifest](https://code.visualstudio.com/api/references/extension-manifest)): `package.json` with:
  - `engines.vscode` (required compatibility range);
  - `activationEvents` (lazy load);
  - `contributes` (declarative contribution points);
  - `main`;
  - `capabilities.untrustedWorkspaces`.
- **Isolation:** extensions run in an Extension Host (local Node, web worker or remote), away from the UI. This is for stability and responsiveness, not a security sandbox ([Extension Host](https://code.visualstudio.com/api/advanced-topics/extension-host)).

### Home Assistant

- **Location:** custom integrations load from `<config>/custom_components/<domain>` and can override a built-in integration with the same domain. HA warns against overriding, because the override stops receiving updates ([File structure](https://developers.home-assistant.io/docs/creating_integration_file_structure)).
- **Manifest** ([Integration manifest](https://developers.home-assistant.io/docs/creating_integration_manifest)):
  - `domain`, `name`, `version` (required only for custom integrations);
  - `requirements` (pip packages installed before load);
  - `dependencies`, `config_flow` (UI settings flow), `iot_class`.
- **Community store:** HACS installs GitHub repos with this layout, from releases or the default branch ([HACS publish docs](https://hacs.xyz/docs/publish/integration/)).
- No sandbox: integrations are in-process Python.

### Logseq

- **Isolation:** each plugin runs in an iframe sandbox and talks to the host over postMessage (Postmate). A "shadow" mode runs plugins in the same origin ([LSPlugin.caller.ts](https://github.com/logseq/logseq/blob/master/libs/src/LSPlugin.caller.ts)).
- **API:** namespaced: `logseq.App`, `logseq.Editor`, `logseq.DB`, `logseq.UI` and others ([plugin docs](https://plugins-doc.logseq.com/)).
- **Marketplace:** authors open a PR adding `packages/<id>/manifest.json` (title, description, author, repo, …) to the marketplace repo.
- **Same-origin flag:** the `effect` flag ("sandbox is running under the same origin with host") gets stricter review ([logseq/marketplace](https://github.com/logseq/marketplace)).

### Joplin

- **Desktop isolation:** each plugin runs in a separate process (a hidden BrowserWindow). Every API call goes through a proxy that serializes it over IPC, and callbacks are replaced by string IDs ([plugin spec](https://joplinapp.org/help/dev/spec/plugins/)).
- **Mobile:** plugins also run isolated, using a `RemoteMessenger` over postMessage.
- **Manifest** ([manifest reference](https://joplinapp.org/help/api/references/plugin_manifest/)):
  - `manifest_version`, `version`, `app_min_version`;
  - `app_min_version_mobile`;
  - `platforms` (`desktop` and/or `mobile`);
  - categories, screenshots and icons.

Joplin is the closest analogue to a cross-platform app that runs community plugins on mobile.

### Tasker

Out-of-process plugin APKs over the intent protocol, discovered from installed apps (see section 1).

### Flutter apps with plugin systems

- Mangayomi (a Flutter manga and anime reader) runs community "sources" written in JavaScript or Dart ([README](https://github.com/kodjodevf/mangayomi), [extensions repo](https://github.com/kodjodevf/mangayomi-extensions)).
- Its `pubspec.yaml` bundles `flutter_qjs` (QuickJS) and `d4rt` (a Dart interpreter) ([pubspec](https://github.com/kodjodevf/mangayomi/blob/main/pubspec.yaml)).
- Sources are listed in JSON indexes served from GitHub Pages, which users add by URL.
- Plugins supply logic (scraping), while all UI is host-rendered. This is the proven interpreted pattern in Flutter today.
- The official extensions repo was archived in Dec 2025.

### Flutter's own compile-time pattern

Federated plugins split code into an app-facing package, a platform-interface package and implementation packages. Implementations are "endorsed" by being listed as dependencies ([Developing packages](https://docs.flutter.dev/packages-and-plugins/developing-packages)). Pub's `path` and `git` dependencies let an app pull a plugin package from a folder or repo at build time ([Dart dependencies](https://dart.dev/tools/pub/dependencies)).

## 3. What transfers to this app

Context: an Android and iOS Flutter app, distributed as open source APKs on GitHub, plugins include on-device AI, and the fork must stay easy to merge with upstream.

Patterns that transfer regardless of how code is loaded:

- **Manifest** with `id`, `name`, `version`, `minAppVersion`, `platforms`, and declared capabilities or permissions (Obsidian, Joplin, VS Code `engines`).
- **Lifecycle** `onload`/`onunload` with registered disposables, so disable works without a restart (Obsidian).
- **Declarative contributions** (VS Code `contributes`): plugins register into host extension points such as timer events, interval transitions and settings sections. They do not patch host widgets. This also keeps the upstream diff small: upstream files get a few hook calls, and everything else lives in fork-owned packages.
- **Host-rendered settings** from a schema (Obsidian 1.13 `getSettingDefinitions`, Logseq settings schema). Each plugin's data is stored namespaced by `id`.
- **Global kill switch** like Obsidian's restricted mode, plus per-plugin enable toggles.
- **Version gating** via `minAppVersion` (and `versions.json` if remote install ever exists).

Patterns that do not transfer, or not yet:

- **Unsandboxed downloaded JS** (Obsidian, Home Assistant). This is impossible for native-speed Dart, and an unsandboxed interpreter would break Apple 4.7.2 and 4.7.3 if iOS ever ships through the store.
- **Separate-process isolation** (VS Code, Joplin). It needs a JS runtime per plugin. The Flutter equivalent is one interpreter instance per plugin, with a serialized message API (the Joplin proxy model).
- **Runtime-downloaded native code.** On-device AI runtimes are native, so AI plugins must be compiled in under every option. Neither Shorebird ([FAQ](https://docs.shorebird.dev/code-push/faq/)) nor interpreters can add native code.

## 4. Viable architectures

| # | Architecture | How plugins arrive | iOS / store fit | Sandbox | Native code (AI) | Plugin UI | Upstream merge cost | Main risk |
|---|---|---|---|---|---|---|---|---|
| A | **Compiled-in Dart plugins behind a host interface.** Plugin packages in the repo, a registry, manifest, lifecycle, and per-plugin enable and settings. | Built into the APK. A new plugin means a new release. | Full: nothing downloaded | None (trusted, first-party) | Yes, any Flutter package | Full Flutter widgets in host extension points | Low: a few hook calls in upstream files, the rest in fork-owned packages | No third-party plugins without rebuilding |
| B | **A plus build-time selection.** Plugins as separate pub packages pulled by `path`/`git` deps; flavors or `--dart-define` choose the set. | At build time. Anyone can fork and build a custom APK. | Full | None | Yes | Full | Low | Community plugins need a build toolchain, so this is not end-user install |
| C | **A plus a JS or Dart-bytecode interpreter tier.** QuickJS or JavaScriptCore via `flutter_js`, or `dart_eval`. Plugins are scripts plus a manifest, downloaded from a GitHub-hosted index. | At runtime, from a URL or index (Obsidian/Mangayomi style) | Allowed for JS plug-ins under Apple 4.7 with consent and index rules; fine for Play (interpreter exception) and sideload | Yes: only the host API is exposed, with capability grants per manifest | No; scripts call host-provided AI services | Declarative only: host widgets from a schema or `rfw` | Medium: a host API, a bridge, an index, an install UI | API design and maintenance; package maturity (`flutter_js` unverified, `dart_eval` incomplete); 10 to 50 times slower logic |
| D | **A plus a WebAssembly tier** (`wasm_run_flutter`) | At runtime, as `.wasm` modules | Same as C, but "JS plug-ins" wording does not obviously cover Wasm | Strong (Wasm memory isolation) | No | Declarative only | Medium-high: a Rust FFI dependency and a toolchain for authors | Young package (0.2.x); harder authoring; interpreter-speed Wasmi on iOS |
| E | **Separate Android plugin APKs** (Tasker model): intents or bound services, discovery via `<queries>` | Users install another APK | No iOS equivalent | OS process isolation and per-app Android permissions | Yes, inside the plugin APK | None in the host (or the plugin's own activity) | Low in Dart, plus Kotlin glue | Android only; primitive data bundles; poor UX for timer-synchronous hooks |

Ruled out as plugin mechanisms: deferred components (Play-only, same-build code) and Shorebird (publisher patches, not third-party extensibility, no native code).

**Suggested path:** start with **A** (with **B**'s package layout, so each plugin is its own pub package). Make the host API serializable (plain data in, plain data out, events by name) so that **C** can be added later if community plugins become a goal. **E** is worth noting only if an Android-only automation integration is ever wanted.

## Open questions for the design ticket

- Which host extension points the first plugins need: timer lifecycle events, interval transitions, an audio and announcement channel, settings sections, the history/export pipeline.
- Where plugin packages live (`packages/` vs `aiext/`), recorded as an ADR per `aiext/README.md`.
- Whether iOS App Store distribution is ever a goal. This decides whether Apple 4.7 constraints shape tier C.
