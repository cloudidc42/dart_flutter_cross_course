# Part 17: Libraries และ Packages ใน Dart

## บทนำ

Libraries และ Packages เป็นกลไกหลักในการจัดระเบียบและแชร์โค้ดใน Dart การเข้าใจระบบ libraries ช่วยให้สร้างโปรเจกต์ที่มีโครงสร้างดีและนำโค้ดกลับมาใช้ได้

## 17.1 การสร้าง Libraries

### Library พื้นฐาน

แต่ละไฟล์ `.dart` คือ library โดยค่าเริ่มต้น

```dart
// ไฟล์: math_utils.dart
// ไม่ต้องประกาศ library ก็ได้ แต่สามารถตั้งชื่อได้
library math_utils;

const double pi = 3.14159265358979;
const double e = 2.71828182845904;

double circleArea(double radius) => pi * radius * radius;
double circlePerimeter(double radius) => 2 * pi * radius;

class Fraction {
  final int numerator;
  final int denominator;

  const Fraction(this.numerator, this.denominator)
      : assert(denominator != 0, 'ตัวส่วนต้องไม่เป็นศูนย์');

  Fraction operator +(Fraction other) => Fraction(
    numerator * other.denominator + other.numerator * denominator,
    denominator * other.denominator,
  ).simplify();

  Fraction operator *(Fraction other) => Fraction(
    numerator * other.numerator,
    denominator * other.denominator,
  ).simplify();

  Fraction simplify() {
    final g = _gcd(numerator.abs(), denominator.abs());
    return Fraction(numerator ~/ g, denominator ~/ g);
  }

  static int _gcd(int a, int b) => b == 0 ? a : _gcd(b, a % b);

  @override
  String toString() => denominator == 1 ? '$numerator' : '$numerator/$denominator';
}
```

```dart
// ไฟล์: string_utils.dart
library string_utils;

/// ตัดช่องว่างและแปลงเป็นตัวพิมพ์เล็ก
String normalize(String input) => input.trim().toLowerCase();

/// ตรวจสอบ palindrome
bool isPalindrome(String s) {
  final clean = s.toLowerCase().replaceAll(RegExp(r'[^a-z0-9]'), '');
  return clean == clean.split('').reversed.join();
}

/// แปลง camelCase เป็น snake_case
String toSnakeCase(String camel) {
  return camel.replaceAllMapped(
    RegExp(r'([a-z])([A-Z])'),
    (m) => '${m[1]}_${m[2]}',
  ).toLowerCase();
}

extension StringHelper on String {
  bool get isPalindrome => string_utils.isPalindrome(this);
  String get normalized => string_utils.normalize(this);
}
```

### Part files

Part files ให้แบ่ง library เป็นหลายไฟล์แต่ยังเป็น library เดียวกัน

```dart
// ไฟล์: myapp.dart (main library file)
library myapp;

// ประกาศ parts
part 'src/models.dart';
part 'src/services.dart';
part 'src/utils.dart';

// โค้ดใน main library
const String appName = 'MyApp';
const String version = '1.0.0';
```

```dart
// ไฟล์: src/models.dart
part of myapp;

class User {
  final String id;
  final String name;
  final String email;

  const User({required this.id, required this.name, required this.email});

  @override
  String toString() => 'User($name)';
}

class Product {
  final String id;
  final String name;
  final double price;

  const Product({required this.id, required this.name, required this.price});
}
```

```dart
// ไฟล์: src/services.dart
part of myapp;

class UserService {
  final Map<String, User> _users = {};

  void addUser(User user) => _users[user.id] = user;
  User? findUser(String id) => _users[id];
  List<User> get allUsers => _users.values.toList();
}

class ProductService {
  final List<Product> _products = [];

  void addProduct(Product product) => _products.add(product);
  List<Product> get allProducts => List.unmodifiable(_products);
}
```

## 17.2 import/export

### import

```dart
// import library มาตรฐาน
import 'dart:core';     // automatic (ไม่ต้อง import)
import 'dart:math';
import 'dart:convert';
import 'dart:io';
import 'dart:async';
import 'dart:collection';

// import package จาก pub.dev
import 'package:http/http.dart';
import 'package:flutter/material.dart';
import 'package:shared_preferences/shared_preferences.dart';

// import ไฟล์ในโปรเจกต์
import 'math_utils.dart';
import '../utils/string_utils.dart';
import 'package:myapp/src/models.dart';

// import พร้อม prefix เพื่อหลีกเลี่ยงชื่อชน
import 'dart:math' as math;
import 'package:http/http.dart' as http;

void usePrefixedImport() {
  var random = math.Random();
  print(random.nextInt(100));

  var client = http.Client();
  // ...
}
```

### export

```dart
// ไฟล์: lib/utils.dart
// รวบรวม exports ทั้งหมดไว้ที่เดียว

export 'src/math_utils.dart';
export 'src/string_utils.dart';
export 'src/date_utils.dart';

// export บางส่วน
export 'src/advanced_utils.dart' show AdvancedMath, Statistics;
export 'src/internal.dart' hide InternalHelper, _PrivateClass;
```

```dart
// ไฟล์หลักของ package
// lib/mypackage.dart
library mypackage;

export 'src/models/index.dart';
export 'src/services/index.dart';
export 'src/utils/index.dart';

// ผู้ใช้ import แค่ที่เดียว
// import 'package:mypackage/mypackage.dart';
```

## 17.3 show/hide

```dart
// show: import เฉพาะที่ระบุ
import 'dart:math' show Random, sqrt, pi;

void useShow() {
  var r = Random();
  print(sqrt(16)); // 4.0
  print(pi);       // 3.14159...

  // print(Point(1, 2)); // Error: Point ไม่ถูก import
}

// hide: import ทั้งหมดยกเว้นที่ระบุ
import 'dart:core' hide DateTime, Duration;
// (ปกติไม่ควรซ่อน core types)

// ตัวอย่างจริง: หลีกเลี่ยงชื่อชน
import 'package:flutter/material.dart' hide TextStyle;
import 'src/custom_text_style.dart'; // TextStyle ของเราเอง

// หรือใช้ prefix แทน
import 'package:flutter/material.dart' as flutter;
// จะใช้ flutter.TextStyle สำหรับ Flutter's TextStyle
```

## 17.4 Deferred Loading

Deferred loading (lazy loading) โหลด library เมื่อต้องการใช้

```dart
// ประกาศ deferred import
import 'dart:math' deferred as lazyMath;
import 'package:heavy_library/heavy_library.dart' deferred as heavy;

Future<void> useDeferred() async {
  print('ก่อนโหลด...');

  // โหลดเมื่อต้องการ
  await lazyMath.loadLibrary();
  print('โหลดแล้ว');

  // ใช้งานได้
  var result = lazyMath.sqrt(16);
  print('sqrt(16) = $result');
}

// ตัวอย่างใน Flutter
// import 'package:my_chart_library/charts.dart' deferred as charts;

// class MyWidget extends StatefulWidget {
//   Future<void> _loadCharts() async {
//     await charts.loadLibrary();
//     setState(() => _chartsLoaded = true);
//   }

//   Widget build(BuildContext context) {
//     if (!_chartsLoaded) return CircularProgressIndicator();
//     return charts.BarChart(data: myData);
//   }
// }
```

## 17.5 pub.dev Packages

### pubspec.yaml

```yaml
name: my_flutter_app
description: แอป Flutter ของฉัน
version: 1.0.0+1

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: '>=3.10.0'

dependencies:
  flutter:
    sdk: flutter

  # HTTP
  http: ^1.1.0
  dio: ^5.3.0

  # State management
  provider: ^6.0.0
  riverpod: ^2.4.0
  bloc: ^8.1.0
  flutter_bloc: ^8.1.0

  # Storage
  shared_preferences: ^2.2.0
  hive: ^2.2.3
  hive_flutter: ^1.1.0
  sqflite: ^2.3.0

  # Navigation
  go_router: ^12.0.0

  # JSON
  json_annotation: ^4.8.0

  # UI
  cached_network_image: ^3.3.0
  flutter_svg: ^2.0.0
  lottie: ^2.7.0

  # Utils
  intl: ^0.18.0
  uuid: ^4.1.0
  logger: ^2.0.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0
  build_runner: ^2.4.0
  json_serializable: ^6.7.0
  mockito: ^5.4.0
```

## 17.6 Popular Packages Overview

### HTTP Requests

```dart
// http package
import 'package:http/http.dart' as http;
import 'dart:convert';

class ApiService {
  static const baseUrl = 'https://jsonplaceholder.typicode.com';

  // GET request
  Future<List<Map<String, dynamic>>> getUsers() async {
    final response = await http.get(
      Uri.parse('$baseUrl/users'),
    );

    if (response.statusCode == 200) {
      return List<Map<String, dynamic>>.from(
        jsonDecode(response.body),
      );
    }
    throw Exception('โหลดข้อมูลไม่สำเร็จ: ${response.statusCode}');
  }

  // POST request
  Future<Map<String, dynamic>> createPost({
    required String title,
    required String body,
    required int userId,
  }) async {
    final response = await http.post(
      Uri.parse('$baseUrl/posts'),
      headers: {'Content-Type': 'application/json'},
      body: jsonEncode({
        'title': title,
        'body': body,
        'userId': userId,
      }),
    );

    return jsonDecode(response.body) as Map<String, dynamic>;
  }
}
```

### Shared Preferences

```dart
// shared_preferences
import 'package:shared_preferences/shared_preferences.dart';

class PrefsService {
  static const _keyTheme = 'theme';
  static const _keyLanguage = 'language';
  static const _keyUserId = 'user_id';

  final SharedPreferences _prefs;

  PrefsService(this._prefs);

  static Future<PrefsService> create() async {
    final prefs = await SharedPreferences.getInstance();
    return PrefsService(prefs);
  }

  // Theme
  String get theme => _prefs.getString(_keyTheme) ?? 'light';
  Future<void> setTheme(String theme) => _prefs.setString(_keyTheme, theme);

  // Language
  String get language => _prefs.getString(_keyLanguage) ?? 'th';
  Future<void> setLanguage(String lang) => _prefs.setString(_keyLanguage, lang);

  // User ID
  int? get userId => _prefs.getInt(_keyUserId);
  Future<void> setUserId(int id) => _prefs.setInt(_keyUserId, id);
  Future<void> clearUserId() => _prefs.remove(_keyUserId);

  Future<void> clearAll() => _prefs.clear();
}
```

### Logger

```dart
// logger package
import 'package:logger/logger.dart';

class AppLogger {
  static final _logger = Logger(
    printer: PrettyPrinter(
      methodCount: 2,
      errorMethodCount: 8,
      lineLength: 120,
      colors: true,
      printEmojis: true,
    ),
  );

  static void debug(String message, [dynamic error, StackTrace? stack]) =>
      _logger.d(message, error: error, stackTrace: stack);

  static void info(String message) => _logger.i(message);
  static void warning(String message) => _logger.w(message);
  static void error(String message, [dynamic error, StackTrace? stack]) =>
      _logger.e(message, error: error, stackTrace: stack);
}
```

### json_serializable

```dart
// json_serializable + build_runner
import 'package:json_annotation/json_annotation.dart';

part 'user.g.dart'; // generated file

@JsonSerializable()
class User {
  final int id;
  final String name;
  final String email;

  @JsonKey(name: 'created_at')
  final DateTime createdAt;

  @JsonKey(defaultValue: true)
  final bool isActive;

  const User({
    required this.id,
    required this.name,
    required this.email,
    required this.createdAt,
    required this.isActive,
  });

  factory User.fromJson(Map<String, dynamic> json) =>
      _$UserFromJson(json);

  Map<String, dynamic> toJson() => _$UserToJson(this);
}

// run: dart run build_runner build
// generates: user.g.dart
```

### intl (Internationalization)

```dart
import 'package:intl/intl.dart';

class Formatter {
  static String thaiDate(DateTime date) {
    final formatter = DateFormat('d MMMM y', 'th_TH');
    return formatter.format(date);
  }

  static String currency(double amount, {String symbol = '฿'}) {
    final formatter = NumberFormat.currency(
      locale: 'th_TH',
      symbol: symbol,
      decimalDigits: 2,
    );
    return formatter.format(amount);
  }

  static String percentage(double value, {int decimals = 1}) {
    return NumberFormat.percentPattern().format(value / 100);
  }

  static String compact(num value) {
    return NumberFormat.compact(locale: 'th_TH').format(value);
  }
}

void testFormatter() {
  print(Formatter.thaiDate(DateTime(2024, 1, 15)));
  print(Formatter.currency(1234567.89));
  print(Formatter.compact(1500000)); // 1.5M
}
```

## 17.7 Workshop: Create a Utility Library

สร้าง utility library สำหรับโปรเจกต์

### โครงสร้างไฟล์

```
lib/
├── myutils.dart          # main export file
└── src/
    ├── validation.dart   # validation utilities
    ├── formatting.dart   # formatting utilities
    ├── collections.dart  # collection utilities
    └── result.dart       # Result type
```

```dart
// ===== lib/myutils.dart =====
library myutils;

export 'src/validation.dart';
export 'src/formatting.dart';
export 'src/collections.dart';
export 'src/result.dart';
```

```dart
// ===== lib/src/result.dart =====

/// ผลลัพธ์ที่อาจสำเร็จหรือล้มเหลว
sealed class Result<T> {
  const Result();

  bool get isSuccess => this is Success<T>;
  bool get isFailure => this is Failure<T>;

  T? get valueOrNull =>
      isSuccess ? (this as Success<T>).value : null;

  String? get errorOrNull =>
      isFailure ? (this as Failure<T>).message : null;

  T getOrElse(T defaultValue) =>
      isSuccess ? (this as Success<T>).value : defaultValue;

  T getOrThrow() {
    if (this is Success<T>) return (this as Success<T>).value;
    throw Exception((this as Failure<T>).message);
  }

  Result<R> map<R>(R Function(T) transform) {
    return switch (this) {
      Success(:final value) => Result.ok(transform(value)),
      Failure(:final message) => Result.fail(message),
    };
  }

  Result<R> flatMap<R>(Result<R> Function(T) transform) {
    return switch (this) {
      Success(:final value) => transform(value),
      Failure(:final message) => Result.fail(message),
    };
  }

  void when({
    required void Function(T) success,
    required void Function(String) failure,
  }) {
    switch (this) {
      case Success(:final value): success(value);
      case Failure(:final message): failure(message);
    }
  }

  factory Result.ok(T value) = Success<T>;
  factory Result.fail(String message) = Failure<T>;

  static Result<T> tryCatch<T>(T Function() fn) {
    try {
      return Result.ok(fn());
    } catch (e) {
      return Result.fail(e.toString());
    }
  }

  static Future<Result<T>> tryAsync<T>(Future<T> Function() fn) async {
    try {
      return Result.ok(await fn());
    } catch (e) {
      return Result.fail(e.toString());
    }
  }
}

class Success<T> extends Result<T> {
  final T value;
  const Success(this.value);

  @override
  String toString() => 'Success($value)';
}

class Failure<T> extends Result<T> {
  final String message;
  const Failure(this.message);

  @override
  String toString() => 'Failure($message)';
}
```

```dart
// ===== lib/src/validation.dart =====

/// ข้อมูลผลการตรวจสอบ
class ValidationResult {
  final bool isValid;
  final Map<String, String> errors;

  const ValidationResult({
    required this.isValid,
    this.errors = const {},
  });

  factory ValidationResult.ok() =>
      const ValidationResult(isValid: true);

  factory ValidationResult.fail(Map<String, String> errors) =>
      ValidationResult(isValid: false, errors: errors);

  @override
  String toString() => isValid
      ? 'ValidationResult.ok'
      : 'ValidationResult.fail($errors)';
}

/// Validator functions
class Validators {
  static bool isEmail(String? value) {
    if (value == null || value.isEmpty) return false;
    return RegExp(
      r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$',
    ).hasMatch(value);
  }

  static bool isThaiPhone(String? value) {
    if (value == null) return false;
    final cleaned = value.replaceAll(RegExp(r'[\s-]'), '');
    return RegExp(r'^0[689]\d{8}$').hasMatch(cleaned);
  }

  static bool isMinLength(String? value, int min) {
    return (value?.length ?? 0) >= min;
  }

  static bool isMaxLength(String? value, int max) {
    return (value?.length ?? 0) <= max;
  }

  static bool isPositiveNumber(num? value) {
    return value != null && value > 0;
  }

  static bool isInRange(num? value, num min, num max) {
    return value != null && value >= min && value <= max;
  }

  static bool isUrl(String? value) {
    if (value == null || value.isEmpty) return false;
    return Uri.tryParse(value)?.isAbsolute ?? false;
  }
}

/// Validation rule
typedef ValidationRule<T> = String? Function(T value);

/// ตรวจสอบหลาย fields พร้อมกัน
class FormValidator {
  final Map<String, List<ValidationRule<dynamic>>> _rules = {};

  void addRule<T>(String field, ValidationRule<T> rule) {
    _rules.putIfAbsent(field, () => []).add(
      (dynamic v) => rule(v as T),
    );
  }

  ValidationResult validate(Map<String, dynamic> data) {
    final errors = <String, String>{};

    for (final entry in _rules.entries) {
      final field = entry.key;
      final value = data[field];

      for (final rule in entry.value) {
        final error = rule(value);
        if (error != null) {
          errors[field] = error;
          break; // หยุดที่ error แรก
        }
      }
    }

    return errors.isEmpty
        ? ValidationResult.ok()
        : ValidationResult.fail(errors);
  }
}
```

```dart
// ===== lib/src/formatting.dart =====

import 'dart:math' as math;

/// Format ตัวเลข
class NumberFormatter {
  static String currency(
    double amount, {
    String symbol = '฿',
    int decimals = 2,
  }) {
    final formatted = amount.toStringAsFixed(decimals)
        .replaceAllMapped(
          RegExp(r'(\d)(?=(\d{3})+(?!\d))'),
          (m) => '${m[1]},',
        );
    return '$symbol$formatted';
  }

  static String percentage(double value, {int decimals = 1}) {
    return '${value.toStringAsFixed(decimals)}%';
  }

  static String compact(num value) {
    if (value.abs() >= 1e9) return '${(value / 1e9).toStringAsFixed(1)}B';
    if (value.abs() >= 1e6) return '${(value / 1e6).toStringAsFixed(1)}M';
    if (value.abs() >= 1e3) return '${(value / 1e3).toStringAsFixed(1)}K';
    return value.toString();
  }

  static String fileSize(int bytes) {
    if (bytes < 1024) return '${bytes}B';
    if (bytes < 1024 * 1024) return '${(bytes / 1024).toStringAsFixed(1)}KB';
    if (bytes < 1024 * 1024 * 1024) {
      return '${(bytes / (1024 * 1024)).toStringAsFixed(1)}MB';
    }
    return '${(bytes / (1024 * 1024 * 1024)).toStringAsFixed(2)}GB';
  }
}

/// Format วันที่
class DateFormatter {
  static const _thaiMonths = [
    'ม.ค.', 'ก.พ.', 'มี.ค.', 'เม.ย.', 'พ.ค.', 'มิ.ย.',
    'ก.ค.', 'ส.ค.', 'ก.ย.', 'ต.ค.', 'พ.ย.', 'ธ.ค.',
  ];

  static const _thaiDays = [
    'จันทร์', 'อังคาร', 'พุธ', 'พฤหัสบดี', 'ศุกร์', 'เสาร์', 'อาทิตย์',
  ];

  static String thaiDate(DateTime date, {bool includeBuddhistYear = true}) {
    final year = includeBuddhistYear ? date.year + 543 : date.year;
    return '${date.day} ${_thaiMonths[date.month - 1]} $year';
  }

  static String relative(DateTime date) {
    final now = DateTime.now();
    final diff = now.difference(date);

    if (diff.inMinutes < 1) return 'เมื่อกี้';
    if (diff.inHours < 1) return '${diff.inMinutes} นาทีที่แล้ว';
    if (diff.inDays < 1) return '${diff.inHours} ชั่วโมงที่แล้ว';
    if (diff.inDays < 7) return '${diff.inDays} วันที่แล้ว';
    return thaiDate(date);
  }

  static String duration(Duration d) {
    if (d.inSeconds < 60) return '${d.inSeconds}s';
    if (d.inMinutes < 60) return '${d.inMinutes}m ${d.inSeconds % 60}s';
    if (d.inHours < 24) return '${d.inHours}h ${d.inMinutes % 60}m';
    return '${d.inDays}d ${d.inHours % 24}h';
  }
}
```

```dart
// ===== lib/src/collections.dart =====

/// Extension methods สำหรับ collections

extension ListUtils<T> on List<T> {
  T? get firstOrNull => isEmpty ? null : first;
  T? get lastOrNull => isEmpty ? null : last;

  T? get randomElement {
    if (isEmpty) return null;
    return this[math.Random().nextInt(length)];
  }

  List<List<T>> chunk(int size) {
    return [
      for (var i = 0; i < length; i += size)
        sublist(i, (i + size).clamp(0, length))
    ];
  }

  Map<K, List<T>> groupBy<K>(K Function(T) key) {
    return fold({}, (map, item) {
      map.putIfAbsent(key(item), () => []).add(item);
      return map;
    });
  }

  List<T> distinctBy<K>(K Function(T) key) {
    final seen = <K>{};
    return where((item) => seen.add(key(item))).toList();
  }

  List<(T, int)> withIndex() =>
      asMap().entries.map((e) => (e.value, e.key)).toList();

  List<(T, T)> zip(List<T> other) {
    final len = math.min(length, other.length);
    return List.generate(len, (i) => (this[i], other[i]));
  }
}

extension MapUtils<K, V> on Map<K, V> {
  Map<K, V> merge(Map<K, V> other) => {...this, ...other};

  Map<K, V> filter(bool Function(K, V) predicate) =>
      Map.fromEntries(entries.where((e) => predicate(e.key, e.value)));

  Map<K, R> mapValues<R>(R Function(V) transform) =>
      map((k, v) => MapEntry(k, transform(v)));

  V getOrPut(K key, V Function() defaultValue) {
    if (!containsKey(key)) this[key] = defaultValue();
    return this[key] as V;
  }
}

// ===== Workshop Test =====
void main() {
  print('=== Workshop: Utility Library ===\n');

  // Test Result
  print('1. Result type:');
  var r1 = Result.ok(42);
  var r2 = Result.fail('ข้อผิดพลาด');

  r1.when(
    success: (v) => print('  สำเร็จ: $v'),
    failure: (e) => print('  ล้มเหลว: $e'),
  );

  r2.when(
    success: (v) => print('  สำเร็จ: $v'),
    failure: (e) => print('  ล้มเหลว: $e'),
  );

  var r3 = Result.tryCatch(() => int.parse('42'));
  var r4 = Result.tryCatch(() => int.parse('abc'));
  print('  parse("42"): $r3');
  print('  parse("abc"): $r4');

  // Test Validators
  print('\n2. Validators:');
  print('  email: ${Validators.isEmail("user@example.com")}');
  print('  phone: ${Validators.isThaiPhone("0812345678")}');

  var validator = FormValidator();
  validator.addRule<String>('email', (v) {
    if (!Validators.isEmail(v)) return 'รูปแบบ email ไม่ถูกต้อง';
    return null;
  });
  validator.addRule<String>('password', (v) {
    if (!Validators.isMinLength(v, 8)) return 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร';
    return null;
  });

  var result1 = validator.validate({'email': 'valid@test.com', 'password': 'secure123'});
  var result2 = validator.validate({'email': 'invalid', 'password': '123'});
  print('  valid: $result1');
  print('  invalid: $result2');

  // Test Formatters
  print('\n3. Formatters:');
  print('  currency: ${NumberFormatter.currency(1234567.89)}');
  print('  percentage: ${NumberFormatter.percentage(85.5)}');
  print('  compact: ${NumberFormatter.compact(1500000)}');
  print('  fileSize: ${NumberFormatter.fileSize(1536)}');
  print('  date: ${DateFormatter.thaiDate(DateTime(2024, 1, 15))}');
  print('  relative: ${DateFormatter.relative(DateTime.now().subtract(Duration(hours: 2)))}');

  // Test Collection utils
  print('\n4. Collection Utils:');
  var numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
  print('  chunks: ${numbers.chunk(3)}');

  var words = ['apple', 'banana', 'avocado', 'cherry', 'blueberry'];
  var grouped = words.groupBy((w) => w[0]);
  print('  grouped: $grouped');

  var people = [
    {'name': 'อลิส', 'dept': 'IT'},
    {'name': 'บ็อบ', 'dept': 'HR'},
    {'name': 'ชาร์ลี', 'dept': 'IT'},
    {'name': 'เดฟ', 'dept': 'IT'},
  ];

  var byDept = people.groupBy((p) => p['dept']!);
  byDept.forEach((dept, members) {
    print('  $dept: ${members.map((m) => m['name']).join(', ')}');
  });

  print('\n=== เสร็จสิ้น Workshop ===');
}
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **การสร้าง libraries** - library, part files
2. **import/export** - นำเข้าและส่งออก libraries
3. **show/hide** - ควบคุมสิ่งที่ import
4. **Deferred loading** - โหลด library แบบ lazy
5. **pub.dev packages** - การใช้ packages ยอดนิยม
6. **Workshop** - สร้าง utility library สมบูรณ์

### Best Practices

- จัดโครงสร้าง library ให้ชัดเจน (public API vs internal)
- ใช้ `export` รวม API ไว้ที่ไฟล์เดียว
- ใช้ `show/hide` เมื่อต้องการ import เฉพาะส่วน
- ใช้ prefix เมื่อมีชื่อชน
- ทำ pub.dev package เมื่อต้องการแชร์ code ระหว่างโปรเจกต์
