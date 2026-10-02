# Part 33: Riverpod State Management

## Riverpod คืออะไร

Riverpod เป็น state management library ที่พัฒนาโดย Remi Rousselet (ผู้สร้าง Provider) เป็นการ rewrite Provider ใหม่ทั้งหมดเพื่อแก้ข้อจำกัดหลายอย่าง

### ข้อดีของ Riverpod เหนือ Provider
- ตรวจสอบ compile-time ไม่มี runtime errors
- ไม่ขึ้นกับ BuildContext
- สามารถ access provider จากทุกที่
- รองรับ async ได้ดีมาก
- Testing ง่าย

```yaml
# pubspec.yaml
dependencies:
  flutter_riverpod: ^2.4.0
  riverpod_annotation: ^2.3.0

dev_dependencies:
  riverpod_generator: ^2.3.0
  build_runner: ^2.4.0
```

---

## StateProvider

### StateProvider สำหรับ Simple Values

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

// สร้าง StateProvider
final counterProvider = StateProvider<int>((ref) => 0);
final nameProvider = StateProvider<String>((ref) => 'Guest');
final isDarkModeProvider = StateProvider<bool>((ref) => false);

void main() {
  runApp(
    // ProviderScope ต้องอยู่ที่ root ของ app
    const ProviderScope(
      child: MyApp(),
    ),
  );
}

class MyApp extends ConsumerWidget {
  const MyApp({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final isDarkMode = ref.watch(isDarkModeProvider);
    
    return MaterialApp(
      theme: ThemeData(
        brightness: isDarkMode ? Brightness.dark : Brightness.light,
        useMaterial3: true,
      ),
      home: const CounterPage(),
    );
  }
}

class CounterPage extends ConsumerWidget {
  const CounterPage({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // ref.watch rebuild widget เมื่อ value เปลี่ยน
    final count = ref.watch(counterProvider);
    final name = ref.watch(nameProvider);
    final isDark = ref.watch(isDarkModeProvider);
    
    return Scaffold(
      appBar: AppBar(
        title: Text('Hello, $name'),
        actions: [
          IconButton(
            onPressed: () {
              // ref.read สำหรับ actions ไม่ subscribe
              ref.read(isDarkModeProvider.notifier).state = !isDark;
            },
            icon: Icon(isDark ? Icons.light_mode : Icons.dark_mode),
          ),
        ],
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text('Count: $count', style: const TextStyle(fontSize: 32)),
            const SizedBox(height: 20),
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                FloatingActionButton(
                  onPressed: () {
                    // update state ด้วย notifier
                    ref.read(counterProvider.notifier).state--;
                  },
                  mini: true,
                  child: const Icon(Icons.remove),
                ),
                const SizedBox(width: 16),
                FloatingActionButton(
                  onPressed: () {
                    ref.read(counterProvider.notifier).state++;
                  },
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

## StateNotifierProvider

### StateNotifierProvider สำหรับ Complex State

```dart
// State class (immutable)
class CounterState {
  final int count;
  final int step;
  final List<int> history;
  
  const CounterState({
    this.count = 0,
    this.step = 1,
    this.history = const [],
  });
  
  CounterState copyWith({
    int? count,
    int? step,
    List<int>? history,
  }) {
    return CounterState(
      count: count ?? this.count,
      step: step ?? this.step,
      history: history ?? this.history,
    );
  }
}

// StateNotifier class
class CounterNotifier extends StateNotifier<CounterState> {
  CounterNotifier() : super(const CounterState());
  
  void increment() {
    final newCount = state.count + state.step;
    state = state.copyWith(
      count: newCount,
      history: [...state.history, newCount],
    );
  }
  
  void decrement() {
    final newCount = state.count - state.step;
    state = state.copyWith(
      count: newCount,
      history: [...state.history, newCount],
    );
  }
  
  void setStep(int step) {
    state = state.copyWith(step: step);
  }
  
  void reset() {
    state = const CounterState();
  }
  
  void undo() {
    if (state.history.length > 1) {
      final newHistory = state.history.sublist(0, state.history.length - 1);
      state = state.copyWith(
        count: newHistory.last,
        history: newHistory,
      );
    } else {
      state = state.copyWith(count: 0, history: []);
    }
  }
}

// Provider
final counterNotifierProvider = 
    StateNotifierProvider<CounterNotifier, CounterState>((ref) {
  return CounterNotifier();
});

// Widget ที่ใช้
class AdvancedCounterPage extends ConsumerWidget {
  const AdvancedCounterPage({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(counterNotifierProvider);
    final notifier = ref.read(counterNotifierProvider.notifier);
    
    return Scaffold(
      appBar: AppBar(
        title: const Text('Advanced Counter'),
        actions: [
          IconButton(
            onPressed: state.history.isNotEmpty ? notifier.undo : null,
            icon: const Icon(Icons.undo),
          ),
          IconButton(
            onPressed: notifier.reset,
            icon: const Icon(Icons.refresh),
          ),
        ],
      ),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            Card(
              child: Padding(
                padding: const EdgeInsets.all(24),
                child: Column(
                  children: [
                    Text(
                      '${state.count}',
                      style: const TextStyle(fontSize: 64, fontWeight: FontWeight.bold),
                    ),
                    Text('Step: ${state.step}'),
                  ],
                ),
              ),
            ),
            const SizedBox(height: 16),
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                ElevatedButton.icon(
                  onPressed: notifier.decrement,
                  icon: const Icon(Icons.remove),
                  label: Text('-${state.step}'),
                ),
                const SizedBox(width: 16),
                ElevatedButton.icon(
                  onPressed: notifier.increment,
                  icon: const Icon(Icons.add),
                  label: Text('+${state.step}'),
                ),
              ],
            ),
            const SizedBox(height: 16),
            const Text('Step Size:'),
            Slider(
              min: 1,
              max: 10,
              divisions: 9,
              value: state.step.toDouble(),
              label: '${state.step}',
              onChanged: (value) => notifier.setStep(value.round()),
            ),
            if (state.history.isNotEmpty) ...[
              const Divider(),
              const Text('History:', style: TextStyle(fontWeight: FontWeight.bold)),
              Wrap(
                spacing: 8,
                children: state.history.map((h) => Chip(label: Text('$h'))).toList(),
              ),
            ],
          ],
        ),
      ),
    );
  }
}
```

---

## FutureProvider

### FutureProvider สำหรับ Async Data

```dart
// Data model
class User {
  final int id;
  final String name;
  final String email;
  final String phone;
  
  const User({
    required this.id,
    required this.name,
    required this.email,
    required this.phone,
  });
  
  factory User.fromJson(Map<String, dynamic> json) {
    return User(
      id: json['id'],
      name: json['name'],
      email: json['email'],
      phone: json['phone'],
    );
  }
}

// FutureProvider
final usersProvider = FutureProvider<List<User>>((ref) async {
  // จำลองการดึงข้อมูลจาก API
  await Future.delayed(const Duration(seconds: 2));
  
  return [
    const User(id: 1, name: 'Alice', email: 'alice@example.com', phone: '081-000-0001'),
    const User(id: 2, name: 'Bob', email: 'bob@example.com', phone: '081-000-0002'),
    const User(id: 3, name: 'Charlie', email: 'charlie@example.com', phone: '081-000-0003'),
  ];
});

// Widget ที่ใช้ FutureProvider
class UsersPage extends ConsumerWidget {
  const UsersPage({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // AsyncValue<List<User>>
    final usersAsync = ref.watch(usersProvider);
    
    return Scaffold(
      appBar: AppBar(
        title: const Text('Users'),
        actions: [
          IconButton(
            // invalidate = refresh data
            onPressed: () => ref.invalidate(usersProvider),
            icon: const Icon(Icons.refresh),
          ),
        ],
      ),
      body: usersAsync.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (error, stack) => Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              const Icon(Icons.error_outline, size: 48, color: Colors.red),
              const SizedBox(height: 8),
              Text('Error: $error'),
              ElevatedButton(
                onPressed: () => ref.invalidate(usersProvider),
                child: const Text('Retry'),
              ),
            ],
          ),
        ),
        data: (users) => ListView.builder(
          itemCount: users.length,
          itemBuilder: (context, index) {
            final user = users[index];
            return ListTile(
              leading: CircleAvatar(child: Text('${user.id}')),
              title: Text(user.name),
              subtitle: Text(user.email),
              trailing: Text(user.phone),
            );
          },
        ),
      ),
    );
  }
}
```

### FutureProvider กับ whenOrNull

```dart
class UserWithFallback extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final usersAsync = ref.watch(usersProvider);
    
    return Column(
      children: [
        // แสดง loading indicator overlay
        if (usersAsync.isLoading)
          const LinearProgressIndicator(),
        
        // แสดง error หากมี
        if (usersAsync.hasError)
          ErrorBanner(error: usersAsync.error.toString()),
        
        // แสดง data (หรือ empty list ถ้ายังไม่มี)
        Expanded(
          child: ListView.builder(
            itemCount: usersAsync.valueOrNull?.length ?? 0,
            itemBuilder: (context, index) {
              final user = usersAsync.value![index];
              return ListTile(title: Text(user.name));
            },
          ),
        ),
      ],
    );
  }
}

class ErrorBanner extends StatelessWidget {
  final String error;
  const ErrorBanner({required this.error});
  
  @override
  Widget build(BuildContext context) {
    return Container(
      color: Colors.red[100],
      padding: const EdgeInsets.all(8),
      child: Row(
        children: [
          const Icon(Icons.error, color: Colors.red),
          const SizedBox(width: 8),
          Expanded(child: Text(error, style: const TextStyle(color: Colors.red))),
        ],
      ),
    );
  }
}
```

---

## StreamProvider

### StreamProvider สำหรับ Real-time Data

```dart
// จำลอง real-time data stream
Stream<List<int>> _priceStream() async* {
  final prices = [100, 105, 98, 112, 108, 115, 103];
  for (final price in prices) {
    await Future.delayed(const Duration(seconds: 1));
    yield [price];
  }
}

final stockPriceProvider = StreamProvider<int>((ref) async* {
  int basePrice = 100;
  while (true) {
    await Future.delayed(const Duration(seconds: 2));
    // จำลองการเปลี่ยนแปลงราคาหุ้น
    basePrice += (DateTime.now().second % 10) - 5;
    yield basePrice;
  }
});

// Timer stream
final timerProvider = StreamProvider<DateTime>((ref) {
  return Stream.periodic(
    const Duration(seconds: 1),
    (_) => DateTime.now(),
  );
});

class StockPage extends ConsumerWidget {
  const StockPage({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final priceAsync = ref.watch(stockPriceProvider);
    final timeAsync = ref.watch(timerProvider);
    
    return Scaffold(
      appBar: AppBar(
        title: const Text('Live Stock Price'),
        actions: [
          // แสดงเวลาปัจจุบัน
          Padding(
            padding: const EdgeInsets.all(16),
            child: timeAsync.when(
              loading: () => const SizedBox(),
              error: (_, __) => const SizedBox(),
              data: (time) => Text(
                '${time.hour}:${time.minute.toString().padLeft(2, '0')}:${time.second.toString().padLeft(2, '0')}',
              ),
            ),
          ),
        ],
      ),
      body: Center(
        child: priceAsync.when(
          loading: () => const CircularProgressIndicator(),
          error: (error, _) => Text('Error: $error'),
          data: (price) => Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              const Text('AAPL', style: TextStyle(fontSize: 24)),
              Text(
                '\$$price',
                style: const TextStyle(
                  fontSize: 64,
                  fontWeight: FontWeight.bold,
                  color: Colors.green,
                ),
              ),
              const Text('Live Price', style: TextStyle(color: Colors.grey)),
            ],
          ),
        ),
      ),
    );
  }
}
```

---

## ConsumerWidget และ ref.watch/read

### ConsumerWidget vs StatefulWidget

```dart
// ConsumerWidget - สำหรับ widget ที่ต้องการ WidgetRef
class MyConsumerWidget extends ConsumerWidget {
  const MyConsumerWidget({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(counterProvider);
    return Text('$count');
  }
}

// ConsumerStatefulWidget - เมื่อต้องการทั้ง State และ WidgetRef
class MyStatefulConsumer extends ConsumerStatefulWidget {
  const MyStatefulConsumer({Key? key}) : super(key: key);
  
  @override
  ConsumerState<MyStatefulConsumer> createState() => _MyStatefulConsumerState();
}

class _MyStatefulConsumerState extends ConsumerState<MyStatefulConsumer> {
  // สามารถ access ref ได้ตลอดทั้ง State
  
  @override
  void initState() {
    super.initState();
    // ใช้ ref ใน initState
    Future.microtask(() {
      ref.read(counterProvider.notifier).state = 10;
    });
  }
  
  @override
  Widget build(BuildContext context) {
    // ref ใน ConsumerState ไม่ต้องรับ parameter
    final count = ref.watch(counterProvider);
    return Text('$count');
  }
}
```

### ref.listen สำหรับ Side Effects

```dart
class ListenerExample extends ConsumerWidget {
  const ListenerExample({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // listen เพื่อ react ต่อการเปลี่ยนแปลง แต่ไม่ rebuild
    ref.listen<int>(counterProvider, (previous, next) {
      if (next >= 10) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('Counter reached 10!')),
        );
      }
    });
    
    final count = ref.watch(counterProvider);
    
    return Column(
      children: [
        Text('Count: $count'),
        ElevatedButton(
          onPressed: () => ref.read(counterProvider.notifier).state++,
          child: const Text('Increment'),
        ),
      ],
    );
  }
}
```

---

## Provider Families

### Family Parameters

```dart
// Provider ที่รับ parameter
final userByIdProvider = FutureProvider.family<User, int>((ref, userId) async {
  await Future.delayed(const Duration(milliseconds: 500));
  // จำลองดึง user ตาม ID
  final users = {
    1: const User(id: 1, name: 'Alice', email: 'alice@example.com', phone: '081-111-1111'),
    2: const User(id: 2, name: 'Bob', email: 'bob@example.com', phone: '081-222-2222'),
    3: const User(id: 3, name: 'Charlie', email: 'charlie@example.com', phone: '081-333-3333'),
  };
  final user = users[userId];
  if (user == null) throw Exception('User $userId not found');
  return user;
});

// Provider family กับ String parameter
final searchResultsProvider = FutureProvider.family<List<User>, String>((ref, query) async {
  if (query.isEmpty) return [];
  await Future.delayed(const Duration(milliseconds: 300));
  
  final allUsers = [
    const User(id: 1, name: 'Alice Smith', email: 'alice@example.com', phone: '081-111'),
    const User(id: 2, name: 'Bob Johnson', email: 'bob@example.com', phone: '081-222'),
    const User(id: 3, name: 'Charlie Brown', email: 'charlie@example.com', phone: '081-333'),
    const User(id: 4, name: 'Alice Johnson', email: 'alicej@example.com', phone: '081-444'),
  ];
  
  return allUsers
      .where((u) => u.name.toLowerCase().contains(query.toLowerCase()))
      .toList();
});

// Widget ที่ใช้ family
class UserDetailPage extends ConsumerWidget {
  final int userId;
  
  const UserDetailPage({required this.userId, Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // ส่ง parameter ให้ provider
    final userAsync = ref.watch(userByIdProvider(userId));
    
    return Scaffold(
      appBar: AppBar(title: Text('User #$userId')),
      body: userAsync.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (error, _) => Center(child: Text('Error: $error')),
        data: (user) => ListTile(
          title: Text(user.name),
          subtitle: Text(user.email),
          trailing: Text(user.phone),
        ),
      ),
    );
  }
}

// Search Widget
class SearchPage extends ConsumerStatefulWidget {
  const SearchPage({Key? key}) : super(key: key);
  
  @override
  ConsumerState<SearchPage> createState() => _SearchPageState();
}

class _SearchPageState extends ConsumerState<SearchPage> {
  final _searchController = TextEditingController();
  String _query = '';
  
  @override
  void dispose() {
    _searchController.dispose();
    super.dispose();
  }
  
  @override
  Widget build(BuildContext context) {
    final resultsAsync = ref.watch(searchResultsProvider(_query));
    
    return Scaffold(
      appBar: AppBar(
        title: TextField(
          controller: _searchController,
          decoration: const InputDecoration(
            hintText: 'Search users...',
            border: InputBorder.none,
          ),
          onChanged: (value) => setState(() => _query = value),
        ),
      ),
      body: resultsAsync.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (error, _) => Center(child: Text('Error: $error')),
        data: (users) {
          if (users.isEmpty && _query.isNotEmpty) {
            return const Center(child: Text('No users found'));
          }
          return ListView.builder(
            itemCount: users.length,
            itemBuilder: (context, index) {
              final user = users[index];
              return ListTile(
                leading: CircleAvatar(child: Text(user.name[0])),
                title: Text(user.name),
                subtitle: Text(user.email),
              );
            },
          );
        },
      ),
    );
  }
}
```

---

## Workshop: Todo App with Riverpod

### Data Models

```dart
// models/todo.dart
import 'package:flutter/foundation.dart';

enum TodoFilter { all, active, completed }

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
    String? id,
    String? title,
    bool? isCompleted,
    DateTime? createdAt,
  }) {
    return Todo(
      id: id ?? this.id,
      title: title ?? this.title,
      isCompleted: isCompleted ?? this.isCompleted,
      createdAt: createdAt ?? this.createdAt,
    );
  }
  
  @override
  bool operator ==(Object other) =>
      identical(this, other) ||
      other is Todo && runtimeType == other.runtimeType && id == other.id;
  
  @override
  int get hashCode => id.hashCode;
}
```

### Providers

```dart
// providers/todo_providers.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../models/todo.dart';

// Todo list state
class TodoListNotifier extends StateNotifier<List<Todo>> {
  TodoListNotifier() : super([
    Todo(
      id: '1',
      title: 'Learn Flutter',
      isCompleted: true,
      createdAt: DateTime.now().subtract(const Duration(days: 2)),
    ),
    Todo(
      id: '2',
      title: 'Master Riverpod',
      createdAt: DateTime.now().subtract(const Duration(days: 1)),
    ),
    Todo(
      id: '3',
      title: 'Build awesome apps',
      createdAt: DateTime.now(),
    ),
  ]);
  
  void addTodo(String title) {
    if (title.trim().isEmpty) return;
    state = [
      ...state,
      Todo(
        id: DateTime.now().millisecondsSinceEpoch.toString(),
        title: title.trim(),
        createdAt: DateTime.now(),
      ),
    ];
  }
  
  void toggleTodo(String id) {
    state = state.map((todo) {
      if (todo.id == id) {
        return todo.copyWith(isCompleted: !todo.isCompleted);
      }
      return todo;
    }).toList();
  }
  
  void removeTodo(String id) {
    state = state.where((todo) => todo.id != id).toList();
  }
  
  void updateTitle(String id, String title) {
    if (title.trim().isEmpty) return;
    state = state.map((todo) {
      if (todo.id == id) {
        return todo.copyWith(title: title.trim());
      }
      return todo;
    }).toList();
  }
  
  void clearCompleted() {
    state = state.where((todo) => !todo.isCompleted).toList();
  }
  
  void toggleAll() {
    final allCompleted = state.every((todo) => todo.isCompleted);
    state = state.map((todo) => todo.copyWith(isCompleted: !allCompleted)).toList();
  }
}

final todoListProvider = StateNotifierProvider<TodoListNotifier, List<Todo>>((ref) {
  return TodoListNotifier();
});

// Filter provider
final todoFilterProvider = StateProvider<TodoFilter>((ref) => TodoFilter.all);

// Derived providers
final filteredTodosProvider = Provider<List<Todo>>((ref) {
  final todos = ref.watch(todoListProvider);
  final filter = ref.watch(todoFilterProvider);
  
  switch (filter) {
    case TodoFilter.all:
      return todos;
    case TodoFilter.active:
      return todos.where((todo) => !todo.isCompleted).toList();
    case TodoFilter.completed:
      return todos.where((todo) => todo.isCompleted).toList();
  }
});

final todoStatsProvider = Provider<Map<String, int>>((ref) {
  final todos = ref.watch(todoListProvider);
  return {
    'total': todos.length,
    'completed': todos.where((t) => t.isCompleted).length,
    'active': todos.where((t) => !t.isCompleted).length,
  };
});
```

### Main App

```dart
// main.dart
void main() {
  runApp(
    const ProviderScope(
      child: TodoApp(),
    ),
  );
}

class TodoApp extends StatelessWidget {
  const TodoApp({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Todo Riverpod',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
        useMaterial3: true,
      ),
      home: const TodoPage(),
    );
  }
}
```

### Todo Page

```dart
// screens/todo_page.dart
class TodoPage extends ConsumerWidget {
  const TodoPage({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final stats = ref.watch(todoStatsProvider);
    
    return Scaffold(
      appBar: AppBar(
        title: const Text('My Todos'),
        backgroundColor: Colors.teal,
        foregroundColor: Colors.white,
        bottom: PreferredSize(
          preferredSize: const Size.fromHeight(48),
          child: Container(
            padding: const EdgeInsets.only(bottom: 8, left: 16, right: 16),
            child: Row(
              children: [
                Text(
                  '${stats['completed']}/${stats['total']} completed',
                  style: const TextStyle(color: Colors.white70),
                ),
                const Spacer(),
                if ((stats['completed'] ?? 0) > 0)
                  TextButton(
                    onPressed: () => ref.read(todoListProvider.notifier).clearCompleted(),
                    child: const Text(
                      'Clear completed',
                      style: TextStyle(color: Colors.white70),
                    ),
                  ),
              ],
            ),
          ),
        ),
      ),
      body: Column(
        children: [
          const TodoFilterBar(),
          const Expanded(child: TodoList()),
          const TodoInputField(),
        ],
      ),
    );
  }
}

// Filter Bar
class TodoFilterBar extends ConsumerWidget {
  const TodoFilterBar({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final currentFilter = ref.watch(todoFilterProvider);
    
    return Container(
      color: Colors.teal.withOpacity(0.1),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.center,
        children: TodoFilter.values.map((filter) {
          return Padding(
            padding: const EdgeInsets.symmetric(horizontal: 4, vertical: 8),
            child: FilterChip(
              label: Text(_filterLabel(filter)),
              selected: currentFilter == filter,
              onSelected: (_) => ref.read(todoFilterProvider.notifier).state = filter,
              selectedColor: Colors.teal,
              labelStyle: TextStyle(
                color: currentFilter == filter ? Colors.white : null,
              ),
            ),
          );
        }).toList(),
      ),
    );
  }
  
  String _filterLabel(TodoFilter filter) {
    switch (filter) {
      case TodoFilter.all: return 'All';
      case TodoFilter.active: return 'Active';
      case TodoFilter.completed: return 'Completed';
    }
  }
}

// Todo List
class TodoList extends ConsumerWidget {
  const TodoList({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final todos = ref.watch(filteredTodosProvider);
    
    if (todos.isEmpty) {
      return const Center(
        child: Text(
          'No todos here!',
          style: TextStyle(fontSize: 18, color: Colors.grey),
        ),
      );
    }
    
    return ListView.builder(
      itemCount: todos.length,
      itemBuilder: (context, index) {
        return TodoItem(todo: todos[index]);
      },
    );
  }
}

// Todo Item
class TodoItem extends ConsumerStatefulWidget {
  final Todo todo;
  
  const TodoItem({required this.todo, Key? key}) : super(key: key);
  
  @override
  ConsumerState<TodoItem> createState() => _TodoItemState();
}

class _TodoItemState extends ConsumerState<TodoItem> {
  bool _isEditing = false;
  late final TextEditingController _editController;
  
  @override
  void initState() {
    super.initState();
    _editController = TextEditingController(text: widget.todo.title);
  }
  
  @override
  void dispose() {
    _editController.dispose();
    super.dispose();
  }
  
  void _saveEdit() {
    if (_editController.text.trim().isNotEmpty) {
      ref.read(todoListProvider.notifier).updateTitle(
        widget.todo.id,
        _editController.text,
      );
    }
    setState(() => _isEditing = false);
  }
  
  @override
  Widget build(BuildContext context) {
    final todo = widget.todo;
    
    return Dismissible(
      key: Key(todo.id),
      direction: DismissDirection.endToStart,
      background: Container(
        alignment: Alignment.centerRight,
        padding: const EdgeInsets.only(right: 20),
        color: Colors.red,
        child: const Icon(Icons.delete, color: Colors.white),
      ),
      onDismissed: (_) => ref.read(todoListProvider.notifier).removeTodo(todo.id),
      child: ListTile(
        leading: Checkbox(
          value: todo.isCompleted,
          activeColor: Colors.teal,
          onChanged: (_) => ref.read(todoListProvider.notifier).toggleTodo(todo.id),
        ),
        title: _isEditing
            ? TextField(
                controller: _editController,
                autofocus: true,
                onSubmitted: (_) => _saveEdit(),
                decoration: const InputDecoration(border: UnderlineInputBorder()),
              )
            : Text(
                todo.title,
                style: TextStyle(
                  decoration: todo.isCompleted ? TextDecoration.lineThrough : null,
                  color: todo.isCompleted ? Colors.grey : null,
                ),
              ),
        trailing: Row(
          mainAxisSize: MainAxisSize.min,
          children: [
            if (_isEditing) ...[
              IconButton(
                onPressed: _saveEdit,
                icon: const Icon(Icons.check, color: Colors.green),
              ),
              IconButton(
                onPressed: () => setState(() {
                  _isEditing = false;
                  _editController.text = todo.title;
                }),
                icon: const Icon(Icons.close, color: Colors.red),
              ),
            ] else
              IconButton(
                onPressed: () => setState(() => _isEditing = true),
                icon: const Icon(Icons.edit, color: Colors.grey),
              ),
          ],
        ),
      ),
    );
  }
}

// Input Field
class TodoInputField extends ConsumerStatefulWidget {
  const TodoInputField({Key? key}) : super(key: key);
  
  @override
  ConsumerState<TodoInputField> createState() => _TodoInputFieldState();
}

class _TodoInputFieldState extends ConsumerState<TodoInputField> {
  final _controller = TextEditingController();
  
  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
  
  void _addTodo() {
    if (_controller.text.trim().isNotEmpty) {
      ref.read(todoListProvider.notifier).addTodo(_controller.text);
      _controller.clear();
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(16),
      decoration: BoxDecoration(
        color: Theme.of(context).colorScheme.surface,
        boxShadow: [
          BoxShadow(
            color: Colors.black.withOpacity(0.1),
            blurRadius: 4,
            offset: const Offset(0, -2),
          ),
        ],
      ),
      child: Row(
        children: [
          Expanded(
            child: TextField(
              controller: _controller,
              decoration: const InputDecoration(
                hintText: 'Add new todo...',
                border: OutlineInputBorder(),
                contentPadding: EdgeInsets.symmetric(horizontal: 12, vertical: 8),
              ),
              onSubmitted: (_) => _addTodo(),
            ),
          ),
          const SizedBox(width: 8),
          ElevatedButton(
            onPressed: _addTodo,
            style: ElevatedButton.styleFrom(
              backgroundColor: Colors.teal,
              foregroundColor: Colors.white,
              padding: const EdgeInsets.symmetric(vertical: 12, horizontal: 16),
            ),
            child: const Icon(Icons.add),
          ),
        ],
      ),
    );
  }
}
```

---

## สรุป Riverpod

### ความแตกต่างจาก Provider
| Feature | Provider | Riverpod |
|---------|----------|----------|
| Compile-time safety | ไม่มี | มี |
| Access นอก widget tree | ไม่ได้ | ได้ |
| Multiple providers เดียวกัน | ไม่ได้ | ได้ |
| Auto dispose | ไม่มีในตัว | มี |
| Async support | ต้องทำเอง | Built-in |

### เลือก Provider ไหน
```dart
// ค่า simple
final countProvider = StateProvider<int>((ref) => 0);

// logic ซับซ้อน  
final todoProvider = StateNotifierProvider<TodoNotifier, List<Todo>>((ref) => TodoNotifier());

// async data
final userProvider = FutureProvider<User>((ref) async => await fetchUser());

// real-time data
final chatProvider = StreamProvider<List<Message>>((ref) => chatStream());

// derived/computed values
final completedTodosProvider = Provider<List<Todo>>((ref) {
  return ref.watch(todoProvider).where((t) => t.isCompleted).toList();
});
```
