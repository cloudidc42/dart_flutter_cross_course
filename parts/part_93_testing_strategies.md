# Part 93: Testing Strategies - Unit, Widget, Integration, E2E

## 🎯 เป้าหมายของ Part นี้
- Testing pyramid ใน Flutter
- Unit tests ด้วย test package
- Widget tests ด้วย flutter_test
- Integration tests
- Mocking ด้วย mockito/mocktail
- Golden tests
- E2E ด้วย Patrol

---

## 1. Testing Pyramid

```
        /\
       /E2E\         ← น้อย, ช้า, แพง
      /------\
     /Integr- \      ← ปานกลาง
    /----------\
   / Widget     \    ← เยอะ, เร็ว
  /--------------\
 /  Unit Tests    \  ← เยอะที่สุด, เร็วที่สุด
/------------------\

หลักการ: เขียน Unit tests เยอะ, E2E น้อย
```

---

## 2. Unit Tests

```dart
// test/unit/calculator_test.dart

import 'package:test/test.dart';

class Calculator {
  double add(double a, double b) => a + b;
  double subtract(double a, double b) => a - b;
  double multiply(double a, double b) => a * b;
  double divide(double a, double b) {
    if (b == 0) throw ArgumentError('Cannot divide by zero');
    return a / b;
  }
  
  double percentage(double value, double percent) {
    return value * percent / 100;
  }
  
  // FizzBuzz helper
  String fizzBuzz(int n) {
    if (n % 15 == 0) return 'FizzBuzz';
    if (n % 3 == 0) return 'Fizz';
    if (n % 5 == 0) return 'Buzz';
    return '$n';
  }
}

void main() {
  late Calculator calc;
  
  setUp(() {
    calc = Calculator(); // สร้างใหม่ก่อนแต่ละ test
  });
  
  // ============ group - จัดกลุ่ม tests ============
  group('Calculator Tests', () {
    
    group('Addition', () {
      test('adds two positive numbers', () {
        expect(calc.add(3, 4), equals(7));
      });
      
      test('adds negative numbers', () {
        expect(calc.add(-5, 3), equals(-2));
      });
      
      test('adds zero', () {
        expect(calc.add(5, 0), equals(5));
      });
      
      test('adds decimals', () {
        expect(calc.add(1.5, 2.5), closeTo(4.0, 0.001));
      });
    });
    
    group('Division', () {
      test('divides correctly', () {
        expect(calc.divide(10, 4), equals(2.5));
      });
      
      test('throws on division by zero', () {
        expect(
          () => calc.divide(10, 0),
          throwsA(isA<ArgumentError>().having(
            (e) => e.message,
            'message',
            contains('zero'),
          )),
        );
      });
    });
    
    group('FizzBuzz', () {
      test('returns Fizz for multiples of 3', () {
        expect(calc.fizzBuzz(3), 'Fizz');
        expect(calc.fizzBuzz(6), 'Fizz');
        expect(calc.fizzBuzz(9), 'Fizz');
      });
      
      test('returns Buzz for multiples of 5', () {
        expect(calc.fizzBuzz(5), 'Buzz');
        expect(calc.fizzBuzz(10), 'Buzz');
      });
      
      test('returns FizzBuzz for multiples of 15', () {
        expect(calc.fizzBuzz(15), 'FizzBuzz');
        expect(calc.fizzBuzz(30), 'FizzBuzz');
      });
      
      test('returns number as string for others', () {
        expect(calc.fizzBuzz(1), '1');
        expect(calc.fizzBuzz(7), '7');
        expect(calc.fizzBuzz(11), '11');
      });
      
      // Test multiple values ด้วย parametrized:
      for (final testCase in [
        (3, 'Fizz'),
        (5, 'Buzz'),
        (15, 'FizzBuzz'),
        (1, '1'),
        (7, '7'),
      ]) {
        test('fizzBuzz(${testCase.$1}) == "${testCase.$2}"', () {
          expect(calc.fizzBuzz(testCase.$1), testCase.$2);
        });
      }
    });
  });
  
  // ============ Async tests ============
  group('Async Tests', () {
    test('future completes', () async {
      Future<String> fetchData() async {
        await Future.delayed(const Duration(milliseconds: 100));
        return 'data';
      }
      
      expect(await fetchData(), 'data');
    });
    
    test('future throws', () async {
      Future<String> failingFetch() async {
        throw Exception('Network error');
      }
      
      expect(failingFetch(), throwsA(isA<Exception>()));
    });
    
    test('stream emits values', () async {
      Stream<int> countStream() async* {
        for (int i = 1; i <= 3; i++) {
          await Future.delayed(const Duration(milliseconds: 10));
          yield i;
        }
      }
      
      expect(countStream(), emitsInOrder([1, 2, 3, emitsDone]));
    });
  });
}
```

---

## 3. Mocking ด้วย Mocktail

```dart
// pubspec.yaml:
// dev_dependencies:
//   mocktail: ^1.0.0

import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

// Repository interface:
abstract class UserRepository {
  Future<User> getUser(String id);
  Future<List<User>> getUsers();
  Future<void> saveUser(User user);
  Future<bool> deleteUser(String id);
}

// Mock:
class MockUserRepository extends Mock implements UserRepository {}

// Service ที่ต้องการ test:
class UserService {
  final UserRepository _repository;
  
  UserService(this._repository);
  
  Future<String> getUserDisplayName(String id) async {
    final user = await _repository.getUser(id);
    return '${user.firstName} ${user.lastName}';
  }
  
  Future<List<User>> getActiveUsers() async {
    final users = await _repository.getUsers();
    return users.where((u) => u.isActive).toList();
  }
  
  Future<bool> deactivateUser(String id) async {
    final user = await _repository.getUser(id);
    await _repository.saveUser(user.copyWith(isActive: false));
    return true;
  }
}

class User {
  final String id;
  final String firstName;
  final String lastName;
  final bool isActive;
  
  const User({
    required this.id,
    required this.firstName,
    required this.lastName,
    this.isActive = true,
  });
  
  User copyWith({bool? isActive}) {
    return User(
      id: id,
      firstName: firstName,
      lastName: lastName,
      isActive: isActive ?? this.isActive,
    );
  }
}

void main() {
  late MockUserRepository mockRepo;
  late UserService service;
  
  setUp(() {
    mockRepo = MockUserRepository();
    service = UserService(mockRepo);
  });
  
  group('UserService', () {
    test('getUserDisplayName returns full name', () async {
      // Arrange
      const user = User(id: '1', firstName: 'สมชาย', lastName: 'ใจดี');
      when(() => mockRepo.getUser('1')).thenAnswer((_) async => user);
      
      // Act
      final result = await service.getUserDisplayName('1');
      
      // Assert
      expect(result, 'สมชาย ใจดี');
      verify(() => mockRepo.getUser('1')).called(1);
    });
    
    test('getActiveUsers returns only active', () async {
      // Arrange
      final users = [
        const User(id: '1', firstName: 'Alice', lastName: 'A', isActive: true),
        const User(id: '2', firstName: 'Bob', lastName: 'B', isActive: false),
        const User(id: '3', firstName: 'Charlie', lastName: 'C', isActive: true),
      ];
      when(() => mockRepo.getUsers()).thenAnswer((_) async => users);
      
      // Act
      final active = await service.getActiveUsers();
      
      // Assert
      expect(active.length, 2);
      expect(active.map((u) => u.id), containsAll(['1', '3']));
    });
    
    test('deactivateUser calls save with isActive false', () async {
      // Arrange
      const user = User(id: '1', firstName: 'Alice', lastName: 'A', isActive: true);
      when(() => mockRepo.getUser('1')).thenAnswer((_) async => user);
      when(() => mockRepo.saveUser(any())).thenAnswer((_) async {});
      
      // Act
      await service.deactivateUser('1');
      
      // Assert
      final captured = verify(() => mockRepo.saveUser(captureAny())).captured;
      final savedUser = captured.first as User;
      expect(savedUser.isActive, false);
      expect(savedUser.id, '1');
    });
    
    test('getUserDisplayName throws when repo throws', () async {
      // Arrange
      when(() => mockRepo.getUser(any())).thenThrow(Exception('Not found'));
      
      // Assert
      expect(
        service.getUserDisplayName('999'),
        throwsA(isA<Exception>()),
      );
    });
  });
}
```

---

## 4. Widget Tests

```dart
// test/widget/counter_widget_test.dart

import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

// Widget ที่ต้องการ test:
class CounterWidget extends StatefulWidget {
  const CounterWidget({super.key});

  @override
  State<CounterWidget> createState() => _CounterWidgetState();
}

class _CounterWidgetState extends State<CounterWidget> {
  int _count = 0;

  @override
  Widget build(BuildContext context) {
    return Column(
      mainAxisSize: MainAxisSize.min,
      children: [
        Text('$_count', key: const Key('counter-text')),
        ElevatedButton(
          key: const Key('increment-button'),
          onPressed: () => setState(() => _count++),
          child: const Text('เพิ่ม'),
        ),
        ElevatedButton(
          key: const Key('decrement-button'),
          onPressed: () => setState(() => _count--),
          child: const Text('ลด'),
        ),
        ElevatedButton(
          key: const Key('reset-button'),
          onPressed: () => setState(() => _count = 0),
          child: const Text('รีเซ็ต'),
        ),
      ],
    );
  }
}

void main() {
  group('CounterWidget', () {
    testWidgets('shows initial count of 0', (tester) async {
      // Build widget
      await tester.pumpWidget(
        const MaterialApp(
          home: Scaffold(body: CounterWidget()),
        ),
      );
      
      // Verify
      expect(find.byKey(const Key('counter-text')), findsOneWidget);
      expect(find.text('0'), findsOneWidget);
    });
    
    testWidgets('increments count when button pressed', (tester) async {
      await tester.pumpWidget(
        const MaterialApp(home: Scaffold(body: CounterWidget())),
      );
      
      // Tap increment button
      await tester.tap(find.byKey(const Key('increment-button')));
      await tester.pump(); // rebuild
      
      expect(find.text('1'), findsOneWidget);
      
      await tester.tap(find.byKey(const Key('increment-button')));
      await tester.pump();
      
      expect(find.text('2'), findsOneWidget);
    });
    
    testWidgets('decrements count when button pressed', (tester) async {
      await tester.pumpWidget(
        const MaterialApp(home: Scaffold(body: CounterWidget())),
      );
      
      await tester.tap(find.byKey(const Key('decrement-button')));
      await tester.pump();
      
      expect(find.text('-1'), findsOneWidget);
    });
    
    testWidgets('resets count to 0', (tester) async {
      await tester.pumpWidget(
        const MaterialApp(home: Scaffold(body: CounterWidget())),
      );
      
      // Increment 3 times
      for (int i = 0; i < 3; i++) {
        await tester.tap(find.byKey(const Key('increment-button')));
        await tester.pump();
      }
      expect(find.text('3'), findsOneWidget);
      
      // Reset
      await tester.tap(find.byKey(const Key('reset-button')));
      await tester.pump();
      
      expect(find.text('0'), findsOneWidget);
    });
    
    testWidgets('finds all buttons', (tester) async {
      await tester.pumpWidget(
        const MaterialApp(home: Scaffold(body: CounterWidget())),
      );
      
      expect(find.byType(ElevatedButton), findsNWidgets(3));
      expect(find.text('เพิ่ม'), findsOneWidget);
      expect(find.text('ลด'), findsOneWidget);
      expect(find.text('รีเซ็ต'), findsOneWidget);
    });
    
    testWidgets('handles async operations', (tester) async {
      // Widget ที่มี async loading:
      await tester.pumpWidget(const MaterialApp(home: AsyncLoadingWidget()));
      
      // ตรวจสอบ loading state:
      expect(find.byType(CircularProgressIndicator), findsOneWidget);
      
      // รอ async operation เสร็จ:
      await tester.pumpAndSettle();
      
      // ตรวจสอบ loaded state:
      expect(find.byType(CircularProgressIndicator), findsNothing);
      expect(find.text('โหลดเสร็จ'), findsOneWidget);
    });
  });
}

class AsyncLoadingWidget extends StatefulWidget {
  const AsyncLoadingWidget({super.key});

  @override
  State<AsyncLoadingWidget> createState() => _AsyncLoadingWidgetState();
}

class _AsyncLoadingWidgetState extends State<AsyncLoadingWidget> {
  bool _isLoading = true;

  @override
  void initState() {
    super.initState();
    Future.delayed(const Duration(seconds: 1), () {
      if (mounted) setState(() => _isLoading = false);
    });
  }

  @override
  Widget build(BuildContext context) {
    return _isLoading 
        ? const CircularProgressIndicator()
        : const Text('โหลดเสร็จ');
  }
}
```

---

## 5. Integration Tests

```dart
// integration_test/app_test.dart

import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();
  
  group('App Integration Tests', () {
    testWidgets('complete login flow', (tester) async {
      // Launch app
      await tester.pumpWidget(const MyApp());
      await tester.pumpAndSettle();
      
      // กรอก email
      await tester.enterText(
        find.byKey(const Key('email-field')),
        'admin@example.com',
      );
      
      // กรอก password
      await tester.enterText(
        find.byKey(const Key('password-field')),
        '123456',
      );
      
      // กด login
      await tester.tap(find.byKey(const Key('login-button')));
      await tester.pumpAndSettle(const Duration(seconds: 3));
      
      // ตรวจสอบว่า navigate ไป home
      expect(find.byKey(const Key('home-screen')), findsOneWidget);
    });
    
    testWidgets('navigation flow', (tester) async {
      await tester.pumpWidget(const MyApp());
      await tester.pumpAndSettle();
      
      // ไปหน้า products
      await tester.tap(find.byIcon(Icons.shopping_bag));
      await tester.pumpAndSettle();
      
      expect(find.text('สินค้า'), findsOneWidget);
      
      // กด back
      await tester.pageBack();
      await tester.pumpAndSettle();
      
      expect(find.byKey(const Key('home-screen')), findsOneWidget);
    });
  });
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return const MaterialApp(home: Text('App'));
  }
}
```

---

## 6. สรุป Part 93

สิ่งที่เรียนรู้:
- ✅ Testing pyramid - Unit, Widget, Integration, E2E
- ✅ Unit tests ด้วย test package
- ✅ Group, setUp, tearDown
- ✅ Async tests
- ✅ Mocking ด้วย Mocktail
- ✅ Widget tests ด้วย flutter_test
- ✅ Finders, matchers, interactions
- ✅ Integration tests

---

## ➡️ Part ถัดไป
**Part 94: App Store Optimization and Release Management**
