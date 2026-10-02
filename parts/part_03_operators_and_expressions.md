# Part 03: Operators และ Expressions

## 🎯 เป้าหมายของ Part นี้
- Arithmetic, Comparison, Logical operators ทั้งหมด
- Bitwise operators
- Conditional expressions (ternary, if-null)
- Cascade notation (..)
- Spread operator (...)
- Assignment operators

---

## 1. Arithmetic Operators (ตัวดำเนินการทางคณิตศาสตร์)

```dart
void main() {
  int a = 15;
  int b = 4;
  
  // ============ พื้นฐาน ============
  print(a + b);   // 19 - การบวก
  print(a - b);   // 11 - การลบ
  print(a * b);   // 60 - การคูณ
  print(a / b);   // 3.75 - การหาร (คืน double เสมอ)
  print(a ~/ b);  // 3  - Integer division (คืน int)
  print(a % b);   // 3  - Modulo/Remainder
  print(-a);      // -15 - Unary minus
  
  // ============ Increment / Decrement ============
  int x = 10;
  
  // Prefix (ดำเนินการก่อน แล้วค่อยใช้):
  print(++x);  // 11 (เพิ่มก่อน แล้วส่งค่า 11)
  print(--x);  // 10 (ลดก่อน แล้วส่งค่า 10)
  
  // Postfix (ใช้ค่าก่อน แล้วค่อยดำเนินการ):
  print(x++);  // 10 (ส่งค่า 10 ก่อน แล้วเพิ่มเป็น 11)
  print(x);    // 11
  print(x--);  // 11 (ส่งค่า 11 ก่อน แล้วลดเป็น 10)
  print(x);    // 10
  
  // ============ ตัวอย่างการใช้ % ============
  // ตรวจสอบเลขคู่/คี่
  for (int i = 1; i <= 10; i++) {
    String type = i % 2 == 0 ? 'คู่' : 'คี่';
    print('$i เป็นเลข$type');
  }
  
  // วันในสัปดาห์ (circular)
  int today = 3; // วันพุธ (0=อาทิตย์)
  int after5days = (today + 5) % 7; // 1 = วันจันทร์
  print('5 วันหลังจากวันพุธ = วัน ${after5days}'); // 1
}
```

---

## 2. Comparison Operators (ตัวดำเนินการเปรียบเทียบ)

```dart
void main() {
  int a = 10;
  int b = 20;
  String s1 = 'hello';
  String s2 = 'hello';
  String s3 = 'world';
  
  // ============ พื้นฐาน ============
  print(a == b);   // false - เท่ากัน
  print(a != b);   // true  - ไม่เท่ากัน
  print(a < b);    // true  - น้อยกว่า
  print(a > b);    // false - มากกว่า
  print(a <= b);   // true  - น้อยกว่าหรือเท่ากัน
  print(a >= b);   // false - มากกว่าหรือเท่ากัน
  
  // ============ String comparison ============
  print(s1 == s2);  // true  - เปรียบเทียบเนื้อหา (value equality)
  print(s1 == s3);  // false
  
  // String ordering (alphabetical/lexicographic):
  print(s1.compareTo(s3));  // -1 (h < w)
  print(s3.compareTo(s1));  // 1  (w > h)
  print(s1.compareTo(s2));  // 0  (เท่ากัน)
  
  // ============ Object identity ============
  List<int> list1 = [1, 2, 3];
  List<int> list2 = [1, 2, 3];
  List<int> list3 = list1; // ชี้ไปที่ object เดียวกัน
  
  print(list1 == list2);      // false (คนละ object)
  print(list1 == list3);      // true  (object เดียวกัน)
  print(identical(list1, list2)); // false
  print(identical(list1, list3)); // true
  
  // ============ Type comparison ============
  dynamic val = 42;
  print(val is int);     // true
  print(val is double);  // false
  print(val is num);     // true
  print(val is! String); // true
  
  // ============ Comparable interface ============
  // class สามารถ implement Comparable เพื่อใช้ < > ได้
  print('apple'.compareTo('banana')); // -1
}
```

---

## 3. Logical Operators (ตัวดำเนินการทางตรรกะ)

```dart
void main() {
  bool a = true;
  bool b = false;
  
  // ============ พื้นฐาน ============
  print(a && b);  // false - AND (ทั้งคู่ต้องเป็น true)
  print(a || b);  // true  - OR (อย่างน้อยหนึ่งเป็น true)
  print(!a);      // false - NOT (กลับค่า)
  print(!b);      // true
  
  // ============ Truth table ============
  // AND (&&):
  // true  && true  = true
  // true  && false = false
  // false && true  = false
  // false && false = false
  
  // OR (||):
  // true  || true  = true
  // true  || false = true
  // false || true  = true
  // false || false = false
  
  // ============ Short-circuit evaluation ============
  // && หยุดประเมินทันทีที่เจอ false แรก:
  int counter = 0;
  bool result = false && (++counter > 0); // counter ไม่เพิ่ม!
  print(counter); // 0 (++counter ไม่ถูกประเมิน)
  
  // || หยุดประเมินทันทีที่เจอ true แรก:
  counter = 0;
  result = true || (++counter > 0); // counter ไม่เพิ่ม!
  print(counter); // 0
  
  // ============ ตัวอย่างการใช้งาน ============
  int age = 25;
  bool hasLicense = true;
  bool hasCar = false;
  
  // ตรวจสอบเงื่อนไขหลายๆ อย่าง:
  bool canDrive = age >= 18 && hasLicense;
  print('ขับรถได้: $canDrive'); // true
  
  bool canRent = age >= 21 && (hasLicense || hasCar);
  print('เช่ารถได้: $canRent'); // true
  
  // ============ Null safety กับ logical ============
  String? name = null;
  bool isValidName = name != null && name.isNotEmpty;
  print('ชื่อถูกต้อง: $isValidName'); // false
  
  name = 'Dart';
  isValidName = name != null && name.isNotEmpty;
  print('ชื่อถูกต้อง: $isValidName'); // true
}
```

---

## 4. Bitwise Operators (ตัวดำเนินการระดับบิต)

```dart
void main() {
  int a = 0xF0; // 11110000 = 240
  int b = 0x0F; // 00001111 = 15
  
  // ============ Bitwise operations ============
  print(a & b);   // 0  - AND (00000000)
  print(a | b);   // 255 - OR (11111111)
  print(a ^ b);   // 255 - XOR (11111111)
  print(~a);      // -241 - NOT (bitwise complement)
  
  // ============ Shift operators ============
  int x = 1; // 00000001
  
  print(x << 1);  // 2  - shift left 1 bit (00000010)
  print(x << 2);  // 4  - shift left 2 bits (00000100)
  print(x << 3);  // 8  - shift left 3 bits (00001000)
  
  int y = 8; // 00001000
  print(y >> 1);  // 4  - shift right 1 bit (00000100)
  print(y >> 2);  // 2  - shift right 2 bits (00000010)
  print(y >> 3);  // 1  - shift right 3 bits (00000001)
  
  // ============ Use cases ============
  
  // 1. Permission/Flag system:
  const int READ    = 1 << 0; // 001 = 1
  const int WRITE   = 1 << 1; // 010 = 2
  const int EXECUTE = 1 << 2; // 100 = 4
  
  int userPermission = READ | WRITE; // 011 = 3
  
  bool canRead    = (userPermission & READ)    != 0; // true
  bool canWrite   = (userPermission & WRITE)   != 0; // true
  bool canExecute = (userPermission & EXECUTE) != 0; // false
  
  print('Read: $canRead, Write: $canWrite, Execute: $canExecute');
  
  // เพิ่ม permission:
  userPermission |= EXECUTE; // เพิ่ม execute
  print('New permission: $userPermission'); // 7 (111)
  
  // ลบ permission:
  userPermission &= ~WRITE; // ลบ write
  print('After remove write: $userPermission'); // 5 (101)
  
  // 2. ตรวจสอบเลขคู่/คี่ (เร็วกว่า %):
  for (int i = 1; i <= 8; i++) {
    String type = (i & 1) == 0 ? 'คู่' : 'คี่';
    print('$i: $type');
  }
  
  // 3. คูณ/หาร ด้วย 2 (เร็วกว่า * /:
  int n = 5;
  print(n << 1); // 10 (n * 2)
  print(n << 2); // 20 (n * 4)
  print(n >> 1); // 2  (n / 2, integer)
}
```

---

## 5. Assignment Operators

```dart
void main() {
  int x = 10;
  
  // ============ Compound assignment ============
  x += 5;   // x = x + 5 = 15
  x -= 3;   // x = x - 3 = 12
  x *= 2;   // x = x * 2 = 24
  x ~/= 4;  // x = x ~/ 4 = 6
  x %= 4;   // x = x % 4 = 2
  
  print(x); // 2
  
  // String:
  String s = 'Hello';
  s += ' World'; // s = s + ' World'
  print(s); // Hello World
  
  // ============ Null-aware assignment ============
  String? name = null;
  name ??= 'Default'; // กำหนดเฉพาะเมื่อ null
  print(name); // Default
  
  name ??= 'Other'; // ไม่กำหนดเพราะไม่ null
  print(name); // Default (ไม่เปลี่ยน)
  
  // ============ Bitwise assignment ============
  int flags = 0;
  flags |= 0x01;  // เพิ่ม flag
  flags &= ~0x01; // ลบ flag
  flags ^= 0x02;  // toggle flag
  flags <<= 1;    // shift left
  flags >>= 1;    // shift right
  
  // ============ Destructuring assignment (Dart 3.0+) ============
  // Pattern matching (เรียนเพิ่มใน Part 04)
  var (a, b) = (1, 2);  // Tuple destructuring
  print('a=$a, b=$b');  // a=1, b=2
  
  var [first, second, ...rest] = [1, 2, 3, 4, 5];
  print('first=$first, rest=$rest'); // first=1, rest=[3, 4, 5]
  
  var {'name': userName, 'age': userAge} = {'name': 'Alice', 'age': 30};
  print('$userName is $userAge'); // Alice is 30
}
```

---

## 6. Conditional Expressions

```dart
void main() {
  // ============ Ternary operator ============
  // condition ? trueValue : falseValue
  
  int age = 20;
  String status = age >= 18 ? 'ผู้ใหญ่' : 'เยาวชน';
  print(status); // ผู้ใหญ่
  
  int score = 75;
  String grade = score >= 90 ? 'A' :
                 score >= 80 ? 'B' :
                 score >= 70 ? 'C' :
                 score >= 60 ? 'D' : 'F';
  print('Grade: $grade'); // C
  
  // ============ If-null operator (??) ============
  String? name = null;
  String displayName = name ?? 'ไม่ระบุชื่อ';
  print(displayName); // ไม่ระบุชื่อ
  
  // ============ Conditional member access (?.) ============
  String? str = null;
  int? len = str?.length;  // null (ไม่ crash)
  print(len); // null
  
  str = 'hello';
  len = str?.length;
  print(len); // 5
  
  // ============ Conditional invocation ============
  List<int>? list = null;
  list?.add(1); // ไม่ทำอะไรถ้า null
  
  list = [1, 2, 3];
  list?.add(4); // เพิ่ม 4
  print(list); // [1, 2, 3, 4]
  
  // ============ switch expression (Dart 3.0+) ============
  int day = 3;
  String dayName = switch (day) {
    1 => 'จันทร์',
    2 => 'อังคาร',
    3 => 'พุธ',
    4 => 'พฤหัสบดี',
    5 => 'ศุกร์',
    6 => 'เสาร์',
    7 => 'อาทิตย์',
    _ => 'ไม่รู้จัก',
  };
  print(dayName); // พุธ
  
  // ============ Pattern-based conditionals (Dart 3.0+) ============
  Object value = 42;
  String description = switch (value) {
    int n when n < 0 => 'จำนวนลบ',
    int n when n == 0 => 'ศูนย์',
    int n when n > 0 => 'จำนวนบวก',
    String s => 'ข้อความ: $s',
    _ => 'ไม่รู้จัก',
  };
  print(description); // จำนวนบวก
}
```

---

## 7. Cascade Notation (..)

```dart
class Person {
  String name = '';
  int age = 0;
  String email = '';
  
  void setName(String n) => name = n;
  void setAge(int a) => age = a;
  void setEmail(String e) => email = e;
  
  void introduce() {
    print('ฉันคือ $name, อายุ $age, email: $email');
  }
}

class FlutterWidget {
  List<String> properties = [];
  
  FlutterWidget addColor(String color) {
    properties.add('color: $color');
    return this;
  }
  
  FlutterWidget addPadding(double value) {
    properties.add('padding: $value');
    return this;
  }
  
  FlutterWidget addBorder(double radius) {
    properties.add('borderRadius: $radius');
    return this;
  }
  
  @override
  String toString() => properties.join(', ');
}

void main() {
  // ============ ไม่ใช้ cascade ============
  Person person1 = Person();
  person1.setName('Alice');
  person1.setAge(30);
  person1.setEmail('alice@example.com');
  person1.introduce();
  
  // ============ ใช้ cascade (..) ============
  Person person2 = Person()
    ..setName('Bob')
    ..setAge(25)
    ..setEmail('bob@example.com')
    ..introduce(); // เรียก method และ return person2
  
  // ============ Cascade กับ properties ============
  Person person3 = Person()
    ..name = 'Charlie'  // set property
    ..age = 35
    ..email = 'charlie@example.com';
  person3.introduce();
  
  // ============ Null-aware cascade (?.. ) ============
  Person? nullPerson = null;
  nullPerson?..name = 'Dave'; // ไม่ทำอะไรถ้า null
  
  nullPerson = Person();
  nullPerson
    ?..name = 'Eve'
    ..age = 28;
  nullPerson.introduce(); // Eve
  
  // ============ ใช้กับ List/Map ============
  List<int> numbers = []
    ..add(1)
    ..add(2)
    ..add(3)
    ..addAll([4, 5, 6])
    ..sort()
    ..removeWhere((n) => n % 2 == 0); // ลบเลขคู่
  print(numbers); // [1, 3, 5]
  
  // ============ StringBuilder-style ============
  StringBuffer sb = StringBuffer()
    ..write('Hello')
    ..write(' ')
    ..write('World')
    ..writeln('!');
  print(sb); // Hello World!\n
}
```

---

## 8. Spread Operator (...)

```dart
void main() {
  // ============ List spread ============
  List<int> a = [1, 2, 3];
  List<int> b = [4, 5, 6];
  
  List<int> combined = [...a, ...b];
  print(combined); // [1, 2, 3, 4, 5, 6]
  
  // เพิ่มค่าพร้อมกัน:
  List<int> withExtra = [0, ...a, ...b, 7];
  print(withExtra); // [0, 1, 2, 3, 4, 5, 6, 7]
  
  // ============ Null-aware spread (?...) ============
  List<int>? maybeList = null;
  List<int> safe = [1, 2, ...?maybeList, 3];
  print(safe); // [1, 2, 3] (ไม่ crash)
  
  maybeList = [10, 20];
  safe = [1, 2, ...?maybeList, 3];
  print(safe); // [1, 2, 10, 20, 3]
  
  // ============ Map spread ============
  Map<String, int> scores1 = {'Alice': 90, 'Bob': 85};
  Map<String, int> scores2 = {'Charlie': 88, 'Dave': 92};
  
  Map<String, int> allScores = {...scores1, ...scores2};
  print(allScores); // {Alice: 90, Bob: 85, Charlie: 88, Dave: 92}
  
  // Override ค่าด้วย spread:
  Map<String, int> updated = {
    ...scores1,
    'Bob': 95, // override Bob's score
    ...scores2,
  };
  print(updated); // {Alice: 90, Bob: 95, Charlie: 88, Dave: 92}
  
  // ============ Collection if ============
  bool showAdmin = true;
  List<String> menu = [
    'หน้าแรก',
    'โปรไฟล์',
    if (showAdmin) 'ระบบจัดการ', // เพิ่มเฉพาะเมื่อ condition true
    'ออกจากระบบ',
  ];
  print(menu); // [หน้าแรก, โปรไฟล์, ระบบจัดการ, ออกจากระบบ]
  
  showAdmin = false;
  List<String> menu2 = [
    'หน้าแรก',
    'โปรไฟล์',
    if (showAdmin) 'ระบบจัดการ', // ไม่เพิ่ม
    'ออกจากระบบ',
  ];
  print(menu2); // [หน้าแรก, โปรไฟล์, ออกจากระบบ]
  
  // ============ Collection for ============
  List<int> squares = [
    for (int i = 1; i <= 5; i++) i * i
  ];
  print(squares); // [1, 4, 9, 16, 25]
  
  List<String> names = ['Alice', 'Bob', 'Charlie'];
  List<String> greetings = [
    for (String name in names) 'สวัสดี $name!'
  ];
  print(greetings); // [สวัสดี Alice!, สวัสดี Bob!, สวัสดี Charlie!]
  
  // รวมกัน:
  List<Widget> widgets = [
    // ใน Flutter:
    // for (Product p in products)
    //   if (p.inStock) ProductCard(product: p),
  ];
}
```

---

## 9. Operator Precedence (ลำดับความสำคัญ)

```dart
void main() {
  // ลำดับสูงไปต่ำ:
  // 1. ()  - Parentheses (วงเล็บ)
  // 2. --x, ++x, -x, !x  - Unary prefix
  // 3. x++, x--  - Unary postfix
  // 4. *, /, ~/, %  - Multiplicative
  // 5. +, -  - Additive
  // 6. <<, >>, >>>  - Bitwise shift
  // 7. &  - Bitwise AND
  // 8. ^  - Bitwise XOR
  // 9. |  - Bitwise OR
  // 10. <, >, <=, >=, is, is!, as  - Relational
  // 11. ==, !=  - Equality
  // 12. &&  - Logical AND
  // 13. ||  - Logical OR
  // 14. ??  - If-null
  // 15. ?:  - Conditional
  // 16. =, +=, -= (etc.)  - Assignment
  
  // ตัวอย่าง:
  print(2 + 3 * 4);     // 14 (ไม่ใช่ 20! * ก่อน +)
  print((2 + 3) * 4);   // 20 (วงเล็บก่อน)
  
  print(10 > 5 && 3 < 8);   // true && true = true
  print(10 > 5 || 3 > 8);   // true || false = true
  print(!(10 > 5));           // !true = false
  
  // ระวัง:
  int x = 5;
  print(x++ * 2);  // 10 (ใช้ค่า 5 ก่อน แล้วเพิ่ม)
  print(x);        // 6
  print(++x * 2);  // 14 (เพิ่มก่อน แล้วใช้ค่า 7)
  print(x);        // 7
  
  // ??  มีความสำคัญต่ำกว่า + แต่สูงกว่า =:
  String? nullStr = null;
  String result = nullStr ?? 'default'; // กำหนดทั้ง string
  print(result); // default
  
  // แนะนำ: ใช้วงเล็บเพื่อความชัดเจน
  bool confusing = true || false && false; // true (เหมือน true || (false && false))
  bool clear = true || (false && false);  // true (ชัดเจน)
}
```

---

## 10. Operator Overloading

```dart
class Vector {
  final double x;
  final double y;
  
  const Vector(this.x, this.y);
  
  // Override + operator
  Vector operator +(Vector other) {
    return Vector(x + other.x, y + other.y);
  }
  
  // Override - operator
  Vector operator -(Vector other) {
    return Vector(x - other.x, y - other.y);
  }
  
  // Override * operator (scalar multiplication)
  Vector operator *(double scalar) {
    return Vector(x * scalar, y * scalar);
  }
  
  // Override == operator
  @override
  bool operator ==(Object other) {
    if (other is Vector) {
      return x == other.x && y == other.y;
    }
    return false;
  }
  
  // Override [] operator (index)
  double operator [](int index) {
    switch (index) {
      case 0: return x;
      case 1: return y;
      default: throw RangeError('Index must be 0 or 1');
    }
  }
  
  // Unary minus
  Vector operator -() {
    return Vector(-x, -y);
  }
  
  // Magnitude (ขนาดของ vector)
  double get magnitude => sqrt(x * x + y * y);
  
  @override
  String toString() => 'Vector($x, $y)';
  
  @override
  int get hashCode => Object.hash(x, y);
}

import 'dart:math';

void main() {
  Vector v1 = const Vector(3, 4);
  Vector v2 = const Vector(1, 2);
  
  print(v1 + v2);  // Vector(4.0, 6.0)
  print(v1 - v2);  // Vector(2.0, 2.0)
  print(v1 * 2);   // Vector(6.0, 8.0)
  print(-v1);      // Vector(-3.0, -4.0)
  print(v1 == v2); // false
  print(v1 == const Vector(3, 4)); // true
  print(v1[0]);    // 3.0 (x)
  print(v1[1]);    // 4.0 (y)
  print(v1.magnitude); // 5.0
}
```

---

## 11. ตัวอย่างจริง: Expression Parser

```dart
// expression_evaluator.dart - คำนวณ expression จาก string

class ExpressionEvaluator {
  // คำนวณ expression ง่ายๆ (เพียง 2 ตัวเลข)
  double evaluate(String expression) {
    expression = expression.trim();
    
    // หา operator
    List<String> operators = ['+', '-', '*', '/', '%'];
    
    for (String op in operators) {
      int idx = expression.lastIndexOf(op);
      if (idx > 0) {
        String left = expression.substring(0, idx).trim();
        String right = expression.substring(idx + 1).trim();
        
        double? a = double.tryParse(left);
        double? b = double.tryParse(right);
        
        if (a != null && b != null) {
          return switch (op) {
            '+' => a + b,
            '-' => a - b,
            '*' => a * b,
            '/' => b != 0 ? a / b : throw ArgumentError('Division by zero'),
            '%' => a % b,
            _ => throw ArgumentError('Unknown operator: $op'),
          };
        }
      }
    }
    
    // ถ้าเป็นตัวเลขเดี่ยว
    double? single = double.tryParse(expression);
    if (single != null) return single;
    
    throw FormatException('Invalid expression: $expression');
  }
  
  // ตรวจสอบความถูกต้อง
  bool isValid(String expression) {
    try {
      evaluate(expression);
      return true;
    } catch (_) {
      return false;
    }
  }
}

void main() {
  ExpressionEvaluator eval = ExpressionEvaluator();
  
  List<String> expressions = [
    '10 + 5',
    '20 - 8',
    '6 * 7',
    '100 / 4',
    '17 % 5',
    '0',
  ];
  
  for (String expr in expressions) {
    try {
      double result = eval.evaluate(expr);
      print('$expr = $result');
    } catch (e) {
      print('Error: $e');
    }
  }
  
  // Test null safety และ conditional:
  String? input = null;
  String expression = input ?? '10 + 5';
  double? result;
  
  try {
    result = eval.evaluate(expression);
  } catch (_) {
    result = null;
  }
  
  print('Result: ${result?.toStringAsFixed(2) ?? "error"}');
  
  // Bitwise operations:
  int flags = 0;
  const int FLAG_A = 1 << 0;
  const int FLAG_B = 1 << 1;
  const int FLAG_C = 1 << 2;
  
  flags |= FLAG_A;  // เปิด A
  flags |= FLAG_C;  // เปิด C
  
  print('Flags binary: ${flags.toRadixString(2).padLeft(3, '0')}'); // 101
  print('Flag A: ${(flags & FLAG_A) != 0}'); // true
  print('Flag B: ${(flags & FLAG_B) != 0}'); // false
  print('Flag C: ${(flags & FLAG_C) != 0}'); // true
}
```

---

## 12. สรุป Part 03

สิ่งที่เรียนรู้:
- ✅ Arithmetic operators: +, -, *, /, ~/, %, ++, --
- ✅ Comparison operators: ==, !=, <, >, <=, >=
- ✅ Logical operators: &&, ||, ! และ short-circuit evaluation
- ✅ Bitwise operators: &, |, ^, ~, <<, >>
- ✅ Assignment operators: =, +=, -=, ??=
- ✅ Conditional expressions: ternary (?:), if-null (??)
- ✅ Cascade notation (..) สำหรับ method chaining
- ✅ Spread operator (...) และ collection if/for
- ✅ Operator overloading
- ✅ Operator precedence

---

## ➡️ Part ถัดไป
**Part 04: Control Flow - if/else, switch, loops**

เราจะเรียนรู้:
- if, else if, else
- switch statements (Dart 3.x patterns)
- for, while, do-while loops
- break, continue, labels
- Pattern matching (Dart 3.0+)
