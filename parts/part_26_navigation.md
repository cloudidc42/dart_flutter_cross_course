# Part 26: Navigation and Routing

## บทนำ

Navigation คือการเปลี่ยนหน้าจอใน Flutter มี 2 แบบหลักคือ Navigator 1.0 (Imperative) และ Navigator 2.0 / GoRouter (Declarative)

---

## Navigator 1.0

Navigator 1.0 คือ Stack-based navigation ที่ push/pop screens

### Push และ Pop พื้นฐาน

```dart
// Push - เปิดหน้าใหม่
Navigator.of(context).push(
  MaterialPageRoute(
    builder: (context) => const DetailScreen(),
  ),
);

// Push พร้อมส่งข้อมูล
Navigator.of(context).push(
  MaterialPageRoute(
    builder: (context) => DetailScreen(product: myProduct),
  ),
);

// Pop - ปิดหน้าปัจจุบัน
Navigator.of(context).pop();

// Pop พร้อม return ค่ากลับ
Navigator.of(context).pop('result data');

// รับค่าที่ pop กลับมา
final result = await Navigator.of(context).push(
  MaterialPageRoute(
    builder: (context) => EditScreen(item: item),
  ),
);
if (result != null) {
  print('Got result: $result');
}
```

### Page Transitions

```dart
// Default transitions
MaterialPageRoute(builder: (context) => const NextScreen())    // Material slide
CupertinoPageRoute(builder: (context) => const NextScreen())   // iOS slide

// Custom transition
Navigator.of(context).push(
  PageRouteBuilder(
    pageBuilder: (context, animation, secondaryAnimation) {
      return const NextScreen();
    },
    transitionsBuilder: (context, animation, secondaryAnimation, child) {
      // Fade transition
      return FadeTransition(opacity: animation, child: child);
    },
    transitionDuration: const Duration(milliseconds: 400),
  ),
);

// Slide transition
PageRouteBuilder(
  pageBuilder: (_, __, ___) => const NextScreen(),
  transitionsBuilder: (context, animation, secondaryAnimation, child) {
    const begin = Offset(1.0, 0.0); // จากขวาไปซ้าย
    const end = Offset.zero;
    final tween = Tween(begin: begin, end: end)
        .chain(CurveTween(curve: Curves.easeInOut));
    
    return SlideTransition(
      position: animation.drive(tween),
      child: child,
    );
  },
)

// Scale transition
PageRouteBuilder(
  pageBuilder: (_, __, ___) => const NextScreen(),
  transitionsBuilder: (context, animation, secondaryAnimation, child) {
    return ScaleTransition(
      scale: Tween<double>(begin: 0.0, end: 1.0)
          .animate(CurvedAnimation(parent: animation, curve: Curves.easeInOut)),
      child: child,
    );
  },
)
```

---

## Named Routes

Named routes ใช้ string แทน widget class สำหรับ navigation

```dart
// กำหนด routes ใน MaterialApp
MaterialApp(
  initialRoute: '/',
  routes: {
    '/': (context) => const HomeScreen(),
    '/about': (context) => const AboutScreen(),
    '/settings': (context) => const SettingsScreen(),
    '/product': (context) => const ProductScreen(),
  },
)

// ใช้งาน
Navigator.pushNamed(context, '/about');
Navigator.pushNamed(context, '/settings');
Navigator.pop(context);

// Push และ replace (ไม่สามารถ pop กลับ)
Navigator.pushReplacementNamed(context, '/home');

// Push และ clear ทุก route ก่อนหน้า (ใช้ตอน logout)
Navigator.pushNamedAndRemoveUntil(
  context,
  '/login',
  (route) => false,  // ลบทุก route
);

// Pop จนถึง route ที่ต้องการ
Navigator.popUntil(
  context,
  ModalRoute.withName('/home'),
);
```

### Route Arguments

```dart
// ส่ง arguments ด้วย named routes
Navigator.pushNamed(
  context,
  '/product',
  arguments: {'productId': '123', 'source': 'home'},
);

// รับ arguments
class ProductScreen extends StatelessWidget {
  const ProductScreen({super.key});

  @override
  Widget build(BuildContext context) {
    // รับ arguments
    final args = ModalRoute.of(context)!.settings.arguments as Map<String, String>;
    final productId = args['productId'];
    
    return Scaffold(
      appBar: AppBar(title: Text('Product $productId')),
      body: ...,
    );
  }
}

// ดีกว่า: กำหนด argument type ชัดเจน
class ProductScreenArgs {
  final String productId;
  final String source;
  
  const ProductScreenArgs({required this.productId, required this.source});
}

// ส่ง
Navigator.pushNamed(
  context,
  '/product',
  arguments: ProductScreenArgs(productId: '123', source: 'home'),
);

// รับ
final args = ModalRoute.of(context)!.settings.arguments as ProductScreenArgs;
```

### onGenerateRoute

```dart
// จัดการ route แบบ dynamic
MaterialApp(
  onGenerateRoute: (settings) {
    final uri = Uri.parse(settings.name ?? '/');
    
    switch (uri.path) {
      case '/':
        return MaterialPageRoute(
          builder: (_) => const HomeScreen(),
        );
      
      case '/product':
        final id = uri.queryParameters['id'] ?? '';
        return MaterialPageRoute(
          builder: (_) => ProductScreen(productId: id),
        );
      
      default:
        return MaterialPageRoute(
          builder: (_) => const NotFoundScreen(),
        );
    }
  },
)
```

---

## WillPopScope (onPopInvoked)

ควบคุมการ pop ของ back button

```dart
// ล้าสมัยใน Flutter 3.12+
// ใช้ PopScope แทน WillPopScope

class EditScreen extends StatelessWidget {
  const EditScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return PopScope(
      // canPop: false = ปิดการ pop ทั้งหมด
      // canPop: true = pop ปกติ (default)
      canPop: false,
      onPopInvoked: (didPop) async {
        if (didPop) return; // ถ้า pop ไปแล้ว ไม่ต้องทำอะไร
        
        // แสดง dialog ยืนยันก่อน pop
        final bool? shouldPop = await showDialog<bool>(
          context: context,
          builder: (context) => AlertDialog(
            title: const Text('ออกจากหน้านี้?'),
            content: const Text('การเปลี่ยนแปลงที่ยังไม่ได้บันทึกจะหายไป'),
            actions: [
              TextButton(
                onPressed: () => Navigator.of(context).pop(false),
                child: const Text('ยกเลิก'),
              ),
              TextButton(
                onPressed: () => Navigator.of(context).pop(true),
                child: const Text('ออก'),
              ),
            ],
          ),
        );
        
        if (shouldPop == true && context.mounted) {
          Navigator.of(context).pop();
        }
      },
      child: Scaffold(
        appBar: AppBar(title: const Text('แก้ไข')),
        body: const Text('Form content...'),
      ),
    );
  }
}
```

---

## Bottom Navigation

BottomNavigationBar สำหรับหน้าหลักที่มีหลาย tab

```dart
class MainScreen extends StatefulWidget {
  const MainScreen({super.key});

  @override
  State<MainScreen> createState() => _MainScreenState();
}

class _MainScreenState extends State<MainScreen> {
  int _selectedIndex = 0;

  // เก็บ page states ด้วย IndexedStack
  static const List<Widget> _pages = [
    HomeTab(),
    SearchTab(),
    CartTab(),
    ProfileTab(),
  ];

  void _onItemTapped(int index) {
    setState(() => _selectedIndex = index);
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: IndexedStack(
        // IndexedStack คง state ของทุก tab ไว้
        // ต่างจาก pages[_selectedIndex] ที่ rebuild ทุกครั้ง
        index: _selectedIndex,
        children: _pages,
      ),
      bottomNavigationBar: NavigationBar(
        // Material 3 style
        selectedIndex: _selectedIndex,
        onDestinationSelected: _onItemTapped,
        destinations: const [
          NavigationDestination(
            icon: Icon(Icons.home_outlined),
            selectedIcon: Icon(Icons.home),
            label: 'หน้าหลัก',
          ),
          NavigationDestination(
            icon: Icon(Icons.search_outlined),
            selectedIcon: Icon(Icons.search),
            label: 'ค้นหา',
          ),
          NavigationDestination(
            icon: Badge(
              label: Text('3'),
              child: Icon(Icons.shopping_cart_outlined),
            ),
            selectedIcon: Badge(
              label: Text('3'),
              child: Icon(Icons.shopping_cart),
            ),
            label: 'ตะกร้า',
          ),
          NavigationDestination(
            icon: Icon(Icons.person_outlined),
            selectedIcon: Icon(Icons.person),
            label: 'โปรไฟล์',
          ),
        ],
      ),
    );
  }
}
```

### BottomNavigationBar แบบเก่า (Material 2)

```dart
BottomNavigationBar(
  currentIndex: _selectedIndex,
  onTap: _onItemTapped,
  type: BottomNavigationBarType.fixed,
  selectedItemColor: Theme.of(context).colorScheme.primary,
  unselectedItemColor: Colors.grey,
  items: const [
    BottomNavigationBarItem(
      icon: Icon(Icons.home_outlined),
      activeIcon: Icon(Icons.home),
      label: 'หน้าหลัก',
    ),
    BottomNavigationBarItem(
      icon: Icon(Icons.search),
      label: 'ค้นหา',
    ),
    BottomNavigationBarItem(
      icon: Icon(Icons.person_outlined),
      activeIcon: Icon(Icons.person),
      label: 'โปรไฟล์',
    ),
  ],
)
```

---

## Drawer Navigation

Drawer สำหรับ side menu

```dart
Scaffold(
  appBar: AppBar(
    title: const Text('My App'),
    // hamburger button สร้างอัตโนมัติถ้ามี Drawer
  ),
  drawer: _buildDrawer(context),
  endDrawer: _buildEndDrawer(context), // ด้านขวา
  body: const Center(child: Text('Content')),
)

Widget _buildDrawer(BuildContext context) {
  final theme = Theme.of(context);
  
  return Drawer(
    child: ListView(
      padding: EdgeInsets.zero,
      children: [
        // Header
        UserAccountsDrawerHeader(
          decoration: BoxDecoration(
            gradient: LinearGradient(
              colors: [
                theme.colorScheme.primary,
                theme.colorScheme.secondary,
              ],
            ),
          ),
          accountName: const Text('John Doe'),
          accountEmail: const Text('john@example.com'),
          currentAccountPicture: const CircleAvatar(
            backgroundImage: NetworkImage('https://picsum.photos/seed/avatar/100/100'),
          ),
          otherAccountsPictures: [
            CircleAvatar(
              backgroundColor: theme.colorScheme.surface,
              child: const Icon(Icons.add),
            ),
          ],
        ),
        
        // Menu items
        ListTile(
          leading: const Icon(Icons.home),
          title: const Text('หน้าหลัก'),
          selected: true,
          onTap: () {
            Navigator.pop(context); // ปิด drawer
            // Navigate to home
          },
        ),
        
        ListTile(
          leading: const Icon(Icons.shopping_bag),
          title: const Text('คำสั่งซื้อ'),
          trailing: const Badge(label: Text('3')),
          onTap: () {
            Navigator.pop(context);
            Navigator.pushNamed(context, '/orders');
          },
        ),
        
        ListTile(
          leading: const Icon(Icons.favorite),
          title: const Text('รายการโปรด'),
          onTap: () {
            Navigator.pop(context);
          },
        ),
        
        const Divider(),
        
        ListTile(
          leading: const Icon(Icons.settings),
          title: const Text('ตั้งค่า'),
          onTap: () {
            Navigator.pop(context);
            Navigator.pushNamed(context, '/settings');
          },
        ),
        
        ListTile(
          leading: const Icon(Icons.help),
          title: const Text('ช่วยเหลือ'),
          onTap: () {},
        ),
        
        const Divider(),
        
        ListTile(
          leading: const Icon(Icons.logout, color: Colors.red),
          title: const Text('ออกจากระบบ', style: TextStyle(color: Colors.red)),
          onTap: () {
            showDialog(
              context: context,
              builder: (context) => AlertDialog(
                title: const Text('ออกจากระบบ?'),
                actions: [
                  TextButton(
                    onPressed: () => Navigator.pop(context),
                    child: const Text('ยกเลิก'),
                  ),
                  TextButton(
                    onPressed: () {
                      Navigator.pop(context); // close dialog
                      Navigator.pop(context); // close drawer
                      Navigator.pushNamedAndRemoveUntil(
                        context, '/login', (route) => false,
                      );
                    },
                    child: const Text('ออก',
                      style: TextStyle(color: Colors.red)),
                  ),
                ],
              ),
            );
          },
        ),
      ],
    ),
  );
}
```

---

## Workshop: Multi-Screen Todo App

สร้าง Todo App ที่มีหลายหน้าจอพร้อม navigation

### lib/main.dart

```dart
import 'package:flutter/material.dart';
import 'screens/splash_screen.dart';
import 'screens/home_screen.dart';
import 'screens/todo_list_screen.dart';
import 'screens/add_todo_screen.dart';
import 'screens/todo_detail_screen.dart';
import 'screens/settings_screen.dart';
import 'models/todo.dart';

void main() {
  runApp(const TodoApp());
}

class TodoApp extends StatelessWidget {
  const TodoApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Multi-Screen Todo',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
        useMaterial3: true,
      ),
      initialRoute: '/splash',
      onGenerateRoute: (settings) {
        switch (settings.name) {
          case '/splash':
            return MaterialPageRoute(
              builder: (_) => const SplashScreen(),
            );
          case '/':
          case '/home':
            return MaterialPageRoute(
              builder: (_) => const HomeScreen(),
            );
          case '/todos':
            return MaterialPageRoute(
              builder: (_) => const TodoListScreen(),
            );
          case '/todos/add':
            return MaterialPageRoute(
              builder: (_) => const AddTodoScreen(),
              fullscreenDialog: true,
            );
          case '/todos/detail':
            final todo = settings.arguments as Todo;
            return MaterialPageRoute(
              builder: (_) => TodoDetailScreen(todo: todo),
            );
          case '/settings':
            return MaterialPageRoute(
              builder: (_) => const SettingsScreen(),
            );
          default:
            return MaterialPageRoute(
              builder: (_) => const NotFoundScreen(),
            );
        }
      },
    );
  }
}

class NotFoundScreen extends StatelessWidget {
  const NotFoundScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            const Icon(Icons.error_outline, size: 80, color: Colors.grey),
            const SizedBox(height: 16),
            const Text('ไม่พบหน้าที่ต้องการ', style: TextStyle(fontSize: 20)),
            const SizedBox(height: 16),
            ElevatedButton(
              onPressed: () => Navigator.pushNamedAndRemoveUntil(
                context, '/home', (route) => false),
              child: const Text('กลับหน้าหลัก'),
            ),
          ],
        ),
      ),
    );
  }
}
```

### lib/models/todo.dart

```dart
class Todo {
  final String id;
  final String title;
  final String? description;
  final String category;
  final bool isCompleted;
  final DateTime createdAt;
  final DateTime? dueDate;
  final int priority; // 1-3 (low-high)

  const Todo({
    required this.id,
    required this.title,
    this.description,
    required this.category,
    this.isCompleted = false,
    required this.createdAt,
    this.dueDate,
    this.priority = 1,
  });

  Todo copyWith({
    String? title,
    String? description,
    String? category,
    bool? isCompleted,
    DateTime? dueDate,
    int? priority,
  }) {
    return Todo(
      id: id,
      title: title ?? this.title,
      description: description ?? this.description,
      category: category ?? this.category,
      isCompleted: isCompleted ?? this.isCompleted,
      createdAt: createdAt,
      dueDate: dueDate ?? this.dueDate,
      priority: priority ?? this.priority,
    );
  }

  String get priorityLabel => ['ต่ำ', 'ปานกลาง', 'สูง'][priority - 1];
  Color get priorityColor => [Colors.green, Colors.orange, Colors.red][priority - 1];
}
```

### lib/screens/splash_screen.dart

```dart
import 'package:flutter/material.dart';

class SplashScreen extends StatefulWidget {
  const SplashScreen({super.key});

  @override
  State<SplashScreen> createState() => _SplashScreenState();
}

class _SplashScreenState extends State<SplashScreen>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _fadeAnimation;
  late Animation<double> _scaleAnimation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 1500),
    );

    _fadeAnimation = Tween<double>(begin: 0, end: 1).animate(
      CurvedAnimation(parent: _controller, curve: const Interval(0, 0.6)),
    );

    _scaleAnimation = Tween<double>(begin: 0.5, end: 1).animate(
      CurvedAnimation(
          parent: _controller, curve: const Interval(0, 0.6, curve: Curves.elasticOut)),
    );

    _controller.forward();

    // Navigate to home after 2.5 seconds
    Future.delayed(const Duration(milliseconds: 2500), () {
      if (mounted) {
        Navigator.pushReplacementNamed(context, '/home');
      }
    });
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Theme.of(context).colorScheme.primary,
      body: Center(
        child: AnimatedBuilder(
          animation: _controller,
          builder: (context, child) {
            return FadeTransition(
              opacity: _fadeAnimation,
              child: ScaleTransition(
                scale: _scaleAnimation,
                child: Column(
                  mainAxisAlignment: MainAxisAlignment.center,
                  children: [
                    Container(
                      width: 100,
                      height: 100,
                      decoration: BoxDecoration(
                        color: Colors.white,
                        borderRadius: BorderRadius.circular(24),
                      ),
                      child: const Icon(Icons.check_circle,
                          size: 60, color: Colors.teal),
                    ),
                    const SizedBox(height: 24),
                    const Text(
                      'Todo App',
                      style: TextStyle(
                        color: Colors.white,
                        fontSize: 32,
                        fontWeight: FontWeight.bold,
                      ),
                    ),
                    const SizedBox(height: 8),
                    const Text(
                      'จัดการงานของคุณให้ง่ายขึ้น',
                      style: TextStyle(color: Colors.white70, fontSize: 16),
                    ),
                  ],
                ),
              ),
            );
          },
        ),
      ),
    );
  }
}
```

### lib/screens/home_screen.dart

```dart
import 'package:flutter/material.dart';

class HomeScreen extends StatefulWidget {
  const HomeScreen({super.key});

  @override
  State<HomeScreen> createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> {
  int _selectedIndex = 0;

  static const List<Widget> _tabs = [
    TodoListScreen(),
    // StatsScreen(), // สถิติ
    // HistoryScreen(), // ประวัติ
  ];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('📋 Todo App'),
        actions: [
          IconButton(
            icon: const Icon(Icons.settings),
            onPressed: () => Navigator.pushNamed(context, '/settings'),
          ),
        ],
      ),
      drawer: _buildDrawer(context),
      body: const TodoListScreen(),
      floatingActionButton: FloatingActionButton.extended(
        onPressed: () async {
          final result = await Navigator.pushNamed(context, '/todos/add');
          if (result != null) {
            ScaffoldMessenger.of(context).showSnackBar(
              const SnackBar(content: Text('เพิ่ม Todo แล้ว')),
            );
          }
        },
        icon: const Icon(Icons.add),
        label: const Text('เพิ่ม Todo'),
      ),
    );
  }

  Widget _buildDrawer(BuildContext context) {
    return Drawer(
      child: ListView(
        padding: EdgeInsets.zero,
        children: [
          DrawerHeader(
            decoration: BoxDecoration(
              color: Theme.of(context).colorScheme.primary,
            ),
            child: const Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              mainAxisAlignment: MainAxisAlignment.end,
              children: [
                CircleAvatar(
                  radius: 30,
                  backgroundColor: Colors.white,
                  child: Icon(Icons.person, size: 36, color: Colors.teal),
                ),
                SizedBox(height: 8),
                Text('John Doe',
                  style: TextStyle(color: Colors.white, fontSize: 18)),
                Text('john@example.com',
                  style: TextStyle(color: Colors.white70, fontSize: 12)),
              ],
            ),
          ),
          ListTile(
            leading: const Icon(Icons.list),
            title: const Text('Todo ทั้งหมด'),
            selected: _selectedIndex == 0,
            onTap: () {
              setState(() => _selectedIndex = 0);
              Navigator.pop(context);
            },
          ),
          ListTile(
            leading: const Icon(Icons.today),
            title: const Text('วันนี้'),
            onTap: () => Navigator.pop(context),
          ),
          ListTile(
            leading: const Icon(Icons.flag),
            title: const Text('สำคัญ'),
            onTap: () => Navigator.pop(context),
          ),
          const Divider(),
          ListTile(
            leading: const Icon(Icons.settings),
            title: const Text('ตั้งค่า'),
            onTap: () {
              Navigator.pop(context);
              Navigator.pushNamed(context, '/settings');
            },
          ),
        ],
      ),
    );
  }
}

// Placeholder screens
class TodoListScreen extends StatelessWidget {
  const TodoListScreen({super.key});
  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: 5,
      itemBuilder: (context, index) => ListTile(
        leading: const Icon(Icons.check_box_outline_blank),
        title: Text('Todo item ${index + 1}'),
        subtitle: const Text('กดเพื่อดูรายละเอียด'),
        trailing: const Icon(Icons.chevron_right),
        onTap: () => Navigator.pushNamed(
          context,
          '/todos/detail',
          arguments: Todo(
            id: '$index',
            title: 'Todo item ${index + 1}',
            category: 'งาน',
            createdAt: DateTime.now(),
          ),
        ),
      ),
    );
  }
}

class AddTodoScreen extends StatelessWidget {
  const AddTodoScreen({super.key});
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('เพิ่ม Todo')),
      body: Center(
        child: ElevatedButton(
          onPressed: () => Navigator.pop(context, 'new_todo'),
          child: const Text('บันทึก (ส่งค่ากลับ)'),
        ),
      ),
    );
  }
}

class TodoDetailScreen extends StatelessWidget {
  final Todo todo;
  const TodoDetailScreen({super.key, required this.todo});
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text(todo.title)),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text('ID: ${todo.id}'),
            Text('หมวดหมู่: ${todo.category}'),
            Text('สร้างเมื่อ: ${todo.createdAt}'),
          ],
        ),
      ),
    );
  }
}

class SettingsScreen extends StatelessWidget {
  const SettingsScreen({super.key});
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('ตั้งค่า')),
      body: const Center(child: Text('Settings')),
    );
  }
}

// Import Todo model
import '../models/todo.dart';
```

---

## Navigator 2.0 Basics (GoRouter)

GoRouter เป็น package ยอดนิยมสำหรับ declarative routing

```dart
// pubspec.yaml
// dependencies:
//   go_router: ^13.0.0

import 'package:go_router/go_router.dart';

// Setup router
final _router = GoRouter(
  initialLocation: '/',
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => const HomeScreen(),
    ),
    GoRoute(
      path: '/todos',
      builder: (context, state) => const TodoListScreen(),
      routes: [
        GoRoute(
          path: ':id',
          builder: (context, state) {
            final id = state.pathParameters['id']!;
            return TodoDetailScreen(todoId: id);
          },
        ),
        GoRoute(
          path: 'add',
          builder: (context, state) => const AddTodoScreen(),
        ),
      ],
    ),
    GoRoute(
      path: '/settings',
      builder: (context, state) => const SettingsScreen(),
    ),
  ],
);

// ใช้ router
MaterialApp.router(
  routerConfig: _router,
)

// Navigate
context.go('/todos');           // Go (replace history)
context.push('/todos/123');     // Push
context.pop();                  // Pop

// ส่ง query params
context.push('/todos?filter=active');

// รับ query params
final filter = state.uri.queryParameters['filter'];
```

---

## สรุปบทที่ 26

ในบทนี้เราได้เรียนรู้:

1. **Navigator push/pop**: การเปลี่ยนหน้าพื้นฐาน
2. **Named Routes**: การใช้ string route
3. **Route Arguments**: การส่งข้อมูลระหว่างหน้า
4. **WillPopScope/PopScope**: ควบคุม back button
5. **Bottom Navigation**: TabBar navigation
6. **Drawer Navigation**: Side menu
7. **Workshop**: Multi-screen Todo App

บทต่อไปเราจะเรียน Forms และ User Input
