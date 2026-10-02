# Part 18: Testing ใน Dart

## บทนำ

การเขียน tests เป็นทักษะสำคัญที่ช่วยให้โค้ดมีคุณภาพ น่าเชื่อถือ และแก้ไขได้ง่าย Dart มี test package ที่ทรงพลังพร้อม mockito สำหรับ mocking

## 18.1 Unit Tests กับ test package

### ติดตั้ง test package

```yaml
# pubspec.yaml
dev_dependencies:
  test: ^1.24.0
  mockito: ^5.4.0
  build_runner: ^2.4.0
```

### โครงสร้าง Test

```dart
// test/calculator_test.dart
import 'package:test/test.dart';
import '../lib/calculator.dart'; // โค้ดที่จะทดสอบ

void main() {
  // group จัดกลุ่ม tests ที่เกี่ยวข้อง
  group('Calculator', () {
    late Calculator calc;

    // setUp ทำงานก่อน test แต่ละตัว
    setUp(() {
      calc = Calculator();
    });

    // tearDown ทำงานหลัง test แต่ละตัว
    tearDown(() {
      // ล้างทรัพยากร ถ้ามี
    });

    test('บวก 2 + 3 = 5', () {
      expect(calc.add(2, 3), equals(5));
    });

    test('ลบ 5 - 3 = 2', () {
      expect(calc.subtract(5, 3), equals(2));
    });

    test('คูณ 4 * 3 = 12', () {
      expect(calc.multiply(4, 3), equals(12));
    });

    group('หาร', () {
      test('หาร 10 / 2 = 5', () {
        expect(calc.divide(10, 2), equals(5.0));
      });

      test('หารด้วยศูนย์ต้อง throw', () {
        expect(
          () => calc.divide(10, 0),
          throwsA(isA<ArgumentError>()),
        );
      });

      test('หารด้วยศูนย์มีข้อความที่ถูกต้อง', () {
        expect(
          () => calc.divide(10, 0),
          throwsA(
            isA<ArgumentError>().having(
              (e) => e.message,
              'message',
              contains('ศูนย์'),
            ),
          ),
        );
      });
    });
  });
}

// Calculator class ที่จะทดสอบ
class Calculator {
  double add(double a, double b) => a + b;
  double subtract(double a, double b) => a - b;
  double multiply(double a, double b) => a * b;

  double divide(double a, double b) {
    if (b == 0) throw ArgumentError('ไม่สามารถหารด้วยศูนย์');
    return a / b;
  }

  double power(double base, int exp) {
    double result = 1;
    for (var i = 0; i < exp.abs(); i++) {
      result *= base;
    }
    return exp < 0 ? 1 / result : result;
  }

  List<int> factors(int n) {
    final factors = <int>[];
    for (var i = 1; i <= n; i++) {
      if (n % i == 0) factors.add(i);
    }
    return factors;
  }
}
```

## 18.2 expect(), group(), setUp()

### Matchers

```dart
import 'package:test/test.dart';

void main() {
  group('Matchers ต่างๆ', () {
    // === Equality ===
    test('equals', () {
      expect(2 + 2, equals(4));
      expect('hello', equals('hello'));
      expect([1, 2, 3], equals([1, 2, 3]));
    });

    test('isNot', () {
      expect(2 + 2, isNot(equals(5)));
      expect('hello', isNot(isEmpty));
    });

    // === Type checks ===
    test('isA / runtimeType', () {
      expect(42, isA<int>());
      expect('hello', isA<String>());
      expect(3.14, isA<double>());
    });

    // === Null checks ===
    test('isNull / isNotNull', () {
      String? nullStr;
      String? nonNull = 'hello';

      expect(nullStr, isNull);
      expect(nonNull, isNotNull);
    });

    // === Boolean ===
    test('isTrue / isFalse', () {
      expect(true, isTrue);
      expect(false, isFalse);
      expect(2 > 1, isTrue);
    });

    // === Numeric ===
    test('greaterThan / lessThan', () {
      expect(5, greaterThan(3));
      expect(3, lessThan(5));
      expect(5, greaterThanOrEqualTo(5));
      expect(5, inInclusiveRange(1, 10));
      expect(3.14, closeTo(3.14159, 0.01));
    });

    // === String ===
    test('string matchers', () {
      expect('hello world', contains('world'));
      expect('hello', startsWith('hel'));
      expect('hello', endsWith('llo'));
      expect('hello world', matches(RegExp(r'^hello\s\w+$')));
    });

    // === Collections ===
    test('collection matchers', () {
      var list = [1, 2, 3, 4, 5];
      expect(list, hasLength(5));
      expect(list, contains(3));
      expect(list, containsAll([1, 3, 5]));
      expect(list, everyElement(greaterThan(0)));
      expect(list, anyElement(equals(3)));
      expect(list, isNotEmpty);
      expect([], isEmpty);
    });

    // === Map ===
    test('map matchers', () {
      var map = {'a': 1, 'b': 2};
      expect(map, containsPair('a', 1));
      expect(map, containsKey('b'));
      expect(map, containsValue(2));
    });

    // === Exceptions ===
    test('throws matchers', () {
      expect(
        () => throw Exception('test'),
        throwsException,
      );
      expect(
        () => throw ArgumentError('bad arg'),
        throwsArgumentError,
      );
      expect(
        () => <int>[].first,
        throwsStateError,
      );
    });

    // === Async ===
    test('completion matchers', () async {
      expect(
        Future.value(42),
        completion(equals(42)),
      );
      expect(
        Future.error(Exception('fail')),
        throwsException,
      );
    });
  });
}
```

### setUp/tearDown patterns

```dart
void main() {
  // setUpAll/tearDownAll ทำงานแค่ครั้งเดียว
  setUpAll(() async {
    print('เริ่มต้น test suite');
    // เตรียม shared resources
  });

  tearDownAll(() async {
    print('สิ้นสุด test suite');
    // ล้าง shared resources
  });

  group('Database Tests', () {
    late FakeDatabase db;

    setUp(() {
      db = FakeDatabase();
      db.connect();
    });

    tearDown(() {
      db.disconnect();
    });

    test('insert และ find', () {
      db.insert({'id': 1, 'name': 'อลิส'});
      final found = db.find(1);
      expect(found, isNotNull);
      expect(found!['name'], equals('อลิส'));
    });

    test('update', () {
      db.insert({'id': 1, 'name': 'อลิส'});
      db.update(1, {'name': 'อลิส (updated)'});
      expect(db.find(1)!['name'], equals('อลิส (updated)'));
    });

    test('delete', () {
      db.insert({'id': 1, 'name': 'อลิส'});
      db.delete(1);
      expect(db.find(1), isNull);
    });
  });
}

class FakeDatabase {
  final Map<int, Map<String, dynamic>> _data = {};
  bool _connected = false;

  void connect() => _connected = true;
  void disconnect() => _connected = false;

  void insert(Map<String, dynamic> record) {
    _data[record['id'] as int] = record;
  }

  Map<String, dynamic>? find(int id) => _data[id];

  void update(int id, Map<String, dynamic> updates) {
    if (_data.containsKey(id)) {
      _data[id] = {..._data[id]!, ...updates};
    }
  }

  void delete(int id) => _data.remove(id);
}
```

### Parameterized Tests

```dart
void main() {
  // ทดสอบหลาย input พร้อมกัน
  group('isPrime', () {
    final primes = [2, 3, 5, 7, 11, 13, 17, 19];
    final nonPrimes = [1, 4, 6, 8, 9, 10, 12, 15];

    for (var n in primes) {
      test('$n เป็น prime', () {
        expect(isPrime(n), isTrue);
      });
    }

    for (var n in nonPrimes) {
      test('$n ไม่ใช่ prime', () {
        expect(isPrime(n), isFalse);
      });
    }
  });

  // หรือใช้ test.forEach
  for (final testCase in [
    (input: '2+2', expected: 4),
    (input: '10-3', expected: 7),
    (input: '4*5', expected: 20),
  ]) {
    test('evalulate("${testCase.input}") = ${testCase.expected}', () {
      // expect(evaluate(testCase.input), equals(testCase.expected));
    });
  }
}

bool isPrime(int n) {
  if (n < 2) return false;
  for (var i = 2; i <= n ~/ 2; i++) {
    if (n % i == 0) return false;
  }
  return true;
}
```

## 18.3 Mocking กับ Mockito

```dart
// ===== Interface =====
abstract class UserRepository {
  Future<User> findById(String id);
  Future<List<User>> findAll();
  Future<void> save(User user);
  Future<void> delete(String id);
}

abstract class EmailService {
  Future<void> sendEmail(String to, String subject, String body);
}

class User {
  final String id;
  final String name;
  final String email;
  User({required this.id, required this.name, required this.email});
}

// ===== Service ที่จะทดสอบ =====
class UserService {
  final UserRepository _repo;
  final EmailService _emailService;

  UserService(this._repo, this._emailService);

  Future<User?> getUser(String id) async {
    try {
      return await _repo.findById(id);
    } catch (e) {
      return null;
    }
  }

  Future<void> registerUser(String name, String email) async {
    final id = 'user_${DateTime.now().millisecondsSinceEpoch}';
    final user = User(id: id, name: name, email: email);

    await _repo.save(user);
    await _emailService.sendEmail(
      email,
      'ยินดีต้อนรับ',
      'สวัสดี $name ยินดีต้อนรับสู่ระบบ',
    );
  }

  Future<List<User>> searchUsers(String query) async {
    final all = await _repo.findAll();
    return all.where((u) =>
      u.name.contains(query) || u.email.contains(query)
    ).toList();
  }
}

// ===== Mock Classes (จะ generate โดย build_runner) =====
// @GenerateMocks([UserRepository, EmailService])
// import 'user_service_test.mocks.dart';

// ตัวอย่าง manual mock
class MockUserRepository implements UserRepository {
  final Map<String, User> _users = {};
  var findByIdCallCount = 0;
  var saveCallCount = 0;
  String? lastSavedUserId;

  @override
  Future<User> findById(String id) async {
    findByIdCallCount++;
    final user = _users[id];
    if (user == null) throw Exception('User not found: $id');
    return user;
  }

  @override
  Future<List<User>> findAll() async {
    return _users.values.toList();
  }

  @override
  Future<void> save(User user) async {
    saveCallCount++;
    lastSavedUserId = user.id;
    _users[user.id] = user;
  }

  @override
  Future<void> delete(String id) async {
    _users.remove(id);
  }

  void addUser(User user) => _users[user.id] = user;
}

class MockEmailService implements EmailService {
  final List<Map<String, String>> sentEmails = [];

  @override
  Future<void> sendEmail(String to, String subject, String body) async {
    sentEmails.add({'to': to, 'subject': subject, 'body': body});
  }

  bool wasSentTo(String email) => sentEmails.any((e) => e['to'] == email);
}

// ===== Test file =====
// test/user_service_test.dart
void main() {
  group('UserService', () {
    late MockUserRepository mockRepo;
    late MockEmailService mockEmail;
    late UserService service;

    setUp(() {
      mockRepo = MockUserRepository();
      mockEmail = MockEmailService();
      service = UserService(mockRepo, mockEmail);
    });

    group('getUser', () {
      test('return user เมื่อพบ', () async {
        final user = User(id: '1', name: 'อลิส', email: 'alice@test.com');
        mockRepo.addUser(user);

        final result = await service.getUser('1');

        expect(result, isNotNull);
        expect(result!.name, equals('อลิส'));
        expect(mockRepo.findByIdCallCount, equals(1));
      });

      test('return null เมื่อไม่พบ', () async {
        final result = await service.getUser('not_exist');
        expect(result, isNull);
      });
    });

    group('registerUser', () {
      test('บันทึกผู้ใช้และส่งอีเมล', () async {
        await service.registerUser('บ็อบ', 'bob@test.com');

        expect(mockRepo.saveCallCount, equals(1));
        expect(mockEmail.wasSentTo('bob@test.com'), isTrue);
      });

      test('อีเมลที่ส่งมีหัวข้อที่ถูกต้อง', () async {
        await service.registerUser('ชาร์ลี', 'charlie@test.com');

        final email = mockEmail.sentEmails.first;
        expect(email['subject'], contains('ยินดีต้อนรับ'));
        expect(email['body'], contains('ชาร์ลี'));
      });
    });

    group('searchUsers', () {
      setUp(() {
        mockRepo.addUser(User(id: '1', name: 'อลิส', email: 'alice@test.com'));
        mockRepo.addUser(User(id: '2', name: 'บ็อบ', email: 'bob@test.com'));
        mockRepo.addUser(User(id: '3', name: 'อลิซาเบธ', email: 'liz@test.com'));
      });

      test('หาด้วยชื่อ', () async {
        final results = await service.searchUsers('อลิ');
        expect(results, hasLength(2));
      });

      test('หาด้วยอีเมล', () async {
        final results = await service.searchUsers('bob@');
        expect(results, hasLength(1));
        expect(results.first.name, equals('บ็อบ'));
      });

      test('ไม่พบผลลัพธ์', () async {
        final results = await service.searchUsers('xyz');
        expect(results, isEmpty);
      });
    });
  });
}
```

## 18.4 Test Coverage

```bash
# รัน tests พร้อม coverage
dart test --coverage=coverage

# สร้าง lcov.info
dart pub global run coverage:format_coverage \
  --packages=.dart_tool/package_config.json \
  --report-on=lib \
  --lcov \
  -i coverage \
  -o coverage/lcov.info

# สร้าง HTML report (ต้องติดตั้ง lcov)
genhtml coverage/lcov.info -o coverage/html

# ดู coverage ใน terminal
dart pub global run coverage:format_coverage \
  --packages=.dart_tool/package_config.json \
  -i coverage \
  --report-on=lib
```

### Coverage-driven testing

```dart
// Calculator class
class ScientificCalculator {
  double add(double a, double b) => a + b;
  double subtract(double a, double b) => a - b;
  double multiply(double a, double b) => a * b;

  double divide(double a, double b) {
    if (b == 0) throw ArgumentError('หารด้วยศูนย์ไม่ได้');
    return a / b;
  }

  double sqrt(double n) {
    if (n < 0) throw ArgumentError('ไม่สามารถหารากที่สองของจำนวนลบได้');
    return _sqrt(n);
  }

  double _sqrt(double n) {
    // Newton's method
    var x = n;
    var y = 1.0;
    const epsilon = 0.000001;
    while (x - y > epsilon) {
      x = (x + y) / 2;
      y = n / x;
    }
    return x;
  }

  double power(double base, int exponent) {
    if (exponent == 0) return 1;
    if (exponent < 0) return 1 / power(base, -exponent);
    var result = 1.0;
    for (var i = 0; i < exponent; i++) {
      result *= base;
    }
    return result;
  }

  int factorial(int n) {
    if (n < 0) throw ArgumentError('ไม่มี factorial ของจำนวนลบ');
    if (n == 0 || n == 1) return 1;
    return n * factorial(n - 1);
  }
}

// Tests ที่ครอบคลุม coverage
void main() {
  group('ScientificCalculator', () {
    late ScientificCalculator calc;

    setUp(() => calc = ScientificCalculator());

    group('sqrt', () {
      test('sqrt(4) = 2', () {
        expect(calc.sqrt(4), closeTo(2.0, 0.0001));
      });

      test('sqrt(0) = 0', () {
        expect(calc.sqrt(0), closeTo(0.0, 0.0001));
      });

      test('sqrt(-1) throws', () {
        expect(() => calc.sqrt(-1), throwsArgumentError);
      });
    });

    group('power', () {
      test('3^2 = 9', () {
        expect(calc.power(3, 2), equals(9.0));
      });

      test('2^0 = 1', () {
        expect(calc.power(2, 0), equals(1.0));
      });

      test('2^-1 = 0.5', () {
        expect(calc.power(2, -1), closeTo(0.5, 0.0001));
      });
    });

    group('factorial', () {
      test('0! = 1', () {
        expect(calc.factorial(0), equals(1));
      });

      test('5! = 120', () {
        expect(calc.factorial(5), equals(120));
      });

      test('negative throws', () {
        expect(() => calc.factorial(-1), throwsArgumentError);
      });
    });
  });
}
```

## 18.5 TDD Approach

Test-Driven Development: เขียน test ก่อน แล้วค่อยเขียน code

```dart
// Step 1: เขียน test ก่อน (Red phase)
// test/shopping_cart_test.dart

void main() {
  group('ShoppingCart', () {
    late ShoppingCart cart;

    setUp(() => cart = ShoppingCart());

    test('ตะกร้าว่างเมื่อสร้างใหม่', () {
      expect(cart.isEmpty, isTrue);
      expect(cart.itemCount, equals(0));
      expect(cart.total, equals(0.0));
    });

    test('เพิ่มสินค้า', () {
      cart.addItem(CartItem(id: '1', name: 'Apple', price: 10.0, quantity: 2));

      expect(cart.itemCount, equals(1));
      expect(cart.total, equals(20.0));
      expect(cart.isEmpty, isFalse);
    });

    test('เพิ่มสินค้าเดิมเพิ่มจำนวน', () {
      cart.addItem(CartItem(id: '1', name: 'Apple', price: 10.0, quantity: 1));
      cart.addItem(CartItem(id: '1', name: 'Apple', price: 10.0, quantity: 2));

      expect(cart.itemCount, equals(1));
      expect(cart.getItem('1')!.quantity, equals(3));
      expect(cart.total, equals(30.0));
    });

    test('ลบสินค้า', () {
      cart.addItem(CartItem(id: '1', name: 'Apple', price: 10.0));
      cart.removeItem('1');

      expect(cart.isEmpty, isTrue);
    });

    test('ล้างตะกร้า', () {
      cart.addItem(CartItem(id: '1', name: 'Apple', price: 10.0));
      cart.addItem(CartItem(id: '2', name: 'Banana', price: 5.0));
      cart.clear();

      expect(cart.isEmpty, isTrue);
      expect(cart.total, equals(0.0));
    });

    test('คำนวณ total พร้อมส่วนลด', () {
      cart.addItem(CartItem(id: '1', name: 'Item', price: 100.0, quantity: 2));
      cart.applyDiscount(10); // 10%

      expect(cart.total, closeTo(180.0, 0.01));
    });

    test('จำนวนสินค้าต้องมากกว่า 0', () {
      expect(
        () => cart.addItem(CartItem(id: '1', name: 'Item', price: 10.0, quantity: 0)),
        throwsArgumentError,
      );
    });

    test('ราคาต้องมากกว่า 0', () {
      expect(
        () => cart.addItem(CartItem(id: '1', name: 'Item', price: -1.0)),
        throwsArgumentError,
      );
    });
  });
}

// Step 2: เขียน code ให้ผ่าน test (Green phase)
class CartItem {
  final String id;
  final String name;
  final double price;
  int quantity;

  CartItem({
    required this.id,
    required this.name,
    required this.price,
    this.quantity = 1,
  }) {
    if (price <= 0) throw ArgumentError('ราคาต้องมากกว่า 0');
    if (quantity <= 0) throw ArgumentError('จำนวนต้องมากกว่า 0');
  }

  double get subtotal => price * quantity;
}

class ShoppingCart {
  final Map<String, CartItem> _items = {};
  double _discountPercent = 0;

  bool get isEmpty => _items.isEmpty;
  int get itemCount => _items.length;

  double get total {
    final subtotal = _items.values.fold(0.0, (sum, item) => sum + item.subtotal);
    return subtotal * (1 - _discountPercent / 100);
  }

  void addItem(CartItem item) {
    if (_items.containsKey(item.id)) {
      _items[item.id]!.quantity += item.quantity;
    } else {
      _items[item.id] = item;
    }
  }

  CartItem? getItem(String id) => _items[id];

  void removeItem(String id) => _items.remove(id);

  void clear() {
    _items.clear();
    _discountPercent = 0;
  }

  void applyDiscount(double percent) {
    if (percent < 0 || percent > 100) {
      throw ArgumentError('ส่วนลดต้องอยู่ระหว่าง 0-100');
    }
    _discountPercent = percent;
  }
}

// Step 3: Refactor (Refactor phase) - ปรับปรุง code ให้ดีขึ้น
// (tests ยังผ่านทุกตัว)
```

## 18.6 Workshop: Test the Calculator from Part 03

ทดสอบ Calculator จาก Part 03 อย่างครบถ้วน

```dart
// lib/calculator.dart (สมมติว่ามีอยู่แล้ว)
import 'dart:math' as math;

class Calculator {
  double _memory = 0;
  final List<String> _history = [];

  // Operations
  double add(double a, double b) => _recordOp('$a + $b', a + b);
  double subtract(double a, double b) => _recordOp('$a - $b', a - b);
  double multiply(double a, double b) => _recordOp('$a * $b', a * b);

  double divide(double a, double b) {
    if (b == 0) throw ArgumentError('ไม่สามารถหารด้วยศูนย์ได้');
    return _recordOp('$a / $b', a / b);
  }

  double sqrt(double n) {
    if (n < 0) throw ArgumentError('ไม่สามารถหารากที่สองของจำนวนลบ: $n');
    return _recordOp('√$n', math.sqrt(n));
  }

  double percentage(double value, double percent) {
    return _recordOp('$value * $percent%', value * percent / 100);
  }

  // Memory
  void memoryStore(double value) => _memory = value;
  double memoryRecall() => _memory;
  void memoryClear() => _memory = 0;
  void memoryAdd(double value) => _memory += value;

  // History
  List<String> get history => List.unmodifiable(_history);
  void clearHistory() => _history.clear();

  double _recordOp(String operation, double result) {
    _history.add('$operation = $result');
    return result;
  }
}

// test/calculator_comprehensive_test.dart
import 'package:test/test.dart';

void main() {
  group('Calculator - Comprehensive Tests', () {
    late Calculator calc;

    setUp(() => calc = Calculator());

    // ===== Basic Operations =====
    group('การดำเนินการพื้นฐาน', () {
      group('การบวก', () {
        test('บวกตัวเลขบวก', () {
          expect(calc.add(3, 5), equals(8));
        });

        test('บวกตัวเลขลบ', () {
          expect(calc.add(-3, -5), equals(-8));
        });

        test('บวกกับศูนย์', () {
          expect(calc.add(5, 0), equals(5));
          expect(calc.add(0, 5), equals(5));
        });

        test('บวกทศนิยม', () {
          expect(calc.add(1.1, 2.2), closeTo(3.3, 0.0001));
        });
      });

      group('การลบ', () {
        test('ลบได้ผลลัพธ์ลบ', () {
          expect(calc.subtract(3, 5), equals(-2));
        });

        test('ลบเองได้ศูนย์', () {
          expect(calc.subtract(5, 5), equals(0));
        });
      });

      group('การคูณ', () {
        test('คูณกับศูนย์', () {
          expect(calc.multiply(5, 0), equals(0));
        });

        test('คูณกับ 1', () {
          expect(calc.multiply(5, 1), equals(5));
        });

        test('คูณลบกับลบ', () {
          expect(calc.multiply(-3, -4), equals(12));
        });
      });

      group('การหาร', () {
        test('หารปกติ', () {
          expect(calc.divide(10, 2), equals(5));
        });

        test('หารได้ทศนิยม', () {
          expect(calc.divide(1, 3), closeTo(0.3333, 0.0001));
        });

        test('หารด้วยศูนย์ throws ArgumentError', () {
          expect(
            () => calc.divide(5, 0),
            throwsA(
              isA<ArgumentError>().having(
                (e) => e.message,
                'message',
                'ไม่สามารถหารด้วยศูนย์ได้',
              ),
            ),
          );
        });
      });
    });

    // ===== Advanced Operations =====
    group('การดำเนินการขั้นสูง', () {
      group('รากที่สอง', () {
        test('sqrt(9) = 3', () {
          expect(calc.sqrt(9), closeTo(3.0, 0.0001));
        });

        test('sqrt(0) = 0', () {
          expect(calc.sqrt(0), closeTo(0.0, 0.0001));
        });

        test('sqrt(2) ≈ 1.414', () {
          expect(calc.sqrt(2), closeTo(1.4142, 0.0001));
        });

        test('sqrt(จำนวนลบ) throws', () {
          expect(
            () => calc.sqrt(-1),
            throwsA(
              isA<ArgumentError>().having(
                (e) => e.message?.toString() ?? '',
                'message',
                contains('-1'),
              ),
            ),
          );
        });
      });

      group('เปอร์เซ็นต์', () {
        test('10% ของ 200 = 20', () {
          expect(calc.percentage(200, 10), equals(20));
        });

        test('100% ของ 50 = 50', () {
          expect(calc.percentage(50, 100), equals(50));
        });

        test('0% ของ 100 = 0', () {
          expect(calc.percentage(100, 0), equals(0));
        });
      });
    });

    // ===== Memory =====
    group('Memory', () {
      test('เก็บและเรียกคืนค่า', () {
        calc.memoryStore(42);
        expect(calc.memoryRecall(), equals(42));
      });

      test('memory เริ่มต้นเป็น 0', () {
        expect(calc.memoryRecall(), equals(0));
      });

      test('clear memory', () {
        calc.memoryStore(100);
        calc.memoryClear();
        expect(calc.memoryRecall(), equals(0));
      });

      test('memoryAdd บวกเพิ่มจากที่มีอยู่', () {
        calc.memoryStore(10);
        calc.memoryAdd(5);
        expect(calc.memoryRecall(), equals(15));
      });
    });

    // ===== History =====
    group('History', () {
      test('บันทึก operation', () {
        calc.add(2, 3);
        expect(calc.history, hasLength(1));
        expect(calc.history.first, contains('2.0 + 3.0'));
      });

      test('บันทึกหลาย operations', () {
        calc.add(1, 2);
        calc.subtract(5, 3);
        calc.multiply(4, 5);
        expect(calc.history, hasLength(3));
      });

      test('clearHistory ล้างประวัติ', () {
        calc.add(1, 2);
        calc.clearHistory();
        expect(calc.history, isEmpty);
      });

      test('history เป็น unmodifiable', () {
        calc.add(1, 2);
        expect(
          () => (calc.history as List).add('manual'),
          throwsUnsupportedError,
        );
      });
    });

    // ===== Edge Cases =====
    group('Edge Cases', () {
      test('ทำงานกับตัวเลขใหญ่มาก', () {
        expect(
          calc.multiply(1e100, 1e100),
          equals(1e200),
        );
      });

      test('ทำงานกับตัวเลขเล็กมาก', () {
        expect(
          calc.add(1e-100, 1e-100),
          closeTo(2e-100, 1e-115),
        );
      });

      test('หลาย operations ต่อกัน', () {
        // (5 + 3) * 2 / 4 = 4
        final sum = calc.add(5, 3);
        final product = calc.multiply(sum, 2);
        final result = calc.divide(product, 4);
        expect(result, equals(4));
      });
    });
  });
}
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Unit tests** - โครงสร้างและการเขียน tests พื้นฐาน
2. **Matchers** - ประเภทต่างๆ ของ expect()
3. **setUp/tearDown** - lifecycle ของ tests
4. **Mocking** - สร้าง mock objects สำหรับ dependencies
5. **Test coverage** - วัดและปรับปรุง coverage
6. **TDD** - เขียน test ก่อน แล้วค่อยเขียน code

### คำสั่งรัน Tests

```bash
# รัน tests ทั้งหมด
dart test

# รัน test ไฟล์เดียว
dart test test/calculator_test.dart

# รัน tests ที่ชื่อ match
dart test --name "หาร"

# รัน tests ใน group
dart test --name "Calculator"

# รัน พร้อม output verbose
dart test --reporter expanded

# รัน พร้อม coverage
dart test --coverage=coverage
```

### Best Practices

- เขียน test ที่ readable เหมือนเอกสาร
- ตั้งชื่อ test ให้อธิบายว่าทำอะไร
- หนึ่ง assertion ต่อหนึ่ง test (ทำได้หลาย assertion แต่ควรเกี่ยวข้องกัน)
- ทดสอบ edge cases ด้วย (0, -1, null, empty)
- ใช้ TDD เมื่อทำได้
- เป้าหมาย coverage 80%+ สำหรับ business logic
