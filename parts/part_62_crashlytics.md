# Part 62: Crashlytics และ Analytics

## ความสำคัญของ Monitoring

การ monitor แอปหลัง release ช่วยให้:
- รู้จัก crash ก่อนที่ user จะรายงาน
- เข้าใจพฤติกรรมผู้ใช้
- ตัดสินใจ product ด้วยข้อมูล
- ทดสอบ A/B สำหรับ feature ใหม่

```yaml
# pubspec.yaml
dependencies:
  firebase_core: ^3.0.0
  firebase_crashlytics: ^4.0.0
  firebase_analytics: ^11.0.0
  firebase_remote_config: ^5.0.0
```

---

## 1. Firebase Crashlytics Setup

### การตั้งค่า Crashlytics

```dart
// main.dart
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_crashlytics/firebase_crashlytics.dart';
import 'package:flutter/foundation.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );
  
  // Setup Crashlytics
  await _setupCrashlytics();
  
  runApp(MyApp());
}

Future<void> _setupCrashlytics() async {
  // Enable Crashlytics เฉพาะ production
  // (ปิดใน debug mode เพื่อไม่ให้รบกวน development)
  await FirebaseCrashlytics.instance
      .setCrashlyticsCollectionEnabled(!kDebugMode);
  
  // จับ Flutter framework errors
  FlutterError.onError = (errorDetails) {
    FirebaseCrashlytics.instance.recordFlutterFatalError(errorDetails);
  };
  
  // จับ async errors ที่ Flutter ไม่ handle
  PlatformDispatcher.instance.onError = (error, stack) {
    FirebaseCrashlytics.instance.recordError(error, stack, fatal: true);
    return true;
  };
}
```

### Error Boundary Widget

```dart
// widgets/error_boundary.dart
class ErrorBoundary extends StatefulWidget {
  final Widget child;
  final Widget Function(Object error, StackTrace? stack)? errorBuilder;

  const ErrorBoundary({required this.child, this.errorBuilder});

  @override
  _ErrorBoundaryState createState() => _ErrorBoundaryState();
}

class _ErrorBoundaryState extends State<ErrorBoundary> {
  Object? _error;
  StackTrace? _stackTrace;

  @override
  Widget build(BuildContext context) {
    if (_error != null) {
      return widget.errorBuilder?.call(_error!, _stackTrace) ??
          _DefaultErrorWidget(
            error: _error!,
            onRetry: () => setState(() {
              _error = null;
              _stackTrace = null;
            }),
          );
    }

    return widget.child;
  }
}

class _DefaultErrorWidget extends StatelessWidget {
  final Object error;
  final VoidCallback onRetry;

  const _DefaultErrorWidget({required this.error, required this.onRetry});

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Padding(
        padding: EdgeInsets.all(24),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(Icons.error_outline, size: 64, color: Colors.red),
            SizedBox(height: 16),
            Text(
              'เกิดข้อผิดพลาด',
              style: TextStyle(fontSize: 20, fontWeight: FontWeight.bold),
            ),
            SizedBox(height: 8),
            Text(
              'กรุณาลองใหม่อีกครั้ง',
              textAlign: TextAlign.center,
              style: TextStyle(color: Colors.grey[600]),
            ),
            SizedBox(height: 24),
            ElevatedButton(
              onPressed: onRetry,
              child: Text('ลองใหม่'),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## 2. Custom Error Logging

### CrashlyticsService

```dart
// services/crashlytics_service.dart
class CrashlyticsService {
  static final CrashlyticsService _instance = CrashlyticsService._();
  factory CrashlyticsService() => _instance;
  CrashlyticsService._();

  final _crashlytics = FirebaseCrashlytics.instance;

  // ตั้งค่า user identifier
  Future<void> setUser({
    required String userId,
    String? email,
    String? name,
  }) async {
    await _crashlytics.setUserIdentifier(userId);
    if (email != null) await _crashlytics.setCustomKey('email', email);
    if (name != null) await _crashlytics.setCustomKey('name', name);
  }

  // เพิ่ม custom key-value pairs
  Future<void> setCustomData(Map<String, dynamic> data) async {
    for (final entry in data.entries) {
      final value = entry.value;
      if (value is String) {
        await _crashlytics.setCustomKey(entry.key, value);
      } else if (value is int) {
        await _crashlytics.setCustomKey(entry.key, value);
      } else if (value is double) {
        await _crashlytics.setCustomKey(entry.key, value);
      } else if (value is bool) {
        await _crashlytics.setCustomKey(entry.key, value);
      } else {
        await _crashlytics.setCustomKey(entry.key, value.toString());
      }
    }
  }

  // Log non-fatal error
  Future<void> logError(
    Object error,
    StackTrace? stackTrace, {
    String? reason,
    bool fatal = false,
    Map<String, dynamic>? context,
  }) async {
    if (context != null) {
      await setCustomData(context);
    }

    await _crashlytics.recordError(
      error,
      stackTrace,
      reason: reason,
      fatal: fatal,
    );
  }

  // Log breadcrumb (สำหรับ debug)
  Future<void> log(String message) async {
    await _crashlytics.log(message);
  }

  // ทดสอบ crash (debug only)
  void testCrash() {
    if (kDebugMode) {
      _crashlytics.crash();
    }
  }
}

// ตัวอย่างการใช้งาน
class UserService {
  final _crashlytics = CrashlyticsService();

  Future<User?> getUser(String userId) async {
    try {
      await _crashlytics.log('Fetching user: $userId');
      
      final user = await _apiService.getUser(userId);
      return user;
    } catch (e, stack) {
      await _crashlytics.logError(
        e,
        stack,
        reason: 'Failed to fetch user',
        context: {'userId': userId},
      );
      return null;
    }
  }
}
```

### Global Error Handler

```dart
// services/error_handler.dart
class ErrorHandler {
  static final _crashlytics = CrashlyticsService();
  static final _analytics = AnalyticsService();

  static Future<void> handleError(
    Object error,
    StackTrace? stack, {
    bool fatal = false,
    Map<String, dynamic>? context,
  }) async {
    // Log to Crashlytics
    await _crashlytics.logError(
      error,
      stack,
      fatal: fatal,
      context: context,
    );

    // Log to Analytics
    await _analytics.logEvent(
      'app_error',
      parameters: {
        'error_type': error.runtimeType.toString(),
        'is_fatal': fatal,
        ...?context,
      },
    );

    // Show user-friendly message ถ้าไม่ fatal
    if (!fatal) {
      _showErrorSnackbar();
    }
  }

  static void _showErrorSnackbar() {
    final context = navigatorKey.currentContext;
    if (context != null) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(
          content: Text('เกิดข้อผิดพลาด กรุณาลองใหม่อีกครั้ง'),
          backgroundColor: Colors.red,
          action: SnackBarAction(
            label: 'ปิด',
            textColor: Colors.white,
            onPressed: () {},
          ),
        ),
      );
    }
  }
}
```

---

## 3. Firebase Analytics

### Analytics Setup และ Events พื้นฐาน

```dart
// services/analytics_service.dart
import 'package:firebase_analytics/firebase_analytics.dart';

class AnalyticsService {
  static final AnalyticsService _instance = AnalyticsService._();
  factory AnalyticsService() => _instance;
  AnalyticsService._();

  final FirebaseAnalytics _analytics = FirebaseAnalytics.instance;

  // Screen tracking
  Future<void> logScreenView({
    required String screenName,
    String? screenClass,
  }) async {
    await _analytics.logScreenView(
      screenName: screenName,
      screenClass: screenClass ?? screenName,
    );
  }

  // User login
  Future<void> logLogin({required String method}) async {
    await _analytics.logLogin(loginMethod: method);
  }

  // User signup
  Future<void> logSignUp({required String method}) async {
    await _analytics.logSignUp(signUpMethod: method);
  }

  // Search
  Future<void> logSearch({required String searchTerm}) async {
    await _analytics.logSearch(searchTerm: searchTerm);
  }

  // E-commerce events
  Future<void> logViewItem({
    required String itemId,
    required String itemName,
    required String itemCategory,
    required double price,
    String? currency,
  }) async {
    await _analytics.logViewItem(
      currency: currency ?? 'THB',
      value: price,
      items: [
        AnalyticsEventItem(
          itemId: itemId,
          itemName: itemName,
          itemCategory: itemCategory,
          price: price,
        ),
      ],
    );
  }

  Future<void> logAddToCart({
    required String itemId,
    required String itemName,
    required double price,
    required int quantity,
  }) async {
    await _analytics.logAddToCart(
      currency: 'THB',
      value: price * quantity,
      items: [
        AnalyticsEventItem(
          itemId: itemId,
          itemName: itemName,
          price: price,
          quantity: quantity,
        ),
      ],
    );
  }

  Future<void> logPurchase({
    required String transactionId,
    required double revenue,
    required List<Map<String, dynamic>> items,
  }) async {
    await _analytics.logPurchase(
      transactionId: transactionId,
      currency: 'THB',
      value: revenue,
      items: items
          .map((item) => AnalyticsEventItem(
                itemId: item['id'] as String,
                itemName: item['name'] as String,
                price: item['price'] as double,
                quantity: item['quantity'] as int,
              ))
          .toList(),
    );
  }

  // Custom events
  Future<void> logEvent({
    required String name,
    Map<String, dynamic>? parameters,
  }) async {
    await _analytics.logEvent(
      name: name,
      parameters: parameters?.map(
        (key, value) => MapEntry(key, value?.toString() ?? ''),
      ),
    );
  }

  // User properties
  Future<void> setUserProperty({
    required String name,
    required String? value,
  }) async {
    await _analytics.setUserProperty(name: name, value: value);
  }

  // Set user ID
  Future<void> setUserId(String? id) async {
    await _analytics.setUserId(id: id);
  }
}
```

### Navigator Observer สำหรับ Screen Tracking

```dart
// analytics/analytics_observer.dart
class AnalyticsRouteObserver extends NavigatorObserver {
  final AnalyticsService _analytics;

  AnalyticsRouteObserver(this._analytics);

  @override
  void didPush(Route route, Route? previousRoute) {
    super.didPush(route, previousRoute);
    _trackScreen(route);
  }

  @override
  void didReplace({Route? newRoute, Route? oldRoute}) {
    super.didReplace(newRoute: newRoute, oldRoute: oldRoute);
    if (newRoute != null) _trackScreen(newRoute);
  }

  void _trackScreen(Route route) {
    final screenName = route.settings.name;
    if (screenName != null) {
      _analytics.logScreenView(screenName: screenName);
    }
  }
}

// การใช้งานใน MaterialApp
MaterialApp(
  navigatorObservers: [
    AnalyticsRouteObserver(AnalyticsService()),
    FirebaseAnalyticsObserver(analytics: FirebaseAnalytics.instance),
  ],
  // ...
)
```

---

## 4. Custom Events

### Event Schema ที่ดี

```dart
// analytics/events.dart
// กำหนด event names เป็น constants เพื่อป้องกัน typos
class AnalyticsEvents {
  // User
  static const String userLogin = 'user_login';
  static const String userLogout = 'user_logout';
  static const String userSignUp = 'user_sign_up';
  static const String userProfileUpdate = 'user_profile_update';
  
  // Content
  static const String contentView = 'content_view';
  static const String contentShare = 'content_share';
  static const String contentLike = 'content_like';
  static const String contentBookmark = 'content_bookmark';
  
  // Search
  static const String searchPerformed = 'search_performed';
  static const String searchResultClick = 'search_result_click';
  static const String searchNoResults = 'search_no_results';
  
  // Onboarding
  static const String onboardingStart = 'onboarding_start';
  static const String onboardingStep = 'onboarding_step';
  static const String onboardingComplete = 'onboarding_complete';
  static const String onboardingSkip = 'onboarding_skip';
  
  // Feature usage
  static const String featureUsed = 'feature_used';
  static const String featureEnabled = 'feature_enabled';
  static const String featureDisabled = 'feature_disabled';
  
  // Error
  static const String appError = 'app_error';
  static const String networkError = 'network_error';
  static const String permissionDenied = 'permission_denied';
}

// Parameters
class AnalyticsParams {
  static const String userId = 'user_id';
  static const String screenName = 'screen_name';
  static const String contentId = 'content_id';
  static const String contentType = 'content_type';
  static const String searchTerm = 'search_term';
  static const String resultCount = 'result_count';
  static const String errorType = 'error_type';
  static const String errorMessage = 'error_message';
  static const String featureName = 'feature_name';
  static const String stepName = 'step_name';
  static const String stepNumber = 'step_number';
}

// ตัวอย่างการใช้งาน
class SearchScreen extends StatefulWidget {
  @override
  _SearchScreenState createState() => _SearchScreenState();
}

class _SearchScreenState extends State<SearchScreen> {
  final _analytics = AnalyticsService();
  
  Future<void> _performSearch(String query) async {
    // Log search event
    await _analytics.logEvent(
      name: AnalyticsEvents.searchPerformed,
      parameters: {
        AnalyticsParams.searchTerm: query,
        AnalyticsParams.screenName: 'search',
      },
    );
    
    final results = await _searchService.search(query);
    
    // Log results
    if (results.isEmpty) {
      await _analytics.logEvent(
        name: AnalyticsEvents.searchNoResults,
        parameters: {
          AnalyticsParams.searchTerm: query,
        },
      );
    }
    
    // Update UI
    setState(() => _results = results);
  }
}
```

---

## 5. A/B Testing ด้วย Remote Config

### Firebase Remote Config Setup

```dart
// services/remote_config_service.dart
import 'package:firebase_remote_config/firebase_remote_config.dart';

class RemoteConfigService {
  static final RemoteConfigService _instance = RemoteConfigService._();
  factory RemoteConfigService() => _instance;
  RemoteConfigService._();

  final _remoteConfig = FirebaseRemoteConfig.instance;

  Future<void> initialize() async {
    // ค่าเริ่มต้น (fallback ถ้าดึงจาก server ไม่ได้)
    await _remoteConfig.setDefaults({
      'show_new_ui': false,
      'max_free_downloads': 3,
      'checkout_button_color': 'blue',
      'onboarding_version': 'v1',
      'feature_dark_mode': false,
      'promotion_banner_enabled': false,
      'promotion_text': '',
    });

    // ตั้งค่า fetch interval
    await _remoteConfig.setConfigSettings(
      RemoteConfigSettings(
        fetchTimeout: Duration(seconds: 30),
        minimumFetchInterval: kDebugMode
            ? Duration.zero // ดึงทันทีใน debug
            : Duration(hours: 1), // ดึงทุก 1 ชั่วโมงใน production
      ),
    );

    // Fetch และ activate
    await _remoteConfig.fetchAndActivate();
  }

  // Getters สำหรับแต่ละ config
  bool get showNewUI => _remoteConfig.getBool('show_new_ui');
  int get maxFreeDownloads => _remoteConfig.getInt('max_free_downloads');
  String get checkoutButtonColor =>
      _remoteConfig.getString('checkout_button_color');
  bool get featureDarkMode => _remoteConfig.getBool('feature_dark_mode');
  bool get promotionBannerEnabled =>
      _remoteConfig.getBool('promotion_banner_enabled');
  String get promotionText => _remoteConfig.getString('promotion_text');

  // Listen for changes
  Stream<RemoteConfigUpdate> get updates => _remoteConfig.onConfigUpdated;

  // Refresh manually
  Future<bool> refresh() async {
    return await _remoteConfig.fetchAndActivate();
  }
}
```

### A/B Test Implementation

```dart
// ab_testing/ab_test_manager.dart
class ABTestManager {
  static final _remoteConfig = RemoteConfigService();
  static final _analytics = AnalyticsService();
  
  // ตรวจสอบว่า user อยู่ใน variant ไหน
  static String getVariant(String testName) {
    return FirebaseRemoteConfig.instance.getString(testName);
  }
  
  // Log exposure (เมื่อ user เห็น variant)
  static Future<void> logExposure({
    required String testName,
    required String variant,
  }) async {
    await _analytics.logEvent(
      name: 'ab_test_exposure',
      parameters: {
        'test_name': testName,
        'variant': variant,
      },
    );
  }
  
  // Log conversion
  static Future<void> logConversion({
    required String testName,
    required String variant,
    required String conversionEvent,
  }) async {
    await _analytics.logEvent(
      name: 'ab_test_conversion',
      parameters: {
        'test_name': testName,
        'variant': variant,
        'conversion_event': conversionEvent,
      },
    );
  }
}

// Widget สำหรับ A/B Test
class CheckoutButtonABTest extends StatelessWidget {
  final VoidCallback onPressed;

  const CheckoutButtonABTest({required this.onPressed});

  @override
  Widget build(BuildContext context) {
    final variant = ABTestManager.getVariant('checkout_button_test');
    
    // Log exposure เมื่อ widget build
    WidgetsBinding.instance.addPostFrameCallback((_) {
      ABTestManager.logExposure(
        testName: 'checkout_button_test',
        variant: variant,
      );
    });
    
    switch (variant) {
      case 'variant_a':
        return ElevatedButton(
          onPressed: () {
            ABTestManager.logConversion(
              testName: 'checkout_button_test',
              variant: variant,
              conversionEvent: 'checkout_click',
            );
            onPressed();
          },
          style: ElevatedButton.styleFrom(
            backgroundColor: Colors.green,
          ),
          child: Text('ชำระเงิน'),
        );
      
      case 'variant_b':
        return OutlinedButton.icon(
          onPressed: () {
            ABTestManager.logConversion(
              testName: 'checkout_button_test',
              variant: variant,
              conversionEvent: 'checkout_click',
            );
            onPressed();
          },
          icon: Icon(Icons.shopping_cart_checkout),
          label: Text('ดำเนินการชำระเงิน'),
        );
      
      default: // control
        return ElevatedButton(
          onPressed: () {
            ABTestManager.logConversion(
              testName: 'checkout_button_test',
              variant: variant,
              conversionEvent: 'checkout_click',
            );
            onPressed();
          },
          child: Text('ชำระเงิน'),
        );
    }
  }
}
```

---

## Workshop: Analytics Setup สมบูรณ์

```dart
// main.dart - Complete Analytics Setup
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );
  
  // Setup services
  await Future.wait([
    _setupCrashlytics(),
    RemoteConfigService().initialize(),
  ]);
  
  runApp(MyApp());
}

// app.dart
class MyApp extends StatelessWidget {
  final _analytics = AnalyticsService();
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      navigatorKey: navigatorKey,
      navigatorObservers: [
        FirebaseAnalyticsObserver(
          analytics: FirebaseAnalytics.instance,
        ),
      ],
      home: SplashScreen(),
    );
  }
}

// splash_screen.dart
class SplashScreen extends StatefulWidget {
  @override
  _SplashScreenState createState() => _SplashScreenState();
}

class _SplashScreenState extends State<SplashScreen> {
  final _analytics = AnalyticsService();
  final _remoteConfig = RemoteConfigService();
  
  @override
  void initState() {
    super.initState();
    _initialize();
  }
  
  Future<void> _initialize() async {
    // Log app open
    await _analytics.logEvent(name: 'app_open');
    
    // Check user session
    final user = await AuthService().getCurrentUser();
    
    if (user != null) {
      // Set analytics user
      await _analytics.setUserId(user.id);
      await _analytics.setUserProperty(
        name: 'account_type',
        value: user.accountType,
      );
      
      // Set Crashlytics user
      await CrashlyticsService().setUser(
        userId: user.id,
        email: user.email,
        name: user.name,
      );
      
      // Navigate to home
      Navigator.pushReplacementNamed(context, '/home');
    } else {
      Navigator.pushReplacementNamed(context, '/login');
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            FlutterLogo(size: 100),
            SizedBox(height: 24),
            CircularProgressIndicator(),
          ],
        ),
      ),
    );
  }
}
```

---

## สรุป

Crashlytics และ Analytics ประกอบด้วย:

1. **Crashlytics** - จับ crash และ error อัตโนมัติ
2. **Custom Logging** - บันทึก breadcrumbs และ custom keys
3. **Firebase Analytics** - track user behavior
4. **Custom Events** - วัด business metrics สำคัญ
5. **A/B Testing** - ทดสอบ variants ด้วย Remote Config
6. **Error Boundary** - จัดการ UI errors อย่างสวยงาม
