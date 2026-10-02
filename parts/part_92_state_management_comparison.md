# Part 92: State Management Comparison

## 🎯 เป้าหมายของ Part นี้
- เปรียบเทียบ Provider, Riverpod, BLoC, GetX
- รู้ว่าเมื่อไหร่ใช้อะไร
- Migration strategies
- Performance comparison
- Real-world use cases

---

## 1. ภาพรวม State Management

```
setState    → Simple, no library needed
Provider    → Official, ChangeNotifier
Riverpod    → Type-safe, Provider++
BLoC        → Scalable, Events/States
GetX        → Feature-rich, minimal boilerplate
Zustand     → (web-inspired, simple)
```

---

## 2. Provider - ตัวอย่างสมบูรณ์

```dart
// pubspec.yaml:
// provider: ^6.1.0

import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

// ============ ViewModel ============
class CartViewModel extends ChangeNotifier {
  final List<CartItem> _items = [];
  
  List<CartItem> get items => List.unmodifiable(_items);
  
  double get total => _items.fold(0, (sum, item) => sum + item.subtotal);
  
  int get itemCount => _items.fold(0, (sum, item) => sum + item.quantity);
  
  void addItem(Product product) {
    final existing = _items.indexWhere((i) => i.product.id == product.id);
    if (existing >= 0) {
      _items[existing] = _items[existing].copyWith(
        quantity: _items[existing].quantity + 1,
      );
    } else {
      _items.add(CartItem(product: product, quantity: 1));
    }
    notifyListeners();
  }
  
  void removeItem(String productId) {
    _items.removeWhere((i) => i.product.id == productId);
    notifyListeners();
  }
  
  void updateQuantity(String productId, int quantity) {
    if (quantity <= 0) {
      removeItem(productId);
      return;
    }
    final index = _items.indexWhere((i) => i.product.id == productId);
    if (index >= 0) {
      _items[index] = _items[index].copyWith(quantity: quantity);
      notifyListeners();
    }
  }
  
  void clear() {
    _items.clear();
    notifyListeners();
  }
}

class CartItem {
  final Product product;
  final int quantity;
  
  const CartItem({required this.product, required this.quantity});
  
  double get subtotal => product.price * quantity;
  
  CartItem copyWith({int? quantity}) {
    return CartItem(product: product, quantity: quantity ?? this.quantity);
  }
}

class Product {
  final String id;
  final String name;
  final double price;
  final String imageUrl;
  
  const Product({
    required this.id, 
    required this.name, 
    required this.price,
    required this.imageUrl,
  });
}

// ============ Setup ============
void main() {
  runApp(
    MultiProvider(
      providers: [
        ChangeNotifierProvider(create: (_) => CartViewModel()),
        // เพิ่ม providers อื่นๆ
      ],
      child: const MyApp(),
    ),
  );
}

// ============ Usage ============
class ProductCard extends StatelessWidget {
  final Product product;
  const ProductCard({super.key, required this.product});

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Column(
        children: [
          Text(product.name),
          Text('฿${product.price}'),
          ElevatedButton(
            onPressed: () {
              // ใช้ read เมื่อไม่ต้องการ rebuild
              context.read<CartViewModel>().addItem(product);
            },
            child: const Text('เพิ่มลงตะกร้า'),
          ),
        ],
      ),
    );
  }
}

class CartIcon extends StatelessWidget {
  const CartIcon({super.key});

  @override
  Widget build(BuildContext context) {
    // ใช้ watch เมื่อต้องการ rebuild เมื่อข้อมูลเปลี่ยน
    final count = context.watch<CartViewModel>().itemCount;
    return Badge(
      label: Text('$count'),
      child: const Icon(Icons.shopping_cart),
    );
  }
}

// เลือก rebuild เฉพาะส่วน:
class CartTotal extends StatelessWidget {
  const CartTotal({super.key});

  @override
  Widget build(BuildContext context) {
    // select ช่วยให้ rebuild เฉพาะเมื่อ total เปลี่ยน
    final total = context.select<CartViewModel, double>((vm) => vm.total);
    return Text('รวม: ฿${total.toStringAsFixed(2)}');
  }
}
```

---

## 3. Riverpod - ตัวอย่างสมบูรณ์

```dart
// pubspec.yaml:
// flutter_riverpod: ^2.4.0

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

// ============ Providers ============

// Simple state provider:
final counterProvider = StateProvider<int>((ref) => 0);

// StateNotifier (for complex state):
class CartNotifier extends StateNotifier<List<CartItem>> {
  CartNotifier() : super([]);
  
  void addItem(Product product) {
    final existing = state.indexWhere((i) => i.product.id == product.id);
    if (existing >= 0) {
      state = [
        ...state.sublist(0, existing),
        state[existing].copyWith(quantity: state[existing].quantity + 1),
        ...state.sublist(existing + 1),
      ];
    } else {
      state = [...state, CartItem(product: product, quantity: 1)];
    }
  }
  
  void removeItem(String productId) {
    state = state.where((i) => i.product.id != productId).toList();
  }
  
  void clear() => state = [];
}

final cartProvider = StateNotifierProvider<CartNotifier, List<CartItem>>(
  (ref) => CartNotifier(),
);

// Computed providers:
final cartTotalProvider = Provider<double>((ref) {
  final items = ref.watch(cartProvider);
  return items.fold(0, (sum, item) => sum + item.subtotal);
});

final cartCountProvider = Provider<int>((ref) {
  final items = ref.watch(cartProvider);
  return items.fold(0, (sum, item) => sum + item.quantity);
});

// Async provider (FutureProvider):
final productsProvider = FutureProvider<List<Product>>((ref) async {
  await Future.delayed(const Duration(seconds: 1));
  return [
    const Product(id: '1', name: 'MacBook Pro', price: 89900, imageUrl: ''),
    const Product(id: '2', name: 'iPhone 15', price: 32900, imageUrl: ''),
  ];
});

// ============ Setup ============
void main() {
  runApp(
    const ProviderScope( // ครอบ app ด้วย ProviderScope
      child: MyApp(),
    ),
  );
}

// ============ Usage ============
// ConsumerWidget แทน StatelessWidget
class ProductListPage extends ConsumerWidget {
  const ProductListPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // watch กับ AsyncValue
    final asyncProducts = ref.watch(productsProvider);
    
    return asyncProducts.when(
      loading: () => const Scaffold(
        body: Center(child: CircularProgressIndicator()),
      ),
      error: (error, stack) => Scaffold(
        body: Center(child: Text('Error: $error')),
      ),
      data: (products) => Scaffold(
        appBar: AppBar(
          title: const Text('สินค้า'),
          actions: [
            Consumer(builder: (context, ref, _) {
              final count = ref.watch(cartCountProvider);
              return Badge(
                label: Text('$count'),
                child: const Icon(Icons.shopping_cart),
              );
            }),
          ],
        ),
        body: ListView.builder(
          itemCount: products.length,
          itemBuilder: (context, index) {
            final product = products[index];
            return ListTile(
              title: Text(product.name),
              subtitle: Text('฿${product.price}'),
              trailing: ElevatedButton(
                onPressed: () => ref.read(cartProvider.notifier).addItem(product),
                child: const Text('เพิ่ม'),
              ),
            );
          },
        ),
      ),
    );
  }
}

// ConsumerStatefulWidget เมื่อต้องการ lifecycle
class CartPage extends ConsumerStatefulWidget {
  const CartPage({super.key});

  @override
  ConsumerState<CartPage> createState() => _CartPageState();
}

class _CartPageState extends ConsumerState<CartPage> {
  @override
  Widget build(BuildContext context) {
    final items = ref.watch(cartProvider);
    final total = ref.watch(cartTotalProvider);
    
    return Scaffold(
      appBar: AppBar(title: const Text('ตะกร้าสินค้า')),
      body: Column(
        children: [
          Expanded(
            child: ListView.builder(
              itemCount: items.length,
              itemBuilder: (context, index) {
                final item = items[index];
                return ListTile(
                  title: Text(item.product.name),
                  subtitle: Text('฿${item.subtotal}'),
                  trailing: Row(
                    mainAxisSize: MainAxisSize.min,
                    children: [
                      Text('x${item.quantity}'),
                      IconButton(
                        icon: const Icon(Icons.delete),
                        onPressed: () => ref
                            .read(cartProvider.notifier)
                            .removeItem(item.product.id),
                      ),
                    ],
                  ),
                );
              },
            ),
          ),
          Padding(
            padding: const EdgeInsets.all(16),
            child: Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                Text('รวม: ฿${total.toStringAsFixed(2)}',
                    style: const TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
                ElevatedButton(
                  onPressed: items.isEmpty ? null : () {
                    ref.read(cartProvider.notifier).clear();
                    ScaffoldMessenger.of(context).showSnackBar(
                      const SnackBar(content: Text('ชำระเงินเรียบร้อย!')),
                    );
                  },
                  child: const Text('ชำระเงิน'),
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}
```

---

## 4. BLoC Pattern - ตัวอย่างสมบูรณ์

```dart
// pubspec.yaml:
// flutter_bloc: ^8.1.0

import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:equatable/equatable.dart';

// ============ EVENTS ============
abstract class AuthEvent extends Equatable {
  @override
  List<Object?> get props => [];
}

class AuthLoginRequested extends AuthEvent {
  final String email;
  final String password;
  
  const AuthLoginRequested({required this.email, required this.password});
  
  @override
  List<Object?> get props => [email, password];
}

class AuthLogoutRequested extends AuthEvent {}

class AuthCheckRequested extends AuthEvent {}

// ============ STATES ============
abstract class AuthState extends Equatable {
  @override
  List<Object?> get props => [];
}

class AuthInitial extends AuthState {}

class AuthLoading extends AuthState {}

class AuthAuthenticated extends AuthState {
  final User user;
  
  const AuthAuthenticated(this.user);
  
  @override
  List<Object?> get props => [user];
}

class AuthUnauthenticated extends AuthState {}

class AuthError extends AuthState {
  final String message;
  
  const AuthError(this.message);
  
  @override
  List<Object?> get props => [message];
}

class User {
  final String id;
  final String email;
  final String name;
  
  const User({required this.id, required this.email, required this.name});
}

// ============ BLOC ============
class AuthBloc extends Bloc<AuthEvent, AuthState> {
  AuthBloc() : super(AuthInitial()) {
    on<AuthLoginRequested>(_onLoginRequested);
    on<AuthLogoutRequested>(_onLogoutRequested);
    on<AuthCheckRequested>(_onCheckRequested);
  }
  
  Future<void> _onLoginRequested(
    AuthLoginRequested event,
    Emitter<AuthState> emit,
  ) async {
    emit(AuthLoading());
    
    try {
      await Future.delayed(const Duration(seconds: 2));
      
      if (event.email == 'admin@example.com' && event.password == '123456') {
        emit(AuthAuthenticated(User(
          id: '1',
          email: event.email,
          name: 'Admin User',
        )));
      } else {
        emit(const AuthError('Email หรือ Password ไม่ถูกต้อง'));
      }
    } catch (e) {
      emit(AuthError('เกิดข้อผิดพลาด: $e'));
    }
  }
  
  Future<void> _onLogoutRequested(
    AuthLogoutRequested event,
    Emitter<AuthState> emit,
  ) async {
    await Future.delayed(const Duration(milliseconds: 500));
    emit(AuthUnauthenticated());
  }
  
  Future<void> _onCheckRequested(
    AuthCheckRequested event,
    Emitter<AuthState> emit,
  ) async {
    // ตรวจสอบ stored session
    emit(AuthUnauthenticated());
  }
}

// ============ CUBIT (simpler BLoC) ============
class CounterCubit extends Cubit<int> {
  CounterCubit() : super(0);
  
  void increment() => emit(state + 1);
  void decrement() => emit(state - 1);
  void reset() => emit(0);
}

// ============ USAGE ============
class AuthApp extends StatelessWidget {
  const AuthApp({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (_) => AuthBloc()..add(AuthCheckRequested()),
      child: MaterialApp(
        home: BlocBuilder<AuthBloc, AuthState>(
          builder: (context, state) {
            if (state is AuthAuthenticated) {
              return HomePage(user: state.user);
            }
            return const LoginPage();
          },
        ),
      ),
    );
  }
}

class LoginPage extends StatefulWidget {
  const LoginPage({super.key});

  @override
  State<LoginPage> createState() => _LoginPageState();
}

class _LoginPageState extends State<LoginPage> {
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('เข้าสู่ระบบ')),
      body: BlocConsumer<AuthBloc, AuthState>(
        listener: (context, state) {
          if (state is AuthError) {
            ScaffoldMessenger.of(context).showSnackBar(
              SnackBar(
                content: Text(state.message),
                backgroundColor: Colors.red,
              ),
            );
          }
        },
        builder: (context, state) {
          return Padding(
            padding: const EdgeInsets.all(16),
            child: Column(
              children: [
                TextField(
                  controller: _emailController,
                  decoration: const InputDecoration(labelText: 'Email'),
                ),
                TextField(
                  controller: _passwordController,
                  decoration: const InputDecoration(labelText: 'Password'),
                  obscureText: true,
                ),
                const SizedBox(height: 24),
                if (state is AuthLoading)
                  const CircularProgressIndicator()
                else
                  ElevatedButton(
                    onPressed: () {
                      context.read<AuthBloc>().add(
                        AuthLoginRequested(
                          email: _emailController.text,
                          password: _passwordController.text,
                        ),
                      );
                    },
                    child: const Text('เข้าสู่ระบบ'),
                  ),
              ],
            ),
          );
        },
      ),
    );
  }
}

class HomePage extends StatelessWidget {
  final User user;
  const HomePage({super.key, required this.user});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('สวัสดี ${user.name}!'),
        actions: [
          IconButton(
            icon: const Icon(Icons.logout),
            onPressed: () {
              context.read<AuthBloc>().add(AuthLogoutRequested());
            },
          ),
        ],
      ),
      body: Center(child: Text('ยินดีต้อนรับ ${user.email}')),
    );
  }
}
```

---

## 5. สรุปการเลือก State Management

```dart
// Decision tree:
//
// แอปเล็ก, ง่าย?
//   └─> setState ก็พอ
//
// ต้องการ reactive + ง่าย + ขนาดกลาง?
//   └─> Provider (ChangeNotifier)
//
// ต้องการ type-safety + testability สูง?
//   └─> Riverpod (StateNotifier/AsyncNotifier)
//
// ขนาดใหญ่, team ใหญ่, events เยอะ?
//   └─> BLoC
//
// อยากได้ทุกอย่างแบบ all-in-one ตั้งแต่ routing?
//   └─> GetX (ระวัง: opinionated มาก)

// ตัวเลขเปรียบเทียบ (approximate):
// Provider:  ⭐⭐⭐⭐ Simple, official backing
// Riverpod:  ⭐⭐⭐⭐⭐ Best type safety, most flexible
// BLoC:      ⭐⭐⭐⭐ Best for large teams
// GetX:      ⭐⭐⭐ Fast to write, hard to test
```

---

## ➡️ Part ถัดไป
**Part 93: Testing Strategies - Unit, Widget, Integration, E2E**
