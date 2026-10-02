# Part 31: State Management with setState

## การจัดการ State ด้วย setState

### ทำความเข้าใจ State ใน Flutter

State คือข้อมูลที่ widget ใช้ในการแสดงผล และสามารถเปลี่ยนแปลงได้ตลอดอายุการใช้งาน เมื่อ state เปลี่ยน Flutter จะ rebuild widget เพื่อแสดงข้อมูลใหม่

```dart
// StatelessWidget - ไม่มี state
class MyText extends StatelessWidget {
  final String text;
  const MyText({required this.text});
  
  @override
  Widget build(BuildContext context) {
    return Text(text);
  }
}

// StatefulWidget - มี state ที่เปลี่ยนได้
class Counter extends StatefulWidget {
  const Counter({Key? key}) : super(key: key);
  
  @override
  State<Counter> createState() => _CounterState();
}

class _CounterState extends State<Counter> {
  int _count = 0;  // นี่คือ state
  
  @override
  Widget build(BuildContext context) {
    return Text('Count: $_count');
  }
}
```

---

## setState Deep Dive

### วิธีการทำงานของ setState

`setState()` บอก Flutter ว่า state ของ widget นี้มีการเปลี่ยนแปลง และต้องการ rebuild

```dart
class _CounterState extends State<Counter> {
  int _count = 0;
  String _message = '';
  List<String> _items = [];
  
  void _increment() {
    setState(() {
      // ทุกอย่างใน block นี้จะถูก rebuild
      _count++;
      _message = 'Count is now $_count';
      _items.add('Item $_count');
    });
  }
  
  // วิธีที่ไม่ถูกต้อง - ไม่ทำให้ UI update
  void _incrementWrong() {
    _count++;  // เปลี่ยน state แต่ไม่ call setState
    // UI จะไม่ update!
  }
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text('Count: $_count'),
        Text(_message),
        ...(_items.map((item) => Text(item))),
        ElevatedButton(
          onPressed: _increment,
          child: const Text('Increment'),
        ),
      ],
    );
  }
}
```

### setState กับ Async Operations

```dart
class _DataState extends State<DataWidget> {
  String _data = '';
  bool _isLoading = false;
  String _error = '';
  
  Future<void> _fetchData() async {
    // เริ่ม loading
    setState(() {
      _isLoading = true;
      _error = '';
    });
    
    try {
      // จำลองการ fetch data
      await Future.delayed(const Duration(seconds: 2));
      final result = 'Data loaded successfully';
      
      // ตรวจสอบว่า widget ยัง mounted อยู่หรือไม่
      if (mounted) {
        setState(() {
          _data = result;
          _isLoading = false;
        });
      }
    } catch (e) {
      if (mounted) {
        setState(() {
          _error = e.toString();
          _isLoading = false;
        });
      }
    }
  }
  
  @override
  Widget build(BuildContext context) {
    if (_isLoading) {
      return const CircularProgressIndicator();
    }
    if (_error.isNotEmpty) {
      return Text('Error: $_error');
    }
    return Column(
      children: [
        Text(_data),
        ElevatedButton(
          onPressed: _fetchData,
          child: const Text('Fetch Data'),
        ),
      ],
    );
  }
}
```

### เมื่อไหร่ที่ควรใช้ setState

```dart
class _FormState extends State<FormWidget> {
  final _formKey = GlobalKey<FormState>();
  String _name = '';
  String _email = '';
  bool _agreeToTerms = false;
  
  // setState เหมาะกับ local UI state
  void _toggleTerms(bool? value) {
    setState(() {
      _agreeToTerms = value ?? false;
    });
  }
  
  void _submitForm() {
    if (_formKey.currentState!.validate() && _agreeToTerms) {
      _formKey.currentState!.save();
      // ส่งข้อมูล
      print('Name: $_name, Email: $_email');
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return Form(
      key: _formKey,
      child: Column(
        children: [
          TextFormField(
            decoration: const InputDecoration(labelText: 'Name'),
            validator: (value) {
              if (value == null || value.isEmpty) {
                return 'Please enter your name';
              }
              return null;
            },
            onSaved: (value) => _name = value ?? '',
          ),
          TextFormField(
            decoration: const InputDecoration(labelText: 'Email'),
            validator: (value) {
              if (value == null || !value.contains('@')) {
                return 'Please enter a valid email';
              }
              return null;
            },
            onSaved: (value) => _email = value ?? '',
          ),
          CheckboxListTile(
            title: const Text('I agree to terms'),
            value: _agreeToTerms,
            onChanged: _toggleTerms,
          ),
          ElevatedButton(
            onPressed: _submitForm,
            child: const Text('Submit'),
          ),
        ],
      ),
    );
  }
}
```

---

## Lifting State Up

### ปัญหาเมื่อ State อยู่ต่ำเกินไป

เมื่อหลาย widget ต้องการ share state เดียวกัน เราต้อง "lift" state ขึ้นไปใน parent widget ที่ใกล้ที่สุด

```dart
// ปัญหา: State แยกกันใน 2 widget
class _DisplayState extends State<DisplayWidget> {
  int _count = 0;  // Widget A มี count ของตัวเอง
  @override
  Widget build(BuildContext context) => Text('Count: $_count');
}

class _ButtonState extends State<ButtonWidget> {
  int _count = 0;  // Widget B มี count ของตัวเอง
  // ปัญหา: ทั้งสอง widget ไม่ได้ share count เดียวกัน!
  @override
  Widget build(BuildContext context) => ElevatedButton(
    onPressed: () => setState(() => _count++),
    child: const Text('Increment'),
  );
}
```

### วิธีแก้: Lift State Up

```dart
// Parent widget ที่เก็บ shared state
class CounterPage extends StatefulWidget {
  const CounterPage({Key? key}) : super(key: key);
  
  @override
  State<CounterPage> createState() => _CounterPageState();
}

class _CounterPageState extends State<CounterPage> {
  int _count = 0;  // State อยู่ที่ parent
  
  void _increment() {
    setState(() {
      _count++;
    });
  }
  
  void _decrement() {
    setState(() {
      if (_count > 0) _count--;
    });
  }
  
  void _reset() {
    setState(() {
      _count = 0;
    });
  }
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // ส่ง state ลงไปยัง child widgets
        CountDisplay(count: _count),
        CountControls(
          onIncrement: _increment,
          onDecrement: _decrement,
          onReset: _reset,
        ),
      ],
    );
  }
}

// Child widget รับ state ผ่าน constructor
class CountDisplay extends StatelessWidget {
  final int count;
  const CountDisplay({required this.count});
  
  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(16),
      decoration: BoxDecoration(
        color: count > 5 ? Colors.green : Colors.blue,
        borderRadius: BorderRadius.circular(8),
      ),
      child: Text(
        'Count: $count',
        style: const TextStyle(fontSize: 24, color: Colors.white),
      ),
    );
  }
}

// Child widget รับ callbacks สำหรับ actions
class CountControls extends StatelessWidget {
  final VoidCallback onIncrement;
  final VoidCallback onDecrement;
  final VoidCallback onReset;
  
  const CountControls({
    required this.onIncrement,
    required this.onDecrement,
    required this.onReset,
  });
  
  @override
  Widget build(BuildContext context) {
    return Row(
      mainAxisAlignment: MainAxisAlignment.spaceEvenly,
      children: [
        IconButton(
          onPressed: onDecrement,
          icon: const Icon(Icons.remove),
        ),
        ElevatedButton(
          onPressed: onReset,
          child: const Text('Reset'),
        ),
        IconButton(
          onPressed: onIncrement,
          icon: const Icon(Icons.add),
        ),
      ],
    );
  }
}
```

---

## State Sharing Between Widgets

### Pattern 1: Callback Pattern

```dart
class ParentWidget extends StatefulWidget {
  @override
  State<ParentWidget> createState() => _ParentWidgetState();
}

class _ParentWidgetState extends State<ParentWidget> {
  String _selectedItem = '';
  
  void _onItemSelected(String item) {
    setState(() {
      _selectedItem = item;
    });
  }
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // แสดงผลลัพธ์
        SelectedItemDisplay(item: _selectedItem),
        // Widget ที่สร้าง events
        ItemList(onItemSelected: _onItemSelected),
      ],
    );
  }
}

class ItemList extends StatelessWidget {
  final Function(String) onItemSelected;
  
  const ItemList({required this.onItemSelected});
  
  @override
  Widget build(BuildContext context) {
    final items = ['Apple', 'Banana', 'Cherry'];
    return Column(
      children: items
          .map((item) => ListTile(
                title: Text(item),
                onTap: () => onItemSelected(item),
              ))
          .toList(),
    );
  }
}

class SelectedItemDisplay extends StatelessWidget {
  final String item;
  
  const SelectedItemDisplay({required this.item});
  
  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(16),
      child: Text(
        item.isEmpty ? 'No item selected' : 'Selected: $item',
        style: const TextStyle(fontSize: 18),
      ),
    );
  }
}
```

### Pattern 2: InheritedWidget

```dart
// Custom InheritedWidget สำหรับ share data
class AppData extends InheritedWidget {
  final int counter;
  final VoidCallback onIncrement;
  
  const AppData({
    required this.counter,
    required this.onIncrement,
    required Widget child,
  }) : super(child: child);
  
  static AppData of(BuildContext context) {
    final data = context.dependOnInheritedWidgetOfExactType<AppData>();
    if (data == null) throw Exception('AppData not found in tree');
    return data;
  }
  
  @override
  bool updateShouldNotify(AppData oldWidget) {
    return counter != oldWidget.counter;
  }
}

// Root widget ที่ wrap ด้วย InheritedWidget
class AppRoot extends StatefulWidget {
  @override
  State<AppRoot> createState() => _AppRootState();
}

class _AppRootState extends State<AppRoot> {
  int _counter = 0;
  
  @override
  Widget build(BuildContext context) {
    return AppData(
      counter: _counter,
      onIncrement: () => setState(() => _counter++),
      child: const MyApp(),
    );
  }
}

// Widget ที่ใช้ข้อมูลจาก InheritedWidget
class CounterText extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final data = AppData.of(context);
    return Text('Counter: ${data.counter}');
  }
}

class IncrementButton extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final data = AppData.of(context);
    return ElevatedButton(
      onPressed: data.onIncrement,
      child: const Text('Increment'),
    );
  }
}
```

---

## When setState is Not Enough

### ปัญหาที่ setState ไม่สามารถแก้ได้ดี

```dart
// 1. State ที่ต้องแชร์ระหว่างหลาย screen
// ต้องส่ง state ผ่าน Navigator arguments ซึ่งซับซ้อนมาก

// 2. State ที่ต้องการ persist ข้าม sessions
// setState ไม่สามารถบันทึก state เมื่อ app ปิด

// 3. Complex business logic
// setState ทำให้ logic และ UI ปนกัน

// 4. Deep widget tree
// Lifting state up ไกลเกินไป ทำให้โค้ดซับซ้อน

// ตัวอย่างปัญหา: Shopping cart ที่ต้องแชร์ข้าม screens
class _HomePageState extends State<HomePage> {
  List<String> _cart = []; // ปัญหา: cart ควรอยู่ใน app level
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(
        children: [
          Text('Cart: ${_cart.length} items'),
          ElevatedButton(
            onPressed: () {
              // ปัญหา: cart ของหน้า Home ไม่ sync กับหน้า Cart
              Navigator.push(
                context,
                MaterialPageRoute(
                  builder: (_) => CartPage(
                    cart: _cart,
                    onRemove: (item) => setState(() => _cart.remove(item)),
                  ),
                ),
              );
            },
            child: const Text('View Cart'),
          ),
        ],
      ),
    );
  }
}
```

### สัญญาณว่าควรเปลี่ยนไปใช้ State Management Library

```
1. Widget tree ลึกมาก และต้องส่ง props หลายชั้น (prop drilling)
2. State เดียวกันใช้ใน หลาย screen
3. Business logic ซับซ้อน มีหลาย operations
4. ต้องการ caching หรือ persistence
5. Team ใหญ่ ต้องการ structure ชัดเจน
6. Testing ยาก เพราะ logic อยู่ใน widget
```

---

## Performance Considerations

### การใช้ const Constructor

```dart
// ไม่ดี: rebuild ทุกครั้ง
class Counter extends StatefulWidget {
  @override
  State<Counter> createState() => _CounterState();
}

class _CounterState extends State<Counter> {
  int _count = 0;
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text('Count: $_count'),
        Row(  // Row นี้ถูก rebuild ทุกครั้งที่ count เปลี่ยน
          children: [
            ElevatedButton(  // Button เหล่านี้ไม่ได้เปลี่ยน แต่ถูก rebuild
              onPressed: () => setState(() => _count++),
              child: Text('Add'),
            ),
          ],
        ),
      ],
    );
  }
}
```

### แยก Widget ให้เล็กลงเพื่อ Performance

```dart
class OptimizedCounter extends StatefulWidget {
  @override
  State<OptimizedCounter> createState() => _OptimizedCounterState();
}

class _OptimizedCounterState extends State<OptimizedCounter> {
  int _count = 0;
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text('Count: $_count'),  // เฉพาะตัวนี้ที่ rebuild
        const CounterButtons(),  // const ทำให้ไม่ rebuild
      ],
    );
  }
}

// const widget ที่ไม่ขึ้นกับ state
class CounterButtons extends StatelessWidget {
  const CounterButtons();
  
  @override
  Widget build(BuildContext context) {
    return Row(
      children: const [
        ElevatedButton(
          onPressed: null,  // ในความเป็นจริงต้องผ่าน callback
          child: Text('Add'),
        ),
      ],
    );
  }
}
```

### ใช้ shouldRebuild เพื่อ Optimize

```dart
class SmartCounter extends StatefulWidget {
  @override
  State<SmartCounter> createState() => _SmartCounterState();
}

class _SmartCounterState extends State<SmartCounter> {
  int _count = 0;
  bool _showDetails = false;
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // แยก widget ที่ขึ้นกับ state ต่างกัน
        CountDisplay(count: _count),
        DetailsSection(
          count: _count,
          isVisible: _showDetails,
        ),
        Row(
          children: [
            ElevatedButton(
              onPressed: () => setState(() => _count++),
              child: const Text('Increment'),
            ),
            ElevatedButton(
              onPressed: () => setState(() => _showDetails = !_showDetails),
              child: const Text('Toggle Details'),
            ),
          ],
        ),
      ],
    );
  }
}

class CountDisplay extends StatelessWidget {
  final int count;
  const CountDisplay({required this.count});
  
  @override
  Widget build(BuildContext context) {
    print('CountDisplay rebuilt');  // ดู rebuild pattern
    return Text('Count: $count', style: const TextStyle(fontSize: 24));
  }
}

class DetailsSection extends StatelessWidget {
  final int count;
  final bool isVisible;
  
  const DetailsSection({required this.count, required this.isVisible});
  
  @override
  Widget build(BuildContext context) {
    print('DetailsSection rebuilt');
    if (!isVisible) return const SizedBox.shrink();
    return Column(
      children: [
        Text('Squared: ${count * count}'),
        Text('Doubled: ${count * 2}'),
      ],
    );
  }
}
```

### การใช้ ValueNotifier สำหรับ Performance

```dart
class ValueNotifierExample extends StatefulWidget {
  @override
  State<ValueNotifierExample> createState() => _ValueNotifierExampleState();
}

class _ValueNotifierExampleState extends State<ValueNotifierExample> {
  // ValueNotifier สำหรับ update เฉพาะส่วนที่ต้องการ
  final _counter = ValueNotifier<int>(0);
  final _message = ValueNotifier<String>('Hello');
  
  @override
  void dispose() {
    _counter.dispose();
    _message.dispose();
    super.dispose();
  }
  
  @override
  Widget build(BuildContext context) {
    print('Main widget built');  // จะไม่ rebuild เมื่อใช้ ValueListenableBuilder
    return Column(
      children: [
        // เฉพาะ widget นี้ที่ rebuild เมื่อ _counter เปลี่ยน
        ValueListenableBuilder<int>(
          valueListenable: _counter,
          builder: (context, count, child) {
            return Text('Count: $count');
          },
        ),
        // เฉพาะ widget นี้ที่ rebuild เมื่อ _message เปลี่ยน
        ValueListenableBuilder<String>(
          valueListenable: _message,
          builder: (context, message, child) {
            return Text('Message: $message');
          },
        ),
        ElevatedButton(
          onPressed: () => _counter.value++,
          child: const Text('Increment Counter'),
        ),
        ElevatedButton(
          onPressed: () => _message.value = 'Updated at ${DateTime.now()}',
          child: const Text('Update Message'),
        ),
      ],
    );
  }
}
```

---

## Workshop: Shopping Cart with setState

### โครงสร้างโปรเจค

```
shopping_cart_setstate/
├── main.dart
├── models/
│   └── product.dart
├── screens/
│   ├── home_screen.dart
│   └── cart_screen.dart
└── widgets/
    ├── product_card.dart
    └── cart_badge.dart
```

### Product Model

```dart
// models/product.dart
class Product {
  final String id;
  final String name;
  final double price;
  final String imageUrl;
  final String description;
  int quantity;
  
  Product({
    required this.id,
    required this.name,
    required this.price,
    required this.imageUrl,
    required this.description,
    this.quantity = 0,
  });
  
  Product copyWith({
    String? id,
    String? name,
    double? price,
    String? imageUrl,
    String? description,
    int? quantity,
  }) {
    return Product(
      id: id ?? this.id,
      name: name ?? this.name,
      price: price ?? this.price,
      imageUrl: imageUrl ?? this.imageUrl,
      description: description ?? this.description,
      quantity: quantity ?? this.quantity,
    );
  }
}
```

### App Root with State

```dart
// main.dart
import 'package:flutter/material.dart';

void main() {
  runApp(const ShoppingApp());
}

class ShoppingApp extends StatelessWidget {
  const ShoppingApp({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Shopping Cart',
      theme: ThemeData(
        primarySwatch: Colors.indigo,
        useMaterial3: true,
      ),
      home: const ShoppingRoot(),
    );
  }
}

// Root widget ที่เก็บ shopping cart state
class ShoppingRoot extends StatefulWidget {
  const ShoppingRoot({Key? key}) : super(key: key);
  
  @override
  State<ShoppingRoot> createState() => _ShoppingRootState();
}

class _ShoppingRootState extends State<ShoppingRoot> {
  // Cart state อยู่ที่ level สูงสุด
  final List<CartItem> _cartItems = [];
  int _currentIndex = 0;
  
  // Products data
  final List<Product> _products = [
    Product(
      id: '1',
      name: 'iPhone 15 Pro',
      price: 42900,
      imageUrl: 'https://via.placeholder.com/200',
      description: 'Apple iPhone 15 Pro 256GB',
    ),
    Product(
      id: '2',
      name: 'Samsung Galaxy S24',
      price: 35900,
      imageUrl: 'https://via.placeholder.com/200',
      description: 'Samsung Galaxy S24 Ultra',
    ),
    Product(
      id: '3',
      name: 'MacBook Air M3',
      price: 49900,
      imageUrl: 'https://via.placeholder.com/200',
      description: 'MacBook Air 13-inch M3 chip',
    ),
    Product(
      id: '4',
      name: 'iPad Pro',
      price: 29900,
      imageUrl: 'https://via.placeholder.com/200',
      description: 'iPad Pro 12.9-inch M2',
    ),
    Product(
      id: '5',
      name: 'AirPods Pro',
      price: 8990,
      imageUrl: 'https://via.placeholder.com/200',
      description: 'AirPods Pro 2nd generation',
    ),
  ];
  
  // Cart operations
  void _addToCart(Product product) {
    setState(() {
      final existingIndex = _cartItems.indexWhere((item) => item.product.id == product.id);
      if (existingIndex >= 0) {
        // เพิ่ม quantity ถ้ามีอยู่แล้ว
        _cartItems[existingIndex] = CartItem(
          product: _cartItems[existingIndex].product,
          quantity: _cartItems[existingIndex].quantity + 1,
        );
      } else {
        // เพิ่มใหม่
        _cartItems.add(CartItem(product: product, quantity: 1));
      }
    });
    
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(
        content: Text('${product.name} added to cart'),
        duration: const Duration(seconds: 1),
        action: SnackBarAction(
          label: 'View Cart',
          onPressed: () => setState(() => _currentIndex = 1),
        ),
      ),
    );
  }
  
  void _removeFromCart(String productId) {
    setState(() {
      _cartItems.removeWhere((item) => item.product.id == productId);
    });
  }
  
  void _updateQuantity(String productId, int quantity) {
    setState(() {
      if (quantity <= 0) {
        _cartItems.removeWhere((item) => item.product.id == productId);
      } else {
        final index = _cartItems.indexWhere((item) => item.product.id == productId);
        if (index >= 0) {
          _cartItems[index] = CartItem(
            product: _cartItems[index].product,
            quantity: quantity,
          );
        }
      }
    });
  }
  
  void _clearCart() {
    setState(() {
      _cartItems.clear();
    });
  }
  
  // Computed values
  int get _totalItems => _cartItems.fold(0, (sum, item) => sum + item.quantity);
  
  double get _totalPrice => _cartItems.fold(
    0.0,
    (sum, item) => sum + (item.product.price * item.quantity),
  );
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(_currentIndex == 0 ? 'Products' : 'Shopping Cart'),
        backgroundColor: Colors.indigo,
        foregroundColor: Colors.white,
        actions: [
          if (_currentIndex == 0)
            Stack(
              children: [
                IconButton(
                  onPressed: () => setState(() => _currentIndex = 1),
                  icon: const Icon(Icons.shopping_cart),
                ),
                if (_totalItems > 0)
                  Positioned(
                    right: 8,
                    top: 8,
                    child: Container(
                      padding: const EdgeInsets.all(4),
                      decoration: const BoxDecoration(
                        color: Colors.red,
                        shape: BoxShape.circle,
                      ),
                      child: Text(
                        '$_totalItems',
                        style: const TextStyle(
                          color: Colors.white,
                          fontSize: 10,
                          fontWeight: FontWeight.bold,
                        ),
                      ),
                    ),
                  ),
              ],
            ),
        ],
      ),
      body: IndexedStack(
        index: _currentIndex,
        children: [
          ProductsScreen(
            products: _products,
            cartItems: _cartItems,
            onAddToCart: _addToCart,
          ),
          CartScreen(
            cartItems: _cartItems,
            totalPrice: _totalPrice,
            onRemove: _removeFromCart,
            onUpdateQuantity: _updateQuantity,
            onClear: _clearCart,
          ),
        ],
      ),
      bottomNavigationBar: BottomNavigationBar(
        currentIndex: _currentIndex,
        onTap: (index) => setState(() => _currentIndex = index),
        items: [
          const BottomNavigationBarItem(
            icon: Icon(Icons.store),
            label: 'Products',
          ),
          BottomNavigationBarItem(
            icon: Badge(
              isLabelVisible: _totalItems > 0,
              label: Text('$_totalItems'),
              child: const Icon(Icons.shopping_cart),
            ),
            label: 'Cart',
          ),
        ],
      ),
    );
  }
}

// Data classes
class CartItem {
  final Product product;
  final int quantity;
  
  CartItem({required this.product, required this.quantity});
}
```

### Products Screen

```dart
// screens/home_screen.dart
class ProductsScreen extends StatelessWidget {
  final List<Product> products;
  final List<CartItem> cartItems;
  final Function(Product) onAddToCart;
  
  const ProductsScreen({
    required this.products,
    required this.cartItems,
    required this.onAddToCart,
  });
  
  int _getQuantityInCart(String productId) {
    final item = cartItems.where((item) => item.product.id == productId);
    if (item.isEmpty) return 0;
    return item.first.quantity;
  }
  
  @override
  Widget build(BuildContext context) {
    return GridView.builder(
      padding: const EdgeInsets.all(16),
      gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
        crossAxisCount: 2,
        crossAxisSpacing: 16,
        mainAxisSpacing: 16,
        childAspectRatio: 0.7,
      ),
      itemCount: products.length,
      itemBuilder: (context, index) {
        final product = products[index];
        final quantityInCart = _getQuantityInCart(product.id);
        
        return ProductCard(
          product: product,
          quantityInCart: quantityInCart,
          onAddToCart: () => onAddToCart(product),
        );
      },
    );
  }
}

class ProductCard extends StatelessWidget {
  final Product product;
  final int quantityInCart;
  final VoidCallback onAddToCart;
  
  const ProductCard({
    required this.product,
    required this.quantityInCart,
    required this.onAddToCart,
  });
  
  @override
  Widget build(BuildContext context) {
    return Card(
      elevation: 2,
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.circular(12),
      ),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Expanded(
            flex: 3,
            child: Container(
              decoration: BoxDecoration(
                color: Colors.grey[200],
                borderRadius: const BorderRadius.vertical(
                  top: Radius.circular(12),
                ),
              ),
              child: Center(
                child: Icon(
                  Icons.phone_android,
                  size: 60,
                  color: Colors.indigo[300],
                ),
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
                      fontSize: 12,
                    ),
                    maxLines: 2,
                    overflow: TextOverflow.ellipsis,
                  ),
                  const Spacer(),
                  Row(
                    children: [
                      Expanded(
                        child: Text(
                          '฿${product.price.toStringAsFixed(0)}',
                          style: const TextStyle(
                            color: Colors.indigo,
                            fontWeight: FontWeight.bold,
                          ),
                        ),
                      ),
                      if (quantityInCart > 0)
                        Container(
                          padding: const EdgeInsets.symmetric(
                            horizontal: 6,
                            vertical: 2,
                          ),
                          decoration: BoxDecoration(
                            color: Colors.green,
                            borderRadius: BorderRadius.circular(10),
                          ),
                          child: Text(
                            '$quantityInCart',
                            style: const TextStyle(
                              color: Colors.white,
                              fontSize: 10,
                            ),
                          ),
                        ),
                    ],
                  ),
                  const SizedBox(height: 4),
                  SizedBox(
                    width: double.infinity,
                    child: ElevatedButton(
                      onPressed: onAddToCart,
                      style: ElevatedButton.styleFrom(
                        padding: const EdgeInsets.symmetric(vertical: 4),
                        backgroundColor: Colors.indigo,
                        foregroundColor: Colors.white,
                      ),
                      child: const Text('Add to Cart', style: TextStyle(fontSize: 11)),
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
}
```

### Cart Screen

```dart
// screens/cart_screen.dart
class CartScreen extends StatelessWidget {
  final List<CartItem> cartItems;
  final double totalPrice;
  final Function(String) onRemove;
  final Function(String, int) onUpdateQuantity;
  final VoidCallback onClear;
  
  const CartScreen({
    required this.cartItems,
    required this.totalPrice,
    required this.onRemove,
    required this.onUpdateQuantity,
    required this.onClear,
  });
  
  @override
  Widget build(BuildContext context) {
    if (cartItems.isEmpty) {
      return const Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Icon(Icons.shopping_cart_outlined, size: 80, color: Colors.grey),
            SizedBox(height: 16),
            Text(
              'Your cart is empty',
              style: TextStyle(fontSize: 18, color: Colors.grey),
            ),
          ],
        ),
      );
    }
    
    return Column(
      children: [
        Expanded(
          child: ListView.builder(
            itemCount: cartItems.length,
            itemBuilder: (context, index) {
              final item = cartItems[index];
              return CartItemTile(
                item: item,
                onRemove: () => onRemove(item.product.id),
                onQuantityChanged: (qty) => onUpdateQuantity(item.product.id, qty),
              );
            },
          ),
        ),
        Container(
          padding: const EdgeInsets.all(16),
          decoration: BoxDecoration(
            color: Colors.white,
            boxShadow: [
              BoxShadow(
                color: Colors.grey.withOpacity(0.2),
                spreadRadius: 1,
                blurRadius: 5,
                offset: const Offset(0, -3),
              ),
            ],
          ),
          child: Column(
            children: [
              Row(
                mainAxisAlignment: MainAxisAlignment.spaceBetween,
                children: [
                  const Text(
                    'Total:',
                    style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold),
                  ),
                  Text(
                    '฿${totalPrice.toStringAsFixed(2)}',
                    style: const TextStyle(
                      fontSize: 20,
                      fontWeight: FontWeight.bold,
                      color: Colors.indigo,
                    ),
                  ),
                ],
              ),
              const SizedBox(height: 12),
              Row(
                children: [
                  Expanded(
                    child: OutlinedButton(
                      onPressed: () {
                        showDialog(
                          context: context,
                          builder: (ctx) => AlertDialog(
                            title: const Text('Clear Cart'),
                            content: const Text('Remove all items from cart?'),
                            actions: [
                              TextButton(
                                onPressed: () => Navigator.pop(ctx),
                                child: const Text('Cancel'),
                              ),
                              ElevatedButton(
                                onPressed: () {
                                  onClear();
                                  Navigator.pop(ctx);
                                },
                                style: ElevatedButton.styleFrom(
                                  backgroundColor: Colors.red,
                                  foregroundColor: Colors.white,
                                ),
                                child: const Text('Clear'),
                              ),
                            ],
                          ),
                        );
                      },
                      child: const Text('Clear Cart'),
                    ),
                  ),
                  const SizedBox(width: 12),
                  Expanded(
                    flex: 2,
                    child: ElevatedButton(
                      onPressed: () {
                        showDialog(
                          context: context,
                          builder: (ctx) => AlertDialog(
                            title: const Text('Order Confirmed!'),
                            content: Text(
                              'Your order of ฿${totalPrice.toStringAsFixed(2)} has been placed.',
                            ),
                            actions: [
                              ElevatedButton(
                                onPressed: () {
                                  onClear();
                                  Navigator.pop(ctx);
                                },
                                child: const Text('OK'),
                              ),
                            ],
                          ),
                        );
                      },
                      style: ElevatedButton.styleFrom(
                        backgroundColor: Colors.indigo,
                        foregroundColor: Colors.white,
                        padding: const EdgeInsets.symmetric(vertical: 12),
                      ),
                      child: const Text('Checkout', style: TextStyle(fontSize: 16)),
                    ),
                  ),
                ],
              ),
            ],
          ),
        ),
      ],
    );
  }
}

class CartItemTile extends StatelessWidget {
  final CartItem item;
  final VoidCallback onRemove;
  final Function(int) onQuantityChanged;
  
  const CartItemTile({
    required this.item,
    required this.onRemove,
    required this.onQuantityChanged,
  });
  
  @override
  Widget build(BuildContext context) {
    return Dismissible(
      key: Key(item.product.id),
      direction: DismissDirection.endToStart,
      background: Container(
        alignment: Alignment.centerRight,
        padding: const EdgeInsets.only(right: 20),
        color: Colors.red,
        child: const Icon(Icons.delete, color: Colors.white),
      ),
      onDismissed: (_) => onRemove(),
      child: ListTile(
        leading: Container(
          width: 60,
          height: 60,
          decoration: BoxDecoration(
            color: Colors.grey[200],
            borderRadius: BorderRadius.circular(8),
          ),
          child: const Icon(Icons.phone_android, color: Colors.indigo),
        ),
        title: Text(
          item.product.name,
          style: const TextStyle(fontWeight: FontWeight.bold),
        ),
        subtitle: Text('฿${item.product.price.toStringAsFixed(0)} each'),
        trailing: Row(
          mainAxisSize: MainAxisSize.min,
          children: [
            IconButton(
              onPressed: () => onQuantityChanged(item.quantity - 1),
              icon: const Icon(Icons.remove_circle_outline),
              color: Colors.indigo,
            ),
            Text(
              '${item.quantity}',
              style: const TextStyle(fontSize: 16, fontWeight: FontWeight.bold),
            ),
            IconButton(
              onPressed: () => onQuantityChanged(item.quantity + 1),
              icon: const Icon(Icons.add_circle_outline),
              color: Colors.indigo,
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## สรุป

### เมื่อไหร่ใช้ setState
- State ที่ใช้ใน widget เดียว
- UI state ง่ายๆ (toggle, counter)
- App ขนาดเล็กที่ไม่ซับซ้อน

### ข้อดี
- ง่าย เข้าใจได้เลย
- Built-in ใน Flutter ไม่ต้อง install package เพิ่ม
- เหมาะกับ prototype และ simple apps

### ข้อเสีย
- Prop drilling เมื่อ widget tree ลึก
- ยาก scale สำหรับ app ใหญ่
- Logic และ UI ปนกัน
- ยาก test

### การเลือกใช้
```
Simple UI state → setState
Share ระหว่าง 2-3 widgets → Lifting State Up
Share ทั้ง app → Provider/Riverpod/GetX
Complex logic → BLoC
```
