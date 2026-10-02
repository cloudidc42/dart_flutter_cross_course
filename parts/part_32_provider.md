# Part 32: Provider State Management

## Provider คืออะไร

Provider เป็น state management solution ที่ได้รับการแนะนำจาก Flutter team เป็น wrapper รอบ InheritedWidget ที่ใช้งานง่ายกว่า

```yaml
# pubspec.yaml
dependencies:
  flutter:
    sdk: flutter
  provider: ^6.1.0
```

---

## ChangeNotifier และ ChangeNotifierProvider

### ChangeNotifier คืออะไร

`ChangeNotifier` เป็น class ที่ให้ notification เมื่อ state เปลี่ยน widget ที่ subscribe จะถูก rebuild อัตโนมัติ

```dart
// สร้าง Counter ChangeNotifier
import 'package:flutter/foundation.dart';

class CounterModel extends ChangeNotifier {
  int _count = 0;
  
  int get count => _count;
  
  void increment() {
    _count++;
    notifyListeners();  // แจ้ง widgets ที่ subscribe ว่ามีการเปลี่ยนแปลง
  }
  
  void decrement() {
    if (_count > 0) {
      _count--;
      notifyListeners();
    }
  }
  
  void reset() {
    _count = 0;
    notifyListeners();
  }
}
```

### ChangeNotifierProvider

```dart
// main.dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

void main() {
  runApp(
    // Provide CounterModel ให้กับ widget tree ทั้งหมด
    ChangeNotifierProvider(
      create: (context) => CounterModel(),
      child: const MyApp(),
    ),
  );
}

class MyApp extends StatelessWidget {
  const MyApp({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Provider Demo',
      theme: ThemeData(primarySwatch: Colors.blue, useMaterial3: true),
      home: const CounterPage(),
    );
  }
}

class CounterPage extends StatelessWidget {
  const CounterPage({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Provider Counter')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            const Text('You have pressed the button this many times:'),
            // อ่านค่าจาก Provider
            Text(
              '${context.watch<CounterModel>().count}',
              style: Theme.of(context).textTheme.headlineMedium,
            ),
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                ElevatedButton(
                  onPressed: () => context.read<CounterModel>().decrement(),
                  child: const Icon(Icons.remove),
                ),
                const SizedBox(width: 16),
                ElevatedButton(
                  onPressed: () => context.read<CounterModel>().increment(),
                  child: const Icon(Icons.add),
                ),
              ],
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## Consumer และ Provider.of

### context.watch vs context.read vs context.select

```dart
class CounterDisplay extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // context.watch: rebuild เมื่อ CounterModel เปลี่ยน
    // ใช้ใน build method เพื่อ read และ rebuild
    final counter = context.watch<CounterModel>();
    return Text('Count: ${counter.count}');
  }
}

class IncrementButton extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // context.read: ไม่ subscribe ไม่ rebuild
    // ใช้ใน callbacks เช่น onPressed
    return ElevatedButton(
      onPressed: () => context.read<CounterModel>().increment(),
      child: const Text('Increment'),
    );
  }
}

class SelectiveDisplay extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // context.select: rebuild เฉพาะเมื่อ selected value เปลี่ยน
    // มีประสิทธิภาพสูงกว่า watch
    final count = context.select<CounterModel, int>((model) => model.count);
    return Text('Count: $count');
  }
}
```

### Consumer Widget

```dart
class CounterPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Consumer Demo')),
      body: Column(
        children: [
          // Consumer rebuild เฉพาะตัวเอง ไม่ใช่ทั้ง Column
          Consumer<CounterModel>(
            builder: (context, counter, child) {
              // child คือส่วนที่ไม่เปลี่ยนแปลง ไม่ต้อง rebuild
              return Column(
                children: [
                  Text('Count: ${counter.count}'),
                  Text('Doubled: ${counter.count * 2}'),
                  // child จาก parameter ด้านล่าง
                  child!,
                ],
              );
            },
            // ส่วนนี้ไม่ rebuild เมื่อ counter เปลี่ยน
            child: const Text('This never rebuilds'),
          ),
          // ปุ่มนี้ไม่ต้อง rebuild เมื่อ count เปลี่ยน
          ElevatedButton(
            onPressed: () => context.read<CounterModel>().increment(),
            child: const Text('Increment'),
          ),
        ],
      ),
    );
  }
}
```

### Provider.of

```dart
class OldStyleWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // Provider.of กับ listen: true = เหมือน context.watch
    final counter = Provider.of<CounterModel>(context);
    
    // Provider.of กับ listen: false = เหมือน context.read
    final counterNoListen = Provider.of<CounterModel>(context, listen: false);
    
    return Column(
      children: [
        Text('Count: ${counter.count}'),
        ElevatedButton(
          onPressed: () => counterNoListen.increment(),
          child: const Text('Increment'),
        ),
      ],
    );
  }
}
```

---

## MultiProvider

### การใช้งาน MultiProvider

```dart
// สร้าง models หลายตัว
class UserModel extends ChangeNotifier {
  String _name = '';
  String _email = '';
  bool _isLoggedIn = false;
  
  String get name => _name;
  String get email => _email;
  bool get isLoggedIn => _isLoggedIn;
  
  Future<void> login(String email, String password) async {
    // จำลองการ login
    await Future.delayed(const Duration(seconds: 1));
    _name = 'John Doe';
    _email = email;
    _isLoggedIn = true;
    notifyListeners();
  }
  
  void logout() {
    _name = '';
    _email = '';
    _isLoggedIn = false;
    notifyListeners();
  }
}

class ThemeModel extends ChangeNotifier {
  bool _isDarkMode = false;
  Color _primaryColor = Colors.blue;
  
  bool get isDarkMode => _isDarkMode;
  Color get primaryColor => _primaryColor;
  
  void toggleDarkMode() {
    _isDarkMode = !_isDarkMode;
    notifyListeners();
  }
  
  void setPrimaryColor(Color color) {
    _primaryColor = color;
    notifyListeners();
  }
}

class CartModel extends ChangeNotifier {
  final List<CartItem> _items = [];
  
  List<CartItem> get items => List.unmodifiable(_items);
  int get itemCount => _items.fold(0, (sum, item) => sum + item.quantity);
  double get total => _items.fold(
    0.0,
    (sum, item) => sum + (item.price * item.quantity),
  );
  
  void addItem(String name, double price) {
    final index = _items.indexWhere((item) => item.name == name);
    if (index >= 0) {
      _items[index] = CartItem(
        name: name,
        price: price,
        quantity: _items[index].quantity + 1,
      );
    } else {
      _items.add(CartItem(name: name, price: price, quantity: 1));
    }
    notifyListeners();
  }
  
  void removeItem(String name) {
    _items.removeWhere((item) => item.name == name);
    notifyListeners();
  }
  
  void clear() {
    _items.clear();
    notifyListeners();
  }
}

class CartItem {
  final String name;
  final double price;
  final int quantity;
  
  CartItem({required this.name, required this.price, required this.quantity});
}

// ใช้ MultiProvider ใน main.dart
void main() {
  runApp(
    MultiProvider(
      providers: [
        ChangeNotifierProvider(create: (_) => UserModel()),
        ChangeNotifierProvider(create: (_) => ThemeModel()),
        ChangeNotifierProvider(create: (_) => CartModel()),
      ],
      child: const MyApp(),
    ),
  );
}

class MyApp extends StatelessWidget {
  const MyApp({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    // ใช้ ThemeModel สำหรับ theme
    final themeModel = context.watch<ThemeModel>();
    
    return MaterialApp(
      title: 'MultiProvider Demo',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(
          seedColor: themeModel.primaryColor,
          brightness: themeModel.isDarkMode ? Brightness.dark : Brightness.light,
        ),
        useMaterial3: true,
      ),
      home: const HomePage(),
    );
  }
}
```

### การ Access หลาย Providers

```dart
class HomePage extends StatelessWidget {
  const HomePage({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    final user = context.watch<UserModel>();
    final cart = context.watch<CartModel>();
    final theme = context.watch<ThemeModel>();
    
    return Scaffold(
      appBar: AppBar(
        title: Text(user.isLoggedIn ? 'Hello, ${user.name}' : 'Please Login'),
        actions: [
          IconButton(
            onPressed: () => theme.toggleDarkMode(),
            icon: Icon(theme.isDarkMode ? Icons.light_mode : Icons.dark_mode),
          ),
          Badge(
            label: Text('${cart.itemCount}'),
            isLabelVisible: cart.itemCount > 0,
            child: IconButton(
              onPressed: () {},
              icon: const Icon(Icons.shopping_cart),
            ),
          ),
        ],
      ),
      body: user.isLoggedIn 
          ? _buildContent(context, cart)
          : _buildLoginPrompt(context),
    );
  }
  
  Widget _buildContent(BuildContext context, CartModel cart) {
    return Column(
      children: [
        Text('Total: ฿${cart.total.toStringAsFixed(2)}'),
        ElevatedButton(
          onPressed: () => cart.addItem('Test Product', 99.0),
          child: const Text('Add to Cart'),
        ),
        ElevatedButton(
          onPressed: () => context.read<UserModel>().logout(),
          child: const Text('Logout'),
        ),
      ],
    );
  }
  
  Widget _buildLoginPrompt(BuildContext context) {
    return Center(
      child: ElevatedButton(
        onPressed: () => context.read<UserModel>().login('test@test.com', 'password'),
        child: const Text('Login'),
      ),
    );
  }
}
```

---

## ProxyProvider

### ใช้เมื่อ Provider ขึ้นกัน Provider อื่น

```dart
// AuthService ที่ CartService ต้องการ
class AuthService extends ChangeNotifier {
  String? _userId;
  String? get userId => _userId;
  bool get isAuthenticated => _userId != null;
  
  void login(String userId) {
    _userId = userId;
    notifyListeners();
  }
  
  void logout() {
    _userId = null;
    notifyListeners();
  }
}

// CartService ที่ต้องการ AuthService
class CartService extends ChangeNotifier {
  final AuthService _authService;
  final Map<String, int> _items = {};
  
  CartService(this._authService);
  
  Map<String, int> get items => Map.unmodifiable(_items);
  
  void addItem(String productId) {
    if (!_authService.isAuthenticated) {
      throw Exception('Must be logged in to add to cart');
    }
    _items[productId] = (_items[productId] ?? 0) + 1;
    notifyListeners();
  }
  
  void removeItem(String productId) {
    _items.remove(productId);
    notifyListeners();
  }
  
  Future<void> syncWithServer() async {
    if (!_authService.isAuthenticated) return;
    final userId = _authService.userId;
    print('Syncing cart for user: $userId');
    // ส่ง cart ไป server
    await Future.delayed(const Duration(seconds: 1));
    print('Cart synced!');
  }
}

// ใช้ ProxyProvider
void main() {
  runApp(
    MultiProvider(
      providers: [
        ChangeNotifierProvider(create: (_) => AuthService()),
        // ProxyProvider สร้าง CartService ที่ขึ้นกับ AuthService
        ChangeNotifierProxyProvider<AuthService, CartService>(
          create: (context) => CartService(context.read<AuthService>()),
          update: (context, auth, previousCart) {
            // เมื่อ AuthService เปลี่ยน CartService จะได้รับการ update
            return previousCart ?? CartService(auth);
          },
        ),
      ],
      child: const MyApp(),
    ),
  );
}
```

---

## Best Practices

### 1. แยก ChangeNotifier ตาม Domain

```dart
// ไม่ดี: ทุกอย่างอยู่ใน AppModel เดียว
class AppModel extends ChangeNotifier {
  // User state
  String _userName = '';
  bool _isLoggedIn = false;
  
  // Cart state
  List<CartItem> _cartItems = [];
  
  // Theme state
  bool _isDark = false;
  
  // ... method อีกมากมาย
}

// ดีกว่า: แยกตาม feature
class UserModel extends ChangeNotifier { /* ... */ }
class CartModel extends ChangeNotifier { /* ... */ }
class ThemeModel extends ChangeNotifier { /* ... */ }
```

### 2. ใช้ context.select แทน context.watch

```dart
// ไม่ดี: rebuild ทุกครั้งที่ UserModel เปลี่ยน
class UserNameDisplay extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final user = context.watch<UserModel>();
    return Text(user.name);  // rebuild แม้แต่เมื่อ email เปลี่ยน
  }
}

// ดีกว่า: rebuild เฉพาะเมื่อ name เปลี่ยน
class OptimizedUserNameDisplay extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final name = context.select<UserModel, String>((user) => user.name);
    return Text(name);
  }
}
```

### 3. ไม่ควร call context.read ใน build method

```dart
// ไม่ดี
class BadWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final count = context.read<CounterModel>().count;  // ไม่ subscribe!
    return Text('$count');  // จะไม่ update
  }
}

// ดี
class GoodWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final count = context.watch<CounterModel>().count;  // subscribe ถูกต้อง
    return Text('$count');
  }
}
```

### 4. อย่า listen ใน initState หรือ dispose

```dart
class MyWidget extends StatefulWidget {
  @override
  State<MyWidget> createState() => _MyWidgetState();
}

class _MyWidgetState extends State<MyWidget> {
  @override
  void initState() {
    super.initState();
    // ถูกต้อง: ใช้ listen: false ใน initState
    final counter = context.read<CounterModel>();
    counter.addListener(_onCounterChanged);
  }
  
  void _onCounterChanged() {
    // handle change
  }
  
  @override
  void dispose() {
    context.read<CounterModel>().removeListener(_onCounterChanged);
    super.dispose();
  }
  
  @override
  Widget build(BuildContext context) {
    return Text('${context.watch<CounterModel>().count}');
  }
}
```

---

## Workshop: Shopping Cart with Provider

### โครงสร้าง

```
shopping_provider/
├── main.dart
├── models/
│   ├── product.dart
│   └── cart_model.dart
├── screens/
│   ├── product_list_screen.dart
│   └── cart_screen.dart
└── widgets/
    └── product_card.dart
```

### Cart Model

```dart
// models/cart_model.dart
import 'package:flutter/foundation.dart';

class Product {
  final String id;
  final String name;
  final double price;
  final String category;
  
  const Product({
    required this.id,
    required this.name,
    required this.price,
    required this.category,
  });
}

class CartItem {
  final Product product;
  int quantity;
  
  CartItem({required this.product, this.quantity = 1});
  
  double get subtotal => product.price * quantity;
}

class CartModel extends ChangeNotifier {
  final List<CartItem> _items = [];
  
  List<CartItem> get items => List.unmodifiable(_items);
  
  int get itemCount => _items.fold(0, (sum, item) => sum + item.quantity);
  
  double get subtotal => _items.fold(0.0, (sum, item) => sum + item.subtotal);
  
  double get shipping => subtotal > 500 ? 0 : 50;
  
  double get total => subtotal + shipping;
  
  bool isInCart(String productId) {
    return _items.any((item) => item.product.id == productId);
  }
  
  int quantityOf(String productId) {
    final item = _items.where((item) => item.product.id == productId);
    return item.isEmpty ? 0 : item.first.quantity;
  }
  
  void addItem(Product product) {
    final index = _items.indexWhere((item) => item.product.id == product.id);
    if (index >= 0) {
      _items[index].quantity++;
    } else {
      _items.add(CartItem(product: product));
    }
    notifyListeners();
  }
  
  void removeItem(String productId) {
    _items.removeWhere((item) => item.product.id == productId);
    notifyListeners();
  }
  
  void updateQuantity(String productId, int quantity) {
    final index = _items.indexWhere((item) => item.product.id == productId);
    if (index >= 0) {
      if (quantity <= 0) {
        _items.removeAt(index);
      } else {
        _items[index].quantity = quantity;
      }
      notifyListeners();
    }
  }
  
  void clear() {
    _items.clear();
    notifyListeners();
  }
}
```

### Product Model และ Data

```dart
// models/product.dart
class ProductRepository {
  static final List<Product> products = [
    const Product(
      id: '1',
      name: 'iPhone 15 Pro',
      price: 42900,
      category: 'Phones',
    ),
    const Product(
      id: '2',
      name: 'Samsung Galaxy S24 Ultra',
      price: 39900,
      category: 'Phones',
    ),
    const Product(
      id: '3',
      name: 'MacBook Pro M3',
      price: 69900,
      category: 'Computers',
    ),
    const Product(
      id: '4',
      name: 'iPad Air',
      price: 22900,
      category: 'Tablets',
    ),
    const Product(
      id: '5',
      name: 'AirPods Pro',
      price: 8990,
      category: 'Audio',
    ),
    const Product(
      id: '6',
      name: 'Apple Watch Ultra',
      price: 31900,
      category: 'Wearables',
    ),
  ];
}
```

### Main App

```dart
// main.dart
void main() {
  runApp(
    ChangeNotifierProvider(
      create: (_) => CartModel(),
      child: const ShoppingApp(),
    ),
  );
}

class ShoppingApp extends StatelessWidget {
  const ShoppingApp({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Shopping Cart',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
        useMaterial3: true,
      ),
      home: const MainScreen(),
    );
  }
}

class MainScreen extends StatefulWidget {
  const MainScreen({Key? key}) : super(key: key);
  
  @override
  State<MainScreen> createState() => _MainScreenState();
}

class _MainScreenState extends State<MainScreen> {
  int _currentIndex = 0;
  
  @override
  Widget build(BuildContext context) {
    final cartCount = context.select<CartModel, int>((cart) => cart.itemCount);
    
    return Scaffold(
      body: IndexedStack(
        index: _currentIndex,
        children: const [
          ProductListScreen(),
          CartScreen(),
        ],
      ),
      bottomNavigationBar: NavigationBar(
        selectedIndex: _currentIndex,
        onDestinationSelected: (index) => setState(() => _currentIndex = index),
        destinations: [
          const NavigationDestination(
            icon: Icon(Icons.store_outlined),
            selectedIcon: Icon(Icons.store),
            label: 'Products',
          ),
          NavigationDestination(
            icon: Badge(
              isLabelVisible: cartCount > 0,
              label: Text('$cartCount'),
              child: const Icon(Icons.shopping_cart_outlined),
            ),
            selectedIcon: Badge(
              isLabelVisible: cartCount > 0,
              label: Text('$cartCount'),
              child: const Icon(Icons.shopping_cart),
            ),
            label: 'Cart',
          ),
        ],
      ),
    );
  }
}
```

### Product List Screen

```dart
// screens/product_list_screen.dart
class ProductListScreen extends StatefulWidget {
  const ProductListScreen({Key? key}) : super(key: key);
  
  @override
  State<ProductListScreen> createState() => _ProductListScreenState();
}

class _ProductListScreenState extends State<ProductListScreen> {
  String _selectedCategory = 'All';
  
  List<String> get categories {
    final cats = ProductRepository.products.map((p) => p.category).toSet().toList();
    cats.sort();
    return ['All', ...cats];
  }
  
  List<Product> get filteredProducts {
    if (_selectedCategory == 'All') {
      return ProductRepository.products;
    }
    return ProductRepository.products
        .where((p) => p.category == _selectedCategory)
        .toList();
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Products'),
        bottom: PreferredSize(
          preferredSize: const Size.fromHeight(60),
          child: SingleChildScrollView(
            scrollDirection: Axis.horizontal,
            padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
            child: Row(
              children: categories.map((cat) {
                return Padding(
                  padding: const EdgeInsets.only(right: 8),
                  child: FilterChip(
                    label: Text(cat),
                    selected: _selectedCategory == cat,
                    onSelected: (selected) {
                      setState(() => _selectedCategory = cat);
                    },
                  ),
                );
              }).toList(),
            ),
          ),
        ),
      ),
      body: GridView.builder(
        padding: const EdgeInsets.all(16),
        gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
          crossAxisCount: 2,
          crossAxisSpacing: 16,
          mainAxisSpacing: 16,
          childAspectRatio: 0.75,
        ),
        itemCount: filteredProducts.length,
        itemBuilder: (context, index) {
          return ProductCard(product: filteredProducts[index]);
        },
      ),
    );
  }
}

// widgets/product_card.dart
class ProductCard extends StatelessWidget {
  final Product product;
  
  const ProductCard({required this.product});
  
  @override
  Widget build(BuildContext context) {
    // ใช้ select เพื่อ rebuild เฉพาะเมื่อ quantity ของ product นี้เปลี่ยน
    final quantity = context.select<CartModel, int>(
      (cart) => cart.quantityOf(product.id),
    );
    
    return Card(
      elevation: 2,
      shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Expanded(
            flex: 3,
            child: Container(
              decoration: BoxDecoration(
                gradient: LinearGradient(
                  colors: [
                    Theme.of(context).colorScheme.primary.withOpacity(0.1),
                    Theme.of(context).colorScheme.primary.withOpacity(0.3),
                  ],
                ),
                borderRadius: const BorderRadius.vertical(top: Radius.circular(12)),
              ),
              child: Stack(
                children: [
                  Center(
                    child: Icon(
                      _getCategoryIcon(product.category),
                      size: 60,
                      color: Theme.of(context).colorScheme.primary,
                    ),
                  ),
                  if (quantity > 0)
                    Positioned(
                      top: 8,
                      right: 8,
                      child: Container(
                        padding: const EdgeInsets.all(6),
                        decoration: const BoxDecoration(
                          color: Colors.green,
                          shape: BoxShape.circle,
                        ),
                        child: Text(
                          '$quantity',
                          style: const TextStyle(
                            color: Colors.white,
                            fontSize: 12,
                            fontWeight: FontWeight.bold,
                          ),
                        ),
                      ),
                    ),
                ],
              ),
            ),
          ),
          Expanded(
            flex: 2,
            child: Padding(
              padding: const EdgeInsets.all(8),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(
                    product.name,
                    style: const TextStyle(
                      fontWeight: FontWeight.bold,
                      fontSize: 13,
                    ),
                    maxLines: 2,
                    overflow: TextOverflow.ellipsis,
                  ),
                  const Spacer(),
                  Text(
                    '฿${product.price.toStringAsFixed(0)}',
                    style: TextStyle(
                      color: Theme.of(context).colorScheme.primary,
                      fontWeight: FontWeight.bold,
                      fontSize: 14,
                    ),
                  ),
                  const SizedBox(height: 4),
                  SizedBox(
                    width: double.infinity,
                    child: ElevatedButton(
                      onPressed: () {
                        context.read<CartModel>().addItem(product);
                        ScaffoldMessenger.of(context).showSnackBar(
                          SnackBar(
                            content: Text('${product.name} added!'),
                            duration: const Duration(seconds: 1),
                          ),
                        );
                      },
                      style: ElevatedButton.styleFrom(
                        padding: const EdgeInsets.symmetric(vertical: 6),
                        textStyle: const TextStyle(fontSize: 12),
                      ),
                      child: const Text('Add to Cart'),
                    ),
                  ),
                ],
              ),
            ),
          ),
        ],
      ),
    );
  }
  
  IconData _getCategoryIcon(String category) {
    switch (category) {
      case 'Phones': return Icons.phone_android;
      case 'Computers': return Icons.laptop;
      case 'Tablets': return Icons.tablet;
      case 'Audio': return Icons.headphones;
      case 'Wearables': return Icons.watch;
      default: return Icons.devices;
    }
  }
}
```

### Cart Screen

```dart
// screens/cart_screen.dart
class CartScreen extends StatelessWidget {
  const CartScreen({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Shopping Cart'),
        actions: [
          Consumer<CartModel>(
            builder: (context, cart, _) {
              if (cart.items.isEmpty) return const SizedBox.shrink();
              return TextButton.icon(
                onPressed: () => _showClearConfirmation(context, cart),
                icon: const Icon(Icons.delete_sweep),
                label: const Text('Clear'),
              );
            },
          ),
        ],
      ),
      body: Consumer<CartModel>(
        builder: (context, cart, _) {
          if (cart.items.isEmpty) {
            return const EmptyCart();
          }
          return Column(
            children: [
              Expanded(
                child: ListView.separated(
                  itemCount: cart.items.length,
                  separatorBuilder: (_, __) => const Divider(height: 1),
                  itemBuilder: (context, index) {
                    final item = cart.items[index];
                    return CartItemWidget(
                      item: item,
                      onRemove: () => cart.removeItem(item.product.id),
                      onQuantityChanged: (qty) => cart.updateQuantity(
                        item.product.id,
                        qty,
                      ),
                    );
                  },
                ),
              ),
              CartSummary(cart: cart),
            ],
          );
        },
      ),
    );
  }
  
  void _showClearConfirmation(BuildContext context, CartModel cart) {
    showDialog(
      context: context,
      builder: (ctx) => AlertDialog(
        title: const Text('Clear Cart'),
        content: const Text('Are you sure you want to remove all items?'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(ctx),
            child: const Text('Cancel'),
          ),
          ElevatedButton(
            onPressed: () {
              cart.clear();
              Navigator.pop(ctx);
            },
            style: ElevatedButton.styleFrom(backgroundColor: Colors.red),
            child: const Text('Clear', style: TextStyle(color: Colors.white)),
          ),
        ],
      ),
    );
  }
}

class EmptyCart extends StatelessWidget {
  const EmptyCart({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return Center(
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          Icon(
            Icons.shopping_cart_outlined,
            size: 100,
            color: Colors.grey[300],
          ),
          const SizedBox(height: 16),
          Text(
            'Your cart is empty',
            style: TextStyle(
              fontSize: 18,
              color: Colors.grey[500],
              fontWeight: FontWeight.w500,
            ),
          ),
          const SizedBox(height: 8),
          Text(
            'Add some products to get started',
            style: TextStyle(color: Colors.grey[400]),
          ),
        ],
      ),
    );
  }
}

class CartItemWidget extends StatelessWidget {
  final CartItem item;
  final VoidCallback onRemove;
  final Function(int) onQuantityChanged;
  
  const CartItemWidget({
    required this.item,
    required this.onRemove,
    required this.onQuantityChanged,
  });
  
  @override
  Widget build(BuildContext context) {
    return ListTile(
      contentPadding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
      leading: Container(
        width: 56,
        height: 56,
        decoration: BoxDecoration(
          color: Theme.of(context).colorScheme.primaryContainer,
          borderRadius: BorderRadius.circular(8),
        ),
        child: Icon(
          Icons.phone_android,
          color: Theme.of(context).colorScheme.primary,
        ),
      ),
      title: Text(
        item.product.name,
        style: const TextStyle(fontWeight: FontWeight.w600),
      ),
      subtitle: Text(
        '฿${item.product.price.toStringAsFixed(0)} each',
        style: TextStyle(color: Colors.grey[600]),
      ),
      trailing: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          Column(
            mainAxisAlignment: MainAxisAlignment.center,
            crossAxisAlignment: CrossAxisAlignment.end,
            children: [
              Text(
                '฿${item.subtotal.toStringAsFixed(0)}',
                style: const TextStyle(
                  fontWeight: FontWeight.bold,
                  fontSize: 16,
                ),
              ),
              Row(
                mainAxisSize: MainAxisSize.min,
                children: [
                  InkWell(
                    onTap: () => onQuantityChanged(item.quantity - 1),
                    child: const Icon(Icons.remove_circle_outline, size: 20),
                  ),
                  Padding(
                    padding: const EdgeInsets.symmetric(horizontal: 8),
                    child: Text(
                      '${item.quantity}',
                      style: const TextStyle(fontWeight: FontWeight.bold),
                    ),
                  ),
                  InkWell(
                    onTap: () => onQuantityChanged(item.quantity + 1),
                    child: const Icon(Icons.add_circle_outline, size: 20),
                  ),
                  const SizedBox(width: 8),
                  InkWell(
                    onTap: onRemove,
                    child: const Icon(Icons.delete_outline, size: 20, color: Colors.red),
                  ),
                ],
              ),
            ],
          ),
        ],
      ),
    );
  }
}

class CartSummary extends StatelessWidget {
  final CartModel cart;
  
  const CartSummary({required this.cart});
  
  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(20),
      decoration: BoxDecoration(
        color: Theme.of(context).colorScheme.surface,
        boxShadow: [
          BoxShadow(
            color: Colors.black.withOpacity(0.1),
            blurRadius: 8,
            offset: const Offset(0, -4),
          ),
        ],
      ),
      child: Column(
        children: [
          _SummaryRow(label: 'Subtotal', value: cart.subtotal),
          _SummaryRow(
            label: 'Shipping',
            value: cart.shipping,
            note: cart.shipping == 0 ? ' (FREE)' : '',
          ),
          const Divider(),
          Row(
            mainAxisAlignment: MainAxisAlignment.spaceBetween,
            children: [
              const Text(
                'Total',
                style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold),
              ),
              Text(
                '฿${cart.total.toStringAsFixed(2)}',
                style: TextStyle(
                  fontSize: 20,
                  fontWeight: FontWeight.bold,
                  color: Theme.of(context).colorScheme.primary,
                ),
              ),
            ],
          ),
          const SizedBox(height: 16),
          SizedBox(
            width: double.infinity,
            child: ElevatedButton(
              onPressed: () => _showCheckoutDialog(context),
              style: ElevatedButton.styleFrom(
                padding: const EdgeInsets.symmetric(vertical: 16),
              ),
              child: Text(
                'Checkout (${cart.itemCount} items)',
                style: const TextStyle(fontSize: 16),
              ),
            ),
          ),
        ],
      ),
    );
  }
  
  void _showCheckoutDialog(BuildContext context) {
    showDialog(
      context: context,
      builder: (ctx) => AlertDialog(
        title: const Text('Order Confirmed!'),
        content: Text(
          'Thank you for your order!\nTotal: ฿${cart.total.toStringAsFixed(2)}',
        ),
        actions: [
          ElevatedButton(
            onPressed: () {
              context.read<CartModel>().clear();
              Navigator.pop(ctx);
            },
            child: const Text('OK'),
          ),
        ],
      ),
    );
  }
}

class _SummaryRow extends StatelessWidget {
  final String label;
  final double value;
  final String note;
  
  const _SummaryRow({
    required this.label,
    required this.value,
    this.note = '',
  });
  
  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceBetween,
        children: [
          Text(
            '$label$note',
            style: TextStyle(color: Colors.grey[600]),
          ),
          Text('฿${value.toStringAsFixed(2)}'),
        ],
      ),
    );
  }
}
```

---

## สรุป Provider

### ข้อดี
- ง่ายกว่า InheritedWidget มาก
- Flutter team แนะนำ
- เหมาะกับ app ขนาดกลาง
- มี devtools support

### ข้อเสีย
- ไม่มี built-in async support ดีเท่า Riverpod
- ต้องระวัง ChangeNotifier memory leaks
- บาง use case ต้องเขียน boilerplate

### Best Practices
```dart
// 1. ใช้ const constructors
const MyWidget();

// 2. ใช้ context.select แทน context.watch เมื่อเป็นไปได้
final name = context.select<UserModel, String>((u) => u.name);

// 3. แยก Provider ตาม feature
MultiProvider(providers: [
  ChangeNotifierProvider(create: (_) => UserModel()),
  ChangeNotifierProvider(create: (_) => CartModel()),
]);

// 4. ใช้ context.read ใน callbacks เท่านั้น
ElevatedButton(
  onPressed: () => context.read<CounterModel>().increment(),
);
```
