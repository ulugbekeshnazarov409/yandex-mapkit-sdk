# Architecture at scale — the map engineering constitution

> Assume **100k users, 1M properties, 10k concurrent map sessions** for every decision.
> A map is not a UI widget — it is a spatial operating system. Optimize rendering,
> memory, network, battery, scalability, and GIS correctness together. Read this BEFORE
> writing any non-trivial map code.

## AI pre-flight checklist (answer all 8 before generating code)

1. **Zoom strategy** — what loads at which zoom band? (see table below)
2. **Clustering strategy** — client (<1000 objects) or server (PostGIS)?
3. **Caching strategy** — which of the 4 cache levels apply?
4. **Spatial indexes** — which GiST / BTree / composite indexes exist?
5. **Viewport queries** — what bounds + zoom does the fetch send?
6. **Layer architecture** — which of the 7 layers are involved?
7. **Offline strategy** — what is cached for offline-first?
8. **Realtime strategy** — tracking cadence + interpolation?

Never generate a naive "render all markers" implementation. If a request implies one,
push back and apply this doc.

## Layered architecture — business logic OUT of map components

```
MapScreen (dumb view: renders ready objects, forwards gestures)
   ↓
Map Engine (camera, layers, marker pool, lifecycle)
   ↓
Spatial Services (viewport queries, search, routing, clustering client glue)
   ↓
PostGIS (geometry, indexes, ST_* queries, server clustering)
   ↓
Yandex APIs (tiles, geocoder, router, suggest)
```

The map component must contain **no** filtering, clustering, search, or route math.
It receives `{ objects, clusters, bounds, metadata, pagination }` and draws them.

## The 7 layers (never mix object types across them)

`Map Layer` · `Marker Layer` · `Route Layer` · `User Layer` · `Cluster Layer` ·
`Heatmap Layer` · `Polygon Layer`. Each is its own `MapObjectCollection` (Android) /
collection (iOS) / map object source. Route geometry never lives in the marker layer;
clusters never mix with raw markers. Destroy a layer's collection to free it wholesale.

## Coordinates — WGS84, lat/lon, validated

- Native MapKit `Point(latitude, longitude)`. JS API v3 uses `[lng, lat]` — never swap.
- Validate every coordinate that crosses a boundary:
  `lat ∈ [-90, 90]`, `lon ∈ [-180, 180]`. Reject malformed geometry (security).
- Store everything as WGS84 (SRID 4326) in PostGIS.

## Zoom strategy — load detail by zoom band

| Zoom | Load |
|---|---|
| 1–6 | countries only |
| 7–10 | regions |
| 11–14 | districts |
| 15–18 | properties (individual) |
| 19+ | building level |

Never load detailed objects at low zoom. The viewport query carries `zoom` so the
backend returns the right granularity (countries → regions → districts → properties).

## Viewport fetching — never fetch the country

Fetch by the current visible region only. Query payload:
```json
{ "northEast": { "lat": .., "lon": .. },
  "southWest": { "lat": .., "lon": .. },
  "zoom": 14 }
```
Pipeline: `camera idle → debounce → bounds + zoom → backend → visible objects`.
Only objects intersecting the viewport may be rendered. Drive it from the camera-idle
listener (Android `CameraListener` with `finished == true`; iOS map camera callback;
RN `onCameraPositionChangeEnd`).

PostGIS viewport query:
```sql
SELECT id, ST_Y(geom) lat, ST_X(geom) lon, price, currency, type, status, images_count
FROM properties
WHERE status = 'active'
  AND geom && ST_MakeEnvelope(:west, :south, :east, :north, 4326)  -- GiST index hit
ORDER BY price
LIMIT 500;
```

## Marker rules — markers are expensive

- **Never render all markers.** Viewport-only. 10k markers on screen = jank + OOM.
- **Reuse marker views** (pool); cache marker images; lazy-load icons. Avoid image rerenders.
- Property markers should show **price**, not a generic pin (`$50k`, `$120k`) — visible
  pricing lifts conversion. Render the price label as a cached image/ViewProvider.
- Default markers beat custom ones unless custom is required.

## Clustering — server-side above 1000 objects

- Client clustering is allowed **only** under ~1000 objects.
- Above that, cluster on the **backend** with PostGIS and send ready clusters to the map:
  ```sql
  -- density clustering for the current viewport + zoom
  SELECT cid, COUNT(*) AS n,
         ST_Y(ST_Centroid(ST_Collect(geom))) AS lat,
         ST_X(ST_Centroid(ST_Collect(geom))) AS lon
  FROM (
    SELECT geom, ST_ClusterDBSCAN(geom, eps := :eps_for_zoom, minpoints := 1) OVER () AS cid
    FROM properties
    WHERE geom && ST_MakeEnvelope(:west,:south,:east,:north,4326)
  ) s
  GROUP BY cid;
  ```
  Use `ST_ClusterDBSCAN` (density) or `ST_ClusterKMeans` (fixed k). The map only draws the
  returned cluster bubbles (count label) + the few real markers.

## Property model (marketplace) — every property has

`id, location (geometry), price, currency, type, status, images_count, created_at, updated_at`.
Spatial index is **mandatory**:
```sql
CREATE INDEX idx_properties_geom ON properties USING GIST (geom);          -- spatial
CREATE INDEX idx_properties_status_geom ON properties USING GIST (geom)
  WHERE status = 'active';                                                  -- partial
CREATE INDEX idx_properties_price ON properties (price);                    -- btree
CREATE INDEX idx_properties_type_status ON properties (type, status);       -- composite
```
No production GIS without indexes.

## Polygons — districts/regions/neighborhoods as geometry

Store as polygons (SRID 4326). Use spatial predicates, never manual coordinate math:
`ST_Contains`, `ST_Intersects`, `ST_Within`, `ST_Distance`, `ST_Buffer`, `ST_Centroid`,
`ST_MakePoint`. Example "which district is this point in":
```sql
SELECT d.id, d.name FROM districts d
WHERE ST_Contains(d.geom, ST_SetSRID(ST_MakePoint(:lon, :lat), 4326)) LIMIT 1;
```

## Search architecture — separated services

`Autocomplete` · `Geocoder` · `Reverse Geocoder` · `POI Search` · `Address Resolver` —
each its own service. Address search pipeline:
```
input → debounce(≈250ms) → cache lookup → Yandex search → normalize → result
```
Never call the API on every keystroke. Reverse geocoding:
`coordinate → address → region → district → street`, cached aggressively. (API details:
`search-routing.md`.)

## Route engine — its own layer

Routes live in the **Route Layer**, never mixed with markers. Driving/pedestrian/
masstransit/bicycle via the MapKit routers (`search-routing.md`). Draw the route polyline
into the route collection; clear that collection to remove a route.

## Realtime tracking — cadence + interpolation

- Send GPS by mode, never every second: **walking 5–10s, bike 3–5s, car 1–3s.** Battery first.
- Driver/object marker: store `speed, bearing, heading, accuracy, timestamp`.
  **Interpolate** between updates (animate along the path) — never teleport the marker.
- User location: always show the accuracy circle; filter GPS noise (Kalman preferred);
  use GPS confidence; never trust raw coordinates.

## Caching — 4 levels

`L1 memory` → `L2 React Query` → `L3 disk` → `L4 backend`. Cache tiles, searches,
district polygons, routes, favorites. **Offline-first** preferred: the last viewport,
its polygons, and recent searches survive a network drop.

## Memory — strict budget

Destroy unused layers and markers. Reuse marker views. Avoid image rerenders. On screen
leave / tab switch, clear the marker + cluster collections; rebuild from cache on return.

## React Native specifics

- Use the **native** Yandex SDK (`react-native-yamap`), not a web wrapper. **Dev build**
  only — Expo Go unsupported.
- Memoize the map component (`React.memo`); never let parent re-renders re-render the map.
- Virtualize any marker LIST UI (FlashList). Minimize bridge traffic — batch marker
  updates, send cluster/marker arrays, not per-marker calls.

## Performance budgets (monitor these)

| Operation | Target |
|---|---|
| Autocomplete response | < 150 ms |
| Reverse geocoding | < 300 ms |
| Viewport fetch | < 500 ms |
| Route generation | < 1000 ms |

Track: map-open time, marker render count, route-gen time, GPS accuracy, search latency,
cluster-gen latency, memory consumption, crash rate.

## Security

Validate all coordinates, bounds, and polygons server-side. Reject malformed-geometry
attacks (self-intersecting polygons, out-of-range coords, huge envelopes). Rate-limit
search and protect route endpoints (they're expensive). Never trust client geometry.

## API design

The map API returns `{ objects, clusters, bounds, metadata, pagination }` — never raw
table rows. The client draws what it's given; the server owns granularity, clustering,
and limits.
