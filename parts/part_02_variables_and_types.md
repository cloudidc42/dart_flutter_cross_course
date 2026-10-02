# Part 02: ตัวแปร, ประเภทข้อมูล และ Null Safety

## 🎯 เป้าหมายของ Part นี้
- เข้าใจประเภทข้อมูลทั้งหมดใน Dart
- ใช้ var, final, const ได้อย่างถูกต้อง
- เข้าใจและใช้ Null Safety
- Type casting และ Type checking
- String manipulation

---

## 1. ประเภทข้อมูลพื้นฐาน (Built-in Types)

Dart มีประเภทข้อมูลหลักดังนี้:

```dart
void main() {
  // ============ Numbers ============
  int age = 25;           // จำนวนเต็ม
  double height = 175.5;  // จำนวนทศนิยม
  num score = 98;         // int หรือ double ก็ได้
  
  // Number operations
  print(age.runtimeType);      // int
  print(height.runtimeType);   // double
  print(score.runtimeType);    // int
  
  // Integer methods
  print(age.isEven);      // false (25 คี่)
  print(age.isOdd);       // true
  print(age.abs());       // 25 (absolute value)
  print(-10.abs());       // 10
  
  // Double methods
  print(height.floor());  // 175 (ปัดลง)
  print(height.ceil());   // 176 (ปัดขึ้น)
  print(height.round());  // 176 (ปัดปกติ)
  print(height.toInt());  // 175 (ตัดทศนิยม)
  
  // Math operations
  print(10 / 3);          // 3.3333... (double division)
  print(10 ~/ 3);         // 3 (integer division)
  print(10 % 3);          // 1 (modulo/remainder)
  
  // ============ Strings ============
  String name = 'Dart';
  String greeting = "สวัสดี";
  
  // String interpolation
  String message = 'สวัสดี $name! ยินดีต้อนรับ';
  String complex = '${name.toUpperCase()} is great!';
  
  // Multi-line strings
  String poem = '''
  Dart เป็นภาษา
  ที่สวยงาม
  และทรงพลัง
  ''';
  
  String html = """
  <div>
    <p>Hello World</p>
  </div>
  """;
  
  // Raw strings (ไม่แปล escape sequences)
  String path = r'C:\Users\name\documents'; // r prefix
  String regex = r'\d+\.\d+'; // regular expression
  
  // ============ Booleans ============
  bool isFlutterAwesome = true;
  bool isComplicated = false;
  
  // Boolean expressions
  print(5 > 3);   // true
  print(5 == 5);  // true
  print(5 != 3);  // true
  print(!true);   // false
  
  // ============ Null ============
  // ใน Dart 3.x, ตัวแปรปกติไม่สามารถเป็น null ได้
  // int x = null; // ERROR! ทำไม่ได้
  
  // ถ้าต้องการให้เป็น null ได้ ใช้ ? หลัง type:
  int? nullableInt = null;
  String? nullableString = null;
  print(nullableInt);   // null
  print(nullableString); // null
}
```

---

## 2. var, final, const - ความแตกต่าง

```dart
void main() {
  // ============ var ============
  // Type inference - Dart คาดเดา type เอง
  var number = 42;        // ถูก infer เป็น int
  var text = 'hello';     // ถูก infer เป็น String
  var flag = true;        // ถูก infer เป็น bool
  var decimal = 3.14;     // ถูก infer เป็น double
  
  // สามารถเปลี่ยนค่าได้ (แต่เปลี่ยน type ไม่ได้)
  number = 100;    // ✅ OK
  text = 'world';  // ✅ OK
  // number = 'text'; // ❌ ERROR: type mismatch
  
  // ============ final ============
  // กำหนดค่าได้ครั้งเดียว (runtime constant)
  final String myName = 'Dart Developer';
  final currentTime = DateTime.now(); // กำหนดตอน runtime
  
  // myName = 'Other'; // ❌ ERROR: can't assign to final
  
  // final ใน class field
  // สามารถกำหนดใน constructor ได้
  
  // ============ const ============
  // compile-time constant - ต้องรู้ค่าตอน compile
  const double pi = 3.14159;
  const int maxItems = 100;
  const String appName = 'My Flutter App';
  
  // appName = 'Other'; // ❌ ERROR: can't assign to const
  
  // const ใช้กับ collections ได้:
  const List<String> colors = ['red', 'green', 'blue'];
  // colors.add('yellow'); // ❌ ERROR: const list is immutable
  
  // ============ late ============
  // กำหนดค่าทีหลัง แต่ต้องกำหนดก่อนใช้
  late String lazyValue;
  // ... logic ...
  lazyValue = 'I was set later'; // กำหนดค่า
  print(lazyValue); // ✅ OK
  
  // late + final = กำหนดได้ครั้งเดียว แต่ทีหลังได้
  late final String lateConst;
  lateConst = 'Set once, later';
  // lateConst = 'Error'; // ❌ ERROR
}
```

### เมื่อไหร่ใช้อะไร?
```dart
// ใช้ var เมื่อ:
var list = [1, 2, 3]; // local variable ที่เปลี่ยนค่าได้

// ใช้ final เมื่อ:
final user = fetchUser(); // ค่าที่กำหนดครั้งเดียวใน runtime

// ใช้ const เมื่อ:
const double gravity = 9.8; // ค่าคงที่ที่รู้ตอน compile

// กฎง่ายๆ:
// - เปลี่ยนค่าได้ → var
// - เปลี่ยนค่าไม่ได้ runtime → final
// - เปลี่ยนค่าไม่ได้ compile time → const
```

---

## 3. Null Safety เชิงลึก

Dart 2.12+ มี Sound Null Safety ที่ช่วยป้องกัน Null Pointer Exceptions:

```dart
void main() {
  // ============ Non-nullable (default) ============
  String name = 'Dart';
  // name = null; // ❌ ERROR: String is non-nullable
  
  // ============ Nullable ============
  String? nullableName = null;
  print(nullableName); // null
  
  // ============ Null-aware operators ============
  
  // ?. - Null-conditional operator
  String? str = null;
  print(str?.length);      // null (ไม่ crash)
  print(str?.toUpperCase()); // null
  
  str = 'hello';
  print(str?.length);      // 5
  
  // ?? - Null coalescing operator
  String? value = null;
  String result = value ?? 'default'; // ถ้า null ใช้ default
  print(result); // 'default'
  
  value = 'actual value';
  result = value ?? 'default';
  print(result); // 'actual value'
  
  // ??= - Null coalescing assignment
  String? config;
  config ??= 'default config'; // กำหนดเฉพาะเมื่อ null
  print(config); // 'default config'
  
  config ??= 'other config'; // จะไม่กำหนดเพราะไม่ null แล้ว
  print(config); // 'default config'
  
  // ! - Null assertion operator (use with caution!)
  String? maybeNull = 'not null';
  String definitelyNotNull = maybeNull!; // บอก Dart ว่าแน่ใจว่าไม่ null
  print(definitelyNotNull); // 'not null'
  
  // ⚠️ ถ้า maybeNull เป็น null จริงๆ จะ throw error:
  // String? danger = null;
  // String crash = danger!; // ❌ Null check operator used on null value
}
```

### Null Safety ใน Functions:
```dart
// รับ nullable parameter:
void greet(String? name) {
  // วิธีที่ 1: ตรวจสอบด้วย if
  if (name != null) {
    print('สวัสดี $name');
  } else {
    print('สวัสดีคุณ');
  }
}

// วิธีที่ 2: ใช้ ?? operator
void greet2(String? name) {
  print('สวัสดี ${name ?? 'คุณ'}');
}

// รับ non-nullable parameter:
void greet3(String name) {
  print('สวัสดี $name'); // ปลอดภัย ไม่ต้องตรวจสอบ
}

// คืนค่า nullable:
String? findUser(int id) {
  if (id == 1) return 'Alice';
  return null; // อาจคืน null
}

// คืนค่า non-nullable:
String getUser(int id) {
  return 'User $id'; // ต้องคืนค่าเสมอ ห้ามคืน null
}
```

---

## 4. String Methods ที่สำคัญ

```dart
void main() {
  String text = 'สวัสดี Dart Flutter!';
  
  // ============ ข้อมูลพื้นฐาน ============
  print(text.length);           // 20
  print(text.isEmpty);          // false
  print(text.isNotEmpty);       // true
  
  // ============ การค้นหา ============
  print(text.contains('Dart'));     // true
  print(text.startsWith('สวัสดี')); // true
  print(text.endsWith('!'));        // true
  print(text.indexOf('Dart'));      // 8
  print(text.lastIndexOf('t'));     // 19
  
  // ============ การแก้ไข ============
  print(text.toUpperCase());  // สวัสดี DART FLUTTER!
  print(text.toLowerCase());  // สวัสดี dart flutter!
  
  print(text.trim());         // ลบ spaces หน้า-หลัง
  print('  hello  '.trim()); // 'hello'
  print('  hello  '.trimLeft());  // 'hello  '
  print('  hello  '.trimRight()); // '  hello'
  
  print(text.replaceAll('!', '.')); // สวัสดี Dart Flutter.
  print(text.replaceFirst('า', 'A')); // สวัสดA Dart Flutter!
  
  // ============ การแบ่ง ============
  String csv = 'apple,banana,cherry,date';
  List<String> fruits = csv.split(',');
  print(fruits); // [apple, banana, cherry, date]
  
  print(text.substring(8, 12)); // Dart
  print(text.substring(13));    // Flutter!
  
  // ============ การรวม ============
  List<String> words = ['Hello', 'World', 'Dart'];
  print(words.join(' ')); // Hello World Dart
  print(words.join(', ')); // Hello, World, Dart
  
  // ============ การตรวจสอบ ============
  String number = '12345';
  print(int.tryParse(number));    // 12345
  print(double.tryParse('3.14')); // 3.14
  print(int.tryParse('abc'));     // null (ไม่ใช่ตัวเลข)
  
  // ============ String Buffer ============
  // ใช้สำหรับต่อ String หลายๆ ครั้ง (มีประสิทธิภาพกว่า + operator)
  StringBuffer buffer = StringBuffer();
  buffer.write('Hello');
  buffer.write(' ');
  buffer.write('World');
  buffer.writeln('!'); // เพิ่ม newline
  buffer.writeAll(['a', 'b', 'c'], '-'); // a-b-c
  print(buffer.toString()); // Hello World!\na-b-c
  
  // ============ String Formatting ============
  double price = 1234.5678;
  print(price.toStringAsFixed(2));     // 1234.57
  print(price.toStringAsPrecision(6)); // 1234.57
  
  int hex = 255;
  print(hex.toRadixString(16));  // ff (hexadecimal)
  print(hex.toRadixString(2));   // 11111111 (binary)
  print(hex.toRadixString(8));   // 377 (octal)
}
```

---

## 5. Numbers - การคำนวณขั้นสูง

```dart
import 'dart:math'; // import ไลบรารี math

void main() {
  // ============ dart:math ============
  print(max(10, 20));      // 20
  print(min(10, 20));      // 10
  print(sqrt(16));         // 4.0
  print(pow(2, 8));        // 256
  print(log(100));         // 4.605... (natural log)
  print(log(100) / log(10)); // 2.0 (log base 10)
  print(sin(pi / 2));      // 1.0
  print(cos(0));           // 1.0
  
  // ============ Random Numbers ============
  Random random = Random();
  print(random.nextInt(10));     // 0-9
  print(random.nextInt(100));    // 0-99
  print(random.nextDouble());    // 0.0 - 1.0
  print(random.nextBool());      // true หรือ false
  
  // Random ที่ซ้ำได้ (reproducible) ด้วย seed:
  Random seeded = Random(42);
  print(seeded.nextInt(100)); // เสมอ 0 ด้วย seed 42
  
  // ============ Number Parsing ============
  String numStr = '42';
  int parsed = int.parse(numStr);     // 42
  double parsedD = double.parse('3.14'); // 3.14
  
  // Safe parsing (ไม่ throw exception):
  int? safe = int.tryParse('abc');    // null
  int? ok = int.tryParse('42');       // 42
  
  // ============ Number Formatting ============
  double money = 1234567.89;
  
  // ใช้ NumberFormat จาก intl package:
  // NumberFormat formatter = NumberFormat('#,###.##');
  // print(formatter.format(money)); // 1,234,567.89
  
  // แบบง่าย:
  print(money.toStringAsFixed(2)); // 1234567.89
}
```

---

## 6. Type Checking และ Casting

```dart
void main() {
  dynamic value = 42;
  
  // ============ is operator ============
  print(value is int);     // true
  print(value is String);  // false
  print(value is num);     // true (int is subtype of num)
  
  // is! operator:
  print(value is! String); // true
  
  // ============ Type Casting ============
  // แบบ explicit:
  Object obj = 'Hello';
  String str = obj as String; // cast to String
  print(str.length); // 5
  
  // ⚠️ ถ้า cast ไม่ถูก type จะ throw CastError:
  // Object num = 42;
  // String fail = num as String; // ❌ ERROR
  
  // ============ Safe Casting ============
  Object something = 42;
  
  // วิธีที่ 1: ตรวจสอบก่อน cast
  if (something is int) {
    print(something + 1); // 43 (Dart รู้ว่าเป็น int แล้ว = type promotion)
  }
  
  // วิธีที่ 2: ใช้ as? (nullable safe cast) - ไม่มีใน Dart
  // Dart ใช้วิธี check แล้ว cast แทน
  
  // ============ int ↔ double conversion ============
  int intVal = 42;
  double doubleVal = intVal.toDouble(); // 42.0
  double d = 3.99;
  int i = d.toInt();    // 3 (ตัดทศนิยม ไม่ปัด!)
  int r = d.round();    // 4 (ปัดปกติ)
  int f = d.floor();    // 3 (ปัดลง)
  int c = d.ceil();     // 4 (ปัดขึ้น)
  
  // ============ String ↔ Number ============
  int n = int.parse('42');
  double dd = double.parse('3.14');
  String s1 = 42.toString();
  String s2 = 3.14.toString();
  String s3 = 255.toRadixString(16); // 'ff'
  
  // ============ dynamic vs Object ============
  dynamic x = 'hello';
  x = 42;       // ✅ dynamic สามารถเปลี่ยน type ได้
  x = true;     // ✅
  
  Object y = 'hello';
  y = 42;       // ✅ Object ก็เปลี่ยนได้
  // y.length;  // ❌ ไม่สามารถเรียก String methods บน Object
  
  // ใช้ dynamic เมื่อต้องการ flexibility
  // ใช้ Object เมื่อต้องการ type safety แต่ไม่รู้ type ล่วงหน้า
}
```

---

## 7. Collections Types Overview

```dart
void main() {
  // ============ List ============
  List<String> fruits = ['apple', 'banana', 'cherry'];
  var numbers = [1, 2, 3, 4, 5]; // inferred as List<int>
  
  print(fruits[0]);      // apple
  print(fruits.length);  // 3
  fruits.add('date');
  fruits.remove('banana');
  
  // ============ Map ============
  Map<String, int> scores = {
    'Alice': 95,
    'Bob': 87,
    'Charlie': 92,
  };
  
  print(scores['Alice']);  // 95
  scores['Dave'] = 88;
  scores.remove('Bob');
  
  // ============ Set ============
  Set<String> unique = {'apple', 'banana', 'apple'}; // ไม่มีซ้ำ
  print(unique); // {apple, banana}
  unique.add('cherry');
  unique.add('apple'); // ไม่เพิ่ม เพราะมีแล้ว
  
  // ============ Iterable ============
  Iterable<int> range = Iterable.generate(5); // 0,1,2,3,4
  print(range.toList()); // [0, 1, 2, 3, 4]
}
```

---

## 8. ตัวอย่างจริง: User Profile Data

```dart
// user_profile.dart - ตัวอย่างการใช้ types ต่างๆ

class UserProfile {
  final String id;
  final String name;
  final String? email;    // nullable - อาจไม่มี
  final int age;
  final double? rating;   // nullable rating
  final bool isVerified;
  final DateTime createdAt;
  final List<String> hobbies;
  final Map<String, String> socialLinks;
  
  const UserProfile({
    required this.id,
    required this.name,
    this.email,           // optional
    required this.age,
    this.rating,          // optional
    this.isVerified = false, // default value
    required this.createdAt,
    this.hobbies = const [], // default empty list
    this.socialLinks = const {}, // default empty map
  });
  
  // Computed property
  String get displayName => name.isEmpty ? 'Anonymous' : name;
  
  String get ageGroup {
    if (age < 18) return 'เยาวชน';
    if (age < 30) return 'วัยรุ่น';
    if (age < 50) return 'วัยกลางคน';
    return 'ผู้สูงอายุ';
  }
  
  bool get hasEmail => email != null;
  
  String get ratingDisplay => rating != null 
      ? '${rating!.toStringAsFixed(1)} ⭐'
      : 'ยังไม่ได้รับคะแนน';
  
  @override
  String toString() {
    return '''
UserProfile:
  id: $id
  name: $displayName ($ageGroup)
  email: ${email ?? 'ไม่ระบุ'}
  age: $age
  rating: $ratingDisplay
  verified: ${isVerified ? '✅' : '❌'}
  hobbies: ${hobbies.join(', ')}
    ''';
  }
}

void main() {
  // สร้าง user profile
  final user1 = UserProfile(
    id: 'u001',
    name: 'สมชาย ใจดี',
    email: 'somchai@example.com',
    age: 28,
    rating: 4.7,
    isVerified: true,
    createdAt: DateTime(2023, 1, 15),
    hobbies: ['ท่องเที่ยว', 'อ่านหนังสือ', 'เล่นดนตรี'],
    socialLinks: {
      'github': 'github.com/somchai',
      'linkedin': 'linkedin.com/in/somchai',
    },
  );
  
  // User ที่ไม่มี email
  final user2 = UserProfile(
    id: 'u002',
    name: 'สมหญิง รักเรียน',
    age: 22,
    createdAt: DateTime.now(),
  );
  
  print(user1);
  print(user2);
  
  // Null safety operations:
  String emailDisplay = user1.email ?? 'ไม่มี email';
  print('Email: $emailDisplay');
  
  // Conditional member access:
  String? emailUpper = user2.email?.toUpperCase();
  print('Email upper: ${emailUpper ?? 'null'}');
  
  // Type checking:
  print(user1.rating is double); // true
  print(user1.age is int);       // true
  
  // null check before use:
  if (user1.rating != null) {
    double roundedRating = user1.rating!.roundToDouble();
    print('Rounded rating: $roundedRating');
  }
}
```

---

## 9. Workshop: Calculator ด้วย Type Safety

```dart
// calculator.dart
import 'dart:io';

class Calculator {
  // ประวัติการคำนวณ
  final List<String> history = [];
  
  // คำนวณ 2 ตัวเลข
  double calculate(double a, double b, String operator) {
    double result;
    
    switch (operator) {
      case '+':
        result = a + b;
      case '-':
        result = a - b;
      case '*':
      case 'x':
        result = a * b;
      case '/':
        if (b == 0) throw ArgumentError('ห้ามหารด้วยศูนย์!');
        result = a / b;
      case '^':
        result = _power(a, b.toInt());
      case '%':
        result = a % b;
      default:
        throw ArgumentError('ไม่รู้จัก operator: $operator');
    }
    
    String calculation = '$a $operator $b = $result';
    history.add(calculation);
    return result;
  }
  
  double _power(double base, int exp) {
    if (exp == 0) return 1;
    if (exp < 0) return 1 / _power(base, -exp);
    return base * _power(base, exp - 1);
  }
  
  void printHistory() {
    if (history.isEmpty) {
      print('ยังไม่มีประวัติการคำนวณ');
      return;
    }
    print('=== ประวัติการคำนวณ ===');
    for (int i = 0; i < history.length; i++) {
      print('${i + 1}. ${history[i]}');
    }
  }
  
  void clearHistory() {
    history.clear();
    print('ล้างประวัติแล้ว');
  }
}

void main() {
  Calculator calc = Calculator();
  
  // ทดสอบการคำนวณ
  print(calc.calculate(10, 5, '+'));   // 15.0
  print(calc.calculate(10, 5, '-'));   // 5.0
  print(calc.calculate(10, 5, '*'));   // 50.0
  print(calc.calculate(10, 5, '/'));   // 2.0
  print(calc.calculate(2, 8, '^'));    // 256.0
  
  // Type safety
  double? result;
  try {
    result = calc.calculate(10, 0, '/');
  } catch (e) {
    print('Error: $e');
    result = null; // กำหนดเป็น null เมื่อ error
  }
  
  print('ผลลัพธ์: ${result?.toStringAsFixed(2) ?? "undefined"}');
  
  // แสดงประวัติ
  calc.printHistory();
  
  // Parse จาก string
  String? input = '42.5';
  double? parsed = double.tryParse(input);
  if (parsed != null) {
    print('Parsed: $parsed');
    print('Type: ${parsed.runtimeType}');
  }
}
```

---

## 10. สรุป Part 02

สิ่งที่เรียนรู้:
- ✅ ประเภทข้อมูลพื้นฐาน: int, double, num, String, bool
- ✅ var, final, const, late - ความแตกต่างและการใช้งาน
- ✅ Null Safety: nullable (?), null-aware operators (?., ??, ??=, !)
- ✅ String methods: length, trim, split, replaceAll, substring, join
- ✅ Number operations: math, parsing, formatting
- ✅ Type checking (is, is!) และ Type casting (as)
- ✅ Collections overview: List, Map, Set
- ✅ Dynamic vs Object

### Quiz ทบทวน:
```dart
// คำตอบของแบบฝึกหัด:

// 1. อะไรแตกต่างระหว่าง var กับ final?
// var: เปลี่ยนค่าได้
// final: เปลี่ยนค่าไม่ได้หลังจากกำหนดครั้งแรก

// 2. อะไรคือ Null Safety?
// การที่ Dart บังคับให้ประกาศชัดเจนว่าตัวแปรใดที่รับค่า null ได้
// ช่วยป้องกัน NullPointerException ตั้งแต่ compile time

// 3. ผลลัพธ์ของโค้ดนี้คืออะไร?
String? name = null;
print(name?.length ?? -1); // -1 (เพราะ name เป็น null)

// 4. อะไรคือความแตกต่างระหว่าง / และ ~/?
print(7 / 2);   // 3.5 (double division)
print(7 ~/ 2);  // 3 (integer division)
```

---

## ➡️ Part ถัดไป
**Part 03: Operators และ Expressions**

เราจะเรียนรู้:
- Arithmetic, Comparison, Logical operators
- Bitwise operators
- Conditional expressions
- Cascade notation (..)
- Spread operator (...)
