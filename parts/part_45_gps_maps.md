# Part 45: GPS และ Maps

## บทนำ

การใช้งาน GPS และ Maps เป็นฟีเจอร์ที่ขาดไม่ได้ในแอปส่วนใหญ่ เช่น แอปส่งอาหาร แอปนำทาง หรือแอปติดตามตำแหน่ง ใน Flutter เราใช้ `geolocator` สำหรับ GPS และ `google_maps_flutter` สำหรับแสดงแผนที่

---

## 45.1 Geolocator Package

### ติดตั้ง

```yaml
# pubspec.yaml
dependencies:
  geolocator: ^11.0.0
  google_maps_flutter: ^2.6.0
  geocoding: ^3.0.0
  flutter_polyline_points: ^2.0.0
```

### ตั้งค่า Android

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<manifest>
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION"/>
    <uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION"/>
    
    <application>
        <!-- Google Maps API Key -->
        <meta-data
            android:name="com.google.android.geo.API_KEY"
            android:value="YOUR_API_KEY_HERE"/>
    </application>
</manifest>
```

### ตั้งค่า iOS

```xml
<!-- ios/Runner/Info.plist -->
<key>NSLocationWhenInUseUsageDescription</key>
<string>แอปต้องการตำแหน่งของคุณเพื่อแสดงบริการใกล้เคียง</string>

<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>แอปต้องการตำแหน่งของคุณแม้ในขณะที่แอปไม่ได้เปิดอยู่</string>

<key>NSLocationAlwaysUsageDescription</key>
<string>แอปต้องการตำแหน่งของคุณตลอดเวลา</string>
```

---

## 45.2 Location Service

```dart
import 'package:geolocator/geolocator.dart';
import 'package:geocoding/geocoding.dart';

class LocationService {
  // ตรวจสอบและขอ permission
  static Future<bool> requestPermission() async {
    // ตรวจสอบว่า location service เปิดอยู่ไหม
    bool serviceEnabled = await Geolocator.isLocationServiceEnabled();
    if (!serviceEnabled) {
      // ขอให้เปิด location service
      await Geolocator.openLocationSettings();
      return false;
    }
    
    // ตรวจสอบ permission
    LocationPermission permission = await Geolocator.checkPermission();
    
    if (permission == LocationPermission.denied) {
      permission = await Geolocator.requestPermission();
      if (permission == LocationPermission.denied) {
        return false;
      }
    }
    
    if (permission == LocationPermission.deniedForever) {
      // ไม่สามารถขอ permission ได้ ต้องไปที่ settings
      await Geolocator.openAppSettings();
      return false;
    }
    
    return true;
  }
  
  // ได้ตำแหน่งปัจจุบันครั้งเดียว
  static Future<Position?> getCurrentPosition() async {
    final hasPermission = await requestPermission();
    if (!hasPermission) return null;
    
    try {
      return await Geolocator.getCurrentPosition(
        desiredAccuracy: LocationAccuracy.high,
        timeLimit: const Duration(seconds: 10),
      );
    } catch (e) {
      print('ไม่สามารถหาตำแหน่งได้: $e');
      return null;
    }
  }
  
  // ได้ตำแหน่งแบบ last known (เร็วกว่า)
  static Future<Position?> getLastKnownPosition() async {
    final hasPermission = await requestPermission();
    if (!hasPermission) return null;
    
    return await Geolocator.getLastKnownPosition();
  }
  
  // Stream ตำแหน่งแบบต่อเนื่อง
  static Stream<Position> getPositionStream({
    LocationAccuracy accuracy = LocationAccuracy.high,
    int distanceFilter = 10, // meters
  }) {
    const locationSettings = LocationSettings(
      accuracy: LocationAccuracy.high,
      distanceFilter: 10,
    );
    
    return Geolocator.getPositionStream(
      locationSettings: locationSettings,
    );
  }
  
  // คำนวณระยะทางระหว่าง 2 จุด (meters)
  static double calculateDistance({
    required double startLat,
    required double startLng,
    required double endLat,
    required double endLng,
  }) {
    return Geolocator.distanceBetween(
      startLat, startLng, endLat, endLng,
    );
  }
  
  // แปลง coordinates เป็นที่อยู่
  static Future<String> getAddressFromCoordinates({
    required double latitude,
    required double longitude,
  }) async {
    try {
      final placemarks = await placemarkFromCoordinates(
        latitude, longitude,
      );
      
      if (placemarks.isEmpty) return 'ไม่พบที่อยู่';
      
      final place = placemarks.first;
      final parts = [
        place.subThoroughfare,
        place.thoroughfare,
        place.subLocality,
        place.locality,
        place.administrativeArea,
        place.country,
      ].where((p) => p != null && p.isNotEmpty).toList();
      
      return parts.join(', ');
    } catch (e) {
      return 'ไม่สามารถหาที่อยู่ได้';
    }
  }
  
  // แปลงที่อยู่เป็น coordinates
  static Future<List<Location>> getCoordinatesFromAddress(
    String address,
  ) async {
    try {
      return await locationFromAddress(address);
    } catch (e) {
      return [];
    }
  }
}
```

---

## 45.3 Google Maps Flutter

### แสดงแผนที่พื้นฐาน

```dart
import 'package:google_maps_flutter/google_maps_flutter.dart';

class BasicMapPage extends StatefulWidget {
  const BasicMapPage({super.key});
  
  @override
  State<BasicMapPage> createState() => _BasicMapPageState();
}

class _BasicMapPageState extends State<BasicMapPage> {
  GoogleMapController? _mapController;
  
  static const _initialPosition = CameraPosition(
    target: LatLng(13.7563, 100.5018), // Bangkok
    zoom: 12,
  );
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('แผนที่')),
      body: GoogleMap(
        initialCameraPosition: _initialPosition,
        onMapCreated: (controller) => _mapController = controller,
        mapType: MapType.normal,
        myLocationEnabled: true,
        myLocationButtonEnabled: true,
        zoomControlsEnabled: true,
        compassEnabled: true,
        trafficEnabled: false,
      ),
    );
  }
}
```

### Custom Markers

```dart
class MapWithMarkers extends StatefulWidget {
  const MapWithMarkers({super.key});
  
  @override
  State<MapWithMarkers> createState() => _MapWithMarkersState();
}

class _MapWithMarkersState extends State<MapWithMarkers> {
  GoogleMapController? _mapController;
  final Set<Marker> _markers = {};
  final Map<MarkerId, MapLocation> _locations = {};
  
  @override
  void initState() {
    super.initState();
    _loadMarkers();
  }
  
  Future<void> _loadMarkers() async {
    final locations = [
      MapLocation(
        id: '1',
        name: 'สยามพารากอน',
        position: const LatLng(13.7467, 100.5332),
        category: LocationCategory.shopping,
      ),
      MapLocation(
        id: '2',
        name: 'มาบุญครอง',
        position: const LatLng(13.7444, 100.5301),
        category: LocationCategory.shopping,
      ),
      MapLocation(
        id: '3',
        name: 'โรงพยาบาลจุฬา',
        position: const LatLng(13.7348, 100.5340),
        category: LocationCategory.hospital,
      ),
    ];
    
    final Set<Marker> markers = {};
    
    for (final location in locations) {
      final icon = await _createCustomMarkerIcon(location.category);
      
      markers.add(
        Marker(
          markerId: MarkerId(location.id),
          position: location.position,
          icon: icon,
          infoWindow: InfoWindow(
            title: location.name,
            snippet: location.category.label,
            onTap: () => _onMarkerInfoTapped(location),
          ),
          onTap: () => _onMarkerTapped(location),
        ),
      );
      
      _locations[MarkerId(location.id)] = location;
    }
    
    setState(() => _markers.addAll(markers));
  }
  
  Future<BitmapDescriptor> _createCustomMarkerIcon(
    LocationCategory category,
  ) async {
    // ใช้ asset icon แทน default pin
    switch (category) {
      case LocationCategory.shopping:
        return BitmapDescriptor.fromAssetImage(
          const ImageConfiguration(size: Size(48, 48)),
          'assets/markers/shopping.png',
        );
      case LocationCategory.hospital:
        return BitmapDescriptor.fromAssetImage(
          const ImageConfiguration(size: Size(48, 48)),
          'assets/markers/hospital.png',
        );
      default:
        return BitmapDescriptor.defaultMarker;
    }
  }
  
  void _onMarkerTapped(MapLocation location) {
    // แสดง info หรือ animate ไปที่ marker
    _mapController?.animateCamera(
      CameraUpdate.newCameraPosition(
        CameraPosition(
          target: location.position,
          zoom: 15,
        ),
      ),
    );
  }
  
  void _onMarkerInfoTapped(MapLocation location) {
    showModalBottomSheet(
      context: context,
      builder: (context) => LocationDetailSheet(location: location),
    );
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('ค้นหาสถานที่')),
      body: GoogleMap(
        initialCameraPosition: const CameraPosition(
          target: LatLng(13.7467, 100.5332),
          zoom: 13,
        ),
        onMapCreated: (controller) => _mapController = controller,
        markers: _markers,
        myLocationEnabled: true,
        myLocationButtonEnabled: true,
      ),
    );
  }
}

// Location Model
class MapLocation {
  final String id;
  final String name;
  final LatLng position;
  final LocationCategory category;
  
  const MapLocation({
    required this.id,
    required this.name,
    required this.position,
    required this.category,
  });
}

enum LocationCategory {
  shopping('ห้างสรรพสินค้า'),
  hospital('โรงพยาบาล'),
  restaurant('ร้านอาหาร'),
  gas('ปั๊มน้ำมัน');
  
  final String label;
  const LocationCategory(this.label);
}
```

---

## 45.4 Polylines - เส้นทาง

```dart
import 'package:flutter_polyline_points/flutter_polyline_points.dart';

class RouteMap extends StatefulWidget {
  final LatLng origin;
  final LatLng destination;
  
  const RouteMap({
    super.key,
    required this.origin,
    required this.destination,
  });
  
  @override
  State<RouteMap> createState() => _RouteMapState();
}

class _RouteMapState extends State<RouteMap> {
  GoogleMapController? _mapController;
  final Set<Polyline> _polylines = {};
  final Set<Marker> _markers = {};
  
  @override
  void initState() {
    super.initState();
    _setupMarkersAndRoute();
  }
  
  Future<void> _setupMarkersAndRoute() async {
    // เพิ่ม markers จุดเริ่มต้นและปลายทาง
    setState(() {
      _markers.addAll([
        Marker(
          markerId: const MarkerId('origin'),
          position: widget.origin,
          icon: BitmapDescriptor.defaultMarkerWithHue(
            BitmapDescriptor.hueGreen,
          ),
          infoWindow: const InfoWindow(title: 'จุดเริ่มต้น'),
        ),
        Marker(
          markerId: const MarkerId('destination'),
          position: widget.destination,
          icon: BitmapDescriptor.defaultMarkerWithHue(
            BitmapDescriptor.hueRed,
          ),
          infoWindow: const InfoWindow(title: 'ปลายทาง'),
        ),
      ]);
    });
    
    // ดึง polyline points
    await _getPolylinePoints();
    
    // Fit camera ให้แสดงเส้นทางทั้งหมด
    _fitCameraToRoute();
  }
  
  Future<void> _getPolylinePoints() async {
    final polylinePoints = PolylinePoints();
    
    final result = await polylinePoints.getRouteBetweenCoordinates(
      googleApiKey: 'YOUR_API_KEY',
      request: PolylineRequest(
        origin: PointLatLng(
          widget.origin.latitude,
          widget.origin.longitude,
        ),
        destination: PointLatLng(
          widget.destination.latitude,
          widget.destination.longitude,
        ),
        mode: TravelMode.driving,
      ),
    );
    
    if (result.points.isNotEmpty) {
      final points = result.points
          .map((p) => LatLng(p.latitude, p.longitude))
          .toList();
      
      setState(() {
        _polylines.add(
          Polyline(
            polylineId: const PolylineId('route'),
            points: points,
            color: Colors.blue,
            width: 5,
            startCap: Cap.roundCap,
            endCap: Cap.roundCap,
            jointType: JointType.round,
          ),
        );
      });
    }
  }
  
  void _fitCameraToRoute() {
    if (_mapController == null) return;
    
    final bounds = LatLngBounds(
      southwest: LatLng(
        [widget.origin.latitude, widget.destination.latitude].reduce(min),
        [widget.origin.longitude, widget.destination.longitude].reduce(min),
      ),
      northeast: LatLng(
        [widget.origin.latitude, widget.destination.latitude].reduce(max),
        [widget.origin.longitude, widget.destination.longitude].reduce(max),
      ),
    );
    
    _mapController!.animateCamera(
      CameraUpdate.newLatLngBounds(bounds, 80),
    );
  }
  
  @override
  Widget build(BuildContext context) {
    return GoogleMap(
      initialCameraPosition: CameraPosition(
        target: widget.origin,
        zoom: 12,
      ),
      onMapCreated: (controller) {
        _mapController = controller;
        _fitCameraToRoute();
      },
      markers: _markers,
      polylines: _polylines,
      myLocationEnabled: true,
    );
  }
}
```

---

## 45.5 Workshop: Location Tracker App

```dart
// location_tracker_app.dart
class LocationTrackerPage extends StatefulWidget {
  const LocationTrackerPage({super.key});
  
  @override
  State<LocationTrackerPage> createState() => _LocationTrackerPageState();
}

class _LocationTrackerPageState extends State<LocationTrackerPage> {
  GoogleMapController? _mapController;
  Position? _currentPosition;
  String _currentAddress = '';
  bool _isTracking = false;
  StreamSubscription<Position>? _positionSubscription;
  
  final List<LatLng> _trackingPath = [];
  final Set<Polyline> _polylines = {};
  final Set<Marker> _markers = {};
  
  // Stats
  double _totalDistance = 0;
  DateTime? _startTime;
  
  @override
  void initState() {
    super.initState();
    _loadCurrentLocation();
  }
  
  Future<void> _loadCurrentLocation() async {
    final position = await LocationService.getCurrentPosition();
    if (position == null || !mounted) return;
    
    final address = await LocationService.getAddressFromCoordinates(
      latitude: position.latitude,
      longitude: position.longitude,
    );
    
    setState(() {
      _currentPosition = position;
      _currentAddress = address;
    });
    
    _mapController?.animateCamera(
      CameraUpdate.newCameraPosition(
        CameraPosition(
          target: LatLng(position.latitude, position.longitude),
          zoom: 15,
        ),
      ),
    );
    
    _updateCurrentMarker(position);
  }
  
  void _updateCurrentMarker(Position position) {
    final marker = Marker(
      markerId: const MarkerId('current'),
      position: LatLng(position.latitude, position.longitude),
      infoWindow: InfoWindow(
        title: 'ตำแหน่งของฉัน',
        snippet: _currentAddress,
      ),
      icon: BitmapDescriptor.defaultMarkerWithHue(
        BitmapDescriptor.hueAzure,
      ),
    );
    
    setState(() {
      _markers.removeWhere(
        (m) => m.markerId == const MarkerId('current'),
      );
      _markers.add(marker);
    });
  }
  
  void _startTracking() {
    setState(() {
      _isTracking = true;
      _startTime = DateTime.now();
      _trackingPath.clear();
      _totalDistance = 0;
    });
    
    _positionSubscription = LocationService.getPositionStream(
      distanceFilter: 5,
    ).listen((position) {
      final newPoint = LatLng(position.latitude, position.longitude);
      
      if (_trackingPath.isNotEmpty) {
        final lastPoint = _trackingPath.last;
        _totalDistance += Geolocator.distanceBetween(
          lastPoint.latitude,
          lastPoint.longitude,
          newPoint.latitude,
          newPoint.longitude,
        );
      }
      
      _trackingPath.add(newPoint);
      
      // อัปเดต polyline
      setState(() {
        _currentPosition = position;
        _polylines.clear();
        _polylines.add(
          Polyline(
            polylineId: const PolylineId('tracking'),
            points: List.from(_trackingPath),
            color: Colors.blue,
            width: 4,
          ),
        );
      });
      
      // เลื่อนกล้องตาม
      _mapController?.animateCamera(
        CameraUpdate.newLatLng(newPoint),
      );
      
      _updateCurrentMarker(position);
    });
  }
  
  Future<void> _stopTracking() async {
    await _positionSubscription?.cancel();
    _positionSubscription = null;
    
    if (_trackingPath.isNotEmpty) {
      // เพิ่ม start และ end markers
      setState(() {
        _isTracking = false;
        _markers.add(Marker(
          markerId: const MarkerId('start'),
          position: _trackingPath.first,
          icon: BitmapDescriptor.defaultMarkerWithHue(
            BitmapDescriptor.hueGreen,
          ),
          infoWindow: const InfoWindow(title: 'จุดเริ่มต้น'),
        ));
        _markers.add(Marker(
          markerId: const MarkerId('end'),
          position: _trackingPath.last,
          icon: BitmapDescriptor.defaultMarkerWithHue(
            BitmapDescriptor.hueRed,
          ),
          infoWindow: const InfoWindow(title: 'จุดสิ้นสุด'),
        ));
      });
      
      _showTrackingSummary();
    } else {
      setState(() => _isTracking = false);
    }
  }
  
  void _showTrackingSummary() {
    final duration = DateTime.now().difference(_startTime!);
    final distanceKm = _totalDistance / 1000;
    final speedKmh = distanceKm / (duration.inSeconds / 3600);
    
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('สรุปการติดตาม'),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            _StatRow(
              icon: Icons.straighten,
              label: 'ระยะทาง',
              value: '${distanceKm.toStringAsFixed(2)} km',
            ),
            _StatRow(
              icon: Icons.timer,
              label: 'เวลา',
              value: _formatDuration(duration),
            ),
            _StatRow(
              icon: Icons.speed,
              label: 'ความเร็วเฉลี่ย',
              value: '${speedKmh.toStringAsFixed(1)} km/h',
            ),
          ],
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('ตกลง'),
          ),
        ],
      ),
    );
  }
  
  String _formatDuration(Duration duration) {
    final hours = duration.inHours;
    final minutes = duration.inMinutes % 60;
    final seconds = duration.inSeconds % 60;
    
    if (hours > 0) return '$hours:${minutes.toString().padLeft(2, '0')} ชั่วโมง';
    if (minutes > 0) return '$minutes:${seconds.toString().padLeft(2, '0')} นาที';
    return '$seconds วินาที';
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Stack(
        children: [
          // Google Map
          GoogleMap(
            initialCameraPosition: const CameraPosition(
              target: LatLng(13.7563, 100.5018),
              zoom: 14,
            ),
            onMapCreated: (controller) {
              _mapController = controller;
              if (_currentPosition != null) {
                controller.animateCamera(
                  CameraUpdate.newLatLng(
                    LatLng(
                      _currentPosition!.latitude,
                      _currentPosition!.longitude,
                    ),
                  ),
                );
              }
            },
            markers: _markers,
            polylines: _polylines,
            myLocationEnabled: false,
            zoomControlsEnabled: false,
          ),
          
          // Info Panel
          Positioned(
            top: MediaQuery.of(context).padding.top + 16,
            left: 16,
            right: 16,
            child: Card(
              child: Padding(
                padding: const EdgeInsets.all(16),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    if (_currentAddress.isNotEmpty)
                      Row(
                        children: [
                          const Icon(
                            Icons.location_on,
                            color: Colors.red,
                            size: 16,
                          ),
                          const SizedBox(width: 4),
                          Expanded(
                            child: Text(
                              _currentAddress,
                              style: const TextStyle(fontSize: 12),
                              maxLines: 2,
                              overflow: TextOverflow.ellipsis,
                            ),
                          ),
                        ],
                      ),
                    if (_isTracking) ...[
                      const SizedBox(height: 8),
                      Row(
                        mainAxisAlignment: MainAxisAlignment.spaceAround,
                        children: [
                          _MiniStat(
                            label: 'ระยะทาง',
                            value: '${(_totalDistance / 1000).toStringAsFixed(2)} km',
                          ),
                          _MiniStat(
                            label: 'จุดที่บันทึก',
                            value: '${_trackingPath.length}',
                          ),
                        ],
                      ),
                    ],
                  ],
                ),
              ),
            ),
          ),
          
          // FAB Controls
          Positioned(
            bottom: 24,
            right: 16,
            child: Column(
              mainAxisSize: MainAxisSize.min,
              children: [
                FloatingActionButton(
                  heroTag: 'locate',
                  onPressed: _loadCurrentLocation,
                  mini: true,
                  child: const Icon(Icons.my_location),
                ),
                const SizedBox(height: 8),
                FloatingActionButton.extended(
                  heroTag: 'track',
                  onPressed: _isTracking ? _stopTracking : _startTracking,
                  backgroundColor: _isTracking ? Colors.red : Colors.blue,
                  icon: Icon(_isTracking ? Icons.stop : Icons.play_arrow),
                  label: Text(_isTracking ? 'หยุดติดตาม' : 'เริ่มติดตาม'),
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
  
  @override
  void dispose() {
    _positionSubscription?.cancel();
    _mapController?.dispose();
    super.dispose();
  }
}

class _StatRow extends StatelessWidget {
  final IconData icon;
  final String label;
  final String value;
  
  const _StatRow({
    required this.icon,
    required this.label,
    required this.value,
  });
  
  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Row(
        children: [
          Icon(icon, size: 20, color: Colors.blue),
          const SizedBox(width: 8),
          Text('$label: '),
          Text(value, style: const TextStyle(fontWeight: FontWeight.bold)),
        ],
      ),
    );
  }
}

class _MiniStat extends StatelessWidget {
  final String label;
  final String value;
  
  const _MiniStat({required this.label, required this.value});
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text(
          value,
          style: const TextStyle(
            fontWeight: FontWeight.bold,
            fontSize: 16,
          ),
        ),
        Text(label, style: const TextStyle(fontSize: 11, color: Colors.grey)),
      ],
    );
  }
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- การใช้ `geolocator` เพื่อดึงตำแหน่ง GPS
- การแสดงแผนที่ด้วย `google_maps_flutter`
- การสร้าง Custom Markers
- การวาดเส้นทาง (Polylines)
- Workshop: Location Tracker App แบบสมบูรณ์

**แบบฝึกหัดเพิ่มเติม:**
1. เพิ่ม geofencing (แจ้งเตือนเมื่อเข้า/ออกพื้นที่)
2. Implement street view
3. สร้าง heatmap จากตำแหน่งที่บันทึก
4. เพิ่ม turn-by-turn navigation
