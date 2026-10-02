# Part 16: Functional Programming ใน Dart

## บทนำ

Functional Programming (FP) คือรูปแบบการเขียนโปรแกรมที่เน้นการใช้ฟังก์ชันเป็นหน่วยพื้นฐาน หลีกเลี่ยง side effects และ state ที่เปลี่ยนแปลงได้ Dart ไม่ใช่ภาษา FP แบบ pure แต่รองรับ FP concepts หลายอย่าง

## 16.1 Pure Functions

ฟังก์ชัน pure คือฟังก์ชันที่:
1. ให้ผลลัพธ์เดิมเสมอสำหรับ input เดิม
2. ไม่มี side effects

```dart
// Pure function
int add(int a, int b) => a + b;
String greet(String name) => 'สวัสดี, $name!';
bool isEven(int n) => n % 2 == 0;

// Impure function (มี side effects)
var counter = 0;
void incrementCounter() {
  counter++; // ปรับเปลี่ยน global state = side effect
}

DateTime getTime() => DateTime.now(); // ผลลัพธ์ต่างกันทุกครั้ง = impure

// การแปลง impure เป็น pure
// ส่ง state ผ่าน parameter แทน
int increment(int current) => current + 1;

// ส่ง dependency ผ่าน parameter
String formatDate(DateTime date) =>
    '${date.year}-${date.month.toString().padLeft(2, '0')}-${date.day.toString().padLeft(2, '0')}';

// ตัวอย่าง: คำนวณภาษี (pure)
double calculateTax(double income, double rate) => income * rate;

double calculateNetIncome(double gross, double taxRate) =>
    gross - calculateTax(gross, taxRate);

void testPureFunctions() {
  print('=== Pure Functions ===');

  // ผลลัพธ์เหมือนกันทุกครั้ง
  print(add(3, 4));     // 7
  print(add(3, 4));     // 7 เสมอ
  print(isEven(6));     // true
  print(isEven(6));     // true เสมอ

  // คำนวณภาษี
  final income = 100000.0;
  final taxRate = 0.17;
  print('รายได้สุทธิ: ${calculateNetIncome(income, taxRate)}');
}
```

### Higher-Order Functions

```dart
// ฟังก์ชันที่รับ function เป็น parameter
List<T> filter<T>(List<T> list, bool Function(T) predicate) {
  return list.where(predicate).toList();
}

List<R> transform<T, R>(List<T> list, R Function(T) mapper) {
  return list.map(mapper).toList();
}

T reduce<T>(List<T> list, T Function(T, T) combiner) {
  return list.reduce(combiner);
}

// ฟังก์ชันที่ return function
bool Function(int) greaterThan(int threshold) =>
    (n) => n > threshold;

bool Function(String) startsWith(String prefix) =>
    (s) => s.startsWith(prefix);

void testHigherOrder() {
  print('\n=== Higher-Order Functions ===');

  var numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

  // filter, transform, reduce
  var evens = filter(numbers, isEven);
  var doubled = transform(evens, (n) => n * 2);
  var sum = reduce(doubled, (a, b) => a + b);

  print('เลขคู่ทวีคูณรวม: $sum');

  // ฟังก์ชันที่ return function
  var isAdult = greaterThan(17);
  var ages = [15, 18, 21, 13, 25];
  var adults = filter(ages, isAdult);
  print('ผู้ใหญ่: $adults');

  var names = ['อลิส', 'บ็อบ', 'อนัน', 'เบล', 'อาเล็กซ์'];
  var aNames = filter(names, startsWith('อ'));
  print('ชื่อขึ้นต้นด้วย อ: $aNames');
}
```

## 16.2 Immutability

```dart
// Immutable data structures
class ImmutablePoint {
  final double x;
  final double y;

  const ImmutablePoint(this.x, this.y);

  // แทนที่จะแก้ไข ให้สร้าง copy ใหม่
  ImmutablePoint translate(double dx, double dy) =>
      ImmutablePoint(x + dx, y + dy);

  ImmutablePoint scale(double factor) =>
      ImmutablePoint(x * factor, y * factor);

  @override
  String toString() => 'Point($x, $y)';

  @override
  bool operator ==(Object other) =>
      other is ImmutablePoint && x == other.x && y == other.y;

  @override
  int get hashCode => Object.hash(x, y);
}

// Immutable list operations
extension ImmutableList<T> on List<T> {
  List<T> append(T item) => [...this, item];
  List<T> prepend(T item) => [item, ...this];
  List<T> removeAt(int index) => [
    ...sublist(0, index),
    ...sublist(index + 1),
  ];
  List<T> replace(int index, T item) => [
    ...sublist(0, index),
    item,
    ...sublist(index + 1),
  ];
}

// Immutable state management
class AppState {
  final List<String> todos;
  final bool isLoading;
  final String? error;

  const AppState({
    this.todos = const [],
    this.isLoading = false,
    this.error,
  });

  AppState copyWith({
    List<String>? todos,
    bool? isLoading,
    String? error,
    bool clearError = false,
  }) {
    return AppState(
      todos: todos ?? this.todos,
      isLoading: isLoading ?? this.isLoading,
      error: clearError ? null : (error ?? this.error),
    );
  }

  @override
  String toString() =>
      'AppState(todos: ${todos.length}, loading: $isLoading, error: $error)';
}

void testImmutability() {
  print('\n=== Immutability ===');

  // Immutable point
  final point = ImmutablePoint(1.0, 2.0);
  final moved = point.translate(3.0, 4.0);
  print('ต้นทาง: $point');
  print('เคลื่อนที่: $moved');
  print('ต้นทางยังเหมือนเดิม: $point');

  // Immutable list
  final list = [1, 2, 3];
  final withFour = list.append(4);
  final withZero = list.prepend(0);
  print('เดิม: $list');
  print('เพิ่ม 4: $withFour');
  print('เพิ่ม 0 ข้างหน้า: $withZero');
  print('เดิมยังเหมือนกัน: $list');

  // State transitions
  var state = const AppState();
  state = state.copyWith(isLoading: true);
  state = state.copyWith(
    todos: state.todos.append('ซื้อของ'),
    isLoading: false,
  );
  state = state.copyWith(todos: state.todos.append('ออกกำลังกาย'));
  print('State: $state');
}
```

## 16.3 map/filter/reduce Patterns

```dart
class Product {
  final String name;
  final String category;
  final double price;
  final int stock;

  const Product({
    required this.name,
    required this.category,
    required this.price,
    required this.stock,
  });

  @override
  String toString() => '$name (฿${price.toStringAsFixed(0)})';
}

final products = [
  Product(name: 'Flutter Book', category: 'books', price: 350, stock: 15),
  Product(name: 'Dart Guide', category: 'books', price: 280, stock: 8),
  Product(name: 'Keyboard', category: 'electronics', price: 1200, stock: 5),
  Product(name: 'Mouse', category: 'electronics', price: 500, stock: 0),
  Product(name: 'Notebook', category: 'stationery', price: 50, stock: 100),
  Product(name: 'Pen Set', category: 'stationery', price: 120, stock: 50),
  Product(name: 'Headphones', category: 'electronics', price: 2500, stock: 3),
];

void demonstrateCollectionOps() {
  print('\n=== map/filter/reduce ===');

  // map: แปลงข้อมูล
  var productNames = products.map((p) => p.name).toList();
  print('ชื่อสินค้า: $productNames');

  var discountedPrices = products.map((p) => ({
    'name': p.name,
    'originalPrice': p.price,
    'discountedPrice': p.price * 0.9,
  })).toList();

  // where (filter): กรอง
  var inStock = products.where((p) => p.stock > 0).toList();
  print('มีสินค้า: ${inStock.length}/${products.length}');

  var electronics = products
      .where((p) => p.category == 'electronics')
      .where((p) => p.stock > 0)
      .toList();
  print('Electronics มีของ: $electronics');

  // reduce/fold: รวมค่า
  var totalValue = products
      .where((p) => p.stock > 0)
      .fold(0.0, (sum, p) => sum + (p.price * p.stock));
  print('มูลค่าสินค้าทั้งหมด: ฿${totalValue.toStringAsFixed(0)}');

  // ราคาเฉลี่ย
  var avgPrice = products.map((p) => p.price).reduce((a, b) => a + b)
      / products.length;
  print('ราคาเฉลี่ย: ฿${avgPrice.toStringAsFixed(2)}');

  // groupBy (custom)
  var byCategory = products.fold<Map<String, List<Product>>>(
    {},
    (map, product) {
      map.putIfAbsent(product.category, () => []).add(product);
      return map;
    },
  );

  print('\nแบ่งตามหมวดหมู่:');
  byCategory.forEach((category, prods) {
    print('  $category: ${prods.map((p) => p.name).join(', ')}');
  });
}

// Chaining operations
void demonstrateChaining() {
  print('\n=== Method Chaining ===');

  // หาสินค้าที่แพงที่สุด 3 อันดับที่มีของอยู่
  var top3 = products
      .where((p) => p.stock > 0)
      .toList()
      ..sort((a, b) => b.price.compareTo(a.price));
  var top3Items = top3.take(3).toList();

  print('สินค้าราคาแพงสุด 3 อันดับ:');
  top3Items.asMap().forEach((i, p) =>
    print('  ${i + 1}. ${p.name}: ฿${p.price}'));

  // สถิติตามหมวดหมู่
  var stats = products
      .fold<Map<String, Map<String, dynamic>>>(
        {},
        (map, p) {
          final cat = map.putIfAbsent(p.category, () => {
            'count': 0,
            'totalPrice': 0.0,
            'totalStock': 0,
          });
          cat['count'] = (cat['count'] as int) + 1;
          cat['totalPrice'] = (cat['totalPrice'] as double) + p.price;
          cat['totalStock'] = (cat['totalStock'] as int) + p.stock;
          return map;
        },
      );

  print('\nสถิติตามหมวดหมู่:');
  stats.forEach((cat, data) {
    final count = data['count'] as int;
    final avgPrice = (data['totalPrice'] as double) / count;
    print('  $cat: $count สินค้า, เฉลี่ย ฿${avgPrice.toStringAsFixed(0)}, '
        'สต็อกรวม ${data['totalStock']}');
  });
}
```

## 16.4 Currying และ Partial Application

```dart
// Currying: แปลงฟังก์ชัน n args เป็น chain ของ 1-arg functions
// f(a, b, c) → f(a)(b)(c)

// Manual curry
int Function(int) add(int a) => (int b) => a + b;
int Function(double) multiply(double factor) => (int n) => (n * factor).toInt();

// Generic curry helper
R Function(B) curry2<A, B, R>(R Function(A, B) fn) {
  return (A a) => (B b) => fn(a, b);
}

// Partial application: กำหนดค่า argument บางตัวล่วงหน้า
double Function(double) calculateTaxWithRate(double rate) =>
    (double income) => income * rate;

double Function(double) calculateDiscount(double percent) =>
    (double price) => price * (1 - percent / 100);

void testCurrying() {
  print('\n=== Currying & Partial Application ===');

  // Using curried add
  var addFive = add(5);
  var addTen = add(10);

  print('addFive(3) = ${addFive(3)}');     // 8
  print('addTen(3) = ${addTen(3)}');       // 13
  print('addFive(10) = ${addFive(10)}');   // 15

  // Partial application กับภาษี
  var calculateVat = calculateTaxWithRate(0.07);      // VAT 7%
  var calculateIncomeTax = calculateTaxWithRate(0.17); // Income tax 17%

  print('\nภาษี:');
  print('VAT ของ 1000: ${calculateVat(1000)}');
  print('Income tax ของ 50000: ${calculateIncomeTax(50000)}');

  // discount functions
  var discount10 = calculateDiscount(10);
  var discount20 = calculateDiscount(20);
  var discount50 = calculateDiscount(50);

  var price = 1000.0;
  print('\nส่วนลด:');
  print('ราคาหลังลด 10%: ${discount10(price)}');
  print('ราคาหลังลด 20%: ${discount20(price)}');
  print('ราคาหลังลด 50%: ${discount50(price)}');

  // Compose functions
  var numbers = [100.0, 200.0, 500.0, 1000.0];
  var finalPrices = numbers
      .map(discount20)   // ลด 20% ก่อน
      .map(calculateVat) // คำนวณ VAT
      .toList();
  print('\nราคาสุดท้าย (ลด 20% + VAT 7%): $finalPrices');
}

// Function composition
typedef Transformer<T> = T Function(T);

Transformer<T> compose<T>(List<Transformer<T>> transformers) {
  return (T input) => transformers.fold(input, (value, fn) => fn(value));
}

Transformer<T> pipe<T>(Transformer<T> first, Transformer<T> second) =>
    (T input) => second(first(input));

void testComposition() {
  print('\n=== Function Composition ===');

  // String transformations
  Transformer<String> trim = (s) => s.trim();
  Transformer<String> toLowerCase = (s) => s.toLowerCase();
  Transformer<String> replaceSpaces = (s) => s.replaceAll(' ', '_');

  var normalize = compose([trim, toLowerCase, replaceSpaces]);

  var inputs = ['  Hello World  ', '  DART FLUTTER  ', ' Compose This '];
  var normalized = inputs.map(normalize).toList();
  print('Normalized: $normalized');

  // Numeric transformations
  Transformer<double> addVat = (p) => p * 1.07;
  Transformer<double> roundToInt = (p) => p.roundToDouble();
  Transformer<double> applyDiscount = (p) => p * 0.9;

  var priceProcessor = compose([applyDiscount, addVat, roundToInt]);

  var prices = [100.0, 250.0, 599.0];
  var finalPrices = prices.map(priceProcessor).toList();
  print('ราคาสุดท้าย: $finalPrices');
}
```

## 16.5 Monads: Option/Result Pattern

```dart
// Option/Maybe monad
sealed class Option<T> {
  const Option();

  bool get isSome => this is Some<T>;
  bool get isNone => this is None<T>;

  T? get value => isSome ? (this as Some<T>).value : null;

  Option<R> map<R>(R Function(T) fn) {
    return switch (this) {
      Some(:final value) => Some(fn(value)),
      None() => None<R>(),
    };
  }

  Option<R> flatMap<R>(Option<R> Function(T) fn) {
    return switch (this) {
      Some(:final value) => fn(value),
      None() => None<R>(),
    };
  }

  T getOrElse(T defaultValue) {
    return switch (this) {
      Some(:final value) => value,
      None() => defaultValue,
    };
  }

  T getOrCompute(T Function() compute) {
    return switch (this) {
      Some(:final value) => value,
      None() => compute(),
    };
  }

  Option<T> filter(bool Function(T) predicate) {
    return switch (this) {
      Some(:final value) when predicate(value) => this,
      _ => None<T>(),
    };
  }

  void ifSome(void Function(T) action) {
    if (this is Some<T>) action((this as Some<T>).value);
  }
}

class Some<T> extends Option<T> {
  @override
  final T value;
  const Some(this.value);
  @override
  String toString() => 'Some($value)';
}

class None<T> extends Option<T> {
  const None();
  @override
  String toString() => 'None';
}

// Result monad (สำหรับ error handling)
sealed class Result<T, E> {
  const Result();

  bool get isOk => this is Ok<T, E>;
  bool get isErr => this is Err<T, E>;

  Result<R, E> map<R>(R Function(T) fn) {
    return switch (this) {
      Ok(:final value) => Ok(fn(value)),
      Err(:final error) => Err(error),
    };
  }

  Result<R, E> flatMap<R>(Result<R, E> Function(T) fn) {
    return switch (this) {
      Ok(:final value) => fn(value),
      Err(:final error) => Err(error),
    };
  }

  Result<T, F> mapError<F>(F Function(E) fn) {
    return switch (this) {
      Ok(:final value) => Ok(value),
      Err(:final error) => Err(fn(error)),
    };
  }

  T getOrElse(T defaultValue) {
    return switch (this) {
      Ok(:final value) => value,
      Err() => defaultValue,
    };
  }

  void fold({
    required void Function(T) onOk,
    required void Function(E) onErr,
  }) {
    switch (this) {
      case Ok(:final value): onOk(value);
      case Err(:final error): onErr(error);
    }
  }
}

class Ok<T, E> extends Result<T, E> {
  final T value;
  const Ok(this.value);
  @override
  String toString() => 'Ok($value)';
}

class Err<T, E> extends Result<T, E> {
  final E error;
  const Err(this.error);
  @override
  String toString() => 'Err($error)';
}

// ตัวอย่างการใช้งาน
Option<int> safeDiv(int a, int b) {
  if (b == 0) return const None();
  return Some(a ~/ b);
}

Result<int, String> parseInt(String s) {
  final n = int.tryParse(s);
  if (n == null) return Err('ไม่สามารถแปลง "$s" เป็นตัวเลขได้');
  return Ok(n);
}

void testMonads() {
  print('\n=== Option/Result Monads ===');

  // Option chaining
  var result = safeDiv(10, 2)
      .map((n) => n * 3)
      .filter((n) => n > 10);

  print('10/2*3 (filter > 10): $result');

  var noResult = safeDiv(10, 0)
      .map((n) => n * 3);

  print('10/0*3: $noResult');

  // Option getOrElse
  var value = safeDiv(10, 0).getOrElse(0);
  print('10/0 หรือ 0: $value');

  // Result chaining
  var parsed = parseInt('42')
      .map((n) => n * 2)
      .map((n) => '84 = $n');

  print('\nParseInt("42"): $parsed');

  var failed = parseInt('abc')
      .map((n) => n * 2);

  print('ParseInt("abc"): $failed');

  // Result fold
  parseInt('hello').fold(
    onOk: (v) => print('ค่า: $v'),
    onErr: (e) => print('Error: $e'),
  );

  // Chaining Results
  var computation = parseInt('10')
      .flatMap((n) => n > 0 ? Ok(n * 100) : Err('ต้องเป็นจำนวนบวก'))
      .map((n) => 'ผลลัพธ์: $n');

  print(computation);
}
```

## 16.6 Workshop: Data Transformation Pipeline

```dart
import 'dart:convert';

// ===== Data Models =====

class RawSale {
  final String date;
  final String product;
  final String? quantity;
  final String? price;
  final String? region;

  RawSale({
    required this.date,
    required this.product,
    this.quantity,
    this.price,
    this.region,
  });

  factory RawSale.fromJson(Map<String, dynamic> json) => RawSale(
    date: json['date'] as String? ?? '',
    product: json['product'] as String? ?? '',
    quantity: json['quantity']?.toString(),
    price: json['price']?.toString(),
    region: json['region'] as String?,
  );
}

class Sale {
  final DateTime date;
  final String product;
  final int quantity;
  final double price;
  final String region;

  Sale({
    required this.date,
    required this.product,
    required this.quantity,
    required this.price,
    required this.region,
  });

  double get total => price * quantity;
  String get month => '${date.year}-${date.month.toString().padLeft(2, '0')}';
}

class SalesSummary {
  final String key;
  final int totalQuantity;
  final double totalRevenue;
  final int transactionCount;

  SalesSummary({
    required this.key,
    required this.totalQuantity,
    required this.totalRevenue,
    required this.transactionCount,
  });

  double get averageOrderValue => transactionCount > 0
      ? totalRevenue / transactionCount
      : 0.0;

  @override
  String toString() =>
      '$key: qty=$totalQuantity, revenue=฿${totalRevenue.toStringAsFixed(0)}, '
      'transactions=$transactionCount, avg=฿${averageOrderValue.toStringAsFixed(0)}';
}

// ===== Pipeline Components =====

// Step 1: Parse raw data
Result<Sale, String> parseSale(RawSale raw) {
  if (raw.date.isEmpty) return Err('วันที่ว่างเปล่า');

  final date = DateTime.tryParse(raw.date);
  if (date == null) return Err('รูปแบบวันที่ไม่ถูกต้อง: ${raw.date}');

  final quantity = int.tryParse(raw.quantity ?? '');
  if (quantity == null || quantity <= 0) {
    return Err('จำนวนไม่ถูกต้อง: ${raw.quantity}');
  }

  final price = double.tryParse(raw.price ?? '');
  if (price == null || price <= 0) {
    return Err('ราคาไม่ถูกต้อง: ${raw.price}');
  }

  return Ok(Sale(
    date: date,
    product: raw.product.trim(),
    quantity: quantity,
    price: price,
    region: raw.region?.trim() ?? 'ไม่ระบุ',
  ));
}

// Step 2: Filter
bool isValidSale(Sale sale) =>
    sale.product.isNotEmpty &&
    sale.quantity > 0 &&
    sale.price > 0;

bool isInDateRange(Sale sale, DateTime start, DateTime end) =>
    sale.date.isAfter(start.subtract(Duration(days: 1))) &&
    sale.date.isBefore(end.add(Duration(days: 1)));

// Step 3: Aggregate
SalesSummary aggregateSales(String key, List<Sale> sales) {
  return sales.fold(
    SalesSummary(
      key: key,
      totalQuantity: 0,
      totalRevenue: 0.0,
      transactionCount: 0,
    ),
    (summary, sale) => SalesSummary(
      key: summary.key,
      totalQuantity: summary.totalQuantity + sale.quantity,
      totalRevenue: summary.totalRevenue + sale.total,
      transactionCount: summary.transactionCount + 1,
    ),
  );
}

// ===== Pipeline Builder =====

class SalesPipeline {
  final List<RawSale> _rawData;

  SalesPipeline(this._rawData);

  // Parse and collect errors
  ({List<Sale> sales, List<String> errors}) parse() {
    final results = _rawData.map(parseSale).toList();

    final sales = results
        .whereType<Ok<Sale, String>>()
        .map((r) => r.value)
        .toList();

    final errors = results
        .whereType<Err<Sale, String>>()
        .map((r) => r.error)
        .toList();

    return (sales: sales, errors: errors);
  }

  List<Sale> parseAndFilter({
    DateTime? startDate,
    DateTime? endDate,
    List<String>? products,
    List<String>? regions,
  }) {
    var (:sales, :errors) = parse();

    if (errors.isNotEmpty) {
      print('คำเตือน: ข้ามข้อมูล ${errors.length} รายการที่ไม่ถูกต้อง');
    }

    return sales
        .where(isValidSale)
        .where((s) => startDate == null || isInDateRange(
          s, startDate, endDate ?? DateTime.now()))
        .where((s) => products == null || products.contains(s.product))
        .where((s) => regions == null || regions.contains(s.region))
        .toList();
  }

  Map<String, SalesSummary> summarizeBy(
    String Function(Sale) groupKey,
    List<Sale> sales,
  ) {
    final grouped = sales.fold<Map<String, List<Sale>>>(
      {},
      (map, sale) {
        map.putIfAbsent(groupKey(sale), () => []).add(sale);
        return map;
      },
    );

    return grouped.map((key, sales) =>
        MapEntry(key, aggregateSales(key, sales)));
  }
}

// ===== Workshop Main =====

void main() {
  print('=== Workshop: Data Transformation Pipeline ===\n');

  // สร้างข้อมูลทดสอบ
  final rawData = [
    RawSale(date: '2024-01-15', product: 'Laptop', quantity: '2', price: '25000', region: 'กรุงเทพฯ'),
    RawSale(date: '2024-01-16', product: 'Mouse', quantity: '5', price: '500', region: 'เชียงใหม่'),
    RawSale(date: '2024-01-17', product: 'Keyboard', quantity: '3', price: '1200', region: 'กรุงเทพฯ'),
    RawSale(date: '2024-01-20', product: 'Laptop', quantity: '1', price: '25000', region: 'ภูเก็ต'),
    RawSale(date: '2024-02-01', product: 'Mouse', quantity: '10', price: '500', region: 'กรุงเทพฯ'),
    RawSale(date: '2024-02-05', product: 'Monitor', quantity: '2', price: '8000', region: 'เชียงใหม่'),
    // ข้อมูลที่มีปัญหา
    RawSale(date: '', product: 'Invalid', quantity: '1', price: '100'),
    RawSale(date: '2024-01-bad', product: 'Bad Date', quantity: '1', price: '100'),
    RawSale(date: '2024-01-18', product: 'Zero Qty', quantity: '0', price: '100'),
    RawSale(date: '2024-01-19', product: 'No Price', quantity: '1', price: null),
  ];

  final pipeline = SalesPipeline(rawData);

  // Test 1: Parse กับ error collection
  print('1. Parse ข้อมูล:');
  var (:sales, :errors) = pipeline.parse();
  print('  สำเร็จ: ${sales.length}/${rawData.length}');
  print('  ข้อผิดพลาด: ${errors.length}');
  for (var e in errors) print('  - $e');

  // Test 2: Filter
  print('\n2. Filter ข้อมูล (มกราคม 2024):');
  final janSales = pipeline.parseAndFilter(
    startDate: DateTime(2024, 1, 1),
    endDate: DateTime(2024, 1, 31),
  );
  print('  ยอดขายเดือน ม.ค.: ${janSales.length} รายการ');

  // Test 3: สรุปตามสินค้า
  print('\n3. สรุปยอดขายตามสินค้า:');
  final allSales = pipeline.parseAndFilter();
  final byProduct = pipeline.summarizeBy((s) => s.product, allSales);
  byProduct.values.toList()
    ..sort((a, b) => b.totalRevenue.compareTo(a.totalRevenue))
    ..forEach(print);

  // Test 4: สรุปตามภูมิภาค
  print('\n4. สรุปยอดขายตามภูมิภาค:');
  final byRegion = pipeline.summarizeBy((s) => s.region, allSales);
  byRegion.values.toList()
    ..sort((a, b) => b.totalRevenue.compareTo(a.totalRevenue))
    ..forEach(print);

  // Test 5: Functional pipeline
  print('\n5. Functional Pipeline - Top 3 สินค้าแต่ละเดือน:');
  final byMonth = pipeline.summarizeBy((s) => s.month, allSales);

  final monthlySorted = byMonth.entries
      .map((e) => e.value)
      .toList()
    ..sort((a, b) => a.key.compareTo(b.key));

  for (final month in monthlySorted) {
    print('  ${month.key}: ฿${month.totalRevenue.toStringAsFixed(0)}');
  }

  // Test 6: Pure function composition
  print('\n6. คำนวณสถิติ:');
  final revenues = allSales.map((s) => s.total).toList();
  if (revenues.isNotEmpty) {
    final total = revenues.fold(0.0, (a, b) => a + b);
    final max = revenues.reduce((a, b) => a > b ? a : b);
    final min = revenues.reduce((a, b) => a < b ? a : b);
    final avg = total / revenues.length;

    print('  ยอดขายรวม: ฿${total.toStringAsFixed(0)}');
    print('  ยอดสูงสุด: ฿${max.toStringAsFixed(0)}');
    print('  ยอดต่ำสุด: ฿${min.toStringAsFixed(0)}');
    print('  ยอดเฉลี่ย: ฿${avg.toStringAsFixed(0)}');
  }

  print('\n=== เสร็จสิ้น Workshop ===');
}
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Pure functions** - ไม่มี side effects, ผลลัพธ์แน่นอน
2. **Immutability** - ข้อมูลไม่เปลี่ยน แต่สร้าง copy ใหม่
3. **map/filter/reduce** - patterns พื้นฐานของ FP
4. **Currying & Partial Application** - แปลง multi-arg functions
5. **Monads** - Option และ Result pattern
6. **Workshop** - Data transformation pipeline

### ประโยชน์ของ FP

- **Testability** - Pure functions ทดสอบง่าย
- **Predictability** - ผลลัพธ์แน่นอน ไม่ต้องกังวล state
- **Composability** - รวมฟังก์ชันเล็กๆ เป็น pipeline ใหญ่
- **Concurrency** - Immutable data ปลอดภัยกับ parallel operations

### เมื่อไหร่ควรใช้ FP

- Data transformation และ processing
- Business logic ที่ซับซ้อน
- เมื่อต้องการ testability สูง
- เมื่อทำงานกับ concurrent operations
