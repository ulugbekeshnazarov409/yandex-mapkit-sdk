# Android (Kotlin) — Yandex MapKit v4

## 1. Gradle

`settings.gradle` / project `build.gradle` repositories:
```gradle
repositories {
    mavenCentral()
    google()
}
```

Module `build.gradle`:
```gradle
dependencies {
    // -full = map + routing + search + suggest + geocoding + panorama
    // -lite = map + traffic + location only
    implementation "com.yandex.android:maps.mobile:4.+:full@aar" // pin a real version, e.g. 4.19.0-full
    // check Maven Central for the latest: com.yandex.android:maps.mobile
}
```
Add `INTERNET` (+ location) permissions to `AndroidManifest.xml`:
```xml
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION"/>
```

## 2. Set the key (Application)

```kotlin
class App : Application() {
    override fun onCreate() {
        super.onCreate()
        MapKitFactory.setApiKey(BuildConfig.YANDEX_MAPKIT_KEY)  // before initialize()
        MapKitFactory.setLocale("ru_RU")                        // optional; before initialize()
    }
}
```
Register it: `<application android:name=".App" …>`.

## 3. MapView + lifecycle (Activity)

Layout:
```xml
<com.yandex.mapkit.mapview.MapView
    android:id="@+id/mapview"
    android:layout_width="match_parent"
    android:layout_height="match_parent"/>
```

```kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var mapView: MapView

    override fun onCreate(savedInstanceState: Bundle?) {
        MapKitFactory.initialize(this)          // MUST run before the MapView is used
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        mapView = findViewById(R.id.mapview)

        val map = mapView.mapWindow.map         // v4: mapWindow.map (not mapView.map)
        map.move(
            CameraPosition(Point(41.3111, 69.2797), /*zoom*/ 14f, /*azimuth*/ 0f, /*tilt*/ 0f),
            Animation(Animation.Type.SMOOTH, 1f),
            null,
        )
    }

    // Bind MapKit + MapView to the lifecycle — required, or you leak native resources.
    override fun onStart() {
        super.onStart()
        MapKitFactory.getInstance().onStart()
        mapView.onStart()
    }
    override fun onStop() {
        mapView.onStop()
        MapKitFactory.getInstance().onStop()
        super.onStop()
    }
}
```

> `Point(latitude, longitude)` — Yandex is **lat, lon**.

## 4. Camera

```kotlin
val map = mapView.mapWindow.map
// instant
map.move(CameraPosition(Point(41.31, 69.28), 16f, 0f, 0f))
// animated + completion
map.move(
    CameraPosition(Point(41.31, 69.28), 16f, 30f, 45f),
    Animation(Animation.Type.SMOOTH, 0.6f),
) { /* onMoveFinished */ }

// fit a set of points
val geometry = Geometry.fromBoundingBox(/* BoundingBox */)
val position = map.cameraPosition(geometry)
map.move(position, Animation(Animation.Type.SMOOTH, 0.5f), null)

// read current
val cp: CameraPosition = map.cameraPosition
```

## 5. Map objects (markers, lines, polygons, circles)

```kotlin
val objects: MapObjectCollection = map.mapObjects

// Placemark (marker)
val pin = objects.addPlacemark().apply {
    geometry = Point(41.31, 69.28)
    setIcon(ImageProvider.fromResource(this@MainActivity, R.drawable.ic_pin))
    setIconStyle(IconStyle().apply { anchor = PointF(0.5f, 1.0f); scale = 1.2f })
    // custom Android View as the marker instead of an image:
    // setView(ViewProvider(myComposeOrXmlView))
    userData = placeModel            // attach your data for the tap listener
}
pin.addTapListener { mapObject, point ->
    val data = (mapObject as PlacemarkMapObject).userData
    true // handled
}

// Polyline
val line = objects.addPolyline(Polyline(listOf(Point(41.31, 69.28), Point(41.33, 69.30)))).apply {
    setStrokeColor(0xFF38463A.toInt())
    strokeWidth = 4f
    // dashLength / gapLength for dashed lines
}

// Polygon (outer ring + optional holes)
val poly = objects.addPolygon(Polygon(LinearRing(ringPoints), emptyList())).apply {
    fillColor = 0x3338463A
    strokeColor = 0xFF38463A.toInt()
    strokeWidth = 2f
}

// Circle (center + radius in METERS)
val circle = objects.addCircle(Circle(Point(41.31, 69.28), 500f)).apply {
    setFillColor(0x22C9A227)
    setStrokeColor(0xFFC9A227.toInt())
    strokeWidth = 2f
}

// Group + bulk-remove
val group: MapObjectCollection = objects.addCollection()
group.clear()      // remove everything in the collection
objects.remove(pin) // remove one
```

## 6. Map gestures & events

```kotlin
// Tap / long-tap anywhere on the map
map.addInputListener(object : InputListener {
    override fun onMapTap(map: Map, point: Point) { /* drop a marker, etc. */ }
    override fun onMapLongTap(map: Map, point: Point) {}
})

// Camera movement (pan/zoom), with finish flag + reason
map.addCameraListener { _, cameraPosition, reason, finished ->
    if (finished) { /* fetch data for the new viewport */ }
    // reason: GESTURES vs APPLICATION
}

// Toggle gestures
map.isRotateGesturesEnabled = true
map.isTiltGesturesEnabled = false
map.isScrollGesturesEnabled = true
map.isZoomGesturesEnabled = true
```

## 7. User location layer (+ permission)

```kotlin
// 1) request ACCESS_FINE_LOCATION at runtime FIRST (ActivityResultContracts.RequestPermission)
// 2) then:
val userLayer: UserLocationLayer =
    MapKitFactory.getInstance().createUserLocationLayer(mapView.mapWindow)
userLayer.isVisible = true
userLayer.isHeadingEnabled = true   // rotate the arrow with the device compass

userLayer.setObjectListener(object : UserLocationObjectListener {
    override fun onObjectAdded(view: UserLocationView) {
        // customise the pin/accuracy circle
        view.arrow.setIcon(ImageProvider.fromResource(this@MainActivity, R.drawable.ic_arrow))
        view.accuracyCircle.fillColor = 0x224285F4
    }
    override fun onObjectRemoved(view: UserLocationView) {}
    override fun onObjectUpdated(view: UserLocationView, event: ObjectEvent) {}
})

// follow the user
userLayer.cameraPosition()?.let { map.move(it) }
```

## 8. Layers & map type

```kotlin
map.mapType = MapType.VECTOR_MAP            // or SATELLITE, HYBRID
val traffic = MapKitFactory.getInstance().createTrafficLayer(mapView.mapWindow)
traffic.isTrafficVisible = true
map.isNightModeEnabled = true               // dark map
```

## 9. ProGuard / R8
MapKit ships consumer rules; usually nothing extra. If you hit reflection/Native crashes
in release, add: `-keep class com.yandex.** { *; }` and keep your model classes used as
`userData`.

## Notes
- Search, suggest, geocoding, and routing live in `search-routing.md` (need the `-full` variant).
- Clustering, ViewProvider markers, and styling depth are in `objects-styling.md`.
- Java works too — same classes; translate `apply {}` to setters and lambdas to anonymous
  classes.
