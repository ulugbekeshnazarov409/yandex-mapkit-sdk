# Troubleshooting — when the Yandex map misbehaves

Symptom → cause → fix, ordered roughly by how often each one bites. Most map bugs are
**init order**, a **GC'd Session**, or a **lifecycle** mistake — check those first.

---

## Map is blank / crashes at launch

**Cause:** the API key was never set, or `setApiKey` / `initialize` ran **after** the
MapView was created. The key + init must happen **before** any map exists.

**Fix — correct order per platform:**

- **Android** — set the key in `Application.onCreate()`, call `initialize(this)` at the
  very top of the Activity's `onCreate`, **then** inflate/find the `MapView`:
  ```kotlin
  // App.onCreate()
  MapKitFactory.setApiKey(BuildConfig.YANDEX_MAPKIT_KEY)  // 1. key first
  // MainActivity.onCreate()
  MapKitFactory.initialize(this)                          // 2. before the MapView
  super.onCreate(savedInstanceState)
  setContentView(R.layout.activity_main)
  mapView = findViewById(R.id.mapview)                    // 3. now create the view
  ```
- **iOS** — in `application(_:didFinishLaunchingWithOptions:)`, set the key, **then**
  instantiate MapKit, before any `YMKMapView` loads:
  ```swift
  YMKMapKit.setApiKey("YOUR_API_KEY")   // 1.
  _ = YMKMapKit.sharedInstance()        // 2. instantiate before any map
  ```
- **RN** — `YaMap.init(key)` once at app start (index/root), **and** set the key natively
  in `AppDelegate` for iOS (see below). JS init alone is not enough on iOS.

---

## "MapKit not initialized" / native crash on first map call

**Cause:** `MapKitFactory.initialize(context)` is missing or runs too late (Android); or
`YMKMapKit.sharedInstance()` was never instantiated (iOS). Setting the key is **not** the
same as initializing.

**Fix:**
- Android: `MapKitFactory.initialize(this)` must run **before** `mapView.mapWindow.map`
  is touched — put it first in `onCreate`, before `setContentView`.
- iOS: call `_ = YMKMapKit.sharedInstance()` in `AppDelegate` after `setApiKey`. Without
  the instantiation the SDK stays uninitialized even though the key is set.

---

## Search / route / suggest / geocode callback never fires

**The classic bug.** The `Session` (search/driving/suggest) was a local variable, went out
of scope, and got garbage-collected — so the in-flight request was cancelled and your
listener never runs. No error is thrown; it's just silent.

**Fix — store the Session in a field/property (not a local):**
```kotlin
private var searchSession: Session? = null          // field — survives the call
private var drivingSession: DrivingSession? = null
private var suggestSession: SuggestSession = searchManager.createSuggestSession()

searchSession = searchManager.submit(query, geometry, SearchOptions(), listener) // assign to the field
```
```swift
var searchSession: YMKSearchSession?                // property, not a local
searchSession = searchManager.submit(withText: …) { response, error in … }
```
Same rule for `DrivingSession`, `SuggestSession`, geocode submits. Cancel in-flight work
with `searchSession?.cancel()` / `suggestSession.reset()` on new input — but always keep
the reference. (See `search-routing.md`.)

---

## Lifecycle leaks / black map after returning to the screen

**Cause (Android):** you forgot to bind MapKit **and** the MapView to `onStart`/`onStop`.
Native resources leak; the second time the screen appears the map renders black.

**Fix:**
```kotlin
override fun onStart() {
    super.onStart()
    MapKitFactory.getInstance().onStart()   // both — MapKit AND the view
    mapView.onStart()
}
override fun onStop() {
    mapView.onStop()
    MapKitFactory.getInstance().onStop()
    super.onStop()
}
```
iOS manages its own lifecycle (no `onStart/onStop`), but still release the
controller/map normally. In RN, the wrapper handles this; if you mount/unmount the map
manually, memoize it (`React.memo`) so parent re-renders don't churn it.

---

## Coordinates wrong / markers land in the ocean

**Cause:** swapped latitude and longitude. Native MapKit is **`Point(latitude, longitude)`**
— lat first. The JS API v3 (`ymaps3`) uses `[lng, lat]` — the opposite order. Copying a
coordinate from a web example without flipping puts your marker off the coast of Africa
(0,0) or in the wrong hemisphere.

**Fix:** native is always lat-first (`Point(41.31, 69.28)` = Tashkent). Validate every
coordinate that crosses a boundary: `lat ∈ [-90, 90]`, `lon ∈ [-180, 180]`; reject
anything outside. (Coordinate discipline: `architecture.md`.)

---

## User location not showing

**Cause:** runtime location permission was not granted **before** you created the user
layer, or (iOS) the `Info.plist` usage string is missing.

**Fix:**
- Request `ACCESS_FINE_LOCATION` at runtime **first**, then call
  `MapKitFactory.getInstance().createUserLocationLayer(mapView.mapWindow)` and set
  `isVisible = true`.
- iOS: add `NSLocationWhenInUseUsageDescription` to `Info.plist` and request
  `CLLocationManager` authorization before `createUserLocationLayer`.
- RN/Expo: declare the `expo-location` permission strings in `app.config.ts` and grant at
  runtime before `showUserPosition`. (Setup in `react-native.md`.)

---

## Expo Go shows nothing / red screen

**Cause:** `react-native-yamap` is a **native module**. Expo Go ships a fixed native
runtime and cannot load it, so the map never appears (or throws a red-screen "native
module not found").

**Fix:** build a **dev client** — `npx expo prebuild` then
`eas build --profile development`, run with `npx expo start --dev-client`. Expo Go is
unsupported, full stop. (See `react-native.md` and project memory: native rebuild gate.)

---

## iOS blank map in React Native

**Cause:** the key was set only in JS (`YaMap.init`) but **not** natively in `AppDelegate`.
On iOS the native side needs `setApiKey` too.

**Fix:** add to `AppDelegate.mm` inside `didFinishLaunchingWithOptions`:
```objc
#import <YandexMapsMobile/YMKMapKitFactory.h>
[YMKMapKit setApiKey:@"YOUR_API_KEY"];
```
With Expo, drive this from the config plugin so prebuild patches `AppDelegate` — don't
hand-edit `ios/` permanently. (See `react-native.md`.)

---

## Android build fails resolving `com.yandex.android:maps.mobile`

**Cause:** Maven Central is not in the project's repositories, so Gradle can't find the
dependency.

**Fix:** add `mavenCentral()` to the repositories block:
```gradle
repositories {
    mavenCentral()
    google()
}
```
In RN/Expo this is handled by the config plugin; if a custom/local plugin is used, make
sure it adds the Maven repo.

---

## Version mismatch — lite vs full (`ClassNotFoundException`)

**Cause:** you're calling search / suggest / geocoding / routing on the **`-lite`** variant.
Lite ships only map display, traffic, location, and `UserLocationLayer` — the
`SearchManager` / `DrivingRouter` classes aren't in it, so you get `ClassNotFound` /
`NoClassDefFound` at runtime.

**Fix:** depend on the **`-full`** variant (Android
`com.yandex.android:maps.mobile:<ver>-full`, full iOS pod). Only stay on `-lite` if you
truly need nothing beyond map + traffic + location. (Variant matrix: `SKILL.md`.)

---

## Too many markers = jank / OOM

**Cause:** rendering every object instead of viewport-only. 10k placemarks on screen =
dropped frames and out-of-memory.

**Fix:** render **viewport objects only** (drive fetch from the camera-idle listener,
debounced), and above ~1000 objects switch to **server-side clustering** (PostGIS) — the
map draws only the returned cluster bubbles plus the few real markers. Reuse marker images
/ `ViewProvider`; don't re-decode per marker. Full rules in `architecture.md`.

---

## ProGuard / R8 crash in release builds only

**Cause:** R8 stripped/obfuscated MapKit reflection targets or your `userData` model
classes, so a release build crashes where debug works.

**Fix:** MapKit ships consumer rules, so usually nothing is needed — but if you hit
reflection/native crashes in release, add keep rules:
```proguard
-keep class com.yandex.** { *; }
# keep your model classes attached to placemarks as userData:
-keep class com.yourapp.model.** { *; }
```

---

## Quota / billing / 403

**Cause:** the API key is invalid, restricted, or your MAU plan is exceeded. MapKit
pricing is **MAU-based** (Free ≤ 25K MAU, then Basic / Advanced), not per-request.

**Fix:** verify the key in the Yandex **Developer Dashboard** (MapKit Mobile SDK), confirm
it's the right project/key, and check the current MAU usage against your plan. Regenerate
the key if it was leaked or rotated. (Key setup: `SKILL.md`.)

---

## Map labels / UI in the wrong language

**Cause:** locale was not set, or `setLocale` ran **after** `initialize`.

**Fix:** call `MapKitFactory.setLocale("ru_RU")` (Android) / `YMKMapKit.setLocale("ru_RU")`
(iOS) **before** `initialize` / `sharedInstance()`, alongside `setApiKey`. Setting it later
has no effect on the already-initialized engine.

---

## Quick self-check when the map doesn't work

1. **Key + init before the view?** `setApiKey` → `initialize`/`sharedInstance()` → create
   MapView — in that order, before anything touches the map.
2. **Did you keep the Session?** Search/route/suggest/geocode Session stored in a field, not
   a local. (No callback = GC'd Session.)
3. **Lifecycle bound?** Android `MapKitFactory.getInstance().onStart()/onStop()` **and**
   `mapView.onStart()/onStop()`.
4. **Coordinates lat/lon, in range?** Native `Point(lat, lon)`; not `[lng, lat]`;
   `lat ∈ [-90,90]`, `lon ∈ [-180,180]`.
5. **Right variant + repo?** `-full` for search/routing; `mavenCentral()` present (Android);
   `AppDelegate setApiKey` present (iOS/RN).
6. **Dev build, not Expo Go?** RN/Expo needs a prebuild / dev client; permissions granted
   before the user-location layer.
