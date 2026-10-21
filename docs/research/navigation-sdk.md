# Turn-by-turn navigation SDK options for React Native + MapLibre

Resolves [issue #4](https://github.com/AarjanM/scooterplanner/issues/4) (part of #1).
Researched 2026-10-21 against primary sources only (GitHub repos, READMEs, issues,
releases; Expo, Valhalla and Android docs; npm and pub.dev registries). Every
"last release" date is the date observed on that day. Anything not confirmed from a
primary source is marked **UNVERIFIED**.

Vocabulary follows `GLOSSARY.md`: *Turn-by-turn navigation* is live guidance along a
*Route plan* with the rider's position tracked and instructions spoken; a *Route plan*
is a *Legal route* for the rider's *Vehicle class* (Fiets, Snorfiets, Bromfiets).

## TL;DR

**Stay on React Native + Expo.** A usable, permissively licensed navigation layer
exists for exactly this stack: **Ferrostar** (Rust core, BSD-3-Clause) ships a
React Native SDK in-tree, built on `@maplibre/maplibre-react-native`, with an Expo
example app using `expo-location` and `expo-speech`. Its React Native packages are
**not yet published to npm** (pre-alpha, tracking issue open), so v1 must build or
vendor them from a pinned tag; that is the single biggest risk and should be the
first spike. The MapLibre Navigation SDKs (Android/iOS, Mapbox v0.x forks) have no
React Native or Flutter bindings at all, and the iOS one points new projects at
Ferrostar itself. Flutter is **not** clearly better: it has excellent MapLibre map
bindings but no open navigation layer (Ferrostar's Flutter work has not started,
the only Flutter nav plugins are vendor-locked to VietMap API keys), so Flutter
would mean hand-rolling the whole navigation engine in Dart.

## 1. Options table

| Component | What it is | Platforms | Expo compatibility | Maintenance (observed 2026-10-21) | Licence |
|---|---|---|---|---|---|
| [maplibre/maplibre-navigation-android](https://github.com/maplibre/maplibre-navigation-android) | Fork of Mapbox Navigation SDK v0.19; being converted to Kotlin Multiplatform (`navigation-core` targets Android, iOS, JVM, JS, wasmJS; `navigation-ui-android` Android only) | Android native (KMP core also iOS/JVM/JS) | None. No config plugin, no RN bridge. Would need a hand-written Expo Module wrapping Kotlin, plus a separate iOS story | Active: 5.0.1 released 2026-10-19, 5.0.0 2026-08-26, 213 stars, 22 open issues. Zero issues about React Native; one closed Flutter-porting issue [#154](https://github.com/maplibre/maplibre-navigation-android/issues/154) | MIT, "No Telemetry" |
| [maplibre/maplibre-navigation-ios](https://github.com/maplibre/maplibre-navigation-ios) | Fork of Mapbox Navigation SDK v0.21 (UIKit `NavigationViewController`) | iOS native, SPM only ([CocoaPods support #34](https://github.com/maplibre/maplibre-navigation-ios/issues/34) open since 2024) | None. SPM-only is awkward in Expo (see Ferrostar [#947](https://github.com/stadiamaps/ferrostar/issues/947) for the same SPM-vs-CocoaPods problem) | Slower: 4.1.1 released 2026-08-03, 4.1.0 2025-08-08, 67 stars, 27 open issues. README says: "For new projects, you may want to check out Ferrostar". Open voice bug [#121](https://github.com/maplibre/maplibre-navigation-ios/issues/121) (unDuckAudio hangs) | MIT + ISC ([LICENSE.md](https://github.com/maplibre/maplibre-navigation-ios/blob/main/LICENSE.md)) |
| [@maplibre/maplibre-react-native](https://github.com/maplibre/maplibre-react-native) | MapLibre Native map view for RN | Android (MapLibre Native 13.6.1), iOS (6.31.0) | Yes: config plugin `"@maplibre/maplibre-react-native"` in `app.json`; iOS plugin adds `$MLRN.post_install(installer)`; **Expo Go not supported**, development build required ([Expo setup](https://maplibre.org/maplibre-react-native/docs/setup/expo/)). RN >= 0.80; **New Architecture only from v11** ([getting started](https://maplibre.org/maplibre-react-native/docs/setup/getting-started/)) | Very active: v11.5.0 released 2026-10-04, 670 stars, 165,643 npm downloads/week | MIT |
| [stadiamaps/ferrostar](https://github.com/stadiamaps/ferrostar) | "FOSS navigation SDK built from the ground up". Rust core (navigation state machine, snapping, Valhalla/OSRM API adapters) + Swift/Kotlin/Web/React Native layers ([architecture](https://stadiamaps.github.io/ferrostar/architecture.html)) | iOS >= 15, Android API >= 25 ([platform targets](https://github.com/stadiamaps/ferrostar/blob/main/guide/src/platform-support-targets.md)); Web; **React Native via three in-tree packages** | Yes by design: the [RN guide](https://github.com/stadiamaps/ferrostar/blob/main/guide/src/react-native-getting-started.md) says an Expo app must use a development build and `npx expo prebuild`; the [example app](https://github.com/stadiamaps/ferrostar/blob/main/react-native/example/package.json) is Expo (`expo ^55`, `expo-location`, `expo-speech`, `@maplibre/maplibre-react-native 11.3.10`) | Very active: 0.58.0 released 2026-10-13 (releases every 1-3 weeks), 443 stars, 149 open issues. RN layer rewritten in [PR #883](https://github.com/stadiamaps/ferrostar/pull/883) merged 2026-09-15 (speech, foreground service, landscape, map control, docs). **Guide intro still calls RN "pre-alpha"**; [#961 "Publish React Native package to NPM"](https://github.com/stadiamaps/ferrostar/issues/961) is open and all three package names return 404 on the npm registry | BSD-3-Clause ([LICENSE.txt](https://github.com/stadiamaps/ferrostar/blob/main/LICENSE.txt); package.json `license: BSD-3-Clause`) |
| [bjtrounson/expo-ferrostar](https://github.com/bjtrounson/expo-ferrostar) | Earlier community Expo Module for Ferrostar | Android only (iOS unchecked in README) | Expo Module | Dead: last push 2024-11-28, "not ready for production", not on npm. Author went on to write Ferrostar's in-tree RN layer (PR #883) | none declared |
| [@vietmap/vietmap-react-native-navigation](https://github.com/vietmap-company/vietmap-react-native-navigation) | VietMap's RN wrapper around the MapLibre navigation forks | Android, iOS, New Architecture required | Bare RN manifest edits; no config plugin documented | Last push 2026-06-15, 1 star | BSD-3-Clause, **but requires a VietMap API key for tiles and routing** (README checklist item 1) -> unusable with self-hosted tiles |
| [maplibre/flutter-maplibre-gl](https://github.com/maplibre/flutter-maplibre-gl) (`maplibre_gl`) | Flutter MapLibre map | Android, iOS, Web | n/a | v0.27.1 released 2026-09-10, 376 stars, 130,020 pub downloads/30d, 160/160 pub points | BSD-style ([LICENSE](https://github.com/maplibre/flutter-maplibre-gl/blob/main/LICENSE)) |
| [josxha/flutter-maplibre](https://github.com/josxha/flutter-maplibre) (`maplibre`) | Newer Flutter MapLibre binding | Android, iOS, Web, desktop | n/a | v0.3.6 released 2026-08-25, 161 stars, 15,262 pub downloads/30d | BSD-style |
| [vietmap_flutter_navigation](https://pub.dev/packages/vietmap_flutter_navigation) | Flutter nav plugin on MapLibre nav forks | Android, iOS | n/a | 4.2.0 released 2026-04-22, 655 pub downloads/30d | BSD-3-Clause, **hard-wired to `VietMapURL` / `VietMapAPIBaseURL` + API key** |
| [thanhdt-vietmap/maplibre-navigation-flutter](https://github.com/thanhdt-vietmap/maplibre-navigation-flutter) | Attempted generic Flutter port | Android only | n/a | README: "in development... only support for android". 0 stars, last push 2025-04-25, not on pub.dev | BSD-3-Clause |
| `flutter_maplibre_navigation` (named in the ticket) | — | — | — | **Does not exist on pub.dev** (404) | — |

## 2. `@maplibre/maplibre-react-native` for rendering

- Expo: supported through its config plugin; development build mandatory, Expo Go
  excluded ([docs](https://maplibre.org/maplibre-react-native/docs/setup/expo/)).
  Matches Expo's own guidance that any package with native code needs CNG/prebuild
  and a development build ([Expo CNG docs](https://docs.expo.dev/workflow/continuous-native-generation/)).
- Navigation-relevant API: `Camera` prop `trackUserLocation` with `"course"` mode
  (centre on user, bearing = direction of travel) ([Camera docs](https://maplibre.org/maplibre-react-native/docs/components/camera/));
  `UserLocation` puck with `heading`, `accuracy`, `minDisplacement`
  ([source doc](https://github.com/maplibre/maplibre-react-native/blob/main/docs/content/components/annotations/user-location.md)).
- Offline-readiness for later: `OfflineManager.createPack` (style URL + bbox + zoom
  range), `getPacks`, `invalidatePack`, `mergeOfflineRegions(path)` to sideload a
  prebuilt tile database, ambient cache controls, `setTileCountLimit`
  ([OfflineManager docs](https://maplibre.org/maplibre-react-native/docs/modules/offline-manager/)).
  Progress events are throttled over the bridge. Offline *routing* is a separate
  problem (see open questions).
- Constraint to carry into the architecture: v11 is New Architecture only, so the
  Expo SDK / RN version must be New-Arch (Expo SDK 55+ in Ferrostar's example).

## 3. Ferrostar: what it gives a TypeScript app, and what remains

Ferrostar's Rust core owns the hard parts ([architecture](https://stadiamaps.github.io/ferrostar/architecture.html),
[Configuring Navigation Behavior](https://github.com/stadiamaps/ferrostar/blob/main/guide/src/configuring-the-navigation-controller.md)):

- Snapping the user location to the route line (`snappedLocationCourseFiltering: CourseFiltering.SnapToRoute`).
- Step advance: `stepAdvanceDistanceEntryAndExit(30, 5, 32)`, `stepAdvanceDistanceToEndOfStep(10, 32)`, custom conditions.
- Off-route detection: `RouteDeviationTracking.StaticThreshold({minimumHorizontalAccuracy, maxAcceptableDeviation})` or a custom `RouteDeviationDetector`.
- Route request/response adapters for Valhalla and OSRM-compatible JSON (`WellKnownRouteProvider.Valhalla`, or a custom `RouteProvider`).
- Visual/spoken instruction state per step (the RN example wires `expo-speech` through the `SpeechEngine` interface, [`core/src/SpeechEngine.ts`](https://github.com/stadiamaps/ferrostar/blob/main/react-native/core/src/SpeechEngine.ts)).

The RN layer is three packages, all at 0.58.0 in-tree
([RN guide](https://github.com/stadiamaps/ferrostar/blob/main/guide/src/react-native-getting-started.md)):
`@stadiamaps/ferrostar-uniffi-react-native` (generated Rust bindings),
`@stadiamaps/ferrostar-core-react-native` (`FerrostarProvider`, `LocationProvider`,
`ForegroundService`, `SpeechEngine`, `RouteProvider`), and
`@stadiamaps/ferrostar-maplibre-react-native` (navigation UI on `@maplibre/maplibre-react-native` + `react-native-svg`).

Still hand-rolled by ScooterPlanner even with Ferrostar:

| Concern | Who does it | Notes |
|---|---|---|
| Location feed | App: `expo-location` -> `ManualLocationProvider.updateLocation()` | Ferrostar "does not request location permission or select a React Native location library for you" |
| Foreground/background keep-alive | App: `ForegroundService` adapter around `expo-location` `startLocationUpdatesAsync(... foregroundService)` | Adapter "coordinates lifecycle events; it does not itself grant background execution" |
| Voice | App: `SpeechEngine` on `expo-speech` | Example app does exactly this |
| Recalculation policy | App | "The Rust core does not expose any primitives for handling recalculation" - app decides when being off route triggers a new *Legal route* request |
| Instruction text in Dutch | Routing engine | Valhalla emits `voiceInstructions`/`bannerInstructions` per step when `format=osrm` and the flags are set ([Valhalla route API](https://valhalla.github.io/valhalla/api/route/api-reference/)); language is a server-side concern |
| UI theming / Vehicle class picker | App | Ferrostar UI components can be composed individually or replaced |

## 4. If no SDK fit: honest hand-rolled TypeScript estimate

Only needed as plan B (Ferrostar RN packaging turns out unworkable). Building a
navigation engine in TS on top of raw route legs from Valhalla (`format=osrm`,
`shape_format=polyline6`, `voice_instructions=true`, `banner_instructions=true`):

| Piece | Library help | Hand-rolled work |
|---|---|---|
| Decode geometry | [`@mapbox/polyline`](https://github.com/mapbox/polyline) 1.2.1 (2023-09-14, BSD-3-Clause, 338k dl/wk) | trivial |
| Snapping / map matching | [`@turf/nearest-point-on-line`](https://www.npmjs.com/package/@turf/nearest-point-on-line) 7.4.0 (2026-08-03, MIT, 1.6M dl/wk) gives the projected point and `location` (distance along line) | Medium: windowed search along current step, speed/heading plausibility filter, GPS-accuracy gating, hysteresis so the puck does not jump between parallel Bromfietspad and Rijbaan (this is the hard case for mopeds) |
| Off-route detection | [`@turf/point-to-line-distance`](https://www.npmjs.com/package/@turf/point-to-line-distance) 7.4.0 | Small-medium: threshold scaled by accuracy, N consecutive misses, debounce before reroute |
| Step advance / instruction timing | none | Medium: distance-to-manoeuvre from snapped `location`, announce windows (e.g. 400 m / 100 m / now) scaled by speed (25 vs 45 km/h), "then" chaining, arrival handling |
| Instruction text | Server-side Valhalla narrative; fallback [`osrm-text-instructions`](https://github.com/Project-OSRM/osrm-text-instructions.js) 0.15.0 (npm 2024-03-03, repo pushed 2026-10-20, BSD-2-Clause) | small if server emits Dutch text |
| Voice | [`expo-speech`](https://docs.expo.dev/versions/latest/sdk/speech/) 58.0.3 (2026-10-01, MIT, 399k dl/wk): `speak` queues utterances, `onDone`, `useApplicationAudioSession` (iOS) for ducking | Small, but see unverified items on background audio |
| Camera follow | `Camera trackUserLocation="course"` | small |
| Simulated location for tests | none | small |

Rough size: a few weeks of focused work for a usable engine plus ongoing tuning; the
snapping/off-route logic is where most bugs will live. Ferrostar already has this
tuned and tested in Rust ([e.g. #875 deviation fix](https://github.com/stadiamaps/ferrostar/issues/875)),
which is the argument for not hand-rolling.

## 5. The Flutter picture

- Maps: strong. `maplibre_gl` 0.27.1 (2026-09-10) and `maplibre` 0.3.6 (2026-08-25),
  both BSD-licensed and healthy (above).
- Navigation: **nothing reusable and open.**
  - Ferrostar Flutter: [#16 Dart binding generation](https://github.com/stadiamaps/ferrostar/issues/16)
    and [#106 Frontend components](https://github.com/stadiamaps/ferrostar/issues/106)
    open since 2024 with one comment each; the guide lists Flutter as not supported.
  - `vietmap_flutter_navigation` 4.2.0 is bound to VietMap's API key and base URL.
  - `thanhdt-vietmap/maplibre-navigation-flutter`: Android only, abandoned 2025-04.
  - `graphhooper_route_navigation` 0.0.9 (2025-06-16): draws GraphHopper routes on a
    Mapbox map; no guidance engine.
  - `flutter_maplibre_navigation`: does not exist on pub.dev.
- So Flutter would mean the full plan-B engine from section 4, in Dart, with
  [`turf`](https://pub.dev/packages/turf) 0.0.14 (2026-09-24),
  [`geolocator`](https://pub.dev/packages/geolocator) 14.1.2 (2026-10-19) and
  [`flutter_tts`](https://pub.dev/packages/flutter_tts) 4.2.5 (2026-01-05). Wrapping
  the native MapLibre nav SDKs via platform views is the other route and hit
  packaging trouble even for VietMap ([#154](https://github.com/maplibre/maplibre-navigation-android/issues/154)).
- Verdict: Flutter is not clearly better; for Turn-by-turn navigation it is clearly
  worse today.

## 6. Background location and foreground-service requirements (Expo)

Source: [expo-location docs, SDK 58](https://docs.expo.dev/versions/latest/sdk/location/)
(package 58.0.18 published 2026-10-19), [expo-task-manager](https://docs.expo.dev/versions/latest/sdk/task-manager/)
(58.0.12, 2026-10-15), [Android foreground service types](https://developer.android.com/develop/background-work/services/fgs/service-types).

Android:
- Config plugin: `isAndroidForegroundServiceEnabled` adds `FOREGROUND_SERVICE` and
  `FOREGROUND_SERVICE_LOCATION` (the latter mandatory on Android 14 / API 34);
  `androidForegroundServiceIcon` (96x96 white PNG). Set
  `isAndroidBackgroundLocationEnabled` only if `ACCESS_BACKGROUND_LOCATION` is truly
  needed: it triggers **Google Play review**.
- Runtime: `Location.startLocationUpdatesAsync(taskName, { foregroundService: { notificationTitle, notificationBody, notificationColor?, killServiceOnDestroy? } })`
  with a `TaskManager.defineTask` at module scope. Android requires fine/coarse
  location granted before the `location`-type service starts, and a `location`
  foreground service cannot be started from the background without
  `ACCESS_BACKGROUND_LOCATION`. For riding with the screen on or the app recently
  backgrounded, a foreground service with while-in-use permission should suffice
  (that is also how Ferrostar's native Android layer does it, adding
  `POST_NOTIFICATIONS` for API 33+ ([Ferrostar Android FGS guide](https://github.com/stadiamaps/ferrostar/blob/main/guide/src/android-foreground-service.md))).
- A terminated app is not restarted on location events.

iOS:
- Config plugin: `isIosBackgroundLocationEnabled` adds `location` to `UIBackgroundModes`.
- Expo docs state background updates need `requestBackgroundPermissionsAsync`
  ("Always"); an "Allow Once" grant makes that silently fail. **UNVERIFIED** (Apple's
  page did not render for fetching): Apple normally allows continued updates after
  backgrounding with When-In-Use + `location` background mode when updates were
  started in the foreground; whether Expo's implementation needs Always for that
  case must be tested on device.
- iOS-only task options: `activityType`, `pausesUpdatesAutomatically`,
  `showsBackgroundLocationIndicator`.

Both: none of this works in Expo Go; development builds only. Voice: `expo-speech`
docs do not state whether speech continues in the background or how it ducks other
audio on Android (**UNVERIFIED**, needs a device test); on iOS no sound in silent mode.

## 7. Recommendation

**Stay with React Native + Expo + TypeScript.** Adopt Ferrostar as the navigation
layer, with the plan-B TypeScript engine (section 4) as the documented fallback if
the packaging spike fails.

Component list (React Native):

| Layer | Component | Version observed |
|---|---|---|
| Framework | Expo SDK 55+ (New Architecture), development builds via `expo prebuild` | Ferrostar example: `expo ^55`, RN 0.83 |
| Map | `@maplibre/maplibre-react-native` + config plugin, self-hosted style/tiles; `OfflineManager` later | 11.5.0 |
| Navigation core | `@stadiamaps/ferrostar-uniffi-react-native`, `@stadiamaps/ferrostar-core-react-native` | 0.58.0 (from git tag until npm publish) |
| Navigation UI | `@stadiamaps/ferrostar-maplibre-react-native` + `react-native-svg`, or own components on the core | 0.58.0 |
| Location | `expo-location` + `expo-task-manager`, foreground service on Android, `location` background mode on iOS | 58.0.18 / 58.0.12 |
| Voice | `expo-speech` via Ferrostar `SpeechEngine` | 58.0.3 |
| Routing | Self-hosted Valhalla, `format=osrm`, `voice_instructions`, `banner_instructions`; `motor_scooter` costing (`top_speed` 20-120, default 45) as a starting point only; `bicycle` for Fiets | — |
| Geo utils | `@turf/*` for any app-side distance work | 7.4.0 |

If the team nevertheless flips to Flutter: `maplibre_gl` 0.27.1, `geolocator` 14.1.2,
`flutter_tts` 4.2.5, `turf` 0.0.14, and a from-scratch Dart navigation engine
(snapping, deviation, step advance, announce timing) plus native foreground-service
plumbing. Expect that engine to be the long pole.

## 8. Unverified items and new questions

- **UNVERIFIED**: that Ferrostar RN 0.58.0 builds inside an Expo SDK 55/58 dev build
  from source today (requires bun >= 1.3.13 and the uniffi toolchain per PR #883;
  [#947](https://github.com/stadiamaps/ferrostar/issues/947) shows an Expo 57
  integrator needing vendored XCFrameworks and CocoaPods). Spike needed.
- **UNVERIFIED**: iOS When-In-Use vs Always for Expo background updates; Android
  `expo-speech` ducking/background behaviour.
- **UNVERIFIED**: `react-native-background-geolocation` 5.8.0 (MIT on npm) is
  commonly cited as needing a paid Android licence; not checked here and not needed
  if `expo-location`'s foreground service suffices.
- Valhalla `motor_scooter` is a generic moped profile; it does not encode the Dutch
  G11/G12a/G13 distinctions or Rijbaan zones. Legal-route costing is a routing
  ticket, not a navigation-SDK one.
- Rerouting policy (when an off-route rider gets a fresh Legal route) is left to the
  app by Ferrostar and needs its own design.
