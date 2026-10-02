# Part 75: Bluetooth and IoT - บลูทูธและ IoT

## บทนำ

Flutter สามารถเชื่อมต่อกับอุปกรณ์ IoT ผ่าน Bluetooth Low Energy (BLE) ได้อย่างง่ายดาย ด้วย `flutter_blue_plus` package เราสามารถ scan, connect และรับส่งข้อมูลกับอุปกรณ์ BLE ต่างๆ เช่น เซนเซอร์ wearables และ smart home devices

## 1. flutter_blue_plus

### การติดตั้ง

```yaml
# pubspec.yaml
dependencies:
  flutter_blue_plus: ^1.31.9
  permission_handler: ^11.3.1
```

### Android Configuration

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
  
  <!-- Bluetooth permissions -->
  <uses-permission android:name="android.permission.BLUETOOTH_SCAN"
    android:usesPermissionFlags="neverForLocation" />
  <uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
  <uses-permission android:name="android.permission.BLUETOOTH_ADVERTISE" />
  
  <!-- Location permission สำหรับ Android < 12 -->
  <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
  <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

  <!-- Bluetooth feature -->
  <uses-feature android:name="android.hardware.bluetooth_le" android:required="true" />
  
  <application ...>
  </application>
</manifest>
```

### iOS Configuration

```xml
<!-- ios/Runner/Info.plist -->
<dict>
  <key>NSBluetoothAlwaysUsageDescription</key>
  <string>แอปนี้ใช้ Bluetooth เพื่อเชื่อมต่อกับอุปกรณ์ IoT</string>
  <key>NSBluetoothPeripheralUsageDescription</key>
  <string>แอปนี้ใช้ Bluetooth เพื่อสื่อสารกับอุปกรณ์</string>
</dict>
```

## 2. BLE Scanning and Connecting

### Permission Manager

```dart
// lib/ble/ble_permissions.dart
import 'package:permission_handler/permission_handler.dart';
import 'dart:io';

class BlePermissions {
  static Future<bool> requestPermissions() async {
    if (Platform.isAndroid) {
      // Android 12+ ต้องการ BLUETOOTH_SCAN และ BLUETOOTH_CONNECT
      final results = await [
        Permission.bluetoothScan,
        Permission.bluetoothConnect,
        Permission.locationWhenInUse,
      ].request();

      return results.values.every(
        (status) => status == PermissionStatus.granted,
      );
    }

    if (Platform.isIOS) {
      final status = await Permission.bluetooth.request();
      return status == PermissionStatus.granted;
    }

    return true;
  }

  static Future<bool> checkPermissions() async {
    if (Platform.isAndroid) {
      return await Permission.bluetoothScan.isGranted &&
          await Permission.bluetoothConnect.isGranted;
    }

    if (Platform.isIOS) {
      return await Permission.bluetooth.isGranted;
    }

    return true;
  }
}
```

### BLE Scanner

```dart
// lib/ble/ble_scanner.dart
import 'dart:async';
import 'package:flutter_blue_plus/flutter_blue_plus.dart';

class BleDevice {
  final BluetoothDevice device;
  final int rssi;
  final List<Guid> serviceUuids;
  final String? localName;

  const BleDevice({
    required this.device,
    required this.rssi,
    required this.serviceUuids,
    this.localName,
  });

  String get name =>
      localName ?? device.platformName.isEmpty
          ? 'Unknown Device'
          : device.platformName;

  String get id => device.remoteId.str;

  bool get hasName => name != 'Unknown Device';
}

class BleScanner {
  final _devicesController = StreamController<List<BleDevice>>.broadcast();
  final _devices = <String, BleDevice>{};
  StreamSubscription? _scanSubscription;
  bool _isScanning = false;

  Stream<List<BleDevice>> get devicesStream => _devicesController.stream;
  bool get isScanning => _isScanning;

  Future<void> startScan({
    Duration timeout = const Duration(seconds: 10),
    List<Guid>? withServices,
  }) async {
    if (_isScanning) return;

    _devices.clear();
    _isScanning = true;

    try {
      // เริ่ม scan
      await FlutterBluePlus.startScan(
        timeout: timeout,
        withServices: withServices ?? [],
      );

      // ฟัง scan results
      _scanSubscription = FlutterBluePlus.scanResults.listen((results) {
        for (final result in results) {
          final device = BleDevice(
            device: result.device,
            rssi: result.rssi,
            serviceUuids: result.advertisementData.serviceUuids,
            localName: result.advertisementData.localName.isEmpty
                ? null
                : result.advertisementData.localName,
          );

          _devices[device.id] = device;
        }

        _devicesController.add(_devices.values.toList()
          ..sort((a, b) => b.rssi.compareTo(a.rssi)));
      });

      // รอให้ scan เสร็จ
      await Future.delayed(timeout);
    } finally {
      _isScanning = false;
      await stopScan();
    }
  }

  Future<void> stopScan() async {
    await FlutterBluePlus.stopScan();
    await _scanSubscription?.cancel();
    _scanSubscription = null;
  }

  void dispose() {
    _scanSubscription?.cancel();
    _devicesController.close();
  }
}
```

### BLE Connection Manager

```dart
// lib/ble/ble_connection.dart
import 'dart:async';
import 'package:flutter_blue_plus/flutter_blue_plus.dart';

enum ConnectionState {
  disconnected,
  connecting,
  connected,
  disconnecting,
}

class BleConnection {
  final BluetoothDevice device;
  
  final _stateController = StreamController<ConnectionState>.broadcast();
  StreamSubscription? _stateSubscription;
  
  ConnectionState _state = ConnectionState.disconnected;
  List<BluetoothService>? _services;

  BleConnection({required this.device});

  Stream<ConnectionState> get stateStream => _stateController.stream;
  ConnectionState get state => _state;
  List<BluetoothService>? get services => _services;

  bool get isConnected => _state == ConnectionState.connected;

  Future<void> connect() async {
    if (_state == ConnectionState.connected) return;

    _updateState(ConnectionState.connecting);

    try {
      await device.connect(
        timeout: const Duration(seconds: 10),
        autoConnect: false,
      );

      // ฟัง connection state changes
      _stateSubscription = device.connectionState.listen((state) {
        if (state == BluetoothConnectionState.connected) {
          _updateState(ConnectionState.connected);
        } else if (state == BluetoothConnectionState.disconnected) {
          _updateState(ConnectionState.disconnected);
          _services = null;
        }
      });

      // Discover services
      _services = await device.discoverServices();
      _updateState(ConnectionState.connected);
    } catch (e) {
      _updateState(ConnectionState.disconnected);
      rethrow;
    }
  }

  Future<void> disconnect() async {
    _updateState(ConnectionState.disconnecting);
    await device.disconnect();
    await _stateSubscription?.cancel();
    _services = null;
  }

  BluetoothService? getService(String uuid) {
    return _services?.firstWhere(
      (s) => s.uuid.str.toLowerCase() == uuid.toLowerCase(),
      orElse: () => throw Exception('Service $uuid not found'),
    );
  }

  BluetoothCharacteristic? getCharacteristic(
    String serviceUuid,
    String charUuid,
  ) {
    final service = getService(serviceUuid);
    return service?.characteristics.firstWhere(
      (c) => c.uuid.str.toLowerCase() == charUuid.toLowerCase(),
      orElse: () => throw Exception('Characteristic $charUuid not found'),
    );
  }

  void _updateState(ConnectionState newState) {
    _state = newState;
    _stateController.add(newState);
  }

  void dispose() {
    _stateSubscription?.cancel();
    _stateController.close();
  }
}
```

## 3. Reading/Writing Characteristics

### BLE Data Handler

```dart
// lib/ble/ble_data_handler.dart
import 'dart:async';
import 'dart:typed_data';
import 'package:flutter_blue_plus/flutter_blue_plus.dart';

class BleDataHandler {
  final BluetoothCharacteristic characteristic;
  
  StreamSubscription? _notifySubscription;
  final _dataController = StreamController<List<int>>.broadcast();

  BleDataHandler({required this.characteristic});

  Stream<List<int>> get dataStream => _dataController.stream;

  // อ่านค่า (สำหรับ readable characteristics)
  Future<List<int>> read() async {
    return await characteristic.read();
  }

  // เขียนค่า
  Future<void> write(List<int> data, {bool withResponse = true}) async {
    await characteristic.write(
      data,
      withoutResponse: !withResponse,
    );
  }

  // เขียน string
  Future<void> writeString(String text, {bool withResponse = true}) async {
    final bytes = text.codeUnits;
    await write(bytes, withResponse: withResponse);
  }

  // เขียน integer (2 bytes, big-endian)
  Future<void> writeInt16(int value) async {
    final bytes = ByteData(2);
    bytes.setInt16(0, value, Endian.big);
    await write(bytes.buffer.asUint8List().toList());
  }

  // Subscribe to notifications
  Future<void> startNotifications() async {
    if (!characteristic.properties.notify &&
        !characteristic.properties.indicate) {
      throw Exception('Characteristic does not support notifications');
    }

    await characteristic.setNotifyValue(true);

    _notifySubscription = characteristic.lastValueStream.listen((data) {
      if (data.isNotEmpty) {
        _dataController.add(data);
      }
    });
  }

  Future<void> stopNotifications() async {
    await characteristic.setNotifyValue(false);
    await _notifySubscription?.cancel();
    _notifySubscription = null;
  }

  void dispose() {
    _notifySubscription?.cancel();
    _dataController.close();
  }
}

// Helper functions สำหรับแปลง data
class BleDataConverter {
  // แปลง bytes เป็น float temperature
  static double bytesToTemperature(List<int> bytes) {
    if (bytes.length < 2) return 0;
    final value = (bytes[0] << 8) | bytes[1];
    return value / 100.0; // สมมติว่า เก็บเป็น °C * 100
  }

  // แปลง bytes เป็น humidity
  static double bytesToHumidity(List<int> bytes) {
    if (bytes.isEmpty) return 0;
    return bytes[0].toDouble();
  }

  // แปลง bytes เป็น battery level
  static int bytesToBatteryLevel(List<int> bytes) {
    if (bytes.isEmpty) return 0;
    return bytes[0];
  }

  // แปลง bytes เป็น string
  static String bytesToString(List<int> bytes) {
    return String.fromCharCodes(bytes);
  }
}
```

## 4. Workshop: BLE Temperature Sensor

### Temperature Sensor Service

```dart
// lib/ble/temperature_sensor.dart
import 'dart:async';
import 'package:flutter_blue_plus/flutter_blue_plus.dart';
import 'ble_connection.dart';
import 'ble_data_handler.dart';

// UUIDs สำหรับ Environmental Sensing Service (มาตรฐาน BLE)
class EnvironmentalSensingUUIDs {
  static const serviceUuid = '0000181a-0000-1000-8000-00805f9b34fb';
  static const temperatureUuid = '00002a6e-0000-1000-8000-00805f9b34fb';
  static const humidityUuid = '00002a6f-0000-1000-8000-00805f9b34fb';
  static const pressureUuid = '00002a6d-0000-1000-8000-00805f9b34fb';
}

class TemperatureReading {
  final double temperature;
  final double? humidity;
  final DateTime timestamp;

  const TemperatureReading({
    required this.temperature,
    this.humidity,
    required this.timestamp,
  });
}

class TemperatureSensorService {
  final BleConnection _connection;
  BleDataHandler? _tempHandler;
  BleDataHandler? _humidityHandler;

  final _readingsController =
      StreamController<TemperatureReading>.broadcast();
  
  final List<TemperatureReading> _history = [];
  static const int _maxHistorySize = 100;

  TemperatureSensorService({required BleConnection connection})
      : _connection = connection;

  Stream<TemperatureReading> get readingsStream => _readingsController.stream;
  List<TemperatureReading> get history => List.unmodifiable(_history);

  Future<void> initialize() async {
    if (!_connection.isConnected) {
      throw Exception('BLE device not connected');
    }

    try {
      // Setup temperature characteristic
      final tempChar = _connection.getCharacteristic(
        EnvironmentalSensingUUIDs.serviceUuid,
        EnvironmentalSensingUUIDs.temperatureUuid,
      );

      if (tempChar != null) {
        _tempHandler = BleDataHandler(characteristic: tempChar);
        await _tempHandler!.startNotifications();

        _tempHandler!.dataStream.listen((data) {
          final temp = BleDataConverter.bytesToTemperature(data);
          _addReading(temperature: temp);
        });
      }

      // Setup humidity characteristic (optional)
      try {
        final humChar = _connection.getCharacteristic(
          EnvironmentalSensingUUIDs.serviceUuid,
          EnvironmentalSensingUUIDs.humidityUuid,
        );

        if (humChar != null) {
          _humidityHandler = BleDataHandler(characteristic: humChar);
          await _humidityHandler!.startNotifications();
        }
      } catch (_) {
        // Humidity not available
      }
    } catch (e) {
      rethrow;
    }
  }

  void _addReading({required double temperature, double? humidity}) {
    final reading = TemperatureReading(
      temperature: temperature,
      humidity: humidity,
      timestamp: DateTime.now(),
    );

    _history.add(reading);
    if (_history.length > _maxHistorySize) {
      _history.removeAt(0);
    }

    _readingsController.add(reading);
  }

  // อ่านค่าทันที (poll)
  Future<TemperatureReading> readOnce() async {
    final tempData = await _tempHandler?.read() ?? [];
    double? humidity;

    if (_humidityHandler != null) {
      final humData = await _humidityHandler!.read();
      humidity = BleDataConverter.bytesToHumidity(humData);
    }

    final reading = TemperatureReading(
      temperature: BleDataConverter.bytesToTemperature(tempData),
      humidity: humidity,
      timestamp: DateTime.now(),
    );

    _addReading(
      temperature: reading.temperature,
      humidity: reading.humidity,
    );

    return reading;
  }

  void dispose() {
    _tempHandler?.dispose();
    _humidityHandler?.dispose();
    _readingsController.close();
  }
}
```

### Temperature Monitor Screen

```dart
// lib/screens/temperature_monitor.dart
import 'dart:async';
import 'package:flutter/material.dart';
import '../ble/ble_scanner.dart';
import '../ble/ble_connection.dart';
import '../ble/ble_permissions.dart';
import '../ble/temperature_sensor.dart';

class TemperatureMonitorApp extends StatelessWidget {
  const TemperatureMonitorApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'BLE Temperature Monitor',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
        useMaterial3: true,
      ),
      home: const DeviceScanScreen(),
    );
  }
}

class DeviceScanScreen extends StatefulWidget {
  const DeviceScanScreen({super.key});

  @override
  State<DeviceScanScreen> createState() => _DeviceScanScreenState();
}

class _DeviceScanScreenState extends State<DeviceScanScreen> {
  final _scanner = BleScanner();
  bool _isScanning = false;
  List<BleDevice> _devices = [];

  @override
  void initState() {
    super.initState();
    _scanner.devicesStream.listen((devices) {
      if (mounted) setState(() => _devices = devices);
    });
  }

  Future<void> _startScan() async {
    final hasPermission = await BlePermissions.requestPermissions();
    if (!hasPermission) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('ต้องการสิทธิ์ Bluetooth และ Location')),
        );
      }
      return;
    }

    setState(() {
      _isScanning = true;
      _devices = [];
    });

    await _scanner.startScan(timeout: const Duration(seconds: 10));

    if (mounted) setState(() => _isScanning = false);
  }

  void _connectToDevice(BleDevice bleDevice) {
    Navigator.push(
      context,
      MaterialPageRoute(
        builder: (context) => TemperatureScreen(bleDevice: bleDevice),
      ),
    );
  }

  @override
  void dispose() {
    _scanner.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('BLE Devices'),
        actions: [
          if (_isScanning)
            const Padding(
              padding: EdgeInsets.all(8),
              child: SizedBox(
                width: 20,
                height: 20,
                child: CircularProgressIndicator(strokeWidth: 2),
              ),
            ),
        ],
      ),
      body: Column(
        children: [
          // Scan button
          Padding(
            padding: const EdgeInsets.all(16),
            child: FilledButton.icon(
              onPressed: _isScanning ? null : _startScan,
              icon: const Icon(Icons.bluetooth_searching),
              label: Text(_isScanning ? 'กำลังค้นหา...' : 'ค้นหาอุปกรณ์'),
            ),
          ),

          // Device list
          Expanded(
            child: _devices.isEmpty
                ? Center(
                    child: Column(
                      mainAxisSize: MainAxisSize.min,
                      children: [
                        const Icon(
                          Icons.bluetooth,
                          size: 64,
                          color: Colors.grey,
                        ),
                        const SizedBox(height: 16),
                        Text(
                          _isScanning
                              ? 'กำลังค้นหาอุปกรณ์...'
                              : 'กด "ค้นหาอุปกรณ์" เพื่อเริ่ม',
                          style: const TextStyle(color: Colors.grey),
                        ),
                      ],
                    ),
                  )
                : ListView.builder(
                    itemCount: _devices.length,
                    itemBuilder: (context, index) {
                      final device = _devices[index];
                      return ListTile(
                        leading: const Icon(Icons.bluetooth),
                        title: Text(device.name),
                        subtitle: Text(device.id),
                        trailing: Text(
                          '${device.rssi} dBm',
                          style: TextStyle(
                            color: _rssiColor(device.rssi),
                          ),
                        ),
                        onTap: () => _connectToDevice(device),
                      );
                    },
                  ),
          ),
        ],
      ),
    );
  }

  Color _rssiColor(int rssi) {
    if (rssi > -60) return Colors.green;
    if (rssi > -80) return Colors.orange;
    return Colors.red;
  }
}

class TemperatureScreen extends StatefulWidget {
  final BleDevice bleDevice;

  const TemperatureScreen({super.key, required this.bleDevice});

  @override
  State<TemperatureScreen> createState() => _TemperatureScreenState();
}

class _TemperatureScreenState extends State<TemperatureScreen> {
  late BleConnection _connection;
  TemperatureSensorService? _sensorService;
  TemperatureReading? _latestReading;
  String _status = 'กำลังเชื่อมต่อ...';

  @override
  void initState() {
    super.initState();
    _connection = BleConnection(device: widget.bleDevice.device);
    _connect();
  }

  Future<void> _connect() async {
    try {
      await _connection.connect();

      _sensorService = TemperatureSensorService(connection: _connection);
      await _sensorService!.initialize();

      _sensorService!.readingsStream.listen((reading) {
        if (mounted) {
          setState(() {
            _latestReading = reading;
            _status = 'เชื่อมต่อแล้ว';
          });
        }
      });

      if (mounted) setState(() => _status = 'รอข้อมูล...');
    } catch (e) {
      if (mounted) setState(() => _status = 'เชื่อมต่อไม่สำเร็จ: $e');
    }
  }

  @override
  void dispose() {
    _sensorService?.dispose();
    _connection.disconnect();
    _connection.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.bleDevice.name),
        actions: [
          IconButton(
            icon: const Icon(Icons.info_outline),
            onPressed: () => _showDeviceInfo(),
          ),
        ],
      ),
      body: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          children: [
            // Status
            Container(
              padding: const EdgeInsets.symmetric(
                horizontal: 16,
                vertical: 8,
              ),
              decoration: BoxDecoration(
                color: _statusColor().withOpacity(0.1),
                borderRadius: BorderRadius.circular(20),
                border: Border.all(color: _statusColor()),
              ),
              child: Row(
                mainAxisSize: MainAxisSize.min,
                children: [
                  Icon(_statusIcon(), color: _statusColor(), size: 16),
                  const SizedBox(width: 8),
                  Text(_status, style: TextStyle(color: _statusColor())),
                ],
              ),
            ),

            const SizedBox(height: 48),

            // Temperature display
            if (_latestReading != null) ...[
              const Text(
                'อุณหภูมิ',
                style: TextStyle(fontSize: 18, color: Colors.grey),
              ),
              Text(
                '${_latestReading!.temperature.toStringAsFixed(1)}°C',
                style: TextStyle(
                  fontSize: 80,
                  fontWeight: FontWeight.bold,
                  color: _temperatureColor(_latestReading!.temperature),
                ),
              ),

              if (_latestReading!.humidity != null) ...[
                const SizedBox(height: 24),
                const Text(
                  'ความชื้น',
                  style: TextStyle(fontSize: 18, color: Colors.grey),
                ),
                Text(
                  '${_latestReading!.humidity!.toStringAsFixed(0)}%',
                  style: const TextStyle(
                    fontSize: 48,
                    fontWeight: FontWeight.w300,
                  ),
                ),
              ],

              const SizedBox(height: 16),
              Text(
                'อัปเดตล่าสุด: ${_formatTime(_latestReading!.timestamp)}',
                style: const TextStyle(color: Colors.grey),
              ),
            ] else
              const CircularProgressIndicator(),
          ],
        ),
      ),
    );
  }

  Color _statusColor() {
    if (_status.contains('เชื่อมต่อแล้ว') || _status.contains('รอข้อมูล')) {
      return Colors.green;
    }
    if (_status.contains('กำลัง')) return Colors.orange;
    return Colors.red;
  }

  IconData _statusIcon() {
    if (_status.contains('เชื่อมต่อแล้ว')) return Icons.bluetooth_connected;
    if (_status.contains('กำลัง')) return Icons.bluetooth_searching;
    return Icons.bluetooth_disabled;
  }

  Color _temperatureColor(double temp) {
    if (temp < 20) return Colors.blue;
    if (temp < 30) return Colors.green;
    if (temp < 35) return Colors.orange;
    return Colors.red;
  }

  String _formatTime(DateTime time) {
    return '${time.hour.toString().padLeft(2, '0')}:'
        '${time.minute.toString().padLeft(2, '0')}:'
        '${time.second.toString().padLeft(2, '0')}';
  }

  void _showDeviceInfo() {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('ข้อมูลอุปกรณ์'),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text('ชื่อ: ${widget.bleDevice.name}'),
            Text('ID: ${widget.bleDevice.id}'),
            Text('RSSI: ${widget.bleDevice.rssi} dBm'),
          ],
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('ปิด'),
          ),
        ],
      ),
    );
  }
}
```

## สรุป

BLE และ IoT ใน Flutter:
1. **flutter_blue_plus** เป็น package ที่ดีที่สุดสำหรับ BLE
2. **ต้องขอ permission** บน Android และ iOS
3. **UUID** ใช้ระบุ services และ characteristics
4. **Notifications** ดีกว่า polling สำหรับข้อมูล real-time

## แบบทดสอบ

1. อธิบายความแตกต่างระหว่าง BLE Peripheral และ Central
2. ทำไม `withoutResponse: true` ถึงเร็วกว่า แต่มีข้อเสียอะไร?
3. เพิ่มกราฟแสดงประวัติอุณหภูมิโดยใช้ `fl_chart`
4. สร้าง reconnection logic เมื่อ BLE disconnects โดยไม่คาดคิด
