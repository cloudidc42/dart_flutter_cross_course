# Part 07: OOP - Classes and Objects

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- สร้าง class และ object ได้อย่างถูกต้อง
- ใช้งาน constructors หลายรูปแบบ (default, named, factory, const)
- เข้าใจ instance/static fields และ methods
- ใช้ getters และ setters อย่างมีประสิทธิภาพ
- Override toString(), hashCode, == operator
- ปกป้องข้อมูลด้วย private members
- สร้าง BankAccount class จากโจทย์จริง

---

## 7.1 พื้นฐาน Class และ Object

Class คือ "แบบพิมพ์เขียว" ส่วน Object คือ "ผลิตภัณฑ์" ที่สร้างจากแบบพิมพ์เขียวนั้น

```dart
// การประกาศ class พื้นฐาน
class Person {
  // Fields (instance variables)
  String name;
  int age;
  String email;
  
  // Constructor
  Person(this.name, this.age, this.email);
  
  // Method
  void introduce() {
    print('สวัสดี ฉันชื่อ $name อายุ $age ปี');
  }
  
  String getInfo() {
    return 'ชื่อ: $name | อายุ: $age | Email: $email';
  }
}

void main() {
  // สร้าง object จาก class
  Person person1 = Person('สมชาย ใจดี', 25, 'somchai@email.com');
  Person person2 = Person('มานี รักเรียน', 30, 'manee@email.com');
  
  // เรียกใช้ method
  person1.introduce(); // สวัสดี ฉันชื่อ สมชาย ใจดี อายุ 25 ปี
  person2.introduce(); // สวัสดี ฉันชื่อ มานี รักเรียน อายุ 30 ปี
  
  // เข้าถึง fields
  print(person1.name);  // สมชาย ใจดี
  person1.age = 26;     // แก้ไข
  
  // Object เป็น reference type
  Person ref = person1; // ชี้ไปที่ object เดียวกัน
  ref.name = 'สมชาย แก้ไขแล้ว';
  print(person1.name); // สมชาย แก้ไขแล้ว (เปลี่ยนตาม ref)
}
```

---

## 7.2 Constructors

### Default Constructor

```dart
class Product {
  String name;
  double price;
  int stock;
  String category;
  
  // Default constructor
  Product(this.name, this.price, this.stock, this.category);
  
  // Initializer list - รันก่อน constructor body
  Product.withValidation(String name, double price, int stock, String category)
    : name = name,
      price = price > 0 ? price : 0,
      stock = stock >= 0 ? stock : 0,
      category = category.isNotEmpty ? category : 'ไม่ระบุ' {
    print('สร้าง product: $name');
  }
}

void main() {
  Product p1 = Product('กาแฟ', 60.0, 100, 'เครื่องดื่ม');
  print('${p1.name}: ฿${p1.price}');
  
  Product p2 = Product.withValidation('ชา', -10, -5, '');
  print('${p2.name}: ฿${p2.price}, สต็อก: ${p2.stock}, หมวด: ${p2.category}');
  // กาแฟ: ฿60.0
  // สร้าง product: ชา
  // ชา: ฿0.0, สต็อก: 0, หมวด: ไม่ระบุ
}
```

### Named Constructors

```dart
class Point {
  final double x;
  final double y;
  
  // Primary constructor
  Point(this.x, this.y);
  
  // Named constructor - จุดกำเนิด
  Point.origin() : x = 0, y = 0;
  
  // Named constructor - จาก Map
  Point.fromMap(Map<String, double> map)
    : x = map['x'] ?? 0,
      y = map['y'] ?? 0;
  
  // Named constructor - จาก List
  Point.fromList(List<double> list)
    : x = list[0],
      y = list[1];
  
  // Named constructor - Polar coordinates
  Point.polar(double radius, double angle)
    : x = radius * _cos(angle),
      y = radius * _sin(angle);
  
  // Helper static methods
  static double _cos(double angle) => angle; // simplified
  static double _sin(double angle) => angle; // simplified
  
  double distanceTo(Point other) {
    double dx = x - other.x;
    double dy = y - other.y;
    return (dx * dx + dy * dy);  // simplified, no sqrt
  }
  
  @override
  String toString() => 'Point($x, $y)';
}

void main() {
  Point p1 = Point(3.0, 4.0);
  Point p2 = Point.origin();
  Point p3 = Point.fromMap({'x': 5.0, 'y': 6.0});
  Point p4 = Point.fromList([7.0, 8.0]);
  
  print(p1); // Point(3.0, 4.0)
  print(p2); // Point(0.0, 0.0)
  print(p3); // Point(5.0, 6.0)
  print(p4); // Point(7.0, 8.0)
}
```

### Factory Constructor

Factory constructor ใช้เมื่อต้องการ:
1. Return existing instance (Singleton pattern)
2. Return subclass instance
3. ทำ validation ซับซ้อนก่อนสร้าง

```dart
class DatabaseConnection {
  // Singleton pattern - มี instance เดียวเท่านั้น
  static DatabaseConnection? _instance;
  
  final String host;
  final int port;
  final String database;
  bool _isConnected = false;
  
  // Private constructor
  DatabaseConnection._internal(this.host, this.port, this.database);
  
  // Factory constructor - คืน instance เดิมถ้ามี
  factory DatabaseConnection({
    String host = 'localhost',
    int port = 5432,
    String database = 'mydb',
  }) {
    _instance ??= DatabaseConnection._internal(host, port, database);
    return _instance!;
  }
  
  void connect() {
    _isConnected = true;
    print('เชื่อมต่อ $database ที่ $host:$port แล้ว');
  }
  
  void disconnect() {
    _isConnected = false;
    print('ตัดการเชื่อมต่อแล้ว');
  }
  
  bool get isConnected => _isConnected;
  
  @override
  String toString() => 'DB($host:$port/$database, connected: $_isConnected)';
}

// Factory constructor สำหรับสร้าง subclass
abstract class Shape {
  String get name;
  double get area;
  
  // Factory constructor ที่เลือก subclass ตาม type
  factory Shape.create(String type, Map<String, double> params) {
    switch (type) {
      case 'circle':
        return Circle(params['radius'] ?? 1.0);
      case 'rectangle':
        return Rectangle(params['width'] ?? 1.0, params['height'] ?? 1.0);
      default:
        throw ArgumentError('ไม่รู้จัก shape type: $type');
    }
  }
}

class Circle extends Shape {
  final double radius;
  Circle(this.radius);
  
  @override
  String get name => 'วงกลม';
  
  @override
  double get area => 3.14159 * radius * radius;
}

class Rectangle extends Shape {
  final double width;
  final double height;
  Rectangle(this.width, this.height);
  
  @override
  String get name => 'สี่เหลี่ยม';
  
  @override
  double get area => width * height;
}

void main() {
  // Singleton test
  DatabaseConnection db1 = DatabaseConnection(host: '10.0.0.1', database: 'shop');
  DatabaseConnection db2 = DatabaseConnection();
  
  print(identical(db1, db2)); // true - เป็น object เดียวกัน
  db1.connect();
  print(db2.isConnected); // true
  
  // Factory Shape
  Shape circle = Shape.create('circle', {'radius': 5.0});
  Shape rect = Shape.create('rectangle', {'width': 4.0, 'height': 6.0});
  
  print('${circle.name}: พื้นที่ = ${circle.area.toStringAsFixed(2)}');
  print('${rect.name}: พื้นที่ = ${rect.area.toStringAsFixed(2)}');
}
```

### Const Constructor

```dart
// const constructor: ทุก field ต้องเป็น final
class Color {
  final int red;
  final int green;
  final int blue;
  final double alpha;
  
  const Color(this.red, this.green, this.blue, {this.alpha = 1.0});
  
  // Named const constructors
  static const Color red = Color(255, 0, 0);
  static const Color green = Color(0, 255, 0);
  static const Color blue = Color(0, 0, 255);
  static const Color white = Color(255, 255, 255);
  static const Color black = Color(0, 0, 0);
  static const Color transparent = Color(0, 0, 0, alpha: 0.0);
  
  String toHex() {
    return '#${red.toRadixString(16).padLeft(2, '0')}'
           '${green.toRadixString(16).padLeft(2, '0')}'
           '${blue.toRadixString(16).padLeft(2, '0')}';
  }
  
  @override
  String toString() => 'Color(r:$red, g:$green, b:$blue, a:$alpha)';
}

void main() {
  // const object - สร้าง compile time
  const Color primaryColor = Color(33, 150, 243);
  const Color accentColor = Color(255, 87, 34);
  
  // ใช้ predefined colors
  print(Color.red);   // Color(r:255, g:0, b:0, a:1.0)
  print(Color.blue.toHex()); // #0000ff
  
  // const list of const objects
  const List<Color> palette = [
    Color(255, 0, 0),
    Color(0, 255, 0),
    Color(0, 0, 255),
  ];
  
  // const objects ที่เหมือนกันจะเป็น instance เดียวกัน
  const c1 = Color(255, 0, 0);
  const c2 = Color(255, 0, 0);
  print(identical(c1, c2)); // true! เป็น instance เดียวกัน
}
```

---

## 7.3 Instance Fields และ Methods

```dart
class Employee {
  // Instance fields
  String name;
  String department;
  double baseSalary;
  List<String> skills;
  DateTime hireDate;
  
  // Instance counter (ควรใช้ static แต่ตัวอย่างนี้แสดง instance)
  int overtimeHours = 0;
  
  Employee({
    required this.name,
    required this.department,
    required this.baseSalary,
    List<String>? skills,
    DateTime? hireDate,
  }) : skills = skills ?? [],
       hireDate = hireDate ?? DateTime.now();
  
  // Computed property (getter)
  int get yearsOfService {
    return DateTime.now().difference(hireDate).inDays ~/ 365;
  }
  
  double get totalSalary {
    double overtimePay = baseSalary / 30 / 8 * overtimeHours * 1.5;
    return baseSalary + overtimePay;
  }
  
  // Instance methods
  void addSkill(String skill) {
    if (!skills.contains(skill)) {
      skills.add(skill);
      print('เพิ่ม skill "$skill" ให้ $name แล้ว');
    }
  }
  
  void addOvertimeHours(int hours) {
    if (hours > 0) {
      overtimeHours += hours;
    }
  }
  
  void promote(String newDepartment, double salaryIncrease) {
    department = newDepartment;
    baseSalary += salaryIncrease;
    print('ยินดีด้วย $name! ได้รับการเลื่อนตำแหน่งไปยัง $newDepartment');
  }
  
  Map<String, dynamic> toMap() {
    return {
      'name': name,
      'department': department,
      'baseSalary': baseSalary,
      'skills': skills,
      'hireDate': hireDate.toIso8601String(),
      'yearsOfService': yearsOfService,
      'totalSalary': totalSalary,
    };
  }
  
  @override
  String toString() {
    return 'Employee($name, $department, ฿${baseSalary.toStringAsFixed(0)}/เดือน)';
  }
}

void main() {
  Employee emp1 = Employee(
    name: 'สมชาย ใจดี',
    department: 'IT',
    baseSalary: 35000,
    skills: ['Dart', 'Flutter'],
    hireDate: DateTime(2020, 5, 15),
  );
  
  emp1.addSkill('Firebase');
  emp1.addSkill('Dart'); // ไม่เพิ่ม เพราะมีแล้ว
  emp1.addOvertimeHours(20);
  
  print(emp1);
  print('ทำงานมา: ${emp1.yearsOfService} ปี');
  print('เงินเดือนรวม: ฿${emp1.totalSalary.toStringAsFixed(2)}');
  print('Skills: ${emp1.skills.join(', ')}');
  
  emp1.promote('Senior IT', 5000);
  print(emp1);
}
```

---

## 7.4 Static Fields และ Methods

```dart
class AppConfig {
  // Static fields - แชร์ระหว่าง instances ทั้งหมด
  static const String appName = 'My Flutter App';
  static const String version = '1.0.0';
  static String environment = 'development';
  static int _instanceCount = 0;
  
  // Instance fields
  final String userId;
  final DateTime createdAt;
  
  AppConfig(this.userId) : createdAt = DateTime.now() {
    _instanceCount++;
  }
  
  // Static methods
  static void setEnvironment(String env) {
    if (['development', 'staging', 'production'].contains(env)) {
      environment = env;
      print('เปลี่ยน environment เป็น: $env');
    } else {
      throw ArgumentError('environment ไม่ถูกต้อง: $env');
    }
  }
  
  static bool isProduction() => environment == 'production';
  
  static int getInstanceCount() => _instanceCount;
  
  static void printInfo() {
    print('=== App Info ===');
    print('App: $appName v$version');
    print('Environment: $environment');
    print('Instances: $_instanceCount');
  }
  
  // Instance method ที่ใช้ static field
  String getFullInfo() {
    return '[$environment] User: $userId at ${createdAt.toLocal()}';
  }
  
  @override
  String toString() => 'AppConfig(user: $userId, env: $environment)';
}

class MathUtils {
  // Utility class - ใช้แต่ static methods, ไม่ต้อง instantiate
  MathUtils._(); // private constructor ป้องกันการสร้าง instance
  
  static const double pi = 3.141592653589793;
  static const double e = 2.718281828459045;
  
  static double circleArea(double radius) => pi * radius * radius;
  
  static double factorial(int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);
  }
  
  static int gcd(int a, int b) {
    while (b != 0) {
      int temp = b;
      b = a % b;
      a = temp;
    }
    return a;
  }
  
  static int lcm(int a, int b) => (a * b) ~/ gcd(a, b);
  
  static bool isPrime(int n) {
    if (n < 2) return false;
    if (n == 2) return true;
    if (n % 2 == 0) return false;
    for (int i = 3; i * i <= n; i += 2) {
      if (n % i == 0) return false;
    }
    return true;
  }
  
  static List<int> primeFactors(int n) {
    List<int> factors = [];
    for (int i = 2; i * i <= n; i++) {
      while (n % i == 0) {
        factors.add(i);
        n ~/= i;
      }
    }
    if (n > 1) factors.add(n);
    return factors;
  }
}

void main() {
  // Static fields/methods - เรียกผ่าน class name
  AppConfig.setEnvironment('staging');
  AppConfig.printInfo();
  
  AppConfig config1 = AppConfig('user001');
  AppConfig config2 = AppConfig('user002');
  
  print('จำนวน instances: ${AppConfig.getInstanceCount()}'); // 2
  print(config1.getFullInfo());
  
  // MathUtils - ใช้ static methods โดยตรง
  print('\n=== Math Utils ===');
  print('พื้นที่วงกลม r=5: ${MathUtils.circleArea(5).toStringAsFixed(2)}');
  print('6! = ${MathUtils.factorial(6).toInt()}');
  print('GCD(12, 8) = ${MathUtils.gcd(12, 8)}');
  print('LCM(4, 6) = ${MathUtils.lcm(4, 6)}');
  print('17 เป็นจำนวนเฉพาะ? ${MathUtils.isPrime(17)}');
  print('ตัวประกอบเฉพาะของ 360: ${MathUtils.primeFactors(360)}');
}
```

---

## 7.5 Getters และ Setters

```dart
class Temperature {
  // Backing field (private)
  double _celsius;
  
  Temperature(this._celsius);
  
  // Getter - อ่านค่า
  double get celsius => _celsius;
  
  // Setter - กำหนดค่าพร้อม validation
  set celsius(double value) {
    if (value < -273.15) {
      throw ArgumentError('อุณหภูมิต่ำกว่า absolute zero ไม่ได้ (-273.15°C)');
    }
    _celsius = value;
  }
  
  // Computed getters
  double get fahrenheit => _celsius * 9 / 5 + 32;
  
  set fahrenheit(double value) {
    celsius = (value - 32) * 5 / 9;
  }
  
  double get kelvin => _celsius + 273.15;
  
  set kelvin(double value) {
    celsius = value - 273.15;
  }
  
  bool get isBoiling => _celsius >= 100;
  bool get isFreezing => _celsius <= 0;
  
  String get description {
    if (_celsius < 0) return 'หนาวมาก';
    if (_celsius < 15) return 'หนาว';
    if (_celsius < 25) return 'เย็นสบาย';
    if (_celsius < 35) return 'อุ่น';
    if (_celsius < 40) return 'ร้อน';
    return 'ร้อนมาก';
  }
  
  @override
  String toString() => '${_celsius.toStringAsFixed(1)}°C (${fahrenheit.toStringAsFixed(1)}°F)';
}

class BoundedValue {
  double _value;
  final double min;
  final double max;
  
  BoundedValue(double initial, {required this.min, required this.max})
      : _value = initial.clamp(min, max);
  
  double get value => _value;
  
  set value(double newValue) {
    _value = newValue.clamp(min, max);
  }
  
  double get percentage => ((_value - min) / (max - min) * 100);
  
  void increment([double by = 1]) => value = _value + by;
  void decrement([double by = 1]) => value = _value - by;
  
  @override
  String toString() => '$_value [$min-$max] (${percentage.toStringAsFixed(0)}%)';
}

void main() {
  Temperature temp = Temperature(25.0);
  
  print(temp);
  print('Kelvin: ${temp.kelvin}K');
  print('สถานะ: ${temp.description}');
  
  temp.fahrenheit = 212; // ตั้งค่าด้วย Fahrenheit
  print('หลังตั้ง 212°F: $temp');
  print('กำลังเดือด: ${temp.isBoiling}');
  
  try {
    temp.celsius = -300; // ควร throw error
  } catch (e) {
    print('Error: $e');
  }
  
  // BoundedValue
  print('\n=== BoundedValue ===');
  BoundedValue volume = BoundedValue(50, min: 0, max: 100);
  print('Volume: $volume');
  
  volume.increment(30);
  print('หลังเพิ่ม 30: $volume');
  
  volume.increment(50); // เกิน max
  print('หลังเพิ่มอีก 50 (เกิน max): $volume'); // จะถูก clamp ที่ 100
  
  BoundedValue hp = BoundedValue(100, min: 0, max: 100);
  hp.decrement(75);
  print('HP: $hp');
}
```

---

## 7.6 toString(), hashCode, และ == Operator

```dart
class Student {
  final String id;
  final String name;
  final int grade;
  final double gpa;
  
  const Student({
    required this.id,
    required this.name,
    required this.grade,
    required this.gpa,
  });
  
  // toString() - แสดงผลแบบ human-readable
  @override
  String toString() {
    return 'Student{id: $id, name: $name, grade: $grade, GPA: ${gpa.toStringAsFixed(2)}}';
  }
  
  // == operator - เปรียบเทียบ equality ตาม value
  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true; // เป็น object เดียวกัน
    if (other is! Student) return false;     // ต้องเป็น Student
    return id == other.id;                   // เปรียบเทียบด้วย id
  }
  
  // hashCode ต้อง consistent กับ == operator
  // ถ้า a == b แล้ว a.hashCode ต้อง == b.hashCode
  @override
  int get hashCode => id.hashCode;
  
  // copyWith pattern
  Student copyWith({
    String? id,
    String? name,
    int? grade,
    double? gpa,
  }) {
    return Student(
      id: id ?? this.id,
      name: name ?? this.name,
      grade: grade ?? this.grade,
      gpa: gpa ?? this.gpa,
    );
  }
  
  // toMap สำหรับ serialization
  Map<String, dynamic> toMap() => {
    'id': id,
    'name': name,
    'grade': grade,
    'gpa': gpa,
  };
  
  // fromMap สำหรับ deserialization
  factory Student.fromMap(Map<String, dynamic> map) {
    return Student(
      id: map['id'] as String,
      name: map['name'] as String,
      grade: map['grade'] as int,
      gpa: (map['gpa'] as num).toDouble(),
    );
  }
}

void main() {
  Student s1 = Student(id: 'S001', name: 'สมชาย', grade: 10, gpa: 3.5);
  Student s2 = Student(id: 'S001', name: 'สมชาย', grade: 10, gpa: 3.5);
  Student s3 = Student(id: 'S002', name: 'มานี', grade: 11, gpa: 3.8);
  
  // toString()
  print(s1); // Student{id: S001, name: สมชาย, grade: 10, GPA: 3.50}
  
  // == operator
  print(s1 == s2); // true (same id)
  print(s1 == s3); // false
  
  // hashCode
  print(s1.hashCode == s2.hashCode); // true
  print(s1.hashCode == s3.hashCode); // false
  
  // ใช้ใน Set (ต้องการ hashCode ที่ถูกต้อง)
  Set<Student> studentSet = {s1, s2, s3}; // s2 ซ้ำกับ s1
  print('Set size: ${studentSet.length}'); // 2 (ไม่นับ s2 ซ้ำ)
  
  // ใช้ใน Map key
  Map<Student, String> grades = {
    s1: 'A',
    s3: 'A+',
  };
  print('เกรดของ s2: ${grades[s2]}'); // A (หาได้เพราะ s2 == s1)
  
  // copyWith
  Student promoted = s1.copyWith(grade: 11, gpa: 3.7);
  print('อัปเดต: $promoted');
  
  // Serialization
  Map<String, dynamic> json = s1.toMap();
  print('JSON: $json');
  
  Student restored = Student.fromMap(json);
  print('Restored: $restored');
  print('เหมือนกัน: ${s1 == restored}'); // true
}
```

---

## 7.7 Encapsulation - Private Members

```dart
class SecureWallet {
  // Private fields - เข้าถึงได้เฉพาะใน class (หรือ library)
  String _pin;
  double _balance;
  List<Map<String, dynamic>> _transactions;
  bool _isLocked;
  int _failedAttempts;
  
  static const int maxFailedAttempts = 3;
  
  SecureWallet({required String pin, double initialBalance = 0})
    : _pin = pin,
      _balance = initialBalance,
      _transactions = [],
      _isLocked = false,
      _failedAttempts = 0;
  
  // Public getters - อ่านได้แต่แก้ไขตรงไม่ได้
  double get balance => _isLocked ? 0 : _balance;
  bool get isLocked => _isLocked;
  int get transactionCount => _transactions.length;
  
  // Public methods - interface สำหรับใช้งาน
  bool unlock(String pin) {
    if (_isLocked) {
      print('กระเป๋าถูกล็อค กรุณาติดต่อธนาคาร');
      return false;
    }
    
    if (pin == _pin) {
      _failedAttempts = 0;
      return true;
    } else {
      _failedAttempts++;
      if (_failedAttempts >= maxFailedAttempts) {
        _isLocked = true;
        print('ล็อคบัญชี: กรอก PIN ผิดเกิน $maxFailedAttempts ครั้ง');
      } else {
        print('PIN ไม่ถูกต้อง (${maxFailedAttempts - _failedAttempts} ครั้งที่เหลือ)');
      }
      return false;
    }
  }
  
  bool deposit(String pin, double amount) {
    if (!unlock(pin)) return false;
    if (amount <= 0) {
      print('จำนวนเงินต้องมากกว่า 0');
      return false;
    }
    
    _balance += amount;
    _addTransaction('ฝากเงิน', amount, _balance);
    print('ฝากเงินสำเร็จ: ฿${amount.toStringAsFixed(2)} ยอดคงเหลือ: ฿${_balance.toStringAsFixed(2)}');
    return true;
  }
  
  bool withdraw(String pin, double amount) {
    if (!unlock(pin)) return false;
    if (amount <= 0) {
      print('จำนวนเงินต้องมากกว่า 0');
      return false;
    }
    if (amount > _balance) {
      print('ยอดเงินไม่เพียงพอ');
      return false;
    }
    
    _balance -= amount;
    _addTransaction('ถอนเงิน', -amount, _balance);
    print('ถอนเงินสำเร็จ: ฿${amount.toStringAsFixed(2)} ยอดคงเหลือ: ฿${_balance.toStringAsFixed(2)}');
    return true;
  }
  
  bool changePin(String oldPin, String newPin) {
    if (!unlock(oldPin)) return false;
    if (newPin.length != 6 || !RegExp(r'^\d+$').hasMatch(newPin)) {
      print('PIN ต้องเป็นตัวเลข 6 หลัก');
      return false;
    }
    _pin = newPin;
    print('เปลี่ยน PIN สำเร็จ');
    return true;
  }
  
  List<Map<String, dynamic>> getTransactionHistory(String pin) {
    if (!unlock(pin)) return [];
    return List.unmodifiable(_transactions);
  }
  
  // Private helper method
  void _addTransaction(String type, double amount, double balanceAfter) {
    _transactions.add({
      'type': type,
      'amount': amount,
      'balanceAfter': balanceAfter,
      'timestamp': DateTime.now().toIso8601String(),
    });
  }
  
  @override
  String toString() {
    if (_isLocked) return 'SecureWallet(LOCKED)';
    return 'SecureWallet(balance: ฿${_balance.toStringAsFixed(2)})';
  }
}

void main() {
  print('=== ทดสอบ SecureWallet ===\n');
  
  SecureWallet wallet = SecureWallet(pin: '123456', initialBalance: 1000);
  
  // ฝากเงิน
  wallet.deposit('123456', 5000);
  
  // ถอนเงิน
  wallet.withdraw('123456', 2000);
  
  // ลองใส่ PIN ผิด
  print('\n--- ทดสอบ PIN ผิด ---');
  wallet.withdraw('000000', 100);
  wallet.withdraw('111111', 100);
  wallet.withdraw('222222', 100); // ครั้งที่ 3 = ล็อค
  
  // หลังล็อค
  print('\n--- หลังล็อค ---');
  print(wallet);
  wallet.deposit('123456', 100); // ไม่ได้
  
  // ดูประวัติธุรกรรม
  print('\n--- ประวัติก่อนล็อค ---');
  // สร้าง wallet ใหม่
  SecureWallet wallet2 = SecureWallet(pin: '654321', initialBalance: 0);
  wallet2.deposit('654321', 10000);
  wallet2.withdraw('654321', 3000);
  wallet2.deposit('654321', 5000);
  
  List<Map<String, dynamic>> history = wallet2.getTransactionHistory('654321');
  print('ประวัติธุรกรรม:');
  for (var t in history) {
    print('  ${t['type']}: ฿${t['amount']} (ยอดหลัง: ฿${t['balanceAfter']})');
  }
}
```

---

## 7.8 Workshop: BankAccount Class

```dart
// bank_account.dart
// ระบบบัญชีธนาคารที่ใช้หลักการ OOP

enum AccountType {
  savings,    // ออมทรัพย์
  checking,   // กระแสรายวัน
  fixed,      // ฝากประจำ
}

enum TransactionType {
  deposit,    // ฝาก
  withdrawal, // ถอน
  transfer,   // โอน
  interest,   // ดอกเบี้ย
  fee,        // ค่าธรรมเนียม
}

class Transaction {
  final String id;
  final TransactionType type;
  final double amount;
  final double balanceBefore;
  final double balanceAfter;
  final String description;
  final DateTime timestamp;
  final String? referenceId; // สำหรับโอน
  
  Transaction({
    required this.id,
    required this.type,
    required this.amount,
    required this.balanceBefore,
    required this.balanceAfter,
    required this.description,
    DateTime? timestamp,
    this.referenceId,
  }) : timestamp = timestamp ?? DateTime.now();
  
  @override
  String toString() {
    String sign = amount >= 0 ? '+' : '';
    return '[${timestamp.toLocal().toString().substring(0, 16)}] '
           '${type.name}: $sign฿${amount.toStringAsFixed(2)} '
           '(คงเหลือ: ฿${balanceAfter.toStringAsFixed(2)}) '
           '- $description';
  }
}

class BankAccount {
  // Private fields
  final String _accountNumber;
  final String _ownerName;
  final AccountType _type;
  double _balance;
  final List<Transaction> _transactions;
  bool _isActive;
  int _transactionCounter;
  
  // Static config
  static const Map<AccountType, double> interestRates = {
    AccountType.savings: 0.005,    // 0.5% ต่อปี
    AccountType.checking: 0.001,   // 0.1% ต่อปี
    AccountType.fixed: 0.02,       // 2% ต่อปี
  };
  
  static const double withdrawalFee = 20.0; // ค่าธรรมเนียมถอนนอกเวลา
  static const double transferFee = 25.0;   // ค่าธรรมเนียมโอน
  
  // Constructor
  BankAccount({
    required String accountNumber,
    required String ownerName,
    required AccountType type,
    double initialDeposit = 0,
  }) : _accountNumber = accountNumber,
       _ownerName = ownerName,
       _type = type,
       _balance = 0,
       _transactions = [],
       _isActive = true,
       _transactionCounter = 0 {
    if (initialDeposit > 0) {
      _deposit(initialDeposit, 'เปิดบัญชี');
    }
  }
  
  // Factory constructor
  factory BankAccount.savings({
    required String accountNumber,
    required String ownerName,
    double initialDeposit = 0,
  }) {
    return BankAccount(
      accountNumber: accountNumber,
      ownerName: ownerName,
      type: AccountType.savings,
      initialDeposit: initialDeposit,
    );
  }
  
  factory BankAccount.checking({
    required String accountNumber,
    required String ownerName,
    double initialDeposit = 0,
  }) {
    return BankAccount(
      accountNumber: accountNumber,
      ownerName: ownerName,
      type: AccountType.checking,
      initialDeposit: initialDeposit,
    );
  }
  
  // Getters
  String get accountNumber => _accountNumber;
  String get ownerName => _ownerName;
  AccountType get type => _type;
  double get balance => _balance;
  bool get isActive => _isActive;
  int get transactionCount => _transactions.length;
  
  String get maskedAccountNumber {
    if (_accountNumber.length <= 4) return _accountNumber;
    return 'XXXX-XXXX-${_accountNumber.substring(_accountNumber.length - 4)}';
  }
  
  double get annualInterestRate => interestRates[_type] ?? 0;
  
  // Private helper
  String _generateTxId() {
    _transactionCounter++;
    return 'TX${DateTime.now().millisecondsSinceEpoch}${_transactionCounter.toString().padLeft(4, '0')}';
  }
  
  void _addTransaction({
    required TransactionType type,
    required double amount,
    required String description,
    String? referenceId,
  }) {
    _transactions.add(Transaction(
      id: _generateTxId(),
      type: type,
      amount: amount,
      balanceBefore: _balance,
      balanceAfter: _balance,
      description: description,
      referenceId: referenceId,
    ));
  }
  
  bool _deposit(double amount, String description) {
    if (amount <= 0) return false;
    double before = _balance;
    _balance += amount;
    _transactions.add(Transaction(
      id: _generateTxId(),
      type: TransactionType.deposit,
      amount: amount,
      balanceBefore: before,
      balanceAfter: _balance,
      description: description,
    ));
    return true;
  }
  
  bool _withdraw(double amount, String description) {
    if (amount <= 0 || amount > _balance) return false;
    double before = _balance;
    _balance -= amount;
    _transactions.add(Transaction(
      id: _generateTxId(),
      type: TransactionType.withdrawal,
      amount: -amount,
      balanceBefore: before,
      balanceAfter: _balance,
      description: description,
    ));
    return true;
  }
  
  // Public operations
  bool deposit(double amount) {
    if (!_isActive) {
      print('ไม่สามารถทำธุรกรรม: บัญชีถูกระงับ');
      return false;
    }
    if (amount <= 0) {
      print('จำนวนเงินต้องมากกว่า 0');
      return false;
    }
    
    bool success = _deposit(amount, 'ฝากเงิน');
    if (success) {
      print('ฝากเงินสำเร็จ: ฿${amount.toStringAsFixed(2)} ยอดคงเหลือ: ฿${_balance.toStringAsFixed(2)}');
    }
    return success;
  }
  
  bool withdraw(double amount, {bool isAfterHours = false}) {
    if (!_isActive) {
      print('ไม่สามารถทำธุรกรรม: บัญชีถูกระงับ');
      return false;
    }
    
    double fee = isAfterHours ? withdrawalFee : 0;
    double total = amount + fee;
    
    if (total > _balance) {
      print('ยอดเงินไม่เพียงพอ (ต้องการ ฿${total.toStringAsFixed(2)}, มี ฿${_balance.toStringAsFixed(2)})');
      return false;
    }
    
    bool success = _withdraw(total, 'ถอนเงิน${isAfterHours ? " (นอกเวลา)" : ""}');
    if (success) {
      if (fee > 0) print('หักค่าธรรมเนียม: ฿$fee');
      print('ถอนเงินสำเร็จ: ฿${amount.toStringAsFixed(2)} ยอดคงเหลือ: ฿${_balance.toStringAsFixed(2)}');
    }
    return success;
  }
  
  bool transferTo(BankAccount target, double amount) {
    if (!_isActive || !target._isActive) {
      print('ไม่สามารถโอนเงิน: บัญชีถูกระงับ');
      return false;
    }
    
    double total = amount + transferFee;
    if (total > _balance) {
      print('ยอดเงินไม่เพียงพอสำหรับโอน (ต้องการ ฿${total.toStringAsFixed(2)})');
      return false;
    }
    
    String txId = _generateTxId();
    
    // หักจากต้นทาง
    double beforeSrc = _balance;
    _balance -= total;
    _transactions.add(Transaction(
      id: txId,
      type: TransactionType.transfer,
      amount: -total,
      balanceBefore: beforeSrc,
      balanceAfter: _balance,
      description: 'โอนเงินไป ${target.maskedAccountNumber}',
      referenceId: target._accountNumber,
    ));
    
    // เพิ่มที่ปลายทาง
    double beforeTgt = target._balance;
    target._balance += amount;
    target._transactions.add(Transaction(
      id: txId,
      type: TransactionType.transfer,
      amount: amount,
      balanceBefore: beforeTgt,
      balanceAfter: target._balance,
      description: 'รับโอนจาก ${maskedAccountNumber}',
      referenceId: _accountNumber,
    ));
    
    print('โอนเงินสำเร็จ: ฿${amount.toStringAsFixed(2)} ไปยัง ${target.ownerName}');
    print('หักค่าธรรมเนียม: ฿$transferFee');
    return true;
  }
  
  void applyInterest() {
    double interest = _balance * (annualInterestRate / 12); // รายเดือน
    if (interest > 0) {
      double before = _balance;
      _balance += interest;
      _transactions.add(Transaction(
        id: _generateTxId(),
        type: TransactionType.interest,
        amount: interest,
        balanceBefore: before,
        balanceAfter: _balance,
        description: 'ดอกเบี้ยรายเดือน (${(annualInterestRate * 100).toStringAsFixed(1)}%/ปี)',
      ));
      print('ดอกเบี้ยเดือนนี้: ฿${interest.toStringAsFixed(2)}');
    }
  }
  
  List<Transaction> getHistory({int? last}) {
    if (last != null) {
      return _transactions.reversed.take(last).toList().reversed.toList();
    }
    return List.unmodifiable(_transactions);
  }
  
  void printStatement() {
    print('\n=== สรุปบัญชี ===');
    print('เลขบัญชี: $maskedAccountNumber');
    print('ชื่อเจ้าของ: $_ownerName');
    print('ประเภท: ${_type.name}');
    print('ยอดคงเหลือ: ฿${_balance.toStringAsFixed(2)}');
    print('อัตราดอกเบี้ย: ${(annualInterestRate * 100).toStringAsFixed(1)}% ต่อปี');
    print('สถานะ: ${_isActive ? "ใช้งานได้" : "ระงับ"}');
    
    if (_transactions.isNotEmpty) {
      print('\nประวัติล่าสุด (5 รายการ):');
      getHistory(last: 5).forEach(print);
    }
  }
  
  void closeAccount() {
    _isActive = false;
    print('ปิดบัญชี: $maskedAccountNumber');
  }
  
  @override
  String toString() => 'BankAccount($_ownerName, $maskedAccountNumber, ฿${_balance.toStringAsFixed(2)})';
  
  @override
  bool operator ==(Object other) =>
      other is BankAccount && _accountNumber == other._accountNumber;
  
  @override
  int get hashCode => _accountNumber.hashCode;
}

void main() {
  print('=== ระบบธนาคาร ===\n');
  
  // สร้างบัญชี
  BankAccount savings = BankAccount.savings(
    accountNumber: '1234567890',
    ownerName: 'สมชาย ใจดี',
    initialDeposit: 5000,
  );
  
  BankAccount checking = BankAccount.checking(
    accountNumber: '0987654321',
    ownerName: 'มานี รักเรียน',
    initialDeposit: 10000,
  );
  
  // ธุรกรรม
  savings.deposit(3000);
  savings.withdraw(1500);
  savings.withdraw(500, isAfterHours: true);
  
  // โอนเงิน
  checking.transferTo(savings, 2000);
  
  // ดอกเบี้ย
  savings.applyInterest();
  
  // แสดงสรุป
  savings.printStatement();
  checking.printStatement();
  
  // ทดสอบ equality
  print('\n=== ทดสอบ Equality ===');
  BankAccount same = BankAccount.savings(
    accountNumber: '1234567890',
    ownerName: 'ชื่ออื่น',
    initialDeposit: 0,
  );
  
  print('savings == same: ${savings == same}'); // true (same account number)
  print('savings == checking: ${savings == checking}'); // false
}
```

---

## สรุป Part 07

ใน Part นี้เราได้เรียนรู้:

### Constructors:
| ประเภท | ใช้เมื่อ |
|--------|---------|
| **Default** | กรณีทั่วไป |
| **Named** | หลาย initialization paths |
| **Factory** | Singleton, Subclass selection, Complex validation |
| **Const** | Compile-time constants, Immutable objects |

### Key Concepts:
- **Encapsulation**: ใช้ `_` prefix สำหรับ private members
- **Getters/Setters**: ควบคุมการอ่าน/เขียน field
- **Static**: ข้อมูลและ logic ที่แชร์ระหว่าง instances
- **toString/hashCode/==**: Custom equality และ string representation

### Best Practices:
1. ใช้ `final` สำหรับ fields ที่ไม่ควรเปลี่ยน
2. ใช้ private fields กับ public getters เสมอ
3. Override `==` และ `hashCode` ไปพร้อมกันเสมอ
4. ใช้ `copyWith` pattern สำหรับ immutable objects

## ➡️ Part ถัดไป

**Part 08: Inheritance and Polymorphism** - เราจะเรียนรู้การสืบทอด class, การ override methods, abstract classes และ polymorphism
