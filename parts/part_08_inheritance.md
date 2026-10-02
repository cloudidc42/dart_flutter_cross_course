# Part 08: Inheritance and Polymorphism

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ `extends` เพื่อสืบทอด class ได้
- ใช้ `super` เรียก constructor และ methods ของ parent class
- Override methods ด้วย `@override`
- สร้างและใช้งาน abstract classes
- เข้าใจ polymorphism และประโยชน์ของมัน
- ใช้ `is` และ `as` สำหรับ type checking
- สร้าง Shape hierarchy เป็น workshop

---

## 8.1 พื้นฐาน Inheritance

Inheritance ช่วยให้ class ลูก (subclass) รับ fields และ methods จาก class แม่ (superclass)

```dart
// Superclass (Parent class)
class Vehicle {
  String brand;
  String model;
  int year;
  double fuelLevel; // 0.0 - 1.0
  
  Vehicle({
    required this.brand,
    required this.model,
    required this.year,
    this.fuelLevel = 1.0,
  });
  
  // Methods ที่ subclass จะได้รับ
  void start() {
    print('$brand $model กำลังสตาร์ท...');
  }
  
  void stop() {
    print('$brand $model หยุดแล้ว');
  }
  
  void refuel(double amount) {
    fuelLevel = (fuelLevel + amount).clamp(0.0, 1.0);
    print('เติมน้ำมัน: ระดับน้ำมัน ${(fuelLevel * 100).toStringAsFixed(0)}%');
  }
  
  String getInfo() {
    return '$year $brand $model';
  }
  
  @override
  String toString() => 'Vehicle(${getInfo()})';
}

// Subclass (Child class) - รับ fields และ methods จาก Vehicle
class Car extends Vehicle {
  int doors;
  String transmission; // 'manual' หรือ 'automatic'
  
  Car({
    required super.brand,
    required super.model,
    required super.year,
    this.doors = 4,
    this.transmission = 'automatic',
    super.fuelLevel,
  });
  
  // Method ใหม่ที่มีเฉพาะ Car
  void openTrunk() {
    print('เปิดฝากระโปรงท้าย');
  }
  
  // Override method ของ parent
  @override
  String getInfo() {
    return '${super.getInfo()} ($doors ประตู, $transmission)';
  }
  
  @override
  String toString() => 'Car(${getInfo()})';
}

class Motorcycle extends Vehicle {
  bool hasSidecar;
  String engineType; // '2-stroke', '4-stroke'
  
  Motorcycle({
    required super.brand,
    required super.model,
    required super.year,
    this.hasSidecar = false,
    this.engineType = '4-stroke',
    super.fuelLevel,
  });
  
  void doWheely() {
    print('$brand $model ยกล้อหน้า! (อย่าทำที่บ้าน)');
  }
  
  @override
  String getInfo() {
    return '${super.getInfo()} ($engineType${hasSidecar ? ", มี sidecar" : ""})';
  }
  
  @override
  String toString() => 'Motorcycle(${getInfo()})';
}

void main() {
  Car car = Car(
    brand: 'Toyota',
    model: 'Camry',
    year: 2023,
    doors: 4,
  );
  
  Motorcycle bike = Motorcycle(
    brand: 'Honda',
    model: 'CBR500R',
    year: 2022,
    engineType: '4-stroke',
  );
  
  // ใช้ inherited methods
  car.start();
  car.refuel(0.3);
  car.openTrunk(); // เฉพาะ Car
  
  bike.start();
  bike.doWheely(); // เฉพาะ Motorcycle
  
  print(car.getInfo());
  print(bike.getInfo());
  
  // Polymorphism - ใช้ parent type
  Vehicle v1 = car;
  Vehicle v2 = bike;
  
  v1.start(); // เรียก Car.start() (inherited from Vehicle)
  v2.start(); // เรียก Motorcycle.start() (inherited from Vehicle)
  
  print(v1.getInfo()); // เรียก Car.getInfo() - polymorphism!
  print(v2.getInfo()); // เรียก Motorcycle.getInfo() - polymorphism!
}
```

---

## 8.2 super Keyword

```dart
class Animal {
  String name;
  int age;
  String sound;
  
  Animal({
    required this.name,
    required this.age,
    required this.sound,
  }) {
    print('สร้าง Animal: $name');
  }
  
  void makeSound() {
    print('$name ร้องว่า: $sound');
  }
  
  void eat(String food) {
    print('$name กำลังกิน $food');
  }
  
  String describe() {
    return '$name (อายุ $age ปี)';
  }
}

class Dog extends Animal {
  String breed; // สายพันธุ์
  bool isVaccinated;
  
  // ใช้ super() เรียก parent constructor
  Dog({
    required super.name,
    required super.age,
    required this.breed,
    this.isVaccinated = false,
  }) : super(sound: 'โฮ่ง!') {
    print('สร้าง Dog: $name (${breed})');
  }
  
  void fetch(String item) {
    print('$name วิ่งไปเอา $item กลับมา');
  }
  
  // Override และเรียก super
  @override
  void makeSound() {
    super.makeSound(); // เรียก parent method ก่อน
    print('  ... และโผไปหาเจ้าของ');
  }
  
  @override
  String describe() {
    return '${super.describe()} - $breed (${isVaccinated ? "ฉีดวัคซีนแล้ว" : "ยังไม่ฉีด"})';
  }
}

class GoldenRetriever extends Dog {
  String furColor;
  
  GoldenRetriever({
    required super.name,
    required super.age,
    this.furColor = 'ทอง',
    super.isVaccinated,
  }) : super(breed: 'Golden Retriever');
  
  void swim() {
    print('$name ว่ายน้ำเก่งมาก!');
  }
  
  @override
  String describe() {
    return '${super.describe()}, ขน: $furColor';
  }
}

void main() {
  print('=== สร้าง Animals ===\n');
  
  Dog dog = Dog(
    name: 'บัดดี้',
    age: 3,
    breed: 'Labrador',
    isVaccinated: true,
  );
  
  print('\n=== Methods ===');
  dog.makeSound(); // เรียก super.makeSound() ด้วย
  dog.eat('กระดูก');
  dog.fetch('ลูกบอล');
  print(dog.describe());
  
  print('\n=== GoldenRetriever ===');
  GoldenRetriever golden = GoldenRetriever(
    name: 'บัตเตอร์คัพ',
    age: 2,
    furColor: 'ทองอ่อน',
    isVaccinated: true,
  );
  
  golden.makeSound();
  golden.swim();
  print(golden.describe());
  
  // Multi-level inheritance chain
  print('\nประเภท:');
  print(golden is GoldenRetriever); // true
  print(golden is Dog);            // true
  print(golden is Animal);         // true
}
```

---

## 8.3 Method Overriding

```dart
class Shape {
  String color;
  
  Shape({this.color = 'ขาว'});
  
  // Methods ที่ subclass ควร override
  double get area => 0;
  double get perimeter => 0;
  
  void draw() {
    print('วาด Shape สี$color');
  }
  
  void printInfo() {
    print('รูปทรง: ${runtimeType}');
    print('สี: $color');
    print('พื้นที่: ${area.toStringAsFixed(2)} ตร.หน่วย');
    print('เส้นรอบรูป: ${perimeter.toStringAsFixed(2)} หน่วย');
  }
}

class Circle extends Shape {
  final double radius;
  
  Circle(this.radius, {super.color = 'แดง'});
  
  @override
  double get area => 3.14159 * radius * radius;
  
  @override
  double get perimeter => 2 * 3.14159 * radius;
  
  @override
  void draw() {
    print('วาดวงกลมรัศมี $radius หน่วย สี$color');
  }
}

class Rectangle extends Shape {
  final double width;
  final double height;
  
  Rectangle(this.width, this.height, {super.color = 'น้ำเงิน'});
  
  @override
  double get area => width * height;
  
  @override
  double get perimeter => 2 * (width + height);
  
  @override
  void draw() {
    print('วาดสี่เหลี่ยม ${width}x${height} หน่วย สี$color');
  }
}

class Square extends Rectangle {
  // Square เป็น special case ของ Rectangle
  Square(double side, {super.color = 'เขียว'}) : super(side, side);
  
  @override
  void draw() {
    print('วาดจตุรัสด้าน ${width} หน่วย สี$color');
  }
}

class Triangle extends Shape {
  final double base;
  final double height;
  final double sideA;
  final double sideB;
  final double sideC;
  
  Triangle({
    required this.base,
    required this.height,
    required this.sideA,
    required this.sideB,
    required this.sideC,
    super.color = 'เหลือง',
  });
  
  // Factory constructor สำหรับสามเหลี่ยมด้านเท่า
  factory Triangle.equilateral(double side, {String color = 'ม่วง'}) {
    double h = side * (3.0) / 2;
    return Triangle(
      base: side,
      height: h,
      sideA: side,
      sideB: side,
      sideC: side,
      color: color,
    );
  }
  
  // Factory constructor สำหรับสามเหลี่ยมมุมฉาก
  factory Triangle.rightAngle(double leg1, double leg2, {String color = 'ส้ม'}) {
    double hypotenuse = (leg1 * leg1 + leg2 * leg2);
    return Triangle(
      base: leg1,
      height: leg2,
      sideA: leg1,
      sideB: leg2,
      sideC: hypotenuse,
      color: color,
    );
  }
  
  @override
  double get area => 0.5 * base * height;
  
  @override
  double get perimeter => sideA + sideB + sideC;
  
  @override
  void draw() {
    print('วาดสามเหลี่ยม ฐาน=$base สูง=$height สี$color');
  }
}

void main() {
  List<Shape> shapes = [
    Circle(5, color: 'แดง'),
    Rectangle(4, 6, color: 'น้ำเงิน'),
    Square(3, color: 'เขียว'),
    Triangle.equilateral(6, color: 'ม่วง'),
    Triangle.rightAngle(3, 4, color: 'ส้ม'),
  ];
  
  print('=== รูปทรงทั้งหมด ===\n');
  
  for (Shape shape in shapes) {
    shape.draw(); // Polymorphism - เรียก method ของ subclass ที่ถูกต้อง
    print('  พื้นที่: ${shape.area.toStringAsFixed(2)}');
    print('  เส้นรอบรูป: ${shape.perimeter.toStringAsFixed(2)}');
    print();
  }
  
  // คำนวณรวม
  double totalArea = shapes.fold(0, (sum, s) => sum + s.area);
  print('พื้นที่รวมทั้งหมด: ${totalArea.toStringAsFixed(2)} ตร.หน่วย');
  
  // หาพื้นที่มากที่สุด
  Shape largest = shapes.reduce((a, b) => a.area > b.area ? a : b);
  print('พื้นที่ใหญ่สุด: ${largest.runtimeType} (${largest.area.toStringAsFixed(2)})');
}
```

---

## 8.4 Abstract Classes

Abstract class ไม่สามารถสร้าง instance ได้ แต่ใช้เป็น "สัญญา" ที่ subclass ต้องปฏิบัติตาม

```dart
// Abstract class - เป็น template
abstract class PaymentMethod {
  final String name;
  final String currency;
  
  PaymentMethod({required this.name, this.currency = 'THB'});
  
  // Abstract methods - subclass ต้อง implement
  Future<bool> processPayment(double amount);
  Future<bool> refund(String transactionId, double amount);
  bool validatePaymentInfo();
  
  // Concrete method ที่ทุก subclass ได้ใช้
  void printReceipt(double amount, String transactionId) {
    print('=== ใบเสร็จ ===');
    print('วิธีชำระ: $name');
    print('จำนวน: ฿${amount.toStringAsFixed(2)} $currency');
    print('Transaction ID: $transactionId');
    print('วันที่: ${DateTime.now().toLocal().toString().substring(0, 16)}');
    print('===============');
  }
  
  // Template method pattern
  Future<String?> pay(double amount) async {
    print('กำลังดำเนินการชำระเงินด้วย $name...');
    
    if (!validatePaymentInfo()) {
      print('ข้อมูลการชำระเงินไม่ถูกต้อง');
      return null;
    }
    
    bool success = await processPayment(amount);
    
    if (success) {
      String txId = 'TX${DateTime.now().millisecondsSinceEpoch}';
      printReceipt(amount, txId);
      return txId;
    } else {
      print('การชำระเงินล้มเหลว');
      return null;
    }
  }
}

class CreditCard extends PaymentMethod {
  final String cardNumber;
  final String cardHolderName;
  final String expiryDate;
  final String cvv;
  
  CreditCard({
    required this.cardNumber,
    required this.cardHolderName,
    required this.expiryDate,
    required this.cvv,
  }) : super(name: 'บัตรเครดิต');
  
  String get maskedCardNumber {
    return 'XXXX-XXXX-XXXX-${cardNumber.substring(cardNumber.length - 4)}';
  }
  
  @override
  bool validatePaymentInfo() {
    if (cardNumber.length != 16) return false;
    if (cvv.length != 3) return false;
    return true;
  }
  
  @override
  Future<bool> processPayment(double amount) async {
    // จำลองการเรียก API
    await Future.delayed(Duration(milliseconds: 100));
    print('ชำระด้วยบัตร $maskedCardNumber จำนวน ฿$amount');
    return amount <= 50000; // จำลัง limit
  }
  
  @override
  Future<bool> refund(String transactionId, double amount) async {
    await Future.delayed(Duration(milliseconds: 100));
    print('คืนเงิน ฿$amount ไปยังบัตร $maskedCardNumber');
    return true;
  }
}

class BankTransfer extends PaymentMethod {
  final String bankName;
  final String accountNumber;
  final String accountName;
  
  BankTransfer({
    required this.bankName,
    required this.accountNumber,
    required this.accountName,
  }) : super(name: 'โอนธนาคาร');
  
  @override
  bool validatePaymentInfo() {
    return accountNumber.length >= 10 && accountName.isNotEmpty;
  }
  
  @override
  Future<bool> processPayment(double amount) async {
    await Future.delayed(Duration(milliseconds: 200));
    print('โอนเงิน ฿$amount จาก $bankName เลขที่ $accountNumber');
    return true;
  }
  
  @override
  Future<bool> refund(String transactionId, double amount) async {
    print('แจ้งขอคืนเงิน Transaction: $transactionId จำนวน ฿$amount');
    return true;
  }
}

class PromptPay extends PaymentMethod {
  final String promptPayId; // เบอร์โทร หรือ เลขบัตรประชาชน
  
  PromptPay({required this.promptPayId})
      : super(name: 'พร้อมเพย์');
  
  @override
  bool validatePaymentInfo() {
    // เบอร์โทรต้องเป็น 10 หลัก หรือ เลขบัตรประชาชน 13 หลัก
    return promptPayId.length == 10 || promptPayId.length == 13;
  }
  
  @override
  Future<bool> processPayment(double amount) async {
    await Future.delayed(Duration(milliseconds: 50));
    print('โอนผ่าน PromptPay หมายเลข $promptPayId จำนวน ฿$amount');
    return amount <= 300000;
  }
  
  @override
  Future<bool> refund(String transactionId, double amount) async {
    print('ยังไม่รองรับการคืนเงินผ่าน PromptPay อัตโนมัติ');
    print('กรุณาติดต่อธนาคาร: TX $transactionId');
    return false;
  }
}

void main() async {
  print('=== ทดสอบระบบชำระเงิน ===\n');
  
  List<PaymentMethod> paymentMethods = [
    CreditCard(
      cardNumber: '4111111111111111',
      cardHolderName: 'สมชาย ใจดี',
      expiryDate: '12/25',
      cvv: '123',
    ),
    BankTransfer(
      bankName: 'กสิกรไทย',
      accountNumber: '1234567890',
      accountName: 'บริษัท ตัวอย่าง จำกัด',
    ),
    PromptPay(promptPayId: '0812345678'),
  ];
  
  for (PaymentMethod method in paymentMethods) {
    print('--- ${method.name} ---');
    await method.pay(1500.0);
    print();
  }
  
  // Polymorphism - ใช้ abstract type
  PaymentMethod selectedMethod = paymentMethods[2]; // PromptPay
  String? txId = await selectedMethod.pay(500.0);
  
  if (txId != null) {
    await selectedMethod.refund(txId, 500.0);
  }
}
```

---

## 8.5 Polymorphism

Polymorphism แปลว่า "หลายรูปแบบ" - object ประเภทต่างๆ ตอบสนองต่อ method เดียวกันได้

```dart
abstract class Notification {
  final String title;
  final String message;
  final DateTime sentAt;
  
  Notification({
    required this.title,
    required this.message,
    DateTime? sentAt,
  }) : sentAt = sentAt ?? DateTime.now();
  
  // Abstract - แต่ละ subclass implement เอง
  Future<void> send(String recipient);
  
  // Concrete - ทุก subclass ใช้ร่วมกัน
  String format() {
    return '[$title] $message';
  }
}

class EmailNotification extends Notification {
  final String fromEmail;
  final List<String> cc;
  final bool hasAttachment;
  
  EmailNotification({
    required super.title,
    required super.message,
    required this.fromEmail,
    this.cc = const [],
    this.hasAttachment = false,
  });
  
  @override
  Future<void> send(String recipient) async {
    print('📧 ส่ง Email ถึง: $recipient');
    print('   จาก: $fromEmail');
    print('   เรื่อง: $title');
    print('   เนื้อหา: $message');
    if (cc.isNotEmpty) print('   CC: ${cc.join(", ")}');
    if (hasAttachment) print('   มีไฟล์แนบ');
  }
}

class SMSNotification extends Notification {
  final String fromNumber;
  
  SMSNotification({
    required super.title,
    required super.message,
    required this.fromNumber,
  });
  
  @override
  Future<void> send(String recipient) async {
    // SMS จำกัด 160 ตัวอักษร
    String smsContent = format().length > 160 
        ? '${format().substring(0, 157)}...'
        : format();
    print('📱 ส่ง SMS ถึง: $recipient');
    print('   จาก: $fromNumber');
    print('   ข้อความ: $smsContent');
  }
}

class PushNotification extends Notification {
  final String appId;
  final Map<String, dynamic> data;
  
  PushNotification({
    required super.title,
    required super.message,
    required this.appId,
    this.data = const {},
  });
  
  @override
  Future<void> send(String recipient) async {
    print('🔔 Push Notification ถึง Device: $recipient');
    print('   App: $appId');
    print('   Title: $title');
    print('   Body: $message');
    if (data.isNotEmpty) print('   Data: $data');
  }
}

class LineNotification extends Notification {
  final String lineToken;
  
  LineNotification({
    required super.title,
    required super.message,
    required this.lineToken,
  });
  
  @override
  Future<void> send(String recipient) async {
    print('💬 LINE Notify ถึง: $recipient');
    print('   Token: ${lineToken.substring(0, 8)}...');
    print('   ข้อความ: ${format()}');
  }
}

// Notification Service ที่ใช้ Polymorphism
class NotificationService {
  final List<Notification> _queue = [];
  
  void addToQueue(Notification notification) {
    _queue.add(notification);
  }
  
  // ส่งทุก notification ไปยัง recipient - Polymorphism ในทางปฏิบัติ
  Future<void> sendAll(String recipient) async {
    print('=== กำลังส่ง ${_queue.length} การแจ้งเตือน ===');
    for (Notification notification in _queue) {
      await notification.send(recipient); // เรียก method ของ subclass ที่เหมาะสม
      print();
    }
    _queue.clear();
  }
  
  // ส่งเฉพาะประเภท - ใช้ is keyword
  Future<void> sendType<T extends Notification>(String recipient) async {
    List<Notification> typed = _queue.whereType<T>().toList();
    for (Notification n in typed) {
      await n.send(recipient);
    }
  }
}

void main() async {
  NotificationService service = NotificationService();
  
  // เพิ่ม notifications หลายประเภท
  service.addToQueue(EmailNotification(
    title: 'ยืนยันการสั่งซื้อ',
    message: 'คำสั่งซื้อ #12345 ได้รับการยืนยันแล้ว',
    fromEmail: 'no-reply@shop.com',
    cc: ['support@shop.com'],
    hasAttachment: true,
  ));
  
  service.addToQueue(SMSNotification(
    title: 'OTP',
    message: 'รหัส OTP ของคุณคือ 123456 (หมดอายุใน 5 นาที)',
    fromNumber: 'MYSHOP',
  ));
  
  service.addToQueue(PushNotification(
    title: 'สินค้าลดราคา!',
    message: 'Flash Sale เริ่มแล้ว ลดสูงสุด 70%',
    appId: 'com.example.myapp',
    data: {'screen': 'sale', 'discount': 70},
  ));
  
  service.addToQueue(LineNotification(
    title: 'แจ้งเตือน',
    message: 'พัสดุของคุณออกจากคลังสินค้าแล้ว',
    lineToken: 'AbCdEfGhIjKlMnOpQrStUv',
  ));
  
  // Polymorphism ในทางปฏิบัติ
  await service.sendAll('user@example.com');
}
```

---

## 8.6 is และ as - Type Checking

```dart
abstract class MediaContent {
  String title;
  String creator;
  int duration; // วินาที
  
  MediaContent({
    required this.title,
    required this.creator,
    required this.duration,
  });
  
  String get durationString {
    int mins = duration ~/ 60;
    int secs = duration % 60;
    return '${mins.toString().padLeft(2, '0')}:${secs.toString().padLeft(2, '0')}';
  }
}

class Video extends MediaContent {
  int resolution; // 480, 720, 1080, 4320
  bool hasSubtitle;
  
  Video({
    required super.title,
    required super.creator,
    required super.duration,
    this.resolution = 1080,
    this.hasSubtitle = false,
  });
  
  void downloadInResolution(int res) {
    print('ดาวน์โหลด "$title" ความละเอียด ${res}p');
  }
}

class Audio extends MediaContent {
  int bitrate; // 128, 256, 320 kbps
  String format; // mp3, flac, wav
  
  Audio({
    required super.title,
    required super.creator,
    required super.duration,
    this.bitrate = 256,
    this.format = 'mp3',
  });
  
  void adjustBass(int level) {
    print('ปรับ Bass ของ "$title" เป็น level $level');
  }
}

class Podcast extends Audio {
  int episodeNumber;
  String season;
  
  Podcast({
    required super.title,
    required super.creator,
    required super.duration,
    required this.episodeNumber,
    required this.season,
    super.bitrate,
  }) : super(format: 'mp3');
  
  void addToBookmark(int position) {
    print('บันทึก bookmark ที่ $position วินาที ใน "${title}"');
  }
}

void main() {
  List<MediaContent> library = [
    Video(title: 'บทเรียน Flutter', creator: 'อาจารย์สมชาย', duration: 3600, resolution: 1080),
    Audio(title: 'เพลงไทยสากล', creator: 'นักร้อง ก', duration: 240, bitrate: 320),
    Podcast(title: 'Tech Talk EP 10', creator: 'ทีม Dev', duration: 3000, episodeNumber: 10, season: 'S1'),
    Video(title: 'ภาพยนตร์ไทย', creator: 'กำกับ ข', duration: 7200, hasSubtitle: true),
    Podcast(title: 'Business Insight EP 5', creator: 'นักธุรกิจ ค', duration: 2400, episodeNumber: 5, season: 'S2'),
  ];
  
  print('=== Library ทั้งหมด ===');
  for (MediaContent content in library) {
    print('${content.runtimeType}: "${content.title}" (${content.durationString})');
  }
  
  print('\n=== Videos เท่านั้น ===');
  for (MediaContent content in library) {
    if (content is Video) {
      // หลังจาก is check แล้ว Dart auto-cast เป็น Video
      print('  ${content.title} - ${content.resolution}p');
      content.downloadInResolution(720); // เรียก Video-specific method ได้
    }
  }
  
  print('\n=== Podcasts เท่านั้น ===');
  for (MediaContent content in library) {
    if (content is Podcast) {
      print('  ${content.title} - ${content.season} EP${content.episodeNumber}');
    }
  }
  
  print('\n=== Audio content (รวม Podcast) ===');
  // Podcast extends Audio ดังนั้น is Audio = true สำหรับทั้ง Audio และ Podcast
  for (MediaContent content in library) {
    if (content is Audio) {
      print('  ${content.runtimeType}: "${content.title}" - ${content.bitrate}kbps');
    }
  }
  
  // as - forced type cast (ระวัง! จะ throw error ถ้า cast ไม่ได้)
  print('\n=== ใช้ as ===');
  MediaContent first = library.first;
  if (first is Video) {
    Video v = first as Video; // หลัง is check แล้ว as ไม่จำเป็น แต่ใช้ได้
    print('Video: ${v.title}, subtitle: ${v.hasSubtitle}');
  }
  
  // whereType - กรองด้วย type
  List<Video> videos = library.whereType<Video>().toList();
  List<Podcast> podcasts = library.whereType<Podcast>().toList();
  
  print('\nVideos: ${videos.length} รายการ');
  print('Podcasts: ${podcasts.length} รายการ');
  
  // สถิติแยกตามประเภท
  int totalVideoTime = videos.fold(0, (sum, v) => sum + v.duration);
  int totalPodcastTime = podcasts.fold(0, (sum, p) => sum + p.duration);
  
  print('เวลาดู video รวม: ${totalVideoTime ~/ 60} นาที');
  print('เวลาฟัง podcast รวม: ${totalPodcastTime ~/ 60} นาที');
}
```

---

## 8.7 Workshop: Shape Hierarchy

```dart
// shapes.dart
// ระบบรูปทรงเรขาคณิต - ใช้ Inheritance และ Polymorphism

import 'dart:math' as math;

// Abstract base class
abstract class Shape {
  String color;
  bool filled;
  
  Shape({this.color = 'ดำ', this.filled = false});
  
  // Abstract getters - ต้อง implement ใน subclass
  double get area;
  double get perimeter;
  String get name;
  
  // Concrete methods
  bool contains(double x, double y) => false; // Override ตามต้องการ
  
  Shape scale(double factor);   // Return new scaled shape
  Shape translate(double dx, double dy); // Return moved shape
  
  void describe() {
    print('=== $name ===');
    print('สี: $color (${filled ? "ทึบ" : "กรอบ"})');
    print('พื้นที่: ${area.toStringAsFixed(4)} ตร.หน่วย');
    print('เส้นรอบรูป: ${perimeter.toStringAsFixed(4)} หน่วย');
  }
  
  @override
  String toString() => '$name(area=${area.toStringAsFixed(2)})';
}

// Circle
class Circle extends Shape {
  double centerX;
  double centerY;
  final double radius;
  
  Circle({
    required this.radius,
    this.centerX = 0,
    this.centerY = 0,
    super.color,
    super.filled,
  }) {
    if (radius <= 0) throw ArgumentError('รัศมีต้องมากกว่า 0');
  }
  
  @override
  String get name => 'วงกลม';
  
  @override
  double get area => math.pi * radius * radius;
  
  @override
  double get perimeter => 2 * math.pi * radius;
  
  @override
  bool contains(double x, double y) {
    double dx = x - centerX;
    double dy = y - centerY;
    return (dx * dx + dy * dy) <= radius * radius;
  }
  
  @override
  Circle scale(double factor) {
    return Circle(
      radius: radius * factor,
      centerX: centerX,
      centerY: centerY,
      color: color,
      filled: filled,
    );
  }
  
  @override
  Circle translate(double dx, double dy) {
    return Circle(
      radius: radius,
      centerX: centerX + dx,
      centerY: centerY + dy,
      color: color,
      filled: filled,
    );
  }
  
  @override
  void describe() {
    super.describe();
    print('รัศมี: $radius');
    print('ศูนย์กลาง: ($centerX, $centerY)');
  }
}

// Rectangle
class Rectangle extends Shape {
  double x;
  double y;
  final double width;
  final double height;
  
  Rectangle({
    required this.width,
    required this.height,
    this.x = 0,
    this.y = 0,
    super.color,
    super.filled,
  }) {
    if (width <= 0 || height <= 0) throw ArgumentError('กว้างและสูงต้องมากกว่า 0');
  }
  
  @override
  String get name => 'สี่เหลี่ยมผืนผ้า';
  
  @override
  double get area => width * height;
  
  @override
  double get perimeter => 2 * (width + height);
  
  bool get isSquare => width == height;
  
  double get diagonal => math.sqrt(width * width + height * height);
  
  @override
  bool contains(double px, double py) {
    return px >= x && px <= x + width && py >= y && py <= y + height;
  }
  
  @override
  Rectangle scale(double factor) {
    return Rectangle(
      width: width * factor,
      height: height * factor,
      x: x,
      y: y,
      color: color,
      filled: filled,
    );
  }
  
  @override
  Rectangle translate(double dx, double dy) {
    return Rectangle(
      width: width,
      height: height,
      x: x + dx,
      y: y + dy,
      color: color,
      filled: filled,
    );
  }
  
  @override
  void describe() {
    super.describe();
    print('กว้าง: $width, สูง: $height');
    print('ตำแหน่ง: ($x, $y)');
    print('เส้นทแยงมุม: ${diagonal.toStringAsFixed(4)}');
    if (isSquare) print('(เป็นจตุรัส)');
  }
}

// Square - special case of Rectangle
class Square extends Rectangle {
  Square({
    required double side,
    super.x,
    super.y,
    super.color,
    super.filled,
  }) : super(width: side, height: side);
  
  double get side => width;
  
  @override
  String get name => 'จตุรัส';
  
  @override
  Square scale(double factor) {
    return Square(
      side: side * factor,
      x: x,
      y: y,
      color: color,
      filled: filled,
    );
  }
  
  @override
  Square translate(double dx, double dy) {
    return Square(
      side: side,
      x: x + dx,
      y: y + dy,
      color: color,
      filled: filled,
    );
  }
}

// Triangle
class Triangle extends Shape {
  final double x1, y1; // จุดยอดที่ 1
  final double x2, y2; // จุดยอดที่ 2
  final double x3, y3; // จุดยอดที่ 3
  
  Triangle({
    required this.x1, required this.y1,
    required this.x2, required this.y2,
    required this.x3, required this.y3,
    super.color,
    super.filled,
  });
  
  // Factory constructors
  factory Triangle.equilateral({
    required double side,
    double cx = 0,
    double cy = 0,
    String color = 'ดำ',
    bool filled = false,
  }) {
    double h = side * math.sqrt(3) / 2;
    return Triangle(
      x1: cx, y1: cy + h * 2 / 3,
      x2: cx - side / 2, y2: cy - h / 3,
      x3: cx + side / 2, y3: cy - h / 3,
      color: color,
      filled: filled,
    );
  }
  
  factory Triangle.rightAngle({
    required double base,
    required double height,
    double x = 0,
    double y = 0,
    String color = 'ดำ',
    bool filled = false,
  }) {
    return Triangle(
      x1: x, y1: y,
      x2: x + base, y2: y,
      x3: x, y3: y + height,
      color: color,
      filled: filled,
    );
  }
  
  @override
  String get name => 'สามเหลี่ยม';
  
  // คำนวณด้วย Heron's formula
  double get _sideA => math.sqrt((x2-x3)*(x2-x3) + (y2-y3)*(y2-y3));
  double get _sideB => math.sqrt((x1-x3)*(x1-x3) + (y1-y3)*(y1-y3));
  double get _sideC => math.sqrt((x1-x2)*(x1-x2) + (y1-y2)*(y1-y2));
  
  @override
  double get perimeter => _sideA + _sideB + _sideC;
  
  @override
  double get area {
    // Shoelace formula
    return ((x1 * (y2 - y3) + x2 * (y3 - y1) + x3 * (y1 - y2)) / 2).abs();
  }
  
  String get triangleType {
    double a = _sideA, b = _sideB, c = _sideC;
    if ((a - b).abs() < 0.001 && (b - c).abs() < 0.001) return 'ด้านเท่า';
    if ((a - b).abs() < 0.001 || (b - c).abs() < 0.001 || (a - c).abs() < 0.001) return 'หน้าจั่ว';
    return 'ด้านไม่เท่า';
  }
  
  @override
  Triangle scale(double factor) {
    double cx = (x1 + x2 + x3) / 3;
    double cy = (y1 + y2 + y3) / 3;
    return Triangle(
      x1: cx + (x1 - cx) * factor, y1: cy + (y1 - cy) * factor,
      x2: cx + (x2 - cx) * factor, y2: cy + (y2 - cy) * factor,
      x3: cx + (x3 - cx) * factor, y3: cy + (y3 - cy) * factor,
      color: color,
      filled: filled,
    );
  }
  
  @override
  Triangle translate(double dx, double dy) {
    return Triangle(
      x1: x1 + dx, y1: y1 + dy,
      x2: x2 + dx, y2: y2 + dy,
      x3: x3 + dx, y3: y3 + dy,
      color: color,
      filled: filled,
    );
  }
  
  @override
  void describe() {
    super.describe();
    print('ด้าน: ${_sideA.toStringAsFixed(2)}, ${_sideB.toStringAsFixed(2)}, ${_sideC.toStringAsFixed(2)}');
    print('ประเภท: $triangleType');
  }
}

// ShapeCanvas - ใช้ polymorphism เต็มรูปแบบ
class ShapeCanvas {
  final List<Shape> _shapes = [];
  final double width;
  final double height;
  
  ShapeCanvas({this.width = 100, this.height = 100});
  
  void addShape(Shape shape) {
    _shapes.add(shape);
    print('เพิ่ม ${shape.name} ลงใน Canvas');
  }
  
  void removeShape(Shape shape) {
    _shapes.remove(shape);
  }
  
  // Polymorphism - วาดทุกรูป
  void drawAll() {
    print('\n=== Canvas ${width}x${height} ===');
    print('จำนวนรูป: ${_shapes.length}');
    for (int i = 0; i < _shapes.length; i++) {
      print('\nรูปที่ ${i+1}:');
      _shapes[i].describe(); // Polymorphism!
    }
  }
  
  // คำนวณสถิติ
  double get totalArea => _shapes.fold(0, (sum, s) => sum + s.area);
  
  Shape? get largestShape {
    if (_shapes.isEmpty) return null;
    return _shapes.reduce((a, b) => a.area > b.area ? a : b);
  }
  
  Shape? get smallestShape {
    if (_shapes.isEmpty) return null;
    return _shapes.reduce((a, b) => a.area < b.area ? a : b);
  }
  
  List<Shape> getShapesByType<T extends Shape>() {
    return _shapes.whereType<T>().toList();
  }
  
  Map<String, List<Shape>> groupByType() {
    Map<String, List<Shape>> groups = {};
    for (Shape shape in _shapes) {
      groups.putIfAbsent(shape.name, () => []);
      groups[shape.name]!.add(shape);
    }
    return groups;
  }
  
  List<Shape> scaleAll(double factor) {
    return _shapes.map((s) => s.scale(factor)).toList();
  }
  
  void printStats() {
    print('\n=== สถิติ Canvas ===');
    print('รูปทั้งหมด: ${_shapes.length} รูป');
    print('พื้นที่รวม: ${totalArea.toStringAsFixed(2)} ตร.หน่วย');
    
    if (largestShape != null) {
      print('รูปใหญ่สุด: ${largestShape!.name} (${largestShape!.area.toStringAsFixed(2)})');
    }
    if (smallestShape != null) {
      print('รูปเล็กสุด: ${smallestShape!.name} (${smallestShape!.area.toStringAsFixed(2)})');
    }
    
    print('\nแยกตามประเภท:');
    groupByType().forEach((type, shapes) {
      double typeArea = shapes.fold(0, (sum, s) => sum + s.area);
      print('  $type: ${shapes.length} รูป (พื้นที่รวม: ${typeArea.toStringAsFixed(2)})');
    });
  }
}

void main() {
  print('=== ระบบ Shape Canvas ===\n');
  
  ShapeCanvas canvas = ShapeCanvas(width: 500, height: 500);
  
  // เพิ่มรูปทรงต่างๆ
  canvas.addShape(Circle(radius: 10, centerX: 50, centerY: 50, color: 'แดง', filled: true));
  canvas.addShape(Circle(radius: 5, centerX: 100, centerY: 100, color: 'ชมพู'));
  canvas.addShape(Rectangle(width: 20, height: 30, x: 10, y: 10, color: 'น้ำเงิน', filled: true));
  canvas.addShape(Square(side: 15, x: 200, y: 200, color: 'เขียว'));
  canvas.addShape(Triangle.equilateral(side: 12, cx: 300, cy: 300, color: 'เหลือง', filled: true));
  canvas.addShape(Triangle.rightAngle(base: 10, height: 8, x: 400, y: 400, color: 'ม่วง'));
  
  // วาดทุกรูป (ใช้ polymorphism)
  canvas.drawAll();
  
  // แสดงสถิติ
  canvas.printStats();
  
  // ทดสอบ contains
  Circle c = Circle(radius: 10, centerX: 50, centerY: 50);
  print('\n=== ทดสอบ contains ===');
  print('จุด (55, 55) อยู่ในวงกลม: ${c.contains(55, 55)}'); // true
  print('จุด (70, 70) อยู่ในวงกลม: ${c.contains(70, 70)}'); // false
  
  // Scale
  print('\n=== Scale รูปทั้งหมด x2 ===');
  List<Shape> scaled = canvas.scaleAll(2.0);
  for (Shape s in scaled.take(3)) {
    print('${s.name}: พื้นที่ = ${s.area.toStringAsFixed(2)}');
  }
  
  // ดูเฉพาะ Circles
  print('\n=== Circles เท่านั้น ===');
  List<Circle> circles = canvas.getShapesByType<Circle>();
  for (Circle circle in circles) {
    print('วงกลม รัศมี: ${circle.radius} สี: ${circle.color}');
  }
}
```

---

## สรุป Part 08

ใน Part นี้เราได้เรียนรู้:

### Inheritance Concepts:
| Keyword | ความหมาย |
|---------|---------|
| `extends` | สืบทอดจาก class แม่ |
| `super` | เรียก constructor/method ของ parent |
| `@override` | บอกว่า override method ของ parent |
| `abstract` | class/method ที่ต้อง implement ใน subclass |

### Polymorphism:
- Object ของ subclass สามารถใช้ในที่ที่ต้องการ superclass ได้
- Method ที่เรียกจะเป็น method ของ subclass ที่แท้จริง (runtime dispatch)
- ทำให้โค้ดยืดหยุ่น เพิ่ม type ใหม่ได้โดยไม่แก้ code เดิม

### Type Checking:
- `is` - ตรวจสอบ type (auto-cast หลัง check)
- `as` - บังคับ cast (ระวัง throws error)
- `whereType<T>()` - กรอง collection ตาม type
- `runtimeType` - ดู type จริงๆ ของ object

## ➡️ Part ถัดไป

**Part 09: Mixins and Interfaces** - เราจะเรียนรู้การใช้ mixin เพื่อ code reuse, interface pattern ใน Dart และความแตกต่างระหว่าง extends, implements, with
