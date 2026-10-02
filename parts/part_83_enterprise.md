# Part 83: Enterprise Architecture - สถาปัตยกรรมระดับองค์กร

## บทนำ

Enterprise Flutter apps มีความต้องการพิเศษ เช่น multi-flavor builds, environment configuration, security hardening และการรองรับ Mobile Device Management (MDM) ส่วนนี้จะครอบคลุมทุกด้านของ enterprise app development

## 1. Multi-flavor Apps

### Flutter Flavors คืออะไร?

Flavors ช่วยให้เราสร้าง app หลาย versions จากโค้ดเดียวกัน เช่น:
- **Development** - สำหรับ developers (debug mode, mock data)
- **Staging** - สำหรับ QA (test environment)
- **Production** - สำหรับ users จริง

### Android Flavor Setup

```groovy
// android/app/build.gradle
android {
    flavorDimensions "app"
    
    productFlavors {
        development {
            dimension "app"
            applicationId "com.example.app.dev"
            resValue "string", "app_name", "MyApp Dev"
            // ใช้ icon แตกต่างกัน
            manifestPlaceholders = [
                appIcon: "@mipmap/ic_launcher_dev"
            ]
        }
        
        staging {
            dimension "app"
            applicationId "com.example.app.staging"
            resValue "string", "app_name", "MyApp Staging"
            manifestPlaceholders = [
                appIcon: "@mipmap/ic_launcher_staging"
            ]
        }
        
        production {
            dimension "app"
            applicationId "com.example.app"
            resValue "string", "app_name", "MyApp"
            manifestPlaceholders = [
                appIcon: "@mipmap/ic_launcher"
            ]
        }
    }
}
```

### iOS Flavor Setup

```bash
# สร้าง Xcode configurations
# Project → Info → Configurations:
# - Debug-Development
# - Debug-Staging
# - Debug-Production
# - Release-Development
# - Release-Staging
# - Release-Production
```

```ruby
# ios/Podfile
target 'Runner' do
  # ...
end

post_install do |installer|
  installer.pods_project.targets.each do |target|
    flutter_additional_ios_build_settings(target)
    target.build_configurations.each do |config|
      if config.name.include?("Development")
        config.build_settings['BUNDLE_IDENTIFIER'] = 'com.example.app.dev'
        config.build_settings['PRODUCT_BUNDLE_IDENTIFIER'] = 'com.example.app.dev'
      elsif config.name.include?("Staging")
        config.build_settings['BUNDLE_IDENTIFIER'] = 'com.example.app.staging'
        config.build_settings['PRODUCT_BUNDLE_IDENTIFIER'] = 'com.example.app.staging'
      end
    end
  end
end
```

## 2. Environment Configuration

### App Config

```dart
// lib/config/app_config.dart
import 'package:flutter/foundation.dart';

enum Environment {
  development,
  staging,
  production,
}

class AppConfig {
  final Environment environment;
  final String apiBaseUrl;
  final String wsBaseUrl;
  final bool enableLogging;
  final bool enableCrashReporting;
  final bool enableAnalytics;
  final int apiTimeoutSeconds;
  final String sentryDsn;

  const AppConfig({
    required this.environment,
    required this.apiBaseUrl,
    required this.wsBaseUrl,
    required this.enableLogging,
    required this.enableCrashReporting,
    required this.enableAnalytics,
    required this.apiTimeoutSeconds,
    required this.sentryDsn,
  });

  bool get isDevelopment => environment == Environment.development;
  bool get isStaging => environment == Environment.staging;
  bool get isProduction => environment == Environment.production;

  // Factory สำหรับแต่ละ environment
  factory AppConfig.development() => const AppConfig(
        environment: Environment.development,
        apiBaseUrl: 'https://api.dev.example.com',
        wsBaseUrl: 'wss://ws.dev.example.com',
        enableLogging: true,
        enableCrashReporting: false,
        enableAnalytics: false,
        apiTimeoutSeconds: 30,
        sentryDsn: '',
      );

  factory AppConfig.staging() => const AppConfig(
        environment: Environment.staging,
        apiBaseUrl: 'https://api.staging.example.com',
        wsBaseUrl: 'wss://ws.staging.example.com',
        enableLogging: true,
        enableCrashReporting: true,
        enableAnalytics: false,
        apiTimeoutSeconds: 20,
        sentryDsn: 'https://staging@sentry.io/project',
      );

  factory AppConfig.production() => const AppConfig(
        environment: Environment.production,
        apiBaseUrl: 'https://api.example.com',
        wsBaseUrl: 'wss://ws.example.com',
        enableLogging: false,
        enableCrashReporting: true,
        enableAnalytics: true,
        apiTimeoutSeconds: 15,
        sentryDsn: 'https://prod@sentry.io/project',
      );
}
```

### Entry Points สำหรับแต่ละ Flavor

```dart
// lib/main_development.dart
import 'package:flutter/material.dart';
import 'config/app_config.dart';
import 'app/app.dart';

void main() {
  AppConfig.initialize(AppConfig.development());
  runApp(const MyApp());
}
```

```dart
// lib/main_staging.dart
import 'package:flutter/material.dart';
import 'config/app_config.dart';
import 'app/app.dart';

void main() {
  AppConfig.initialize(AppConfig.staging());
  runApp(const MyApp());
}
```

```dart
// lib/main_production.dart
import 'package:flutter/material.dart';
import 'config/app_config.dart';
import 'app/app.dart';

void main() {
  AppConfig.initialize(AppConfig.production());
  runApp(const MyApp());
}
```

### Run ด้วย Flavor

```bash
# Development
flutter run --flavor development -t lib/main_development.dart

# Staging
flutter run --flavor staging -t lib/main_staging.dart

# Production
flutter run --flavor production -t lib/main_production.dart

# Build
flutter build apk --flavor production -t lib/main_production.dart
flutter build ipa --flavor production -t lib/main_production.dart
```

## 3. Enterprise Security

### Certificate Pinning

```dart
// lib/security/certificate_pinning.dart
import 'dart:io';
import 'package:dio/dio.dart';
import 'package:flutter/services.dart';

class CertificatePinningInterceptor extends Interceptor {
  final List<String> _pinnedCertificates;

  CertificatePinningInterceptor({required List<String> pinnedCertificates})
      : _pinnedCertificates = pinnedCertificates;

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    handler.next(options);
  }
}

// ตั้งค่า HttpClient ด้วย certificate pinning
Future<HttpClient> createSecureHttpClient() async {
  final securityContext = SecurityContext();

  // โหลด certificate จาก assets
  final certData = await rootBundle.load('assets/certs/server.crt');
  securityContext.setTrustedCertificatesBytes(certData.buffer.asUint8List());

  final client = HttpClient(context: securityContext);

  // ตั้งค่า host verification
  client.badCertificateCallback = (cert, host, port) {
    // ตรวจสอบ certificate fingerprint
    final fingerprint = _getCertFingerprint(cert);
    return _isValidFingerprint(fingerprint, host);
  };

  return client;
}

String _getCertFingerprint(X509Certificate cert) {
  // คำนวณ SHA-256 fingerprint
  return cert.sha256.map((b) => b.toRadixString(16).padLeft(2, '0')).join(':');
}

bool _isValidFingerprint(String fingerprint, String host) {
  const validFingerprints = {
    'api.example.com':
        'AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99',
  };

  return validFingerprints[host] == fingerprint;
}
```

### Secure Storage

```dart
// lib/security/secure_storage.dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class SecureStorageService {
  static const _storage = FlutterSecureStorage(
    aOptions: AndroidOptions(
      encryptedSharedPreferences: true,
    ),
    iOptions: IOSOptions(
      accessibility: KeychainAccessibility.first_unlock_this_device,
    ),
  );

  // Keys
  static const String _authTokenKey = 'auth_token';
  static const String _refreshTokenKey = 'refresh_token';
  static const String _biometricKeyKey = 'biometric_key';
  static const String _pinHashKey = 'pin_hash';

  Future<void> saveAuthToken(String token) async {
    await _storage.write(key: _authTokenKey, value: token);
  }

  Future<String?> getAuthToken() async {
    return await _storage.read(key: _authTokenKey);
  }

  Future<void> saveRefreshToken(String token) async {
    await _storage.write(key: _refreshTokenKey, value: token);
  }

  Future<String?> getRefreshToken() async {
    return await _storage.read(key: _refreshTokenKey);
  }

  Future<void> clearAll() async {
    await _storage.deleteAll();
  }
}
```

### Jailbreak/Root Detection

```dart
// lib/security/device_security.dart
import 'dart:io';
import 'package:flutter/services.dart';

class DeviceSecurity {
  static const MethodChannel _channel =
      MethodChannel('com.example.app/security');

  static Future<bool> isDeviceRooted() async {
    if (!Platform.isAndroid) return false;

    try {
      return await _channel.invokeMethod('isRooted') ?? false;
    } on PlatformException {
      return false;
    }
  }

  static Future<bool> isDeviceJailbroken() async {
    if (!Platform.isIOS) return false;

    try {
      return await _channel.invokeMethod('isJailbroken') ?? false;
    } on PlatformException {
      return false;
    }
  }

  static Future<bool> isDeviceSecure() async {
    final isRooted = await isDeviceRooted();
    final isJailbroken = await isDeviceJailbroken();
    return !isRooted && !isJailbroken;
  }
}
```

### Biometric Authentication

```dart
// lib/security/biometric_auth.dart
import 'package:local_auth/local_auth.dart';

class BiometricAuthService {
  final LocalAuthentication _localAuth = LocalAuthentication();

  Future<bool> isAvailable() async {
    final canCheck = await _localAuth.canCheckBiometrics;
    final isDeviceSupported = await _localAuth.isDeviceSupported();
    return canCheck && isDeviceSupported;
  }

  Future<List<BiometricType>> getAvailableBiometrics() async {
    return await _localAuth.getAvailableBiometrics();
  }

  Future<bool> authenticate({String reason = 'ยืนยันตัวตนเพื่อดำเนินการต่อ'}) async {
    try {
      return await _localAuth.authenticate(
        localizedReason: reason,
        options: const AuthenticationOptions(
          stickyAuth: true,
          biometricOnly: false, // อนุญาต PIN เป็น fallback
        ),
      );
    } catch (e) {
      return false;
    }
  }
}
```

## 4. MDM/EMM Considerations

### MDM Configuration Support

```dart
// lib/mdm/mdm_config.dart
import 'package:flutter/services.dart';
import 'dart:convert';

class MdmConfig {
  static const MethodChannel _channel =
      MethodChannel('com.example.app/mdm');

  // Android: อ่าน Managed Configuration
  // iOS: อ่าน AppConfig.plist จาก MDM
  static Future<Map<String, dynamic>?> getManagedConfig() async {
    try {
      final jsonString = await _channel.invokeMethod<String>('getManagedConfig');
      if (jsonString == null) return null;
      return json.decode(jsonString) as Map<String, dynamic>;
    } on PlatformException {
      return null;
    }
  }

  // ตัวอย่าง config ที่ IT admin จัดการ
  static Future<EnterpriseConfig> getEnterpriseConfig() async {
    final managedConfig = await getManagedConfig();

    return EnterpriseConfig(
      serverUrl: managedConfig?['server_url'] as String? ??
          'https://api.example.com',
      allowBiometric: managedConfig?['allow_biometric'] as bool? ?? true,
      sessionTimeout: managedConfig?['session_timeout'] as int? ?? 30,
      requireVPN: managedConfig?['require_vpn'] as bool? ?? false,
      allowScreenCapture:
          managedConfig?['allow_screen_capture'] as bool? ?? true,
    );
  }
}

class EnterpriseConfig {
  final String serverUrl;
  final bool allowBiometric;
  final int sessionTimeout; // minutes
  final bool requireVPN;
  final bool allowScreenCapture;

  const EnterpriseConfig({
    required this.serverUrl,
    required this.allowBiometric,
    required this.sessionTimeout,
    required this.requireVPN,
    required this.allowScreenCapture,
  });
}
```

## 5. Workshop: Enterprise App Setup

### Complete Enterprise Setup

```dart
// lib/main_production.dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:sentry_flutter/sentry_flutter.dart';
import 'config/app_config.dart';
import 'security/device_security.dart';
import 'mdm/mdm_config.dart';
import 'app/app.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // ล็อค orientation สำหรับ phones
  await SystemChrome.setPreferredOrientations([
    DeviceOrientation.portraitUp,
    DeviceOrientation.portraitDown,
  ]);

  // โหลด MDM config ก่อน
  final enterpriseConfig = await MdmConfig.getEnterpriseConfig();

  // Initialize AppConfig ด้วย MDM values
  AppConfig.initialize(AppConfig.production().copyWith(
    apiBaseUrl: enterpriseConfig.serverUrl,
  ));

  // ตรวจสอบ device security
  final isSecure = await DeviceSecurity.isDeviceSecure();
  if (!isSecure) {
    // แสดง security warning หรือปฏิเสธการใช้งาน
    runApp(const SecurityWarningApp());
    return;
  }

  // Initialize Sentry สำหรับ crash reporting
  await SentryFlutter.init(
    (options) {
      options.dsn = AppConfig.instance.sentryDsn;
      options.tracesSampleRate = 0.1; // 10% sampling
    },
    appRunner: () => runApp(const MyApp()),
  );
}

class SecurityWarningApp extends StatelessWidget {
  const SecurityWarningApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Scaffold(
        body: Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              const Icon(Icons.security, size: 64, color: Colors.red),
              const SizedBox(height: 16),
              const Text(
                'ไม่สามารถใช้งานได้',
                style: TextStyle(
                  fontSize: 20,
                  fontWeight: FontWeight.bold,
                ),
              ),
              const SizedBox(height: 8),
              const Text(
                'อุปกรณ์นี้ผ่านการ root/jailbreak\n'
                'ไม่สามารถใช้งานแอปนี้ได้เพื่อความปลอดภัย',
                textAlign: TextAlign.center,
                style: TextStyle(color: Colors.grey),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

### Session Management

```dart
// lib/security/session_manager.dart
import 'dart:async';
import 'package:flutter/widgets.dart';

class SessionManager with WidgetsBindingObserver {
  final int timeoutMinutes;
  final VoidCallback onSessionExpired;
  Timer? _timer;
  DateTime? _lastActivity;

  SessionManager({
    required this.timeoutMinutes,
    required this.onSessionExpired,
  });

  void initialize() {
    WidgetsBinding.instance.addObserver(this);
    _resetTimer();
  }

  void recordActivity() {
    _lastActivity = DateTime.now();
    _resetTimer();
  }

  void _resetTimer() {
    _timer?.cancel();
    _timer = Timer(
      Duration(minutes: timeoutMinutes),
      _handleTimeout,
    );
  }

  void _handleTimeout() {
    onSessionExpired();
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    if (state == AppLifecycleState.resumed) {
      // ตรวจสอบว่า session หมดอายุขณะ app อยู่ background หรือไม่
      if (_lastActivity != null) {
        final elapsed = DateTime.now().difference(_lastActivity!);
        if (elapsed.inMinutes >= timeoutMinutes) {
          onSessionExpired();
          return;
        }
      }
      _resetTimer();
    } else if (state == AppLifecycleState.paused) {
      _timer?.cancel();
    }
  }

  void dispose() {
    _timer?.cancel();
    WidgetsBinding.instance.removeObserver(this);
  }
}
```

## สรุป

Enterprise Architecture:
1. **Flavors** - แยก development, staging, production อย่างชัดเจน
2. **Security** - Certificate pinning, biometric, device security check
3. **MDM** - รองรับ IT-managed configurations
4. **Session management** - Automatic timeout สำหรับ security

## แบบทดสอบ

1. อธิบายความแตกต่างระหว่าง debug, profile และ release build modes
2. ทำไม Certificate Pinning สำคัญสำหรับ enterprise apps?
3. สร้าง `ScreenRecordingPrevention` widget สำหรับ sensitive screens
4. อธิบาย MDM คืออะไรและส่งผลต่อ app development อย่างไร
