# Search, Suggest, Geocoding & Routing (the `full` variant)

Requires the `-full` Android dep / full iOS pod. **Every call returns a Session — store it
in a field/state or it's garbage-collected and your callback never fires.** (Top bug.)
Architecture: separate Autocomplete / Geocoder / Reverse / POI / Resolver services;
debounce input; cache aggressively; meet the latency budgets (autocomplete <150ms,
reverse <300ms).

---

## Search (POI / business / place by text) — Android Kotlin

```kotlin
private lateinit var searchManager: SearchManager
private var searchSession: Session? = null   // KEEP this reference

fun setup() {
    SearchFactory.initialize(context)
    searchManager = SearchFactory.getInstance()
        .createSearchManager(SearchManagerType.COMBINED) // ONLINE | OFFLINE | COMBINED
}

fun search(query: String, visibleRegion: VisibleRegion) {
    searchSession = searchManager.submit(
        query,
        VisibleRegionUtils.toPolygon(visibleRegion), // bias results to the viewport
        SearchOptions().apply { searchTypes = SearchType.BIZ.value or SearchType.GEO.value },
        object : Session.SearchListener {
            override fun onSearchResponse(response: Response) {
                for (item in response.collection.children) {
                    val obj = item.obj ?: continue
                    val point = obj.geometry.firstOrNull()?.point   // lat/lon
                    val name = obj.name
                    val address = obj.metadataContainer
                        .getItem(ToponymObjectMetadata::class.java)?.address?.formattedAddress
                    // draw markers in the Marker Layer
                }
            }
            override fun onSearchError(error: Error) { /* NetworkError / RemoteError */ }
        },
    )
}
// paginate: if (searchSession?.hasNextPage() == true) searchSession?.fetchNextPage(listener)
```

## Suggest (autocomplete) — Android

```kotlin
private var suggestSession: SuggestSession = searchManager.createSuggestSession()

// debounce keystrokes (~250ms), cache by query, THEN:
fun suggest(text: String, window: BoundingBox) {
    suggestSession.suggest(text, window, SuggestOptions(), object : SuggestSession.SuggestListener {
        override fun onResponse(items: List<SuggestItem>) {
            // items: title.text, subtitle?.text, action (SUBSTITUTE vs SEARCH), center point
        }
        override fun onError(error: Error) {}
    })
}
suggestSession.reset() // cancel in-flight on new keystroke
```

## Geocoder — forward (address → point) & reverse (point → address)

```kotlin
// FORWARD: address string → coordinates
searchSession = searchManager.submit("Tashkent, Amir Temur 1", null, SearchOptions(), listener)
// read point from response.collection.children[0].obj.geometry[0].point

// REVERSE: coordinates → address (point → address → region → district → street)
searchSession = searchManager.submit(
    Point(41.31, 69.28), /*zoom*/ 16, SearchOptions(), object : Session.SearchListener {
        override fun onSearchResponse(r: Response) {
            val toponym = r.collection.children.firstOrNull()?.obj
                ?.metadataContainer?.getItem(ToponymObjectMetadata::class.java)
            val addr = toponym?.address                        // formattedAddress + components[]
            // components carry kind: COUNTRY / PROVINCE / LOCALITY / DISTRICT / STREET / HOUSE
        }
        override fun onSearchError(e: Error) {}
    },
)
```
Cache reverse-geocode results aggressively (coords rarely move within a tile).

## Routing — driving / pedestrian / masstransit / bicycle (Android)

```kotlin
private lateinit var drivingRouter: DrivingRouter
private var drivingSession: DrivingSession? = null

fun setup() {
    DirectionsFactory.initialize(context)
    drivingRouter = DirectionsFactory.getInstance()
        .createDrivingRouter(DrivingRouterType.COMBINED)
}

fun route(from: Point, to: Point) {
    val points = listOf(RequestPoint(from, RequestPointType.WAYPOINT, null, null),
                        RequestPoint(to,   RequestPointType.WAYPOINT, null, null))
    drivingSession = drivingRouter.requestRoutes(
        points, DrivingOptions(), VehicleOptions(),
        object : DrivingSession.DrivingRouteListener {
            override fun onDrivingRoutes(routes: MutableList<DrivingRoute>) {
                val route = routes.firstOrNull() ?: return
                // draw in the ROUTE layer (separate collection):
                routeCollection.addPolyline(route.geometry)
                val meta = route.metadata.weight      // time.text, timeWithTraffic.text, distance.text
            }
            override fun onDrivingRoutesError(error: Error) {}
        },
    )
}
```
Other routers (same shape): `PedestrianRouter` (`TransportFactory`/`RoutingFactory`),
`MasstransitRouter`, `BicycleRouter` — each returns its own Session + Route with
`.geometry` (a `Polyline`) you add to the route collection.

## iOS (Swift) — same concepts, YMK* names

```swift
let searchManager = YMKSearch.sharedInstance().createSearchManager(with: .combined)
var searchSession: YMKSearchSession?           // KEEP it
searchSession = searchManager.submit(withText: "cafe",
    geometry: YMKVisibleRegionUtils.toPolygon(with: map.visibleRegion),
    searchOptions: YMKSearchOptions()) { (response, error) in
    guard let response = response else { return }
    for item in response.collection.children { /* item.obj?.geometry.first?.point */ }
}

// routing
let drivingRouter = YMKDirections.sharedInstance().createDrivingRouter(with: .combined)
var drivingSession: YMKDrivingSession?
drivingSession = drivingRouter.requestRoutes(with: points,
    drivingOptions: YMKDrivingDrivingOptions(), vehicleOptions: YMKDrivingVehicleOptions()) { routes, error in
    if let route = routes?.first { self.routeCollection.addPolyline(with: route.geometry) }
}
```

## React Native
`map.current?.findRoutes(points, ["driving"|"pedestrian"|"masstransit"|"bicycle"], cb)` for
routes (draw the returned points in a `<Polyline>`). For search/geocoding in RN, call your
**own backend** (which uses the HTTP Geocoder/Search API or PostGIS) — keeps keys server-side
and lets you cache + rate-limit per the constitution.

## NaviKit (turn-by-turn) — overview
NaviKit is a **separate SDK** layered on MapKit: it consumes a route and emits guidance
(maneuvers, voice, speed, lane info, reroute). Add the NaviKit dependency, build a route
with its navigation router, attach a `Navigation` object to the map, and subscribe to
guidance updates. Use it only for real turn-by-turn; for "show a route line" the routers
above are enough.

## Rules
- Store every Session; `reset()`/cancel in-flight on new input.
- Debounce suggest/search; cache by normalized query; respect latency budgets.
- Draw routes in the **Route Layer** only; clear that collection to remove a route.
- Bias search/suggest to the current viewport (`VisibleRegion` → polygon/bbox).
