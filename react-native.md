# React Native / Expo — react-native-yamap

Native Yandex MapKit for RN via **`react-native-yamap`** (volga-volga). It's a **native
module** → you need a **development build / prebuild**. **Expo Go does NOT work.**
(Constitution: use the native SDK, never a web wrapper.)

## 1. Install

```bash
npm i react-native-yamap
# iOS
npx pod-install
```

## 2. iOS native init (required)

react-native-yamap needs the key set in the native `AppDelegate` at launch. In
`ios/<App>/AppDelegate.mm` (or `.swift`):

```objc
#import <YandexMapsMobile/YMKMapKitFactory.h>
// inside application:didFinishLaunchingWithOptions:
[YMKMapKit setApiKey:@"YOUR_API_KEY"];
[YMKMapKit setLocale:@"ru_RU"];   // optional
```

Android needs nothing extra beyond autolinking; the key is set from JS (below).

## 3. Expo (SDK 50+/56) — config plugin, then a dev build

Expo manages native projects, so don't hand-edit `ios/` permanently — drive it from
`app.config.ts` and a **config plugin** (the lib ships/has community plugins that patch
`AppDelegate` + add the Maven repo + permissions). Convert `app.json` → `app.config.ts`:

```ts
// app.config.ts
export default {
  expo: {
    plugins: [
      ["react-native-yamap", { apiKey: process.env.YANDEX_MAPKIT_KEY }],
      // location permission strings:
      ["expo-location", { locationWhenInUsePermission: "Show your position on the map." }],
    ],
    ios: { infoPlist: { NSLocationWhenInUseUsageDescription: "Show your position on the map." } },
    android: { permissions: ["ACCESS_FINE_LOCATION", "ACCESS_COARSE_LOCATION"] },
  },
};
```

Then build a dev client (NOT Expo Go):
```bash
npx expo prebuild
eas build --profile development --platform android   # or ios
# run JS against the dev build:
npx expo start --dev-client
```
> If no maintained plugin matches your SDK, write a tiny local plugin that (iOS) inserts
> the `YMKMapKit setApiKey` lines into AppDelegate and (Android) adds the Maven repo +
> permissions. See the `expo-module` / config-plugins docs.

## 4. Initialize + first map

```tsx
import YaMap, { Marker, Polyline, Polygon, Circle } from "react-native-yamap";
import { useRef, useEffect } from "react";

YaMap.init(process.env.YANDEX_MAPKIT_KEY!); // once, app start (index/root)

export function MapScreen() {
  const map = useRef<YaMap>(null);
  return (
    <YaMap
      ref={map}
      style={{ flex: 1 }}
      initialRegion={{ lat: 41.3111, lon: 69.2797, zoom: 13 }}
      mapType="vector"               // 'vector' | 'satellite' | 'hybrid'
      nightMode
      showUserPosition               // needs location permission granted first
      onMapLoaded={() => {}}
      onCameraPositionChangeEnd={(e) => {
        // e.nativeEvent: { point, zoom, azimuth, tilt, ... }
        // → debounce → viewport fetch (see below)
      }}
    >
      {/* Markers/objects as children */}
    </YaMap>
  );
}
```

## 5. Markers, objects, camera

```tsx
// price marker (constitution: show price, not a pin) with a custom child view
<Marker point={{ lat, lon }} onPress={() => openCard(id)} anchor={{ x: 0.5, y: 1 }}>
  <View style={styles.priceTag}><Text>${(price/1000).toFixed(0)}k</Text></View>
</Marker>

// or an image marker (cache the source)
<Marker point={{ lat, lon }} source={require("../assets/pin.png")} scale={1.2} />

<Polyline points={routePoints} strokeColor="#38463A" strokeWidth={4} />
<Polygon points={ring} fillColor="#3338463A" strokeColor="#38463A" strokeWidth={2} />
<Circle center={{ lat, lon }} radius={500} fillColor="#22C9A227" strokeColor="#C9A227" />

// imperative camera
map.current?.setCenter({ lat, lon, zoom: 16 }, 0 /*azimuth*/, 0 /*tilt*/, 0.5 /*durationSec*/);
map.current?.fitAllMarkers();
map.current?.getVisibleRegion((r) => { /* r.bottomLeft / topRight -> bounds for fetch */ });
```

## 6. Routes

```tsx
map.current?.findRoutes(
  [{ lat: a.lat, lon: a.lon }, { lat: b.lat, lon: b.lon }],
  ["driving"],                       // 'driving' | 'pedestrian' | 'masstransit' | 'bicycle'
  (event) => {
    const route = event.routes[0];
    setRoutePoints(route.sections.flatMap(s => s.points)); // draw in a <Polyline> (Route Layer)
  },
);
```

## 7. Scale rules applied (from architecture.md)

```tsx
// Memoize the map — never re-render it on every parent state change.
const Map = React.memo(MapScreen);

// Viewport fetch on camera idle, debounced; render ONLY returned objects.
const onIdle = useMemo(() => debounce((bounds, zoom) => {
  // L1/L2 cache check (React Query) -> backend -> setObjects(visibleOnly)
  queryClient.fetchQuery({ queryKey: ["map", bounds, zoom], queryFn: () => fetchViewport(bounds, zoom) });
}, 250), []);
```
- Render viewport objects only; for >1000 objects request **server clusters** and render
  cluster bubbles + the few real markers (see `architecture.md`).
- Keep marker images cached; reuse; lazy-load. Don't pass huge arrays across the bridge
  every frame — batch.

## Gotchas
- Expo Go = instant failure (native module). Always dev build. (See project memory: native rebuild gate.)
- iOS: forgetting the AppDelegate `setApiKey` → blank map / crash. The JS `YaMap.init` is
  not always enough on iOS; set it natively too.
- Android Maven repo missing → build fails resolving `com.yandex.android:maps.mobile`.
- Match `react-native-yamap` version to your RN/Expo SDK; check the repo's compatibility table.
