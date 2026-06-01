# yandex-mapkit-sdk

[![skills.sh](https://skills.sh/b/ulugbekeshnazarov409/yandex-mapkit-sdk)](https://skills.sh/ulugbekeshnazarov409/yandex-mapkit-sdk)

An [agent skill](https://www.skills.sh) for the **Yandex MapKit Mobile SDK** (v4.x) —
native maps, geolocation, search, geocoding, routing, and turn-by-turn navigation on
**Android (Kotlin/Java), iOS (Swift), and React Native / Expo** (`react-native-yamap`).

It teaches the agent to treat a map as a **GIS system, not a UI component** — viewport
loading, server-side clustering, spatial indexing, and offline — and to design for
**100k users / 1M map objects / 10k concurrent sessions**, not a naive "render all markers"
implementation.

## Install

```bash
npx skills add ulugbekeshnazarov409/yandex-mapkit-sdk
```

Works with Claude Code, Cursor, Windsurf, and other skills-aware agents.

## What it covers

- **API key + variants** — Developer Dashboard, MAU pricing, `lite` vs `full` vs NaviKit
- **Android (Kotlin)** — Gradle, `MapKitFactory` init + lifecycle, `MapView`, camera,
  placemarks/polylines/polygons/circles, gestures, user-location layer, traffic, ProGuard
- **iOS (Swift)** — CocoaPods, `AppDelegate` init, `YMKMapView`, objects, listeners
- **React Native / Expo** — `react-native-yamap`, config plugin + dev build (NOT Expo Go),
  `<YaMap>` components, methods, AppDelegate wiring
- **Search / Suggest / Geocoder / Routing** — all four routers, forward + reverse geocoding,
  the "keep the Session alive" rule, NaviKit overview
- **Objects & styling** — icons/ViewProvider price markers, clustering, map types, traffic,
  JSON styling
- **Architecture at scale** — the GIS constitution: viewport fetch, zoom-band loading,
  PostGIS server clustering (`ST_ClusterDBSCAN`), the 7-layer model, caching, realtime
  tracking, perf budgets, security, and an 8-point pre-flight checklist
- **Troubleshooting & recipes** — init-order crashes, GC'd sessions, lifecycle leaks, plus
  copy-paste end-to-end tasks

## Structure

```
yandex-mapkit-sdk/
├── SKILL.md               # entry point: mindset, variants, key, mental model, golden rules
├── architecture.md        # READ FIRST at scale — the GIS engineering constitution
├── android.md             # Kotlin: full setup → map, objects, listeners, user location
├── ios.md                 # Swift: CocoaPods, AppDelegate, YMKMapView, objects
├── react-native.md        # react-native-yamap + Expo dev build
├── search-routing.md      # Search, Suggest, Geocoder (fwd/rev), routers, NaviKit
├── objects-styling.md     # objects, clustering, styling, map types, traffic
├── troubleshooting.md     # symptom → cause → fix
└── recipes.md             # copy-paste end-to-end tasks
```

## Companion skill

For **web / React / Next.js** Yandex maps (the JavaScript API v3 / `ymaps3`), use
[`yandex-maps-v3`](https://github.com/ulugbekeshnazarov409/yandex-maps-v3) — a different
SDK. This repo is **native mobile** only.
