# Part 79: Feature Flags - Feature Flags และ A/B Testing

## บทนำ

Feature Flags (หรือ Feature Toggles) คือกลไกที่ให้เราเปิด/ปิดฟีเจอร์ในแอปได้แบบ real-time โดยไม่ต้อง deploy ใหม่ เหมาะสำหรับ A/B testing, gradual rollout และ emergency kill switch

## 1. Firebase Remote Config

### การติดตั้ง

```yaml
# pubspec.yaml
dependencies:
  firebase_core: ^2.27.0
  firebase_remote_config: ^4.3.8
  firebase_analytics: ^10.8.9  # สำหรับ A/B testing
```

### Firebase Setup

```dart
// lib/main.dart
import 'package:firebase_core/firebase_core.dart';
import 'firebase_options.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );
  runApp(const MyApp());
}
```

### Remote Config Service

```dart
// lib/feature_flags/remote_config_service.dart
import 'dart:convert';
import 'package:firebase_remote_config/firebase_remote_config.dart';
import 'package:flutter/foundation.dart';

class RemoteConfigService {
  final FirebaseRemoteConfig _remoteConfig = FirebaseRemoteConfig.instance;
  bool _initialized = false;

  // Default values - ค่าที่ใช้เมื่อ fetch ไม่ได้
  static const Map<String, dynamic> _defaults = {
    'new_checkout_flow': false,
    'dark_mode_enabled': true,
    'max_cart_items': 10,
    'discount_percentage': 0,
    'show_referral_banner': false,
    'home_layout': 'grid',
    'experiment_variant': 'control',
    'api_version': 'v1',
    'maintenance_message': '',
    'force_update_version': '',
  };

  Future<void> initialize() async {
    if (_initialized) return;

    // ตั้งค่า default values
    await _remoteConfig.setDefaults(_defaults);

    // ตั้งค่า fetch interval
    await _remoteConfig.setConfigSettings(
      RemoteConfigSettings(
        fetchTimeout: const Duration(seconds: 10),
        minimumFetchInterval: kDebugMode
            ? Duration.zero // ใน debug mode fetch ทุกครั้ง
            : const Duration(hours: 1),
      ),
    );

    // Fetch และ activate
    await fetchAndActivate();

    _initialized = true;
  }

  Future<bool> fetchAndActivate() async {
    try {
      return await _remoteConfig.fetchAndActivate();
    } catch (e) {
      debugPrint('Remote config fetch error: $e');
      return false;
    }
  }

  // Getters สำหรับ flags ต่างๆ
  bool get newCheckoutFlow => _remoteConfig.getBool('new_checkout_flow');

  bool get darkModeEnabled => _remoteConfig.getBool('dark_mode_enabled');

  int get maxCartItems => _remoteConfig.getInt('max_cart_items');

  double get discountPercentage =>
      _remoteConfig.getDouble('discount_percentage');

  bool get showReferralBanner => _remoteConfig.getBool('show_referral_banner');

  String get homeLayout => _remoteConfig.getString('home_layout');

  String get experimentVariant =>
      _remoteConfig.getString('experiment_variant');

  String get apiVersion => _remoteConfig.getString('api_version');

  String get maintenanceMessage =>
      _remoteConfig.getString('maintenance_message');

  bool get isInMaintenance => maintenanceMessage.isNotEmpty;

  String get forceUpdateVersion =>
      _remoteConfig.getString('force_update_version');

  // Generic getter
  T getValue<T>(String key, T defaultValue) {
    try {
      if (T == bool) return _remoteConfig.getBool(key) as T;
      if (T == int) return _remoteConfig.getInt(key) as T;
      if (T == double) return _remoteConfig.getDouble(key) as T;
      if (T == String) return _remoteConfig.getString(key) as T;
      return defaultValue;
    } catch (_) {
      return defaultValue;
    }
  }

  // JSON config
  Map<String, dynamic>? getJsonConfig(String key) {
    try {
      final jsonString = _remoteConfig.getString(key);
      if (jsonString.isEmpty) return null;
      return json.decode(jsonString) as Map<String, dynamic>;
    } catch (_) {
      return null;
    }
  }

  // ฟัง updates แบบ real-time
  Stream<RemoteConfigUpdate> get updatesStream =>
      _remoteConfig.onConfigUpdated;
}
```

## 2. Feature Toggle Patterns

### Feature Flag Provider

```dart
// lib/feature_flags/feature_flag_provider.dart
import 'package:flutter/widgets.dart';
import 'remote_config_service.dart';

class FeatureFlagProvider extends InheritedNotifier<FeatureFlagNotifier> {
  const FeatureFlagProvider({
    super.key,
    required FeatureFlagNotifier super.notifier,
    required super.child,
  });

  static FeatureFlagNotifier of(BuildContext context) {
    final provider = context
        .dependOnInheritedWidgetOfExactType<FeatureFlagProvider>();
    assert(provider != null, 'FeatureFlagProvider not found in widget tree');
    return provider!.notifier!;
  }
}

class FeatureFlagNotifier extends ChangeNotifier {
  final RemoteConfigService _remoteConfig;

  FeatureFlagNotifier({required RemoteConfigService remoteConfig})
      : _remoteConfig = remoteConfig {
    // ฟัง real-time updates
    remoteConfig.updatesStream.listen((_) {
      notifyListeners();
    });
  }

  bool isEnabled(String flagName) {
    return _remoteConfig.getValue(flagName, false);
  }

  T getValue<T>(String key, T defaultValue) {
    return _remoteConfig.getValue(key, defaultValue);
  }

  // Named flags
  bool get newCheckoutFlow => _remoteConfig.newCheckoutFlow;
  bool get darkModeEnabled => _remoteConfig.darkModeEnabled;
  int get maxCartItems => _remoteConfig.maxCartItems;
  double get discountPercentage => _remoteConfig.discountPercentage;
  String get homeLayout => _remoteConfig.homeLayout;
  String get experimentVariant => _remoteConfig.experimentVariant;
  bool get isInMaintenance => _remoteConfig.isInMaintenance;
  String get maintenanceMessage => _remoteConfig.maintenanceMessage;
}
```

### Feature Flag Widget

```dart
// lib/feature_flags/feature_flag_widget.dart
import 'package:flutter/widgets.dart';
import 'feature_flag_provider.dart';

// Widget ที่แสดง/ซ่อนตาม feature flag
class FeatureFlag extends StatelessWidget {
  final String flagName;
  final Widget child;
  final Widget? fallback;

  const FeatureFlag({
    super.key,
    required this.flagName,
    required this.child,
    this.fallback,
  });

  @override
  Widget build(BuildContext context) {
    final flags = FeatureFlagProvider.of(context);
    final isEnabled = flags.isEnabled(flagName);

    if (isEnabled) return child;
    return fallback ?? const SizedBox.shrink();
  }
}

// Widget สำหรับ A/B test
class ABTest extends StatelessWidget {
  final String flagName;
  final String variantA; // control
  final String variantB; // treatment
  final Widget widgetA;
  final Widget widgetB;
  final Widget? fallback;

  const ABTest({
    super.key,
    required this.flagName,
    this.variantA = 'control',
    this.variantB = 'treatment',
    required this.widgetA,
    required this.widgetB,
    this.fallback,
  });

  @override
  Widget build(BuildContext context) {
    final flags = FeatureFlagProvider.of(context);
    final variant = flags.getValue(flagName, variantA);

    if (variant == variantA) return widgetA;
    if (variant == variantB) return widgetB;
    return fallback ?? widgetA;
  }
}
```

## 3. A/B Testing

### Analytics Integration

```dart
// lib/analytics/analytics_service.dart
import 'package:firebase_analytics/firebase_analytics.dart';

class AnalyticsService {
  final FirebaseAnalytics _analytics = FirebaseAnalytics.instance;

  // Log A/B test exposure
  Future<void> logExperimentExposure({
    required String experimentName,
    required String variant,
  }) async {
    await _analytics.logEvent(
      name: 'experiment_exposure',
      parameters: {
        'experiment_name': experimentName,
        'variant': variant,
      },
    );
  }

  // Log conversion event
  Future<void> logConversion({
    required String experimentName,
    required String eventName,
    Map<String, dynamic>? parameters,
  }) async {
    await _analytics.logEvent(
      name: eventName,
      parameters: {
        'experiment_name': experimentName,
        ...?parameters,
      },
    );
  }

  // User properties สำหรับ segmentation
  Future<void> setUserProperty({
    required String name,
    required String value,
  }) async {
    await _analytics.setUserProperty(name: name, value: value);
  }
}
```

### A/B Test Manager

```dart
// lib/feature_flags/ab_test_manager.dart
import 'analytics_service.dart';
import 'remote_config_service.dart';

class ABTestManager {
  final RemoteConfigService _remoteConfig;
  final AnalyticsService _analytics;
  final Set<String> _loggedExposures = {};

  ABTestManager({
    required RemoteConfigService remoteConfig,
    required AnalyticsService analytics,
  })  : _remoteConfig = remoteConfig,
        _analytics = analytics;

  // ดูว่า user อยู่ใน variant ไหน
  String getVariant(String experimentName, {String defaultVariant = 'control'}) {
    final variant = _remoteConfig.getValue(experimentName, defaultVariant);

    // Log exposure ครั้งแรกที่ user เจอ experiment นี้
    if (!_loggedExposures.contains(experimentName)) {
      _loggedExposures.add(experimentName);
      _analytics.logExperimentExposure(
        experimentName: experimentName,
        variant: variant,
      );
    }

    return variant;
  }

  bool isVariant(String experimentName, String variantName) {
    return getVariant(experimentName) == variantName;
  }

  // Log conversion เมื่อ user ทำ action ที่ต้องการ
  void logConversion({
    required String experimentName,
    required String eventName,
    Map<String, dynamic>? parameters,
  }) {
    _analytics.logConversion(
      experimentName: experimentName,
      eventName: eventName,
      parameters: {
        'variant': getVariant(experimentName),
        ...?parameters,
      },
    );
  }
}
```

## 4. Gradual Rollouts

### Rollout Strategy

```dart
// lib/feature_flags/rollout_strategy.dart
import 'dart:math';

class RolloutStrategy {
  // Percentage-based rollout
  static bool isInRollout({
    required String userId,
    required String flagName,
    required double percentage, // 0.0 - 100.0
  }) {
    if (percentage <= 0) return false;
    if (percentage >= 100) return true;

    // สร้าง hash จาก userId + flagName เพื่อให้ consistent
    final hash = _hashString('$userId:$flagName');
    final bucket = (hash % 100).abs().toDouble();
    return bucket < percentage;
  }

  static int _hashString(String input) {
    int hash = 5381;
    for (final codeUnit in input.codeUnits) {
      hash = ((hash << 5) + hash) + codeUnit;
    }
    return hash;
  }
}
```

## 5. Workshop: Feature Flag System

### Complete Feature Flag System

```dart
// lib/feature_flags/feature_flag_service.dart
import 'dart:async';
import 'package:flutter/foundation.dart';
import 'remote_config_service.dart';
import 'rollout_strategy.dart';

class FeatureFlagService extends ChangeNotifier {
  final RemoteConfigService _remoteConfig;
  String? _userId;

  // Local overrides สำหรับ testing
  final Map<String, dynamic> _overrides = {};

  FeatureFlagService({required RemoteConfigService remoteConfig})
      : _remoteConfig = remoteConfig;

  void setUserId(String userId) {
    _userId = userId;
  }

  // Override flag สำหรับ testing (debug mode เท่านั้น)
  void override(String flagName, dynamic value) {
    assert(kDebugMode, 'Overrides only available in debug mode');
    _overrides[flagName] = value;
    notifyListeners();
  }

  void clearOverride(String flagName) {
    _overrides.remove(flagName);
    notifyListeners();
  }

  void clearAllOverrides() {
    _overrides.clear();
    notifyListeners();
  }

  bool getBool(String flagName, {bool defaultValue = false}) {
    // Check local override ก่อน
    if (kDebugMode && _overrides.containsKey(flagName)) {
      return _overrides[flagName] as bool? ?? defaultValue;
    }

    return _remoteConfig.getValue(flagName, defaultValue);
  }

  String getString(String flagName, {String defaultValue = ''}) {
    if (kDebugMode && _overrides.containsKey(flagName)) {
      return _overrides[flagName] as String? ?? defaultValue;
    }

    return _remoteConfig.getValue(flagName, defaultValue);
  }

  int getInt(String flagName, {int defaultValue = 0}) {
    if (kDebugMode && _overrides.containsKey(flagName)) {
      return _overrides[flagName] as int? ?? defaultValue;
    }

    return _remoteConfig.getValue(flagName, defaultValue);
  }

  // Rollout-aware flag
  bool isEnabledForUser(
    String flagName, {
    double rolloutPercentage = 100.0,
    bool defaultValue = false,
  }) {
    final baseEnabled = getBool(flagName, defaultValue: defaultValue);
    if (!baseEnabled) return false;

    if (_userId == null) return baseEnabled;

    return RolloutStrategy.isInRollout(
      userId: _userId!,
      flagName: flagName,
      percentage: rolloutPercentage,
    );
  }

  Future<void> refresh() async {
    await _remoteConfig.fetchAndActivate();
    notifyListeners();
  }
}
```

### Debug Panel Widget

```dart
// lib/feature_flags/debug_panel.dart
import 'package:flutter/material.dart';
import 'feature_flag_service.dart';

class FeatureFlagDebugPanel extends StatelessWidget {
  final FeatureFlagService flagService;
  final List<DebugFlagItem> flags;

  const FeatureFlagDebugPanel({
    super.key,
    required this.flagService,
    required this.flags,
  });

  @override
  Widget build(BuildContext context) {
    if (!const bool.fromEnvironment('dart.vm.product')) {
      return const SizedBox.shrink(); // Hide in release mode
    }

    return ListView(
      children: [
        const ListTile(
          title: Text(
            'Feature Flags Debug',
            style: TextStyle(fontWeight: FontWeight.bold),
          ),
        ),
        const Divider(),
        ...flags.map((flag) => _buildFlagItem(context, flag)),
        const SizedBox(height: 16),
        Padding(
          padding: const EdgeInsets.symmetric(horizontal: 16),
          child: OutlinedButton(
            onPressed: () => flagService.clearAllOverrides(),
            child: const Text('Reset All Overrides'),
          ),
        ),
      ],
    );
  }

  Widget _buildFlagItem(BuildContext context, DebugFlagItem flag) {
    return ListenableBuilder(
      listenable: flagService,
      builder: (context, _) {
        switch (flag.type) {
          case FlagType.bool:
            final currentValue = flagService.getBool(flag.name);
            return SwitchListTile(
              title: Text(flag.displayName),
              subtitle: Text(flag.name),
              value: currentValue,
              onChanged: (value) => flagService.override(flag.name, value),
            );

          case FlagType.string:
            final currentValue = flagService.getString(flag.name);
            return ListTile(
              title: Text(flag.displayName),
              subtitle: Text('$currentValue (${flag.name})'),
              trailing: flag.options != null
                  ? DropdownButton<String>(
                      value: currentValue,
                      items: flag.options!
                          .map((o) => DropdownMenuItem(
                                value: o,
                                child: Text(o),
                              ))
                          .toList(),
                      onChanged: (value) {
                        if (value != null) {
                          flagService.override(flag.name, value);
                        }
                      },
                    )
                  : null,
            );

          default:
            return const SizedBox.shrink();
        }
      },
    );
  }
}

enum FlagType { bool, string, int }

class DebugFlagItem {
  final String name;
  final String displayName;
  final FlagType type;
  final List<String>? options;

  const DebugFlagItem({
    required this.name,
    required this.displayName,
    required this.type,
    this.options,
  });
}
```

### Usage Example

```dart
// lib/screens/home_screen.dart
import 'package:flutter/material.dart';
import '../feature_flags/feature_flag_provider.dart';
import '../feature_flags/feature_flag_widget.dart';

class HomeScreen extends StatelessWidget {
  const HomeScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final flags = FeatureFlagProvider.of(context);

    return Scaffold(
      appBar: AppBar(
        title: const Text('หน้าแรก'),
      ),
      body: Column(
        children: [
          // แสดง maintenance banner ถ้า flag เปิด
          if (flags.isInMaintenance)
            Container(
              color: Colors.orange,
              padding: const EdgeInsets.all(8),
              child: Text(
                flags.maintenanceMessage,
                style: const TextStyle(color: Colors.white),
              ),
            ),

          // A/B test: Grid vs List layout
          Expanded(
            child: ABTest(
              flagName: 'home_layout_experiment',
              variantA: 'grid',
              variantB: 'list',
              widgetA: const ProductGridView(),
              widgetB: const ProductListView(),
            ),
          ),

          // Feature flag: แสดง referral banner เฉพาะบาง user
          FeatureFlag(
            flagName: 'show_referral_banner',
            child: const ReferralBanner(),
          ),
        ],
      ),
      floatingActionButton: FeatureFlag(
        flagName: 'new_checkout_flow',
        child: FloatingActionButton.extended(
          onPressed: () => Navigator.pushNamed(context, '/checkout-v2'),
          label: const Text('Checkout ใหม่'),
          icon: const Icon(Icons.shopping_cart_checkout),
        ),
        fallback: FloatingActionButton(
          onPressed: () => Navigator.pushNamed(context, '/cart'),
          child: const Icon(Icons.shopping_cart),
        ),
      ),
    );
  }
}

class ProductGridView extends StatelessWidget {
  const ProductGridView({super.key});

  @override
  Widget build(BuildContext context) => const Center(
        child: Text('Grid Layout'),
      );
}

class ProductListView extends StatelessWidget {
  const ProductListView({super.key});

  @override
  Widget build(BuildContext context) => const Center(
        child: Text('List Layout'),
      );
}

class ReferralBanner extends StatelessWidget {
  const ReferralBanner({super.key});

  @override
  Widget build(BuildContext context) => Container(
        color: Colors.blue[50],
        padding: const EdgeInsets.all(16),
        child: const Text('แนะนำเพื่อนรับส่วนลด 20%!'),
      );
}
```

## สรุป

Feature Flags:
1. **Firebase Remote Config** เป็น solution ที่ง่ายและ scalable
2. **A/B Testing** ช่วยทดสอบ hypothesis ก่อนตัดสินใจ
3. **Gradual Rollout** ลดความเสี่ยงในการ release ฟีเจอร์ใหม่
4. **Debug Panel** ช่วย QA ทดสอบ flags ได้ง่าย

## แบบทดสอบ

1. อธิบายความแตกต่างระหว่าง Feature Flag และ A/B Test
2. ทำไมควรใช้ hash ของ userId แทน random number ใน rollout?
3. สร้าง Kill Switch ที่ force close ฟีเจอร์ทั้งหมดได้ทันที
4. อธิบาย risks ของ feature flags ถ้าไม่มีการ cleanup
