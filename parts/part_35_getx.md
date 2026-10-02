# Part 35: GetX State Management

## GetX คืออะไร

GetX เป็น all-in-one solution ที่รวม State Management, Navigation, Dependency Injection และอื่นๆ ไว้ในแพ็คเกจเดียว

```yaml
# pubspec.yaml
dependencies:
  get: ^4.6.6
```

### GetX ให้อะไรบ้าง
- State Management (Reactive + Simple)
- Route Management (Navigation)
- Dependency Injection
- Internationalization
- Utils (Snackbar, Dialog, BottomSheet)

---

## GetxController

### สร้าง Controller พื้นฐาน

```dart
import 'package:get/get.dart';

// Simple Counter Controller
class CounterController extends GetxController {
  // .obs ทำให้ variable เป็น reactive
  var count = 0.obs;
  var step = 1.obs;
  var history = <int>[].obs;
  
  void increment() {
    count.value += step.value;
    history.add(count.value);
  }
  
  void decrement() {
    if (count.value - step.value >= 0) {
      count.value -= step.value;
      history.add(count.value);
    }
  }
  
  void reset() {
    count.value = 0;
    history.clear();
  }
  
  void setStep(int newStep) {
    step.value = newStep;
  }
  
  // lifecycle methods
  @override
  void onInit() {
    super.onInit();
    print('CounterController initialized');
  }
  
  @override
  void onClose() {
    print('CounterController disposed');
    super.onClose();
  }
}

// Complex state controller
class UserController extends GetxController {
  var users = <User>[].obs;
  var isLoading = false.obs;
  var errorMessage = ''.obs;
  var selectedUser = Rxn<User>();  // Rxn = nullable Rx
  
  @override
  void onInit() {
    super.onInit();
    fetchUsers();
  }
  
  Future<void> fetchUsers() async {
    isLoading.value = true;
    errorMessage.value = '';
    
    try {
      await Future.delayed(const Duration(seconds: 2));
      users.assignAll([
        User(id: '1', name: 'Alice', email: 'alice@example.com'),
        User(id: '2', name: 'Bob', email: 'bob@example.com'),
        User(id: '3', name: 'Charlie', email: 'charlie@example.com'),
      ]);
    } catch (e) {
      errorMessage.value = e.toString();
    } finally {
      isLoading.value = false;
    }
  }
  
  void selectUser(User user) {
    selectedUser.value = user;
  }
  
  void deleteUser(String id) {
    users.removeWhere((user) => user.id == id);
    if (selectedUser.value?.id == id) {
      selectedUser.value = null;
    }
  }
}

class User {
  final String id;
  final String name;
  final String email;
  
  User({required this.id, required this.name, required this.email});
}
```

---

## Obx และ GetBuilder

### Obx - Reactive Widget

```dart
// main.dart
import 'package:flutter/material.dart';
import 'package:get/get.dart';

void main() {
  runApp(const GetXApp());
}

class GetXApp extends StatelessWidget {
  const GetXApp({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return GetMaterialApp(  // ใช้ GetMaterialApp แทน MaterialApp
      title: 'GetX Demo',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.purple),
        useMaterial3: true,
      ),
      home: const CounterPage(),
    );
  }
}

class CounterPage extends StatelessWidget {
  const CounterPage({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    // Get.put ลงทะเบียน controller
    final controller = Get.put(CounterController());
    
    return Scaffold(
      appBar: AppBar(
        title: const Text('GetX Counter'),
        actions: [
          IconButton(
            onPressed: controller.reset,
            icon: const Icon(Icons.refresh),
          ),
        ],
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            // Obx rebuild เมื่อ reactive variable เปลี่ยน
            Obx(() => Text(
              '${controller.count}',
              style: const TextStyle(fontSize: 72, fontWeight: FontWeight.bold),
            )),
            
            // Obx สำหรับส่วนต่างๆ
            Obx(() => Text('Step: ${controller.step}')),
            
            const SizedBox(height: 24),
            
            Slider(
              min: 1,
              max: 10,
              divisions: 9,
              // ต้องใช้ Obx เพราะ step เป็น reactive
              value: controller.step.value.toDouble(),
              label: '${controller.step.value}',
              onChanged: (value) => controller.setStep(value.round()),
            ),
            
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                FloatingActionButton(
                  onPressed: controller.decrement,
                  mini: true,
                  child: const Icon(Icons.remove),
                ),
                const SizedBox(width: 24),
                FloatingActionButton(
                  onPressed: controller.increment,
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

### GetBuilder - Simple State Builder

```dart
// GetBuilder ใช้กับ GetxController ที่ไม่ใช้ .obs
class SimpleController extends GetxController {
  int count = 0;
  
  void increment() {
    count++;
    update();  // ต้อง call update() เพื่อ rebuild
  }
  
  void decrement() {
    if (count > 0) count--;
    update();
  }
}

class SimpleCounterPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return GetBuilder<SimpleController>(
      init: SimpleController(),  // init controller ที่นี่
      builder: (controller) {
        return Column(
          children: [
            Text('${controller.count}'),
            ElevatedButton(
              onPressed: controller.increment,
              child: const Text('Increment'),
            ),
          ],
        );
      },
    );
  }
}

// Obx vs GetBuilder
// Obx:
// - ใช้กับ .obs variables
// - Rebuild เฉพาะ reactive variables ที่เปลี่ยน
// - ไม่ต้องระบุ controller type
// 
// GetBuilder:
// - ใช้กับ update()
// - ควบคุม rebuild ได้ผ่าน id
// - ดีกว่าสำหรับ list updates

class SectionController extends GetxController {
  int sectionA = 0;
  int sectionB = 0;
  
  void updateSectionA() {
    sectionA++;
    update(['section_a']);  // update เฉพาะ section_a
  }
  
  void updateSectionB() {
    sectionB++;
    update(['section_b']);  // update เฉพาะ section_b
  }
}

class SectionPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final controller = Get.put(SectionController());
    
    return Column(
      children: [
        // Rebuild เฉพาะเมื่อ section_a update
        GetBuilder<SectionController>(
          id: 'section_a',
          builder: (ctrl) => Text('Section A: ${ctrl.sectionA}'),
        ),
        // Rebuild เฉพาะเมื่อ section_b update
        GetBuilder<SectionController>(
          id: 'section_b',
          builder: (ctrl) => Text('Section B: ${ctrl.sectionB}'),
        ),
        ElevatedButton(
          onPressed: controller.updateSectionA,
          child: const Text('Update A'),
        ),
        ElevatedButton(
          onPressed: controller.updateSectionB,
          child: const Text('Update B'),
        ),
      ],
    );
  }
}
```

---

## GetX Navigation

### Named Routes

```dart
// routes/app_pages.dart
import 'package:get/get.dart';

abstract class AppRoutes {
  static const String home = '/home';
  static const String detail = '/detail';
  static const String profile = '/profile';
  static const String settings = '/settings';
}

class AppPages {
  static final routes = [
    GetPage(
      name: AppRoutes.home,
      page: () => const HomePage(),
      binding: HomeBinding(),  // Dependency binding
    ),
    GetPage(
      name: AppRoutes.detail,
      page: () => const DetailPage(),
      transition: Transition.rightToLeft,
    ),
    GetPage(
      name: AppRoutes.profile,
      page: () => const ProfilePage(),
      middlewares: [AuthMiddleware()],  // Guards
    ),
    GetPage(
      name: AppRoutes.settings,
      page: () => const SettingsPage(),
    ),
  ];
}

// ใน main.dart
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return GetMaterialApp(
      initialRoute: AppRoutes.home,
      getPages: AppPages.routes,
    );
  }
}
```

### Navigation Methods

```dart
class NavigationExamples {
  void demonstrateNavigation() {
    // Navigate to named route
    Get.toNamed(AppRoutes.detail);
    
    // Navigate with arguments
    Get.toNamed(
      AppRoutes.detail,
      arguments: {'id': '123', 'name': 'Test'},
    );
    
    // Navigate and remove current page
    Get.offNamed(AppRoutes.home);
    
    // Navigate and remove all pages
    Get.offAllNamed(AppRoutes.home);
    
    // Go back
    Get.back();
    
    // Go back with result
    Get.back(result: 'some result');
    
    // Navigate without named route
    Get.to(() => const DetailPage());
    
    // Navigate with transition
    Get.to(
      () => const DetailPage(),
      transition: Transition.zoom,
      duration: const Duration(milliseconds: 300),
    );
  }
}

// รับ arguments
class DetailPage extends StatelessWidget {
  const DetailPage({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    // รับ arguments
    final args = Get.arguments as Map<String, dynamic>?;
    final id = args?['id'] ?? '';
    final name = args?['name'] ?? 'Unknown';
    
    // รับ parameters จาก URL
    final urlParam = Get.parameters['id'] ?? '';
    
    return Scaffold(
      appBar: AppBar(title: Text('Detail: $name')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text('ID: $id'),
            Text('Name: $name'),
            ElevatedButton(
              onPressed: () => Get.back(result: 'Done with $id'),
              child: const Text('Go Back with Result'),
            ),
          ],
        ),
      ),
    );
  }
}
```

### GetX Dialogs และ Snackbars

```dart
class GetXUtils {
  // Snackbar
  static void showSuccess(String message) {
    Get.snackbar(
      'Success',
      message,
      snackPosition: SnackPosition.BOTTOM,
      backgroundColor: Colors.green,
      colorText: Colors.white,
      duration: const Duration(seconds: 2),
      icon: const Icon(Icons.check_circle, color: Colors.white),
    );
  }
  
  static void showError(String message) {
    Get.snackbar(
      'Error',
      message,
      snackPosition: SnackPosition.BOTTOM,
      backgroundColor: Colors.red,
      colorText: Colors.white,
      duration: const Duration(seconds: 3),
    );
  }
  
  // Default Dialog
  static Future<bool?> showConfirmDialog({
    required String title,
    required String message,
    String confirmText = 'Confirm',
    String cancelText = 'Cancel',
  }) async {
    return await Get.defaultDialog<bool>(
      title: title,
      middleText: message,
      textConfirm: confirmText,
      textCancel: cancelText,
      confirmTextColor: Colors.white,
      onConfirm: () => Get.back(result: true),
      onCancel: () {},
    );
  }
  
  // Bottom Sheet
  static void showBottomSheet(Widget child) {
    Get.bottomSheet(
      child,
      backgroundColor: Colors.white,
      shape: const RoundedRectangleBorder(
        borderRadius: BorderRadius.vertical(top: Radius.circular(20)),
      ),
    );
  }
}

// ตัวอย่างการใช้งาน
class ExamplePage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            ElevatedButton(
              onPressed: () => GetXUtils.showSuccess('Operation completed!'),
              child: const Text('Show Success'),
            ),
            ElevatedButton(
              onPressed: () async {
                final confirmed = await GetXUtils.showConfirmDialog(
                  title: 'Delete Item',
                  message: 'Are you sure you want to delete this item?',
                  confirmText: 'Delete',
                );
                if (confirmed == true) {
                  GetXUtils.showSuccess('Item deleted');
                }
              },
              child: const Text('Show Confirm Dialog'),
            ),
            ElevatedButton(
              onPressed: () => GetXUtils.showBottomSheet(
                Container(
                  padding: const EdgeInsets.all(24),
                  child: const Column(
                    mainAxisSize: MainAxisSize.min,
                    children: [
                      Text('Bottom Sheet Content'),
                    ],
                  ),
                ),
              ),
              child: const Text('Show Bottom Sheet'),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## Dependency Injection with GetX

### Get.put, Get.lazyPut, Get.find

```dart
// Get.put - ลงทะเบียนทันที
final controller = Get.put(CounterController());

// Get.lazyPut - ลงทะเบียนแต่สร้างเมื่อใช้จริง
Get.lazyPut(() => CounterController());

// Get.find - หา instance ที่ลงทะเบียนแล้ว
final controller = Get.find<CounterController>();

// Get.putAsync - สร้าง async
Get.putAsync<UserController>(() async {
  final userController = UserController();
  await userController.init();
  return userController;
});

// Permanent - ไม่ถูก dispose เมื่อ route เปลี่ยน
Get.put(CounterController(), permanent: true);

// Tag - ลงทะเบียนหลาย instance ของ class เดียวกัน
Get.put(CounterController(), tag: 'counter1');
Get.put(CounterController(), tag: 'counter2');
Get.find<CounterController>(tag: 'counter1');
```

### Bindings

```dart
// Binding: ลงทะเบียน dependencies สำหรับ route
class HomeBinding extends Bindings {
  @override
  void dependencies() {
    Get.lazyPut(() => HomeController());
    Get.lazyPut(() => UserController());
  }
}

class DetailBinding extends Bindings {
  @override
  void dependencies() {
    Get.lazyPut(() => DetailController());
  }
}

// ใน AppPages
GetPage(
  name: AppRoutes.home,
  page: () => const HomePage(),
  binding: HomeBinding(),  // Automatically injects dependencies
),
GetPage(
  name: AppRoutes.detail,
  page: () => const DetailPage(),
  binding: DetailBinding(),
),

// HomeController สามารถ access UserController ได้
class HomeController extends GetxController {
  // Get.find จะหา UserController ที่ HomeBinding ลงทะเบียนไว้
  final userController = Get.find<UserController>();
  
  @override
  void onInit() {
    super.onInit();
    userController.fetchUsers();
  }
}
```

---

## Workshop: Todo App with GetX

### Controllers

```dart
// controllers/todo_controller.dart
import 'package:get/get.dart';
import '../models/todo_model.dart';

class TodoController extends GetxController {
  var todos = <TodoModel>[].obs;
  var filterStatus = TodoFilter.all.obs;
  var isLoading = false.obs;
  
  List<TodoModel> get filteredTodos {
    switch (filterStatus.value) {
      case TodoFilter.active:
        return todos.where((t) => !t.isCompleted).toList();
      case TodoFilter.completed:
        return todos.where((t) => t.isCompleted).toList();
      case TodoFilter.all:
      default:
        return todos.toList();
    }
  }
  
  int get completedCount => todos.where((t) => t.isCompleted).length;
  int get activeCount => todos.where((t) => !t.isCompleted).length;
  
  @override
  void onInit() {
    super.onInit();
    _loadSampleTodos();
  }
  
  void _loadSampleTodos() {
    todos.assignAll([
      TodoModel(
        id: '1',
        title: 'Learn Flutter',
        isCompleted: true,
        createdAt: DateTime.now().subtract(const Duration(days: 3)),
      ),
      TodoModel(
        id: '2',
        title: 'Master GetX',
        createdAt: DateTime.now().subtract(const Duration(days: 1)),
      ),
      TodoModel(
        id: '3',
        title: 'Build awesome app',
        createdAt: DateTime.now(),
      ),
    ]);
  }
  
  void addTodo(String title) {
    if (title.trim().isEmpty) {
      Get.snackbar(
        'Error',
        'Title cannot be empty',
        backgroundColor: Colors.red,
        colorText: Colors.white,
      );
      return;
    }
    
    todos.add(TodoModel(
      id: DateTime.now().millisecondsSinceEpoch.toString(),
      title: title.trim(),
      createdAt: DateTime.now(),
    ));
    
    Get.snackbar(
      'Success',
      'Todo added!',
      snackPosition: SnackPosition.BOTTOM,
      duration: const Duration(seconds: 1),
    );
  }
  
  void toggleTodo(String id) {
    final index = todos.indexWhere((t) => t.id == id);
    if (index >= 0) {
      todos[index] = todos[index].copyWith(
        isCompleted: !todos[index].isCompleted,
      );
    }
  }
  
  void deleteTodo(String id) {
    todos.removeWhere((t) => t.id == id);
  }
  
  void updateTitle(String id, String newTitle) {
    final index = todos.indexWhere((t) => t.id == id);
    if (index >= 0) {
      todos[index] = todos[index].copyWith(title: newTitle.trim());
    }
  }
  
  void clearCompleted() {
    todos.removeWhere((t) => t.isCompleted);
  }
  
  void setFilter(TodoFilter filter) {
    filterStatus.value = filter;
  }
}

// models/todo_model.dart
enum TodoFilter { all, active, completed }

class TodoModel {
  final String id;
  final String title;
  final bool isCompleted;
  final DateTime createdAt;
  
  TodoModel({
    required this.id,
    required this.title,
    this.isCompleted = false,
    required this.createdAt,
  });
  
  TodoModel copyWith({
    String? id,
    String? title,
    bool? isCompleted,
    DateTime? createdAt,
  }) {
    return TodoModel(
      id: id ?? this.id,
      title: title ?? this.title,
      isCompleted: isCompleted ?? this.isCompleted,
      createdAt: createdAt ?? this.createdAt,
    );
  }
}
```

### Main App

```dart
// main.dart
import 'package:flutter/material.dart';
import 'package:get/get.dart';

void main() {
  runApp(const TodoApp());
}

class TodoApp extends StatelessWidget {
  const TodoApp({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return GetMaterialApp(
      title: 'GetX Todo',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
        useMaterial3: true,
      ),
      home: const TodoPage(),
      initialBinding: BindingsBuilder(() {
        Get.put(TodoController());
      }),
    );
  }
}
```

### Todo Page

```dart
// pages/todo_page.dart
class TodoPage extends GetView<TodoController> {
  const TodoPage({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('GetX Todos'),
        backgroundColor: Colors.teal,
        foregroundColor: Colors.white,
        actions: [
          Obx(() {
            if (controller.completedCount == 0) return const SizedBox();
            return TextButton(
              onPressed: () async {
                final confirmed = await Get.defaultDialog<bool>(
                  title: 'Clear Completed',
                  middleText: 'Remove all ${controller.completedCount} completed todos?',
                  textConfirm: 'Clear',
                  textCancel: 'Cancel',
                  confirmTextColor: Colors.white,
                  buttonColor: Colors.red,
                  onConfirm: () => Get.back(result: true),
                );
                if (confirmed == true) {
                  controller.clearCompleted();
                }
              },
              child: const Text('Clear Done', style: TextStyle(color: Colors.white)),
            );
          }),
        ],
      ),
      body: Column(
        children: [
          // Stats Header
          Obx(() => Container(
            padding: const EdgeInsets.all(16),
            color: Colors.teal.withOpacity(0.1),
            child: Row(
              mainAxisAlignment: MainAxisAlignment.spaceAround,
              children: [
                _StatChip(
                  label: 'Total',
                  value: '${controller.todos.length}',
                  color: Colors.teal,
                ),
                _StatChip(
                  label: 'Active',
                  value: '${controller.activeCount}',
                  color: Colors.orange,
                ),
                _StatChip(
                  label: 'Done',
                  value: '${controller.completedCount}',
                  color: Colors.green,
                ),
              ],
            ),
          )),
          
          // Filter Chips
          Obx(() => SingleChildScrollView(
            scrollDirection: Axis.horizontal,
            padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
            child: Row(
              children: TodoFilter.values.map((filter) {
                return Padding(
                  padding: const EdgeInsets.only(right: 8),
                  child: FilterChip(
                    label: Text(_filterLabel(filter)),
                    selected: controller.filterStatus.value == filter,
                    onSelected: (_) => controller.setFilter(filter),
                    selectedColor: Colors.teal,
                    labelStyle: TextStyle(
                      color: controller.filterStatus.value == filter
                          ? Colors.white
                          : null,
                    ),
                  ),
                );
              }).toList(),
            ),
          )),
          
          // Todo List
          Expanded(
            child: Obx(() {
              final todos = controller.filteredTodos;
              if (todos.isEmpty) {
                return const Center(
                  child: Text(
                    'No todos here!',
                    style: TextStyle(color: Colors.grey, fontSize: 16),
                  ),
                );
              }
              return ListView.builder(
                itemCount: todos.length,
                itemBuilder: (context, index) {
                  return TodoItemWidget(todo: todos[index]);
                },
              );
            }),
          ),
          
          // Input
          const AddTodoWidget(),
        ],
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

class _StatChip extends StatelessWidget {
  final String label;
  final String value;
  final Color color;
  
  const _StatChip({
    required this.label,
    required this.value,
    required this.color,
  });
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text(
          value,
          style: TextStyle(
            fontSize: 24,
            fontWeight: FontWeight.bold,
            color: color,
          ),
        ),
        Text(
          label,
          style: TextStyle(color: color, fontSize: 12),
        ),
      ],
    );
  }
}
```

### Todo Item Widget

```dart
class TodoItemWidget extends StatefulWidget {
  final TodoModel todo;
  
  const TodoItemWidget({required this.todo, Key? key}) : super(key: key);
  
  @override
  State<TodoItemWidget> createState() => _TodoItemWidgetState();
}

class _TodoItemWidgetState extends State<TodoItemWidget> {
  bool _isEditing = false;
  late final TextEditingController _editController;
  
  final todoController = Get.find<TodoController>();
  
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
      todoController.updateTitle(widget.todo.id, _editController.text);
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
      confirmDismiss: (_) async {
        return await Get.defaultDialog<bool>(
          title: 'Delete Todo',
          middleText: 'Delete "${todo.title}"?',
          textConfirm: 'Delete',
          textCancel: 'Cancel',
          confirmTextColor: Colors.white,
          buttonColor: Colors.red,
          onConfirm: () => Get.back(result: true),
        ) ?? false;
      },
      onDismissed: (_) => todoController.deleteTodo(todo.id),
      child: ListTile(
        leading: Checkbox(
          value: todo.isCompleted,
          activeColor: Colors.teal,
          onChanged: (_) => todoController.toggleTodo(todo.id),
        ),
        title: _isEditing
            ? TextField(
                controller: _editController,
                autofocus: true,
                onSubmitted: (_) => _saveEdit(),
              )
            : Text(
                todo.title,
                style: TextStyle(
                  decoration: todo.isCompleted ? TextDecoration.lineThrough : null,
                  color: todo.isCompleted ? Colors.grey : null,
                ),
              ),
        trailing: _isEditing
            ? Row(
                mainAxisSize: MainAxisSize.min,
                children: [
                  IconButton(
                    onPressed: _saveEdit,
                    icon: const Icon(Icons.check, color: Colors.green),
                  ),
                  IconButton(
                    onPressed: () => setState(() {
                      _isEditing = false;
                      _editController.text = widget.todo.title;
                    }),
                    icon: const Icon(Icons.close, color: Colors.red),
                  ),
                ],
              )
            : IconButton(
                onPressed: () => setState(() => _isEditing = true),
                icon: const Icon(Icons.edit, color: Colors.grey, size: 20),
              ),
      ),
    );
  }
}
```

### Add Todo Widget

```dart
class AddTodoWidget extends StatefulWidget {
  const AddTodoWidget({Key? key}) : super(key: key);
  
  @override
  State<AddTodoWidget> createState() => _AddTodoWidgetState();
}

class _AddTodoWidgetState extends State<AddTodoWidget> {
  final _controller = TextEditingController();
  final _focusNode = FocusNode();
  
  final todoController = Get.find<TodoController>();
  
  @override
  void dispose() {
    _controller.dispose();
    _focusNode.dispose();
    super.dispose();
  }
  
  void _addTodo() {
    if (_controller.text.trim().isNotEmpty) {
      todoController.addTodo(_controller.text);
      _controller.clear();
      _focusNode.requestFocus();
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
            color: Colors.black.withOpacity(0.05),
            blurRadius: 4,
            offset: const Offset(0, -2),
          ),
        ],
      ),
      child: SafeArea(
        child: Row(
          children: [
            Expanded(
              child: TextField(
                controller: _controller,
                focusNode: _focusNode,
                decoration: InputDecoration(
                  hintText: 'Add new todo...',
                  border: OutlineInputBorder(
                    borderRadius: BorderRadius.circular(12),
                  ),
                  contentPadding: const EdgeInsets.symmetric(
                    horizontal: 16,
                    vertical: 12,
                  ),
                ),
                onSubmitted: (_) => _addTodo(),
                textInputAction: TextInputAction.done,
              ),
            ),
            const SizedBox(width: 8),
            ElevatedButton(
              onPressed: _addTodo,
              style: ElevatedButton.styleFrom(
                backgroundColor: Colors.teal,
                foregroundColor: Colors.white,
                padding: const EdgeInsets.all(16),
                shape: RoundedRectangleBorder(
                  borderRadius: BorderRadius.circular(12),
                ),
              ),
              child: const Icon(Icons.add),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## GetX Workers

### Ever, Once, Interval, Debounce

```dart
class WorkerController extends GetxController {
  var searchQuery = ''.obs;
  var count = 0.obs;
  var results = <String>[].obs;
  
  @override
  void onInit() {
    super.onInit();
    
    // ever: ทุกครั้งที่ value เปลี่ยน
    ever(count, (value) {
      print('Count changed to: $value');
    });
    
    // once: ครั้งแรกที่ value เปลี่ยน
    once(count, (value) {
      print('First change: $value');
    });
    
    // interval: เรียกทุก duration
    interval(
      count,
      (value) => print('Interval: $value'),
      time: const Duration(seconds: 1),
    );
    
    // debounce: เรียกหลัง stop typing
    debounce(
      searchQuery,
      (query) => _search(query),
      time: const Duration(milliseconds: 500),
    );
  }
  
  void _search(String query) {
    if (query.isEmpty) {
      results.clear();
      return;
    }
    // จำลองการค้นหา
    results.assignAll(
      ['Apple', 'Banana', 'Cherry']
          .where((item) => item.toLowerCase().contains(query.toLowerCase()))
          .toList(),
    );
  }
}
```

---

## สรุป GetX

### ข้อดี
- All-in-one (State + Navigation + DI)
- เรียนรู้ง่าย เขียนโค้ดน้อย
- Performance ดี
- Hot reload friendly

### ข้อเสีย
- Opinionated มาก
- อาจขัดแย้งกับ Flutter patterns
- Community เล็กกว่า BLoC/Provider
- Update บ่อย อาจ breaking changes

### เมื่อไหร่ใช้ GetX
- ต้องการ rapid development
- Team ชอบ simple syntax
- ต้องการ all-in-one solution
- Prototype หรือ MVP
