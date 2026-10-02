# Part 63: Security Best Practices ใน Flutter

## ความสำคัญของ Security

Mobile app security ปกป้อง:
- ข้อมูลส่วนตัวของผู้ใช้
- Token และ credentials
- Communication ระหว่างแอปและ server
- ป้องกันการ reverse engineering

```yaml
# pubspec.yaml
dependencies:
  flutter_secure_storage: ^9.0.0
  local_auth: ^2.3.0
  http: ^1.2.0
  dio: ^5.4.0
  
dev_dependencies:
  # สำหรับ obfuscation
  # (ใช้ flutter build flags แทน)
```

---

## 1. Certificate Pinning

### ป้องกัน Man-in-the-Middle Attack

```dart
// services/pinned_http_client.dart
import 'dart:io';
import 'package:http/http.dart' as http;

class PinnedHttpClient extends http.BaseClient {
  final http.Client _inner;
  
  // SHA-256 fingerprints ของ server certificate
  static const _pinnedCertificates = [
    'sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=', // Primary
    'sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=', // Backup
  ];
  
  PinnedHttpClient() : _inner = _createPinnedClient();
  
  static http.Client _createPinnedClient() {
    final securityContext = SecurityContext.defaultContext;
    
    // โหลด certificate จาก assets
    // (ต้อง add certificate ไปยัง assets ก่อน)
    return http.Client();
  }
  
  @override
  Future<http.StreamedResponse> send(http.BaseRequest request) {
    return _inner.send(request);
  }
}

// Certificate Pinning ด้วย Dio
class SecureApiClient {
  late final Dio _dio;
  
  SecureApiClient() {
    _dio = Dio(BaseOptions(
      baseUrl: 'https://api.example.com',
      connectTimeout: Duration(seconds: 10),
      receiveTimeout: Duration(seconds: 30),
    ));
    
    // เพิ่ม certificate pinning interceptor
    (_dio.httpClientAdapter as DefaultHttpClientAdapter)
        .onHttpClientCreate = (client) {
      // ตรวจสอบ certificate
      client.badCertificateCallback = (cert, host, port) {
        // ตรวจสอบ fingerprint
        return _validateCertificate(cert);
      };
      return client;
    };
    
    _dio.interceptors.add(CertificatePinningInterceptor());
  }
  
  bool _validateCertificate(X509Certificate cert) {
    final certBytes = cert.der;
    final certSha256 = sha256.convert(certBytes).bytes;
    final certBase64 = base64Encode(certSha256);
    
    // ตรวจสอบว่า certificate match กับที่ pin ไว้
    for (final pin in _pinnedFingerprints) {
      if (pin == 'sha256/$certBase64') {
        return false; // return false = certificate is valid (don't reject)
      }
    }
    
    return true; // return true = reject certificate
  }
  
  static const _pinnedFingerprints = [
    'sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=',
    'sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=',
  ];
}

// Certificate Pinning Interceptor
class CertificatePinningInterceptor extends Interceptor {
  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    if (err.type == DioExceptionType.badCertificate) {
      // Log security event
      SecurityService().logSecurityEvent(
        'certificate_mismatch',
        data: {'url': err.requestOptions.uri.toString()},
      );
      
      // Return custom error
      handler.reject(
        DioException(
          requestOptions: err.requestOptions,
          error: 'Security Error: Certificate validation failed',
          type: DioExceptionType.badCertificate,
        ),
      );
      return;
    }
    
    handler.next(err);
  }
}
```

---

## 2. Code Obfuscation

### Flutter Obfuscation

```bash
# Build พร้อม obfuscation
# Android
flutter build apk --obfuscate --split-debug-info=./debug-info/

# iOS
flutter build ios --obfuscate --split-debug-info=./debug-info/

# AAB
flutter build appbundle --obfuscate --split-debug-info=./debug-info/
```

### ProGuard Rules สำหรับ Android

```pro
# android/app/proguard-rules.pro

# Flutter
-keep class io.flutter.** { *; }
-keep class io.flutter.plugins.** { *; }

# Firebase
-keep class com.google.firebase.** { *; }

# ป้องกัน model classes จากการถูก obfuscate
-keep class com.example.myapp.models.** { *; }

# ป้องกัน serializable classes
-keepnames class * implements java.io.Serializable

# Gson
-keepattributes Signature
-keepattributes *Annotation*
-keep class sun.misc.Unsafe { *; }
-keep class com.google.gson.** { *; }

# OkHttp
-dontwarn okhttp3.**
-keep class okhttp3.** { *; }
```

### ซ่อน API Keys

```dart
// ❌ อย่าเก็บ API keys ใน code โดยตรง
const apiKey = 'my-secret-api-key-12345';

// ✅ ใช้ dart-define สำหรับ secrets
// Build command:
// flutter run --dart-define=API_KEY=my-secret-key
// flutter build apk --dart-define=API_KEY=my-secret-key

class AppConfig {
  // อ่านจาก environment ที่ inject ตอน build
  static const String apiKey = String.fromEnvironment('API_KEY');
  static const String baseUrl = String.fromEnvironment(
    'BASE_URL',
    defaultValue: 'https://api.example.com',
  );
  
  // ตรวจสอบว่ามีค่าครบ
  static void validate() {
    assert(apiKey.isNotEmpty, 'API_KEY is not set');
    assert(baseUrl.isNotEmpty, 'BASE_URL is not set');
  }
}

// .env file (อย่า commit ไปยัง git)
// API_KEY=your-actual-api-key
// BASE_URL=https://api.production.com

// launch.json สำหรับ VS Code
// {
//   "configurations": [{
//     "name": "Debug",
//     "args": [
//       "--dart-define=API_KEY=dev-key",
//       "--dart-define=BASE_URL=https://api.dev.com"
//     ]
//   }]
// }
```

---

## 3. Secure Storage

### flutter_secure_storage

```dart
// services/secure_storage_service.dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class SecureStorageService {
  static const _storage = FlutterSecureStorage(
    aOptions: AndroidOptions(
      encryptedSharedPreferences: true, // ใช้ EncryptedSharedPreferences
    ),
    iOptions: IOSOptions(
      accessibility: KeychainAccessibility.first_unlock_this_device,
      // ข้อมูลจะถูก delete ถ้า app ถูกถอนการติดตั้ง
    ),
  );
  
  // Keys
  static const _accessTokenKey = 'access_token';
  static const _refreshTokenKey = 'refresh_token';
  static const _userIdKey = 'user_id';
  static const _pinKey = 'app_pin';
  
  // Token management
  static Future<void> saveTokens({
    required String accessToken,
    required String refreshToken,
  }) async {
    await Future.wait([
      _storage.write(key: _accessTokenKey, value: accessToken),
      _storage.write(key: _refreshTokenKey, value: refreshToken),
    ]);
  }
  
  static Future<String?> getAccessToken() async {
    return await _storage.read(key: _accessTokenKey);
  }
  
  static Future<String?> getRefreshToken() async {
    return await _storage.read(key: _refreshTokenKey);
  }
  
  static Future<void> clearTokens() async {
    await Future.wait([
      _storage.delete(key: _accessTokenKey),
      _storage.delete(key: _refreshTokenKey),
    ]);
  }
  
  // User data
  static Future<void> saveUserId(String userId) async {
    await _storage.write(key: _userIdKey, value: userId);
  }
  
  static Future<String?> getUserId() async {
    return await _storage.read(key: _userIdKey);
  }
  
  // PIN
  static Future<void> savePin(String pin) async {
    // Hash PIN ก่อนเก็บ
    final hashedPin = _hashPin(pin);
    await _storage.write(key: _pinKey, value: hashedPin);
  }
  
  static Future<bool> verifyPin(String pin) async {
    final stored = await _storage.read(key: _pinKey);
    if (stored == null) return false;
    
    final hashedPin = _hashPin(pin);
    return hashedPin == stored;
  }
  
  static String _hashPin(String pin) {
    final bytes = utf8.encode(pin);
    final digest = sha256.convert(bytes);
    return digest.toString();
  }
  
  // Clear all data (logout)
  static Future<void> clearAll() async {
    await _storage.deleteAll();
  }
}
```

### Token Refresh Pattern

```dart
// interceptors/auth_interceptor.dart
class AuthInterceptor extends Interceptor {
  bool _isRefreshing = false;
  final List<RequestOptions> _pendingRequests = [];
  
  @override
  Future<void> onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) async {
    final token = await SecureStorageService.getAccessToken();
    if (token != null) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    handler.next(options);
  }
  
  @override
  Future<void> onError(
    DioException err,
    ErrorInterceptorHandler handler,
  ) async {
    if (err.response?.statusCode == 401) {
      if (_isRefreshing) {
        // เพิ่ม request ในคิวรอ refresh
        _pendingRequests.add(err.requestOptions);
        return;
      }
      
      _isRefreshing = true;
      
      try {
        // Refresh token
        final newTokens = await _refreshTokens();
        
        // Retry request เดิม
        final response = await _retryRequest(
          err.requestOptions,
          newTokens.accessToken,
        );
        
        // Retry requests ที่รออยู่
        for (final request in _pendingRequests) {
          await _retryRequest(request, newTokens.accessToken);
        }
        _pendingRequests.clear();
        
        handler.resolve(response);
      } catch (e) {
        // Refresh ล้มเหลว - logout
        _pendingRequests.clear();
        await _logout();
        handler.reject(err);
      } finally {
        _isRefreshing = false;
      }
    } else {
      handler.next(err);
    }
  }
  
  Future<TokenPair> _refreshTokens() async {
    final refreshToken = await SecureStorageService.getRefreshToken();
    if (refreshToken == null) throw Exception('No refresh token');
    
    final response = await Dio().post(
      'https://api.example.com/auth/refresh',
      data: {'refresh_token': refreshToken},
    );
    
    final tokens = TokenPair.fromJson(response.data);
    await SecureStorageService.saveTokens(
      accessToken: tokens.accessToken,
      refreshToken: tokens.refreshToken,
    );
    
    return tokens;
  }
  
  Future<Response> _retryRequest(
    RequestOptions options,
    String newToken,
  ) async {
    options.headers['Authorization'] = 'Bearer $newToken';
    return await Dio().fetch(options);
  }
  
  Future<void> _logout() async {
    await SecureStorageService.clearAll();
    // Navigate to login
    navigatorKey.currentState?.pushNamedAndRemoveUntil(
      '/login',
      (route) => false,
    );
  }
}
```

---

## 4. Biometric Authentication

### Local Authentication

```dart
// services/biometric_service.dart
import 'package:local_auth/local_auth.dart';

class BiometricService {
  final _localAuth = LocalAuthentication();
  
  // ตรวจสอบว่าอุปกรณ์รองรับ biometrics
  Future<bool> isAvailable() async {
    try {
      return await _localAuth.canCheckBiometrics;
    } on PlatformException {
      return false;
    }
  }
  
  // ตรวจสอบประเภท biometrics ที่รองรับ
  Future<List<BiometricType>> getAvailableBiometrics() async {
    try {
      return await _localAuth.getAvailableBiometrics();
    } on PlatformException {
      return [];
    }
  }
  
  // Authenticate ด้วย biometrics
  Future<BiometricResult> authenticate({
    String localizedReason = 'กรุณายืนยันตัวตน',
    bool useStrongBiometrics = true,
  }) async {
    try {
      final bool isDeviceSupported =
          await _localAuth.isDeviceSupported();
      if (!isDeviceSupported) {
        return BiometricResult.deviceNotSupported;
      }
      
      final bool canCheck = await _localAuth.canCheckBiometrics;
      if (!canCheck) {
        return BiometricResult.notAvailable;
      }
      
      final bool didAuthenticate = await _localAuth.authenticate(
        localizedReason: localizedReason,
        options: AuthenticationOptions(
          biometricOnly: false, // อนุญาต PIN เป็น fallback
          stickyAuth: true, // ค้าง dialog ไว้แม้ app switch
          useErrorDialogs: true, // แสดง error dialog อัตโนมัติ
        ),
      );
      
      return didAuthenticate
          ? BiometricResult.success
          : BiometricResult.failed;
    } on PlatformException catch (e) {
      switch (e.code) {
        case 'NotEnrolled':
          return BiometricResult.notEnrolled;
        case 'LockedOut':
          return BiometricResult.lockedOut;
        case 'PermanentlyLockedOut':
          return BiometricResult.permanentlyLockedOut;
        default:
          return BiometricResult.error;
      }
    }
  }
  
  // Stop authentication (ถ้ากำลัง authenticate อยู่)
  Future<void> stopAuthentication() async {
    await _localAuth.stopAuthentication();
  }
}

enum BiometricResult {
  success,
  failed,
  notAvailable,
  notEnrolled,
  lockedOut,
  permanentlyLockedOut,
  deviceNotSupported,
  error,
}

// BiometricButton Widget
class BiometricButton extends StatelessWidget {
  final VoidCallback onSuccess;
  final VoidCallback? onFailed;

  const BiometricButton({required this.onSuccess, this.onFailed});

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<bool>(
      future: BiometricService().isAvailable(),
      builder: (context, snapshot) {
        if (snapshot.data != true) return SizedBox.shrink();
        
        return IconButton(
          icon: Icon(Icons.fingerprint, size: 48),
          onPressed: () async {
            final result = await BiometricService().authenticate(
              localizedReason: 'ยืนยันตัวตนเพื่อเข้าสู่ระบบ',
            );
            
            if (result == BiometricResult.success) {
              onSuccess();
            } else {
              onFailed?.call();
              _showError(context, result);
            }
          },
        );
      },
    );
  }
  
  void _showError(BuildContext context, BiometricResult result) {
    String message;
    switch (result) {
      case BiometricResult.notEnrolled:
        message = 'กรุณาตั้งค่า fingerprint ในการตั้งค่าอุปกรณ์';
        break;
      case BiometricResult.lockedOut:
        message = 'ล็อกอยู่ชั่วคราว กรุณาลองใหม่ในอีกสักครู่';
        break;
      default:
        message = 'ยืนยันตัวตนไม่สำเร็จ';
    }
    
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text(message)),
    );
  }
}
```

---

## 5. Root/Jailbreak Detection

```dart
// services/device_security_service.dart
class DeviceSecurityService {
  // ตรวจสอบว่าอุปกรณ์ถูก root/jailbreak
  Future<SecurityStatus> checkDeviceSecurity() async {
    if (Platform.isAndroid) {
      return await _checkAndroidSecurity();
    } else if (Platform.isIOS) {
      return await _checkIOSSecurity();
    }
    return SecurityStatus.unknown;
  }

  Future<SecurityStatus> _checkAndroidSecurity() async {
    // ตรวจสอบสัญญาณของการ root
    final checks = await Future.wait([
      _checkSuperuserApp(),
      _checkSuBinary(),
      _checkBusyBox(),
      _checkWritableSystemPartition(),
    ]);

    final riskScore = checks.where((c) => c).length;

    if (riskScore == 0) return SecurityStatus.secure;
    if (riskScore <= 1) return SecurityStatus.suspicious;
    return SecurityStatus.compromised;
  }

  Future<bool> _checkSuperuserApp() async {
    final superuserPaths = [
      '/system/app/Superuser.apk',
      '/sbin/su',
      '/system/bin/su',
      '/system/xbin/su',
      '/data/local/xbin/su',
      '/data/local/bin/su',
      '/system/sd/xbin/su',
      '/system/bin/failsafe/su',
      '/data/local/su',
    ];

    for (final path in superuserPaths) {
      if (await File(path).exists()) return true;
    }
    return false;
  }

  Future<bool> _checkSuBinary() async {
    try {
      final result = await Process.run('which', ['su']);
      return result.exitCode == 0;
    } catch (_) {
      return false;
    }
  }

  Future<bool> _checkBusyBox() async {
    return await File('/system/xbin/busybox').exists();
  }

  Future<bool> _checkWritableSystemPartition() async {
    try {
      final file = File('/system/test_write');
      await file.writeAsString('test');
      await file.delete();
      return true; // สามารถเขียนได้ = root
    } catch (_) {
      return false; // ไม่สามารถเขียน = ปลอดภัย
    }
  }

  Future<SecurityStatus> _checkIOSSecurity() async {
    final checks = await Future.wait([
      _checkCydiaInstalled(),
      _checkWritableFileSystem(),
      _checkSuspiciousFiles(),
    ]);

    final riskScore = checks.where((c) => c).length;

    if (riskScore == 0) return SecurityStatus.secure;
    if (riskScore <= 1) return SecurityStatus.suspicious;
    return SecurityStatus.compromised;
  }

  Future<bool> _checkCydiaInstalled() async {
    return await File('/Applications/Cydia.app').exists();
  }

  Future<bool> _checkWritableFileSystem() async {
    try {
      final file = File('/private/test_write.txt');
      await file.writeAsString('test');
      await file.delete();
      return true;
    } catch (_) {
      return false;
    }
  }

  Future<bool> _checkSuspiciousFiles() async {
    final jailbreakPaths = [
      '/Library/MobileSubstrate/MobileSubstrate.dylib',
      '/bin/bash',
      '/usr/sbin/sshd',
      '/etc/apt',
    ];

    for (final path in jailbreakPaths) {
      if (await File(path).exists()) return true;
    }
    return false;
  }
}

enum SecurityStatus { secure, suspicious, compromised, unknown }

// ตรวจสอบตอน app start
class SecurityCheckWidget extends StatefulWidget {
  final Widget child;

  const SecurityCheckWidget({required this.child});

  @override
  _SecurityCheckWidgetState createState() => _SecurityCheckWidgetState();
}

class _SecurityCheckWidgetState extends State<SecurityCheckWidget> {
  SecurityStatus? _status;

  @override
  void initState() {
    super.initState();
    _checkSecurity();
  }

  Future<void> _checkSecurity() async {
    final status = await DeviceSecurityService().checkDeviceSecurity();
    setState(() => _status = status);

    if (status == SecurityStatus.compromised) {
      _showCompromisedAlert();
    }
  }

  void _showCompromisedAlert() {
    showDialog(
      context: context,
      barrierDismissible: false,
      builder: (context) => AlertDialog(
        title: Text('ความปลอดภัยของอุปกรณ์'),
        content: Text(
          'อุปกรณ์ของคุณอาจถูก root หรือ jailbreak '
          'ซึ่งอาจทำให้ข้อมูลของคุณไม่ปลอดภัย\n\n'
          'แอปนี้อาจทำงานไม่ถูกต้องบนอุปกรณ์ที่ถูก modify',
        ),
        actions: [
          TextButton(
            onPressed: () {
              Navigator.pop(context);
              // ยังคงอนุญาตให้ใช้งาน แต่แสดง warning
            },
            child: Text('รับทราบ'),
          ),
          ElevatedButton(
            onPressed: () {
              // บังคับออกจากแอป
              exit(0);
            },
            style: ElevatedButton.styleFrom(
              backgroundColor: Colors.red,
            ),
            child: Text('ออกจากแอป'),
          ),
        ],
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return widget.child;
  }
}
```

---

## Workshop: Secure Login Flow

```dart
// screens/secure_login_screen.dart
class SecureLoginScreen extends StatefulWidget {
  @override
  _SecureLoginScreenState createState() => _SecureLoginScreenState();
}

class _SecureLoginScreenState extends State<SecureLoginScreen> {
  final _formKey = GlobalKey<FormState>();
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();
  final _biometricService = BiometricService();
  final _secureStorage = SecureStorageService();
  
  bool _isLoading = false;
  bool _showBiometric = false;
  int _failedAttempts = 0;
  DateTime? _lockoutUntil;

  @override
  void initState() {
    super.initState();
    _checkBiometricAvailability();
    _checkSavedSession();
  }

  Future<void> _checkBiometricAvailability() async {
    final isAvailable = await _biometricService.isAvailable();
    final hasSavedToken = await _secureStorage.getAccessToken() != null;
    
    setState(() {
      _showBiometric = isAvailable && hasSavedToken;
    });
  }

  Future<void> _checkSavedSession() async {
    final token = await _secureStorage.getAccessToken();
    if (token != null) {
      // Validate token ยังใช้ได้
      final isValid = await _validateToken(token);
      if (isValid) {
        _navigateToHome();
      }
    }
  }

  bool _isLockedOut() {
    if (_lockoutUntil == null) return false;
    if (DateTime.now().isBefore(_lockoutUntil!)) return true;
    
    // Reset lockout
    _failedAttempts = 0;
    _lockoutUntil = null;
    return false;
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: SafeArea(
        child: Padding(
          padding: EdgeInsets.all(24),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.stretch,
            children: [
              SizedBox(height: 60),
              
              // Logo
              Center(
                child: FlutterLogo(size: 80),
              ),
              SizedBox(height: 48),
              
              Text(
                'เข้าสู่ระบบ',
                style: Theme.of(context).textTheme.headlineMedium,
                textAlign: TextAlign.center,
              ),
              SizedBox(height: 32),
              
              // Lockout message
              if (_isLockedOut())
                Container(
                  padding: EdgeInsets.all(12),
                  decoration: BoxDecoration(
                    color: Colors.red.shade50,
                    borderRadius: BorderRadius.circular(8),
                  ),
                  child: Text(
                    'ล็อกอินล้มเหลวหลายครั้ง '
                    'กรุณารอ ${_lockoutUntil!.difference(DateTime.now()).inMinutes} นาที',
                    style: TextStyle(color: Colors.red),
                    textAlign: TextAlign.center,
                  ),
                ),
              
              SizedBox(height: 16),
              
              Form(
                key: _formKey,
                child: Column(
                  children: [
                    // Email
                    TextFormField(
                      controller: _emailController,
                      keyboardType: TextInputType.emailAddress,
                      decoration: InputDecoration(
                        labelText: 'อีเมล',
                        prefixIcon: Icon(Icons.email),
                        border: OutlineInputBorder(),
                      ),
                      validator: (v) {
                        if (v?.isEmpty == true) return 'กรุณากรอกอีเมล';
                        if (!RegExp(r'^[^@]+@[^@]+\.[^@]+').hasMatch(v!)) {
                          return 'รูปแบบอีเมลไม่ถูกต้อง';
                        }
                        return null;
                      },
                    ),
                    SizedBox(height: 16),
                    
                    // Password
                    TextFormField(
                      controller: _passwordController,
                      obscureText: true,
                      decoration: InputDecoration(
                        labelText: 'รหัสผ่าน',
                        prefixIcon: Icon(Icons.lock),
                        border: OutlineInputBorder(),
                      ),
                      validator: (v) =>
                          v?.isEmpty == true ? 'กรุณากรอกรหัสผ่าน' : null,
                    ),
                  ],
                ),
              ),
              SizedBox(height: 24),
              
              // Login button
              ElevatedButton(
                onPressed: _isLockedOut() || _isLoading ? null : _handleLogin,
                style: ElevatedButton.styleFrom(
                  minimumSize: Size(double.infinity, 52),
                ),
                child: _isLoading
                    ? CircularProgressIndicator(color: Colors.white)
                    : Text('เข้าสู่ระบบ', style: TextStyle(fontSize: 16)),
              ),
              
              // Biometric login
              if (_showBiometric) ...[
                SizedBox(height: 16),
                Center(
                  child: Text('หรือ', style: TextStyle(color: Colors.grey)),
                ),
                SizedBox(height: 16),
                Center(
                  child: BiometricButton(
                    onSuccess: _handleBiometricSuccess,
                    onFailed: () {
                      ScaffoldMessenger.of(context).showSnackBar(
                        SnackBar(content: Text('ยืนยันตัวตนไม่สำเร็จ')),
                      );
                    },
                  ),
                ),
              ],
            ],
          ),
        ),
      ),
    );
  }

  Future<void> _handleLogin() async {
    if (!_formKey.currentState!.validate()) return;
    
    setState(() => _isLoading = true);
    
    try {
      final result = await AuthService().login(
        email: _emailController.text,
        password: _passwordController.text,
      );
      
      // บันทึก tokens อย่างปลอดภัย
      await _secureStorage.saveTokens(
        accessToken: result.accessToken,
        refreshToken: result.refreshToken,
      );
      
      // Reset failed attempts
      _failedAttempts = 0;
      
      _navigateToHome();
    } catch (e) {
      _failedAttempts++;
      
      // Lockout หลังจาก 5 ครั้ง
      if (_failedAttempts >= 5) {
        setState(() {
          _lockoutUntil = DateTime.now().add(Duration(minutes: 5));
        });
      }
      
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(
            content: Text('อีเมลหรือรหัสผ่านไม่ถูกต้อง'),
            backgroundColor: Colors.red,
          ),
        );
      }
    } finally {
      if (mounted) setState(() => _isLoading = false);
    }
  }

  Future<void> _handleBiometricSuccess() async {
    final token = await _secureStorage.getAccessToken();
    if (token != null) {
      _navigateToHome();
    }
  }

  void _navigateToHome() {
    Navigator.pushReplacementNamed(context, '/home');
  }

  Future<bool> _validateToken(String token) async {
    try {
      await ApiService().validateToken(token);
      return true;
    } catch (_) {
      return false;
    }
  }

  @override
  void dispose() {
    _emailController.dispose();
    _passwordController.dispose();
    super.dispose();
  }
}
```

---

## สรุป

Security Best Practices ใน Flutter ประกอบด้วย:

1. **Certificate Pinning** - ป้องกัน MITM attacks
2. **Obfuscation** - ซ่อน code จาก reverse engineering
3. **Secure Storage** - เก็บ sensitive data อย่างปลอดภัย
4. **Biometric Auth** - ยืนยันตัวตนที่สะดวกและปลอดภัย
5. **Root/Jailbreak Detection** - ตรวจสอบความปลอดภัยของอุปกรณ์
6. **Token Security** - จัดการ tokens อย่างถูกต้อง
