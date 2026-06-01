# Recipes — copy-paste end-to-end tasks

Main snippets are **Android Kotlin** with iOS/RN notes where useful. Every recipe applies
the constitution (`architecture.md`): viewport-only fetching, **server** clusters above
~1000 objects, routes in their own collection, **keep every Session**, debounce input, and
property markers show **price**. Setup (Gradle, key, lifecycle) lives in `android.md` /
`ios.md` / `react-native.md` — referenced, not repeated.

---

## 1. Map centered on a city (with lifecycle)

```kotlin
class CityMapActivity : AppCompatActivity() {
    private lateinit var mapView: MapView

    override fun onCreate(savedInstanceState: Bundle?) {
        MapKitFactory.initialize(this)              // before the view (see android.md)
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_map)
        mapView = findViewById(R.id.mapview)

        mapView.mapWindow.map.move(
            CameraPosition(Point(41.3111, 69.2797), /*zoom*/ 12f, /*azimuth*/ 0f, /*tilt*/ 0f),
            Animation(Animation.Type.SMOOTH, 1f),
            null,
        )
    }

    override fun onStart() { super.onStart(); MapKitFactory.getInstance().onStart(); mapView.onStart() }
    override fun onStop()  { mapView.onStop(); MapKitFactory.getInstance().onStop(); super.onStop() }
}
```
- **iOS:** `map.move(with: YMKCameraPosition(target: YMKPoint(latitude: 41.3111, longitude: 69.2797), zoom: 12, azimuth: 0, tilt: 0), animation: …, cameraCallback: nil)`.
- **RN:** `<YaMap initialRegion={{ lat: 41.3111, lon: 69.2797, zoom: 12 }} />`.

---

## 2. Tap to drop a marker

```kotlin
private val markers: MapObjectCollection by lazy { mapView.mapWindow.map.mapObjects.addCollection() }

mapView.mapWindow.map.addInputListener(object : InputListener {
    override fun onMapTap(map: Map, point: Point) {
        markers.addPlacemark().apply {
            geometry = point
            setIcon(ImageProvider.fromResource(this@CityMapActivity, R.drawable.ic_pin))
            setIconStyle(IconStyle().apply { anchor = PointF(0.5f, 1f); scale = 1.1f })
        }
    }
    override fun onMapLongTap(map: Map, point: Point) {}
})
```
- **iOS:** conform to `YMKMapInputListener`; in `onMapTap(with:point:)` call
  `objects.addPlacemark()` and set geometry/icon.
- Markers go in their own collection (the Marker Layer) so you can `markers.clear()` wholesale.

---

## 3. Viewport-driven property fetch (camera idle → debounce → bounds+zoom → render)

```kotlin
private val markers = mapView.mapWindow.map.mapObjects.addCollection()
private val handler = Handler(Looper.getMainLooper())
private var pending: Runnable? = null

mapView.mapWindow.map.addCameraListener { map, _, _, finished ->
    if (!finished) return@addCameraListener          // only when the camera settles
    pending?.let { handler.removeCallbacks(it) }
    pending = Runnable {
        val region = map.visibleRegion                // bounds from the visible region
        val zoom = map.cameraPosition.zoom
        fetchViewport(region, zoom)                   // pseudo-backend call
    }
    handler.postDelayed(pending!!, 250)               // debounce ~250ms
}

// PSEUDO backend: send bounds + zoom, get back ONLY the visible objects (server owns granularity)
private fun fetchViewport(region: VisibleRegion, zoom: Float) {
    val body = ViewportQuery(
        northEast = LatLon(region.topRight.latitude, region.topRight.longitude),
        southWest = LatLon(region.bottomLeft.latitude, region.bottomLeft.longitude),
        zoom = zoom.toInt(),
    )
    api.fetchProperties(body) { result ->            // L1/L2 cache → backend (see architecture.md)
        runOnUiThread { render(result.markers) }     // render ONLY what the server returned
    }
}

private fun render(items: List<PropertyMarker>) {
    markers.clear()                                   // wipe the layer, redraw viewport-only
    items.forEach { p ->
        markers.addPlacemark().apply {
            geometry = Point(p.lat, p.lon)
            setView(ViewProvider(priceTag(p.price)))  // price marker, not a generic pin
            userData = p.id
        }
    }
}
```
- **Never render everything** — only objects intersecting the viewport. The backend returns
  the right granularity for the zoom band (countries → … → properties).
- **RN:** debounce `onCameraPositionChangeEnd`, read bounds via
  `map.current?.getVisibleRegion(cb)`, fetch through React Query, render returned markers only.

---

## 4. Server-cluster rendering (backend returns `{ clusters, markers }`)

Above ~1000 objects the **backend** clusters (PostGIS `ST_ClusterDBSCAN`) and returns ready
cluster bubbles plus the few individual markers. The map just draws them — no client
clustering.

```kotlin
private val clusterLayer = mapView.mapWindow.map.mapObjects.addCollection()  // cluster bubbles
private val markerLayer  = mapView.mapWindow.map.mapObjects.addCollection()  // real markers

private fun renderServerClusters(resp: ClusterResponse) {  // { clusters:[{lat,lon,count}], markers:[…] }
    clusterLayer.clear(); markerLayer.clear()

    resp.clusters.forEach { c ->                           // one placemark per cluster, count label
        clusterLayer.addPlacemark().apply {
            geometry = Point(c.lat, c.lon)
            setView(ViewProvider(clusterBubble(c.count)))  // bubble view showing the count
            addTapListener { _, _ ->
                mapView.mapWindow.map.move(                 // zoom into the cluster
                    CameraPosition(Point(c.lat, c.lon), mapView.mapWindow.map.cameraPosition.zoom + 2f, 0f, 0f),
                    Animation(Animation.Type.SMOOTH, 0.4f), null)
                true
            }
        }
    }
    resp.markers.forEach { m ->                             // individual price markers
        markerLayer.addPlacemark().apply {
            geometry = Point(m.lat, m.lon)
            setView(ViewProvider(priceTag(m.price)))
            userData = m.id
        }
    }
}
```
> For the **client-side** path (only under ~1000 objects) use
> `map.mapObjects.addClusterizedPlacemarkCollection(ClusterListener)` +
> `clusterPlacemarks(radius, minZoom)` — see `objects-styling.md`. Default to server
> clustering at scale.

---

## 5. Route between two points (separate route collection)

```kotlin
private lateinit var drivingRouter: DrivingRouter
private var drivingSession: DrivingSession? = null                       // KEEP the Session
private val routeCollection = mapView.mapWindow.map.mapObjects.addCollection()  // Route Layer only

fun setupRouting() {
    DirectionsFactory.initialize(this)
    drivingRouter = DirectionsFactory.getInstance().createDrivingRouter(DrivingRouterType.COMBINED)
}

fun route(from: Point, to: Point) {
    val points = listOf(
        RequestPoint(from, RequestPointType.WAYPOINT, null, null),
        RequestPoint(to,   RequestPointType.WAYPOINT, null, null),
    )
    drivingSession = drivingRouter.requestRoutes(
        points, DrivingOptions(), VehicleOptions(),
        object : DrivingSession.DrivingRouteListener {
            override fun onDrivingRoutes(routes: MutableList<DrivingRoute>) {
                val r = routes.firstOrNull() ?: return
                routeCollection.clear()                       // clear old route first
                routeCollection.addPolyline(r.geometry).apply {
                    setStrokeColor(0xFF38463A.toInt()); strokeWidth = 5f
                }
                // r.metadata.weight → time.text / timeWithTraffic.text / distance.text
            }
            override fun onDrivingRoutesError(error: Error) {}
        },
    )
}
```
- Route geometry lives **only** in `routeCollection` — never the marker layer. Clear that
  collection to remove the route.
- Other modes (`PedestrianRouter`, `MasstransitRouter`, `BicycleRouter`) follow the same
  shape — see `search-routing.md`.
- **RN:** `map.current?.findRoutes([a, b], ["driving"], e => setRoutePoints(e.routes[0].sections.flatMap(s => s.points)))` then draw a `<Polyline>`.

---

## 6. Search nearby + show results as markers

```kotlin
private lateinit var searchManager: SearchManager
private var searchSession: Session? = null                    // KEEP the Session
private val resultLayer = mapView.mapWindow.map.mapObjects.addCollection()

fun setupSearch() {
    SearchFactory.initialize(this)
    searchManager = SearchFactory.getInstance().createSearchManager(SearchManagerType.COMBINED)
}

fun searchNearby(query: String) {
    val region = mapView.mapWindow.map.visibleRegion          // bias to what the user sees
    searchSession = searchManager.submit(
        query,
        VisibleRegionUtils.toPolygon(region),
        SearchOptions().apply { searchTypes = SearchType.BIZ.value or SearchType.GEO.value },
        object : Session.SearchListener {
            override fun onSearchResponse(response: Response) {
                resultLayer.clear()
                for (item in response.collection.children) {
                    val point = item.obj?.geometry?.firstOrNull()?.point ?: continue
                    resultLayer.addPlacemark().apply {
                        geometry = point
                        setIcon(ImageProvider.fromResource(this@CityMapActivity, R.drawable.ic_poi))
                        userData = item.obj?.name
                    }
                }
            }
            override fun onSearchError(error: Error) {}
        },
    )
}
```
- Bias results to the current `visibleRegion`; debounce the query box (~250ms) and cache by
  normalized query. Full Search/Suggest/Geocoder reference: `search-routing.md`.
- **iOS:** `YMKSearch.sharedInstance().createSearchManager(with: .combined)`, keep
  `var searchSession: YMKSearchSession?`.

---

## 7. Follow user location

```kotlin
// 1) request permission FIRST
private val permission = registerForActivityResult(ActivityResultContracts.RequestPermission()) { granted ->
    if (granted) enableUserLocation()
}
// call: permission.launch(Manifest.permission.ACCESS_FINE_LOCATION)

private lateinit var userLayer: UserLocationLayer

private fun enableUserLocation() {
    userLayer = MapKitFactory.getInstance().createUserLocationLayer(mapView.mapWindow)
    userLayer.isVisible = true
    userLayer.isHeadingEnabled = true                         // arrow rotates with the compass
    userLayer.setObjectListener(object : UserLocationObjectListener {
        override fun onObjectAdded(view: UserLocationView) {
            view.accuracyCircle.fillColor = 0x224285F4         // always show the accuracy circle
        }
        override fun onObjectRemoved(view: UserLocationView) {}
        override fun onObjectUpdated(view: UserLocationView, event: ObjectEvent) {
            userLayer.cameraPosition()?.let {                  // camera follows the user
                mapView.mapWindow.map.move(it, Animation(Animation.Type.SMOOTH, 0.3f), null)
            }
        }
    })
}
```
- Permission **before** `createUserLocationLayer`; iOS also needs
  `NSLocationWhenInUseUsageDescription`. **RN:** grant `expo-location`, then `showUserPosition`.
- Don't teleport — animate the camera; show the accuracy circle (see `architecture.md`).

---

## 8. Price markers via ViewProvider (custom price-tag view, cached)

Property markers show **price**, not a generic pin. Render the tag once and **cache** the
`ViewProvider` per price label so you don't rebuild the View every frame.

```kotlin
private val tagCache = HashMap<String, ViewProvider>()        // key: price label → cached view

private fun priceTag(price: Int): ViewProvider {
    val label = "$${(price / 1000)}k"                         // e.g. $120k
    return tagCache.getOrPut(label) {
        val view = layoutInflater.inflate(R.layout.price_tag, null)
        view.findViewById<TextView>(R.id.price).text = label
        ViewProvider(view)
    }
}

// use it on any placemark:
markerLayer.addPlacemark().apply {
    geometry = Point(p.lat, p.lon)
    setView(priceTag(p.price))                                // cached ViewProvider
    userData = p.id
}
```
- Cache by the **rendered label**, not per object — many markers share `$120k`.
- **RN:** a `<Marker point={{lat,lon}}><View style={priceTag}><Text>${k}</Text></View></Marker>`
  child view; memoize the component. **iOS:** build a `UIView` price tag and
  `setView(with:)`. More on icons/ViewProvider: `objects-styling.md`.
