# Part 14: Null Safety Deep Dive

## บทนำ

Null Safety เป็นหนึ่งในฟีเจอร์สำคัญที่สุดของ Dart เวอร์ชัน 2.12+ ช่วยป้องกัน `NullPointerException` ณ เวลา compile ไม่ใช่เวลา runtime ทำให้โค้ดปลอดภัยและเชื่อถือได้มากขึ้น

## 14.1 Nullable vs Non-nullable

### พื้นฐาน

```dart
// Non-nullable (default) - ต้องมีค่าเสมอ
String name = 'สมชาย';       // OK
// name = null;               // Compile error!

// Nullable - ใส่ ? เพื่อบอกว่าอาจเป็น null ได้
String? nullableName = 'สมชาย'; // OK
nullableName = null;             // OK

// Nullable types ต่างๆ
int? nullableInt = null;
double? nullableDouble = 3.14;
List<String>? nullableList = null;
Map<String, int>? nullableMap = {'a': 1};

// การตรวจสอบ null
void processName(String? name) {
  if (name != null) {
    // ใน block นี้ name เป็น String (non-nullable)
    print(name.toUpperCase()); // OK
  }
}
```

### Null-aware operators

```dart
// ?. (null-safe method call)
String? text = null;
print(text?.length);        // null (ไม่ crash)
print(text?.toUpperCase()); // null

String? email = 'user@example.com';
print(email?.split('@')); // ['user', 'example.com']

// ?? (null coalescing)
String? username = null;
String displayName = username ?? 'Guest';
print(displayName); // Guest

int? count = null;
int total = count ?? 0;
print(total); // 0

// ??= (null-aware assignment)
String? cache;
cache ??= 'computed value'; // กำหนดค่าถ้า null
print(cache); // computed value
cache ??= 'other value'; // ไม่เปลี่ยนถ้าไม่ null
print(cache); // computed value

// ?[] (null-aware index)
List<String>? list = null;
String? first = list?[0]; // null

Map<String, int>? map = {'key': 42};
int? value = map?['key']; // 42

// Chaining null-aware operators
class Address {
  String? city;
  Address({this.city});
}

class Person {
  Address? address;
  Person({this.address});
}

Person? person = Person(
  address: Address(city: 'กรุงเทพฯ'),
);

String? city = person?.address?.city;
print(city); // กรุงเทพฯ

Person? noPerson = null;
String? noCity = noPerson?.address?.city;
print(noCity); // null
```

### Null checks และ Type promotion

```dart
void typePromotion() {
  String? nullableStr = 'สวัสดี';

  // if-null check
  if (nullableStr != null) {
    // Dart รู้ว่า nullableStr เป็น String แน่ๆ ใน block นี้
    print(nullableStr.length); // OK! ไม่ต้อง !
  }

  // Promoted ใน else ก็ไม่ได้
  // print(nullableStr.length); // Still nullable here

  // นอกจากเราจะ return ใน if
  if (nullableStr == null) return;
  // ตั้งแต่บรรทัดนี้ nullableStr เป็น String แน่ๆ
  print(nullableStr.length); // OK!

  // Null check ใน logical expressions
  String? value = 'hello';
  if (value != null && value.length > 3) {
    print('ยาวกว่า 3 ตัวอักษร: $value');
  }
}

// Pattern matching (Dart 3+)
void patternMatching(String? value) {
  switch (value) {
    case null:
      print('ค่าเป็น null');
    case '':
      print('ค่าว่างเปล่า');
    case final s when s.length > 10:
      print('ยาวมาก: $s');
    case final s:
      print('ค่า: $s');
  }
}
```

## 14.2 Late Variables

`late` บอก Dart ว่าตัวแปรจะถูกกำหนดค่าก่อนใช้งาน แต่ไม่ต้องกำหนดค่าตอนประกาศ

```dart
// late initialization
class DatabaseConnection {
  late String _connectionString;
  late bool _isConnected;

  void connect(String host, int port) {
    _connectionString = 'host=$host;port=$port';
    _isConnected = true;
    print('เชื่อมต่อ: $_connectionString');
  }

  void query(String sql) {
    if (!_isConnected) {
      throw StateError('ยังไม่ได้เชื่อมต่อ database');
    }
    print('Query: $sql');
  }
}

// late ที่ไม่เคยใช้งาน = ไม่เสียงาน (lazy)
class ExpensiveResource {
  late final String _data = _loadData(); // โหลดเมื่อเรียกใช้ครั้งแรก

  String _loadData() {
    print('กำลังโหลดข้อมูล...');
    return 'ข้อมูลที่โหลดแล้ว';
  }

  void use() {
    print(_data); // โหลดตอนนี้
    print(_data); // ไม่โหลดซ้ำ (cached)
  }
}

// late final - กำหนดครั้งเดียว
class Config {
  late final String apiKey;
  late final String baseUrl;

  void initialize(Map<String, String> env) {
    apiKey = env['API_KEY'] ?? '';
    baseUrl = env['BASE_URL'] ?? 'https://api.default.com';
    // apiKey = 'other'; // Error: late final ถูก assign แล้ว
  }
}

// ตัวอย่าง Flutter: late initialization
class MyWidget {
  // ใน Flutter, StatefulWidget ใช้ late บ่อย
  late final TextEditingController _controller;
  late final AnimationController _animController;

  void initState() {
    // กำหนดค่าใน initState
    _controller = TextEditingController();
    // _animController = AnimationController(vsync: this, ...);
  }

  void dispose() {
    _controller.dispose();
  }
}

// late กับ abstract class
abstract class Animal {
  late String name; // subclass ต้องกำหนด
}

class Dog extends Animal {
  Dog(String name) {
    this.name = name;
  }
}
```

### เมื่อ late ไม่ปลอดภัย

```dart
class DangerousLate {
  late String value;

  void useWithoutSet() {
    print(value); // LateInitializationError ถ้ายังไม่ set!
  }
}

// ป้องกันด้วยการใช้ nullable แทน
class SaferApproach {
  String? _value;
  bool _initialized = false;

  void setValue(String v) {
    _value = v;
    _initialized = true;
  }

  String getValue() {
    if (!_initialized) throw StateError('ยังไม่ได้กำหนดค่า');
    return _value!;
  }
}
```

## 14.3 Null Assertions

`!` operator บอก Dart ว่าค่านี้ไม่ใช่ null (ถ้าผิด = runtime error)

```dart
// Bang operator (!)
String? nullable = 'hello';
String nonNull = nullable!; // บอก Dart ว่าแน่ใจว่าไม่ null

// เมื่อใช้อย่างไม่ระวัง
void dangerousAssert(String? value) {
  print(value!.length); // NullCheckFailedException ถ้า value เป็น null!
}

// ควรตรวจสอบก่อนใช้ !
void safeAssert(String? value) {
  if (value == null) return;
  print(value.length); // Promoted, ไม่ต้องใช้ !
}

// หรือใช้ ?? แทน !
void betterApproach(String? value) {
  final length = value?.length ?? 0;
  print(length);
}

// ตัวอย่างที่ ! สมเหตุสมผล
class ParsedData {
  final Map<String, dynamic> _data;

  ParsedData(this._data);

  // เราแน่ใจว่า 'id' มีค่า (validated ก่อนแล้ว)
  int get id => (_data['id'] as int?)!;

  // แต่ดีกว่าถ้าใช้
  int get idSafe => _data['id'] as int? ?? 0;
}

// Null assertion ใน Flutter
class FlutterExample {
  BuildContext? _context;

  void showSnackBar(String message) {
    // context อาจเป็น null ถ้า widget unmounted
    if (_context != null && _context!.mounted) {
      ScaffoldMessenger.of(_context!).showSnackBar(
        SnackBar(content: Text(message)),
      );
    }
  }
}
```

### asserting non-null ใน function signatures

```dart
// ใช้ required กับ named parameters
void createUser({
  required String name,
  required String email,
  int? age,
}) {
  print('สร้าง: $name ($email)');
}

// required ทำให้ compiler ตรวจสอบ
void testRequired() {
  createUser(name: 'สมชาย', email: 'somchai@test.com'); // OK
  createUser(name: 'สมหญิง', email: 'somying@test.com', age: 25); // OK
  // createUser(name: 'test'); // Compile error: email required
}

// Non-nullable return types
String getGreeting(String? name) {
  return 'สวัสดี, ${name ?? 'ผู้มาเยือน'}!';
}

// Careful with List.first on potentially empty lists
void listExample() {
  List<String> items = ['a', 'b', 'c'];
  String first = items.first; // OK ถ้า list ไม่ว่าง

  List<String> empty = [];
  // String bad = empty.first; // StateError at runtime!
  String safe = empty.firstOrNull ?? 'default';
}

extension<T> on List<T> {
  T? get firstOrNull => isEmpty ? null : first;
}
```

## 14.4 Migration Patterns

### จาก pre-null-safety เป็น null-safe

```dart
// ก่อน null safety (Dart < 2.12)
// String getName(Map<String, String> user) {
//   return user['name']; // อาจ return null
// }

// หลัง null safety
String getName(Map<String, String> user) {
  return user['name'] ?? 'Unknown';
}

// หรือ return nullable
String? getNameNullable(Map<String, String> user) {
  return user['name'];
}

// ก่อน: parameter อาจเป็น null
// void process(String value) {
//   if (value == null) return;
//   ...
// }

// หลัง: ชัดเจนว่า nullable
void processOld(String? value) {
  if (value == null) return;
  // ประมวลผล
}

// หรือบังคับว่าต้องไม่ null
void processStrict(String value) {
  // ผู้เรียกต้องรับผิดชอบให้ non-null
}

// ก่อน: class ที่ field อาจ null
// class User {
//   String name; // อาจไม่ถูก initialize
//   User(this.name);
// }

// หลัง: ชัดเจน
class User {
  final String name;      // ต้องมีค่าเสมอ
  final String? email;    // อาจไม่มี
  final int age;          // ต้องมี

  const User({
    required this.name,
    this.email,
    required this.age,
  });
}
```

### Null-safe API design

```dart
// Pattern 1: Return nullable ถ้าอาจไม่มีค่า
class UserRepository {
  final Map<String, User> _store = {};

  // ดีกว่าการ throw exception สำหรับ "not found"
  User? findById(String id) => _store[id];

  // ถ้าต้องการ non-null, throw exception
  User getById(String id) {
    final user = _store[id];
    if (user == null) throw ArgumentError('ไม่พบผู้ใช้: $id');
    return user;
  }

  // หรือ return default
  User getOrDefault(String id) {
    return _store[id] ?? User(name: 'Unknown', age: 0);
  }
}

// Pattern 2: Validation ก่อน action
class OrderService {
  String? _customerId;
  List<String> _items = [];

  void setCustomer(String id) => _customerId = id;
  void addItem(String item) => _items.add(item);

  void placeOrder() {
    final customerId = _customerId;
    if (customerId == null) {
      throw StateError('กรุณาเลือกลูกค้าก่อน');
    }
    if (_items.isEmpty) {
      throw StateError('กรุณาเพิ่มสินค้าก่อน');
    }

    print('สั่งซื้อสำหรับ: $customerId, สินค้า: $_items');
  }
}

// Pattern 3: Builder pattern กับ null safety
class QueryBuilder {
  String? _table;
  List<String> _conditions = [];
  int? _limit;

  QueryBuilder from(String table) {
    _table = table;
    return this;
  }

  QueryBuilder where(String condition) {
    _conditions.add(condition);
    return this;
  }

  QueryBuilder limit(int n) {
    _limit = n;
    return this;
  }

  String build() {
    final table = _table ?? (throw StateError('ต้องระบุ table'));
    var query = 'SELECT * FROM $table';
    if (_conditions.isNotEmpty) {
      query += ' WHERE ${_conditions.join(' AND ')}';
    }
    if (_limit != null) {
      query += ' LIMIT $_limit';
    }
    return query;
  }
}

void testQueryBuilder() {
  final query = QueryBuilder()
      .from('users')
      .where('age > 18')
      .where('active = true')
      .limit(10)
      .build();
  print(query);
}
```

## 14.5 Best Practices

### Prefer non-nullable

```dart
// ❌ ใช้ nullable โดยไม่จำเป็น
class BadExample {
  String? name; // ทำไม? name ต้องมีค่าเสมอ
  int? count;   // ถ้า count ไม่มีค่า ควรเป็น 0

  BadExample({this.name, this.count});
}

// ✅ ใช้ non-nullable เมื่อทำได้
class GoodExample {
  String name;
  int count;

  GoodExample({required this.name, this.count = 0});
}

// ❌ Null check ซ้ำๆ
void badNullHandling(String? value) {
  if (value != null) {
    print(value.length);
  }
  if (value != null) {
    print(value.toUpperCase());
  }
}

// ✅ Null check ครั้งเดียว
void goodNullHandling(String? value) {
  if (value == null) return;
  // ตั้งแต่นี้ value เป็น non-null
  print(value.length);
  print(value.toUpperCase());
}
```

### Null-safe collections

```dart
void nullSafeCollections() {
  // List ที่มี nullable items
  List<String?> nullableList = ['a', null, 'b', null, 'c'];

  // กรอง null ออก
  List<String> nonNullList = nullableList
      .whereType<String>()
      .toList();
  print(nonNullList); // [a, b, c]

  // หรือใช้ where
  List<String> filtered = nullableList
      .where((item) => item != null)
      .map((item) => item!) // ปลอดภัยหลัง filter
      .toList();

  // Map ที่ value เป็น nullable
  Map<String, int?> scores = {
    'อลิส': 95,
    'บ็อบ': null,
    'ชาร์ลี': 87,
  };

  // หาค่าเฉลี่ยโดยข้าม null
  var validScores = scores.values
      .whereType<int>()
      .toList();

  if (validScores.isNotEmpty) {
    var avg = validScores.reduce((a, b) => a + b) / validScores.length;
    print('คะแนนเฉลี่ย: $avg');
  }
}
```

### Nullable in class hierarchy

```dart
// Abstract class กับ nullable
abstract class Drawable {
  String? get tooltip;  // nullable - อาจไม่มี tooltip
  String get label;     // non-nullable - ต้องมีเสมอ
  void draw();
}

class Button extends Drawable {
  @override
  final String label;

  @override
  final String? tooltip;

  Button(this.label, {this.tooltip});

  @override
  void draw() => print('Button: $label${tooltip != null ? ' ($tooltip)' : ''}');
}

// Covariant return types
abstract class Repository<T> {
  Future<T?> findById(String id); // nullable เพราะอาจไม่พบ
  Future<List<T>> findAll();      // non-nullable (อาจว่าง)
}

class UserRepo extends Repository<User> {
  @override
  Future<User?> findById(String id) async {
    await Future.delayed(Duration(milliseconds: 100));
    return null; // not found
  }

  @override
  Future<List<User>> findAll() async {
    return [];
  }
}
```

## 14.6 Workshop: Safe Data Parsing

สร้างระบบ parsing ข้อมูลที่ปลอดภัยจาก JSON

```dart
import 'dart:convert';

// ===== Safe Parsing Utilities =====

extension SafeJson on Map<String, dynamic> {
  // ดึงค่าแบบปลอดภัย
  String? getString(String key) {
    final value = this[key];
    if (value == null) return null;
    return value.toString();
  }

  String getStringOrDefault(String key, [String defaultValue = '']) {
    return getString(key) ?? defaultValue;
  }

  int? getInt(String key) {
    final value = this[key];
    if (value == null) return null;
    if (value is int) return value;
    if (value is String) return int.tryParse(value);
    if (value is double) return value.toInt();
    return null;
  }

  int getIntOrDefault(String key, [int defaultValue = 0]) {
    return getInt(key) ?? defaultValue;
  }

  double? getDouble(String key) {
    final value = this[key];
    if (value == null) return null;
    if (value is double) return value;
    if (value is int) return value.toDouble();
    if (value is String) return double.tryParse(value);
    return null;
  }

  bool? getBool(String key) {
    final value = this[key];
    if (value == null) return null;
    if (value is bool) return value;
    if (value is String) {
      return value.toLowerCase() == 'true';
    }
    if (value is int) return value != 0;
    return null;
  }

  List<T>? getList<T>(String key) {
    final value = this[key];
    if (value == null) return null;
    if (value is List) return value.whereType<T>().toList();
    return null;
  }

  Map<String, dynamic>? getMap(String key) {
    final value = this[key];
    if (value == null) return null;
    if (value is Map<String, dynamic>) return value;
    return null;
  }

  DateTime? getDateTime(String key) {
    final value = getString(key);
    if (value == null) return null;
    return DateTime.tryParse(value);
  }
}

// ===== Models with safe parsing =====

class Address {
  final String street;
  final String city;
  final String? state;
  final String country;
  final String? zip;

  Address({
    required this.street,
    required this.city,
    this.state,
    required this.country,
    this.zip,
  });

  factory Address.fromJson(Map<String, dynamic>? json) {
    if (json == null) {
      return Address(
        street: 'ไม่ระบุ',
        city: 'ไม่ระบุ',
        country: 'ไม่ระบุ',
      );
    }

    return Address(
      street: json.getStringOrDefault('street', 'ไม่ระบุ'),
      city: json.getStringOrDefault('city', 'ไม่ระบุ'),
      state: json.getString('state'),
      country: json.getStringOrDefault('country', 'ไทย'),
      zip: json.getString('zip'),
    );
  }

  @override
  String toString() => '$street, $city${state != null ? ', $state' : ''}, $country';
}

class UserData {
  final int id;
  final String name;
  final String? email;
  final int age;
  final bool isActive;
  final Address address;
  final List<String> tags;
  final DateTime? createdAt;

  UserData({
    required this.id,
    required this.name,
    this.email,
    required this.age,
    required this.isActive,
    required this.address,
    required this.tags,
    this.createdAt,
  });

  factory UserData.fromJson(Map<String, dynamic> json) {
    return UserData(
      id: json.getIntOrDefault('id'),
      name: json.getStringOrDefault('name', 'ไม่ระบุชื่อ'),
      email: json.getString('email'),
      age: json.getIntOrDefault('age', 0),
      isActive: json.getBool('is_active') ?? true,
      address: Address.fromJson(json.getMap('address')),
      tags: json.getList<String>('tags') ?? [],
      createdAt: json.getDateTime('created_at'),
    );
  }

  // toJson สำหรับ serialization
  Map<String, dynamic> toJson() => {
    'id': id,
    'name': name,
    if (email != null) 'email': email,
    'age': age,
    'is_active': isActive,
    'address': {
      'street': address.street,
      'city': address.city,
      if (address.state != null) 'state': address.state,
      'country': address.country,
      if (address.zip != null) 'zip': address.zip,
    },
    'tags': tags,
    if (createdAt != null) 'created_at': createdAt!.toIso8601String(),
  };

  @override
  String toString() =>
      'UserData(id: $id, name: $name, age: $age, email: ${email ?? "N/A"})';
}

// ===== Safe Collection Parsing =====

List<T> safeParseList<T>(
  dynamic raw,
  T? Function(dynamic) parser,
) {
  if (raw == null) return [];
  if (raw is! List) return [];
  return raw
      .map(parser)
      .whereType<T>()
      .toList();
}

UserData? safeParseUser(dynamic raw) {
  if (raw == null) return null;
  if (raw is! Map<String, dynamic>) return null;
  try {
    return UserData.fromJson(raw);
  } catch (e) {
    print('Warning: ไม่สามารถ parse user: $e');
    return null;
  }
}

// ===== Parser with Result =====

sealed class ParseResult<T> {}

class ParseOk<T> extends ParseResult<T> {
  final T value;
  ParseOk(this.value);
}

class ParseError<T> extends ParseResult<T> {
  final String message;
  final List<String> fieldErrors;
  ParseError(this.message, {this.fieldErrors = const []});
}

ParseResult<UserData> parseUserWithValidation(Map<String, dynamic> json) {
  final errors = <String>[];

  final id = json.getInt('id');
  if (id == null || id <= 0) {
    errors.add('id ต้องเป็นตัวเลขบวก');
  }

  final name = json.getString('name');
  if (name == null || name.isEmpty) {
    errors.add('name ต้องไม่ว่างเปล่า');
  }

  final age = json.getInt('age');
  if (age != null && (age < 0 || age > 150)) {
    errors.add('age ไม่อยู่ในช่วงที่เหมาะสม');
  }

  if (errors.isNotEmpty) {
    return ParseError('ข้อมูลไม่ถูกต้อง', fieldErrors: errors);
  }

  try {
    return ParseOk(UserData.fromJson(json));
  } catch (e) {
    return ParseError('เกิดข้อผิดพลาดในการ parse: $e');
  }
}

// ===== Workshop Main =====

void main() {
  print('=== Workshop: Safe Data Parsing ===\n');

  // Test 1: สมบูรณ์
  print('1. Parse ข้อมูลสมบูรณ์:');
  final completeJson = {
    'id': 1,
    'name': 'สมชาย ใจดี',
    'email': 'somchai@example.com',
    'age': 30,
    'is_active': true,
    'address': {
      'street': '123 ถ.สุขุมวิท',
      'city': 'กรุงเทพฯ',
      'country': 'ไทย',
      'zip': '10110',
    },
    'tags': ['vip', 'premium'],
    'created_at': '2024-01-15T08:30:00.000',
  };

  final user1 = UserData.fromJson(completeJson);
  print('  $user1');
  print('  ที่อยู่: ${user1.address}');
  print('  Tags: ${user1.tags}');
  print('  สร้างเมื่อ: ${user1.createdAt}');

  // Test 2: ข้อมูลไม่สมบูรณ์
  print('\n2. Parse ข้อมูลไม่สมบูรณ์:');
  final incompleteJson = {
    'id': '2', // string แทน int
    'name': 'สมหญิง',
    // ไม่มี email, age, address
  };

  final user2 = UserData.fromJson(incompleteJson);
  print('  $user2');
  print('  อายุ: ${user2.age} (default)');
  print('  Active: ${user2.isActive}');

  // Test 3: Parse list ของ users
  print('\n3. Parse list ของ users:');
  final rawList = [
    {'id': 1, 'name': 'อลิส'},
    {'id': 2, 'name': 'บ็อบ', 'age': 25},
    null,
    'invalid',
    {'id': 4, 'name': 'เดฟ'},
  ];

  final users = safeParseList(rawList, safeParseUser);
  print('  โหลดได้ ${users.length}/${rawList.length} users');
  users.forEach((u) => print('  - $u'));

  // Test 4: Validation
  print('\n4. Parse พร้อม Validation:');
  final testCases = [
    {'id': 5, 'name': 'อีฟ', 'age': 28},
    {'id': -1, 'name': '', 'age': 200},
    {'name': 'แฟรงก์'},
  ];

  for (final testCase in testCases) {
    switch (parseUserWithValidation(testCase)) {
      case ParseOk(:final value):
        print('  ✅ ${value.name}');
      case ParseError(:final message, :final fieldErrors):
        print('  ❌ $message: ${fieldErrors.join(', ')}');
    }
  }

  // Test 5: JSON string parsing
  print('\n5. Parse JSON string:');
  const jsonString = '''
  {
    "id": 10,
    "name": "จอห์น",
    "email": "john@example.com",
    "age": 35,
    "tags": ["developer", "flutter"]
  }
  ''';

  try {
    final json = jsonDecode(jsonString) as Map<String, dynamic>;
    final user = UserData.fromJson(json);
    print('  $user');
    print('  Tags: ${user.tags}');
  } catch (e) {
    print('  ❌ Parse error: $e');
  }

  print('\n=== เสร็จสิ้น Workshop ===');
}
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Nullable vs Non-nullable** - `String` vs `String?` และ null-aware operators
2. **Late variables** - ใช้เมื่อต้องการ initialize ทีหลัง
3. **Null assertions** - `!` operator ใช้ด้วยความระวัง
4. **Migration patterns** - วิธีแปลงโค้ดเก่าให้ null-safe
5. **Best practices** - ออกแบบ API ที่ null-safe

### Quick Reference

```dart
// ? ใน type declaration
String? name;           // nullable

// ? ใน method calls
name?.toUpperCase();    // null-safe call

// ?? null coalescing
name ?? 'default';      // ค่า default ถ้า null

// ??= null-aware assignment
name ??= 'value';       // กำหนดเฉพาะถ้า null

// ! null assertion
name!.length;           // assert ว่าไม่ null

// late
late String value;      // กำหนดทีหลัง

// required
void foo({required String x}); // ต้องส่งค่า
```

### Golden Rules

1. ใช้ nullable เฉพาะเมื่อ "ไม่มีค่า" มีความหมาย
2. ไม่ใช้ `!` ถ้าไม่แน่ใจ 100%
3. ใช้ `late` เฉพาะเมื่อรู้ว่าจะ initialize ก่อนใช้แน่ๆ
4. prefer `??` แทน `if-null` เมื่อต้องการ default value
5. ออกแบบ API ให้ nullable ชัดเจนจาก signature
