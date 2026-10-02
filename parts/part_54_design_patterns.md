# Part 54: Design Patterns ใน Flutter

## บทนำ

Design Patterns คือ solutions ที่พิสูจน์แล้วสำหรับปัญหาที่เจอบ่อยในการออกแบบซอฟต์แวร์ แบ่งเป็น 3 กลุ่ม: Creational, Structural และ Behavioral

---

## 54.1 Singleton Pattern

### วัตถุประสงค์: ให้มี instance เดียวของ class ตลอด lifecycle

```dart
// Singleton พื้นฐาน
class AppConfig {
  static final AppConfig _instance = AppConfig._internal();
  
  factory AppConfig() => _instance;
  
  AppConfig._internal();
  
  // Settings
  String apiBaseUrl = 'https://api.example.com';
  String appVersion = '1.0.0';
  bool isDarkMode = false;
  bool isDebugMode = false;
  
  void configure({
    String? apiBaseUrl,
    bool? isDarkMode,
    bool? isDebugMode,
  }) {
    if (apiBaseUrl != null) this.apiBaseUrl = apiBaseUrl;
    if (isDarkMode != null) this.isDarkMode = isDarkMode;
    if (isDebugMode != null) this.isDebugMode = isDebugMode;
  }
}

// Singleton แบบ thread-safe
class DatabaseConnection {
  static DatabaseConnection? _instance;
  static final _lock = Object();
  
  DatabaseConnection._();
  
  static DatabaseConnection get instance {
    if (_instance == null) {
      synchronized(_lock, () {
        _instance ??= DatabaseConnection._();
      });
    }
    return _instance!;
  }
  
  Future<void> connect() async {
    print('Connecting to database...');
  }
  
  Future<void> disconnect() async {
    print('Disconnecting from database...');
  }
}

// Logger Singleton
class Logger {
  static final Logger _instance = Logger._internal();
  
  factory Logger() => _instance;
  
  Logger._internal();
  
  final List<LogEntry> _logs = [];
  
  void log(String message, {LogLevel level = LogLevel.info}) {
    final entry = LogEntry(
      message: message,
      level: level,
      timestamp: DateTime.now(),
    );
    _logs.add(entry);
    
    if (kDebugMode) {
      final prefix = switch (level) {
        LogLevel.info => '📘 INFO',
        LogLevel.warning => '⚠️ WARN',
        LogLevel.error => '🔴 ERROR',
        LogLevel.debug => '🔍 DEBUG',
      };
      print('$prefix [${entry.timestamp}]: $message');
    }
  }
  
  List<LogEntry> getLogs({LogLevel? level}) {
    if (level == null) return List.unmodifiable(_logs);
    return _logs.where((l) => l.level == level).toList();
  }
  
  void clear() => _logs.clear();
}

enum LogLevel { debug, info, warning, error }

class LogEntry {
  final String message;
  final LogLevel level;
  final DateTime timestamp;
  
  const LogEntry({
    required this.message,
    required this.level,
    required this.timestamp,
  });
}
```

---

## 54.2 Factory Pattern

### วัตถุประสงค์: สร้าง objects โดยไม่ต้องระบุ concrete class

```dart
// Abstract Product
abstract class NotificationWidget extends StatelessWidget {
  const NotificationWidget({super.key});
  
  String get title;
  String get body;
  IconData get icon;
  Color get iconColor;
}

// Concrete Products
class ChatNotificationWidget extends NotificationWidget {
  final String senderName;
  final String message;
  
  const ChatNotificationWidget({
    super.key,
    required this.senderName,
    required this.message,
  });
  
  @override
  String get title => senderName;
  
  @override
  String get body => message;
  
  @override
  IconData get icon => Icons.chat;
  
  @override
  Color get iconColor => Colors.blue;
  
  @override
  Widget build(BuildContext context) {
    return ListTile(
      leading: CircleAvatar(
        backgroundColor: iconColor.withOpacity(0.1),
        child: Icon(icon, color: iconColor),
      ),
      title: Text(title),
      subtitle: Text(body, maxLines: 1, overflow: TextOverflow.ellipsis),
    );
  }
}

class OrderNotificationWidget extends NotificationWidget {
  final String orderId;
  final String status;
  
  const OrderNotificationWidget({
    super.key,
    required this.orderId,
    required this.status,
  });
  
  @override
  String get title => 'คำสั่งซื้อ #$orderId';
  
  @override
  String get body => 'สถานะ: $status';
  
  @override
  IconData get icon => Icons.shopping_bag;
  
  @override
  Color get iconColor => Colors.orange;
  
  @override
  Widget build(BuildContext context) {
    return ListTile(
      leading: CircleAvatar(
        backgroundColor: iconColor.withOpacity(0.1),
        child: Icon(icon, color: iconColor),
      ),
      title: Text(title, style: const TextStyle(fontWeight: FontWeight.bold)),
      subtitle: Text(body),
    );
  }
}

// Factory Method
class NotificationWidgetFactory {
  static NotificationWidget create(Map<String, dynamic> data) {
    final type = data['type'] as String;
    
    return switch (type) {
      'chat' => ChatNotificationWidget(
        senderName: data['senderName'] ?? '',
        message: data['message'] ?? '',
      ),
      'order' => OrderNotificationWidget(
        orderId: data['orderId'] ?? '',
        status: data['status'] ?? '',
      ),
      _ => throw UnknownNotificationTypeException(type),
    };
  }
}

// Abstract Factory
abstract class ThemeFactory {
  Color get primaryColor;
  Color get backgroundColor;
  Color get textColor;
  TextTheme createTextTheme();
  ButtonThemeData createButtonTheme();
  ThemeData createTheme();
}

class LightThemeFactory implements ThemeFactory {
  @override
  Color get primaryColor => const Color(0xFF1976D2);
  
  @override
  Color get backgroundColor => Colors.white;
  
  @override
  Color get textColor => Colors.black87;
  
  @override
  TextTheme createTextTheme() {
    return TextTheme(
      headlineLarge: TextStyle(color: textColor, fontWeight: FontWeight.bold),
      bodyLarge: TextStyle(color: textColor),
    );
  }
  
  @override
  ButtonThemeData createButtonTheme() {
    return ButtonThemeData(
      buttonColor: primaryColor,
      textTheme: ButtonTextTheme.primary,
    );
  }
  
  @override
  ThemeData createTheme() {
    return ThemeData(
      brightness: Brightness.light,
      primaryColor: primaryColor,
      scaffoldBackgroundColor: backgroundColor,
      textTheme: createTextTheme(),
      buttonTheme: createButtonTheme(),
    );
  }
}

class DarkThemeFactory implements ThemeFactory {
  @override
  Color get primaryColor => const Color(0xFF90CAF9);
  
  @override
  Color get backgroundColor => const Color(0xFF121212);
  
  @override
  Color get textColor => Colors.white;
  
  @override
  TextTheme createTextTheme() {
    return TextTheme(
      headlineLarge: TextStyle(color: textColor, fontWeight: FontWeight.bold),
      bodyLarge: TextStyle(color: textColor),
    );
  }
  
  @override
  ButtonThemeData createButtonTheme() {
    return ButtonThemeData(buttonColor: primaryColor);
  }
  
  @override
  ThemeData createTheme() {
    return ThemeData(
      brightness: Brightness.dark,
      primaryColor: primaryColor,
      scaffoldBackgroundColor: backgroundColor,
      textTheme: createTextTheme(),
    );
  }
}
```

---

## 54.3 Builder Pattern

```dart
// สร้าง complex widget step by step
class AlertDialogBuilder {
  String? _title;
  String? _message;
  Widget? _content;
  final List<Widget> _actions = [];
  bool _barrierDismissible = true;
  
  AlertDialogBuilder withTitle(String title) {
    _title = title;
    return this;
  }
  
  AlertDialogBuilder withMessage(String message) {
    _message = message;
    return this;
  }
  
  AlertDialogBuilder withContent(Widget content) {
    _content = content;
    return this;
  }
  
  AlertDialogBuilder addCancelButton({
    String label = 'ยกเลิก',
    VoidCallback? onPressed,
    required BuildContext context,
  }) {
    _actions.add(
      TextButton(
        onPressed: onPressed ?? () => Navigator.pop(context),
        child: Text(label, style: const TextStyle(color: Colors.grey)),
      ),
    );
    return this;
  }
  
  AlertDialogBuilder addConfirmButton({
    required String label,
    required VoidCallback onPressed,
    bool isDestructive = false,
  }) {
    _actions.add(
      ElevatedButton(
        onPressed: onPressed,
        style: ElevatedButton.styleFrom(
          backgroundColor: isDestructive ? Colors.red : null,
        ),
        child: Text(label),
      ),
    );
    return this;
  }
  
  AlertDialogBuilder withBarrierDismissible(bool value) {
    _barrierDismissible = value;
    return this;
  }
  
  Future<T?> show<T>(BuildContext context) {
    return showDialog<T>(
      context: context,
      barrierDismissible: _barrierDismissible,
      builder: (context) => AlertDialog(
        title: _title != null ? Text(_title!) : null,
        content: _content ?? (_message != null ? Text(_message!) : null),
        actions: _actions,
      ),
    );
  }
}

// ใช้งาน
// AlertDialogBuilder()
//   .withTitle('ยืนยันการลบ')
//   .withMessage('คุณต้องการลบรายการนี้ใช่ไหม?')
//   .addCancelButton(context: context)
//   .addConfirmButton(
//     label: 'ลบ',
//     isDestructive: true,
//     onPressed: () { delete(); Navigator.pop(context); },
//   )
//   .show(context);
```

---

## 54.4 Observer Pattern

```dart
// Observer pattern ผ่าน ChangeNotifier
class CartNotifier extends ChangeNotifier {
  final List<CartItem> _items = [];
  
  List<CartItem> get items => List.unmodifiable(_items);
  
  int get itemCount => _items.fold(0, (sum, item) => sum + item.quantity);
  
  double get total => _items.fold(
    0.0,
    (sum, item) => sum + (item.price * item.quantity),
  );
  
  void addItem(CartItem item) {
    final index = _items.indexWhere((i) => i.id == item.id);
    
    if (index != -1) {
      _items[index] = _items[index].copyWith(
        quantity: _items[index].quantity + item.quantity,
      );
    } else {
      _items.add(item);
    }
    
    notifyListeners(); // แจ้ง observers ทั้งหมด
  }
  
  void removeItem(String itemId) {
    _items.removeWhere((item) => item.id == itemId);
    notifyListeners();
  }
  
  void clear() {
    _items.clear();
    notifyListeners();
  }
}

// Custom Observer ผ่าน Stream
class EventBus {
  static final EventBus _instance = EventBus._internal();
  
  factory EventBus() => _instance;
  
  EventBus._internal();
  
  final Map<Type, StreamController> _controllers = {};
  
  Stream<T> on<T>() {
    if (!_controllers.containsKey(T)) {
      _controllers[T] = StreamController<T>.broadcast();
    }
    return (_controllers[T] as StreamController<T>).stream;
  }
  
  void emit<T>(T event) {
    if (_controllers.containsKey(T)) {
      (_controllers[T] as StreamController<T>).add(event);
    }
  }
  
  void dispose<T>() {
    _controllers[T]?.close();
    _controllers.remove(T);
  }
}

// Events
class UserLoggedInEvent {
  final String userId;
  const UserLoggedInEvent(this.userId);
}

class CartUpdatedEvent {
  final int itemCount;
  const CartUpdatedEvent(this.itemCount);
}

// ใช้งาน
// EventBus().on<UserLoggedInEvent>().listen((event) {
//   print('User logged in: ${event.userId}');
// });
// EventBus().emit(UserLoggedInEvent('user123'));
```

---

## 54.5 Strategy Pattern

```dart
// Strategy สำหรับ sorting
abstract class SortStrategy<T> {
  List<T> sort(List<T> items);
  String get name;
}

class PriceSortStrategy implements SortStrategy<Product> {
  final bool ascending;
  
  const PriceSortStrategy({this.ascending = true});
  
  @override
  List<Product> sort(List<Product> items) {
    final sorted = List<Product>.from(items);
    sorted.sort((a, b) => ascending
        ? a.price.compareTo(b.price)
        : b.price.compareTo(a.price));
    return sorted;
  }
  
  @override
  String get name => 'ราคา ${ascending ? "น้อย→มาก" : "มาก→น้อย"}';
}

class RatingSortStrategy implements SortStrategy<Product> {
  @override
  List<Product> sort(List<Product> items) {
    final sorted = List<Product>.from(items);
    sorted.sort((a, b) => b.rating.compareTo(a.rating));
    return sorted;
  }
  
  @override
  String get name => 'คะแนนสูงสุด';
}

class NewestSortStrategy implements SortStrategy<Product> {
  @override
  List<Product> sort(List<Product> items) {
    final sorted = List<Product>.from(items);
    sorted.sort((a, b) => b.createdAt.compareTo(a.createdAt));
    return sorted;
  }
  
  @override
  String get name => 'ใหม่ล่าสุด';
}

class ProductListNotifier extends ChangeNotifier {
  List<Product> _products = [];
  SortStrategy<Product>? _sortStrategy;
  
  List<Product> get products {
    if (_sortStrategy == null) return _products;
    return _sortStrategy!.sort(_products);
  }
  
  void setSortStrategy(SortStrategy<Product> strategy) {
    _sortStrategy = strategy;
    notifyListeners();
  }
  
  void setProducts(List<Product> products) {
    _products = products;
    notifyListeners();
  }
}
```

---

## 54.6 Command Pattern

```dart
// Command สำหรับ undo/redo
abstract class Command {
  void execute();
  void undo();
  String get description;
}

class AddTodoCommand implements Command {
  final TodoRepository _repository;
  final Todo _todo;
  
  AddTodoCommand({
    required TodoRepository repository,
    required Todo todo,
  })  : _repository = repository,
        _todo = todo;
  
  @override
  void execute() => _repository.add(_todo);
  
  @override
  void undo() => _repository.delete(_todo.id);
  
  @override
  String get description => 'เพิ่ม: ${_todo.title}';
}

class DeleteTodoCommand implements Command {
  final TodoRepository _repository;
  final Todo _todo;
  
  DeleteTodoCommand({
    required TodoRepository repository,
    required Todo todo,
  })  : _repository = repository,
        _todo = todo;
  
  @override
  void execute() => _repository.delete(_todo.id);
  
  @override
  void undo() => _repository.add(_todo);
  
  @override
  String get description => 'ลบ: ${_todo.title}';
}

class CommandHistory {
  final List<Command> _history = [];
  int _currentIndex = -1;
  
  bool get canUndo => _currentIndex >= 0;
  bool get canRedo => _currentIndex < _history.length - 1;
  
  void execute(Command command) {
    // ลบ history หลัง current index
    if (_currentIndex < _history.length - 1) {
      _history.removeRange(_currentIndex + 1, _history.length);
    }
    
    command.execute();
    _history.add(command);
    _currentIndex++;
  }
  
  void undo() {
    if (!canUndo) return;
    _history[_currentIndex].undo();
    _currentIndex--;
  }
  
  void redo() {
    if (!canRedo) return;
    _currentIndex++;
    _history[_currentIndex].execute();
  }
  
  List<String> get historyDescriptions {
    return _history.map((c) => c.description).toList();
  }
}
```

---

## 54.7 Workshop: Plugin System with Patterns

```dart
// Plugin System ที่ใช้หลาย patterns
abstract class Plugin {
  String get id;
  String get name;
  String get version;
  
  void initialize();
  void dispose();
  Widget? buildWidget(BuildContext context);
}

// Plugin Registry (Singleton + Factory)
class PluginRegistry {
  static final PluginRegistry _instance = PluginRegistry._();
  factory PluginRegistry() => _instance;
  PluginRegistry._();
  
  final Map<String, Plugin> _plugins = {};
  final List<VoidCallback> _listeners = [];
  
  // Register plugin
  void register(Plugin plugin) {
    if (_plugins.containsKey(plugin.id)) {
      throw DuplicatePluginException('Plugin ${plugin.id} ลงทะเบียนแล้ว');
    }
    _plugins[plugin.id] = plugin;
    plugin.initialize();
    _notifyListeners();
    Logger().log('Plugin registered: ${plugin.name} v${plugin.version}');
  }
  
  // Unregister plugin
  void unregister(String pluginId) {
    final plugin = _plugins.remove(pluginId);
    if (plugin != null) {
      plugin.dispose();
      _notifyListeners();
    }
  }
  
  // Get plugin by ID
  T? get<T extends Plugin>(String pluginId) {
    return _plugins[pluginId] as T?;
  }
  
  // Get all plugins
  List<Plugin> get allPlugins => List.unmodifiable(_plugins.values);
  
  // Observer
  void addListener(VoidCallback listener) {
    _listeners.add(listener);
  }
  
  void removeListener(VoidCallback listener) {
    _listeners.remove(listener);
  }
  
  void _notifyListeners() {
    for (final listener in _listeners) {
      listener();
    }
  }
}

// Concrete Plugins
class WeatherPlugin extends Plugin {
  @override
  String get id => 'weather';
  
  @override
  String get name => 'Weather Widget';
  
  @override
  String get version => '1.0.0';
  
  String _currentCity = 'Bangkok';
  
  @override
  void initialize() {
    Logger().log('Weather plugin initialized');
  }
  
  @override
  void dispose() {
    Logger().log('Weather plugin disposed');
  }
  
  @override
  Widget? buildWidget(BuildContext context) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          mainAxisSize: MainAxisSize.min,
          children: [
            Row(
              children: [
                const Icon(Icons.wb_sunny, color: Colors.orange),
                const SizedBox(width: 8),
                Text('สภาพอากาศ - $_currentCity'),
              ],
            ),
            const SizedBox(height: 8),
            const Text('32°C  ท้องฟ้าแจ่มใส'),
          ],
        ),
      ),
    );
  }
}

class NewsPlugin extends Plugin {
  @override
  String get id => 'news';
  
  @override
  String get name => 'News Feed';
  
  @override
  String get version => '2.1.0';
  
  @override
  void initialize() {}
  
  @override
  void dispose() {}
  
  @override
  Widget? buildWidget(BuildContext context) {
    return const Card(
      child: ListTile(
        leading: Icon(Icons.newspaper),
        title: Text('ข่าวล่าสุด'),
        subtitle: Text('กด เพื่อดูข่าว'),
      ),
    );
  }
}

// Plugin Host Widget (Observer pattern)
class PluginDashboard extends StatefulWidget {
  const PluginDashboard({super.key});
  
  @override
  State<PluginDashboard> createState() => _PluginDashboardState();
}

class _PluginDashboardState extends State<PluginDashboard> {
  final PluginRegistry _registry = PluginRegistry();
  
  @override
  void initState() {
    super.initState();
    _registry.addListener(_onPluginsChanged);
  }
  
  void _onPluginsChanged() {
    setState(() {});
  }
  
  @override
  Widget build(BuildContext context) {
    final plugins = _registry.allPlugins;
    
    return Column(
      children: [
        Padding(
          padding: const EdgeInsets.all(16),
          child: Row(
            mainAxisAlignment: MainAxisAlignment.spaceBetween,
            children: [
              Text(
                'Plugins (${plugins.length})',
                style: Theme.of(context).textTheme.headlineSmall,
              ),
              IconButton(
                icon: const Icon(Icons.add),
                onPressed: _showAddPluginDialog,
              ),
            ],
          ),
        ),
        if (plugins.isEmpty)
          const Center(
            child: Padding(
              padding: EdgeInsets.all(32),
              child: Text('ยังไม่มี plugins'),
            ),
          )
        else
          ...plugins.map((plugin) {
            final widget = plugin.buildWidget(context);
            if (widget == null) return const SizedBox();
            
            return Padding(
              padding: const EdgeInsets.symmetric(
                horizontal: 16,
                vertical: 4,
              ),
              child: Stack(
                children: [
                  widget,
                  Positioned(
                    top: 4,
                    right: 4,
                    child: IconButton(
                      icon: const Icon(Icons.close, size: 16),
                      onPressed: () => _registry.unregister(plugin.id),
                    ),
                  ),
                ],
              ),
            );
          }),
      ],
    );
  }
  
  void _showAddPluginDialog() {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('เพิ่ม Plugin'),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            ListTile(
              title: const Text('Weather Widget'),
              onTap: () {
                _registry.register(WeatherPlugin());
                Navigator.pop(context);
              },
            ),
            ListTile(
              title: const Text('News Feed'),
              onTap: () {
                _registry.register(NewsPlugin());
                Navigator.pop(context);
              },
            ),
          ],
        ),
      ),
    );
  }
  
  @override
  void dispose() {
    _registry.removeListener(_onPluginsChanged);
    super.dispose();
  }
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้ Design Patterns:

| Pattern | ประเภท | วัตถุประสงค์ |
|---------|--------|------------|
| Singleton | Creational | Instance เดียว |
| Factory | Creational | สร้าง objects |
| Builder | Creational | สร้าง complex objects |
| Observer | Behavioral | แจ้งเตือนเมื่อเปลี่ยน |
| Strategy | Behavioral | เปลี่ยน algorithm |
| Command | Behavioral | Undo/Redo |

**แบบฝึกหัดเพิ่มเติม:**
1. Implement Decorator pattern สำหรับ widget styling
2. สร้าง Template Method pattern สำหรับ form pages
3. Implement Proxy pattern สำหรับ caching
4. สร้าง Composite pattern สำหรับ menu system
