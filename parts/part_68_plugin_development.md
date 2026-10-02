# Part 68: Plugin Development ใน Flutter

## Plugin คืออะไร?

Flutter Plugin คือ package ที่ให้ Flutter code เรียกใช้ native code (Android/iOS) ได้ เช่น:
- Camera
- GPS
- Bluetooth
- Fingerprint
- Device Info

---

## 1. สร้าง Flutter Plugin

### สร้าง Plugin Project

```bash
# สร้าง plugin ใหม่
flutter create --template=plugin --platforms=android,ios device_info_plugin

# โครงสร้างที่ได้
device_info_plugin/
├── android/
│   └── src/main/kotlin/.../DeviceInfoPlugin.kt
├── ios/
│   └── Classes/DeviceInfoPlugin.swift
├── lib/
│   ├── device_info_plugin.dart
│   └── device_info_plugin_platform_interface.dart
├── example/
│   └── lib/main.dart
├── test/
└── pubspec.yaml
```

### Dart API

```dart
// lib/device_info_plugin.dart
library device_info_plugin;

export 'src/device_info_plugin.dart';
export 'src/models/device_info.dart';

// lib/src/device_info_plugin.dart
import 'package:device_info_plugin/src/device_info_plugin_platform_interface.dart';
import 'models/device_info.dart';

class DeviceInfoPlugin {
  // Singleton instance
  static final DeviceInfoPlugin _instance = DeviceInfoPlugin._();
  factory DeviceInfoPlugin() => _instance;
  DeviceInfoPlugin._();

  /// ดึงข้อมูลอุปกรณ์ Android
  Future<AndroidDeviceInfo?> androidInfo() async {
    if (defaultTargetPlatform != TargetPlatform.android) return null;
    return DeviceInfoPluginPlatform.instance.androidInfo();
  }

  /// ดึงข้อมูลอุปกรณ์ iOS
  Future<IosDeviceInfo?> iosInfo() async {
    if (defaultTargetPlatform != TargetPlatform.iOS) return null;
    return DeviceInfoPluginPlatform.instance.iosInfo();
  }

  /// ดึงข้อมูลทั่วไป (ทุก platform)
  Future<DeviceInfo> getDeviceInfo() async {
    return DeviceInfoPluginPlatform.instance.getDeviceInfo();
  }
}
```

### Platform Interface

```dart
// lib/src/device_info_plugin_platform_interface.dart
import 'package:plugin_platform_interface/plugin_platform_interface.dart';

abstract class DeviceInfoPluginPlatform extends PlatformInterface {
  DeviceInfoPluginPlatform() : super(token: _token);

  static final Object _token = Object();
  static DeviceInfoPluginPlatform _instance = MethodChannelDeviceInfoPlugin();

  static DeviceInfoPluginPlatform get instance => _instance;
  static set instance(DeviceInfoPluginPlatform instance) {
    PlatformInterface.verifyToken(instance, _token);
    _instance = instance;
  }

  Future<AndroidDeviceInfo?> androidInfo() {
    throw UnimplementedError('androidInfo() has not been implemented.');
  }

  Future<IosDeviceInfo?> iosInfo() {
    throw UnimplementedError('iosInfo() has not been implemented.');
  }

  Future<DeviceInfo> getDeviceInfo() {
    throw UnimplementedError('getDeviceInfo() has not been implemented.');
  }
}
```

### Method Channel Implementation

```dart
// lib/src/method_channel_device_info_plugin.dart
import 'package:flutter/services.dart';

class MethodChannelDeviceInfoPlugin extends DeviceInfoPluginPlatform {
  // ชื่อ channel ต้องตรงกับ native code
  static const MethodChannel _channel =
      MethodChannel('com.example.device_info_plugin');

  @override
  Future<AndroidDeviceInfo?> androidInfo() async {
    try {
      final Map<dynamic, dynamic>? result =
          await _channel.invokeMethod('getAndroidInfo');
      if (result == null) return null;
      return AndroidDeviceInfo.fromMap(Map<String, dynamic>.from(result));
    } on PlatformException catch (e) {
      throw PlatformException(
        code: e.code,
        message: 'Failed to get Android info: ${e.message}',
      );
    }
  }

  @override
  Future<IosDeviceInfo?> iosInfo() async {
    try {
      final Map<dynamic, dynamic>? result =
          await _channel.invokeMethod('getIosInfo');
      if (result == null) return null;
      return IosDeviceInfo.fromMap(Map<String, dynamic>.from(result));
    } on PlatformException catch (e) {
      throw PlatformException(
        code: e.code,
        message: 'Failed to get iOS info: ${e.message}',
      );
    }
  }

  @override
  Future<DeviceInfo> getDeviceInfo() async {
    final Map<dynamic, dynamic> result =
        await _channel.invokeMethod('getDeviceInfo');
    return DeviceInfo.fromMap(Map<String, dynamic>.from(result));
  }
}
```

---

## 2. Platform-specific Code

### Android Implementation (Kotlin)

```kotlin
// android/src/main/kotlin/.../DeviceInfoPlugin.kt
package com.example.device_info_plugin

import android.os.Build
import android.content.Context
import io.flutter.embedding.engine.plugins.FlutterPlugin
import io.flutter.plugin.common.MethodCall
import io.flutter.plugin.common.MethodChannel
import io.flutter.plugin.common.MethodChannel.MethodCallHandler
import io.flutter.plugin.common.MethodChannel.Result

class DeviceInfoPlugin : FlutterPlugin, MethodCallHandler {
    private lateinit var channel: MethodChannel
    private lateinit var context: Context

    override fun onAttachedToEngine(binding: FlutterPlugin.FlutterPluginBinding) {
        channel = MethodChannel(
            binding.binaryMessenger,
            "com.example.device_info_plugin"
        )
        channel.setMethodCallHandler(this)
        context = binding.applicationContext
    }

    override fun onMethodCall(call: MethodCall, result: Result) {
        when (call.method) {
            "getAndroidInfo" -> {
                result.success(getAndroidInfo())
            }
            "getDeviceInfo" -> {
                result.success(getDeviceInfo())
            }
            else -> {
                result.notImplemented()
            }
        }
    }

    private fun getAndroidInfo(): Map<String, Any?> {
        return mapOf(
            "version" to Build.VERSION.RELEASE,
            "sdkInt" to Build.VERSION.SDK_INT,
            "brand" to Build.BRAND,
            "manufacturer" to Build.MANUFACTURER,
            "model" to Build.MODEL,
            "device" to Build.DEVICE,
            "product" to Build.PRODUCT,
            "hardware" to Build.HARDWARE,
            "isPhysicalDevice" to !isEmulator(),
            "androidId" to getAndroidId(),
        )
    }

    private fun getDeviceInfo(): Map<String, Any?> {
        return mapOf(
            "platform" to "android",
            "osVersion" to Build.VERSION.RELEASE,
            "deviceModel" to "${Build.MANUFACTURER} ${Build.MODEL}",
            "isPhysicalDevice" to !isEmulator(),
            "appVersion" to getAppVersion(),
        )
    }

    private fun isEmulator(): Boolean {
        return (Build.FINGERPRINT.startsWith("generic")
                || Build.FINGERPRINT.startsWith("unknown")
                || Build.MODEL.contains("google_sdk")
                || Build.MODEL.contains("Emulator")
                || Build.MODEL.contains("Android SDK built for x86")
                || Build.MANUFACTURER.contains("Genymotion")
                || (Build.BRAND.startsWith("generic")
                        && Build.DEVICE.startsWith("generic"))
                || "google_sdk" == Build.PRODUCT)
    }

    private fun getAndroidId(): String? {
        return try {
            android.provider.Settings.Secure.getString(
                context.contentResolver,
                android.provider.Settings.Secure.ANDROID_ID
            )
        } catch (e: Exception) {
            null
        }
    }

    private fun getAppVersion(): String {
        return try {
            val packageInfo = context.packageManager.getPackageInfo(
                context.packageName, 0
            )
            packageInfo.versionName ?: "unknown"
        } catch (e: Exception) {
            "unknown"
        }
    }

    override fun onDetachedFromEngine(binding: FlutterPlugin.FlutterPluginBinding) {
        channel.setMethodCallHandler(null)
    }
}
```

### iOS Implementation (Swift)

```swift
// ios/Classes/DeviceInfoPlugin.swift
import Flutter
import UIKit

public class DeviceInfoPlugin: NSObject, FlutterPlugin {
    public static func register(with registrar: FlutterPluginRegistrar) {
        let channel = FlutterMethodChannel(
            name: "com.example.device_info_plugin",
            binaryMessenger: registrar.messenger()
        )
        let instance = DeviceInfoPlugin()
        registrar.addMethodCallDelegate(instance, channel: channel)
    }

    public func handle(_ call: FlutterMethodCall, result: @escaping FlutterResult) {
        switch call.method {
        case "getIosInfo":
            result(getIosInfo())
        case "getDeviceInfo":
            result(getDeviceInfo())
        default:
            result(FlutterMethodNotImplemented)
        }
    }

    private func getIosInfo() -> [String: Any?] {
        let device = UIDevice.current
        return [
            "name": device.name,
            "systemName": device.systemName,
            "systemVersion": device.systemVersion,
            "model": device.model,
            "localizedModel": device.localizedModel,
            "identifierForVendor": device.identifierForVendor?.uuidString,
            "isPhysicalDevice": !isSimulator(),
        ]
    }

    private func getDeviceInfo() -> [String: Any?] {
        let device = UIDevice.current
        return [
            "platform": "ios",
            "osVersion": "\(device.systemName) \(device.systemVersion)",
            "deviceModel": device.model,
            "isPhysicalDevice": !isSimulator(),
            "appVersion": getAppVersion(),
        ]
    }

    private func isSimulator() -> Bool {
        #if targetEnvironment(simulator)
        return true
        #else
        return false
        #endif
    }

    private func getAppVersion() -> String {
        return Bundle.main.object(forInfoDictionaryKey: "CFBundleShortVersionString") as? String ?? "unknown"
    }
}
```

---

## 3. Models

```dart
// lib/src/models/device_info.dart
class DeviceInfo {
  final String platform;
  final String osVersion;
  final String deviceModel;
  final bool isPhysicalDevice;
  final String appVersion;

  DeviceInfo({
    required this.platform,
    required this.osVersion,
    required this.deviceModel,
    required this.isPhysicalDevice,
    required this.appVersion,
  });

  factory DeviceInfo.fromMap(Map<String, dynamic> map) {
    return DeviceInfo(
      platform: map['platform'] as String,
      osVersion: map['osVersion'] as String,
      deviceModel: map['deviceModel'] as String,
      isPhysicalDevice: map['isPhysicalDevice'] as bool,
      appVersion: map['appVersion'] as String,
    );
  }

  @override
  String toString() {
    return 'DeviceInfo($platform, $deviceModel, $osVersion)';
  }
}

class AndroidDeviceInfo {
  final String version;
  final int sdkInt;
  final String brand;
  final String manufacturer;
  final String model;
  final String device;
  final bool isPhysicalDevice;
  final String? androidId;

  AndroidDeviceInfo({
    required this.version,
    required this.sdkInt,
    required this.brand,
    required this.manufacturer,
    required this.model,
    required this.device,
    required this.isPhysicalDevice,
    this.androidId,
  });

  factory AndroidDeviceInfo.fromMap(Map<String, dynamic> map) {
    return AndroidDeviceInfo(
      version: map['version'] as String,
      sdkInt: map['sdkInt'] as int,
      brand: map['brand'] as String,
      manufacturer: map['manufacturer'] as String,
      model: map['model'] as String,
      device: map['device'] as String,
      isPhysicalDevice: map['isPhysicalDevice'] as bool,
      androidId: map['androidId'] as String?,
    );
  }
}

class IosDeviceInfo {
  final String name;
  final String systemName;
  final String systemVersion;
  final String model;
  final bool isPhysicalDevice;
  final String? identifierForVendor;

  IosDeviceInfo({
    required this.name,
    required this.systemName,
    required this.systemVersion,
    required this.model,
    required this.isPhysicalDevice,
    this.identifierForVendor,
  });

  factory IosDeviceInfo.fromMap(Map<String, dynamic> map) {
    return IosDeviceInfo(
      name: map['name'] as String,
      systemName: map['systemName'] as String,
      systemVersion: map['systemVersion'] as String,
      model: map['model'] as String,
      isPhysicalDevice: map['isPhysicalDevice'] as bool,
      identifierForVendor: map['identifierForVendor'] as String?,
    );
  }
}
```

---

## 4. Publishing ไปยัง pub.dev

### pubspec.yaml สำหรับ Plugin

```yaml
name: device_info_plugin
description: A Flutter plugin to get device information.
version: 1.0.0
homepage: https://github.com/yourusername/device_info_plugin

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: '>=3.0.0'

dependencies:
  flutter:
    sdk: flutter
  plugin_platform_interface: ^2.0.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0

flutter:
  plugin:
    platforms:
      android:
        kotlinOptions:
          jvmTarget: '11'
        package: com.example.device_info_plugin
        pluginClass: DeviceInfoPlugin
      ios:
        pluginClass: DeviceInfoPlugin
```

### การเตรียม Package

```bash
# ตรวจสอบ package ก่อน publish
flutter pub publish --dry-run

# ตรวจสอบ issues
dart pub publish --dry-run

# Publish!
flutter pub publish
```

### Workshop: ทดสอบ Plugin

```dart
// example/lib/main.dart
import 'package:flutter/material.dart';
import 'package:device_info_plugin/device_info_plugin.dart';

void main() {
  runApp(PluginExampleApp());
}

class PluginExampleApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Device Info Plugin Example',
      home: DeviceInfoScreen(),
    );
  }
}

class DeviceInfoScreen extends StatefulWidget {
  @override
  _DeviceInfoScreenState createState() => _DeviceInfoScreenState();
}

class _DeviceInfoScreenState extends State<DeviceInfoScreen> {
  final _plugin = DeviceInfoPlugin();
  DeviceInfo? _deviceInfo;
  AndroidDeviceInfo? _androidInfo;
  IosDeviceInfo? _iosInfo;
  bool _isLoading = true;

  @override
  void initState() {
    super.initState();
    _loadDeviceInfo();
  }

  Future<void> _loadDeviceInfo() async {
    try {
      final deviceInfo = await _plugin.getDeviceInfo();
      final androidInfo = await _plugin.androidInfo();
      final iosInfo = await _plugin.iosInfo();

      setState(() {
        _deviceInfo = deviceInfo;
        _androidInfo = androidInfo;
        _iosInfo = iosInfo;
        _isLoading = false;
      });
    } catch (e) {
      setState(() => _isLoading = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Device Info')),
      body: _isLoading
          ? Center(child: CircularProgressIndicator())
          : ListView(
              padding: EdgeInsets.all(16),
              children: [
                if (_deviceInfo != null) ...[
                  _InfoSection(
                    title: 'ข้อมูลทั่วไป',
                    items: {
                      'Platform': _deviceInfo!.platform,
                      'OS Version': _deviceInfo!.osVersion,
                      'Device Model': _deviceInfo!.deviceModel,
                      'Physical Device': _deviceInfo!.isPhysicalDevice.toString(),
                      'App Version': _deviceInfo!.appVersion,
                    },
                  ),
                ],
                if (_androidInfo != null) ...[
                  _InfoSection(
                    title: 'Android Info',
                    items: {
                      'Version': _androidInfo!.version,
                      'SDK': _androidInfo!.sdkInt.toString(),
                      'Brand': _androidInfo!.brand,
                      'Manufacturer': _androidInfo!.manufacturer,
                      'Model': _androidInfo!.model,
                    },
                  ),
                ],
                if (_iosInfo != null) ...[
                  _InfoSection(
                    title: 'iOS Info',
                    items: {
                      'Device Name': _iosInfo!.name,
                      'System': _iosInfo!.systemName,
                      'Version': _iosInfo!.systemVersion,
                      'Model': _iosInfo!.model,
                    },
                  ),
                ],
              ],
            ),
    );
  }
}

class _InfoSection extends StatelessWidget {
  final String title;
  final Map<String, String> items;

  const _InfoSection({required this.title, required this.items});

  @override
  Widget build(BuildContext context) {
    return Card(
      margin: EdgeInsets.only(bottom: 16),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Padding(
            padding: EdgeInsets.all(16),
            child: Text(
              title,
              style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold),
            ),
          ),
          Divider(height: 1),
          ...items.entries.map(
            (entry) => ListTile(
              title: Text(entry.key),
              trailing: Text(
                entry.value,
                style: TextStyle(color: Colors.grey[600]),
              ),
            ),
          ),
        ],
      ),
    );
  }
}
```

---

## สรุป

Plugin Development ใน Flutter ประกอบด้วย:

1. **สร้าง Plugin** - flutter create --template=plugin
2. **Dart API** - Platform interface pattern
3. **Method Channel** - สื่อสารระหว่าง Dart และ Native
4. **Android (Kotlin)** - native implementation
5. **iOS (Swift)** - native implementation
6. **Publishing** - เผยแพร่บน pub.dev
