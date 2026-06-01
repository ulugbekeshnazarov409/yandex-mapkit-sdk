# iOS (Swift) — Yandex MapKit v4

## 1. CocoaPods

```ruby
# Podfile
target 'App' do
  use_frameworks!
  pod 'YandexMapsMobile', '4.+'      # pin a real version; full pod for search/routing
end
```
```bash
pod install   # open the .xcworkspace afterwards
```
Add to `Info.plist`: `NSLocationWhenInUseUsageDescription` (+ `…AlwaysAndWhenInUse…` if
needed) for the user-location layer.

## 2. Set the key (AppDelegate)

```swift
import YandexMapsMobile

func application(_ application: UIApplication,
                didFinishLaunchingWithOptions launchOptions: ...) -> Bool {
    YMKMapKit.setApiKey("YOUR_API_KEY")    // before any map is created
    YMKMapKit.setLocale("ru_RU")           // optional
    _ = YMKMapKit.sharedInstance()         // instantiate MapKit
    return true
}
```

## 3. YMKMapView + camera

Add a `YMKMapView` (in code or a storyboard custom class), then:

```swift
import YandexMapsMobile

final class MapViewController: UIViewController {
    @IBOutlet weak var mapView: YMKMapView!  // or create programmatically and addSubview

    override func viewDidLoad() {
        super.viewDidLoad()
        let map = mapView.mapWindow.map
        map.move(
            with: YMKCameraPosition(
                target: YMKPoint(latitude: 41.3111, longitude: 69.2797),
                zoom: 14, azimuth: 0, tilt: 0),
            animation: YMKAnimation(type: .smooth, duration: 1),
            cameraCallback: nil)
    }
}
```
> `YMKPoint(latitude:longitude:)` — lat first. iOS MapKit manages its own lifecycle; you
> don't need Android's `onStart/onStop` calls, but release the controller/map normally.

## 4. Map objects

```swift
let objects = mapView.mapWindow.map.mapObjects

// Placemark (marker)
let pin = objects.addPlacemark()
pin.geometry = YMKPoint(latitude: 41.31, longitude: 69.28)
pin.setIconWith(UIImage(named: "pin")!,
                style: YMKIconStyle(anchor: CGPoint(x: 0.5, y: 1) as NSValue,
                                    rotationType: nil, zIndex: 1, flat: false,
                                    visible: true, scale: 1.2, tappableArea: nil))
pin.userData = placeModel as AnyObject
pin.addTapListener(with: self)   // conform to YMKMapObjectTapListener

// Polyline
let line = objects.addPolyline(with: YMKPolyline(points: [p1, p2]))
line.strokeColor = UIColor(red: 0.22, green: 0.27, blue: 0.23, alpha: 1)
line.strokeWidth = 4

// Polygon (ring + holes)
let poly = objects.addPolygon(with: YMKPolygon(outerRing: YMKLinearRing(points: ring), innerRings: []))
poly.fillColor = UIColor.systemGreen.withAlphaComponent(0.2)
poly.strokeColor = .systemGreen; poly.strokeWidth = 2

// Circle (radius in METERS)
let circle = objects.addCircle(with: YMKCircle(center: center, radius: 500))
circle.fillColor = UIColor.systemYellow.withAlphaComponent(0.15)
circle.strokeColor = .systemYellow; circle.strokeWidth = 2

let group = objects.add()   // sub-collection (a layer); group.clear() to wipe
```

## 5. Listeners

```swift
// Map taps — conform to YMKMapInputListener
mapView.mapWindow.map.addInputListener(with: self)
func onMapTap(with map: YMKMap, point: YMKPoint) { /* drop marker */ }
func onMapLongTap(with map: YMKMap, point: YMKPoint) {}

// Camera — conform to YMKMapCameraListener
mapView.mapWindow.map.addCameraListener(with: self)
func onCameraPositionChanged(with map: YMKMap, cameraPosition: YMKCameraPosition,
                             cameraUpdateReason: YMKCameraUpdateReason, finished: Bool) {
    if finished { /* viewport fetch */ }
}

// Marker taps — conform to YMKMapObjectTapListener
func onMapObjectTap(with mapObject: YMKMapObject, point: YMKPoint) -> Bool {
    let data = (mapObject as? YMKPlacemarkMapObject)?.userData
    return true
}
```

## 6. User location

```swift
let userLayer = YMKMapKit.sharedInstance().createUserLocationLayer(with: mapView.mapWindow)
userLayer.setVisibleWithOn(true)
userLayer.isHeadingEnabled = true
userLayer.setObjectListenerWith(self)   // YMKUserLocationObjectListener to style arrow/accuracy
```
Request `CLLocationManager` authorization first.

## 7. Layers & map type

```swift
mapView.mapWindow.map.mapType = .vectorMap   // .satellite / .hybrid
mapView.mapWindow.map.isNightModeEnabled = true
let traffic = YMKMapKit.sharedInstance().createTrafficLayer(with: mapView.mapWindow)
traffic.setTrafficVisibleWithOn(true)
```

Search/routing → `search-routing.md`. Clustering, ViewProvider markers, styling →
`objects-styling.md`.
