# Part 09: Mixins and Interfaces

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ `mixin` keyword เพื่อ reuse code ระหว่าง classes ที่ไม่เกี่ยวข้องกัน
- เข้าใจความแตกต่างระหว่าง `extends`, `implements`, และ `with`
- ใช้ multiple mixins ใน class เดียว
- สร้าง interface pattern ใน Dart
- กำหนด abstract methods ใน mixin
- สร้าง Animal behaviors ด้วย mixins เป็น workshop

---

## 9.1 ปัญหาที่ Mixin แก้ไข

ลองนึกภาพปัญหาคลาสสิกนี้:

```dart
// ปัญหา: สัตว์บางชนิดบิน บางชนิดว่ายน้ำ บางชนิดทำได้ทั้งสอง
// Inheritance ธรรมดาทำได้แค่ extends class เดียว

// ❌ แนวทางที่ไม่ดี - code duplication
class FlyingAnimal {
  void fly() => print('บินอยู่');
}

class SwimmingAnimal {
  void swim() => print('ว่ายน้ำอยู่');
}

// Duck ต้องบินได้และว่ายน้ำได้ แต่ extends ได้แค่อย่างเดียว!
// class Duck extends FlyingAnimal, SwimmingAnimal {} // ERROR!

// ✅ ใช้ Mixin แทน
mixin CanFly {
  void fly() {
    print('$runtimeType กำลังบิน');
  }
  
  double get flightSpeed => 50.0; // km/h
}

mixin CanSwim {
  void swim() {
    print('$runtimeType กำลังว่ายน้ำ');
  }
  
  double get swimmingSpeed => 10.0; // km/h
}

mixin CanWalk {
  void walk() {
    print('$runtimeType กำลังเดิน');
  }
  
  int get walkingSpeed => 5; // km/h
}

class Animal {
  String name;
  Animal(this.name);
  
  @override
  String toString() => name;
}

// Duck บินได้ ว่ายน้ำได้ เดินได้
class Duck extends Animal with CanFly, CanSwim, CanWalk {
  Duck(super.name);
}

// Eagle บินได้ เดินได้ แต่ว่ายน้ำไม่ได้
class Eagle extends Animal with CanFly, CanWalk {
  Eagle(super.name);
}

// Fish ว่ายน้ำได้เท่านั้น
class Fish extends Animal with CanSwim {
  Fish(super.name);
}

// Penguin เดินได้ ว่ายน้ำได้ แต่บินไม่ได้
class Penguin extends Animal with CanSwim, CanWalk {
  Penguin(super.name);
}

void main() {
  Duck duck = Duck('เป็ดน้อย');
  duck.fly();
  duck.swim();
  duck.walk();
  print('${duck.name}: บิน ${duck.flightSpeed}km/h, ว่าย ${duck.swimmingSpeed}km/h');
  
  Eagle eagle = Eagle('นกอินทรี');
  eagle.fly();
  eagle.walk();
  // eagle.swim(); // ERROR! Eagle ไม่มี CanSwim
  
  Penguin penguin = Penguin('เพนกวิน');
  penguin.swim();
  penguin.walk();
  // penguin.fly(); // ERROR!
  
  // Type checking กับ mixin
  print('\nType checks:');
  print(duck is CanFly);    // true
  print(duck is CanSwim);   // true
  print(eagle is CanSwim);  // false
  print(penguin is CanFly); // false
}
```

---

## 9.2 Mixin ที่ซับซ้อนขึ้น

```dart
mixin Logging {
  final List<String> _logs = [];
  
  void log(String message) {
    String timestamp = DateTime.now().toString().substring(0, 19);
    String entry = '[$timestamp] ${runtimeType}: $message';
    _logs.add(entry);
    print(entry);
  }
  
  void logWarning(String message) => log('⚠️ WARNING: $message');
  void logError(String message) => log('❌ ERROR: $message');
  void logSuccess(String message) => log('✅ SUCCESS: $message');
  
  List<String> get logs => List.unmodifiable(_logs);
  
  void printAllLogs() {
    print('\n=== Log History (${_logs.length} entries) ===');
    _logs.forEach(print);
  }
  
  void clearLogs() => _logs.clear();
}

mixin Serializable {
  // Abstract - class ต้อง implement
  Map<String, dynamic> toJson();
  
  String toJsonString() {
    Map<String, dynamic> json = toJson();
    StringBuffer buffer = StringBuffer('{');
    bool first = true;
    json.forEach((key, value) {
      if (!first) buffer.write(', ');
      buffer.write('"$key": ');
      if (value is String) {
        buffer.write('"$value"');
      } else {
        buffer.write(value.toString());
      }
      first = false;
    });
    buffer.write('}');
    return buffer.toString();
  }
}

mixin Cacheable {
  static final Map<String, dynamic> _cache = {};
  
  String get cacheKey;
  
  void saveToCache() {
    _cache[cacheKey] = toJson(); // ต้องใช้กับ Serializable
  }
  
  // ต้องประกาศ toJson ด้วย เพื่อใช้กับ saveToCache
  Map<String, dynamic> toJson();
  
  static dynamic getFromCache(String key) => _cache[key];
  static void clearCache() => _cache.clear();
  static bool hasCache(String key) => _cache.containsKey(key);
}

class UserProfile with Logging, Serializable, Cacheable {
  final String id;
  String name;
  String email;
  int age;
  List<String> preferences;
  
  UserProfile({
    required this.id,
    required this.name,
    required this.email,
    required this.age,
    this.preferences = const [],
  }) {
    log('UserProfile สร้างแล้ว: $id');
  }
  
  @override
  String get cacheKey => 'user_$id';
  
  @override
  Map<String, dynamic> toJson() => {
    'id': id,
    'name': name,
    'email': email,
    'age': age,
    'preferences': preferences.join(','),
  };
  
  void updateEmail(String newEmail) {
    if (!newEmail.contains('@')) {
      logError('Email ไม่ถูกต้อง: $newEmail');
      return;
    }
    String old = email;
    email = newEmail;
    log('เปลี่ยน email จาก $old เป็น $newEmail');
    logSuccess('อัปเดต email แล้ว');
    saveToCache();
  }
  
  void addPreference(String pref) {
    if (!preferences.contains(pref)) {
      preferences = [...preferences, pref];
      log('เพิ่ม preference: $pref');
    } else {
      logWarning('มี preference "$pref" อยู่แล้ว');
    }
  }
}

void main() {
  print('=== UserProfile with Mixins ===\n');
  
  UserProfile user = UserProfile(
    id: 'U001',
    name: 'สมชาย ใจดี',
    email: 'somchai@example.com',
    age: 28,
    preferences: ['ดาร์ค', 'มือถือ'],
  );
  
  user.saveToCache();
  user.updateEmail('somchai.new@example.com');
  user.updateEmail('invalid-email'); // จะ log error
  user.addPreference('แท็บเล็ต');
  user.addPreference('มือถือ'); // ซ้ำ จะ warn
  
  print('\nJSON:');
  print(user.toJsonString());
  
  user.printAllLogs();
  
  // ตรวจ cache
  print('\nCache:');
  print('มี cache U001? ${Cacheable.hasCache("user_U001")}');
  print('Cache data: ${Cacheable.getFromCache("user_U001")}');
}
```

---

## 9.3 extends vs implements vs with

```dart
// === extends ===
// - สืบทอด implementation ทั้งหมด
// - ใช้ได้กับ 1 class เท่านั้น
// - subclass เป็น subtype ของ superclass

class Vehicle {
  String brand;
  int year;
  
  Vehicle(this.brand, this.year);
  
  void start() => print('$brand สตาร์ท');
  void stop() => print('$brand หยุด');
  
  String get info => '$year $brand';
}

class Car extends Vehicle {
  int doors;
  
  Car(super.brand, super.year, this.doors);
  
  @override
  void start() {
    super.start();
    print('  (พร้อมขับ $doors ประตู)');
  }
}

// === implements ===
// - ต้อง implement ทุก method/getter เอง (ไม่รับ implementation)
// - ใช้ได้หลาย interfaces
// - เหมาะสำหรับ "สัญญา" ว่าจะมี interface นี้

abstract class Printable {
  void print_();
  String get printContent;
}

abstract class Exportable {
  void exportToCsv(String filename);
  void exportToJson(String filename);
}

abstract class Searchable {
  bool matches(String query);
  double relevanceScore(String query);
}

class Report implements Printable, Exportable, Searchable {
  final String title;
  final String content;
  final DateTime createdAt;
  
  Report(this.title, this.content) : createdAt = DateTime.now();
  
  @override
  void print_() {
    print('=== $title ===');
    print(content);
    print('สร้างเมื่อ: ${createdAt.toLocal()}');
  }
  
  @override
  String get printContent => '$title\n$content';
  
  @override
  void exportToCsv(String filename) {
    print('Export "$title" ไปยัง $filename.csv');
  }
  
  @override
  void exportToJson(String filename) {
    print('Export "$title" ไปยัง $filename.json');
  }
  
  @override
  bool matches(String query) {
    return title.toLowerCase().contains(query.toLowerCase()) ||
           content.toLowerCase().contains(query.toLowerCase());
  }
  
  @override
  double relevanceScore(String query) {
    int titleMatches = RegExp(query, caseSensitive: false).allMatches(title).length;
    int contentMatches = RegExp(query, caseSensitive: false).allMatches(content).length;
    return (titleMatches * 2.0 + contentMatches) / (title.length + content.length) * 100;
  }
}

// === with (Mixin) ===
// - รับ implementation จาก mixin
// - ใช้ได้หลาย mixins
// - ไม่เป็น inheritance แต่เป็น composition

mixin Timestamped {
  DateTime get createdAt => DateTime.now();
  DateTime? updatedAt;
  
  void touch() {
    (this as dynamic).updatedAt = DateTime.now();
  }
  
  String get age {
    Duration diff = DateTime.now().difference(createdAt);
    if (diff.inDays > 0) return '${diff.inDays} วัน';
    if (diff.inHours > 0) return '${diff.inHours} ชั่วโมง';
    if (diff.inMinutes > 0) return '${diff.inMinutes} นาที';
    return 'เพิ่งสร้าง';
  }
}

mixin Auditable {
  final List<String> _auditTrail = [];
  
  void audit(String action, String performedBy) {
    String entry = '${DateTime.now().toIso8601String()}: $action โดย $performedBy';
    _auditTrail.add(entry);
  }
  
  List<String> get auditTrail => List.unmodifiable(_auditTrail);
  
  void printAuditTrail() {
    print('=== Audit Trail ===');
    _auditTrail.forEach(print);
  }
}

class Document with Timestamped, Auditable {
  String title;
  String content;
  String author;
  
  Document({
    required this.title,
    required this.content,
    required this.author,
  }) {
    audit('สร้างเอกสาร "$title"', author);
  }
  
  void edit(String newContent, String editor) {
    content = newContent;
    touch();
    audit('แก้ไขเอกสาร "$title"', editor);
  }
  
  void share(String sharedWith, String sharedBy) {
    audit('แชร์เอกสาร "$title" ให้ $sharedWith', sharedBy);
  }
}

void main() {
  print('=== ทดสอบ extends ===');
  Car car = Car('Honda', 2023, 4);
  car.start();
  print(car.info);
  
  print('\n=== ทดสอบ implements ===');
  Report report = Report(
    'รายงานยอดขายเดือนมกราคม',
    'ยอดขายรวม ฿1,250,000 เพิ่มขึ้น 15% จากเดือนก่อน',
  );
  
  report.print_();
  report.exportToCsv('jan_sales');
  print('ค้นหา "ยอดขาย" พบหรือไม่: ${report.matches("ยอดขาย")}');
  
  print('\n=== ทดสอบ with ===');
  Document doc = Document(
    title: 'นโยบายบริษัท 2024',
    content: 'นโยบายบริษัท...',
    author: 'สมชาย',
  );
  
  doc.edit('นโยบายบริษัท ปรับปรุงแล้ว...', 'มานี');
  doc.share('ทีม HR', 'สมชาย');
  doc.printAuditTrail();
  
  print('\n=== ผสมทั้งสาม ===');
  // class ที่ใช้ทั้ง extends, implements, with
}

// ตัวอย่าง: Class ที่ใช้ทั้งสามในเวลาเดียวกัน
class DatabaseRecord extends Vehicle implements Searchable with Logging {
  String tableName;
  Map<String, dynamic> data;
  
  DatabaseRecord({
    required this.tableName,
    required this.data,
    required String brand,
    required int year,
  }) : super(brand, year);
  
  // ต้อง implement Searchable
  @override
  bool matches(String query) {
    return data.values.any((v) => v.toString().contains(query));
  }
  
  @override
  double relevanceScore(String query) {
    int matches = data.values.where((v) => v.toString().contains(query)).length;
    return matches / data.length * 100;
  }
  
  // override Vehicle.start ได้
  @override
  void start() {
    log('สตาร์ท Vehicle: $brand');
    super.start();
  }
}
```

---

## 9.4 Mixin Constraints - on keyword

```dart
// บังคับว่า mixin นี้ใช้ได้กับ class ที่ extends Serializable เท่านั้น

abstract class Serializable {
  Map<String, dynamic> toMap();
  
  factory Serializable.fromMap(Map<String, dynamic> map) {
    throw UnimplementedError();
  }
}

// on keyword - ต้องใช้กับ Serializable หรือ subclass เท่านั้น
mixin DatabasePersistence on Serializable {
  static final Map<String, Map<String, dynamic>> _database = {};
  
  String get primaryKey;
  String get tableName;
  
  void save() {
    _database[tableName] ??= {};
    _database[tableName]![primaryKey] = toMap(); // เรียก toMap() จาก Serializable
    print('บันทึก $tableName[$primaryKey] แล้ว');
  }
  
  static Map<String, dynamic>? find(String table, String key) {
    return _database[table]?[key];
  }
  
  static List<Map<String, dynamic>> findAll(String table) {
    return _database[table]?.values.toList() ?? [];
  }
  
  void delete() {
    _database[tableName]?.remove(primaryKey);
    print('ลบ $tableName[$primaryKey] แล้ว');
  }
  
  bool exists() {
    return _database[tableName]?.containsKey(primaryKey) ?? false;
  }
}

mixin Validatable on Serializable {
  List<String> validate();
  
  bool get isValid => validate().isEmpty;
  
  void validateOrThrow() {
    List<String> errors = validate();
    if (errors.isNotEmpty) {
      throw ValidationException(errors);
    }
  }
  
  // ใช้ toMap() จาก Serializable เพื่อ validation
  bool hasRequiredFields(List<String> required) {
    Map<String, dynamic> data = toMap();
    return required.every((field) => data.containsKey(field) && data[field] != null);
  }
}

class ValidationException implements Exception {
  final List<String> errors;
  ValidationException(this.errors);
  
  @override
  String toString() => 'ValidationException: ${errors.join(", ")}';
}

// ProductModel ต้อง extend Serializable เพื่อใช้ mixin
class ProductModel extends Serializable with DatabasePersistence, Validatable {
  final String id;
  String name;
  double price;
  int stock;
  String category;
  
  ProductModel({
    required this.id,
    required this.name,
    required this.price,
    required this.stock,
    required this.category,
  });
  
  @override
  String get primaryKey => id;
  
  @override
  String get tableName => 'products';
  
  @override
  Map<String, dynamic> toMap() => {
    'id': id,
    'name': name,
    'price': price,
    'stock': stock,
    'category': category,
  };
  
  @override
  List<String> validate() {
    List<String> errors = [];
    if (name.isEmpty) errors.add('ชื่อสินค้าต้องไม่ว่าง');
    if (price < 0) errors.add('ราคาต้องไม่ติดลบ');
    if (stock < 0) errors.add('สต็อกต้องไม่ติดลบ');
    if (category.isEmpty) errors.add('หมวดหมู่ต้องไม่ว่าง');
    return errors;
  }
  
  @override
  String toString() => 'Product($id, $name, ฿$price, สต็อก: $stock)';
}

void main() {
  print('=== ทดสอบ Mixin Constraints ===\n');
  
  ProductModel p1 = ProductModel(
    id: 'P001',
    name: 'กาแฟอาราบิก้า',
    price: 180.0,
    stock: 50,
    category: 'เครื่องดื่ม',
  );
  
  // Validate ก่อนบันทึก
  print('Valid? ${p1.isValid}');
  print('Errors: ${p1.validate()}');
  
  // บันทึก
  p1.save();
  
  // ค้นหา
  var found = DatabasePersistence.find('products', 'P001');
  print('พบ: $found');
  
  // ทดสอบ invalid
  ProductModel p2 = ProductModel(
    id: 'P002',
    name: '',  // invalid
    price: -10, // invalid
    stock: 20,
    category: 'ขนม',
  );
  
  print('\nProduct ไม่ valid:');
  p2.validate().forEach((e) => print('  - $e'));
  
  try {
    p2.validateOrThrow();
  } catch (e) {
    print('Exception: $e');
  }
  
  // บันทึกหลายรายการ
  List<ProductModel> products = [
    ProductModel(id: 'P003', name: 'ชาเขียว', price: 45.0, stock: 100, category: 'เครื่องดื่ม'),
    ProductModel(id: 'P004', name: 'น้ำส้ม', price: 35.0, stock: 80, category: 'เครื่องดื่ม'),
  ];
  
  for (var p in products) {
    if (p.isValid) p.save();
  }
  
  // ดูทั้งหมด
  print('\nสินค้าทั้งหมดในฐานข้อมูล:');
  DatabasePersistence.findAll('products').forEach((p) {
    print('  ${p['id']}: ${p['name']} ฿${p['price']}');
  });
}
```

---

## 9.5 Interface Pattern ใน Dart

Dart ไม่มี `interface` keyword แต่ใช้ abstract class แทน

```dart
// Interface คือ abstract class ที่ไม่มี implementation
// (หรือมีแค่ default implementation บางส่วน)

// 1. Pure Interface - ไม่มี implementation เลย
abstract class Repository<T, ID> {
  Future<T?> findById(ID id);
  Future<List<T>> findAll();
  Future<T> save(T entity);
  Future<void> deleteById(ID id);
  Future<int> count();
}

// 2. Interface กับ default methods
abstract class Cacheable<T> {
  String getCacheKey(String id);
  
  // Default implementation
  bool shouldCache(T item) => true;
  Duration get cacheDuration => Duration(minutes: 30);
}

// 3. Marker Interface - บอกว่า class มี capability นี้
abstract class Auditable {
  String get auditId;
  DateTime get createdAt;
  String get createdBy;
}

abstract class Deletable {
  bool get isDeleted;
  DateTime? get deletedAt;
  void softDelete();
  void restore();
}

// Model ที่ implements หลาย interfaces
class Product implements Auditable, Deletable {
  final String id;
  String name;
  double price;
  
  @override
  final String auditId;
  
  @override
  final DateTime createdAt;
  
  @override
  final String createdBy;
  
  @override
  bool _isDeleted = false;
  
  @override
  DateTime? _deletedAt;
  
  Product({
    required this.id,
    required this.name,
    required this.price,
    required this.createdBy,
  }) : auditId = 'AUD_$id',
       createdAt = DateTime.now();
  
  @override
  bool get isDeleted => _isDeleted;
  
  @override
  DateTime? get deletedAt => _deletedAt;
  
  @override
  void softDelete() {
    _isDeleted = true;
    _deletedAt = DateTime.now();
    print('Product $id ถูก soft delete แล้ว');
  }
  
  @override
  void restore() {
    _isDeleted = false;
    _deletedAt = null;
    print('Product $id ถูกกู้คืนแล้ว');
  }
  
  @override
  String toString() => 'Product($id, $name, ฿$price${_isDeleted ? " [DELETED]" : ""})';
}

// Repository Implementation
class InMemoryProductRepository implements Repository<Product, String> {
  final Map<String, Product> _store = {};
  
  @override
  Future<Product?> findById(String id) async {
    return _store[id];
  }
  
  @override
  Future<List<Product>> findAll() async {
    return _store.values.where((p) => !p.isDeleted).toList();
  }
  
  @override
  Future<Product> save(Product entity) async {
    _store[entity.id] = entity;
    return entity;
  }
  
  @override
  Future<void> deleteById(String id) async {
    _store[id]?.softDelete();
  }
  
  @override
  Future<int> count() async {
    return _store.values.where((p) => !p.isDeleted).length;
  }
  
  // Additional methods
  Future<List<Product>> findByCategory(String category) async {
    return _store.values
        .where((p) => !p.isDeleted)
        .toList();
  }
}

void main() async {
  print('=== Interface Pattern ===\n');
  
  InMemoryProductRepository repo = InMemoryProductRepository();
  
  // สร้างสินค้า
  Product p1 = Product(id: 'P001', name: 'กาแฟ', price: 60.0, createdBy: 'admin');
  Product p2 = Product(id: 'P002', name: 'ชา', price: 45.0, createdBy: 'admin');
  Product p3 = Product(id: 'P003', name: 'น้ำเปล่า', price: 15.0, createdBy: 'staff');
  
  await repo.save(p1);
  await repo.save(p2);
  await repo.save(p3);
  
  print('จำนวนสินค้า: ${await repo.count()}');
  
  List<Product> all = await repo.findAll();
  print('ทั้งหมด:');
  all.forEach((p) => print('  $p'));
  
  // Soft delete
  await repo.deleteById('P002');
  print('\nหลัง delete P002:');
  print('จำนวนสินค้า: ${await repo.count()}');
  
  Product? found = await repo.findById('P002');
  print('พบ P002? ${found?.isDeleted}'); // ยังอยู่แต่ isDeleted = true
  
  // Restore
  found?.restore();
  print('จำนวนหลัง restore: ${await repo.count()}');
}
```

---

## 9.6 Workshop: Animal Behaviors with Mixins

```dart
// animal_behaviors.dart
// ระบบสัตว์ที่ใช้ Mixin อย่างเต็มรูปแบบ

import 'dart:math' as math;

// === Base Class ===
abstract class Animal {
  final String name;
  final String species;
  int age;
  double weight; // kg
  double energy; // 0-100
  
  Animal({
    required this.name,
    required this.species,
    required this.age,
    required this.weight,
    this.energy = 100,
  });
  
  // Abstract methods
  String get sound;
  String get diet; // 'herbivore', 'carnivore', 'omnivore'
  
  // Concrete methods
  void makeSound() => print('$name (${species}): $sound!');
  
  void eat(String food) {
    energy = (energy + 20).clamp(0, 100);
    print('$name กินอาหาร $food (พลังงาน: ${energy.toStringAsFixed(0)}%)');
  }
  
  void rest() {
    energy = (energy + 30).clamp(0, 100);
    print('$name พักผ่อน (พลังงาน: ${energy.toStringAsFixed(0)}%)');
  }
  
  void useEnergy(double amount) {
    energy = (energy - amount).clamp(0, 100);
  }
  
  @override
  String toString() => '$name ($species, อายุ $age ปี, ${weight}kg)';
}

// === Mixins สำหรับ Behaviors ===

mixin CanFly on Animal {
  double altitude = 0; // เมตร
  double maxAltitude = 1000;
  double flyingSpeed = 50; // km/h
  
  void fly({double targetAltitude = 100}) {
    if (energy < 10) {
      print('$name เหนื่อยเกินไปจะบิน (พลังงาน: ${energy.toStringAsFixed(0)}%)');
      return;
    }
    altitude = targetAltitude.clamp(0, maxAltitude);
    double energyCost = targetAltitude * 0.1;
    useEnergy(energyCost);
    print('$name บินขึ้นที่ความสูง ${altitude.toStringAsFixed(0)} เมตร');
  }
  
  void land() {
    altitude = 0;
    print('$name ร่อนลงจอด');
  }
  
  void soar() {
    if (altitude > 0) {
      print('$name บินวนเวียนบนท้องฟ้า ที่ความสูง ${altitude.toStringAsFixed(0)}m');
    } else {
      print('$name ต้องบินขึ้นก่อน');
    }
  }
  
  bool get isFlying => altitude > 0;
}

mixin CanSwim on Animal {
  double depth = 0; // เมตร (ความลึก)
  double maxDepth = 50;
  double swimmingSpeed = 5; // km/h
  bool isUnderwater = false;
  
  void swim({bool goDeep = false}) {
    if (energy < 5) {
      print('$name เหนื่อยเกินไปจะว่ายน้ำ');
      return;
    }
    double speed = goDeep ? swimmingSpeed * 0.7 : swimmingSpeed;
    useEnergy(5);
    print('$name ว่ายน้ำที่ความเร็ว ${speed.toStringAsFixed(1)} km/h');
    if (goDeep) {
      isUnderwater = true;
      depth = (depth + 5).clamp(0, maxDepth);
      print('  ดำลึก ${depth.toStringAsFixed(1)} เมตร');
    }
  }
  
  void surfaceFromWater() {
    isUnderwater = false;
    depth = 0;
    print('$name โผล่ขึ้นมาผิวน้ำ');
  }
  
  void dive(double targetDepth) {
    if (energy < 15) {
      print('$name ไม่มีพลังงานพอจะดำน้ำ');
      return;
    }
    depth = targetDepth.clamp(0, maxDepth);
    isUnderwater = true;
    useEnergy(15);
    print('$name ดำลงไป ${depth.toStringAsFixed(1)} เมตร');
  }
}

mixin CanClimb on Animal {
  double climbingHeight = 0; // เมตร
  double climbingSpeed = 2; // m/s
  
  void climbUp(double meters) {
    if (energy < 10) {
      print('$name เหนื่อยเกินไปจะปีน');
      return;
    }
    climbingHeight += meters;
    useEnergy(10);
    print('$name ปีนขึ้นไป ${meters}m (ความสูงรวม: ${climbingHeight}m)');
  }
  
  void climbDown() {
    print('$name ไต่ลงมาจากความสูง ${climbingHeight}m');
    climbingHeight = 0;
  }
}

mixin CanHunt on Animal {
  int killCount = 0;
  double huntingSuccess = 0.7; // 70% success rate
  
  bool hunt(String prey) {
    if (energy < 20) {
      print('$name เหนื่อยเกินไปจะล่าเหยื่อ');
      return false;
    }
    
    useEnergy(20);
    bool success = math.Random().nextDouble() < huntingSuccess;
    
    if (success) {
      killCount++;
      energy = (energy + 40).clamp(0, 100);
      print('$name ล่า $prey สำเร็จ! (ล่าสำเร็จทั้งหมด: $killCount ครั้ง)');
      return true;
    } else {
      print('$name ล่า $prey ล้มเหลว');
      return false;
    }
  }
  
  void ambush(String prey) {
    print('$name ซุ่มโจมตี $prey...');
    huntingSuccess = 0.9; // สูงขึ้นเมื่อซุ่มโจมตี
    hunt(prey);
    huntingSuccess = 0.7; // reset
  }
}

mixin CanHibernate on Animal {
  bool isHibernating = false;
  DateTime? hibernationStart;
  
  void startHibernate() {
    isHibernating = true;
    hibernationStart = DateTime.now();
    energy = 100; // เติมพลังงานก่อนนอน
    print('$name เริ่มจำศีล... zzz');
  }
  
  void wakeUp() {
    isHibernating = false;
    energy = 50; // ตื่นมาพลังงานปานกลาง
    Duration? duration = hibernationStart != null 
        ? DateTime.now().difference(hibernationStart!)
        : null;
    print('$name ตื่นจากการจำศีล${duration != null ? " (นาน ${duration.inSeconds} วินาที)" : ""}');
    hibernationStart = null;
  }
}

mixin HasPack on Animal {
  List<Animal> packMembers = [];
  Animal? packLeader;
  
  void joinPack(List<Animal> pack, Animal leader) {
    packMembers = pack;
    packLeader = leader;
    print('$name เข้าร่วมฝูง นำโดย ${leader.name}');
  }
  
  void leavePack() {
    print('$name แยกออกจากฝูง');
    packMembers = [];
    packLeader = null;
  }
  
  void howlTopack() {
    if (packMembers.isNotEmpty) {
      print('$name ร้องเรียกสมาชิกฝูง:');
      for (Animal member in packMembers) {
        if (member != this) {
          member.makeSound();
        }
      }
    }
  }
}

// === Concrete Animals ===

class Eagle extends Animal with CanFly, CanHunt {
  Eagle({
    required super.name,
    super.age = 5,
    super.weight = 4.5,
  }) : super(species: 'นกอินทรีหัวขาว');
  
  Eagle._create({
    required super.name,
    required super.age,
    required super.weight,
  }) : super(species: 'นกอินทรีหัวขาว');
  
  @override
  String get sound => 'กรี๊ด';
  
  @override
  String get diet => 'carnivore';
  
  @override
  double get flyingSpeed => 80; // เร็วกว่าค่า default
  
  @override
  double get maxAltitude => 3000;
  
  void diveBomb(String prey) {
    if (!isFlying) {
      print('ต้องบินก่อนถึงจะ dive bomb ได้');
      return;
    }
    print('$name ดิ่งลงโจมตี $prey ด้วยความเร็วสูง!');
    hunt(prey);
    land();
  }
}

class Duck extends Animal with CanFly, CanSwim {
  Duck({
    required super.name,
    super.age = 2,
    super.weight = 1.5,
  }) : super(species: 'เป็ดแมลลาร์ด');
  
  @override
  String get sound => 'กาก กาก';
  
  @override
  String get diet => 'omnivore';
  
  @override
  double get flyingSpeed => 45;
  
  @override
  double get swimmingSpeed => 3;
  
  void quackAtPeople() {
    print('$name ขอขนมคนเดินผ่านไป!');
    makeSound();
  }
}

class Wolf extends Animal with CanHunt, HasPack {
  Wolf({
    required super.name,
    super.age = 4,
    super.weight = 40,
  }) : super(species: 'หมาป่า');
  
  @override
  String get sound => 'โหวลล';
  
  @override
  String get diet => 'carnivore';
  
  @override
  double get huntingSuccess => 0.85; // หมาป่าล่าเป็นฝูงได้ดีกว่า
  
  void howl() {
    print('$name โหยหวน...');
    if (packMembers.isNotEmpty) {
      howlTopack();
    }
  }
}

class Bear extends Animal with CanSwim, CanClimb, CanHibernate, CanHunt {
  Bear({
    required super.name,
    super.age = 8,
    super.weight = 200,
  }) : super(species: 'หมีสีน้ำตาล');
  
  @override
  String get sound => 'กร๊วด';
  
  @override
  String get diet => 'omnivore';
  
  void fishForSalmon() {
    if (!isHibernating) {
      print('$name ยืนริมแม่น้ำรอจับปลาแซลมอน...');
      hunt('ปลาแซลมอน');
    } else {
      print('$name กำลังจำศีลอยู่ ปลุกไม่ขึ้น');
    }
  }
}

class Penguin extends Animal with CanSwim {
  Penguin({
    required super.name,
    super.age = 3,
    super.weight = 10,
  }) : super(species: 'เพนกวิน');
  
  @override
  String get sound => 'แอ็กแอ็ก';
  
  @override
  String get diet => 'carnivore';
  
  @override
  double get swimmingSpeed => 25; // เร็วมากในน้ำ
  
  @override
  double get maxDepth => 500; // ดำได้ลึกมาก
  
  void waddle() {
    print('$name เดินแบบเพนกวิน ตุ๊บป่อง ตุ๊บป่อง');
  }
  
  void slideOnBelly() {
    print('$name ไถตัวบนน้ำแข็ง!');
    useEnergy(5);
  }
}

// === Wildlife Sanctuary ===
class WildlifeSanctuary {
  final String name;
  final List<Animal> animals = [];
  
  WildlifeSanctuary(this.name);
  
  void addAnimal(Animal animal) {
    animals.add(animal);
    print('$name: รับสัตว์ใหม่ - ${animal.name} (${animal.species})');
  }
  
  void feedAll(String food) {
    print('\n=== เวลาให้อาหาร ===');
    for (Animal animal in animals) {
      animal.eat(food);
    }
  }
  
  void morningRoutine() {
    print('\n=== กิจวัตรเช้า ===');
    for (Animal animal in animals) {
      animal.makeSound();
      
      if (animal is CanFly) {
        (animal as CanFly).fly(targetAltitude: 50);
      }
      
      if (animal is CanSwim) {
        (animal as CanSwim).swim();
      }
      
      if (animal is CanClimb) {
        (animal as CanClimb).climbUp(5);
      }
    }
  }
  
  void printStatus() {
    print('\n=== สถานะ ${name} ===');
    print('จำนวนสัตว์: ${animals.length} ตัว');
    
    Map<String, int> dietStats = {};
    for (Animal a in animals) {
      dietStats[a.diet] = (dietStats[a.diet] ?? 0) + 1;
    }
    
    print('\nอาหาร:');
    dietStats.forEach((diet, count) {
      String thaiDiet = diet == 'carnivore' ? 'กินเนื้อ' : 
                        diet == 'herbivore' ? 'กินพืช' : 'กินทั้งสอง';
      print('  $thaiDiet: $count ตัว');
    });
    
    print('\nความสามารถพิเศษ:');
    int canFlyCount = animals.whereType<CanFly>().length;
    int canSwimCount = animals.whereType<CanSwim>().length;
    print('  บินได้: $canFlyCount ตัว');
    print('  ว่ายน้ำได้: $canSwimCount ตัว');
    
    print('\nพลังงานเฉลี่ย:');
    double avgEnergy = animals.fold(0.0, (sum, a) => sum + a.energy) / animals.length;
    print('  ${avgEnergy.toStringAsFixed(1)}%');
    
    print('\nรายละเอียด:');
    for (Animal a in animals) {
      List<String> abilities = [];
      if (a is CanFly) abilities.add('บิน');
      if (a is CanSwim) abilities.add('ว่ายน้ำ');
      if (a is CanHunt) abilities.add('ล่าเหยื่อ');
      if (a is CanClimb) abilities.add('ปีน');
      if (a is CanHibernate) abilities.add('จำศีล');
      
      String abilityStr = abilities.isEmpty ? '-' : abilities.join(', ');
      print('  ${a.name}: พลังงาน ${a.energy.toStringAsFixed(0)}% | ความสามารถ: $abilityStr');
    }
  }
}

void main() {
  print('=== Wildlife Sanctuary ===\n');
  
  WildlifeSanctuary sanctuary = WildlifeSanctuary('สวนสัตว์เขาดิน');
  
  // สร้างสัตว์
  Eagle eagle = Eagle(name: 'อินทรีทอง', age: 8, weight: 5.0);
  Duck duck1 = Duck(name: 'เป็ดเหลือง', age: 2, weight: 1.2);
  Duck duck2 = Duck(name: 'เป็ดขาว', age: 3, weight: 1.5);
  Wolf alpha = Wolf(name: 'หมาป่าอัลฟ่า', age: 6, weight: 45);
  Wolf beta = Wolf(name: 'หมาป่าเบต้า', age: 4, weight: 38);
  Wolf gamma = Wolf(name: 'หมาป่าแกมมา', age: 3, weight: 35);
  Bear bear = Bear(name: 'หมีใหญ่', age: 12, weight: 220);
  Penguin penguin = Penguin(name: 'เพนกวินจุ๊งหนิง', age: 5, weight: 12);
  
  // ให้หมาป่าอยู่รวมกันเป็นฝูง
  beta.joinPack([alpha, beta, gamma], alpha);
  gamma.joinPack([alpha, beta, gamma], alpha);
  
  // เพิ่มเข้า sanctuary
  [eagle, duck1, duck2, alpha, beta, gamma, bear, penguin].forEach(sanctuary.addAnimal);
  
  // กิจวัตร
  sanctuary.feedAll('อาหารสัตว์');
  sanctuary.morningRoutine();
  
  // Activities เฉพาะ
  print('\n=== กิจกรรมพิเศษ ===');
  
  eagle.fly(targetAltitude: 500);
  eagle.diveBomb('กระต่าย');
  eagle.land();
  
  duck1.swim();
  duck1.fly(targetAltitude: 30);
  duck1.land();
  duck1.quackAtPeople();
  
  alpha.howl();
  alpha.hunt('กวาง');
  
  bear.swim();
  bear.fishForSalmon();
  bear.climbUp(10);
  bear.startHibernate();
  bear.wakeUp();
  
  penguin.swim(goDeep: true);
  penguin.dive(100);
  penguin.surfaceFromWater();
  penguin.waddle();
  penguin.slideOnBelly();
  
  // สรุปสถานะ
  sanctuary.printStatus();
}
```

---

## 9.7 Mixin กับ Flutter (ตัวอย่างจริง)

```dart
// ตัวอย่าง Mixin ที่ใช้บ่อยใน Flutter
// (conceptual - ไม่ต้อง run, เพื่อการศึกษา)

/*
// ใน Flutter, State classes ใช้ mixin อย่างกว้างขวาง

class MyScreen extends StatefulWidget {
  @override
  State<MyScreen> createState() => _MyScreenState();
}

// ใช้ mixin TickerProviderStateMixin สำหรับ Animation
class _MyScreenState extends State<MyScreen>
    with TickerProviderStateMixin {
  
  late AnimationController _controller;
  
  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this, // vsync มาจาก TickerProviderStateMixin
      duration: Duration(seconds: 1),
    );
  }
  
  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
  
  @override
  Widget build(BuildContext context) {
    return Container();
  }
}

// Custom Mixin สำหรับ loading state ที่ใช้ซ้ำได้
mixin LoadingMixin<T extends StatefulWidget> on State<T> {
  bool _isLoading = false;
  
  bool get isLoading => _isLoading;
  
  void startLoading() {
    if (mounted) {
      setState(() => _isLoading = true);
    }
  }
  
  void stopLoading() {
    if (mounted) {
      setState(() => _isLoading = false);
    }
  }
  
  Future<void> runWithLoading(Future<void> Function() action) async {
    startLoading();
    try {
      await action();
    } finally {
      stopLoading();
    }
  }
}

// Mixin สำหรับ form validation
mixin FormValidationMixin<T extends StatefulWidget> on State<T> {
  final formKey = GlobalKey<FormState>();
  
  bool validateForm() {
    return formKey.currentState?.validate() ?? false;
  }
  
  void saveForm() {
    formKey.currentState?.save();
  }
  
  String? validateRequired(String? value, String fieldName) {
    if (value == null || value.trim().isEmpty) {
      return '$fieldName ต้องไม่ว่าง';
    }
    return null;
  }
  
  String? validateEmail(String? value) {
    if (value == null || value.isEmpty) return 'กรุณากรอก Email';
    if (!RegExp(r'^[\w-\.]+@[\w-]+\.\w{2,}$').hasMatch(value)) {
      return 'Email ไม่ถูกต้อง';
    }
    return null;
  }
  
  String? validatePhone(String? value) {
    if (value == null || value.isEmpty) return 'กรุณากรอกเบอร์โทร';
    if (!RegExp(r'^0[0-9]{9}$').hasMatch(value.replaceAll('-', ''))) {
      return 'เบอร์โทรไม่ถูกต้อง (ต้องเป็น 10 หลัก)';
    }
    return null;
  }
}

// ใช้ทั้งสอง mixin พร้อมกัน
class ProfileFormScreen extends StatefulWidget {
  @override
  State<ProfileFormScreen> createState() => _ProfileFormScreenState();
}

class _ProfileFormScreenState extends State<ProfileFormScreen>
    with LoadingMixin, FormValidationMixin {
  
  String _name = '';
  String _email = '';
  
  Future<void> _submitForm() async {
    if (!validateForm()) return;
    
    await runWithLoading(() async {
      // เรียก API
      await Future.delayed(Duration(seconds: 2));
      print('บันทึกข้อมูลแล้ว: $_name, $_email');
    });
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Form(
        key: formKey,
        child: Column(
          children: [
            if (isLoading) CircularProgressIndicator(),
            // ... form fields
          ],
        ),
      ),
    );
  }
}
*/

// Dart-only Mixin Pattern (รัน Terminal ได้)
mixin Disposable {
  bool _isDisposed = false;
  
  bool get isDisposed => _isDisposed;
  
  void checkNotDisposed() {
    if (_isDisposed) throw StateError('Object ถูก dispose แล้ว');
  }
  
  void dispose() {
    if (!_isDisposed) {
      onDispose();
      _isDisposed = true;
      print('${runtimeType} ถูก dispose แล้ว');
    }
  }
  
  // Subclass สามารถ override เพื่อ cleanup resource
  void onDispose() {}
}

mixin Observable<T> {
  final List<void Function(T)> _listeners = [];
  
  void addListener(void Function(T) listener) {
    _listeners.add(listener);
  }
  
  void removeListener(void Function(T) listener) {
    _listeners.remove(listener);
  }
  
  void notifyListeners(T value) {
    for (var listener in [..._listeners]) {
      listener(value);
    }
  }
  
  int get listenerCount => _listeners.length;
}

class StockPrice with Disposable, Observable<double> {
  final String symbol;
  double _price;
  
  StockPrice(this.symbol, double initialPrice) : _price = initialPrice;
  
  double get price => _price;
  
  set price(double newPrice) {
    checkNotDisposed();
    if (newPrice != _price) {
      _price = newPrice;
      notifyListeners(newPrice);
    }
  }
  
  @override
  void onDispose() {
    print('ยกเลิกการติดตามราคา $symbol');
  }
}

void main() {
  print('=== Observable + Disposable Pattern ===\n');
  
  StockPrice stock = StockPrice('PTT', 35.0);
  
  // เพิ่ม listeners
  stock.addListener((price) => print('  Listener 1: ราคา PTT อัปเดต -> ฿$price'));
  stock.addListener((price) {
    if (price > 40) print('  Listener 2: แจ้งเตือน! ราคาสูงกว่า ฿40');
    if (price < 30) print('  Listener 2: แจ้งเตือน! ราคาต่ำกว่า ฿30');
  });
  
  print('Listeners: ${stock.listenerCount}');
  
  // อัปเดตราคา
  print('\nอัปเดตราคา:');
  stock.price = 37.5;
  stock.price = 42.0;
  stock.price = 28.5;
  stock.price = 35.0; // กลับมาเท่าเดิม - ไม่มี notification
  
  print('\nDispose:');
  stock.dispose();
  
  try {
    stock.price = 40.0; // จะ throw error
  } catch (e) {
    print('Error: $e');
  }
}
```

---

## สรุป Part 09

ใน Part นี้เราได้เรียนรู้:

### extends vs implements vs with:
| Feature | extends | implements | with |
|---------|---------|-----------|------|
| รับ implementation | ✅ ทั้งหมด | ❌ ต้อง implement เอง | ✅ ทั้งหมด |
| จำนวนที่ใช้ได้ | 1 | หลาย | หลาย |
| เป็น subtype | ✅ | ✅ | ✅ |
| ใช้เมื่อ | "is-a" | "can-do" contract | Code reuse |

### Mixin Rules:
1. ใช้ `mixin` keyword ประกาศ
2. ไม่สามารถ instantiate ได้โดยตรง
3. ใช้ `on` keyword บังคับ base class
4. MRO (Method Resolution Order): class → mixins (ขวาไปซ้าย) → base

### Use Cases:
- **CanFly, CanSwim**: ความสามารถที่แยกจาก inheritance
- **Logging, Caching**: Cross-cutting concerns
- **Disposable, Observable**: Resource management patterns
- **Flutter**: TickerProviderStateMixin, LoadingMixin, etc.

## ➡️ Part ถัดไป

**Part 10: Generics** - เราจะเรียนรู้ Generic classes, Generic methods, Type constraints และวิธีสร้าง reusable data structures
