# Part 91: App Architecture Patterns - MVC, MVP, MVVM

## 🎯 เป้าหมายของ Part นี้
- เข้าใจ MVC, MVP, MVVM patterns
- เปรียบเทียบข้อดีข้อเสียของแต่ละ pattern
- ใช้ MVVM กับ Flutter อย่างเหมาะสม
- Clean separation of concerns
- Testability ของแต่ละ pattern

---

## 1. MVC (Model-View-Controller)

```
┌──────────────────────────────────────────┐
│                  View                    │
│  (Flutter Widgets - UI only)             │
└──────────────┬───────────────────────────┘
               │ User Actions
               ▼
┌──────────────────────────────────────────┐
│              Controller                  │
│  (Business logic, coordinates M and V)  │
└──────────────┬───────────────────────────┘
               │ Data
               ▼
┌──────────────────────────────────────────┐
│                 Model                    │
│  (Data + Business Rules)                 │
└──────────────────────────────────────────┘
```

### ตัวอย่าง MVC ใน Flutter:

```dart
// ============ MODEL ============
class ProductModel {
  final String id;
  final String name;
  final double price;
  final int stock;
  
  const ProductModel({
    required this.id,
    required this.name,
    required this.price,
    required this.stock,
  });
  
  bool get isAvailable => stock > 0;
  
  ProductModel copyWith({
    String? name,
    double? price,
    int? stock,
  }) {
    return ProductModel(
      id: id,
      name: name ?? this.name,
      price: price ?? this.price,
      stock: stock ?? this.stock,
    );
  }
  
  factory ProductModel.fromJson(Map<String, dynamic> json) {
    return ProductModel(
      id: json['id'] as String,
      name: json['name'] as String,
      price: (json['price'] as num).toDouble(),
      stock: json['stock'] as int,
    );
  }
  
  Map<String, dynamic> toJson() => {
    'id': id,
    'name': name,
    'price': price,
    'stock': stock,
  };
}

// ============ CONTROLLER ============
class ProductController {
  List<ProductModel> _products = [];
  List<ProductModel> get products => List.unmodifiable(_products);
  
  // Callbacks สำหรับ notify View
  Function()? onProductsChanged;
  Function(String)? onError;
  
  Future<void> loadProducts() async {
    try {
      // จำลอง API call
      await Future.delayed(const Duration(seconds: 1));
      _products = [
        const ProductModel(id: '1', name: 'MacBook Pro', price: 89900, stock: 5),
        const ProductModel(id: '2', name: 'iPhone 15', price: 32900, stock: 0),
        const ProductModel(id: '3', name: 'iPad Air', price: 24900, stock: 10),
      ];
      onProductsChanged?.call();
    } catch (e) {
      onError?.call('โหลดสินค้าไม่สำเร็จ: $e');
    }
  }
  
  List<ProductModel> getAvailableProducts() {
    return _products.where((p) => p.isAvailable).toList();
  }
  
  void addProduct(ProductModel product) {
    _products = [..._products, product];
    onProductsChanged?.call();
  }
  
  void removeProduct(String id) {
    _products = _products.where((p) => p.id != id).toList();
    onProductsChanged?.call();
  }
}

// ============ VIEW ============
import 'package:flutter/material.dart';

class ProductListView extends StatefulWidget {
  const ProductListView({super.key});

  @override
  State<ProductListView> createState() => _ProductListViewState();
}

class _ProductListViewState extends State<ProductListView> {
  final ProductController _controller = ProductController();
  bool _isLoading = false;
  String? _error;

  @override
  void initState() {
    super.initState();
    _controller.onProductsChanged = () {
      if (mounted) setState(() {});
    };
    _controller.onError = (error) {
      if (mounted) setState(() => _error = error);
    };
    _loadData();
  }

  Future<void> _loadData() async {
    setState(() => _isLoading = true);
    await _controller.loadProducts();
    if (mounted) setState(() => _isLoading = false);
  }

  @override
  Widget build(BuildContext context) {
    if (_isLoading) {
      return const Scaffold(
        body: Center(child: CircularProgressIndicator()),
      );
    }
    
    if (_error != null) {
      return Scaffold(
        body: Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              Text(_error!, style: const TextStyle(color: Colors.red)),
              ElevatedButton(
                onPressed: _loadData,
                child: const Text('ลองใหม่'),
              ),
            ],
          ),
        ),
      );
    }
    
    return Scaffold(
      appBar: AppBar(title: const Text('สินค้า')),
      body: ListView.builder(
        itemCount: _controller.products.length,
        itemBuilder: (context, index) {
          final product = _controller.products[index];
          return ListTile(
            title: Text(product.name),
            subtitle: Text('฿${product.price.toStringAsFixed(0)}'),
            trailing: product.isAvailable
                ? const Icon(Icons.check_circle, color: Colors.green)
                : const Icon(Icons.cancel, color: Colors.red),
          );
        },
      ),
    );
  }
}
```

---

## 2. MVP (Model-View-Presenter)

```
┌──────────────────────────────────────────┐
│                  View                    │
│  (Passive UI - just renders and reports) │
└──────────────┬───────────────────────────┘
               │ 2-way communication
               │ via interface
               ▼
┌──────────────────────────────────────────┐
│              Presenter                   │
│  (Logic, transforms Model for View)      │
└──────────────┬───────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────┐
│                 Model                    │
│  (Data sources, repositories)            │
└──────────────────────────────────────────┘
```

```dart
// ============ Contract (interfaces) ============
abstract class ILoginView {
  void showLoading();
  void hideLoading();
  void showError(String message);
  void navigateToHome();
}

abstract class ILoginPresenter {
  void login(String email, String password);
  void dispose();
}

// ============ MODEL / Repository ============
class AuthRepository {
  Future<bool> login(String email, String password) async {
    await Future.delayed(const Duration(seconds: 2));
    // จำลอง validation
    return email.isNotEmpty && password.length >= 6;
  }
}

// ============ PRESENTER ============
class LoginPresenter implements ILoginPresenter {
  final ILoginView _view;
  final AuthRepository _repository;
  
  LoginPresenter(this._view, this._repository);
  
  @override
  void login(String email, String password) async {
    // Validate input
    if (email.isEmpty || !email.contains('@')) {
      _view.showError('Email ไม่ถูกต้อง');
      return;
    }
    
    if (password.length < 6) {
      _view.showError('Password ต้องมีอย่างน้อย 6 ตัวอักษร');
      return;
    }
    
    _view.showLoading();
    
    try {
      bool success = await _repository.login(email, password);
      _view.hideLoading();
      
      if (success) {
        _view.navigateToHome();
      } else {
        _view.showError('Email หรือ Password ไม่ถูกต้อง');
      }
    } catch (e) {
      _view.hideLoading();
      _view.showError('เกิดข้อผิดพลาด: $e');
    }
  }
  
  @override
  void dispose() {
    // clean up resources
  }
}

// ============ VIEW ============
class LoginPage extends StatefulWidget implements ILoginView {
  const LoginPage({super.key});

  @override
  State<LoginPage> createState() => _LoginPageState();
}

class _LoginPageState extends State<LoginPage> implements ILoginView {
  late final LoginPresenter _presenter;
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();
  bool _isLoading = false;
  String? _errorMessage;

  @override
  void initState() {
    super.initState();
    _presenter = LoginPresenter(this, AuthRepository());
  }

  @override
  void showLoading() {
    if (mounted) setState(() => _isLoading = true);
  }

  @override
  void hideLoading() {
    if (mounted) setState(() => _isLoading = false);
  }

  @override
  void showError(String message) {
    if (mounted) setState(() => _errorMessage = message);
  }

  @override
  void navigateToHome() {
    if (mounted) {
      Navigator.of(context).pushReplacementNamed('/home');
    }
  }

  @override
  void dispose() {
    _presenter.dispose();
    _emailController.dispose();
    _passwordController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('เข้าสู่ระบบ')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            if (_errorMessage != null)
              Container(
                padding: const EdgeInsets.all(8),
                color: Colors.red.shade100,
                child: Text(_errorMessage!, style: const TextStyle(color: Colors.red)),
              ),
            TextField(
              controller: _emailController,
              decoration: const InputDecoration(labelText: 'Email'),
            ),
            TextField(
              controller: _passwordController,
              decoration: const InputDecoration(labelText: 'Password'),
              obscureText: true,
            ),
            const SizedBox(height: 16),
            _isLoading
                ? const CircularProgressIndicator()
                : ElevatedButton(
                    onPressed: () => _presenter.login(
                      _emailController.text,
                      _passwordController.text,
                    ),
                    child: const Text('เข้าสู่ระบบ'),
                  ),
          ],
        ),
      ),
    );
  }
}
```

---

## 3. MVVM (Model-View-ViewModel) - แนะนำสำหรับ Flutter

```
┌──────────────────────────────────────────┐
│                  View                    │
│  (Flutter Widgets - observes ViewModel)  │
└──────────────┬───────────────────────────┘
               │ data binding / listening
               ▼
┌──────────────────────────────────────────┐
│              ViewModel                   │
│  (State + Logic - exposes observables)   │
└──────────────┬───────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────┐
│               Repository                 │
└──────────────┬───────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────┐
│           Data Sources                   │
│  (API, Database, Local Storage)          │
└──────────────────────────────────────────┘
```

### MVVM ด้วย ChangeNotifier + Provider:

```dart
// ============ MODEL ============
class Todo {
  final String id;
  final String title;
  final bool isCompleted;
  final DateTime createdAt;
  
  const Todo({
    required this.id,
    required this.title,
    this.isCompleted = false,
    required this.createdAt,
  });
  
  Todo copyWith({
    String? title,
    bool? isCompleted,
  }) {
    return Todo(
      id: id,
      title: title ?? this.title,
      isCompleted: isCompleted ?? this.isCompleted,
      createdAt: createdAt,
    );
  }
}

// ============ REPOSITORY ============
abstract class TodoRepository {
  Future<List<Todo>> getAll();
  Future<void> add(Todo todo);
  Future<void> update(Todo todo);
  Future<void> delete(String id);
}

class LocalTodoRepository implements TodoRepository {
  final List<Todo> _storage = []; // in-memory for demo
  
  @override
  Future<List<Todo>> getAll() async {
    await Future.delayed(const Duration(milliseconds: 300));
    return List.from(_storage);
  }
  
  @override
  Future<void> add(Todo todo) async {
    _storage.add(todo);
  }
  
  @override
  Future<void> update(Todo todo) async {
    final index = _storage.indexWhere((t) => t.id == todo.id);
    if (index >= 0) _storage[index] = todo;
  }
  
  @override
  Future<void> delete(String id) async {
    _storage.removeWhere((t) => t.id == id);
  }
}

// ============ VIEWMODEL ============
class TodoViewModel extends ChangeNotifier {
  final TodoRepository _repository;
  
  List<Todo> _todos = [];
  bool _isLoading = false;
  String? _error;
  String _filter = 'all'; // all, active, completed
  
  List<Todo> get todos {
    switch (_filter) {
      case 'active':
        return _todos.where((t) => !t.isCompleted).toList();
      case 'completed':
        return _todos.where((t) => t.isCompleted).toList();
      default:
        return List.from(_todos);
    }
  }
  
  bool get isLoading => _isLoading;
  String? get error => _error;
  String get filter => _filter;
  
  int get totalCount => _todos.length;
  int get completedCount => _todos.where((t) => t.isCompleted).length;
  int get activeCount => _todos.where((t) => !t.isCompleted).length;
  
  TodoViewModel(this._repository) {
    loadTodos();
  }
  
  Future<void> loadTodos() async {
    _setLoading(true);
    try {
      _todos = await _repository.getAll();
      _error = null;
    } catch (e) {
      _error = 'โหลดข้อมูลไม่สำเร็จ: $e';
    } finally {
      _setLoading(false);
    }
  }
  
  Future<void> addTodo(String title) async {
    if (title.trim().isEmpty) return;
    
    final todo = Todo(
      id: DateTime.now().millisecondsSinceEpoch.toString(),
      title: title.trim(),
      createdAt: DateTime.now(),
    );
    
    await _repository.add(todo);
    _todos.add(todo);
    notifyListeners();
  }
  
  Future<void> toggleTodo(String id) async {
    final index = _todos.indexWhere((t) => t.id == id);
    if (index < 0) return;
    
    final updated = _todos[index].copyWith(
      isCompleted: !_todos[index].isCompleted,
    );
    
    await _repository.update(updated);
    _todos[index] = updated;
    notifyListeners();
  }
  
  Future<void> deleteTodo(String id) async {
    await _repository.delete(id);
    _todos.removeWhere((t) => t.id == id);
    notifyListeners();
  }
  
  void setFilter(String filter) {
    _filter = filter;
    notifyListeners();
  }
  
  void _setLoading(bool loading) {
    _isLoading = loading;
    notifyListeners();
  }
}

// ============ VIEW ============
import 'package:provider/provider.dart';

class TodoApp extends StatelessWidget {
  const TodoApp({super.key});

  @override
  Widget build(BuildContext context) {
    return ChangeNotifierProvider(
      create: (_) => TodoViewModel(LocalTodoRepository()),
      child: MaterialApp(
        title: 'Todo MVVM',
        home: const TodoPage(),
      ),
    );
  }
}

class TodoPage extends StatelessWidget {
  const TodoPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Todo App (MVVM)'),
        actions: [
          Consumer<TodoViewModel>(
            builder: (context, vm, _) => Badge(
              label: Text('${vm.activeCount}'),
              child: const Icon(Icons.list_alt),
            ),
          ),
        ],
      ),
      body: Column(
        children: [
          const _FilterBar(),
          const _AddTodoField(),
          const Expanded(child: _TodoList()),
          const _StatusBar(),
        ],
      ),
    );
  }
}

class _FilterBar extends StatelessWidget {
  const _FilterBar();

  @override
  Widget build(BuildContext context) {
    final vm = context.watch<TodoViewModel>();
    
    return SingleChildScrollView(
      scrollDirection: Axis.horizontal,
      padding: const EdgeInsets.all(8),
      child: Row(
        children: ['all', 'active', 'completed'].map((filter) {
          final label = switch (filter) {
            'all' => 'ทั้งหมด',
            'active' => 'ยังไม่เสร็จ',
            'completed' => 'เสร็จแล้ว',
            _ => filter,
          };
          return Padding(
            padding: const EdgeInsets.only(right: 8),
            child: FilterChip(
              label: Text(label),
              selected: vm.filter == filter,
              onSelected: (_) => vm.setFilter(filter),
            ),
          );
        }).toList(),
      ),
    );
  }
}

class _AddTodoField extends StatefulWidget {
  const _AddTodoField();

  @override
  State<_AddTodoField> createState() => _AddTodoFieldState();
}

class _AddTodoFieldState extends State<_AddTodoField> {
  final _controller = TextEditingController();

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
      child: Row(
        children: [
          Expanded(
            child: TextField(
              controller: _controller,
              decoration: const InputDecoration(
                hintText: 'เพิ่มงาน...',
                border: OutlineInputBorder(),
              ),
              onSubmitted: (_) => _addTodo(context),
            ),
          ),
          const SizedBox(width: 8),
          FloatingActionButton.small(
            onPressed: () => _addTodo(context),
            child: const Icon(Icons.add),
          ),
        ],
      ),
    );
  }
  
  void _addTodo(BuildContext context) {
    if (_controller.text.isNotEmpty) {
      context.read<TodoViewModel>().addTodo(_controller.text);
      _controller.clear();
    }
  }
}

class _TodoList extends StatelessWidget {
  const _TodoList();

  @override
  Widget build(BuildContext context) {
    final vm = context.watch<TodoViewModel>();
    
    if (vm.isLoading) {
      return const Center(child: CircularProgressIndicator());
    }
    
    if (vm.error != null) {
      return Center(child: Text(vm.error!));
    }
    
    if (vm.todos.isEmpty) {
      return const Center(child: Text('ไม่มีงาน'));
    }
    
    return ListView.builder(
      itemCount: vm.todos.length,
      itemBuilder: (context, index) {
        final todo = vm.todos[index];
        return Dismissible(
          key: Key(todo.id),
          onDismissed: (_) => vm.deleteTodo(todo.id),
          background: Container(
            color: Colors.red,
            alignment: Alignment.centerRight,
            padding: const EdgeInsets.only(right: 16),
            child: const Icon(Icons.delete, color: Colors.white),
          ),
          child: CheckboxListTile(
            title: Text(
              todo.title,
              style: TextStyle(
                decoration: todo.isCompleted 
                    ? TextDecoration.lineThrough 
                    : null,
                color: todo.isCompleted ? Colors.grey : null,
              ),
            ),
            value: todo.isCompleted,
            onChanged: (_) => vm.toggleTodo(todo.id),
          ),
        );
      },
    );
  }
}

class _StatusBar extends StatelessWidget {
  const _StatusBar();

  @override
  Widget build(BuildContext context) {
    final vm = context.watch<TodoViewModel>();
    
    return Container(
      padding: const EdgeInsets.all(8),
      color: Colors.grey.shade100,
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceBetween,
        children: [
          Text('ทั้งหมด: ${vm.totalCount}'),
          Text('เสร็จ: ${vm.completedCount}'),
          Text('ค้างอยู่: ${vm.activeCount}'),
        ],
      ),
    );
  }
}
```

---

## 4. เปรียบเทียบ Patterns

| ด้าน | MVC | MVP | MVVM |
|-----|-----|-----|------|
| **ความยาก** | ง่าย | กลาง | กลาง-สูง |
| **Testability** | ยาก | ง่าย | ง่ายมาก |
| **Flutter fit** | ดีพอ | ดี | ดีมาก |
| **Boilerplate** | น้อย | กลาง | กลาง |
| **Data binding** | Manual | Manual | Reactive |
| **ขนาดโปรเจค** | เล็ก-กลาง | กลาง | กลาง-ใหญ่ |

### เมื่อไหร่ใช้อะไร:
```
MVC  → แอปเล็ก, prototype, เรียนรู้เบื้องต้น
MVP  → แอปที่ต้องการ testability สูง
MVVM → แอปขนาดกลาง-ใหญ่ที่ต้องการ reactive UI
```

---

## 5. สรุป Part 91

สิ่งที่เรียนรู้:
- ✅ MVC pattern และการใช้งานใน Flutter
- ✅ MVP pattern พร้อม interfaces
- ✅ MVVM pattern ด้วย ChangeNotifier + Provider
- ✅ เปรียบเทียบข้อดีข้อเสียของแต่ละ pattern
- ✅ Best practices สำหรับ Flutter architecture

---

## ➡️ Part ถัดไป
**Part 92: State Management Comparison - Provider vs Riverpod vs BLoC**
