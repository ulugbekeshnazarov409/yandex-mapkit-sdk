# Map objects, clustering & styling

Class names are Android Kotlin; iOS prefixes them with `YMK` and uses `with:` labels
(see `ios.md`). RN exposes a subset as components (`react-native.md`).

## Object types (all live in a `MapObjectCollection` = a layer)

| Object | Geometry | Key props |
|---|---|---|
| `PlacemarkMapObject` | `Point` | icon (`ImageProvider`) or view (`ViewProvider`), `IconStyle`, text, `zIndex`, `opacity`, `isDraggable`, `userData` |
| `PolylineMapObject` | `Polyline` | `strokeColor`, `strokeWidth`, `dashLength`/`gapLength`, `outlineColor`, gradient, `zIndex` |
| `PolygonMapObject` | `Polygon` (outer ring + holes) | `fillColor`, `strokeColor`, `strokeWidth`, `zIndex` |
| `CircleMapObject` | `Circle(center, radiusMeters)` | `fillColor`, `strokeColor`, `strokeWidth` |
| `ClusterizedPlacemarkCollection` | many `Point`s | auto-groups markers into clusters |

Keep references to update/remove. `collection.clear()` wipes a whole layer; `remove(obj)`
removes one. Architecture: one collection per layer (Marker / Route / Cluster / Polygon …).

## Placemark icons & custom views

```kotlin
// image icon + style
placemark.setIcon(ImageProvider.fromResource(ctx, R.drawable.pin))
placemark.setIconStyle(IconStyle().apply {
    anchor = PointF(0.5f, 1.0f)   // bottom-center
    scale = 1.2f
    zIndex = 10f
    rotationType = RotationType.ROTATE   // rotate with the map (e.g. arrows)
    flat = false                          // true = lies on the map plane
})

// CUSTOM VIEW marker (e.g. a price tag) — render an Android View, hand it over:
val tag = layoutInflater.inflate(R.layout.price_tag, null)
tag.findViewById<TextView>(R.id.price).text = "$120k"
placemark.setView(ViewProvider(tag))
// Constitution: property markers show PRICE, not a generic pin. Cache the rendered image;
// don't rebuild the View every frame.
```

## Dragging

```kotlin
placemark.isDraggable = true
placemark.setDragListener(object : MapObjectDragListener {
    override fun onMapObjectDragStart(o: MapObject) {}
    override fun onMapObjectDrag(o: MapObject, point: Point) {}   // live new coords
    override fun onMapObjectDragEnd(o: MapObject) {}
})
```

## Clustering (CLIENT — only under ~1000 objects)

```kotlin
val clusterized: ClusterizedPlacemarkCollection =
    map.mapObjects.addClusterizedPlacemarkCollection(object : ClusterListener {
        override fun onClusterAdded(cluster: Cluster) {
            // style the bubble with the child count
            cluster.appearance.setView(ViewProvider(clusterView(cluster.size)))
            cluster.appearance.setText(cluster.size.toString())
            cluster.addClusterTapListener {
                // zoom into the cluster's bounds
                true
            }
        }
    })

clusterized.addPlacemarks(points, ImageProvider.fromResource(ctx, R.drawable.pin), IconStyle())
// recompute when zoom changes:
clusterized.clusterPlacemarks(/*clusterRadius dp*/ 60.0, /*minZoom*/ 15)
```

> **Above ~1000 objects do NOT cluster on the client.** Request ready clusters from the
> backend (PostGIS `ST_ClusterDBSCAN`/`ST_ClusterKMeans`) and render each returned cluster
> as a single placemark bubble (count label) plus the few individual markers. See
> `architecture.md`.

## Polyline / polygon styling

```kotlin
polyline.apply {
    setStrokeColor(0xFF38463A.toInt()); strokeWidth = 5f
    outlineColor = 0xFFFFFFFF.toInt(); outlineWidth = 1f
    dashLength = 8f; gapLength = 6f          // dashed
    gradientLength = 0f
    zIndex = 100f
}
polygon.apply { fillColor = 0x3338463A; strokeColor = 0xFF38463A.toInt(); strokeWidth = 2f }
```

## Map types, night mode, POI

```kotlin
map.mapType = MapType.VECTOR_MAP            // SATELLITE | HYBRID | NONE
map.isNightModeEnabled = true
map.poiLimit = 0                            // hide built-in POIs to declutter
map.isIndoorEnabled = true                  // indoor plans where available
```

## Traffic layer

```kotlin
val traffic = MapKitFactory.getInstance().createTrafficLayer(mapView.mapWindow)
traffic.isTrafficVisible = true
traffic.addTrafficListener(object : TrafficListener {
    override fun onTrafficChanged(level: TrafficLevel?) {}   // level.level (0..10), color
    override fun onTrafficLoading() {}
    override fun onTrafficExpired() {}
})
```

## Custom map style (JSON)

Yandex supports a style JSON to recolor/hide map features (roads, water, labels…):

```kotlin
map.setMapStyle(/* style JSON string */ assets.readText("map_style.json"))
// "" resets to default. Build the JSON with Yandex's style spec (tags + stylers:
// color, opacity, visibility, geometry, scale).
```

## z-index & opacity (compositing order)

Every object has `zIndex` (higher = on top) and `opacity` (0–1). Order matters: user
layer above clusters above polygons above the base map. Set per-object or per-collection.

## Cleanup (memory)
On screen leave: `markerCollection.clear()`, `routeCollection.clear()`,
`clusterized.clear()`. Reuse marker images (cache `ImageProvider`/bitmaps); don't
re-decode per marker. Destroy layers you don't need.
