# Part 85: Real-World Project: E-Commerce App - โปรเจกต์จริง: แอปร้านค้าออนไลน์

## บทนำ

ส่วนนี้เป็นการรวบรวมทุกอย่างที่เรียนมาในการสร้าง E-Commerce App ที่สมบูรณ์แบบ ครอบคลุมตั้งแต่ product catalog ไปจนถึง order tracking

## สถาปัตยกรรมโปรเจกต์

```
ecommerce_app/
├── lib/
│   ├── app/
│   │   ├── app.dart
│   │   ├── router.dart
│   │   └── di.dart
│   ├── core/
│   │   ├── constants/
│   │   ├── errors/
│   │   ├── network/
│   │   └── theme/
│   ├── features/
│   │   ├── auth/
│   │   ├── products/
│   │   ├── cart/
│   │   ├── checkout/
│   │   └── orders/
│   └── shared/
│       └── widgets/
```

## 1. Product Catalog

### Product Model

```dart
// lib/features/products/domain/entities/product.dart
import 'package:equatable/equatable.dart';

class Product extends Equatable {
  final String id;
  final String name;
  final String description;
  final double price;
  final double? originalPrice;
  final List<String> images;
  final String category;
  final int stock;
  final double rating;
  final int reviewCount;
  final List<String> tags;
  final Map<String, List<String>> variants; // e.g., {'size': ['S','M','L']}

  const Product({
    required this.id,
    required this.name,
    required this.description,
    required this.price,
    this.originalPrice,
    required this.images,
    required this.category,
    required this.stock,
    this.rating = 0,
    this.reviewCount = 0,
    this.tags = const [],
    this.variants = const {},
  });

  bool get isAvailable => stock > 0;
  bool get isOnSale => originalPrice != null && originalPrice! > price;
  bool get isLowStock => stock > 0 && stock <= 5;

  double get discountPercentage {
    if (!isOnSale) return 0;
    return ((originalPrice! - price) / originalPrice! * 100).roundToDouble();
  }

  String get primaryImage => images.isNotEmpty ? images.first : '';

  @override
  List<Object?> get props => [id, name, price, stock];
}
```

### Product List Screen

```dart
// lib/features/products/presentation/screens/product_list_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import '../bloc/product_bloc.dart';
import '../widgets/product_card.dart';
import '../widgets/product_filter.dart';
import '../widgets/search_bar.dart';

class ProductListScreen extends StatefulWidget {
  final String? category;

  const ProductListScreen({super.key, this.category});

  @override
  State<ProductListScreen> createState() => _ProductListScreenState();
}

class _ProductListScreenState extends State<ProductListScreen> {
  @override
  void initState() {
    super.initState();
    context.read<ProductBloc>().add(
      LoadProductsEvent(category: widget.category),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.category ?? 'สินค้าทั้งหมด'),
        bottom: const PreferredSize(
          preferredSize: Size.fromHeight(60),
          child: ProductSearchBar(),
        ),
      ),
      body: Column(
        children: [
          const ProductFilterChips(),
          Expanded(
            child: BlocBuilder<ProductBloc, ProductState>(
              builder: (context, state) {
                return state.when(
                  initial: () => const SizedBox(),
                  loading: () => const Center(
                    child: CircularProgressIndicator(),
                  ),
                  loaded: (products, hasMore) => _buildProductGrid(products),
                  error: (message) => Center(
                    child: Column(
                      mainAxisAlignment: MainAxisAlignment.center,
                      children: [
                        const Icon(Icons.error_outline, size: 48),
                        Text(message),
                        ElevatedButton(
                          onPressed: () => context.read<ProductBloc>().add(
                            LoadProductsEvent(category: widget.category),
                          ),
                          child: const Text('ลองใหม่'),
                        ),
                      ],
                    ),
                  ),
                );
              },
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildProductGrid(List<Product> products) {
    return GridView.builder(
      padding: const EdgeInsets.all(16),
      gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
        crossAxisCount: 2,
        crossAxisSpacing: 12,
        mainAxisSpacing: 12,
        childAspectRatio: 0.7,
      ),
      itemCount: products.length,
      itemBuilder: (context, index) => ProductCard(
        product: products[index],
        onTap: () => _navigateToDetail(products[index]),
        onAddToCart: () => _addToCart(products[index]),
      ),
    );
  }

  void _navigateToDetail(Product product) {
    context.push('/product/${product.id}');
  }

  void _addToCart(Product product) {
    context.read<CartBloc>().add(AddToCartEvent(product: product));
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(
        content: Text('เพิ่ม ${product.name} ลงตะกร้าแล้ว'),
        action: SnackBarAction(
          label: 'ดูตะกร้า',
          onPressed: () => context.push('/cart'),
        ),
      ),
    );
  }
}
```

## 2. Shopping Cart

### Cart Model

```dart
// lib/features/cart/domain/entities/cart.dart
import 'package:equatable/equatable.dart';
import '../../products/domain/entities/product.dart';

class CartItem extends Equatable {
  final Product product;
  final int quantity;
  final Map<String, String> selectedVariants;

  const CartItem({
    required this.product,
    required this.quantity,
    this.selectedVariants = const {},
  });

  double get subtotal => product.price * quantity;

  CartItem copyWith({int? quantity, Map<String, String>? selectedVariants}) {
    return CartItem(
      product: product,
      quantity: quantity ?? this.quantity,
      selectedVariants: selectedVariants ?? this.selectedVariants,
    );
  }

  @override
  List<Object?> get props => [product.id, quantity, selectedVariants];
}

class Cart extends Equatable {
  final List<CartItem> items;
  final String? couponCode;
  final double? discount;

  const Cart({
    this.items = const [],
    this.couponCode,
    this.discount,
  });

  int get itemCount => items.fold(0, (sum, item) => sum + item.quantity);

  double get subtotal =>
      items.fold(0, (sum, item) => sum + item.subtotal);

  double get discountAmount => discount ?? 0;

  double get shippingFee {
    if (subtotal >= 1000) return 0; // ส่งฟรีเมื่อซื้อครบ 1000
    return 50;
  }

  double get total => subtotal - discountAmount + shippingFee;

  bool get isEmpty => items.isEmpty;

  Cart addItem(CartItem newItem) {
    final existingIndex = items.indexWhere(
      (item) => item.product.id == newItem.product.id,
    );

    if (existingIndex >= 0) {
      final updatedItems = List<CartItem>.from(items);
      updatedItems[existingIndex] = items[existingIndex].copyWith(
        quantity: items[existingIndex].quantity + newItem.quantity,
      );
      return Cart(items: updatedItems, couponCode: couponCode, discount: discount);
    }

    return Cart(
      items: [...items, newItem],
      couponCode: couponCode,
      discount: discount,
    );
  }

  Cart removeItem(String productId) {
    return Cart(
      items: items.where((item) => item.product.id != productId).toList(),
      couponCode: couponCode,
      discount: discount,
    );
  }

  Cart updateQuantity(String productId, int quantity) {
    if (quantity <= 0) return removeItem(productId);

    return Cart(
      items: items.map((item) {
        if (item.product.id == productId) {
          return item.copyWith(quantity: quantity);
        }
        return item;
      }).toList(),
      couponCode: couponCode,
      discount: discount,
    );
  }

  @override
  List<Object?> get props => [items, couponCode, discount];
}
```

### Cart Screen

```dart
// lib/features/cart/presentation/screens/cart_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

class CartScreen extends StatelessWidget {
  const CartScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('ตะกร้าสินค้า'),
        actions: [
          BlocBuilder<CartBloc, CartState>(
            builder: (context, state) {
              if (state is CartLoaded && !state.cart.isEmpty) {
                return TextButton(
                  onPressed: () => _confirmClearCart(context),
                  child: const Text('ล้างตะกร้า'),
                );
              }
              return const SizedBox();
            },
          ),
        ],
      ),
      body: BlocBuilder<CartBloc, CartState>(
        builder: (context, state) {
          if (state is CartLoaded) {
            if (state.cart.isEmpty) {
              return _buildEmptyCart(context);
            }
            return _buildCartContent(context, state.cart);
          }
          return const Center(child: CircularProgressIndicator());
        },
      ),
    );
  }

  Widget _buildEmptyCart(BuildContext context) {
    return Center(
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          const Icon(Icons.shopping_cart_outlined, size: 80, color: Colors.grey),
          const SizedBox(height: 16),
          const Text(
            'ตะกร้าของคุณว่างเปล่า',
            style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold),
          ),
          const SizedBox(height: 8),
          const Text(
            'เพิ่มสินค้าที่คุณชอบลงตะกร้า',
            style: TextStyle(color: Colors.grey),
          ),
          const SizedBox(height: 24),
          FilledButton(
            onPressed: () => context.go('/'),
            child: const Text('เลือกซื้อสินค้า'),
          ),
        ],
      ),
    );
  }

  Widget _buildCartContent(BuildContext context, Cart cart) {
    return Column(
      children: [
        Expanded(
          child: ListView.builder(
            padding: const EdgeInsets.all(16),
            itemCount: cart.items.length,
            itemBuilder: (context, index) => CartItemCard(
              item: cart.items[index],
              onRemove: () => context.read<CartBloc>().add(
                RemoveFromCartEvent(productId: cart.items[index].product.id),
              ),
              onQuantityChanged: (qty) => context.read<CartBloc>().add(
                UpdateQuantityEvent(
                  productId: cart.items[index].product.id,
                  quantity: qty,
                ),
              ),
            ),
          ),
        ),
        CartSummaryPanel(cart: cart),
      ],
    );
  }

  Future<void> _confirmClearCart(BuildContext context) async {
    final confirmed = await showDialog<bool>(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('ล้างตะกร้า'),
        content: const Text('ต้องการลบสินค้าทั้งหมดออกจากตะกร้า?'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context, false),
            child: const Text('ยกเลิก'),
          ),
          FilledButton(
            onPressed: () => Navigator.pop(context, true),
            style: FilledButton.styleFrom(
              backgroundColor: Colors.red,
            ),
            child: const Text('ล้างตะกร้า'),
          ),
        ],
      ),
    );

    if (confirmed == true && context.mounted) {
      context.read<CartBloc>().add(const ClearCartEvent());
    }
  }
}

class CartSummaryPanel extends StatelessWidget {
  final Cart cart;

  const CartSummaryPanel({super.key, required this.cart});

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(16),
      decoration: BoxDecoration(
        color: Theme.of(context).colorScheme.surface,
        boxShadow: [
          BoxShadow(
            color: Colors.black.withOpacity(0.1),
            blurRadius: 8,
            offset: const Offset(0, -2),
          ),
        ],
      ),
      child: Column(
        children: [
          _buildSummaryRow('ยอดรวม', '฿${cart.subtotal.toStringAsFixed(2)}'),
          if (cart.discountAmount > 0)
            _buildSummaryRow(
              'ส่วนลด',
              '-฿${cart.discountAmount.toStringAsFixed(2)}',
              color: Colors.green,
            ),
          _buildSummaryRow(
            'ค่าจัดส่ง',
            cart.shippingFee == 0
                ? 'ฟรี'
                : '฿${cart.shippingFee.toStringAsFixed(2)}',
            color: cart.shippingFee == 0 ? Colors.green : null,
          ),
          const Divider(),
          _buildSummaryRow(
            'ยอดสุทธิ',
            '฿${cart.total.toStringAsFixed(2)}',
            isBold: true,
            fontSize: 18,
          ),
          const SizedBox(height: 12),
          SizedBox(
            width: double.infinity,
            child: FilledButton(
              onPressed: () => context.push('/checkout'),
              child: const Text('ดำเนินการชำระเงิน'),
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildSummaryRow(
    String label,
    String value, {
    Color? color,
    bool isBold = false,
    double fontSize = 14,
  }) {
    final style = TextStyle(
      color: color,
      fontWeight: isBold ? FontWeight.bold : FontWeight.normal,
      fontSize: fontSize,
    );

    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceBetween,
        children: [
          Text(label, style: style),
          Text(value, style: style),
        ],
      ),
    );
  }
}
```

## 3. Checkout Flow

### Checkout Screen

```dart
// lib/features/checkout/presentation/screens/checkout_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

class CheckoutScreen extends StatefulWidget {
  const CheckoutScreen({super.key});

  @override
  State<CheckoutScreen> createState() => _CheckoutScreenState();
}

class _CheckoutScreenState extends State<CheckoutScreen> {
  int _currentStep = 0;

  final List<Step> _steps = [
    const Step(
      title: Text('ที่อยู่จัดส่ง'),
      content: AddressForm(),
    ),
    const Step(
      title: Text('วิธีชำระเงิน'),
      content: PaymentForm(),
    ),
    const Step(
      title: Text('สรุปคำสั่งซื้อ'),
      content: OrderSummary(),
    ),
  ];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('ชำระเงิน')),
      body: Stepper(
        currentStep: _currentStep,
        onStepContinue: _handleStepContinue,
        onStepCancel: _handleStepCancel,
        steps: _steps,
        controlsBuilder: (context, details) => Row(
          children: [
            FilledButton(
              onPressed: details.onStepContinue,
              child: Text(
                _currentStep == _steps.length - 1 ? 'สั่งซื้อ' : 'ถัดไป',
              ),
            ),
            if (_currentStep > 0) ...[
              const SizedBox(width: 8),
              OutlinedButton(
                onPressed: details.onStepCancel,
                child: const Text('ย้อนกลับ'),
              ),
            ],
          ],
        ),
      ),
    );
  }

  void _handleStepContinue() {
    if (_currentStep < _steps.length - 1) {
      setState(() => _currentStep++);
    } else {
      _placeOrder();
    }
  }

  void _handleStepCancel() {
    if (_currentStep > 0) {
      setState(() => _currentStep--);
    }
  }

  Future<void> _placeOrder() async {
    context.read<OrderBloc>().add(const PlaceOrderEvent());

    // รอผล
    final result = await context.read<OrderBloc>().stream
        .firstWhere((state) => state is OrderPlaced || state is OrderError);

    if (!mounted) return;

    if (result is OrderPlaced) {
      context.go('/order-confirmation/${result.orderId}');
    } else if (result is OrderError) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(
          content: Text(result.message),
          backgroundColor: Colors.red,
        ),
      );
    }
  }
}
```

## 4. Order Tracking

### Order Model

```dart
// lib/features/orders/domain/entities/order.dart
import 'package:equatable/equatable.dart';

enum OrderStatus {
  pending,
  confirmed,
  processing,
  shipped,
  outForDelivery,
  delivered,
  cancelled,
  refunded,
}

class Order extends Equatable {
  final String id;
  final List<OrderItem> items;
  final double subtotal;
  final double discount;
  final double shippingFee;
  final double total;
  final Address shippingAddress;
  final PaymentMethod paymentMethod;
  final OrderStatus status;
  final DateTime createdAt;
  final DateTime? estimatedDelivery;
  final String? trackingNumber;
  final List<OrderStatusUpdate> statusHistory;

  const Order({
    required this.id,
    required this.items,
    required this.subtotal,
    required this.discount,
    required this.shippingFee,
    required this.total,
    required this.shippingAddress,
    required this.paymentMethod,
    required this.status,
    required this.createdAt,
    this.estimatedDelivery,
    this.trackingNumber,
    this.statusHistory = const [],
  });

  String get statusText {
    switch (status) {
      case OrderStatus.pending:
        return 'รอยืนยัน';
      case OrderStatus.confirmed:
        return 'ยืนยันแล้ว';
      case OrderStatus.processing:
        return 'กำลังเตรียมสินค้า';
      case OrderStatus.shipped:
        return 'จัดส่งแล้ว';
      case OrderStatus.outForDelivery:
        return 'กำลังนำส่ง';
      case OrderStatus.delivered:
        return 'ส่งถึงแล้ว';
      case OrderStatus.cancelled:
        return 'ยกเลิกแล้ว';
      case OrderStatus.refunded:
        return 'คืนเงินแล้ว';
    }
  }

  @override
  List<Object?> get props => [id, status];
}
```

### Order Tracking Screen

```dart
// lib/features/orders/presentation/screens/order_tracking_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:timelines/timelines.dart';

class OrderTrackingScreen extends StatelessWidget {
  final String orderId;

  const OrderTrackingScreen({super.key, required this.orderId});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('คำสั่งซื้อ #$orderId'),
      ),
      body: BlocBuilder<OrderBloc, OrderState>(
        builder: (context, state) {
          if (state is OrderLoaded) {
            return _buildOrderTracking(context, state.order);
          }
          return const Center(child: CircularProgressIndicator());
        },
      ),
    );
  }

  Widget _buildOrderTracking(BuildContext context, Order order) {
    return SingleChildScrollView(
      padding: const EdgeInsets.all(16),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          // Status Card
          _buildStatusCard(context, order),
          const SizedBox(height: 24),

          // Timeline
          Text(
            'ติดตามสถานะ',
            style: Theme.of(context).textTheme.titleMedium,
          ),
          const SizedBox(height: 12),
          _buildStatusTimeline(order),
          const SizedBox(height: 24),

          // Items
          Text(
            'รายการสินค้า',
            style: Theme.of(context).textTheme.titleMedium,
          ),
          ...order.items.map(_buildOrderItem),
          const SizedBox(height: 24),

          // Summary
          _buildOrderSummary(context, order),
        ],
      ),
    );
  }

  Widget _buildStatusCard(BuildContext context, Order order) {
    final color = _statusColor(order.status);

    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Row(
          children: [
            Container(
              padding: const EdgeInsets.all(12),
              decoration: BoxDecoration(
                color: color.withOpacity(0.1),
                shape: BoxShape.circle,
              ),
              child: Icon(
                _statusIcon(order.status),
                color: color,
                size: 32,
              ),
            ),
            const SizedBox(width: 16),
            Expanded(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(
                    order.statusText,
                    style: TextStyle(
                      fontSize: 18,
                      fontWeight: FontWeight.bold,
                      color: color,
                    ),
                  ),
                  if (order.estimatedDelivery != null)
                    Text(
                      'คาดว่าจะถึงวันที่ ${_formatDate(order.estimatedDelivery!)}',
                      style: const TextStyle(color: Colors.grey),
                    ),
                  if (order.trackingNumber != null)
                    Text(
                      'หมายเลขพัสดุ: ${order.trackingNumber}',
                      style: const TextStyle(color: Colors.grey),
                    ),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }

  Widget _buildStatusTimeline(Order order) {
    final statuses = [
      OrderStatus.confirmed,
      OrderStatus.processing,
      OrderStatus.shipped,
      OrderStatus.outForDelivery,
      OrderStatus.delivered,
    ];

    return FixedTimeline.tileBuilder(
      builder: TimelineTileBuilder.fromStyle(
        contentsAlign: ContentsAlign.basic,
        contentsBuilder: (context, index) => Padding(
          padding: const EdgeInsets.all(8),
          child: Text(_statusText(statuses[index])),
        ),
        indicatorStyleBuilder: (context, index) {
          final statusIndex = statuses.indexOf(order.status);
          if (index < statusIndex) return IndicatorStyle.dot;
          if (index == statusIndex) return IndicatorStyle.outlined;
          return IndicatorStyle.outlined;
        },
        itemCount: statuses.length,
      ),
    );
  }

  Widget _buildOrderItem(OrderItem item) {
    return ListTile(
      leading: ClipRRect(
        borderRadius: BorderRadius.circular(8),
        child: Image.network(
          item.productImage,
          width: 56,
          height: 56,
          fit: BoxFit.cover,
        ),
      ),
      title: Text(item.productName),
      subtitle: Text('x${item.quantity}'),
      trailing: Text('฿${item.subtotal.toStringAsFixed(2)}'),
    );
  }

  Widget _buildOrderSummary(BuildContext context, Order order) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            _summaryRow('ยอดรวม', '฿${order.subtotal.toStringAsFixed(2)}'),
            if (order.discount > 0)
              _summaryRow('ส่วนลด', '-฿${order.discount.toStringAsFixed(2)}',
                  color: Colors.green),
            _summaryRow('ค่าจัดส่ง',
                order.shippingFee == 0
                    ? 'ฟรี'
                    : '฿${order.shippingFee.toStringAsFixed(2)}',
                color: order.shippingFee == 0 ? Colors.green : null),
            const Divider(),
            _summaryRow(
              'ยอดสุทธิ',
              '฿${order.total.toStringAsFixed(2)}',
              isBold: true,
            ),
          ],
        ),
      ),
    );
  }

  Widget _summaryRow(String label, String value,
      {Color? color, bool isBold = false}) {
    final style = TextStyle(
      color: color,
      fontWeight: isBold ? FontWeight.bold : FontWeight.normal,
    );
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceBetween,
        children: [
          Text(label, style: style),
          Text(value, style: style),
        ],
      ),
    );
  }

  Color _statusColor(OrderStatus status) {
    switch (status) {
      case OrderStatus.pending:
        return Colors.orange;
      case OrderStatus.confirmed:
      case OrderStatus.processing:
        return Colors.blue;
      case OrderStatus.shipped:
      case OrderStatus.outForDelivery:
        return Colors.purple;
      case OrderStatus.delivered:
        return Colors.green;
      case OrderStatus.cancelled:
      case OrderStatus.refunded:
        return Colors.red;
    }
  }

  IconData _statusIcon(OrderStatus status) {
    switch (status) {
      case OrderStatus.pending:
        return Icons.pending;
      case OrderStatus.confirmed:
        return Icons.check_circle;
      case OrderStatus.processing:
        return Icons.inventory;
      case OrderStatus.shipped:
        return Icons.local_shipping;
      case OrderStatus.outForDelivery:
        return Icons.delivery_dining;
      case OrderStatus.delivered:
        return Icons.done_all;
      case OrderStatus.cancelled:
        return Icons.cancel;
      case OrderStatus.refunded:
        return Icons.currency_exchange;
    }
  }

  String _statusText(OrderStatus status) {
    switch (status) {
      case OrderStatus.confirmed:
        return 'ยืนยันคำสั่งซื้อ';
      case OrderStatus.processing:
        return 'เตรียมสินค้า';
      case OrderStatus.shipped:
        return 'จัดส่งสินค้า';
      case OrderStatus.outForDelivery:
        return 'กำลังนำส่ง';
      case OrderStatus.delivered:
        return 'ส่งถึงมือแล้ว';
      default:
        return status.name;
    }
  }

  String _formatDate(DateTime date) {
    return '${date.day}/${date.month}/${date.year}';
  }
}
```

## สรุป

E-Commerce App ครอบคลุม:
1. **Product Catalog** - Grid/List view, filtering, search
2. **Shopping Cart** - Add/remove/update, price calculation
3. **Checkout Flow** - Multi-step, address, payment
4. **Order Tracking** - Real-time status, timeline

## แบบทดสอบ

1. เพิ่ม product review/rating system ใน app
2. สร้าง wishlist feature ที่ sync กับ backend
3. Implement infinite scroll ใน product list
4. เพิ่ม push notification สำหรับ order status updates
