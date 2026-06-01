---
name: yandex-mapkit-sdk
description: Build maps, geolocation, search, geocoding, routing and navigation with the Yandex MapKit Mobile SDK on Android (Kotlin/Java), iOS (Swift), and React Native / Expo (react-native-yamap). Use for ANY task involving Yandex Maps in a mobile app — showing a map, API key setup, MapView lifecycle, camera control, placemarks/markers, polylines/polygons/circles, marker clustering, user location layer & permissions, tap/camera listeners, the Search/Suggest/Geocoder API, Directions/routing (driving, pedestrian, masstransit, bicycle), turn-by-turn navigation (NaviKit), traffic/satellite layers, map styling, and fixing MapKit build/init/crash errors. Triggers — yandex map, MapKit, MapKitFactory, YMKMapView, react-native-yamap, YaMap, yandex geocoder, yandex routing, drivingRouter, searchManager, user location yandex, marker cluster yandex.
---

# Yandex MapKit Mobile SDK

Yandex's native maps SDK for **Android, iOS, and (via wrappers) Flutter / React Native**.
Renders vector maps, places markers/lines/polygons, shows the user's location, runs
search + geocoding, builds routes, and (with NaviKit) does turn-by-turn navigation.

This skill covers the **v4.x** SDK. Pick the file for your target after reading the
shared essentials below.

## Mindset — you are a GIS engineer, not a UI dev

When generating Yandex MapKit code, treat maps as **GIS systems, not UI components**.
Before writing code always think: viewport loading · clustering · spatial indexing ·
caching · realtime updates · memory · offline. **Never render large datasets directly.**
Design for **100,000 users / 1,000,000 map objects / 10,000 concurrent sessions**. Prefer
PostGIS spatial queries over client-side filtering; server-side clustering over client;
viewport-based loading over "fetch everything". Avoid naive implementations — if a request
implies "render all markers", apply `architecture.md` instead. Everything on a Yandex map
is a **MapObject** (Placemark / Polyline / Polygon / Circle / Cluster / Collection).

## Why Yandex (vs Google Maps)

Google Maps is **map-first**; Yandex MapKit is **GIS-first** — richer geometry, routing,
and address data across **Russia, Kazakhstan, Uzbekistan, Kyrgyzstan, Belarus, Armenia**,
where taxi / delivery / logistics / courier apps rely on it. Choose Yandex when the product
serves those regions or needs strong local geocoding, traffic, and routing.

## SDK variants — pick one

- **`lite`** — map display, traffic layer, `LocationManager`, `UserLocationLayer`. Smaller.
- **`full`** — lite **+** routing (driving / pedestrian / masstransit / bicycle), search,
  suggest, geocoding, panorama. Use `full` if you need search or routes.
- **NaviKit** — a separate SDK on top of MapKit for turn-by-turn voice navigation.

Package suffixes encode the variant: Android `…:maps.mobile:<ver>-lite` / `-full`,
iOS pods `YandexMapsMobile` (lite) vs the full pod.

## API key (required — do this first)

1. Open the Yandex **Developer Dashboard** (yandex.com/dev → MapKit Mobile SDK).
2. Log in, **Connect APIs → MapKit Mobile SDK**, fill project info, pick a plan.
3. Copy the key. Pricing is **MAU-based** (Free ≤ 25K MAU, then Basic / Advanced) — NOT
   per-request.
4. The key is set **once at app startup**, before any map is created:
   - Android: `MapKitFactory.setApiKey("KEY")` in `Application.onCreate()`.
   - iOS: `YMKMapKit.setApiKey("KEY")` in `application(_:didFinishLaunchingWithOptions:)`.
   - RN: `YaMap.init("KEY")` in your root component / index.

> Keep the key out of source control. Android: `local.properties` / BuildConfig /
> Gradle secrets. iOS: xcconfig / Info.plist + build settings. RN: `app.config.ts` extra
> + `process.env`.

## The mental model (same on every platform)

1. **Lifecycle** — call `MapKitFactory.initialize(context)` (Android) / instantiate
   `YMKMapKit.sharedInstance()` (iOS) once, then start/stop MapKit and the MapView with
   the screen lifecycle (`onStart/onStop`). Forgetting this leaks native resources.
2. **Map + camera** — the map shows a `CameraPosition(target: Point(lat, lon), zoom,
   azimuth, tilt)`. Move instantly or `with Animation(SMOOTH, duration)`.
3. **Map objects live in collections** — get `map.mapObjects` (a `MapObjectCollection`)
   and `addPlacemark` / `addPolyline` / `addPolygon` / `addCircle`. Keep references to
   update/remove them. Clustering uses a `ClusterizedPlacemarkCollection`.
4. **Everything async is a listener/session** — search, routing, suggest, and geocoding
   return a **Session** object; you pass a callback and must **keep the session alive**
   (store it in a field) or it's GC'd and the callback never fires. This is the #1 bug.
5. **User location** is a layer: `mapKit.createUserLocationLayer(mapWindow)` +
   runtime location permission.

## Install & first map (summary — full steps in the platform file)

- **Android** → `android.md`: Maven Central dep, `MapKitFactory.setApiKey/initialize`,
  `<MapView>` in XML, lifecycle, camera, objects, listeners, user location, clustering.
- **iOS** → `ios.md`: CocoaPods, `AppDelegate` init, `YMKMapView`, camera, objects, listeners.
- **React Native / Expo** → `react-native.md`: `react-native-yamap`, Expo **config plugin**
  (NOT Expo Go — needs a dev build), `<YaMap>`, markers, methods, AppDelegate wiring.

## Capabilities — deep reference

- **`architecture.md`** — **READ FIRST for any production / at-scale work.** The map
  engineering constitution: assume 100k users / 1M properties / 10k concurrent sessions.
  Viewport-only fetching, zoom-band loading, server-side PostGIS clustering, the 7-layer
  architecture, WGS84 coordinate rules, caching levels, realtime tracking cadence,
  performance budgets, security, and the **8-point AI pre-flight checklist** to answer
  BEFORE generating any map code (never ship a naive "render all markers" map).
- **`search-routing.md`** — Search API, Suggest, Geocoder (forward + reverse), and all four
  routers (driving/pedestrian/masstransit/bicycle) with response parsing + drawing routes.
  Plus a NaviKit overview.
- **`objects-styling.md`** — every map object type, icons/ViewProvider, z-index, opacity,
  dragging, **clustering** in depth, user-location styling, map types (vector / satellite /
  hybrid), traffic layer, and custom JSON map styling.
- **`troubleshooting.md`** — init-order crashes, missing/invalid key, lifecycle leaks,
  ProGuard, Expo Go incompatibility, version mismatches, permissions, quota/billing errors.
- **`recipes.md`** — copy-paste end-to-end tasks: map on a city, tap-to-drop marker, route
  between two points, search nearby + list, follow user location, cluster N markers.

## Golden rules

1. Set the API key + `initialize` **before** inflating/creating any MapView. Wrong order = crash.
2. Bind MapKit + MapView to `onStart`/`onStop` (Android) / view appearance (iOS). Don't leak.
3. **Store every Session** (search/route/suggest/geocode) in a field — else no callback.
4. `Point` is `(latitude, longitude)` — Yandex orders **lat, lon** (not lon, lat).
5. Request location permission before enabling the user-location layer.
6. React Native: this is a native module → **dev client / prebuild required**, never Expo Go.
7. Match the SDK variant to need: add `-full` (Android) / full pod (iOS) only if you use
   search or routing; otherwise `-lite` is lighter.

Read the relevant platform file before writing code, then `search-routing.md` /
`objects-styling.md` for the specific feature, and keep these rules.
