# Part 23: StatelessWidget and StatefulWidget

## บทนำ

ในบทนี้เราจะเจาะลึก StatelessWidget และ StatefulWidget ที่เป็นหัวใจสำคัญของ Flutter development ทำความเข้าใจเมื่อไรควรใช้อะไร และเทคนิคในการเพิ่มประสิทธิภาพ

---

## StatelessWidget เชิงลึก

### เมื่อไรควรใช้ StatelessWidget

ใช้ StatelessWidget เมื่อ:
1. Widget แสดงผลตาม props ที่รับมาเท่านั้น
2. ไม่มี user interaction ที่ต้องเปลี่ยน UI
3. เป็น pure function (input เดิม = output เดิม)

```dart
// ✅ เหมาะกับ StatelessWidget
// - แสดงข้อมูล user
class UserAvatar extends StatelessWidget {
  final String name;
  final String? imageUrl;
  final double size;
  
  const UserAvatar({
    super.key,
    required this.name,
    this.imageUrl,
    this.size = 48,
  });

  @override
  Widget build(BuildContext context) {
    if (imageUrl != null) {
      return CircleAvatar(
        radius: size / 2,
        backgroundImage: NetworkImage(imageUrl!),
      );
    }
    
    // แสดง initials ถ้าไม่มีรูป
    final initials = name.split(' ')
        .take(2)
        .map((word) => word[0].toUpperCase())
        .join('');
    
    return CircleAvatar(
      radius: size / 2,
      backgroundColor: _getColorFromName(name),
      child: Text(
        initials,
        style: TextStyle(
          color: Colors.white,
          fontSize: size * 0.35,
          fontWeight: FontWeight.bold,
        ),
      ),
    );
  }
  
  Color _getColorFromName(String name) {
    final colors = [
      Colors.blue,
      Colors.red,
      Colors.green,
      Colors.purple,
      Colors.orange,
      Colors.teal,
    ];
    return colors[name.hashCode.abs() % colors.length];
  }
}
```

### const Constructor และ Performance

```dart
// ✅ ใช้ const constructor เสมอถ้าทำได้
class MyWidget extends StatelessWidget {
  final String title;
  
  // const constructor
  const MyWidget({super.key, required this.title});
  
  @override
  Widget build(BuildContext context) {
    return Text(title);
  }
}

// การใช้งาน
// ✅ ใช้ const ถ้า value ไม่เปลี่ยน
const MyWidget(title: 'Hello')  // Flutter ไม่ต้อง rebuild นี้

// ❌ ไม่มี const
MyWidget(title: 'Hello')  // Flutter rebuild ทุกครั้ง

// Inline const
Column(
  children: const [
    // ทุก widget ใน list นี้เป็น const
    Text('Item 1'),
    SizedBox(height: 8),
    Text('Item 2'),
  ],
)
```

### Composition Pattern

```dart
// แทนที่จะสร้าง widget ใหญ่ๆ ให้ compose จาก widget เล็กๆ

// ✅ แนะนำ: แยก widget
class ProductCard extends StatelessWidget {
  final Product product;
  
  const ProductCard({super.key, required this.product});

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Column(
        children: [
          ProductImage(imageUrl: product.imageUrl),
          ProductInfo(
            name: product.name,
            price: product.price,
            rating: product.rating,
          ),
          ProductActions(product: product),
        ],
      ),
    );
  }
}

class ProductImage extends StatelessWidget {
  final String imageUrl;
  
  const ProductImage({super.key, required this.imageUrl});

  @override
  Widget build(BuildContext context) {
    return AspectRatio(
      aspectRatio: 16 / 9,
      child: Image.network(
        imageUrl,
        fit: BoxFit.cover,
        errorBuilder: (context, error, stackTrace) {
          return Container(
            color: Colors.grey[200],
            child: const Icon(Icons.broken_image, size: 48),
          );
        },
      ),
    );
  }
}

class ProductInfo extends StatelessWidget {
  final String name;
  final double price;
  final double rating;
  
  const ProductInfo({
    super.key,
    required this.name,
    required this.price,
    required this.rating,
  });

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    
    return Padding(
      padding: const EdgeInsets.all(12),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text(
            name,
            style: theme.textTheme.titleMedium?.copyWith(
              fontWeight: FontWeight.bold,
            ),
            maxLines: 2,
            overflow: TextOverflow.ellipsis,
          ),
          const SizedBox(height: 4),
          Row(
            children: [
              Text(
                '฿${price.toStringAsFixed(0)}',
                style: theme.textTheme.titleLarge?.copyWith(
                  color: theme.colorScheme.primary,
                  fontWeight: FontWeight.bold,
                ),
              ),
              const Spacer(),
              Icon(Icons.star, size: 16, color: Colors.amber[700]),
              Text(
                rating.toStringAsFixed(1),
                style: theme.textTheme.bodySmall,
              ),
            ],
          ),
        ],
      ),
    );
  }
}
```

---

## StatefulWidget เชิงลึก

### เมื่อไรควรใช้ StatefulWidget

```
ใช้ StatefulWidget เมื่อต้องการ:
1. เก็บ state ที่เปลี่ยนได้
2. Response ต่อ user interaction
3. Animate
4. Subscribe to streams/futures
5. Form state
```

### setState ที่ถูกต้อง

```dart
class CorrectStateManagement extends StatefulWidget {
  const CorrectStateManagement({super.key});

  @override
  State<CorrectStateManagement> createState() => _CorrectStateManagementState();
}

class _CorrectStateManagementState extends State<CorrectStateManagement> {
  int _count = 0;
  bool _isLoading = false;
  String _message = '';
  List<String> _items = [];

  // ✅ setState แบบที่ถูกต้อง
  void _increment() {
    setState(() {
      _count++;  // เปลี่ยน state ใน setState callback
    });
  }

  // ✅ เปลี่ยนหลาย state พร้อมกัน
  void _resetAll() {
    setState(() {
      _count = 0;
      _message = 'Reset!';
      _items = [];
    });
  }

  // ✅ setState หลัง async operation
  Future<void> _loadData() async {
    setState(() => _isLoading = true);
    
    try {
      // Simulate API call
      await Future.delayed(const Duration(seconds: 2));
      final newItems = ['Item 1', 'Item 2', 'Item 3'];
      
      if (mounted) {
        setState(() {
          _items = newItems;
          _isLoading = false;
          _message = 'โหลดสำเร็จ!';
        });
      }
    } catch (e) {
      if (mounted) {
        setState(() {
          _isLoading = false;
          _message = 'เกิดข้อผิดพลาด: $e';
        });
      }
    }
  }

  // ❌ อย่าทำแบบนี้ - setState นอก callback
  // void wrong() {
  //   _count++;  // ไม่ได้อยู่ใน setState
  //   setState(() {}); // empty setState
  // }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        if (_isLoading)
          const CircularProgressIndicator()
        else
          Text('Count: $_count'),
        Text(_message),
        ElevatedButton(
          onPressed: _increment,
          child: const Text('Increment'),
        ),
        ElevatedButton(
          onPressed: _isLoading ? null : _loadData,
          child: const Text('Load Data'),
        ),
      ],
    );
  }
}
```

---

## Lifecycle Methods เชิงลึก

### initState

```dart
class InitStateExamples extends StatefulWidget {
  final String userId;
  
  const InitStateExamples({super.key, required this.userId});

  @override
  State<InitStateExamples> createState() => _InitStateExamplesState();
}

class _InitStateExamplesState extends State<InitStateExamples> {
  late final TextEditingController _controller;
  late final ScrollController _scrollController;
  late final AnimationController _animController;
  
  // Data
  User? _user;
  bool _isLoading = true;
  
  @override
  void initState() {
    super.initState();
    
    // 1. Initialize controllers
    _controller = TextEditingController(text: '');
    _scrollController = ScrollController();
    
    // 2. Setup listeners
    _controller.addListener(_onTextChanged);
    _scrollController.addListener(_onScroll);
    
    // 3. Load initial data (ไม่ await ใน initState โดยตรง)
    _loadUser();
    
    // 4. Schedule ทำงานหลัง build() แรก
    WidgetsBinding.instance.addPostFrameCallback((_) {
      // ทำงานหลัง first frame
      print('First frame rendered');
    });
  }
  
  void _onTextChanged() {
    print('Text: ${_controller.text}');
  }
  
  void _onScroll() {
    if (_scrollController.position.pixels >= 
        _scrollController.position.maxScrollExtent - 200) {
      // Load more data when near bottom
      print('Load more!');
    }
  }
  
  Future<void> _loadUser() async {
    try {
      // Simulate API call
      await Future.delayed(const Duration(seconds: 1));
      if (mounted) {
        setState(() {
          _user = User(
            id: widget.userId,
            name: 'John Doe',
            email: 'john@example.com',
          );
          _isLoading = false;
        });
      }
    } catch (e) {
      if (mounted) {
        setState(() => _isLoading = false);
      }
    }
  }

  @override
  void dispose() {
    // ต้อง dispose ทุก controller และ listener
    _controller.removeListener(_onTextChanged);
    _controller.dispose();
    
    _scrollController.removeListener(_onScroll);
    _scrollController.dispose();
    
    // _animController.dispose();
    
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    if (_isLoading) {
      return const CircularProgressIndicator();
    }
    
    if (_user == null) {
      return const Text('ไม่พบผู้ใช้');
    }
    
    return Text(_user!.name);
  }
}

// Simple User model
class User {
  final String id;
  final String name;
  final String email;
  
  const User({required this.id, required this.name, required this.email});
}
```

### didUpdateWidget

```dart
class DidUpdateWidgetExample extends StatefulWidget {
  final String query;
  final int pageSize;
  
  const DidUpdateWidgetExample({
    super.key,
    required this.query,
    this.pageSize = 10,
  });

  @override
  State<DidUpdateWidgetExample> createState() => _DidUpdateWidgetExampleState();
}

class _DidUpdateWidgetExampleState extends State<DidUpdateWidgetExample> {
  List<String> _results = [];
  bool _isLoading = false;
  
  @override
  void initState() {
    super.initState();
    _search(widget.query);
  }
  
  // เรียกเมื่อ parent ส่ง widget ใหม่มา (props เปลี่ยน)
  @override
  void didUpdateWidget(DidUpdateWidgetExample oldWidget) {
    super.didUpdateWidget(oldWidget);
    
    // ค้นหาใหม่เมื่อ query เปลี่ยน
    if (widget.query != oldWidget.query) {
      _search(widget.query);
    }
    
    // โหลดใหม่เมื่อ pageSize เปลี่ยน
    if (widget.pageSize != oldWidget.pageSize) {
      _search(widget.query);
    }
  }
  
  Future<void> _search(String query) async {
    if (query.isEmpty) return;
    
    setState(() => _isLoading = true);
    
    // Simulate search
    await Future.delayed(const Duration(milliseconds: 500));
    
    if (mounted) {
      setState(() {
        _results = List.generate(
          widget.pageSize,
          (i) => 'Result ${i + 1} for "$query"',
        );
        _isLoading = false;
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        if (_isLoading) const LinearProgressIndicator(),
        Expanded(
          child: ListView.builder(
            itemCount: _results.length,
            itemBuilder: (context, index) => ListTile(
              title: Text(_results[index]),
            ),
          ),
        ),
      ],
    );
  }
}
```

### didChangeDependencies

```dart
class ThemeAwareWidget extends StatefulWidget {
  const ThemeAwareWidget({super.key});

  @override
  State<ThemeAwareWidget> createState() => _ThemeAwareWidgetState();
}

class _ThemeAwareWidgetState extends State<ThemeAwareWidget> {
  late ColorScheme _colorScheme;
  
  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    // เรียกเมื่อ Theme หรือ MediaQuery เปลี่ยน
    // ปลอดภัยที่จะใช้ context ที่นี่
    _colorScheme = Theme.of(context).colorScheme;
    
    // ทำงานหลัง theme เปลี่ยน
    print('Theme changed: primary = ${_colorScheme.primary}');
  }

  @override
  Widget build(BuildContext context) {
    return Container(
      color: _colorScheme.primaryContainer,
      padding: const EdgeInsets.all(16),
      child: Text(
        'Theme-aware widget',
        style: TextStyle(color: _colorScheme.onPrimaryContainer),
      ),
    );
  }
}
```

---

## const Constructor สำหรับ Performance

```dart
// Performance test
class PerformanceDemo extends StatefulWidget {
  const PerformanceDemo({super.key});

  @override
  State<PerformanceDemo> createState() => _PerformanceDemoState();
}

class _PerformanceDemoState extends State<PerformanceDemo> {
  int _counter = 0;
  
  @override
  Widget build(BuildContext context) {
    print('PerformanceDemo rebuild');
    
    return Column(
      children: [
        // ✅ const - ไม่ rebuild เมื่อ _counter เปลี่ยน
        const Text('I never rebuild'),
        const SizedBox(height: 8),
        
        // ❌ ไม่มี const - rebuild ทุกครั้ง
        Text('Counter: $_counter'),
        
        // ✅ Widget ที่ไม่ขึ้นกับ state ควรเป็น const
        const Divider(),
        const Padding(
          padding: EdgeInsets.all(8),
          child: Text('Static content'),
        ),
        
        // ✅ Extract เป็น const widget แยก
        const _StaticSection(),
        
        ElevatedButton(
          onPressed: () => setState(() => _counter++),
          child: const Text('Increment'),
        ),
      ],
    );
  }
}

// Widget นี้เป็น const ได้เพราะไม่มี state
class _StaticSection extends StatelessWidget {
  const _StaticSection();

  @override
  Widget build(BuildContext context) {
    print('_StaticSection rebuild'); // จะไม่ print เมื่อ parent rebuild
    return const Card(
      child: Padding(
        padding: EdgeInsets.all(16),
        child: Text('Static section - never rebuilds'),
      ),
    );
  }
}
```

---

## Pattern: Lifting State Up

เมื่อ state ต้องใช้ร่วมกันหลาย widget ควร "lift" ขึ้นไปที่ parent

```dart
// ❌ State อยู่ใน child widget แยกกัน - ไม่ share ได้
class BadPattern extends StatelessWidget {
  const BadPattern({super.key});

  @override
  Widget build(BuildContext context) {
    return Column(
      children: const [
        // แต่ละ widget มี counter ของตัวเอง
        IndependentCounter(),
        IndependentCounter(), // ค่าแยกกัน!
      ],
    );
  }
}

// ✅ Lift state ขึ้น parent
class GoodPattern extends StatefulWidget {
  const GoodPattern({super.key});

  @override
  State<GoodPattern> createState() => _GoodPatternState();
}

class _GoodPatternState extends State<GoodPattern> {
  int _sharedCount = 0;
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text('Shared count: $_sharedCount',
          style: const TextStyle(fontSize: 24),
        ),
        Row(
          mainAxisAlignment: MainAxisAlignment.spaceEvenly,
          children: [
            // ทั้งสอง widget share state เดียวกัน
            CounterButton(
              label: '+1',
              onPressed: () => setState(() => _sharedCount += 1),
            ),
            CounterButton(
              label: '+5',
              onPressed: () => setState(() => _sharedCount += 5),
            ),
            CounterButton(
              label: 'Reset',
              onPressed: () => setState(() => _sharedCount = 0),
            ),
          ],
        ),
      ],
    );
  }
}

class CounterButton extends StatelessWidget {
  final String label;
  final VoidCallback onPressed;
  
  const CounterButton({
    super.key,
    required this.label,
    required this.onPressed,
  });

  @override
  Widget build(BuildContext context) {
    return ElevatedButton(
      onPressed: onPressed,
      child: Text(label),
    );
  }
}
```

---

## Workshop: Todo Item Widget

สร้าง Todo app ที่แสดง todo items แบบ interactive

### lib/models/todo.dart

```dart
class Todo {
  final String id;
  final String title;
  final String? description;
  bool isCompleted;
  final DateTime createdAt;
  DateTime? completedAt;
  
  Todo({
    required this.id,
    required this.title,
    this.description,
    this.isCompleted = false,
    required this.createdAt,
    this.completedAt,
  });
  
  Todo copyWith({
    String? title,
    String? description,
    bool? isCompleted,
    DateTime? completedAt,
  }) {
    return Todo(
      id: id,
      title: title ?? this.title,
      description: description ?? this.description,
      isCompleted: isCompleted ?? this.isCompleted,
      createdAt: createdAt,
      completedAt: completedAt ?? this.completedAt,
    );
  }
}
```

### lib/widgets/todo_item.dart

```dart
import 'package:flutter/material.dart';
import '../models/todo.dart';

// StatefulWidget เพราะมี interaction (swipe, checkbox)
class TodoItem extends StatefulWidget {
  final Todo todo;
  final void Function(String id, bool completed) onToggle;
  final void Function(String id) onDelete;
  final void Function(Todo todo) onEdit;
  
  const TodoItem({
    super.key,
    required this.todo,
    required this.onToggle,
    required this.onDelete,
    required this.onEdit,
  });

  @override
  State<TodoItem> createState() => _TodoItemState();
}

class _TodoItemState extends State<TodoItem> 
    with SingleTickerProviderStateMixin {
  bool _isHovered = false;
  late AnimationController _animController;
  late Animation<double> _checkAnimation;

  @override
  void initState() {
    super.initState();
    _animController = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 300),
    );
    _checkAnimation = Tween<double>(begin: 0, end: 1).animate(
      CurvedAnimation(parent: _animController, curve: Curves.elasticOut),
    );
    
    if (widget.todo.isCompleted) {
      _animController.value = 1;
    }
  }

  @override
  void didUpdateWidget(TodoItem oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (widget.todo.isCompleted != oldWidget.todo.isCompleted) {
      if (widget.todo.isCompleted) {
        _animController.forward();
      } else {
        _animController.reverse();
      }
    }
  }

  @override
  void dispose() {
    _animController.dispose();
    super.dispose();
  }

  void _handleToggle() {
    widget.onToggle(widget.todo.id, !widget.todo.isCompleted);
  }

  void _handleDelete() {
    // Show confirmation
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('ลบ Todo'),
        content: Text('ต้องการลบ "${widget.todo.title}" หรือไม่?'),
        actions: [
          TextButton(
            onPressed: () => Navigator.of(context).pop(),
            child: const Text('ยกเลิก'),
          ),
          TextButton(
            onPressed: () {
              Navigator.of(context).pop();
              widget.onDelete(widget.todo.id);
            },
            style: TextButton.styleFrom(foregroundColor: Colors.red),
            child: const Text('ลบ'),
          ),
        ],
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final todo = widget.todo;
    
    return Dismissible(
      key: ValueKey(todo.id),
      direction: DismissDirection.endToStart,
      background: Container(
        alignment: Alignment.centerRight,
        padding: const EdgeInsets.only(right: 16),
        color: Colors.red,
        child: const Icon(Icons.delete, color: Colors.white, size: 32),
      ),
      confirmDismiss: (_) async {
        bool? result;
        await showDialog(
          context: context,
          builder: (context) => AlertDialog(
            title: const Text('ลบ Todo'),
            content: Text('ลบ "${todo.title}"?'),
            actions: [
              TextButton(
                onPressed: () => Navigator.pop(context, false),
                child: const Text('ยกเลิก'),
              ),
              TextButton(
                onPressed: () => Navigator.pop(context, true),
                child: const Text('ลบ', style: TextStyle(color: Colors.red)),
              ),
            ],
          ),
        ).then((value) => result = value);
        return result ?? false;
      },
      onDismissed: (_) => widget.onDelete(todo.id),
      child: MouseRegion(
        onEnter: (_) => setState(() => _isHovered = true),
        onExit: (_) => setState(() => _isHovered = false),
        child: AnimatedContainer(
          duration: const Duration(milliseconds: 200),
          margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
          decoration: BoxDecoration(
            color: _isHovered
                ? theme.colorScheme.surfaceVariant
                : theme.colorScheme.surface,
            borderRadius: BorderRadius.circular(12),
            border: Border.all(
              color: todo.isCompleted
                  ? theme.colorScheme.outline.withOpacity(0.3)
                  : theme.colorScheme.outline.withOpacity(0.5),
            ),
            boxShadow: _isHovered
                ? [BoxShadow(
                    color: Colors.black.withOpacity(0.1),
                    blurRadius: 8,
                    offset: const Offset(0, 2),
                  )]
                : [],
          ),
          child: ListTile(
            contentPadding: const EdgeInsets.symmetric(
              horizontal: 16, vertical: 8
            ),
            leading: _buildCheckbox(theme, todo),
            title: _buildTitle(theme, todo),
            subtitle: _buildSubtitle(theme, todo),
            trailing: _buildActions(todo),
            onTap: _handleToggle,
          ),
        ),
      ),
    );
  }

  Widget _buildCheckbox(ThemeData theme, Todo todo) {
    return GestureDetector(
      onTap: _handleToggle,
      child: AnimatedBuilder(
        animation: _checkAnimation,
        builder: (context, child) {
          return Container(
            width: 28,
            height: 28,
            decoration: BoxDecoration(
              shape: BoxShape.circle,
              color: todo.isCompleted
                  ? theme.colorScheme.primary
                  : Colors.transparent,
              border: Border.all(
                color: todo.isCompleted
                    ? theme.colorScheme.primary
                    : theme.colorScheme.outline,
                width: 2,
              ),
            ),
            child: todo.isCompleted
                ? Icon(
                    Icons.check,
                    size: 16,
                    color: theme.colorScheme.onPrimary,
                  )
                : null,
          );
        },
      ),
    );
  }

  Widget _buildTitle(ThemeData theme, Todo todo) {
    return Text(
      todo.title,
      style: theme.textTheme.bodyLarge?.copyWith(
        decoration: todo.isCompleted ? TextDecoration.lineThrough : null,
        color: todo.isCompleted
            ? theme.colorScheme.onSurface.withOpacity(0.5)
            : theme.colorScheme.onSurface,
        fontWeight: FontWeight.w500,
      ),
    );
  }

  Widget _buildSubtitle(ThemeData theme, Todo todo) {
    if (todo.description == null && !todo.isCompleted) return const SizedBox.shrink();
    
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        if (todo.description != null) ...[
          const SizedBox(height: 4),
          Text(
            todo.description!,
            style: theme.textTheme.bodySmall?.copyWith(
              color: theme.colorScheme.onSurfaceVariant,
            ),
            maxLines: 2,
            overflow: TextOverflow.ellipsis,
          ),
        ],
        if (todo.isCompleted && todo.completedAt != null) ...[
          const SizedBox(height: 4),
          Text(
            'เสร็จเมื่อ ${_formatDate(todo.completedAt!)}',
            style: theme.textTheme.labelSmall?.copyWith(
              color: theme.colorScheme.primary,
            ),
          ),
        ],
      ],
    );
  }

  Widget _buildActions(Todo todo) {
    return Row(
      mainAxisSize: MainAxisSize.min,
      children: [
        IconButton(
          icon: const Icon(Icons.edit_outlined, size: 20),
          onPressed: () => widget.onEdit(todo),
          tooltip: 'แก้ไข',
        ),
        IconButton(
          icon: const Icon(Icons.delete_outline, size: 20),
          onPressed: _handleDelete,
          tooltip: 'ลบ',
          color: Colors.red[400],
        ),
      ],
    );
  }

  String _formatDate(DateTime date) {
    return '${date.day}/${date.month}/${date.year} ${date.hour}:${date.minute.toString().padLeft(2, '0')}';
  }
}
```

### lib/screens/todo_screen.dart

```dart
import 'package:flutter/material.dart';
import '../models/todo.dart';
import '../widgets/todo_item.dart';

class TodoScreen extends StatefulWidget {
  const TodoScreen({super.key});

  @override
  State<TodoScreen> createState() => _TodoScreenState();
}

class _TodoScreenState extends State<TodoScreen> {
  final List<Todo> _todos = [];
  String _filter = 'all'; // all, active, completed
  
  @override
  void initState() {
    super.initState();
    // Sample data
    _todos.addAll([
      Todo(
        id: '1',
        title: 'เรียน Flutter',
        description: 'ศึกษา StatelessWidget และ StatefulWidget',
        createdAt: DateTime.now(),
      ),
      Todo(
        id: '2',
        title: 'สร้าง Todo App',
        description: 'Workshop ในบทที่ 23',
        createdAt: DateTime.now().subtract(const Duration(hours: 1)),
      ),
      Todo(
        id: '3',
        title: 'ทำ Workshop ทุกบท',
        isCompleted: true,
        createdAt: DateTime.now().subtract(const Duration(days: 1)),
        completedAt: DateTime.now().subtract(const Duration(hours: 2)),
      ),
    ]);
  }

  List<Todo> get _filteredTodos {
    switch (_filter) {
      case 'active':
        return _todos.where((t) => !t.isCompleted).toList();
      case 'completed':
        return _todos.where((t) => t.isCompleted).toList();
      default:
        return _todos;
    }
  }

  void _addTodo(String title, String? description) {
    setState(() {
      _todos.insert(
        0,
        Todo(
          id: DateTime.now().millisecondsSinceEpoch.toString(),
          title: title,
          description: description?.isEmpty == true ? null : description,
          createdAt: DateTime.now(),
        ),
      );
    });
  }

  void _toggleTodo(String id, bool completed) {
    setState(() {
      final index = _todos.indexWhere((t) => t.id == id);
      if (index != -1) {
        _todos[index] = _todos[index].copyWith(
          isCompleted: completed,
          completedAt: completed ? DateTime.now() : null,
        );
      }
    });
  }

  void _deleteTodo(String id) {
    setState(() {
      _todos.removeWhere((t) => t.id == id);
    });
    ScaffoldMessenger.of(context).showSnackBar(
      const SnackBar(content: Text('ลบ Todo แล้ว')),
    );
  }

  void _editTodo(Todo todo) {
    showModalBottomSheet(
      context: context,
      isScrollControlled: true,
      shape: const RoundedRectangleBorder(
        borderRadius: BorderRadius.vertical(top: Radius.circular(20)),
      ),
      builder: (context) => _EditTodoSheet(
        todo: todo,
        onSave: (title, description) {
          setState(() {
            final index = _todos.indexWhere((t) => t.id == todo.id);
            if (index != -1) {
              _todos[index] = _todos[index].copyWith(
                title: title,
                description: description,
              );
            }
          });
        },
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final completedCount = _todos.where((t) => t.isCompleted).length;
    
    return Scaffold(
      appBar: AppBar(
        title: const Text('My Todos'),
        actions: [
          IconButton(
            icon: const Icon(Icons.clear_all),
            onPressed: _todos.any((t) => t.isCompleted)
                ? () {
                    setState(() {
                      _todos.removeWhere((t) => t.isCompleted);
                    });
                  }
                : null,
            tooltip: 'ลบที่เสร็จแล้วทั้งหมด',
          ),
        ],
      ),
      body: Column(
        children: [
          // Progress
          _buildProgress(theme, completedCount),
          
          // Filter chips
          _buildFilters(theme),
          
          // Todo list
          Expanded(
            child: _filteredTodos.isEmpty
                ? _buildEmptyState(theme)
                : ReorderableListView.builder(
                    itemCount: _filteredTodos.length,
                    onReorder: (oldIndex, newIndex) {
                      setState(() {
                        if (newIndex > oldIndex) newIndex--;
                        final todo = _filteredTodos.removeAt(oldIndex);
                        _filteredTodos.insert(newIndex, todo);
                      });
                    },
                    itemBuilder: (context, index) {
                      final todo = _filteredTodos[index];
                      return TodoItem(
                        key: ValueKey(todo.id),
                        todo: todo,
                        onToggle: _toggleTodo,
                        onDelete: _deleteTodo,
                        onEdit: _editTodo,
                      );
                    },
                  ),
          ),
        ],
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: _showAddDialog,
        child: const Icon(Icons.add),
      ),
    );
  }

  Widget _buildProgress(ThemeData theme, int completedCount) {
    final total = _todos.length;
    final progress = total > 0 ? completedCount / total : 0.0;
    
    return Padding(
      padding: const EdgeInsets.all(16),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Row(
            mainAxisAlignment: MainAxisAlignment.spaceBetween,
            children: [
              Text(
                'ความคืบหน้า',
                style: theme.textTheme.titleMedium,
              ),
              Text(
                '$completedCount/$total',
                style: theme.textTheme.titleMedium?.copyWith(
                  color: theme.colorScheme.primary,
                  fontWeight: FontWeight.bold,
                ),
              ),
            ],
          ),
          const SizedBox(height: 8),
          ClipRRect(
            borderRadius: BorderRadius.circular(4),
            child: LinearProgressIndicator(
              value: progress,
              minHeight: 8,
              backgroundColor: theme.colorScheme.surfaceVariant,
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildFilters(ThemeData theme) {
    final filters = [
      ('all', 'ทั้งหมด', _todos.length),
      ('active', 'ยังไม่เสร็จ', _todos.where((t) => !t.isCompleted).length),
      ('completed', 'เสร็จแล้ว', _todos.where((t) => t.isCompleted).length),
    ];
    
    return SingleChildScrollView(
      scrollDirection: Axis.horizontal,
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
      child: Row(
        children: filters.map((filter) {
          final (value, label, count) = filter;
          return Padding(
            padding: const EdgeInsets.only(right: 8),
            child: FilterChip(
              label: Text('$label ($count)'),
              selected: _filter == value,
              onSelected: (_) => setState(() => _filter = value),
            ),
          );
        }).toList(),
      ),
    );
  }

  Widget _buildEmptyState(ThemeData theme) {
    return Center(
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          Icon(
            _filter == 'completed' ? Icons.check_circle : Icons.inbox,
            size: 80,
            color: theme.colorScheme.outline,
          ),
          const SizedBox(height: 16),
          Text(
            _filter == 'completed' ? 'ยังไม่มีที่เสร็จแล้ว' : 'ไม่มี Todo',
            style: theme.textTheme.titleLarge?.copyWith(
              color: theme.colorScheme.outline,
            ),
          ),
          if (_filter == 'all') ...[
            const SizedBox(height: 8),
            Text(
              'กด + เพื่อเพิ่ม Todo',
              style: theme.textTheme.bodyMedium?.copyWith(
                color: theme.colorScheme.outline,
              ),
            ),
          ],
        ],
      ),
    );
  }

  void _showAddDialog() {
    showModalBottomSheet(
      context: context,
      isScrollControlled: true,
      shape: const RoundedRectangleBorder(
        borderRadius: BorderRadius.vertical(top: Radius.circular(20)),
      ),
      builder: (context) => _AddTodoSheet(onAdd: _addTodo),
    );
  }
}

// Bottom Sheet สำหรับเพิ่ม Todo
class _AddTodoSheet extends StatefulWidget {
  final void Function(String title, String? description) onAdd;
  
  const _AddTodoSheet({required this.onAdd});

  @override
  State<_AddTodoSheet> createState() => _AddTodoSheetState();
}

class _AddTodoSheetState extends State<_AddTodoSheet> {
  final _titleController = TextEditingController();
  final _descController = TextEditingController();
  final _formKey = GlobalKey<FormState>();

  @override
  void dispose() {
    _titleController.dispose();
    _descController.dispose();
    super.dispose();
  }

  void _submit() {
    if (_formKey.currentState!.validate()) {
      widget.onAdd(_titleController.text.trim(), _descController.text.trim());
      Navigator.pop(context);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: EdgeInsets.only(
        left: 24,
        right: 24,
        top: 24,
        bottom: MediaQuery.of(context).viewInsets.bottom + 24,
      ),
      child: Form(
        key: _formKey,
        child: Column(
          mainAxisSize: MainAxisSize.min,
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            const Text('เพิ่ม Todo ใหม่',
              style: TextStyle(fontSize: 20, fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 16),
            TextFormField(
              controller: _titleController,
              decoration: const InputDecoration(
                labelText: 'ชื่อ Todo *',
                hintText: 'เช่น เรียน Flutter',
                border: OutlineInputBorder(),
              ),
              autofocus: true,
              validator: (v) => v?.trim().isEmpty == true ? 'กรุณาใส่ชื่อ' : null,
              textInputAction: TextInputAction.next,
            ),
            const SizedBox(height: 12),
            TextFormField(
              controller: _descController,
              decoration: const InputDecoration(
                labelText: 'รายละเอียด (ไม่บังคับ)',
                border: OutlineInputBorder(),
              ),
              maxLines: 3,
              textInputAction: TextInputAction.done,
              onFieldSubmitted: (_) => _submit(),
            ),
            const SizedBox(height: 16),
            ElevatedButton(
              onPressed: _submit,
              child: const Text('เพิ่ม Todo'),
            ),
          ],
        ),
      ),
    );
  }
}

// Bottom Sheet สำหรับแก้ไข Todo
class _EditTodoSheet extends StatefulWidget {
  final Todo todo;
  final void Function(String title, String? description) onSave;
  
  const _EditTodoSheet({required this.todo, required this.onSave});

  @override
  State<_EditTodoSheet> createState() => _EditTodoSheetState();
}

class _EditTodoSheetState extends State<_EditTodoSheet> {
  late final TextEditingController _titleController;
  late final TextEditingController _descController;

  @override
  void initState() {
    super.initState();
    _titleController = TextEditingController(text: widget.todo.title);
    _descController = TextEditingController(text: widget.todo.description ?? '');
  }

  @override
  void dispose() {
    _titleController.dispose();
    _descController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: EdgeInsets.only(
        left: 24,
        right: 24,
        top: 24,
        bottom: MediaQuery.of(context).viewInsets.bottom + 24,
      ),
      child: Column(
        mainAxisSize: MainAxisSize.min,
        crossAxisAlignment: CrossAxisAlignment.stretch,
        children: [
          const Text('แก้ไข Todo',
            style: TextStyle(fontSize: 20, fontWeight: FontWeight.bold),
          ),
          const SizedBox(height: 16),
          TextField(
            controller: _titleController,
            decoration: const InputDecoration(
              labelText: 'ชื่อ Todo',
              border: OutlineInputBorder(),
            ),
          ),
          const SizedBox(height: 12),
          TextField(
            controller: _descController,
            decoration: const InputDecoration(
              labelText: 'รายละเอียด',
              border: OutlineInputBorder(),
            ),
            maxLines: 3,
          ),
          const SizedBox(height: 16),
          ElevatedButton(
            onPressed: () {
              widget.onSave(_titleController.text, _descController.text);
              Navigator.pop(context);
            },
            child: const Text('บันทึก'),
          ),
        ],
      ),
    );
  }
}
```

---

## สรุปบทที่ 23

ในบทนี้เราได้เรียนรู้:

1. **StatelessWidget**: const constructor, composition pattern
2. **StatefulWidget**: setState ที่ถูกต้อง, async safety
3. **Lifecycle Methods**: initState, dispose, didUpdateWidget, didChangeDependencies
4. **Performance**: const widgets, lifting state up
5. **Workshop**: Todo app ที่มี lifecycle management ครบถ้วน

บทต่อไปเราจะเรียน Basic Widgets เช่น Text, Image, Icon, Container
