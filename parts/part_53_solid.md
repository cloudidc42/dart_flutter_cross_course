# Part 53: SOLID Principles ใน Flutter

## บทนำ

SOLID เป็นหลักการออกแบบซอฟต์แวร์ 5 ข้อที่ช่วยให้ code มีคุณภาพดี บำรุงรักษาได้ง่าย และ scale ได้ ย่อมาจาก:

- **S** - Single Responsibility Principle
- **O** - Open/Closed Principle
- **L** - Liskov Substitution Principle
- **I** - Interface Segregation Principle
- **D** - Dependency Inversion Principle

---

## 53.1 Single Responsibility Principle (SRP)

> "Class ควรมีเหตุผลเดียวที่จะต้องเปลี่ยน"

### ❌ ละเมิด SRP

```dart
// UserManager ทำหลายอย่างเกินไป
class UserManager {
  final FirebaseFirestore _db = FirebaseFirestore.instance;
  
  // ดึงข้อมูล user
  Future<User> getUser(String id) async {
    final doc = await _db.collection('users').doc(id).get();
    return User.fromJson(doc.data()!);
  }
  
  // Validate user data
  bool validateEmail(String email) {
    return RegExp(r'^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$')
        .hasMatch(email);
  }
  
  // Send email (ทำเองเลย!)
  Future<void> sendWelcomeEmail(String email, String name) async {
    // ... HTTP call to email service
    print('Sending email to $email');
  }
  
  // Format data เพื่อแสดงผล
  String formatUserDisplayName(User user) {
    return '${user.firstName} ${user.lastName}';
  }
  
  // Log activity
  void logUserActivity(String userId, String action) {
    print('[${DateTime.now()}] User $userId: $action');
  }
}
```

### ✅ ถูกต้องตาม SRP

```dart
// แยก responsibilities ออกไป

// 1. UserRepository - จัดการ data access
class UserRepository {
  final FirebaseFirestore _db;
  
  const UserRepository({required FirebaseFirestore db}) : _db = db;
  
  Future<User?> findById(String id) async {
    final doc = await _db.collection('users').doc(id).get();
    if (!doc.exists) return null;
    return User.fromJson(doc.data()!);
  }
  
  Future<void> save(User user) async {
    await _db.collection('users').doc(user.id).set(user.toJson());
  }
}

// 2. UserValidator - จัดการ validation
class UserValidator {
  static bool isValidEmail(String email) {
    return RegExp(r'^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$')
        .hasMatch(email);
  }
  
  static bool isValidPhone(String phone) {
    return RegExp(r'^[0-9]{10}$').hasMatch(phone.replaceAll('-', ''));
  }
  
  static ValidationResult validateUser(User user) {
    final errors = <String>[];
    
    if (user.firstName.isEmpty) errors.add('กรุณากรอกชื่อ');
    if (user.email.isEmpty) errors.add('กรุณากรอก email');
    if (!isValidEmail(user.email)) errors.add('Email ไม่ถูกต้อง');
    
    return ValidationResult(isValid: errors.isEmpty, errors: errors);
  }
}

// 3. EmailService - จัดการ email
class EmailService {
  Future<void> sendWelcomeEmail({
    required String to,
    required String name,
  }) async {
    // HTTP call to email provider
  }
  
  Future<void> sendPasswordResetEmail({required String to}) async {
    // ...
  }
}

// 4. UserFormatter - จัดการ formatting
class UserFormatter {
  static String displayName(User user) {
    return '${user.firstName} ${user.lastName}'.trim();
  }
  
  static String initials(User user) {
    final first = user.firstName.isNotEmpty ? user.firstName[0] : '';
    final last = user.lastName.isNotEmpty ? user.lastName[0] : '';
    return '$first$last'.toUpperCase();
  }
}

// 5. ActivityLogger - จัดการ logging
class ActivityLogger {
  void log(String userId, String action, {Map<String, dynamic>? data}) {
    final timestamp = DateTime.now().toIso8601String();
    print('[$timestamp] User $userId: $action ${data ?? ''}');
    // สามารถ save ไปยัง analytics service ได้
  }
}

// SRP ช่วยให้แต่ละ class เปลี่ยนได้โดยไม่กระทบส่วนอื่น
// เช่น เปลี่ยน email provider ก็แก้แค่ EmailService
```

---

## 53.2 Open/Closed Principle (OCP)

> "Software entities ควร open สำหรับ extension แต่ closed สำหรับ modification"

### ❌ ละเมิด OCP

```dart
// ต้องแก้ไข class เดิมเพื่อเพิ่ม discount type ใหม่
class OrderCalculator {
  double calculateDiscount(Order order, String discountType) {
    if (discountType == 'PERCENTAGE') {
      return order.total * 0.1;
    } else if (discountType == 'FIXED') {
      return 50.0;
    } else if (discountType == 'BUY_ONE_GET_ONE') {
      return order.total / 2;
    }
    // ต้องแก้ไข method นี้ทุกครั้งที่เพิ่ม type ใหม่!
    return 0;
  }
}
```

### ✅ ถูกต้องตาม OCP

```dart
// Abstract discount strategy
abstract class DiscountStrategy {
  double calculate(Order order);
  String get description;
  bool isApplicable(Order order);
}

// Concrete implementations - เพิ่มได้โดยไม่แก้ class เดิม
class PercentageDiscount implements DiscountStrategy {
  final double percentage;
  
  const PercentageDiscount({required this.percentage});
  
  @override
  double calculate(Order order) => order.total * (percentage / 100);
  
  @override
  String get description => 'ส่วนลด ${percentage.toStringAsFixed(0)}%';
  
  @override
  bool isApplicable(Order order) => order.total > 0;
}

class FixedDiscount implements DiscountStrategy {
  final double amount;
  
  const FixedDiscount({required this.amount});
  
  @override
  double calculate(Order order) => amount;
  
  @override
  String get description => 'ส่วนลด ฿${amount.toStringAsFixed(0)}';
  
  @override
  bool isApplicable(Order order) => order.total >= amount;
}

class BuyOneGetOneDiscount implements DiscountStrategy {
  @override
  double calculate(Order order) {
    final cheapestItemPrice = order.items
        .map((item) => item.price)
        .reduce((a, b) => a < b ? a : b);
    return cheapestItemPrice;
  }
  
  @override
  String get description => 'ซื้อ 1 แถม 1';
  
  @override
  bool isApplicable(Order order) => order.items.length >= 2;
}

// เพิ่ม discount type ใหม่โดยไม่แก้ class เดิม!
class FreeShippingDiscount implements DiscountStrategy {
  final double shippingCost;
  
  const FreeShippingDiscount({required this.shippingCost});
  
  @override
  double calculate(Order order) => shippingCost;
  
  @override
  String get description => 'ส่งฟรี';
  
  @override
  bool isApplicable(Order order) => order.total >= 500;
}

// OrderCalculator ไม่ต้องแก้ไข
class OrderCalculator {
  final List<DiscountStrategy> strategies;
  
  const OrderCalculator({required this.strategies});
  
  double calculateTotalDiscount(Order order) {
    return strategies
        .where((s) => s.isApplicable(order))
        .fold(0.0, (total, s) => total + s.calculate(order));
  }
}
```

---

## 53.3 Liskov Substitution Principle (LSP)

> "Objects ของ subclass ต้องสามารถใช้แทน objects ของ base class ได้ โดยไม่เปลี่ยนพฤติกรรมที่ถูกต้อง"

### ❌ ละเมิด LSP

```dart
class Rectangle {
  double width;
  double height;
  
  Rectangle({required this.width, required this.height});
  
  double get area => width * height;
}

// Square ละเมิด LSP เพราะเปลี่ยนพฤติกรรมของ Rectangle
class Square extends Rectangle {
  Square({required double side}) : super(width: side, height: side);
  
  @override
  set width(double value) {
    super.width = value;
    super.height = value; // เปลี่ยนทั้งคู่ ต่างจาก Rectangle!
  }
  
  @override
  set height(double value) {
    super.width = value;
    super.height = value;
  }
}

void resizeAndPrint(Rectangle rect) {
  rect.width = 10;
  rect.height = 5;
  print(rect.area); // Rectangle: 50, Square: 25 (ผิดความคาดหวัง!)
}
```

### ✅ ถูกต้องตาม LSP

```dart
// ใช้ abstraction แทน inheritance
abstract class Shape {
  double get area;
  double get perimeter;
  String get description;
}

class Rectangle extends Shape {
  final double width;
  final double height;
  
  const Rectangle({required this.width, required this.height});
  
  @override
  double get area => width * height;
  
  @override
  double get perimeter => 2 * (width + height);
  
  @override
  String get description => 'สี่เหลี่ยม ${width}x$height';
}

class Square extends Shape {
  final double side;
  
  const Square({required this.side});
  
  @override
  double get area => side * side;
  
  @override
  double get perimeter => 4 * side;
  
  @override
  String get description => 'จัตุรัส $side';
}

class Circle extends Shape {
  final double radius;
  
  const Circle({required this.radius});
  
  @override
  double get area => pi * radius * radius;
  
  @override
  double get perimeter => 2 * pi * radius;
  
  @override
  String get description => 'วงกลม r=$radius';
}

// ทุก Shape สามารถใช้แทนกันได้
void printShapeInfo(Shape shape) {
  print('${shape.description}: area=${shape.area.toStringAsFixed(2)}');
}

// Widget ตัวอย่าง
class ShapeWidget extends StatelessWidget {
  final Shape shape; // ใช้ base class
  
  const ShapeWidget({super.key, required this.shape});
  
  @override
  Widget build(BuildContext context) {
    return ListTile(
      leading: _buildShapeIcon(),
      title: Text(shape.description),
      subtitle: Text('พื้นที่: ${shape.area.toStringAsFixed(2)}'),
    );
  }
  
  Widget _buildShapeIcon() {
    if (shape is Circle) {
      return const Icon(Icons.circle, color: Colors.blue);
    } else {
      return const Icon(Icons.square, color: Colors.green);
    }
  }
}
```

---

## 53.4 Interface Segregation Principle (ISP)

> "Clients ไม่ควรถูกบังคับให้ depend on interfaces ที่ไม่ได้ใช้"

### ❌ ละเมิด ISP

```dart
// Interface ที่ใหญ่เกินไป - บังคับทุก class implement ทุกอย่าง
abstract class UserActions {
  Future<User> getUser(String id);
  Future<void> updateUser(User user);
  Future<void> deleteUser(String id);
  Future<void> sendEmail(String to, String subject, String body);
  Future<void> sendPushNotification(String userId, String message);
  Future<List<Order>> getUserOrders(String userId);
  Future<void> processPayment(Payment payment);
  Future<void> generateReport(ReportType type);
}

// AdminService ต้อง implement ทุกอย่างแม้บางอย่างไม่เกี่ยว
class AdminService implements UserActions {
  @override
  Future<User> getUser(String id) async { /* ... */ return User(); }
  
  @override
  Future<void> sendEmail(String to, String subject, String body) async {
    throw UnimplementedError(); // Admin ไม่ได้ send email เอง!
  }
  
  @override
  Future<void> processPayment(Payment payment) async {
    throw UnimplementedError(); // Admin ไม่ได้ process payment!
  }
  // ... implement ทุกอย่างแม้ไม่ต้องการ
}
```

### ✅ ถูกต้องตาม ISP

```dart
// แยก interfaces ตาม responsibility

abstract class UserReadable {
  Future<User?> findById(String id);
  Future<List<User>> findAll();
}

abstract class UserWritable {
  Future<void> create(User user);
  Future<void> update(User user);
  Future<void> delete(String id);
}

abstract class EmailSender {
  Future<void> sendEmail(String to, String subject, String body);
}

abstract class NotificationSender {
  Future<void> sendPushNotification(String userId, String message);
}

abstract class PaymentProcessor {
  Future<void> processPayment(Payment payment);
  Future<void> refundPayment(String paymentId);
}

abstract class ReportGenerator {
  Future<void> generateUserReport();
  Future<void> generateSalesReport();
}

// Implement เฉพาะที่ต้องการ
class AdminService implements UserReadable, UserWritable, ReportGenerator {
  @override
  Future<User?> findById(String id) async { /* ... */ return null; }
  
  @override
  Future<List<User>> findAll() async { /* ... */ return []; }
  
  @override
  Future<void> create(User user) async { /* ... */ }
  
  @override
  Future<void> update(User user) async { /* ... */ }
  
  @override
  Future<void> delete(String id) async { /* ... */ }
  
  @override
  Future<void> generateUserReport() async { /* ... */ }
  
  @override
  Future<void> generateSalesReport() async { /* ... */ }
}

class NotificationService implements EmailSender, NotificationSender {
  @override
  Future<void> sendEmail(String to, String subject, String body) async {
    // ส่ง email จริงๆ
  }
  
  @override
  Future<void> sendPushNotification(String userId, String message) async {
    // ส่ง push notification จริงๆ
  }
}

// Widget ใช้ interfaces ที่เหมาะสม
class UserListWidget extends StatelessWidget {
  final UserReadable userService; // ใช้เฉพาะ interface ที่ต้องการ
  
  const UserListWidget({super.key, required this.userService});
  
  @override
  Widget build(BuildContext context) {
    return FutureBuilder<List<User>>(
      future: userService.findAll(),
      builder: (context, snapshot) {
        if (!snapshot.hasData) {
          return const CircularProgressIndicator();
        }
        return ListView.builder(
          itemCount: snapshot.data!.length,
          itemBuilder: (context, index) => ListTile(
            title: Text(snapshot.data![index].name),
          ),
        );
      },
    );
  }
}
```

---

## 53.5 Dependency Inversion Principle (DIP)

> "High-level modules ไม่ควร depend on low-level modules ทั้งคู่ควร depend on abstractions"

### ❌ ละเมิด DIP

```dart
// OrderService depend directly บน concrete implementations
class OrderService {
  // Depend on concrete classes!
  final FirestoreOrderRepository _orderRepo = FirestoreOrderRepository();
  final StripePaymentGateway _payment = StripePaymentGateway();
  final SendgridEmailService _email = SendgridEmailService();
  
  Future<void> placeOrder(Order order) async {
    await _orderRepo.save(order); // ผูกกับ Firestore
    await _payment.charge(order.total); // ผูกกับ Stripe
    await _email.sendConfirmation(order.userId); // ผูกกับ Sendgrid
  }
}
```

### ✅ ถูกต้องตาม DIP

```dart
// Abstract interfaces (abstractions)
abstract class OrderRepository {
  Future<void> save(Order order);
  Future<Order?> findById(String id);
  Future<List<Order>> findByCustomer(String customerId);
}

abstract class PaymentGateway {
  Future<PaymentResult> charge({
    required double amount,
    required String currency,
    required PaymentMethod paymentMethod,
  });
  Future<void> refund(String transactionId);
}

abstract class EmailNotifier {
  Future<void> sendOrderConfirmation(Order order);
  Future<void> sendOrderCancellation(Order order);
}

// High-level module depend on abstractions
class OrderService {
  final OrderRepository _orderRepo;
  final PaymentGateway _payment;
  final EmailNotifier _emailNotifier;
  
  // Inject dependencies (Dependency Injection)
  const OrderService({
    required OrderRepository orderRepository,
    required PaymentGateway paymentGateway,
    required EmailNotifier emailNotifier,
  })  : _orderRepo = orderRepository,
        _payment = paymentGateway,
        _emailNotifier = emailNotifier;
  
  Future<OrderResult> placeOrder({
    required Order order,
    required PaymentMethod paymentMethod,
  }) async {
    try {
      // Process payment
      final paymentResult = await _payment.charge(
        amount: order.total,
        currency: 'THB',
        paymentMethod: paymentMethod,
      );
      
      if (!paymentResult.isSuccessful) {
        return OrderResult.failed(reason: 'การชำระเงินล้มเหลว');
      }
      
      // Save order
      final confirmedOrder = order.confirm();
      await _orderRepo.save(confirmedOrder);
      
      // Send confirmation
      await _emailNotifier.sendOrderConfirmation(confirmedOrder);
      
      return OrderResult.success(order: confirmedOrder);
    } catch (e) {
      return OrderResult.failed(reason: e.toString());
    }
  }
}

// Low-level implementations - สามารถเปลี่ยนได้โดยไม่แก้ OrderService
class FirestoreOrderRepository implements OrderRepository {
  final FirebaseFirestore _db;
  
  const FirestoreOrderRepository({required FirebaseFirestore db}) 
      : _db = db;
  
  @override
  Future<void> save(Order order) async {
    await _db.collection('orders').doc(order.id).set(order.toJson());
  }
  
  @override
  Future<Order?> findById(String id) async {
    final doc = await _db.collection('orders').doc(id).get();
    if (!doc.exists) return null;
    return Order.fromJson(doc.data()!);
  }
  
  @override
  Future<List<Order>> findByCustomer(String customerId) async {
    final snapshot = await _db
        .collection('orders')
        .where('customerId', isEqualTo: customerId)
        .get();
    return snapshot.docs
        .map((doc) => Order.fromJson(doc.data()))
        .toList();
  }
}

// Mock สำหรับ testing
class MockOrderRepository implements OrderRepository {
  final List<Order> _orders = [];
  
  @override
  Future<void> save(Order order) async {
    _orders.removeWhere((o) => o.id == order.id);
    _orders.add(order);
  }
  
  @override
  Future<Order?> findById(String id) async {
    return _orders.firstWhere(
      (o) => o.id == id,
      orElse: () => throw NotFoundException('Order not found'),
    );
  }
  
  @override
  Future<List<Order>> findByCustomer(String customerId) async {
    return _orders.where((o) => o.customerId == customerId).toList();
  }
}

// ใช้งานผ่าน DI Container
void setupDependencies() {
  if (isProduction) {
    GetIt.I.registerLazySingleton<OrderRepository>(
      () => FirestoreOrderRepository(db: FirebaseFirestore.instance),
    );
    GetIt.I.registerLazySingleton<PaymentGateway>(
      () => StripePaymentGateway(apiKey: Config.stripeApiKey),
    );
  } else {
    GetIt.I.registerLazySingleton<OrderRepository>(
      () => MockOrderRepository(),
    );
    GetIt.I.registerLazySingleton<PaymentGateway>(
      () => MockPaymentGateway(),
    );
  }
}
```

---

## 53.6 SOLID ใน Widget Tree

```dart
// SRP: แยก widget ที่มี responsibility เดียว
class UserAvatar extends StatelessWidget {
  final String? photoUrl;
  final String name;
  final double radius;
  
  const UserAvatar({
    super.key,
    this.photoUrl,
    required this.name,
    this.radius = 24,
  });
  
  @override
  Widget build(BuildContext context) {
    // รับผิดชอบแค่แสดง avatar
    return CircleAvatar(/* ... */);
  }
}

// OCP: สร้าง base button ที่ extend ได้
abstract class AppButton extends StatelessWidget {
  final String label;
  final VoidCallback? onPressed;
  final bool isLoading;
  
  const AppButton({
    super.key,
    required this.label,
    this.onPressed,
    this.isLoading = false,
  });
  
  Color get backgroundColor;
  Color get foregroundColor;
  
  @override
  Widget build(BuildContext context) {
    return ElevatedButton(
      onPressed: isLoading ? null : onPressed,
      style: ElevatedButton.styleFrom(
        backgroundColor: backgroundColor,
        foregroundColor: foregroundColor,
      ),
      child: isLoading
          ? const SizedBox(
              width: 20,
              height: 20,
              child: CircularProgressIndicator(strokeWidth: 2),
            )
          : Text(label),
    );
  }
}

class PrimaryButton extends AppButton {
  const PrimaryButton({
    super.key,
    required super.label,
    super.onPressed,
    super.isLoading,
  });
  
  @override
  Color get backgroundColor => Colors.blue;
  
  @override
  Color get foregroundColor => Colors.white;
}

class DangerButton extends AppButton {
  const DangerButton({
    super.key,
    required super.label,
    super.onPressed,
    super.isLoading,
  });
  
  @override
  Color get backgroundColor => Colors.red;
  
  @override
  Color get foregroundColor => Colors.white;
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้ SOLID Principles ทั้ง 5 ข้อ:

| หลักการ | สาระสำคัญ |
|---------|-----------|
| **SRP** | Class มีหน้าที่เดียว |
| **OCP** | เพิ่มได้ แต่ไม่แก้ของเดิม |
| **LSP** | subclass ใช้แทน base class ได้ |
| **ISP** | Interface เล็กและเฉพาะเจาะจง |
| **DIP** | Depend บน abstraction ไม่ใช่ concrete |

**แบบฝึกหัดเพิ่มเติม:**
1. Refactor authentication code ให้ตาม SOLID
2. สร้าง form validation ที่ตาม OCP
3. Implement notification system ที่ตาม DIP
4. Review code เดิมแล้วระบุว่าละเมิดหลักการอะไร
