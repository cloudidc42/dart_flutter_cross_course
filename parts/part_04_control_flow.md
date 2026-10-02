# Part 04: Control Flow - if/else, switch, loops

## 🎯 เป้าหมายของ Part นี้
- if, else if, else - การตัดสินใจ
- switch statements และ Pattern matching (Dart 3.0+)
- for, while, do-while loops
- break, continue, return
- Labels และ nested loops
- Iterable methods แทน loops

---

## 1. if / else if / else

```dart
void main() {
  // ============ if พื้นฐาน ============
  int score = 75;
  
  if (score >= 90) {
    print('เกรด A - ดีเยี่ยม!');
  } else if (score >= 80) {
    print('เกรด B - ดีมาก');
  } else if (score >= 70) {
    print('เกรด C - ดี');
  } else if (score >= 60) {
    print('เกรด D - ผ่าน');
  } else {
    print('เกรด F - ไม่ผ่าน');
  }
  
  // ============ if เป็น expression (Dart 3.0+) ============
  // ใช้ switch expression แทน if-else chain ที่ยาว
  
  // ============ Nested if ============
  int age = 20;
  bool hasMembership = true;
  double balance = 150.0;
  
  if (age >= 18) {
    if (hasMembership) {
      if (balance >= 100) {
        print('สามารถซื้อสินค้าพิเศษได้');
      } else {
        print('ยอดเงินไม่พอ');
      }
    } else {
      print('ต้องสมัครสมาชิกก่อน');
    }
  } else {
    print('อายุไม่ถึง 18 ปี');
  }
  
  // ============ ตัวอย่างการใช้ในชีวิตจริง ============
  
  // ตรวจสอบ password strength:
  String password = 'Dart@2024!';
  
  bool hasUppercase = password.contains(RegExp(r'[A-Z]'));
  bool hasLowercase = password.contains(RegExp(r'[a-z]'));
  bool hasDigit = password.contains(RegExp(r'[0-9]'));
  bool hasSpecial = password.contains(RegExp(r'[!@#$%^&*]'));
  bool isLongEnough = password.length >= 8;
  
  int strength = 0;
  if (hasUppercase) strength++;
  if (hasLowercase) strength++;
  if (hasDigit) strength++;
  if (hasSpecial) strength++;
  if (isLongEnough) strength++;
  
  String passwordStrength = switch (strength) {
    >= 5 => 'แข็งแกร่งมาก 💪',
    >= 4 => 'แข็งแกร่ง 👍',
    >= 3 => 'ปานกลาง 😐',
    >= 2 => 'อ่อนแอ ⚠️',
    _ => 'อ่อนแอมาก ❌',
  };
  
  print('Password strength: $passwordStrength');
}
```

---

## 2. Switch Statement และ Pattern Matching

```dart
void main() {
  // ============ Switch Statement (Classic) ============
  String day = 'Wednesday';
  
  switch (day) {
    case 'Monday':
    case 'Tuesday':
    case 'Wednesday':
    case 'Thursday':
    case 'Friday':
      print('วันทำงาน');
      break;
    case 'Saturday':
    case 'Sunday':
      print('วันหยุด');
      break;
    default:
      print('ไม่รู้จักวัน');
  }
  
  // ============ Switch Expression (Dart 3.0+) ============
  int month = 3;
  String monthName = switch (month) {
    1 => 'มกราคม',
    2 => 'กุมภาพันธ์',
    3 => 'มีนาคม',
    4 => 'เมษายน',
    5 => 'พฤษภาคม',
    6 => 'มิถุนายน',
    7 => 'กรกฎาคม',
    8 => 'สิงหาคม',
    9 => 'กันยายน',
    10 => 'ตุลาคม',
    11 => 'พฤศจิกายน',
    12 => 'ธันวาคม',
    _ => 'ไม่รู้จักเดือน',
  };
  print(monthName); // มีนาคม
  
  // ============ Pattern Matching (Dart 3.0+) ============
  
  // Type patterns:
  Object value = 42;
  String description = switch (value) {
    int n when n < 0  => 'จำนวนลบ: $n',
    int n when n == 0 => 'ศูนย์',
    int n             => 'จำนวนบวก: $n',
    double d          => 'ทศนิยม: $d',
    String s          => 'ข้อความ: $s',
    _                 => 'ไม่รู้จัก',
  };
  print(description); // จำนวนบวก: 42
  
  // Record patterns:
  (String, int) person = ('Alice', 30);
  String result = switch (person) {
    (String name, int age) when age < 18 => '$name เป็นเยาวชน',
    (String name, int age) when age < 60 => '$name เป็นผู้ใหญ่',
    (String name, _)                     => '$name เป็นผู้สูงอายุ',
  };
  print(result); // Alice เป็นผู้ใหญ่
  
  // List patterns:
  List<int> numbers = [1, 2, 3];
  String listResult = switch (numbers) {
    [] => 'ว่างเปล่า',
    [int x] => 'มีหนึ่งตัว: $x',
    [int x, int y] => 'มีสองตัว: $x, $y',
    [int first, ..., int last] => 'หลายตัว: $first....$last',
  };
  print(listResult); // หลายตัว: 1....3
  
  // Map patterns:
  Map<String, dynamic> user = {'name': 'Bob', 'role': 'admin'};
  String access = switch (user) {
    {'role': 'admin'} => 'เข้าถึงทุกอย่าง',
    {'role': 'moderator'} => 'เข้าถึงบางส่วน',
    {'role': String role} => 'Role: $role',
    _ => 'ไม่มีสิทธิ์',
  };
  print(access); // เข้าถึงทุกอย่าง
}
```

---

## 3. for Loops

```dart
void main() {
  // ============ Classic for loop ============
  for (int i = 0; i < 5; i++) {
    print('i = $i');
  }
  
  // ============ for-in loop (for each) ============
  List<String> fruits = ['แอปเปิ้ล', 'กล้วย', 'มะม่วง'];
  
  for (String fruit in fruits) {
    print('ผลไม้: $fruit');
  }
  
  // ============ forEach method ============
  fruits.forEach((fruit) {
    print('Fruit: $fruit');
  });
  
  // หรือแบบสั้น:
  fruits.forEach(print);
  
  // ============ for with index ============
  for (int i = 0; i < fruits.length; i++) {
    print('${i + 1}. ${fruits[i]}');
  }
  
  // ============ asMap() สำหรับ index ============
  fruits.asMap().forEach((index, fruit) {
    print('${index + 1}. $fruit');
  });
  
  // ============ Map iteration ============
  Map<String, int> scores = {'Alice': 90, 'Bob': 85, 'Charlie': 92};
  
  // keys
  for (String name in scores.keys) {
    print('Name: $name');
  }
  
  // values
  for (int score in scores.values) {
    print('Score: $score');
  }
  
  // entries
  for (MapEntry<String, int> entry in scores.entries) {
    print('${entry.key}: ${entry.value}');
  }
  
  // ============ Nested for ============
  // ตาราง multiplication:
  for (int i = 1; i <= 3; i++) {
    for (int j = 1; j <= 3; j++) {
      print('$i x $j = ${i * j}');
    }
  }
  
  // ============ for กับ String ============
  String word = 'Dart';
  for (int i = 0; i < word.length; i++) {
    print(word[i]);
  }
  
  // หรือใช้ runes:
  for (int rune in 'สวัสดี'.runes) {
    print(String.fromCharCode(rune));
  }
}
```

---

## 4. while และ do-while

```dart
void main() {
  // ============ while loop ============
  int count = 0;
  while (count < 5) {
    print('Count: $count');
    count++;
  }
  
  // ============ do-while loop (ทำอย่างน้อย 1 ครั้ง) ============
  int num = 10;
  do {
    print('Num: $num');
    num--;
  } while (num > 8);
  // Output: 10, 9
  
  // ============ ตัวอย่าง: Binary search ============
  List<int> sorted = [1, 3, 5, 7, 9, 11, 13, 15, 17, 19];
  int target = 11;
  
  int left = 0;
  int right = sorted.length - 1;
  int result = -1;
  
  while (left <= right) {
    int mid = (left + right) ~/ 2;
    
    if (sorted[mid] == target) {
      result = mid;
      break;
    } else if (sorted[mid] < target) {
      left = mid + 1;
    } else {
      right = mid - 1;
    }
  }
  
  print('พบ $target ที่ index $result'); // พบ 11 ที่ index 5
  
  // ============ ตัวอย่าง: Input validation ============
  // (ในแอปจริง จะรับ input จาก user)
  List<String> testInputs = ['', '   ', 'abc', ''];
  int idx = 0;
  
  String? validInput;
  do {
    String input = testInputs[idx++].trim();
    if (input.isNotEmpty) {
      validInput = input;
    }
    if (idx >= testInputs.length) break;
  } while (validInput == null);
  
  print('Valid input: $validInput'); // abc
  
  // ============ Infinite loop + break ============
  int iterations = 0;
  while (true) {
    iterations++;
    if (iterations >= 5) break; // หยุดที่ 5 ครั้ง
  }
  print('Iterations: $iterations'); // 5
}
```

---

## 5. break, continue, และ Labels

```dart
void main() {
  // ============ break ============
  // หยุด loop ทันที
  for (int i = 0; i < 10; i++) {
    if (i == 5) break;
    print(i); // 0, 1, 2, 3, 4
  }
  
  // ============ continue ============
  // ข้ามรอบปัจจุบัน ไปรอบถัดไป
  for (int i = 0; i < 10; i++) {
    if (i % 2 == 0) continue; // ข้ามเลขคู่
    print(i); // 1, 3, 5, 7, 9
  }
  
  // ============ Labels ============
  // ใช้กับ nested loops
  
  outerLoop:
  for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
      if (i == 1 && j == 1) {
        break outerLoop; // หยุด outer loop!
      }
      print('($i, $j)');
    }
  }
  // Output: (0,0), (0,1), (0,2), (1,0)
  
  search:
  for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
      if (i == 1 && j == 1) {
        continue search; // ข้ามไปรอบถัดไปของ outer loop
      }
      print('[$i, $j]');
    }
  }
  // Output: [0,0],[0,1],[0,2],[1,0],[2,0],[2,1],[2,2]
  
  // ============ break ใน switch ============
  int x = 2;
  switch (x) {
    case 1:
      print('หนึ่ง');
      break;
    case 2:
      print('สอง');
      break; // จำเป็นต้องมี break ใน switch (classic style)
    case 3:
      print('สาม');
      break;
  }
  
  // ============ ตัวอย่าง: หาจำนวนเฉพาะ ============
  print('\nจำนวนเฉพาะ 1-50:');
  for (int n = 2; n <= 50; n++) {
    bool isPrime = true;
    
    for (int i = 2; i * i <= n; i++) {
      if (n % i == 0) {
        isPrime = false;
        break; // ไม่ต้องตรวจสอบต่อ
      }
    }
    
    if (isPrime) {
      print(n);
    }
  }
}
```

---

## 6. Iterable Methods (แทน loops)

```dart
void main() {
  List<int> numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
  
  // ============ map - แปลงทุก element ============
  List<int> doubled = numbers.map((n) => n * 2).toList();
  print(doubled); // [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
  
  List<String> asStrings = numbers.map((n) => 'Item $n').toList();
  print(asStrings); // [Item 1, Item 2, ...]
  
  // ============ where - กรอง elements ============
  List<int> evens = numbers.where((n) => n % 2 == 0).toList();
  print(evens); // [2, 4, 6, 8, 10]
  
  List<int> greaterThan5 = numbers.where((n) => n > 5).toList();
  print(greaterThan5); // [6, 7, 8, 9, 10]
  
  // ============ reduce - รวมเป็นค่าเดียว ============
  int sum = numbers.reduce((acc, n) => acc + n);
  print(sum); // 55
  
  int product = numbers.reduce((acc, n) => acc * n);
  print(product); // 3628800
  
  // ============ fold - เหมือน reduce แต่กำหนด initial value ============
  int sumWithFold = numbers.fold(0, (acc, n) => acc + n);
  print(sumWithFold); // 55
  
  // fold ใช้ต่าง type ได้:
  String concat = numbers.fold('', (acc, n) => acc + '$n,');
  print(concat); // 1,2,3,4,5,6,7,8,9,10,
  
  // ============ any - ตรวจสอบว่ามีสักตัว ============
  bool hasEven = numbers.any((n) => n % 2 == 0);
  print(hasEven); // true
  
  bool hasNegative = numbers.any((n) => n < 0);
  print(hasNegative); // false
  
  // ============ every - ตรวจสอบว่าทุกตัว ============
  bool allPositive = numbers.every((n) => n > 0);
  print(allPositive); // true
  
  bool allEven = numbers.every((n) => n % 2 == 0);
  print(allEven); // false
  
  // ============ firstWhere / lastWhere ============
  int firstEven = numbers.firstWhere((n) => n % 2 == 0);
  print(firstEven); // 2
  
  int lastEven = numbers.lastWhere((n) => n % 2 == 0);
  print(lastEven); // 10
  
  // orElse เมื่อหาไม่เจอ:
  int? found = numbers.firstWhereOrNull((n) => n > 100); // extension method
  // หรือ:
  int notFound = numbers.firstWhere(
    (n) => n > 100,
    orElse: () => -1,
  );
  print(notFound); // -1
  
  // ============ sort ============
  List<int> unsorted = [3, 1, 4, 1, 5, 9, 2, 6];
  unsorted.sort(); // sort ascending
  print(unsorted); // [1, 1, 2, 3, 4, 5, 6, 9]
  
  unsorted.sort((a, b) => b.compareTo(a)); // sort descending
  print(unsorted); // [9, 6, 5, 4, 3, 2, 1, 1]
  
  // sort objects:
  List<Map<String, dynamic>> students = [
    {'name': 'Bob', 'score': 85},
    {'name': 'Alice', 'score': 92},
    {'name': 'Charlie', 'score': 78},
  ];
  
  students.sort((a, b) => (b['score'] as int).compareTo(a['score'] as int));
  for (var s in students) {
    print('${s['name']}: ${s['score']}');
  }
  // Alice: 92, Bob: 85, Charlie: 78
  
  // ============ take / skip ============
  print(numbers.take(3).toList());  // [1, 2, 3]
  print(numbers.skip(7).toList());  // [8, 9, 10]
  
  // ============ zip (ใช้ package collection) ============
  // หรือทำเอง:
  List<String> names = ['Alice', 'Bob', 'Charlie'];
  List<int> ages = [30, 25, 35];
  
  List<String> pairs = [
    for (int i = 0; i < names.length; i++)
      '${names[i]}: ${ages[i]}'
  ];
  print(pairs); // [Alice: 30, Bob: 25, Charlie: 35]
  
  // ============ expand (flatMap) ============
  List<List<int>> nested = [[1, 2], [3, 4], [5, 6]];
  List<int> flat = nested.expand((list) => list).toList();
  print(flat); // [1, 2, 3, 4, 5, 6]
  
  // ============ distinct / unique ============
  List<int> withDupes = [1, 2, 2, 3, 3, 3, 4];
  List<int> unique = withDupes.toSet().toList();
  print(unique); // [1, 2, 3, 4]
}
```

---

## 7. Workshop: Pattern Matching Calculator

```dart
// grade_system.dart - ระบบให้เกรด

enum Grade { A, B, C, D, F }

class Student {
  final String name;
  final List<double> scores;
  
  const Student(this.name, this.scores);
  
  double get average {
    if (scores.isEmpty) return 0;
    return scores.fold(0.0, (a, b) => a + b) / scores.length;
  }
  
  Grade get grade => switch (average) {
    >= 90 => Grade.A,
    >= 80 => Grade.B,
    >= 70 => Grade.C,
    >= 60 => Grade.D,
    _     => Grade.F,
  };
  
  String get gradeDescription => switch (grade) {
    Grade.A => 'ดีเยี่ยม',
    Grade.B => 'ดีมาก',
    Grade.C => 'ดี',
    Grade.D => 'ผ่าน',
    Grade.F => 'ไม่ผ่าน',
  };
  
  bool get isPassed => grade != Grade.F;
}

void main() {
  List<Student> students = [
    Student('สมชาย', [85, 92, 88, 90]),
    Student('สมหญิง', [70, 65, 72, 68]),
    Student('วีระ', [55, 60, 48, 52]),
    Student('อรุณ', [95, 98, 92, 96]),
    Student('มณี', [78, 82, 75, 80]),
  ];
  
  // แสดงผล:
  print('=== ผลการเรียน ===');
  for (var student in students) {
    String status = student.isPassed ? '✅' : '❌';
    print('$status ${student.name}: '
          '${student.average.toStringAsFixed(1)} '
          '(${student.grade.name}) - ${student.gradeDescription}');
  }
  
  // สถิติ:
  List<Student> passed = students.where((s) => s.isPassed).toList();
  List<Student> failed = students.where((s) => !s.isPassed).toList();
  
  double classAverage = students.map((s) => s.average)
      .fold(0.0, (a, b) => a + b) / students.length;
  
  Student topStudent = students.reduce((a, b) => 
      a.average > b.average ? a : b);
  
  print('\n=== สรุป ===');
  print('นักเรียนทั้งหมด: ${students.length} คน');
  print('ผ่าน: ${passed.length} คน');
  print('ไม่ผ่าน: ${failed.length} คน');
  print('คะแนนเฉลี่ยชั้น: ${classAverage.toStringAsFixed(1)}');
  print('นักเรียนยอดเยี่ยม: ${topStudent.name} '
        '(${topStudent.average.toStringAsFixed(1)})');
  
  // Pattern matching advanced:
  for (var student in students) {
    String message = switch (student) {
      Student(grade: Grade.A) => '${student.name} สุดยอด! 🎉',
      Student(grade: Grade.F, name: String name) => 
          '$name ต้องเรียนเพิ่ม 📚',
      Student(average: >= 80) => '${student.name} ทำได้ดี 👍',
      _ => '${student.name} ยังพอไปได้',
    };
    print(message);
  }
}
```

---

## 8. สรุป Part 04

สิ่งที่เรียนรู้:
- ✅ if/else if/else - การตัดสินใจหลายเงื่อนไข
- ✅ switch statement (classic) และ switch expression (Dart 3.0+)
- ✅ Pattern matching - type, record, list, map patterns
- ✅ for loop, for-in, forEach
- ✅ while loop และ do-while
- ✅ break, continue, labels สำหรับ nested loops
- ✅ Iterable methods: map, where, reduce, fold, any, every, sort
- ✅ Pattern matching ใน switch expressions

---

## ➡️ Part ถัดไป
**Part 05: Functions และ Parameters**

เราจะเรียนรู้:
- Function declaration และ return types
- Optional parameters (named, positional)
- Default values
- Arrow functions
- Higher-order functions
- Closures
- Recursion
