# Part 34: BLoC Pattern

## BLoC คืออะไร

BLoC (Business Logic Component) เป็น pattern สำหรับแยก business logic ออกจาก UI โดยใช้ Streams เป็นหัวใจสำคัญ

```yaml
# pubspec.yaml
dependencies:
  flutter_bloc: ^8.1.3
  equatable: ^2.0.5
```

### ทำความเข้าใจ BLoC Pattern

```
UI (Widget) --[Events]--> BLoC --[States]--> UI (Widget)
```

- **Events**: สิ่งที่เกิดขึ้นใน UI (button press, form submit, etc.)
- **BLoC**: รับ Events, ประมวลผล, ส่ง States กลับ
- **States**: ข้อมูลที่ UI ใช้แสดงผล

---

## Events และ States

### การออกแบบ Events

```dart
// events/counter_event.dart
import 'package:equatable/equatable.dart';

// Base event class
abstract class CounterEvent extends Equatable {
  const CounterEvent();
  
  @override
  List<Object?> get props => [];
}

// Concrete events
class CounterIncrementPressed extends CounterEvent {
  const CounterIncrementPressed();
}

class CounterDecrementPressed extends CounterEvent {
  const CounterDecrementPressed();
}

class CounterResetPressed extends CounterEvent {
  const CounterResetPressed();
}

class CounterStepChanged extends CounterEvent {
  final int step;
  const CounterStepChanged(this.step);
  
  @override
  List<Object?> get props => [step];
}
```

### การออกแบบ States

```dart
// states/counter_state.dart
import 'package:equatable/equatable.dart';

// Simple state - เพียงเก็บค่า
class CounterState extends Equatable {
  final int count;
  final int step;
  final String status;  // 'initial', 'loading', 'success', 'error'
  
  const CounterState({
    this.count = 0,
    this.step = 1,
    this.status = 'initial',
  });
  
  CounterState copyWith({
    int? count,
    int? step,
    String? status,
  }) {
    return CounterState(
      count: count ?? this.count,
      step: step ?? this.step,
      status: status ?? this.status,
    );
  }
  
  @override
  List<Object?> get props => [count, step, status];
}

// หรือใช้ Sealed class pattern (Dart 3+)
sealed class AuthState extends Equatable {
  const AuthState();
  @override
  List<Object?> get props => [];
}

class AuthInitial extends AuthState {
  const AuthInitial();
}

class AuthLoading extends AuthState {
  const AuthLoading();
}

class AuthAuthenticated extends AuthState {
  final String userId;
  final String email;
  const AuthAuthenticated({required this.userId, required this.email});
  
  @override
  List<Object?> get props => [userId, email];
}

class AuthUnauthenticated extends AuthState {
  const AuthUnauthenticated();
}

class AuthError extends AuthState {
  final String message;
  const AuthError(this.message);
  
  @override
  List<Object?> get props => [message];
}
```

---

## BLoC Implementation

### Simple Counter BLoC

```dart
// blocs/counter_bloc.dart
import 'package:flutter_bloc/flutter_bloc.dart';

class CounterBloc extends Bloc<CounterEvent, CounterState> {
  CounterBloc() : super(const CounterState()) {
    // ลงทะเบียน event handlers
    on<CounterIncrementPressed>(_onIncrement);
    on<CounterDecrementPressed>(_onDecrement);
    on<CounterResetPressed>(_onReset);
    on<CounterStepChanged>(_onStepChanged);
  }
  
  void _onIncrement(
    CounterIncrementPressed event,
    Emitter<CounterState> emit,
  ) {
    emit(state.copyWith(count: state.count + state.step));
  }
  
  void _onDecrement(
    CounterDecrementPressed event,
    Emitter<CounterState> emit,
  ) {
    if (state.count - state.step >= 0) {
      emit(state.copyWith(count: state.count - state.step));
    }
  }
  
  void _onReset(
    CounterResetPressed event,
    Emitter<CounterState> emit,
  ) {
    emit(const CounterState());
  }
  
  void _onStepChanged(
    CounterStepChanged event,
    Emitter<CounterState> emit,
  ) {
    emit(state.copyWith(step: event.step));
  }
}
```

### Async BLoC

```dart
// models/post.dart
class Post {
  final int id;
  final int userId;
  final String title;
  final String body;
  
  const Post({
    required this.id,
    required this.userId,
    required this.title,
    required this.body,
  });
  
  factory Post.fromJson(Map<String, dynamic> json) {
    return Post(
      id: json['id'],
      userId: json['userId'],
      title: json['title'],
      body: json['body'],
    );
  }
}

// events/post_event.dart
abstract class PostEvent extends Equatable {
  const PostEvent();
  @override
  List<Object?> get props => [];
}

class PostFetched extends PostEvent {
  const PostFetched();
}

class PostRefreshed extends PostEvent {
  const PostRefreshed();
}

// states/post_state.dart
enum PostStatus { initial, loading, success, failure }

class PostState extends Equatable {
  final PostStatus status;
  final List<Post> posts;
  final String errorMessage;
  
  const PostState({
    this.status = PostStatus.initial,
    this.posts = const [],
    this.errorMessage = '',
  });
  
  PostState copyWith({
    PostStatus? status,
    List<Post>? posts,
    String? errorMessage,
  }) {
    return PostState(
      status: status ?? this.status,
      posts: posts ?? this.posts,
      errorMessage: errorMessage ?? this.errorMessage,
    );
  }
  
  @override
  List<Object?> get props => [status, posts, errorMessage];
}

// blocs/post_bloc.dart
class PostBloc extends Bloc<PostEvent, PostState> {
  PostBloc() : super(const PostState()) {
    on<PostFetched>(_onPostFetched);
    on<PostRefreshed>(_onPostRefreshed);
  }
  
  Future<void> _onPostFetched(
    PostFetched event,
    Emitter<PostState> emit,
  ) async {
    if (state.status == PostStatus.loading) return;
    
    emit(state.copyWith(status: PostStatus.loading));
    
    try {
      final posts = await _fetchPosts();
      emit(state.copyWith(
        status: PostStatus.success,
        posts: posts,
      ));
    } catch (e) {
      emit(state.copyWith(
        status: PostStatus.failure,
        errorMessage: e.toString(),
      ));
    }
  }
  
  Future<void> _onPostRefreshed(
    PostRefreshed event,
    Emitter<PostState> emit,
  ) async {
    emit(const PostState());
    add(const PostFetched());
  }
  
  Future<List<Post>> _fetchPosts() async {
    // จำลองการดึงข้อมูล
    await Future.delayed(const Duration(seconds: 2));
    return [
      const Post(id: 1, userId: 1, title: 'Post 1', body: 'Content of post 1'),
      const Post(id: 2, userId: 1, title: 'Post 2', body: 'Content of post 2'),
      const Post(id: 3, userId: 2, title: 'Post 3', body: 'Content of post 3'),
    ];
  }
}
```

---

## BlocProvider, BlocBuilder, BlocListener

### BlocProvider

```dart
// main.dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'BLoC Demo',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue),
        useMaterial3: true,
      ),
      // BlocProvider ที่ root ให้ทั้ง app access ได้
      home: BlocProvider(
        create: (context) => CounterBloc(),
        child: const CounterPage(),
      ),
    );
  }
}

// หรือใช้ MultiBlocProvider สำหรับหลาย BLoC
class AppWithMultipleBlocs extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MultiBlocProvider(
      providers: [
        BlocProvider(create: (_) => CounterBloc()),
        BlocProvider(create: (_) => PostBloc()),
        // BlocProvider ที่ขึ้นกับ BLoC อื่น
        BlocProvider(
          create: (context) {
            return PostBloc();
          },
        ),
      ],
      child: const MyApp(),
    );
  }
}
```

### BlocBuilder

```dart
// BlocBuilder: rebuild widget เมื่อ state เปลี่ยน
class CounterPage extends StatelessWidget {
  const CounterPage({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Counter BLoC'),
        actions: [
          IconButton(
            onPressed: () => context.read<CounterBloc>().add(const CounterResetPressed()),
            icon: const Icon(Icons.refresh),
          ),
        ],
      ),
      body: BlocBuilder<CounterBloc, CounterState>(
        // buildWhen: กำหนดว่าควร rebuild เมื่อไหร่
        buildWhen: (previous, current) {
          return previous.count != current.count || previous.step != current.step;
        },
        builder: (context, state) {
          return Center(
            child: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                Text(
                  '${state.count}',
                  style: const TextStyle(fontSize: 72, fontWeight: FontWeight.bold),
                ),
                Text('Step: ${state.step}'),
                const SizedBox(height: 20),
                Slider(
                  min: 1,
                  max: 10,
                  divisions: 9,
                  value: state.step.toDouble(),
                  label: '${state.step}',
                  onChanged: (value) => context
                      .read<CounterBloc>()
                      .add(CounterStepChanged(value.round())),
                ),
                Row(
                  mainAxisAlignment: MainAxisAlignment.center,
                  children: [
                    FloatingActionButton(
                      onPressed: () => context
                          .read<CounterBloc>()
                          .add(const CounterDecrementPressed()),
                      mini: true,
                      child: const Icon(Icons.remove),
                    ),
                    const SizedBox(width: 24),
                    FloatingActionButton(
                      onPressed: () => context
                          .read<CounterBloc>()
                          .add(const CounterIncrementPressed()),
                      child: const Icon(Icons.add),
                    ),
                  ],
                ),
              ],
            ),
          );
        },
      ),
    );
  }
}
```

### BlocListener

```dart
// BlocListener: ทำ side effects เมื่อ state เปลี่ยน (ไม่ rebuild UI)
class CounterPageWithListener extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return BlocListener<CounterBloc, CounterState>(
      listenWhen: (previous, current) {
        // ฟังเฉพาะเมื่อ count เป็นทวีคูณของ 5
        return current.count > 0 && current.count % 5 == 0;
      },
      listener: (context, state) {
        // side effect: แสดง snackbar
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(
            content: Text('Reached ${state.count}!'),
            backgroundColor: Colors.green,
          ),
        );
      },
      child: BlocBuilder<CounterBloc, CounterState>(
        builder: (context, state) {
          return Text('${state.count}');
        },
      ),
    );
  }
}

// BlocConsumer: รวม BlocBuilder + BlocListener
class CounterPageWithConsumer extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return BlocConsumer<CounterBloc, CounterState>(
      listenWhen: (previous, current) => current.count % 10 == 0 && current.count > 0,
      listener: (context, state) {
        showDialog(
          context: context,
          builder: (ctx) => AlertDialog(
            title: const Text('Milestone!'),
            content: Text('You reached ${state.count}!'),
            actions: [
              TextButton(
                onPressed: () => Navigator.pop(ctx),
                child: const Text('OK'),
              ),
            ],
          ),
        );
      },
      builder: (context, state) {
        return Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text('${state.count}', style: const TextStyle(fontSize: 64)),
            ElevatedButton(
              onPressed: () => context.read<CounterBloc>().add(const CounterIncrementPressed()),
              child: const Text('Increment'),
            ),
          ],
        );
      },
    );
  }
}
```

---

## Cubit (simpler BLoC)

### Cubit คืออะไร

Cubit เป็น simplified version ของ BLoC ไม่มี Events แค่เรียก methods โดยตรง

```dart
// cubit/counter_cubit.dart
class CounterCubit extends Cubit<int> {
  CounterCubit() : super(0);
  
  void increment() => emit(state + 1);
  void decrement() {
    if (state > 0) emit(state - 1);
  }
  void reset() => emit(0);
}

// ใช้งาน
class CubitPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (_) => CounterCubit(),
      child: BlocBuilder<CounterCubit, int>(
        builder: (context, count) {
          return Column(
            children: [
              Text('$count'),
              ElevatedButton(
                // เรียก method โดยตรง ไม่ต้อง add event
                onPressed: () => context.read<CounterCubit>().increment(),
                child: const Text('Increment'),
              ),
            ],
          );
        },
      ),
    );
  }
}
```

### Cubit กับ Complex State

```dart
// states
class ShoppingCartState extends Equatable {
  final List<CartItem> items;
  final bool isLoading;
  final String? errorMessage;
  
  const ShoppingCartState({
    this.items = const [],
    this.isLoading = false,
    this.errorMessage,
  });
  
  double get total => items.fold(
    0.0,
    (sum, item) => sum + (item.price * item.quantity),
  );
  
  int get itemCount => items.fold(0, (sum, item) => sum + item.quantity);
  
  ShoppingCartState copyWith({
    List<CartItem>? items,
    bool? isLoading,
    String? errorMessage,
  }) {
    return ShoppingCartState(
      items: items ?? this.items,
      isLoading: isLoading ?? this.isLoading,
      errorMessage: errorMessage,
    );
  }
  
  @override
  List<Object?> get props => [items, isLoading, errorMessage];
}

// Cubit
class ShoppingCartCubit extends Cubit<ShoppingCartState> {
  ShoppingCartCubit() : super(const ShoppingCartState());
  
  void addItem(CartItem newItem) {
    final currentItems = List<CartItem>.from(state.items);
    final index = currentItems.indexWhere((item) => item.id == newItem.id);
    
    if (index >= 0) {
      currentItems[index] = currentItems[index].copyWith(
        quantity: currentItems[index].quantity + 1,
      );
    } else {
      currentItems.add(newItem);
    }
    
    emit(state.copyWith(items: currentItems));
  }
  
  void removeItem(String itemId) {
    final newItems = state.items.where((item) => item.id != itemId).toList();
    emit(state.copyWith(items: newItems));
  }
  
  void updateQuantity(String itemId, int quantity) {
    if (quantity <= 0) {
      removeItem(itemId);
      return;
    }
    
    final newItems = state.items.map((item) {
      if (item.id == itemId) {
        return item.copyWith(quantity: quantity);
      }
      return item;
    }).toList();
    
    emit(state.copyWith(items: newItems));
  }
  
  void clear() {
    emit(const ShoppingCartState());
  }
  
  Future<void> checkout() async {
    emit(state.copyWith(isLoading: true));
    try {
      await Future.delayed(const Duration(seconds: 2));
      emit(const ShoppingCartState());
      // สำเร็จ
    } catch (e) {
      emit(state.copyWith(
        isLoading: false,
        errorMessage: e.toString(),
      ));
    }
  }
}

class CartItem extends Equatable {
  final String id;
  final String name;
  final double price;
  final int quantity;
  
  const CartItem({
    required this.id,
    required this.name,
    required this.price,
    this.quantity = 1,
  });
  
  CartItem copyWith({
    String? id,
    String? name,
    double? price,
    int? quantity,
  }) {
    return CartItem(
      id: id ?? this.id,
      name: name ?? this.name,
      price: price ?? this.price,
      quantity: quantity ?? this.quantity,
    );
  }
  
  @override
  List<Object?> get props => [id, name, price, quantity];
}
```

---

## Workshop: Counter App with BLoC

### Full Counter App

```dart
// main.dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:equatable/equatable.dart';

// ===== EVENTS =====
abstract class CounterEvent extends Equatable {
  const CounterEvent();
  @override
  List<Object?> get props => [];
}

class IncrementEvent extends CounterEvent {
  const IncrementEvent();
}

class DecrementEvent extends CounterEvent {
  const DecrementEvent();
}

class ResetEvent extends CounterEvent {
  const ResetEvent();
}

class SetStepEvent extends CounterEvent {
  final int step;
  const SetStepEvent(this.step);
  @override
  List<Object?> get props => [step];
}

// ===== STATE =====
class CounterBlocState extends Equatable {
  final int count;
  final int step;
  final List<String> log;
  
  const CounterBlocState({
    this.count = 0,
    this.step = 1,
    this.log = const [],
  });
  
  bool get canDecrement => count - step >= 0;
  
  CounterBlocState copyWith({
    int? count,
    int? step,
    List<String>? log,
  }) {
    return CounterBlocState(
      count: count ?? this.count,
      step: step ?? this.step,
      log: log ?? this.log,
    );
  }
  
  @override
  List<Object?> get props => [count, step, log];
}

// ===== BLOC =====
class CounterBlocMain extends Bloc<CounterEvent, CounterBlocState> {
  CounterBlocMain() : super(const CounterBlocState()) {
    on<IncrementEvent>(_handleIncrement);
    on<DecrementEvent>(_handleDecrement);
    on<ResetEvent>(_handleReset);
    on<SetStepEvent>(_handleSetStep);
  }
  
  void _handleIncrement(IncrementEvent event, Emitter<CounterBlocState> emit) {
    final newCount = state.count + state.step;
    emit(state.copyWith(
      count: newCount,
      log: [...state.log, '+${state.step} = $newCount'],
    ));
  }
  
  void _handleDecrement(DecrementEvent event, Emitter<CounterBlocState> emit) {
    if (!state.canDecrement) return;
    final newCount = state.count - state.step;
    emit(state.copyWith(
      count: newCount,
      log: [...state.log, '-${state.step} = $newCount'],
    ));
  }
  
  void _handleReset(ResetEvent event, Emitter<CounterBlocState> emit) {
    emit(state.copyWith(count: 0, log: [...state.log, 'Reset to 0']));
  }
  
  void _handleSetStep(SetStepEvent event, Emitter<CounterBlocState> emit) {
    emit(state.copyWith(step: event.step));
  }
}

// ===== MAIN =====
void main() {
  runApp(const CounterBlocApp());
}

class CounterBlocApp extends StatelessWidget {
  const CounterBlocApp({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (_) => CounterBlocMain(),
      child: MaterialApp(
        title: 'Counter BLoC',
        theme: ThemeData(
          colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
          useMaterial3: true,
        ),
        home: const CounterHomePage(),
      ),
    );
  }
}

// ===== PAGES =====
class CounterHomePage extends StatelessWidget {
  const CounterHomePage({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return DefaultTabController(
      length: 2,
      child: Scaffold(
        appBar: AppBar(
          title: const Text('Counter with BLoC'),
          backgroundColor: Colors.indigo,
          foregroundColor: Colors.white,
          bottom: const TabBar(
            tabs: [
              Tab(text: 'Counter', icon: Icon(Icons.calculate)),
              Tab(text: 'Log', icon: Icon(Icons.history)),
            ],
            labelColor: Colors.white,
            unselectedLabelColor: Colors.white70,
          ),
        ),
        body: const TabBarView(
          children: [
            CounterTab(),
            LogTab(),
          ],
        ),
      ),
    );
  }
}

class CounterTab extends StatelessWidget {
  const CounterTab({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return BlocConsumer<CounterBlocMain, CounterBlocState>(
      listenWhen: (prev, curr) => curr.count != prev.count,
      listener: (context, state) {
        if (state.count == 100) {
          ScaffoldMessenger.of(context).showSnackBar(
            const SnackBar(
              content: Text('🎉 Reached 100!'),
              backgroundColor: Colors.amber,
            ),
          );
        }
      },
      builder: (context, state) {
        final bloc = context.read<CounterBlocMain>();
        
        return SingleChildScrollView(
          padding: const EdgeInsets.all(24),
          child: Column(
            children: [
              // Counter Display
              AnimatedContainer(
                duration: const Duration(milliseconds: 200),
                width: double.infinity,
                padding: const EdgeInsets.symmetric(vertical: 40),
                decoration: BoxDecoration(
                  gradient: LinearGradient(
                    colors: [
                      Colors.indigo.shade300,
                      Colors.indigo.shade600,
                    ],
                    begin: Alignment.topLeft,
                    end: Alignment.bottomRight,
                  ),
                  borderRadius: BorderRadius.circular(20),
                  boxShadow: [
                    BoxShadow(
                      color: Colors.indigo.withOpacity(0.3),
                      blurRadius: 15,
                      offset: const Offset(0, 5),
                    ),
                  ],
                ),
                child: Column(
                  children: [
                    Text(
                      '${state.count}',
                      style: const TextStyle(
                        fontSize: 80,
                        fontWeight: FontWeight.bold,
                        color: Colors.white,
                      ),
                    ),
                    Text(
                      'Step: ${state.step}',
                      style: const TextStyle(
                        fontSize: 16,
                        color: Colors.white70,
                      ),
                    ),
                  ],
                ),
              ),
              
              const SizedBox(height: 32),
              
              // Controls
              Row(
                mainAxisAlignment: MainAxisAlignment.spaceEvenly,
                children: [
                  _ActionButton(
                    onPressed: state.canDecrement
                        ? () => bloc.add(const DecrementEvent())
                        : null,
                    icon: Icons.remove,
                    label: '-${state.step}',
                    color: Colors.red,
                  ),
                  _ActionButton(
                    onPressed: () => bloc.add(const ResetEvent()),
                    icon: Icons.refresh,
                    label: 'Reset',
                    color: Colors.grey,
                  ),
                  _ActionButton(
                    onPressed: () => bloc.add(const IncrementEvent()),
                    icon: Icons.add,
                    label: '+${state.step}',
                    color: Colors.green,
                  ),
                ],
              ),
              
              const SizedBox(height: 24),
              
              // Step Slider
              Card(
                child: Padding(
                  padding: const EdgeInsets.all(16),
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      const Text(
                        'Step Size',
                        style: TextStyle(fontWeight: FontWeight.bold),
                      ),
                      Slider(
                        min: 1,
                        max: 20,
                        divisions: 19,
                        value: state.step.toDouble(),
                        label: '${state.step}',
                        onChanged: (value) {
                          bloc.add(SetStepEvent(value.round()));
                        },
                      ),
                    ],
                  ),
                ),
              ),
            ],
          ),
        );
      },
    );
  }
}

class _ActionButton extends StatelessWidget {
  final VoidCallback? onPressed;
  final IconData icon;
  final String label;
  final Color color;
  
  const _ActionButton({
    this.onPressed,
    required this.icon,
    required this.label,
    required this.color,
  });
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        ElevatedButton(
          onPressed: onPressed,
          style: ElevatedButton.styleFrom(
            backgroundColor: onPressed != null ? color : Colors.grey[300],
            foregroundColor: Colors.white,
            shape: const CircleBorder(),
            padding: const EdgeInsets.all(16),
          ),
          child: Icon(icon, size: 28),
        ),
        const SizedBox(height: 4),
        Text(label, style: TextStyle(color: color)),
      ],
    );
  }
}

class LogTab extends StatelessWidget {
  const LogTab({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return BlocBuilder<CounterBlocMain, CounterBlocState>(
      buildWhen: (prev, curr) => prev.log != curr.log,
      builder: (context, state) {
        if (state.log.isEmpty) {
          return const Center(
            child: Text('No actions yet', style: TextStyle(color: Colors.grey)),
          );
        }
        
        return ListView.builder(
          reverse: true,
          itemCount: state.log.length,
          itemBuilder: (context, index) {
            final reversedIndex = state.log.length - 1 - index;
            final logEntry = state.log[reversedIndex];
            
            return ListTile(
              leading: CircleAvatar(
                radius: 16,
                backgroundColor: Colors.indigo[100],
                child: Text(
                  '${reversedIndex + 1}',
                  style: TextStyle(
                    fontSize: 11,
                    color: Colors.indigo[800],
                  ),
                ),
              ),
              title: Text(logEntry),
              dense: true,
            );
          },
        );
      },
    );
  }
}
```

---

## BLoC vs Cubit

### เมื่อไหร่ใช้อะไร

```dart
// ใช้ Cubit เมื่อ:
// - Logic ตรง ง่าย
// - ไม่มี events ซับซ้อน
// - ต้องการโค้ดน้อย
class SimpleCubit extends Cubit<int> {
  SimpleCubit() : super(0);
  void increment() => emit(state + 1);
}

// ใช้ BLoC เมื่อ:
// - Events ซับซ้อน มีหลายประเภท
// - ต้องการ trace events
// - Team ใหญ่ ต้องการ structure ชัด
class ComplexBloc extends Bloc<ComplexEvent, ComplexState> {
  ComplexBloc() : super(ComplexState.initial()) {
    on<EventA>(_handleA);
    on<EventB>(_handleB);
    on<EventC>(_handleC);
  }
  // ...
}
```

---

## สรุป BLoC

### ข้อดี
- Separation of concerns ชัดเจน
- Testing ง่ายมาก
- Predictable state flow
- Great DevTools support
- Community ใหญ่

### ข้อเสีย
- Boilerplate เยอะ
- Learning curve สูง
- อาจ over-engineered สำหรับ app เล็ก

### เลือกใช้เมื่อ
- Enterprise app ขนาดใหญ่
- Team ต้องการ structure ชัดเจน
- ต้องการ testing coverage สูง
- Complex business logic
