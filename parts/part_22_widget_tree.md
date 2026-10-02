# Part 22: Widget Tree and Flutter Architecture

## บทนำ

ใน Flutter ทุกอย่างคือ Widget! Widget คือ building block พื้นฐานของ UI ใน Flutter ทำความเข้าใจ Widget Tree จะช่วยให้เราเขียนโค้ดได้ดีขึ้นและแก้ปัญหาได้เร็วขึ้น

---

## StatelessWidget

StatelessWidget คือ widget ที่ **ไม่มี state ภายใน** (immutable) เหมาะสำหรับ UI ที่แสดงผลตาม input เท่านั้น

```dart
// StatelessWidget พื้นฐาน
class GreetingWidget extends StatelessWidget {
  // final fields เท่านั้น (immutable)
  final String name;
  final Color color;
  
  // const constructor แนะนำเสมอ
  const GreetingWidget({
    super.key,
    required this.name,
    this.color = Colors.black,
  });

  @override
  Widget build(BuildContext context) {
    // build() เรียกทุกครั้งที่ parent rebuild
    return Text(
      'สวัสดี $name',
      style: TextStyle(color: color),
    );
  }
}

// การใช้งาน
const GreetingWidget(name: 'Flutter', color: Colors.blue)
```

### เมื่อไรใช้ StatelessWidget

- แสดงข้อมูลที่ได้รับมาจาก parent เท่านั้น
- ไม่มีการ interact หรือ state เปลี่ยนภายใน widget นี้
- UI ไม่เปลี่ยนเองตามเวลา

```dart
// ตัวอย่าง - Product Card (แสดงข้อมูลอย่างเดียว)
class ProductCard extends StatelessWidget {
  final String name;
  final double price;
  final String imageUrl;
  final VoidCallback? onTap;
  
  const ProductCard({
    super.key,
    required this.name,
    required this.price,
    required this.imageUrl,
    this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    return Card(
      child: InkWell(
        onTap: onTap,
        child: Column(
          children: [
            Image.network(imageUrl),
            Padding(
              padding: const EdgeInsets.all(8.0),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(name, style: const TextStyle(fontWeight: FontWeight.bold)),
                  Text('฿${price.toStringAsFixed(2)}'),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## StatefulWidget

StatefulWidget คือ widget ที่ **มี state ภายใน** ที่เปลี่ยนแปลงได้ เมื่อ state เปลี่ยน widget จะ rebuild

```dart
// StatefulWidget มี 2 class:
// 1. Widget class (immutable)
// 2. State class (mutable)

class CounterWidget extends StatefulWidget {
  final int initialCount;
  
  const CounterWidget({
    super.key,
    this.initialCount = 0,
  });

  @override
  State<CounterWidget> createState() => _CounterWidgetState();
}

// State class มี underscore prefix (private)
class _CounterWidgetState extends State<CounterWidget> {
  // state variables
  late int _count;
  
  @override
  void initState() {
    super.initState();
    // เข้าถึง widget properties ผ่าน widget.xxx
    _count = widget.initialCount;
  }
  
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
  
  @override
  Widget build(BuildContext context) {
    return Row(
      mainAxisSize: MainAxisSize.min,
      children: [
        IconButton(
          icon: const Icon(Icons.remove),
          onPressed: _count > 0 ? _decrement : null,
        ),
        Text(
          '$_count',
          style: const TextStyle(fontSize: 24, fontWeight: FontWeight.bold),
        ),
        IconButton(
          icon: const Icon(Icons.add),
          onPressed: _increment,
        ),
      ],
    );
  }
}
```

### เมื่อไรใช้ StatefulWidget

- ต้องการ track user interaction (tap, swipe, input)
- ข้อมูลเปลี่ยนตามเวลา (timer, animation)
- Form input ที่ต้องเก็บค่า
- Loading state

---

## Widget Lifecycle

### StatelessWidget Lifecycle

```
new widget props → build() → displayed
```

StatelessWidget มี lifecycle ที่เรียบง่าย:
1. Constructor ถูกเรียก
2. `build()` ถูกเรียก
3. Widget แสดงผล
4. เมื่อ parent rebuild, `build()` ถูกเรียกใหม่

### StatefulWidget Lifecycle

```
createState() → initState() → didChangeDependencies() → build() 
    → [didUpdateWidget()] → [setState() → build()]
    → deactivate() → dispose()
```

```dart
class LifecycleDemo extends StatefulWidget {
  final String title;
  
  const LifecycleDemo({super.key, required this.title});

  @override
  State<LifecycleDemo> createState() => _LifecycleDemoState();
}

class _LifecycleDemoState extends State<LifecycleDemo> {
  
  // 1. เรียกครั้งเดียวตอน widget ถูกสร้าง
  // ใช้สำหรับ: initialization, subscribe to streams
  @override
  void initState() {
    super.initState(); // ต้องเรียก super เสมอ
    print('initState: ${widget.title}');
    // ห้ามใช้ context ที่นี่อย่างเต็มที่
    // (BuildContext ยังไม่พร้อม)
  }

  // 2. เรียกหลัง initState และเมื่อ dependency เปลี่ยน
  // dependency คือ InheritedWidget ที่ widget นี้ใช้
  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    print('didChangeDependencies');
    // ปลอดภัยแล้วที่จะใช้ context เต็มรูปแบบ
    // (Theme.of(context), MediaQuery.of(context), etc.)
  }

  // 3. เรียกทุกครั้งที่ต้อง render
  @override
  Widget build(BuildContext context) {
    print('build: ${widget.title}');
    return Scaffold(
      appBar: AppBar(title: Text(widget.title)),
      body: const Center(child: Text('Lifecycle Demo')),
    );
  }

  // 4. เรียกเมื่อ parent ส่ง widget ใหม่มา (config เปลี่ยน)
  @override
  void didUpdateWidget(LifecycleDemo oldWidget) {
    super.didUpdateWidget(oldWidget);
    print('didUpdateWidget: ${oldWidget.title} → ${widget.title}');
    if (oldWidget.title != widget.title) {
      // ทำงานเมื่อ title เปลี่ยน
    }
  }

  // 5. เรียกเมื่อ widget ถูก remove จาก tree ชั่วคราว
  @override
  void deactivate() {
    super.deactivate();
    print('deactivate');
  }

  // 6. เรียกครั้งสุดท้ายก่อน widget ถูกทำลาย
  // ใช้สำหรับ: cancel subscriptions, dispose controllers
  @override
  void dispose() {
    print('dispose');
    super.dispose(); // ต้องเรียก super ที่ท้าย
  }
}
```

### ตัวอย่างการใช้ lifecycle จริง

```dart
class TimerWidget extends StatefulWidget {
  const TimerWidget({super.key});

  @override
  State<TimerWidget> createState() => _TimerWidgetState();
}

class _TimerWidgetState extends State<TimerWidget> {
  late DateTime _currentTime;
  // Timer ต้อง dispose
  late final _timer;
  
  // Controller ต้อง dispose
  late final TextEditingController _controller;
  
  // AnimationController ต้อง dispose
  late final AnimationController _animController;

  @override
  void initState() {
    super.initState();
    _currentTime = DateTime.now();
    _controller = TextEditingController();
    
    // Start timer
    _timer = Timer.periodic(const Duration(seconds: 1), (timer) {
      if (mounted) { // ตรวจสอบว่า widget ยังอยู่ใน tree
        setState(() {
          _currentTime = DateTime.now();
        });
      }
    });
  }

  @override
  void dispose() {
    _timer.cancel();        // Cancel timer
    _controller.dispose();  // Dispose controller
    // _animController.dispose(); // Dispose animation
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Text(
      '${_currentTime.hour}:${_currentTime.minute}:${_currentTime.second}',
      style: const TextStyle(fontSize: 48),
    );
  }
}
```

---

## BuildContext

`BuildContext` คือ handle สำหรับ location ของ widget ใน widget tree เราใช้มันเพื่อ:
- เข้าถึง inherited data (Theme, MediaQuery, etc.)
- Navigate
- Show dialogs/snackbars

```dart
@override
Widget build(BuildContext context) {
  // 1. Theme
  final theme = Theme.of(context);
  final colorScheme = theme.colorScheme;
  
  // 2. MediaQuery - ขนาดหน้าจอ
  final mediaQuery = MediaQuery.of(context);
  final screenWidth = mediaQuery.size.width;
  final screenHeight = mediaQuery.size.height;
  final isLandscape = mediaQuery.orientation == Orientation.landscape;
  
  // 3. Navigator - navigation
  // Navigator.of(context).push(...)
  
  // 4. ScaffoldMessenger - snackbar
  // ScaffoldMessenger.of(context).showSnackBar(...)
  
  // 5. Navigator
  // Navigator.of(context).push(...)
  
  return Container(
    width: screenWidth * 0.8,
    color: colorScheme.primary,
  );
}
```

### BuildContext และ async

ระวัง: ไม่ควรใช้ context หลัง async gap โดยไม่ check `mounted`

```dart
// BAD - อาจ crash ถ้า widget ถูก dispose ระหว่าง await
Future<void> _loadData() async {
  final data = await fetchData(); // async operation
  // Widget อาจถูก dispose แล้ว!
  Navigator.of(context).push(...); // DANGEROUS
}

// GOOD - check mounted ก่อน
Future<void> _loadData() async {
  final data = await fetchData();
  if (!mounted) return; // Widget ถูก dispose แล้ว
  if (mounted) {
    Navigator.of(context).push(...); // SAFE
  }
}

// หรือใช้ context ก่อน async
Future<void> _loadAndNavigate() async {
  // เก็บ navigator ก่อน
  final navigator = Navigator.of(context);
  final data = await fetchData();
  navigator.push(...); // SAFE (ไม่ใช้ context โดยตรง)
}
```

---

## InheritedWidget

InheritedWidget เป็นกลไกที่ Flutter ใช้ share data ตลอด widget tree โดยไม่ต้องส่งผ่าน constructor ทุกชั้น

```dart
// สร้าง InheritedWidget
class UserInheritedWidget extends InheritedWidget {
  final User user;
  final VoidCallback onLogout;
  
  const UserInheritedWidget({
    super.key,
    required this.user,
    required this.onLogout,
    required super.child,
  });
  
  // Static method สำหรับ access
  static UserInheritedWidget of(BuildContext context) {
    final widget = context.dependOnInheritedWidgetOfExactType<UserInheritedWidget>();
    assert(widget != null, 'UserInheritedWidget not found in tree');
    return widget!;
  }
  
  // อาจ return null ถ้าไม่แน่ใจว่ามีอยู่ใน tree
  static UserInheritedWidget? maybeOf(BuildContext context) {
    return context.dependOnInheritedWidgetOfExactType<UserInheritedWidget>();
  }
  
  // เมื่อ widget rebuild, ส่วนไหนควร notify descendants
  @override
  bool updateShouldNotify(UserInheritedWidget oldWidget) {
    return user != oldWidget.user;
  }
}

// ใช้งาน
class App extends StatelessWidget {
  const App({super.key});
  
  @override
  Widget build(BuildContext context) {
    return UserInheritedWidget(
      user: User(name: 'John', email: 'john@example.com'),
      onLogout: () => print('Logged out'),
      child: MaterialApp(
        home: const HomeScreen(),
      ),
    );
  }
}

// ดึงข้อมูล
class ProfileWidget extends StatelessWidget {
  const ProfileWidget({super.key});
  
  @override
  Widget build(BuildContext context) {
    // Widget นี้จะ rebuild เมื่อ user เปลี่ยน
    final userData = UserInheritedWidget.of(context);
    
    return Column(
      children: [
        Text(userData.user.name),
        ElevatedButton(
          onPressed: userData.onLogout,
          child: const Text('Logout'),
        ),
      ],
    );
  }
}
```

### InheritedWidget ที่ใช้บ่อยใน Flutter

```dart
// Theme
Theme.of(context)  // → ThemeData

// MediaQuery
MediaQuery.of(context)  // → MediaQueryData

// Navigator
Navigator.of(context)  // → NavigatorState

// ScaffoldMessenger
ScaffoldMessenger.of(context)  // → ScaffoldMessengerState

// Form
Form.of(context)  // → FormState (ใน context ของ Form)

// DefaultTabController
DefaultTabController.of(context)  // → TabController
```

---

## Keys ใน Flutter

Keys ช่วย Flutter ระบุตัวตนของ widget ใน tree โดยเฉพาะเมื่อมีการย้าย/เรียงลำดับ widget

### ทำไมต้องใช้ Keys?

```dart
// ปัญหา: ไม่มี Key
class _MyListState extends State<MyList> {
  final List<String> items = ['A', 'B', 'C'];
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: items.map((item) => 
        // ไม่มี Key - Flutter ไม่รู้ว่า item ไหนคือ item ไหน
        ColorBox(label: item)
      ).toList(),
    );
  }
}

// แก้ไข: ใช้ ValueKey
class _MyListState extends State<MyList> {
  final List<String> items = ['A', 'B', 'C'];
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: items.map((item) => 
        // ValueKey ช่วย Flutter track widget แต่ละตัว
        ColorBox(key: ValueKey(item), label: item)
      ).toList(),
    );
  }
}
```

### ประเภทของ Keys

```dart
// 1. ValueKey - ใช้ค่า (string, int, etc.) เป็น key
ListView.builder(
  itemBuilder: (context, index) {
    return ListTile(
      key: ValueKey(items[index].id), // ใช้ unique ID
      title: Text(items[index].name),
    );
  },
)

// 2. ObjectKey - ใช้ object reference เป็น key
final user = User(id: 1, name: 'John');
UserCard(key: ObjectKey(user))

// 3. UniqueKey - สร้าง key ใหม่ทุกครั้ง (force rebuild)
// ระวัง: ทำให้ rebuild ทุกครั้งเสมอ
AnimatedSwitcher(
  child: SomeWidget(key: UniqueKey()), // เปลี่ยน key = rebuild + animate
)

// 4. GlobalKey - access state จากที่อื่น
final GlobalKey<FormState> _formKey = GlobalKey<FormState>();

Form(
  key: _formKey,
  child: ...,
)

// Access form state จากที่อื่น
_formKey.currentState?.validate()
_formKey.currentState?.save()

// 5. GlobalKey สำหรับ Scaffold
final GlobalKey<ScaffoldState> _scaffoldKey = GlobalKey<ScaffoldState>();

Scaffold(
  key: _scaffoldKey,
  drawer: Drawer(...),
)

// เปิด drawer จากที่อื่น
_scaffoldKey.currentState?.openDrawer()

// 6. PageStorageKey - บันทึก scroll position
ListView(
  key: PageStorageKey('my-list'),
  ...
)
```

### Key Best Practices

```dart
// ✅ ใช้ Key เมื่อ widget มีลำดับ/ตำแหน่งที่เปลี่ยนได้
children: items.map((item) => 
  MyWidget(key: ValueKey(item.id))
).toList()

// ✅ ใช้ GlobalKey เมื่อต้องการ access state จาก parent
final GlobalKey<MyFormState> _formKey = GlobalKey<MyFormState>();
MyForm(key: _formKey)

// ❌ อย่าสร้าง Key ใน build() โดยไม่จำเป็น
// bad
Widget build(BuildContext context) {
  return MyWidget(key: UniqueKey()); // rebuild ทุกครั้ง!
}

// ✅ สร้าง Key เป็น field ของ State
class _MyState extends State<My> {
  final _key = UniqueKey(); // สร้างครั้งเดียว
  
  Widget build(BuildContext context) {
    return MyWidget(key: _key);
  }
}
```

---

## Widget Tree Visualization

มาดูตัวอย่างการ visualize widget tree:

```dart
// Widget tree นี้:
MaterialApp(
  home: Scaffold(
    appBar: AppBar(
      title: Text('My App'),
    ),
    body: Column(
      children: [
        Text('Item 1'),
        Row(
          children: [
            Icon(Icons.star),
            Text('Rating'),
          ],
        ),
        ElevatedButton(
          onPressed: () {},
          child: Text('Click'),
        ),
      ],
    ),
  ),
)

// มี widget tree ดังนี้:
// MaterialApp
//   └── Scaffold
//       ├── AppBar
//       │   └── Text('My App')
//       └── Column
//           ├── Text('Item 1')
//           ├── Row
//           │   ├── Icon(Icons.star)
//           │   └── Text('Rating')
//           └── ElevatedButton
//               └── Text('Click')
```

---

## Workshop: Widget Tree Visualization

เราจะสร้างแอพที่แสดง widget tree แบบ interactive

### lib/main.dart

```dart
import 'package:flutter/material.dart';
import 'widget_tree_screen.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Widget Tree Visualizer',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
        useMaterial3: true,
      ),
      home: const WidgetTreeScreen(),
    );
  }
}
```

### lib/widget_tree_screen.dart

```dart
import 'package:flutter/material.dart';

// Model for tree nodes
class TreeNode {
  final String name;
  final String? description;
  final Color color;
  final List<TreeNode> children;
  bool isExpanded;
  
  TreeNode({
    required this.name,
    this.description,
    required this.color,
    this.children = const [],
    this.isExpanded = true,
  });
}

class WidgetTreeScreen extends StatefulWidget {
  const WidgetTreeScreen({super.key});

  @override
  State<WidgetTreeScreen> createState() => _WidgetTreeScreenState();
}

class _WidgetTreeScreenState extends State<WidgetTreeScreen> {
  late TreeNode _rootNode;
  int _selectedTab = 0;

  @override
  void initState() {
    super.initState();
    _rootNode = _buildSampleTree();
  }

  TreeNode _buildSampleTree() {
    return TreeNode(
      name: 'MaterialApp',
      description: 'Root of the app',
      color: Colors.blue,
      children: [
        TreeNode(
          name: 'Scaffold',
          description: 'Basic screen structure',
          color: Colors.green,
          children: [
            TreeNode(
              name: 'AppBar',
              description: 'Top navigation bar',
              color: Colors.orange,
              children: [
                TreeNode(
                  name: 'Text("My App")',
                  description: 'AppBar title',
                  color: Colors.deepOrange,
                ),
              ],
            ),
            TreeNode(
              name: 'Column',
              description: 'Vertical layout',
              color: Colors.purple,
              children: [
                TreeNode(
                  name: 'Text("Hello")',
                  description: 'Simple text widget',
                  color: Colors.pink,
                ),
                TreeNode(
                  name: 'Row',
                  description: 'Horizontal layout',
                  color: Colors.teal,
                  children: [
                    TreeNode(
                      name: 'Icon',
                      description: 'Material icon',
                      color: Colors.cyan,
                    ),
                    TreeNode(
                      name: 'Text("Icon label")',
                      description: 'Icon text',
                      color: Colors.lightBlue,
                    ),
                  ],
                ),
                TreeNode(
                  name: 'ElevatedButton',
                  description: 'Clickable button',
                  color: Colors.indigo,
                  children: [
                    TreeNode(
                      name: 'Text("Click me")',
                      description: 'Button label',
                      color: Colors.blue,
                    ),
                  ],
                ),
              ],
            ),
          ],
        ),
      ],
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Widget Tree Visualizer'),
        bottom: TabBar(
          tabs: const [
            Tab(icon: Icon(Icons.account_tree), text: 'Tree View'),
            Tab(icon: Icon(Icons.code), text: 'Code View'),
            Tab(icon: Icon(Icons.info), text: 'Theory'),
          ],
          onTap: (index) => setState(() => _selectedTab = index),
        ),
      ),
      body: IndexedStack(
        index: _selectedTab,
        children: [
          _buildTreeView(),
          _buildCodeView(),
          _buildTheoryView(),
        ],
      ),
    );
  }

  Widget _buildTreeView() {
    return SingleChildScrollView(
      padding: const EdgeInsets.all(16),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          const Text(
            'Widget Tree',
            style: TextStyle(fontSize: 20, fontWeight: FontWeight.bold),
          ),
          const SizedBox(height: 8),
          const Text(
            'แตะที่ node เพื่อดูรายละเอียด / แตะลูกศรเพื่อ expand/collapse',
            style: TextStyle(color: Colors.grey),
          ),
          const SizedBox(height: 16),
          TreeNodeWidget(node: _rootNode, depth: 0),
        ],
      ),
    );
  }

  Widget _buildCodeView() {
    return SingleChildScrollView(
      padding: const EdgeInsets.all(16),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          const Text(
            'Widget Tree Code',
            style: TextStyle(fontSize: 20, fontWeight: FontWeight.bold),
          ),
          const SizedBox(height: 16),
          Container(
            width: double.infinity,
            padding: const EdgeInsets.all(16),
            decoration: BoxDecoration(
              color: const Color(0xFF1E1E1E),
              borderRadius: BorderRadius.circular(12),
            ),
            child: const SelectableText(
              '''MaterialApp(
  home: Scaffold(
    appBar: AppBar(
      title: Text("My App"),
    ),
    body: Column(
      children: [
        Text("Hello"),
        Row(
          children: [
            Icon(Icons.star),
            Text("Icon label"),
          ],
        ),
        ElevatedButton(
          onPressed: () {},
          child: Text("Click me"),
        ),
      ],
    ),
  ),
)''',
              style: TextStyle(
                color: Colors.white,
                fontFamily: 'monospace',
                fontSize: 13,
              ),
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildTheoryView() {
    return SingleChildScrollView(
      padding: const EdgeInsets.all(16),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          _buildTheorySection(
            '🌳 Widget Tree',
            'Widget Tree คือโครงสร้างที่เราเขียนใน Dart code '
            'Widget เป็น immutable (ไม่เปลี่ยนแปลง) เมื่อ state เปลี่ยน '
            'Flutter จะสร้าง Widget ใหม่ทั้งหมด',
            Colors.blue,
          ),
          _buildTheorySection(
            '🔧 Element Tree',
            'Element Tree คือ "live" tree ที่ Flutter สร้างและจัดการ '
            'Element track ว่า Widget ไหนอยู่ที่ position ไหน '
            'และเก็บ State ของ StatefulWidget',
            Colors.green,
          ),
          _buildTheorySection(
            '🎨 Render Tree',
            'Render Tree คือสิ่งที่วาดบนหน้าจอจริงๆ '
            'แต่ละ RenderObject รู้จักขนาด ตำแหน่ง และวิธีวาดตัวเอง '
            'Flutter ใช้ Skia/Impeller วาด pixels บนหน้าจอ',
            Colors.orange,
          ),
          _buildTheorySection(
            '⚡ Performance',
            'Flutter เปรียบเทียบ Widget tree ก่อน-หลัง (reconciliation) '
            'แล้วอัพเดทเฉพาะส่วนที่เปลี่ยน ทำให้มีประสิทธิภาพสูง '
            'ใช้ const constructor และ Keys เพื่อช่วย Flutter ทำงานได้ดีขึ้น',
            Colors.purple,
          ),
        ],
      ),
    );
  }

  Widget _buildTheorySection(String title, String content, Color color) {
    return Container(
      margin: const EdgeInsets.only(bottom: 16),
      padding: const EdgeInsets.all(16),
      decoration: BoxDecoration(
        color: color.withOpacity(0.1),
        borderRadius: BorderRadius.circular(12),
        border: Border.all(color: color.withOpacity(0.3)),
      ),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text(
            title,
            style: TextStyle(
              fontSize: 16,
              fontWeight: FontWeight.bold,
              color: color,
            ),
          ),
          const SizedBox(height: 8),
          Text(
            content,
            style: const TextStyle(height: 1.5),
          ),
        ],
      ),
    );
  }
}

// Widget สำหรับแสดง tree node
class TreeNodeWidget extends StatefulWidget {
  final TreeNode node;
  final int depth;
  
  const TreeNodeWidget({
    super.key,
    required this.node,
    required this.depth,
  });

  @override
  State<TreeNodeWidget> createState() => _TreeNodeWidgetState();
}

class _TreeNodeWidgetState extends State<TreeNodeWidget> {
  bool _isExpanded = true;
  bool _isSelected = false;
  
  @override
  Widget build(BuildContext context) {
    final node = widget.node;
    final hasChildren = node.children.isNotEmpty;
    
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        // Node row
        GestureDetector(
          onTap: () {
            setState(() => _isSelected = !_isSelected);
            if (_isSelected && node.description != null) {
              ScaffoldMessenger.of(context).showSnackBar(
                SnackBar(
                  content: Text('${node.name}: ${node.description}'),
                  duration: const Duration(seconds: 2),
                ),
              );
            }
          },
          child: Container(
            margin: EdgeInsets.only(
              left: widget.depth * 24.0,
              bottom: 4,
            ),
            padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 8),
            decoration: BoxDecoration(
              color: _isSelected 
                ? node.color.withOpacity(0.2) 
                : node.color.withOpacity(0.08),
              borderRadius: BorderRadius.circular(8),
              border: Border.all(
                color: _isSelected 
                  ? node.color 
                  : node.color.withOpacity(0.3),
              ),
            ),
            child: Row(
              mainAxisSize: MainAxisSize.min,
              children: [
                // Expand/collapse button
                if (hasChildren)
                  GestureDetector(
                    onTap: () => setState(() => _isExpanded = !_isExpanded),
                    child: Icon(
                      _isExpanded ? Icons.expand_more : Icons.chevron_right,
                      size: 18,
                      color: node.color,
                    ),
                  )
                else
                  const SizedBox(width: 18),
                
                const SizedBox(width: 8),
                
                // Node name
                Text(
                  node.name,
                  style: TextStyle(
                    color: node.color,
                    fontWeight: FontWeight.w600,
                    fontFamily: 'monospace',
                    fontSize: 13,
                  ),
                ),
                
                // Child count badge
                if (hasChildren) ...[
                  const SizedBox(width: 8),
                  Container(
                    padding: const EdgeInsets.symmetric(
                      horizontal: 6, vertical: 2
                    ),
                    decoration: BoxDecoration(
                      color: node.color.withOpacity(0.2),
                      borderRadius: BorderRadius.circular(10),
                    ),
                    child: Text(
                      '${node.children.length}',
                      style: TextStyle(
                        fontSize: 10,
                        color: node.color,
                        fontWeight: FontWeight.bold,
                      ),
                    ),
                  ),
                ],
              ],
            ),
          ),
        ),
        
        // Children (if expanded)
        if (hasChildren && _isExpanded)
          ...node.children.map((child) => TreeNodeWidget(
            key: ValueKey(child.name),
            node: child,
            depth: widget.depth + 1,
          )),
      ],
    );
  }
}
```

---

## สรุป Widget Lifecycle

```
┌─────────────────────────────────────────────────┐
│              StatefulWidget Lifecycle            │
├─────────────────────────────────────────────────┤
│  Widget ถูกสร้าง                                │
│  ↓                                              │
│  createState() ← สร้าง State object             │
│  ↓                                              │
│  initState() ← initialize state, subscribe      │
│  ↓                                              │
│  didChangeDependencies() ← context พร้อมแล้ว   │
│  ↓                                              │
│  build() ← สร้าง widget tree                   │
│  ↓                                              │
│  ┌──────────────────────────────────────────┐   │
│  │  setState() → build() again (loop)       │   │
│  │  didUpdateWidget() → build() again       │   │
│  │  didChangeDependencies() → build() again │   │
│  └──────────────────────────────────────────┘   │
│  ↓                                              │
│  deactivate() ← ถูก remove จาก tree ชั่วคราว   │
│  ↓                                              │
│  dispose() ← ทำลาย widget, cleanup resources    │
└─────────────────────────────────────────────────┘
```

## สรุปบทที่ 22

ในบทนี้เราได้เรียนรู้:

1. **StatelessWidget vs StatefulWidget**: ความแตกต่างและเวลาใช้
2. **Widget Lifecycle**: initState, build, didUpdateWidget, dispose
3. **BuildContext**: การใช้งานและข้อควรระวัง
4. **InheritedWidget**: การ share data ตลอด tree
5. **Keys**: ประเภทและการใช้งานที่ถูกต้อง

บทต่อไปเราจะเจาะลึก StatelessWidget และ StatefulWidget พร้อมตัวอย่างการใช้งานจริง
