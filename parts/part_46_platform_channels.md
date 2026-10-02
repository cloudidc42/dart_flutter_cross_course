# Part 46: Platform Channels

## บทนำ

Platform Channels คือกลไกที่ Flutter ใช้สื่อสารกับ Native Code (Android Kotlin/Java และ iOS Swift/Obj-C) ใช้เมื่อต้องการเข้าถึง platform-specific APIs ที่ยังไม่มี Flutter plugin รองรับ หรือต้องการ performance สูงสุดจาก native code

---

## 46.1 ประเภทของ Platform Channels

### 1. MethodChannel - เรียกใช้ฟังก์ชัน

ใช้สำหรับการเรียกฟังก์ชันแบบ request-response เหมาะสำหรับการดึงข้อมูลหรือทำงานครั้งเดียว

```
Flutter (Dart)          Native (Kotlin/Swift)
    |                         |
    |--- invokeMethod() ----->|
    |                         | (ทำงาน)
    |<------ result ----------|
```

### 2. EventChannel - รับ Stream ต่อเนื่อง

ใช้สำหรับรับข้อมูลแบบ stream จาก native เช่น sensor data, network status

```
Flutter (Dart)          Native (Kotlin/Swift)
    |                         |
    |--- listen() ----------->|
    |<------ events ----------| (ส่งซ้ำๆ)
    |                         |
    |--- cancel() ----------->|
```

### 3. BasicMessageChannel - ส่งข้อความสองทาง

ใช้สำหรับการสื่อสารสองทางแบบเรียบง่าย

---

## 46.2 MethodChannel

### Dart Side

```dart
import 'package:flutter/services.dart';

class NativeService {
  // สร้าง channel (ชื่อต้องตรงกับ native)
  static const MethodChannel _channel = MethodChannel(
    'com.example.myapp/native',
  );
  
  // เรียก method บน native
  static Future<int> getBatteryLevel() async {
    try {
      final int batteryLevel = await _channel.invokeMethod('getBatteryLevel');
      return batteryLevel;
    } on PlatformException catch (e) {
      print('เกิดข้อผิดพลาด: ${e.message}');
      return -1;
    }
  }
  
  // ส่ง arguments ไป native
  static Future<String> getDeviceInfo({bool detailed = false}) async {
    try {
      final String info = await _channel.invokeMethod(
        'getDeviceInfo',
        {'detailed': detailed},
      );
      return info;
    } on PlatformException catch (e) {
      return 'Error: ${e.message}';
    }
  }
  
  // เรียก method ที่ return Map
  static Future<Map<String, dynamic>?> getSystemInfo() async {
    try {
      final Map<Object?, Object?> result = await _channel.invokeMethod(
        'getSystemInfo',
      );
      return Map<String, dynamic>.from(result);
    } on PlatformException catch (e) {
      print('Error: ${e.message}');
      return null;
    }
  }
  
  // เรียก method โดยไม่ต้องรอ result
  static Future<void> showNativeToast(String message) async {
    await _channel.invokeMethod('showToast', {'message': message});
  }
  
  // Handle calls จาก native มา Dart
  static void setupMethodCallHandler() {
    _channel.setMethodCallHandler((call) async {
      switch (call.method) {
        case 'onNativeEvent':
          final data = call.arguments as Map;
          print('Native event: $data');
          return 'ok';
        default:
          throw PlatformException(
            code: 'NOT_IMPLEMENTED',
            message: 'Method ${call.method} ไม่รองรับ',
          );
      }
    });
  }
}
```

### Android - Kotlin Side

```kotlin
// android/app/src/main/kotlin/com/example/myapp/MainActivity.kt
package com.example.myapp

import android.content.Context
import android.content.Intent
import android.content.IntentFilter
import android.os.BatteryManager
import android.os.Build
import android.widget.Toast
import io.flutter.embedding.android.FlutterActivity
import io.flutter.embedding.engine.FlutterEngine
import io.flutter.plugin.common.MethodChannel

class MainActivity : FlutterActivity() {
    private val CHANNEL = "com.example.myapp/native"
    
    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)
        
        MethodChannel(
            flutterEngine.dartExecutor.binaryMessenger,
            CHANNEL
        ).setMethodCallHandler { call, result ->
            when (call.method) {
                "getBatteryLevel" -> {
                    val batteryLevel = getBatteryLevel()
                    if (batteryLevel != -1) {
                        result.success(batteryLevel)
                    } else {
                        result.error(
                            "UNAVAILABLE",
                            "ไม่สามารถดึงระดับแบตเตอรี่ได้",
                            null
                        )
                    }
                }
                
                "getDeviceInfo" -> {
                    val detailed = call.argument<Boolean>("detailed") ?: false
                    val info = getDeviceInfo(detailed)
                    result.success(info)
                }
                
                "getSystemInfo" -> {
                    val info = getSystemInfo()
                    result.success(info)
                }
                
                "showToast" -> {
                    val message = call.argument<String>("message") ?: ""
                    Toast.makeText(context, message, Toast.LENGTH_SHORT).show()
                    result.success(null)
                }
                
                else -> result.notImplemented()
            }
        }
    }
    
    private fun getBatteryLevel(): Int {
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            val batteryManager = getSystemService(Context.BATTERY_SERVICE) 
                as BatteryManager
            batteryManager.getIntProperty(
                BatteryManager.BATTERY_PROPERTY_CAPACITY
            )
        } else {
            val intent = registerReceiver(
                null,
                IntentFilter(Intent.ACTION_BATTERY_CHANGED)
            )
            val level = intent?.getIntExtra(BatteryManager.EXTRA_LEVEL, -1) ?: -1
            val scale = intent?.getIntExtra(BatteryManager.EXTRA_SCALE, -1) ?: -1
            (level * 100 / scale.toFloat()).toInt()
        }
    }
    
    private fun getDeviceInfo(detailed: Boolean): String {
        return if (detailed) {
            "Model: ${Build.MODEL}\n" +
            "Brand: ${Build.BRAND}\n" +
            "Android: ${Build.VERSION.RELEASE}\n" +
            "SDK: ${Build.VERSION.SDK_INT}"
        } else {
            "${Build.BRAND} ${Build.MODEL}"
        }
    }
    
    private fun getSystemInfo(): Map<String, Any> {
        return mapOf(
            "model" to Build.MODEL,
            "brand" to Build.BRAND,
            "androidVersion" to Build.VERSION.RELEASE,
            "sdkVersion" to Build.VERSION.SDK_INT,
            "manufacturer" to Build.MANUFACTURER,
            "isEmulator" to (Build.FINGERPRINT.startsWith("generic") || 
                           Build.FINGERPRINT.startsWith("unknown"))
        )
    }
}
```

### iOS - Swift Side

```swift
// ios/Runner/AppDelegate.swift
import UIKit
import Flutter

@UIApplicationMain
@objc class AppDelegate: FlutterAppDelegate {
    private let CHANNEL = "com.example.myapp/native"
    
    override func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        
        let controller = window?.rootViewController as! FlutterViewController
        let channel = FlutterMethodChannel(
            name: CHANNEL,
            binaryMessenger: controller.binaryMessenger
        )
        
        channel.setMethodCallHandler { [weak self] (call, result) in
            guard let self = self else { return }
            
            switch call.method {
            case "getBatteryLevel":
                let batteryLevel = self.getBatteryLevel()
                result(batteryLevel)
                
            case "getDeviceInfo":
                let detailed = (call.arguments as? [String: Any])?["detailed"] as? Bool ?? false
                let info = self.getDeviceInfo(detailed: detailed)
                result(info)
                
            case "getSystemInfo":
                let info = self.getSystemInfo()
                result(info)
                
            case "showToast":
                // iOS ไม่มี Toast ใช้ alert แทน
                let message = (call.arguments as? [String: Any])?["message"] as? String ?? ""
                self.showAlert(message: message, on: controller)
                result(nil)
                
            default:
                result(FlutterMethodNotImplemented)
            }
        }
        
        return super.application(application, didFinishLaunchingWithOptions: launchOptions)
    }
    
    private func getBatteryLevel() -> Int {
        UIDevice.current.isBatteryMonitoringEnabled = true
        let level = UIDevice.current.batteryLevel
        if level < 0 { return -1 }
        return Int(level * 100)
    }
    
    private func getDeviceInfo(detailed: Bool) -> String {
        let device = UIDevice.current
        if detailed {
            return "Model: \(device.model)\niOS: \(device.systemVersion)\nName: \(device.name)"
        } else {
            return "\(device.model)"
        }
    }
    
    private func getSystemInfo() -> [String: Any] {
        let device = UIDevice.current
        return [
            "model": device.model,
            "systemVersion": device.systemVersion,
            "name": device.name,
            "isSimulator": TARGET_OS_SIMULATOR == 1
        ]
    }
    
    private func showAlert(message: String, on controller: UIViewController) {
        let alert = UIAlertController(
            title: nil,
            message: message,
            preferredStyle: .alert
        )
        controller.present(alert, animated: true)
        DispatchQueue.main.asyncAfter(deadline: .now() + 2) {
            alert.dismiss(animated: true)
        }
    }
}
```

---

## 46.3 EventChannel

### Dart Side

```dart
class SensorService {
  static const EventChannel _channel = EventChannel(
    'com.example.myapp/sensors',
  );
  
  // Subscribe to accelerometer data
  static Stream<AccelerometerData> get accelerometerStream {
    return _channel.receiveBroadcastStream().map((event) {
      final data = Map<String, dynamic>.from(event);
      return AccelerometerData(
        x: data['x'].toDouble(),
        y: data['y'].toDouble(),
        z: data['z'].toDouble(),
      );
    });
  }
  
  // Subscribe to connectivity changes
  static Stream<bool> get connectivityStream {
    return const EventChannel('com.example.myapp/connectivity')
        .receiveBroadcastStream()
        .map((event) => event as bool);
  }
}

class AccelerometerData {
  final double x, y, z;
  const AccelerometerData({
    required this.x,
    required this.y,
    required this.z,
  });
}
```

### Android - EventChannel Kotlin

```kotlin
// StreamHandler สำหรับ Accelerometer
import android.content.Context
import android.hardware.Sensor
import android.hardware.SensorEvent
import android.hardware.SensorEventListener
import android.hardware.SensorManager
import io.flutter.plugin.common.EventChannel

class AccelerometerStreamHandler(private val context: Context) : 
    EventChannel.StreamHandler {
    
    private var sensorManager: SensorManager? = null
    private var sensorEventListener: SensorEventListener? = null
    
    override fun onListen(arguments: Any?, events: EventChannel.EventSink?) {
        sensorManager = context.getSystemService(Context.SENSOR_SERVICE) 
            as SensorManager
        
        val accelerometer = sensorManager?.getDefaultSensor(
            Sensor.TYPE_ACCELEROMETER
        )
        
        sensorEventListener = object : SensorEventListener {
            override fun onSensorChanged(event: SensorEvent?) {
                event?.let {
                    val data = mapOf(
                        "x" to it.values[0].toDouble(),
                        "y" to it.values[1].toDouble(),
                        "z" to it.values[2].toDouble()
                    )
                    events?.success(data)
                }
            }
            
            override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) {}
        }
        
        sensorManager?.registerListener(
            sensorEventListener,
            accelerometer,
            SensorManager.SENSOR_DELAY_NORMAL
        )
    }
    
    override fun onCancel(arguments: Any?) {
        sensorManager?.unregisterListener(sensorEventListener)
        sensorEventListener = null
        sensorManager = null
    }
}

// ใน MainActivity.kt
val sensorChannel = EventChannel(
    flutterEngine.dartExecutor.binaryMessenger,
    "com.example.myapp/sensors"
)
sensorChannel.setStreamHandler(AccelerometerStreamHandler(context))
```

---

## 46.4 BasicMessageChannel

```dart
// Dart Side
class MessageService {
  static const BasicMessageChannel<String> _channel = 
      BasicMessageChannel<String>(
        'com.example.myapp/messages',
        StringCodec(),
      );
  
  // ส่งข้อความไป native
  static Future<String?> sendMessage(String message) async {
    return await _channel.send(message);
  }
  
  // รับข้อความจาก native
  static void startListening() {
    _channel.setMessageHandler((message) async {
      print('Received from native: $message');
      return 'Flutter received: $message';
    });
  }
}
```

---

## 46.5 Workshop: Battery Level Channel

### Complete Battery Monitor App

```dart
// battery_service.dart
import 'package:flutter/services.dart';

class BatteryService {
  static const MethodChannel _methodChannel = MethodChannel(
    'com.example.myapp/battery',
  );
  
  static const EventChannel _eventChannel = EventChannel(
    'com.example.myapp/battery_stream',
  );
  
  // ดึงระดับแบตเตอรี่ครั้งเดียว
  static Future<int> getBatteryLevel() async {
    try {
      return await _methodChannel.invokeMethod('getBatteryLevel');
    } on PlatformException catch (e) {
      throw Exception('ไม่สามารถดึงระดับแบตเตอรี่ได้: ${e.message}');
    }
  }
  
  // ดึงสถานะการชาร์จ
  static Future<BatteryState> getBatteryState() async {
    try {
      final String state = await _methodChannel.invokeMethod('getBatteryState');
      return BatteryState.values.firstWhere(
        (s) => s.name == state,
        orElse: () => BatteryState.unknown,
      );
    } on PlatformException {
      return BatteryState.unknown;
    }
  }
  
  // Stream ระดับแบตเตอรี่แบบ real-time
  static Stream<BatteryInfo> get batteryStream {
    return _eventChannel.receiveBroadcastStream().map((event) {
      final data = Map<String, dynamic>.from(event);
      return BatteryInfo(
        level: data['level'] as int,
        state: BatteryState.values.firstWhere(
          (s) => s.name == data['state'],
          orElse: () => BatteryState.unknown,
        ),
      );
    });
  }
}

enum BatteryState { charging, discharging, full, unknown }

class BatteryInfo {
  final int level;
  final BatteryState state;
  
  const BatteryInfo({required this.level, required this.state});
}

// Battery Monitor UI
class BatteryMonitorPage extends StatefulWidget {
  const BatteryMonitorPage({super.key});
  
  @override
  State<BatteryMonitorPage> createState() => _BatteryMonitorPageState();
}

class _BatteryMonitorPageState extends State<BatteryMonitorPage> {
  BatteryInfo? _batteryInfo;
  bool _isLoading = false;
  
  @override
  void initState() {
    super.initState();
    _loadBatteryInfo();
    _startStreaming();
  }
  
  Future<void> _loadBatteryInfo() async {
    setState(() => _isLoading = true);
    
    try {
      final level = await BatteryService.getBatteryLevel();
      final state = await BatteryService.getBatteryState();
      
      setState(() {
        _batteryInfo = BatteryInfo(level: level, state: state);
        _isLoading = false;
      });
    } catch (e) {
      setState(() => _isLoading = false);
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text(e.toString())),
        );
      }
    }
  }
  
  StreamSubscription<BatteryInfo>? _subscription;
  
  void _startStreaming() {
    _subscription = BatteryService.batteryStream.listen(
      (info) => setState(() => _batteryInfo = info),
      onError: (e) => print('Stream error: $e'),
    );
  }
  
  Color get _batteryColor {
    final level = _batteryInfo?.level ?? 0;
    if (level <= 20) return Colors.red;
    if (level <= 50) return Colors.orange;
    return Colors.green;
  }
  
  IconData get _batteryIcon {
    switch (_batteryInfo?.state) {
      case BatteryState.charging:
        return Icons.battery_charging_full;
      case BatteryState.full:
        return Icons.battery_full;
      default:
        final level = _batteryInfo?.level ?? 0;
        if (level > 80) return Icons.battery_full;
        if (level > 60) return Icons.battery_5_bar;
        if (level > 40) return Icons.battery_4_bar;
        if (level > 20) return Icons.battery_2_bar;
        return Icons.battery_1_bar;
    }
  }
  
  String get _stateLabel {
    switch (_batteryInfo?.state) {
      case BatteryState.charging:
        return 'กำลังชาร์จ';
      case BatteryState.discharging:
        return 'ใช้งานปกติ';
      case BatteryState.full:
        return 'ชาร์จเต็มแล้ว';
      default:
        return 'ไม่ทราบสถานะ';
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('ระดับแบตเตอรี่'),
        actions: [
          IconButton(
            onPressed: _loadBatteryInfo,
            icon: const Icon(Icons.refresh),
          ),
        ],
      ),
      body: Center(
        child: _isLoading
            ? const CircularProgressIndicator()
            : Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  Icon(
                    _batteryIcon,
                    size: 120,
                    color: _batteryColor,
                  ),
                  const SizedBox(height: 24),
                  Text(
                    '${_batteryInfo?.level ?? 0}%',
                    style: TextStyle(
                      fontSize: 64,
                      fontWeight: FontWeight.bold,
                      color: _batteryColor,
                    ),
                  ),
                  const SizedBox(height: 8),
                  Text(
                    _stateLabel,
                    style: const TextStyle(
                      fontSize: 20,
                      color: Colors.grey,
                    ),
                  ),
                  const SizedBox(height: 32),
                  Padding(
                    padding: const EdgeInsets.symmetric(horizontal: 40),
                    child: ClipRRect(
                      borderRadius: BorderRadius.circular(8),
                      child: LinearProgressIndicator(
                        value: (_batteryInfo?.level ?? 0) / 100,
                        minHeight: 20,
                        backgroundColor: Colors.grey[200],
                        valueColor: AlwaysStoppedAnimation<Color>(_batteryColor),
                      ),
                    ),
                  ),
                ],
              ),
      ),
    );
  }
  
  @override
  void dispose() {
    _subscription?.cancel();
    super.dispose();
  }
}
```

### Android Native Battery Implementation

```kotlin
// BatteryPlugin.kt
package com.example.myapp

import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import android.content.IntentFilter
import android.os.BatteryManager
import android.os.Build
import io.flutter.plugin.common.EventChannel
import io.flutter.plugin.common.MethodChannel

class BatteryPlugin(private val context: Context) {
    
    fun registerMethodChannel(messenger: io.flutter.plugin.common.BinaryMessenger) {
        MethodChannel(messenger, "com.example.myapp/battery")
            .setMethodCallHandler { call, result ->
                when (call.method) {
                    "getBatteryLevel" -> result.success(getBatteryLevel())
                    "getBatteryState" -> result.success(getBatteryState())
                    else -> result.notImplemented()
                }
            }
    }
    
    fun registerEventChannel(messenger: io.flutter.plugin.common.BinaryMessenger) {
        EventChannel(messenger, "com.example.myapp/battery_stream")
            .setStreamHandler(BatteryStreamHandler(context))
    }
    
    private fun getBatteryLevel(): Int {
        val bm = context.getSystemService(Context.BATTERY_SERVICE) as BatteryManager
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            bm.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY)
        } else {
            -1
        }
    }
    
    private fun getBatteryState(): String {
        val intent = context.registerReceiver(
            null, 
            IntentFilter(Intent.ACTION_BATTERY_CHANGED)
        )
        return when (intent?.getIntExtra(BatteryManager.EXTRA_STATUS, -1)) {
            BatteryManager.BATTERY_STATUS_CHARGING -> "charging"
            BatteryManager.BATTERY_STATUS_FULL -> "full"
            BatteryManager.BATTERY_STATUS_DISCHARGING,
            BatteryManager.BATTERY_STATUS_NOT_CHARGING -> "discharging"
            else -> "unknown"
        }
    }
}

class BatteryStreamHandler(private val context: Context) : 
    EventChannel.StreamHandler {
    
    private var broadcastReceiver: BroadcastReceiver? = null
    
    override fun onListen(arguments: Any?, events: EventChannel.EventSink?) {
        broadcastReceiver = object : BroadcastReceiver() {
            override fun onReceive(ctx: Context?, intent: Intent?) {
                intent?.let {
                    val level = it.getIntExtra(BatteryManager.EXTRA_LEVEL, -1)
                    val scale = it.getIntExtra(BatteryManager.EXTRA_SCALE, -1)
                    val batteryPct = (level * 100 / scale.toFloat()).toInt()
                    
                    val state = when (it.getIntExtra(BatteryManager.EXTRA_STATUS, -1)) {
                        BatteryManager.BATTERY_STATUS_CHARGING -> "charging"
                        BatteryManager.BATTERY_STATUS_FULL -> "full"
                        else -> "discharging"
                    }
                    
                    events?.success(mapOf(
                        "level" to batteryPct,
                        "state" to state
                    ))
                }
            }
        }
        
        context.registerReceiver(
            broadcastReceiver,
            IntentFilter(Intent.ACTION_BATTERY_CHANGED)
        )
    }
    
    override fun onCancel(arguments: Any?) {
        context.unregisterReceiver(broadcastReceiver)
        broadcastReceiver = null
    }
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- MethodChannel สำหรับ request-response แบบครั้งเดียว
- EventChannel สำหรับ streaming data จาก native
- BasicMessageChannel สำหรับการสื่อสารสองทาง
- การ implement native code บน Android (Kotlin) และ iOS (Swift)
- Workshop: Battery Level Channel แบบสมบูรณ์

**แบบฝึกหัดเพิ่มเติม:**
1. สร้าง plugin สำหรับดึงข้อมูล WiFi
2. Implement biometric authentication
3. สร้าง file system watcher ด้วย EventChannel
4. เพิ่ม background task execution
