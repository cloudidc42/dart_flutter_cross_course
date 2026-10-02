# Part 52: Domain-Driven Design (DDD)

## บทนำ

Domain-Driven Design (DDD) เป็นแนวทางการออกแบบซอฟต์แวร์ที่เน้นการโฟกัสที่ domain (ธุรกิจ/ปัญหาที่แก้) เป็นหลัก โดยให้ domain experts และ developers ทำงานร่วมกันโดยใช้ภาษาเดียวกัน (Ubiquitous Language)

---

## 52.1 หลักการพื้นฐาน

### Ubiquitous Language

```
❌ คำศัพท์ Developer: "userId record", "orderData", "productEntry"
✅ คำศัพท์ Domain (Ubiquitous Language): "Customer", "Order", "Product"

ทุกคนในทีมใช้คำเดียวกัน ทั้งใน code, document และการพูดคุย
```

### Building Blocks

```
DDD Building Blocks
├── Entities         - มี identity ที่ไม่เปลี่ยน
├── Value Objects    - ไม่มี identity, กำหนดด้วยค่า
├── Aggregates       - กลุ่มของ entities/value objects
├── Domain Services  - logic ที่ไม่เหมาะกับ entity
├── Domain Events    - เหตุการณ์ที่เกิดขึ้นใน domain
├── Repositories     - เข้าถึง aggregates
└── Factories        - สร้าง complex objects
```

---

## 52.2 Entities และ Value Objects

### Entity

```dart
// Entity มี unique identity
// ตัวอย่าง: Customer, Order, Product

abstract class Entity<T> {
  final T id;
  
  const Entity({required this.id});
  
  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    if (other.runtimeType != runtimeType) return false;
    return other is Entity<T> && other.id == id;
  }
  
  @override
  int get hashCode => id.hashCode;
}

// Customer Entity
class Customer extends Entity<CustomerId> {
  final CustomerName name;
  final EmailAddress email;
  final PhoneNumber? phone;
  final Address? address;
  final CustomerStatus status;
  final DateTime registeredAt;
  
  Customer({
    required CustomerId id,
    required this.name,
    required this.email,
    this.phone,
    this.address,
    this.status = CustomerStatus.active,
    required this.registeredAt,
  }) : super(id: id);
  
  Customer activate() => Customer(
    id: id,
    name: name,
    email: email,
    phone: phone,
    address: address,
    status: CustomerStatus.active,
    registeredAt: registeredAt,
  );
  
  Customer deactivate() => Customer(
    id: id,
    name: name,
    email: email,
    phone: phone,
    address: address,
    status: CustomerStatus.inactive,
    registeredAt: registeredAt,
  );
  
  Customer updateAddress(Address newAddress) => Customer(
    id: id,
    name: name,
    email: email,
    phone: phone,
    address: newAddress,
    status: status,
    registeredAt: registeredAt,
  );
}

enum CustomerStatus { active, inactive, suspended }
```

### Value Objects

```dart
// Value Objects - ไม่มี identity, เปรียบเทียบด้วยค่า

abstract class ValueObject<T> {
  const ValueObject();
  
  T get value;
  
  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is ValueObject<T> && other.value == value;
  }
  
  @override
  int get hashCode => value.hashCode;
  
  @override
  String toString() => 'ValueObject($value)';
}

// CustomerId
class CustomerId extends ValueObject<String> {
  @override
  final String value;
  
  const CustomerId(this.value);
  
  factory CustomerId.generate() {
    return CustomerId(
      DateTime.now().millisecondsSinceEpoch.toString(),
    );
  }
  
  static CustomerId? tryParse(String? value) {
    if (value == null || value.isEmpty) return null;
    return CustomerId(value);
  }
}

// EmailAddress Value Object พร้อม validation
class EmailAddress extends ValueObject<String> {
  @override
  final String value;
  
  const EmailAddress._(this.value);
  
  factory EmailAddress.create(String email) {
    if (!_isValid(email)) {
      throw const InvalidEmailException('รูปแบบ email ไม่ถูกต้อง');
    }
    return EmailAddress._(email.toLowerCase().trim());
  }
  
  static bool _isValid(String email) {
    final regex = RegExp(r'^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$');
    return regex.hasMatch(email.trim());
  }
}

// Money Value Object
class Money extends ValueObject<double> {
  @override
  final double value;
  final Currency currency;
  
  const Money({required this.value, required this.currency});
  
  factory Money.zero({Currency currency = Currency.thb}) {
    return Money(value: 0, currency: currency);
  }
  
  factory Money.fromAmount(double amount, {Currency currency = Currency.thb}) {
    if (amount < 0) {
      throw const InvalidMoneyException('จำนวนเงินต้องไม่ติดลบ');
    }
    return Money(value: amount, currency: currency);
  }
  
  Money add(Money other) {
    if (currency != other.currency) {
      throw const CurrencyMismatchException('สกุลเงินไม่ตรงกัน');
    }
    return Money(value: value + other.value, currency: currency);
  }
  
  Money subtract(Money other) {
    if (currency != other.currency) {
      throw const CurrencyMismatchException('สกุลเงินไม่ตรงกัน');
    }
    if (value < other.value) {
      throw const InsufficientFundsException('เงินไม่เพียงพอ');
    }
    return Money(value: value - other.value, currency: currency);
  }
  
  Money multiply(double factor) {
    return Money(value: value * factor, currency: currency);
  }
  
  bool get isZero => value == 0;
  bool get isPositive => value > 0;
  
  String get formatted {
    switch (currency) {
      case Currency.thb:
        return '฿${value.toStringAsFixed(2)}';
      case Currency.usd:
        return '\$${value.toStringAsFixed(2)}';
    }
  }
  
  @override
  bool operator ==(Object other) {
    return other is Money && 
           other.value == value && 
           other.currency == currency;
  }
  
  @override
  int get hashCode => Object.hash(value, currency);
}

enum Currency { thb, usd }

// Address Value Object
class Address extends ValueObject<Map<String, String>> {
  final String street;
  final String district;
  final String province;
  final String postalCode;
  final String country;
  
  const Address({
    required this.street,
    required this.district,
    required this.province,
    required this.postalCode,
    this.country = 'Thailand',
  });
  
  @override
  Map<String, String> get value => {
    'street': street,
    'district': district,
    'province': province,
    'postalCode': postalCode,
    'country': country,
  };
  
  String get fullAddress => 
      '$street, $district, $province $postalCode, $country';
  
  @override
  bool operator ==(Object other) {
    return other is Address &&
           other.street == street &&
           other.district == district &&
           other.province == province &&
           other.postalCode == postalCode;
  }
  
  @override
  int get hashCode => Object.hash(street, district, province, postalCode);
}
```

---

## 52.3 Aggregates

### Order Aggregate

```dart
// Order เป็น Aggregate Root
// OrderItem เป็น Entity ภายใน Aggregate

class Order extends Entity<OrderId> {
  final CustomerId customerId;
  final List<OrderItem> _items;
  final Address deliveryAddress;
  OrderStatus _status;
  final DateTime createdAt;
  DateTime? _confirmedAt;
  DateTime? _deliveredAt;
  
  Order({
    required OrderId id,
    required this.customerId,
    required List<OrderItem> items,
    required this.deliveryAddress,
    OrderStatus status = OrderStatus.pending,
    required this.createdAt,
  })  : _items = List.unmodifiable(items),
        _status = status,
        super(id: id);
  
  List<OrderItem> get items => List.unmodifiable(_items);
  OrderStatus get status => _status;
  DateTime? get confirmedAt => _confirmedAt;
  DateTime? get deliveredAt => _deliveredAt;
  
  // Business rule: ไม่สามารถสั่งสินค้าถ้า order ว่าง
  Money get totalAmount {
    if (_items.isEmpty) {
      return Money.zero();
    }
    return _items.fold(
      Money.zero(),
      (sum, item) => sum.add(item.subtotal),
    );
  }
  
  int get totalItems => _items.fold(0, (sum, item) => sum + item.quantity);
  
  // Business operations
  Order addItem(Product product, int quantity) {
    if (_status != OrderStatus.pending) {
      throw const OrderNotModifiableException(
        'ไม่สามารถแก้ไข order ที่ไม่ได้อยู่ในสถานะรอดำเนินการ',
      );
    }
    
    // ตรวจสอบว่ามีสินค้านี้อยู่แล้วไหม
    final existingIndex = _items.indexWhere(
      (item) => item.productId == product.id,
    );
    
    final newItems = List<OrderItem>.from(_items);
    
    if (existingIndex != -1) {
      // เพิ่ม quantity ถ้ามีอยู่แล้ว
      final existing = _items[existingIndex];
      newItems[existingIndex] = existing.copyWith(
        quantity: existing.quantity + quantity,
      );
    } else {
      newItems.add(OrderItem(
        id: OrderItemId.generate(),
        productId: product.id,
        productName: product.name,
        unitPrice: product.price,
        quantity: quantity,
      ));
    }
    
    return Order(
      id: id,
      customerId: customerId,
      items: newItems,
      deliveryAddress: deliveryAddress,
      status: _status,
      createdAt: createdAt,
    );
  }
  
  Order confirm() {
    if (_status != OrderStatus.pending) {
      throw const InvalidOrderStatusException(
        'สามารถยืนยันได้เฉพาะ order ที่รอดำเนินการ',
      );
    }
    if (_items.isEmpty) {
      throw const EmptyOrderException('ไม่สามารถยืนยัน order ที่ว่าง');
    }
    
    final confirmed = Order(
      id: id,
      customerId: customerId,
      items: _items,
      deliveryAddress: deliveryAddress,
      status: OrderStatus.confirmed,
      createdAt: createdAt,
    );
    confirmed._confirmedAt = DateTime.now();
    return confirmed;
  }
  
  Order deliver() {
    if (_status != OrderStatus.confirmed) {
      throw const InvalidOrderStatusException(
        'สามารถส่งได้เฉพาะ order ที่ยืนยันแล้ว',
      );
    }
    
    final delivered = Order(
      id: id,
      customerId: customerId,
      items: _items,
      deliveryAddress: deliveryAddress,
      status: OrderStatus.delivered,
      createdAt: createdAt,
    );
    delivered._confirmedAt = _confirmedAt;
    delivered._deliveredAt = DateTime.now();
    return delivered;
  }
  
  Order cancel(String reason) {
    if (_status == OrderStatus.delivered) {
      throw const InvalidOrderStatusException(
        'ไม่สามารถยกเลิก order ที่ส่งแล้ว',
      );
    }
    
    return Order(
      id: id,
      customerId: customerId,
      items: _items,
      deliveryAddress: deliveryAddress,
      status: OrderStatus.cancelled,
      createdAt: createdAt,
    );
  }
}

// OrderItem (Entity ภายใน Aggregate)
class OrderItem extends Entity<OrderItemId> {
  final ProductId productId;
  final ProductName productName;
  final Money unitPrice;
  final int quantity;
  
  OrderItem({
    required OrderItemId id,
    required this.productId,
    required this.productName,
    required this.unitPrice,
    required this.quantity,
  }) : super(id: id);
  
  Money get subtotal => unitPrice.multiply(quantity.toDouble());
  
  OrderItem copyWith({int? quantity}) {
    return OrderItem(
      id: id,
      productId: productId,
      productName: productName,
      unitPrice: unitPrice,
      quantity: quantity ?? this.quantity,
    );
  }
}

enum OrderStatus {
  pending,    // รอดำเนินการ
  confirmed,  // ยืนยันแล้ว
  shipped,    // กำลังจัดส่ง
  delivered,  // ส่งแล้ว
  cancelled,  // ยกเลิก
}
```

---

## 52.4 Domain Events

```dart
// Domain Events - เหตุการณ์ที่เกิดขึ้นใน domain

abstract class DomainEvent {
  final String eventId;
  final DateTime occurredAt;
  
  DomainEvent({
    String? eventId,
    DateTime? occurredAt,
  })  : eventId = eventId ?? _generateId(),
        occurredAt = occurredAt ?? DateTime.now();
  
  static String _generateId() {
    return DateTime.now().millisecondsSinceEpoch.toString();
  }
}

// Order Events
class OrderPlacedEvent extends DomainEvent {
  final OrderId orderId;
  final CustomerId customerId;
  final Money totalAmount;
  
  OrderPlacedEvent({
    required this.orderId,
    required this.customerId,
    required this.totalAmount,
  });
}

class OrderConfirmedEvent extends DomainEvent {
  final OrderId orderId;
  
  OrderConfirmedEvent({required this.orderId});
}

class OrderDeliveredEvent extends DomainEvent {
  final OrderId orderId;
  final DateTime deliveredAt;
  
  OrderDeliveredEvent({
    required this.orderId,
    required this.deliveredAt,
  });
}

class OrderCancelledEvent extends DomainEvent {
  final OrderId orderId;
  final String reason;
  
  OrderCancelledEvent({
    required this.orderId,
    required this.reason,
  });
}

// Domain Event Bus
class DomainEventBus {
  static final _instance = DomainEventBus._();
  
  DomainEventBus._();
  
  static DomainEventBus get instance => _instance;
  
  final Map<Type, List<Function>> _handlers = {};
  
  void subscribe<T extends DomainEvent>(void Function(T event) handler) {
    _handlers[T] ??= [];
    _handlers[T]!.add(handler);
  }
  
  void unsubscribe<T extends DomainEvent>(void Function(T event) handler) {
    _handlers[T]?.remove(handler);
  }
  
  void publish(DomainEvent event) {
    final handlers = _handlers[event.runtimeType] ?? [];
    for (final handler in handlers) {
      handler(event);
    }
  }
}
```

---

## 52.5 Workshop: E-commerce Domain Model

```dart
// Workshop: สร้าง E-commerce Domain Model แบบครบวงจร

// Product Aggregate
class Product extends Entity<ProductId> {
  final ProductName name;
  final ProductDescription description;
  final Money price;
  final StockQuantity stock;
  final ProductCategory category;
  final List<ProductImage> images;
  final bool isActive;
  
  const Product({
    required ProductId id,
    required this.name,
    required this.description,
    required this.price,
    required this.stock,
    required this.category,
    this.images = const [],
    this.isActive = true,
  }) : super(id: id);
  
  bool get isInStock => stock.value > 0;
  
  Product reduceStock(int quantity) {
    if (!isInStock) {
      throw const OutOfStockException('สินค้าหมด');
    }
    if (quantity > stock.value) {
      throw const InsufficientStockException(
        'สินค้าไม่เพียงพอ',
      );
    }
    
    return Product(
      id: id,
      name: name,
      description: description,
      price: price,
      stock: StockQuantity(stock.value - quantity),
      category: category,
      images: images,
      isActive: isActive,
    );
  }
  
  Product applyDiscount(double discountPercent) {
    if (discountPercent < 0 || discountPercent > 100) {
      throw const InvalidDiscountException('ส่วนลดต้องอยู่ระหว่าง 0-100%');
    }
    
    final discountedPrice = price.multiply(1 - discountPercent / 100);
    
    return Product(
      id: id,
      name: name,
      description: description,
      price: discountedPrice,
      stock: stock,
      category: category,
      images: images,
      isActive: isActive,
    );
  }
}

// Shopping Cart Aggregate
class ShoppingCart extends Entity<CartId> {
  final CustomerId customerId;
  final List<CartItem> _items;
  
  ShoppingCart({
    required CartId id,
    required this.customerId,
    List<CartItem>? items,
  })  : _items = items ?? [],
        super(id: id);
  
  List<CartItem> get items => List.unmodifiable(_items);
  bool get isEmpty => _items.isEmpty;
  int get itemCount => _items.fold(0, (sum, item) => sum + item.quantity);
  
  Money get subtotal => _items.fold(
    Money.zero(),
    (sum, item) => sum.add(item.totalPrice),
  );
  
  ShoppingCart addProduct(Product product, {int quantity = 1}) {
    if (!product.isActive) {
      throw const ProductNotAvailableException('สินค้าไม่พร้อมจำหน่าย');
    }
    if (!product.isInStock) {
      throw const OutOfStockException('สินค้าหมด');
    }
    
    final existingIndex = _items.indexWhere(
      (item) => item.productId == product.id,
    );
    
    final newItems = List<CartItem>.from(_items);
    
    if (existingIndex != -1) {
      newItems[existingIndex] = newItems[existingIndex].copyWith(
        quantity: newItems[existingIndex].quantity + quantity,
      );
    } else {
      newItems.add(CartItem(
        productId: product.id,
        productName: product.name,
        unitPrice: product.price,
        quantity: quantity,
      ));
    }
    
    return ShoppingCart(
      id: id,
      customerId: customerId,
      items: newItems,
    );
  }
  
  ShoppingCart removeProduct(ProductId productId) {
    return ShoppingCart(
      id: id,
      customerId: customerId,
      items: _items.where((i) => i.productId != productId).toList(),
    );
  }
  
  ShoppingCart clear() => ShoppingCart(
    id: id,
    customerId: customerId,
  );
  
  // แปลง Cart เป็น Order
  Order checkout({
    required Address deliveryAddress,
    required OrderId orderId,
  }) {
    if (isEmpty) {
      throw const EmptyCartException('ตะกร้าว่างเปล่า');
    }
    
    final orderItems = _items.map((cartItem) => OrderItem(
      id: OrderItemId.generate(),
      productId: cartItem.productId,
      productName: cartItem.productName,
      unitPrice: cartItem.unitPrice,
      quantity: cartItem.quantity,
    )).toList();
    
    return Order(
      id: orderId,
      customerId: customerId,
      items: orderItems,
      deliveryAddress: deliveryAddress,
      createdAt: DateTime.now(),
    );
  }
}

// Domain Service - Order Pricing
class OrderPricingService {
  Money calculateTotal({
    required Order order,
    List<Discount> discounts = const [],
  }) {
    var total = order.totalAmount;
    
    for (final discount in discounts) {
      if (discount.isApplicable(order)) {
        total = discount.apply(total);
      }
    }
    
    return total;
  }
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- Ubiquitous Language
- Entities และ Value Objects
- Aggregates และ Aggregate Roots
- Domain Events
- Workshop: E-commerce Domain Model

**แบบฝึกหัดเพิ่มเติม:**
1. เพิ่ม Discount และ Coupon domain model
2. Implement Payment aggregate
3. สร้าง Inventory Management domain
4. เพิ่ม Customer loyalty points system
