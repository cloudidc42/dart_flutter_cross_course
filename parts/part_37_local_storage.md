# Part 37: Local Storage

## การเก็บข้อมูลในเครื่อง

Flutter มีหลาย options สำหรับเก็บข้อมูล local ขึ้นกับความต้องการ

```
SharedPreferences  → Key-value pairs ง่ายๆ (settings, tokens)
FlutterSecureStorage → ข้อมูลที่ต้องการความปลอดภัย (passwords, API keys)
Hive               → NoSQL database เร็ว ง่าย
ObjectBox          → Database ที่มี performance สูงมาก
Sqflite            → SQLite สำหรับ relational data
```

---

## SharedPreferences

### Setup

```yaml
dependencies:
  shared_preferences: ^2.2.2
```

### การใช้งานพื้นฐาน

```dart
import 'package:shared_preferences/shared_preferences.dart';

class PreferencesService {
  static const String _themeKey = 'theme_mode';
  static const String _languageKey = 'language';
  static const String _authTokenKey = 'auth_token';
  static const String _firstLaunchKey = 'first_launch';
  static const String _usernameKey = 'username';
  
  SharedPreferences? _prefs;
  
  Future<void> init() async {
    _prefs = await SharedPreferences.getInstance();
  }
  
  SharedPreferences get prefs {
    if (_prefs == null) throw Exception('PreferencesService not initialized');
    return _prefs!;
  }
  
  // Save methods
  Future<void> setThemeMode(String mode) async {
    await prefs.setString(_themeKey, mode);
  }
  
  Future<void> setLanguage(String lang) async {
    await prefs.setString(_languageKey, lang);
  }
  
  Future<void> setAuthToken(String token) async {
    await prefs.setString(_authTokenKey, token);
  }
  
  Future<void> setUsername(String name) async {
    await prefs.setString(_usernameKey, name);
  }
  
  Future<void> setFirstLaunch(bool value) async {
    await prefs.setBool(_firstLaunchKey, value);
  }
  
  // Read methods (with defaults)
  String getThemeMode() => prefs.getString(_themeKey) ?? 'system';
  String getLanguage() => prefs.getString(_languageKey) ?? 'th';
  String? getAuthToken() => prefs.getString(_authTokenKey);
  String getUsername() => prefs.getString(_usernameKey) ?? 'Guest';
  bool isFirstLaunch() => prefs.getBool(_firstLaunchKey) ?? true;
  
  // Delete methods
  Future<void> clearAuthToken() async {
    await prefs.remove(_authTokenKey);
  }
  
  Future<void> clearAll() async {
    await prefs.clear();
  }
}
```

### SharedPreferences กับ Complex Types

```dart
import 'dart:convert';

class UserPreferences {
  static const String _favoritesKey = 'favorites';
  static const String _recentSearchesKey = 'recent_searches';
  static const String _userProfileKey = 'user_profile';
  
  final SharedPreferences _prefs;
  
  UserPreferences(this._prefs);
  
  // Store List<String>
  Future<void> saveFavorites(List<String> favorites) async {
    await _prefs.setStringList(_favoritesKey, favorites);
  }
  
  List<String> getFavorites() {
    return _prefs.getStringList(_favoritesKey) ?? [];
  }
  
  Future<void> addFavorite(String item) async {
    final favorites = getFavorites();
    if (!favorites.contains(item)) {
      favorites.add(item);
      await saveFavorites(favorites);
    }
  }
  
  Future<void> removeFavorite(String item) async {
    final favorites = getFavorites();
    favorites.remove(item);
    await saveFavorites(favorites);
  }
  
  bool isFavorite(String item) => getFavorites().contains(item);
  
  // Store complex object as JSON
  Future<void> saveUserProfile(Map<String, dynamic> profile) async {
    await _prefs.setString(_userProfileKey, jsonEncode(profile));
  }
  
  Map<String, dynamic>? getUserProfile() {
    final jsonStr = _prefs.getString(_userProfileKey);
    if (jsonStr == null) return null;
    return jsonDecode(jsonStr) as Map<String, dynamic>;
  }
  
  // Store recent searches (circular buffer)
  Future<void> addRecentSearch(String query) async {
    final searches = _prefs.getStringList(_recentSearchesKey) ?? [];
    searches.remove(query);
    searches.insert(0, query);
    if (searches.length > 10) {
      searches.removeLast();
    }
    await _prefs.setStringList(_recentSearchesKey, searches);
  }
  
  List<String> getRecentSearches() {
    return _prefs.getStringList(_recentSearchesKey) ?? [];
  }
}
```

---

## Secure Storage

### Setup

```yaml
dependencies:
  flutter_secure_storage: ^9.0.0
```

### การใช้งาน Secure Storage

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class SecureStorageService {
  static const _storage = FlutterSecureStorage(
    aOptions: AndroidOptions(
      encryptedSharedPreferences: true,  // Android: ใช้ EncryptedSharedPreferences
    ),
    iOptions: IOSOptions(
      accessibility: KeychainAccessibility.first_unlock_this_device,  // iOS: Keychain options
    ),
  );
  
  // Keys
  static const String _accessTokenKey = 'access_token';
  static const String _refreshTokenKey = 'refresh_token';
  static const String _pinCodeKey = 'pin_code';
  static const String _biometricEnabledKey = 'biometric_enabled';
  
  // Auth tokens
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
  
  // PIN code
  static Future<void> savePinCode(String pin) async {
    await _storage.write(key: _pinCodeKey, value: pin);
  }
  
  static Future<bool> verifyPinCode(String pin) async {
    final savedPin = await _storage.read(key: _pinCodeKey);
    return savedPin == pin;
  }
  
  static Future<void> clearPinCode() async {
    await _storage.delete(key: _pinCodeKey);
  }
  
  // Check if all secure data exists
  static Future<Map<String, String?>> getAllSecureData() async {
    return await _storage.readAll();
  }
  
  static Future<void> clearAll() async {
    await _storage.deleteAll();
  }
}
```

---

## Hive Package

### Setup

```yaml
dependencies:
  hive: ^2.2.3
  hive_flutter: ^1.1.0

dev_dependencies:
  hive_generator: ^2.0.1
  build_runner: ^2.4.0
```

### Hive พื้นฐาน

```dart
import 'package:hive_flutter/hive_flutter.dart';

// Initialize Hive
Future<void> initHive() async {
  await Hive.initFlutter();
  
  // Register adapters สำหรับ custom objects
  Hive.registerAdapter(ProductAdapter());
  Hive.registerAdapter(CartItemAdapter());
  
  // เปิด boxes
  await Hive.openBox('settings');
  await Hive.openBox<Product>('products');
  await Hive.openBox<CartItem>('cart');
}

// Simple key-value box
class SettingsHiveService {
  final Box _box = Hive.box('settings');
  
  // Save
  Future<void> setDarkMode(bool value) async {
    await _box.put('dark_mode', value);
  }
  
  Future<void> setNotificationsEnabled(bool value) async {
    await _box.put('notifications', value);
  }
  
  Future<void> setFontSize(double size) async {
    await _box.put('font_size', size);
  }
  
  // Read
  bool isDarkMode() => _box.get('dark_mode', defaultValue: false);
  bool isNotificationsEnabled() => _box.get('notifications', defaultValue: true);
  double getFontSize() => _box.get('font_size', defaultValue: 14.0);
  
  // Watch changes (reactive)
  ValueListenable<Box> get settingsListenable => _box.listenable();
}
```

### Hive กับ Custom Objects

```dart
// model/product.dart
import 'package:hive/hive.dart';

part 'product.g.dart';  // generated by build_runner

@HiveType(typeId: 0)
class Product extends HiveObject {  // extend HiveObject สำหรับ auto-save
  @HiveField(0)
  late String id;
  
  @HiveField(1)
  late String name;
  
  @HiveField(2)
  late double price;
  
  @HiveField(3)
  late String category;
  
  @HiveField(4)
  late bool inStock;
  
  Product({
    required this.id,
    required this.name,
    required this.price,
    required this.category,
    this.inStock = true,
  });
}

// Run: flutter pub run build_runner build

// service/product_hive_service.dart
class ProductHiveService {
  late Box<Product> _box;
  
  Future<void> init() async {
    _box = await Hive.openBox<Product>('products');
  }
  
  // CRUD
  Future<void> addProduct(Product product) async {
    await _box.put(product.id, product);
  }
  
  Future<void> updateProduct(Product product) async {
    await product.save();  // HiveObject auto-save
  }
  
  Future<void> deleteProduct(String id) async {
    await _box.delete(id);
  }
  
  Product? getProduct(String id) => _box.get(id);
  
  List<Product> getAllProducts() => _box.values.toList();
  
  List<Product> getByCategory(String category) {
    return _box.values.where((p) => p.category == category).toList();
  }
  
  // Reactive stream
  Stream<BoxEvent> watchProducts() => _box.watch();
  
  // Reactive ValueListenable
  ValueListenable<Box<Product>> get productsListenable => _box.listenable();
}
```

### Hive Box ที่ Encrypted

```dart
Future<void> openEncryptedBox() async {
  const secureStorage = FlutterSecureStorage();
  
  // สร้างหรือดึง encryption key
  final encryptionKeyString = await secureStorage.read(key: 'hive_encryption_key');
  List<int> encryptionKey;
  
  if (encryptionKeyString == null) {
    // สร้าง key ใหม่
    final key = Hive.generateSecureKey();
    await secureStorage.write(
      key: 'hive_encryption_key',
      value: base64UrlEncode(key),
    );
    encryptionKey = key;
  } else {
    encryptionKey = base64Url.decode(encryptionKeyString);
  }
  
  // เปิด encrypted box
  await Hive.openBox<String>(
    'secure_data',
    encryptionCipher: HiveAesCipher(encryptionKey),
  );
}
```

---

## ObjectBox Basics

### Setup

```yaml
dependencies:
  objectbox: ^2.3.1
  objectbox_flutter_libs: any

dev_dependencies:
  objectbox_generator: any
  build_runner: ^2.4.0
```

### ObjectBox Model

```dart
import 'package:objectbox/objectbox.dart';

@Entity()
class Task {
  @Id()
  int id;
  
  String title;
  String? description;
  bool isCompleted;
  
  @Property(type: PropertyType.date)
  DateTime createdAt;
  
  @Property(type: PropertyType.date)
  DateTime? dueDate;
  
  // Relations
  final category = ToOne<Category>();
  
  Task({
    this.id = 0,
    required this.title,
    this.description,
    this.isCompleted = false,
    DateTime? createdAt,
    this.dueDate,
  }) : createdAt = createdAt ?? DateTime.now();
}

@Entity()
class Category {
  @Id()
  int id;
  
  String name;
  String color;
  
  // One-to-many relation
  @Backlink('category')
  final tasks = ToMany<Task>();
  
  Category({
    this.id = 0,
    required this.name,
    required this.color,
  });
}
```

### ObjectBox Service

```dart
import 'objectbox.g.dart';  // generated

class ObjectBoxService {
  late final Store _store;
  late final Box<Task> _taskBox;
  late final Box<Category> _categoryBox;
  
  Future<void> init() async {
    _store = await openStore();
    _taskBox = _store.box<Task>();
    _categoryBox = _store.box<Category>();
  }
  
  void dispose() => _store.close();
  
  // Task operations
  int addTask(Task task) => _taskBox.put(task);
  
  Task? getTask(int id) => _taskBox.get(id);
  
  List<Task> getAllTasks() => _taskBox.getAll();
  
  List<Task> getActiveTasks() {
    final query = _taskBox
        .query(Task_.isCompleted.equals(false))
        .order(Task_.createdAt, flags: Order.descending)
        .build();
    return query.find();
  }
  
  List<Task> searchTasks(String keyword) {
    final query = _taskBox
        .query(Task_.title.contains(keyword, caseSensitive: false))
        .build();
    return query.find();
  }
  
  void updateTask(Task task) => _taskBox.put(task);
  
  void deleteTask(int id) => _taskBox.remove(id);
  
  // Reactive query
  Stream<List<Task>> watchAllTasks() {
    final query = _taskBox.query().build();
    return query.watch(triggerImmediately: true).map((q) => q.find());
  }
  
  // Category operations
  int addCategory(Category category) => _categoryBox.put(category);
  
  List<Category> getAllCategories() => _categoryBox.getAll();
}
```

---

## Workshop: Settings Screen with Persistence

### App Settings Model

```dart
// models/app_settings.dart
class AppSettings {
  final String themeMode;  // 'light', 'dark', 'system'
  final String language;
  final bool notificationsEnabled;
  final bool soundEnabled;
  final double fontSize;
  final bool biometricEnabled;
  final String userName;
  final String userEmail;
  
  const AppSettings({
    this.themeMode = 'system',
    this.language = 'th',
    this.notificationsEnabled = true,
    this.soundEnabled = true,
    this.fontSize = 14.0,
    this.biometricEnabled = false,
    this.userName = '',
    this.userEmail = '',
  });
  
  AppSettings copyWith({
    String? themeMode,
    String? language,
    bool? notificationsEnabled,
    bool? soundEnabled,
    double? fontSize,
    bool? biometricEnabled,
    String? userName,
    String? userEmail,
  }) {
    return AppSettings(
      themeMode: themeMode ?? this.themeMode,
      language: language ?? this.language,
      notificationsEnabled: notificationsEnabled ?? this.notificationsEnabled,
      soundEnabled: soundEnabled ?? this.soundEnabled,
      fontSize: fontSize ?? this.fontSize,
      biometricEnabled: biometricEnabled ?? this.biometricEnabled,
      userName: userName ?? this.userName,
      userEmail: userEmail ?? this.userEmail,
    );
  }
}
```

### Settings Service

```dart
// services/settings_service.dart
import 'package:shared_preferences/shared_preferences.dart';

class SettingsService extends ChangeNotifier {
  SharedPreferences? _prefs;
  AppSettings _settings = const AppSettings();
  
  AppSettings get settings => _settings;
  
  static const _keys = {
    'theme': 'theme_mode',
    'language': 'language',
    'notifications': 'notifications_enabled',
    'sound': 'sound_enabled',
    'fontSize': 'font_size',
    'biometric': 'biometric_enabled',
    'userName': 'user_name',
    'userEmail': 'user_email',
  };
  
  Future<void> init() async {
    _prefs = await SharedPreferences.getInstance();
    _loadSettings();
  }
  
  void _loadSettings() {
    if (_prefs == null) return;
    
    _settings = AppSettings(
      themeMode: _prefs!.getString(_keys['theme']!) ?? 'system',
      language: _prefs!.getString(_keys['language']!) ?? 'th',
      notificationsEnabled: _prefs!.getBool(_keys['notifications']!) ?? true,
      soundEnabled: _prefs!.getBool(_keys['sound']!) ?? true,
      fontSize: _prefs!.getDouble(_keys['fontSize']!) ?? 14.0,
      biometricEnabled: _prefs!.getBool(_keys['biometric']!) ?? false,
      userName: _prefs!.getString(_keys['userName']!) ?? '',
      userEmail: _prefs!.getString(_keys['userEmail']!) ?? '',
    );
    
    notifyListeners();
  }
  
  Future<void> updateTheme(String mode) async {
    await _prefs?.setString(_keys['theme']!, mode);
    _settings = _settings.copyWith(themeMode: mode);
    notifyListeners();
  }
  
  Future<void> updateLanguage(String lang) async {
    await _prefs?.setString(_keys['language']!, lang);
    _settings = _settings.copyWith(language: lang);
    notifyListeners();
  }
  
  Future<void> updateNotifications(bool enabled) async {
    await _prefs?.setBool(_keys['notifications']!, enabled);
    _settings = _settings.copyWith(notificationsEnabled: enabled);
    notifyListeners();
  }
  
  Future<void> updateSound(bool enabled) async {
    await _prefs?.setBool(_keys['sound']!, enabled);
    _settings = _settings.copyWith(soundEnabled: enabled);
    notifyListeners();
  }
  
  Future<void> updateFontSize(double size) async {
    await _prefs?.setDouble(_keys['fontSize']!, size);
    _settings = _settings.copyWith(fontSize: size);
    notifyListeners();
  }
  
  Future<void> updateBiometric(bool enabled) async {
    await _prefs?.setBool(_keys['biometric']!, enabled);
    _settings = _settings.copyWith(biometricEnabled: enabled);
    notifyListeners();
  }
  
  Future<void> updateProfile({String? name, String? email}) async {
    if (name != null) await _prefs?.setString(_keys['userName']!, name);
    if (email != null) await _prefs?.setString(_keys['userEmail']!, email);
    _settings = _settings.copyWith(userName: name, userEmail: email);
    notifyListeners();
  }
  
  Future<void> resetToDefaults() async {
    await _prefs?.clear();
    _settings = const AppSettings();
    notifyListeners();
  }
}
```

### Main App

```dart
// main.dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  final settingsService = SettingsService();
  await settingsService.init();
  
  runApp(
    ChangeNotifierProvider.value(
      value: settingsService,
      child: const SettingsApp(),
    ),
  );
}

class SettingsApp extends StatelessWidget {
  const SettingsApp({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    final settings = context.watch<SettingsService>().settings;
    
    return MaterialApp(
      title: 'Settings Demo',
      theme: _buildTheme(Brightness.light, settings.fontSize),
      darkTheme: _buildTheme(Brightness.dark, settings.fontSize),
      themeMode: _getThemeMode(settings.themeMode),
      home: const SettingsPage(),
    );
  }
  
  ThemeData _buildTheme(Brightness brightness, double fontSize) {
    return ThemeData(
      brightness: brightness,
      useMaterial3: true,
      colorScheme: ColorScheme.fromSeed(
        seedColor: Colors.teal,
        brightness: brightness,
      ),
      textTheme: TextTheme(
        bodyMedium: TextStyle(fontSize: fontSize),
        bodySmall: TextStyle(fontSize: fontSize - 2),
        titleMedium: TextStyle(fontSize: fontSize + 2),
      ),
    );
  }
  
  ThemeMode _getThemeMode(String mode) {
    switch (mode) {
      case 'light': return ThemeMode.light;
      case 'dark': return ThemeMode.dark;
      default: return ThemeMode.system;
    }
  }
}
```

### Settings Page

```dart
// screens/settings_page.dart
class SettingsPage extends StatelessWidget {
  const SettingsPage({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Settings'),
        actions: [
          IconButton(
            onPressed: () => _showResetDialog(context),
            icon: const Icon(Icons.restore),
            tooltip: 'Reset to defaults',
          ),
        ],
      ),
      body: const SingleChildScrollView(
        child: Column(
          children: [
            _ProfileSection(),
            _AppearanceSection(),
            _NotificationSection(),
            _SecuritySection(),
            _AboutSection(),
          ],
        ),
      ),
    );
  }
  
  void _showResetDialog(BuildContext context) {
    showDialog(
      context: context,
      builder: (ctx) => AlertDialog(
        title: const Text('Reset Settings'),
        content: const Text('Reset all settings to defaults?'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(ctx),
            child: const Text('Cancel'),
          ),
          ElevatedButton(
            onPressed: () {
              context.read<SettingsService>().resetToDefaults();
              Navigator.pop(ctx);
            },
            style: ElevatedButton.styleFrom(backgroundColor: Colors.red),
            child: const Text('Reset', style: TextStyle(color: Colors.white)),
          ),
        ],
      ),
    );
  }
}

class _SectionHeader extends StatelessWidget {
  final String title;
  const _SectionHeader({required this.title});
  
  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.fromLTRB(16, 24, 16, 8),
      child: Text(
        title,
        style: TextStyle(
          color: Theme.of(context).colorScheme.primary,
          fontWeight: FontWeight.bold,
          fontSize: 13,
        ),
      ),
    );
  }
}

class _ProfileSection extends StatelessWidget {
  const _ProfileSection();
  
  @override
  Widget build(BuildContext context) {
    final settings = context.watch<SettingsService>().settings;
    
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        const _SectionHeader(title: 'PROFILE'),
        Card(
          margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
          child: Column(
            children: [
              ListTile(
                leading: CircleAvatar(
                  backgroundColor: Theme.of(context).colorScheme.primary,
                  child: Text(
                    settings.userName.isEmpty ? '?' : settings.userName[0].toUpperCase(),
                    style: const TextStyle(color: Colors.white),
                  ),
                ),
                title: Text(settings.userName.isEmpty ? 'Set your name' : settings.userName),
                subtitle: Text(settings.userEmail.isEmpty ? 'Set your email' : settings.userEmail),
                trailing: const Icon(Icons.arrow_forward_ios, size: 16),
                onTap: () => _showEditProfileDialog(context, settings),
              ),
            ],
          ),
        ),
      ],
    );
  }
  
  void _showEditProfileDialog(BuildContext context, AppSettings settings) {
    final nameController = TextEditingController(text: settings.userName);
    final emailController = TextEditingController(text: settings.userEmail);
    
    showDialog(
      context: context,
      builder: (ctx) => AlertDialog(
        title: const Text('Edit Profile'),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            TextField(
              controller: nameController,
              decoration: const InputDecoration(
                labelText: 'Name',
                prefixIcon: Icon(Icons.person),
              ),
            ),
            const SizedBox(height: 8),
            TextField(
              controller: emailController,
              decoration: const InputDecoration(
                labelText: 'Email',
                prefixIcon: Icon(Icons.email),
              ),
              keyboardType: TextInputType.emailAddress,
            ),
          ],
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(ctx),
            child: const Text('Cancel'),
          ),
          ElevatedButton(
            onPressed: () {
              context.read<SettingsService>().updateProfile(
                name: nameController.text,
                email: emailController.text,
              );
              Navigator.pop(ctx);
            },
            child: const Text('Save'),
          ),
        ],
      ),
    );
  }
}

class _AppearanceSection extends StatelessWidget {
  const _AppearanceSection();
  
  @override
  Widget build(BuildContext context) {
    final settingsService = context.watch<SettingsService>();
    final settings = settingsService.settings;
    
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        const _SectionHeader(title: 'APPEARANCE'),
        Card(
          margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
          child: Column(
            children: [
              ListTile(
                leading: const Icon(Icons.color_lens),
                title: const Text('Theme'),
                trailing: SegmentedButton<String>(
                  segments: const [
                    ButtonSegment(value: 'light', icon: Icon(Icons.light_mode, size: 16)),
                    ButtonSegment(value: 'system', icon: Icon(Icons.brightness_auto, size: 16)),
                    ButtonSegment(value: 'dark', icon: Icon(Icons.dark_mode, size: 16)),
                  ],
                  selected: {settings.themeMode},
                  onSelectionChanged: (selection) {
                    settingsService.updateTheme(selection.first);
                  },
                ),
              ),
              ListTile(
                leading: const Icon(Icons.text_fields),
                title: const Text('Font Size'),
                subtitle: Text('${settings.fontSize.toStringAsFixed(0)}pt'),
                trailing: SizedBox(
                  width: 150,
                  child: Slider(
                    min: 10,
                    max: 20,
                    value: settings.fontSize,
                    onChanged: settingsService.updateFontSize,
                  ),
                ),
              ),
              ListTile(
                leading: const Icon(Icons.language),
                title: const Text('Language'),
                trailing: DropdownButton<String>(
                  value: settings.language,
                  underline: const SizedBox(),
                  items: const [
                    DropdownMenuItem(value: 'th', child: Text('Thai')),
                    DropdownMenuItem(value: 'en', child: Text('English')),
                  ],
                  onChanged: (value) {
                    if (value != null) settingsService.updateLanguage(value);
                  },
                ),
              ),
            ],
          ),
        ),
      ],
    );
  }
}

class _NotificationSection extends StatelessWidget {
  const _NotificationSection();
  
  @override
  Widget build(BuildContext context) {
    final settingsService = context.watch<SettingsService>();
    final settings = settingsService.settings;
    
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        const _SectionHeader(title: 'NOTIFICATIONS'),
        Card(
          margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
          child: Column(
            children: [
              SwitchListTile(
                secondary: const Icon(Icons.notifications),
                title: const Text('Push Notifications'),
                subtitle: const Text('Receive app notifications'),
                value: settings.notificationsEnabled,
                onChanged: settingsService.updateNotifications,
              ),
              SwitchListTile(
                secondary: const Icon(Icons.volume_up),
                title: const Text('Sound'),
                subtitle: const Text('Play sounds for notifications'),
                value: settings.soundEnabled,
                onChanged: settings.notificationsEnabled
                    ? settingsService.updateSound
                    : null,
              ),
            ],
          ),
        ),
      ],
    );
  }
}

class _SecuritySection extends StatelessWidget {
  const _SecuritySection();
  
  @override
  Widget build(BuildContext context) {
    final settingsService = context.watch<SettingsService>();
    final settings = settingsService.settings;
    
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        const _SectionHeader(title: 'SECURITY'),
        Card(
          margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
          child: Column(
            children: [
              SwitchListTile(
                secondary: const Icon(Icons.fingerprint),
                title: const Text('Biometric Login'),
                subtitle: const Text('Use fingerprint or face ID'),
                value: settings.biometricEnabled,
                onChanged: settingsService.updateBiometric,
              ),
              ListTile(
                leading: const Icon(Icons.lock),
                title: const Text('Change PIN'),
                trailing: const Icon(Icons.arrow_forward_ios, size: 16),
                onTap: () {
                  ScaffoldMessenger.of(context).showSnackBar(
                    const SnackBar(content: Text('PIN change coming soon')),
                  );
                },
              ),
            ],
          ),
        ),
      ],
    );
  }
}

class _AboutSection extends StatelessWidget {
  const _AboutSection();
  
  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        const _SectionHeader(title: 'ABOUT'),
        Card(
          margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
          child: Column(
            children: [
              const ListTile(
                leading: Icon(Icons.info),
                title: Text('App Version'),
                trailing: Text('1.0.0'),
              ),
              ListTile(
                leading: const Icon(Icons.privacy_tip),
                title: const Text('Privacy Policy'),
                trailing: const Icon(Icons.open_in_new, size: 16),
                onTap: () {},
              ),
              ListTile(
                leading: const Icon(Icons.description),
                title: const Text('Terms of Service'),
                trailing: const Icon(Icons.open_in_new, size: 16),
                onTap: () {},
              ),
            ],
          ),
        ),
        const SizedBox(height: 24),
      ],
    );
  }
}
```

---

## สรุป Local Storage

### เลือกใช้อะไรเมื่อไหร่

| Storage | เหมาะกับ | ขนาด | Security |
|---------|---------|------|---------|
| SharedPreferences | Settings, tokens | เล็ก | ไม่ encrypted |
| Secure Storage | Passwords, API keys | เล็ก | Encrypted |
| Hive | App data, cache | กลาง | Optional encryption |
| ObjectBox | Complex queries | ใหญ่ | Built-in |
| SQLite | Relational data | ใหญ่ | ไม่ encrypted |

### Best Practices
```dart
// 1. Initialize ก่อนใช้
await Hive.initFlutter();
await SharedPreferences.getInstance();

// 2. ใช้ constants สำหรับ keys
static const String _tokenKey = 'auth_token';

// 3. Handle null values
final token = prefs.getString(_tokenKey) ?? '';

// 4. ใช้ secure storage สำหรับ sensitive data
final secureStorage = FlutterSecureStorage();
await secureStorage.write(key: 'password', value: 'secret');
```
