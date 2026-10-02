# Part 56: Memory Management ใน Flutter

## ความสำคัญของการจัดการ Memory

การจัดการ Memory อย่างถูกต้องเป็นสิ่งสำคัญมากในการพัฒนา Flutter App เพราะถ้า Memory รั่ว (Memory Leak) จะทำให้:
- App ทำงานช้าลงเรื่อยๆ
- App crash โดยไม่คาดคิด
- แบตเตอรี่หมดเร็ว
- ผู้ใช้ได้รับประสบการณ์ที่แย่

---

## 1. Memory Leaks ใน Flutter

### Memory Leak คืออะไร?

Memory Leak เกิดขึ้นเมื่อ Object ที่ไม่ได้ใช้งานแล้ว แต่ยังถูก Reference อยู่ ทำให้ Garbage Collector ไม่สามารถเก็บคืน Memory ได้

```dart
// ❌ ตัวอย่าง Memory Leak แบบง่าย
class BadWidget extends StatefulWidget {
  @override
  _BadWidgetState createState() => _BadWidgetState();
}

class _BadWidgetState extends State<BadWidget> {
  Timer? _timer;

  @override
  void initState() {
    super.initState();
    // Timer นี้จะทำงานตลอดไปแม้ Widget ถูก dispose แล้ว!
    _timer = Timer.periodic(Duration(seconds: 1), (timer) {
      print('ยังทำงานอยู่...');
    });
  }

  // ❌ ไม่มีการ dispose Timer!
  @override
  Widget build(BuildContext context) {
    return Text('Bad Widget');
  }
}
```

```dart
// ✅ วิธีที่ถูกต้อง
class GoodWidget extends StatefulWidget {
  @override
  _GoodWidgetState createState() => _GoodWidgetState();
}

class _GoodWidgetState extends State<GoodWidget> {
  Timer? _timer;

  @override
  void initState() {
    super.initState();
    _timer = Timer.periodic(Duration(seconds: 1), (timer) {
      print('ทำงานอยู่...');
    });
  }

  @override
  void dispose() {
    _timer?.cancel(); // ✅ ยกเลิก Timer เมื่อ Widget ถูกทำลาย
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Text('Good Widget');
  }
}
```

### ประเภทของ Memory Leaks ใน Flutter

```dart
// 1. Animation Controller ที่ไม่ได้ dispose
class AnimationLeakExample extends StatefulWidget {
  @override
  _AnimationLeakExampleState createState() => _AnimationLeakExampleState();
}

class _AnimationLeakExampleState extends State<AnimationLeakExample>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: Duration(seconds: 2),
    )..repeat();
  }

  @override
  void dispose() {
    _controller.dispose(); // ✅ จำเป็นต้อง dispose!
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _controller,
      builder: (context, child) {
        return Transform.rotate(
          angle: _controller.value * 2 * 3.14159,
          child: Icon(Icons.refresh),
        );
      },
    );
  }
}
```

```dart
// 2. ScrollController ที่ไม่ได้ dispose
class ScrollLeakExample extends StatefulWidget {
  @override
  _ScrollLeakExampleState createState() => _ScrollLeakExampleState();
}

class _ScrollLeakExampleState extends State<ScrollLeakExample> {
  final ScrollController _scrollController = ScrollController();

  @override
  void initState() {
    super.initState();
    _scrollController.addListener(_onScroll);
  }

  void _onScroll() {
    print('Scroll position: ${_scrollController.offset}');
  }

  @override
  void dispose() {
    _scrollController.removeListener(_onScroll);
    _scrollController.dispose(); // ✅ ต้อง dispose!
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      controller: _scrollController,
      itemCount: 100,
      itemBuilder: (context, index) => ListTile(title: Text('Item $index')),
    );
  }
}
```

---

## 2. dispose() อย่างถูกต้อง

### Pattern การ dispose ที่ดี

```dart
class ProperDisposeWidget extends StatefulWidget {
  @override
  _ProperDisposeWidgetState createState() => _ProperDisposeWidgetState();
}

class _ProperDisposeWidgetState extends State<ProperDisposeWidget>
    with SingleTickerProviderStateMixin {
  // ประกาศ resources ทั้งหมด
  late AnimationController _animationController;
  late TextEditingController _textController;
  late FocusNode _focusNode;
  late ScrollController _scrollController;
  StreamSubscription? _subscription;
  Timer? _debounceTimer;

  @override
  void initState() {
    super.initState();

    // Initialize ทั้งหมดใน initState
    _animationController = AnimationController(
      vsync: this,
      duration: Duration(milliseconds: 300),
    );

    _textController = TextEditingController();
    _focusNode = FocusNode();
    _scrollController = ScrollController();

    // เพิ่ม listeners
    _textController.addListener(_onTextChanged);
    _focusNode.addListener(_onFocusChanged);
    _scrollController.addListener(_onScrollChanged);
  }

  void _onTextChanged() {
    // debounce การค้นหา
    _debounceTimer?.cancel();
    _debounceTimer = Timer(Duration(milliseconds: 500), () {
      print('ค้นหา: ${_textController.text}');
    });
  }

  void _onFocusChanged() {
    print('Focus: ${_focusNode.hasFocus}');
  }

  void _onScrollChanged() {
    print('Scroll: ${_scrollController.offset}');
  }

  @override
  void dispose() {
    // ลำดับการ dispose สำคัญ!
    // 1. ยกเลิก async operations ก่อน
    _debounceTimer?.cancel();
    _subscription?.cancel();

    // 2. ลบ listeners ก่อน dispose
    _textController.removeListener(_onTextChanged);
    _focusNode.removeListener(_onFocusChanged);
    _scrollController.removeListener(_onScrollChanged);

    // 3. Dispose controllers
    _animationController.dispose();
    _textController.dispose();
    _focusNode.dispose();
    _scrollController.dispose();

    // 4. เรียก super.dispose() เสมอ
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        TextField(
          controller: _textController,
          focusNode: _focusNode,
        ),
        Expanded(
          child: ListView.builder(
            controller: _scrollController,
            itemCount: 50,
            itemBuilder: (context, index) => ListTile(
              title: Text('Item $index'),
            ),
          ),
        ),
      ],
    );
  }
}
```

### Dispose Pattern สำหรับ ChangeNotifier

```dart
// Provider ที่ต้อง dispose resources
class UserProvider extends ChangeNotifier {
  StreamSubscription? _userSubscription;
  Timer? _refreshTimer;
  final _dio = Dio();

  UserProvider() {
    _startListening();
  }

  void _startListening() {
    _userSubscription = FirebaseAuth.instance.authStateChanges().listen(
      (user) {
        notifyListeners();
      },
    );

    _refreshTimer = Timer.periodic(Duration(minutes: 5), (_) {
      _refreshData();
    });
  }

  Future<void> _refreshData() async {
    // refresh data
  }

  @override
  void dispose() {
    _userSubscription?.cancel();
    _refreshTimer?.cancel();
    _dio.close(); // ปิด HTTP client
    super.dispose();
  }
}
```

---

## 3. WeakReference

### WeakReference คืออะไร?

WeakReference ช่วยให้ Object สามารถถูก Garbage Collected ได้แม้ว่าจะยังมี Reference อยู่

```dart
import 'dart:core';

class CacheManager {
  // ใช้ WeakReference เพื่อป้องกัน Memory Leak
  final Map<String, WeakReference<ExpensiveObject>> _cache = {};

  ExpensiveObject? get(String key) {
    final ref = _cache[key];
    if (ref == null) return null;

    final obj = ref.target; // อาจเป็น null ถ้าถูก GC แล้ว
    if (obj == null) {
      _cache.remove(key); // ทำความสะอาด cache
    }
    return obj;
  }

  void put(String key, ExpensiveObject obj) {
    _cache[key] = WeakReference(obj);
  }

  void cleanup() {
    // ลบ entries ที่ถูก GC แล้ว
    _cache.removeWhere((key, ref) => ref.target == null);
  }
}

class ExpensiveObject {
  final String data;
  final List<int> largeBuffer;

  ExpensiveObject(this.data) : largeBuffer = List.filled(1000000, 0);

  @override
  String toString() => 'ExpensiveObject($data)';
}

// การใช้งาน
void weakReferenceExample() {
  final cache = CacheManager();

  // สร้าง object และเก็บใน cache
  var obj1 = ExpensiveObject('data1');
  cache.put('key1', obj1);

  print(cache.get('key1')); // แสดง: ExpensiveObject(data1)

  // ถ้า obj1 ถูก set เป็น null และ GC ทำงาน
  // cache.get('key1') จะคืนค่า null
  obj1 = ExpensiveObject('replaced'); // ไม่ใช่ null assignment จริง
}
```

### Expando สำหรับ WeakMap-like behavior

```dart
// Expando เป็นเหมือน WeakMap ใน Dart
class MetadataStore {
  final _metadata = Expando<Map<String, dynamic>>();

  void setMetadata(Object object, String key, dynamic value) {
    var meta = _metadata[object];
    if (meta == null) {
      meta = {};
      _metadata[object] = meta;
    }
    meta[key] = value;
  }

  dynamic getMetadata(Object object, String key) {
    return _metadata[object]?[key];
  }
}

// ตัวอย่างการใช้งาน
void expandoExample() {
  final store = MetadataStore();
  
  var widget = SomeWidget();
  store.setMetadata(widget, 'created_at', DateTime.now());
  store.setMetadata(widget, 'user_id', 'user123');
  
  print(store.getMetadata(widget, 'user_id')); // user123
  
  // เมื่อ widget ถูก GC, metadata ก็จะถูก GC ด้วย
}
```

---

## 4. StreamSubscription Cleanup

### การจัดการ StreamSubscription อย่างถูกต้อง

```dart
class StreamManagementWidget extends StatefulWidget {
  @override
  _StreamManagementWidgetState createState() =>
      _StreamManagementWidgetState();
}

class _StreamManagementWidgetState extends State<StreamManagementWidget> {
  // เก็บ subscriptions ทั้งหมดไว้ใน list
  final List<StreamSubscription> _subscriptions = [];

  @override
  void initState() {
    super.initState();
    _setupSubscriptions();
  }

  void _setupSubscriptions() {
    // Subscription 1: Network changes
    final networkSub = NetworkInfo().onConnectivityChanged.listen(
      (result) {
        print('Network changed: $result');
      },
    );
    _subscriptions.add(networkSub);

    // Subscription 2: User data
    final userSub = FirebaseFirestore.instance
        .collection('users')
        .doc('userId')
        .snapshots()
        .listen(
      (snapshot) {
        if (mounted) {
          setState(() {
            // update state
          });
        }
      },
    );
    _subscriptions.add(userSub);

    // Subscription 3: Messages
    final messageSub = messageStream.listen(
      (message) {
        if (mounted) {
          // handle message
        }
      },
      onError: (error) {
        print('Error: $error');
      },
      cancelOnError: false, // ไม่หยุดเมื่อเกิด error
    );
    _subscriptions.add(messageSub);
  }

  @override
  void dispose() {
    // Cancel ทุก subscription ใน dispose
    for (final sub in _subscriptions) {
      sub.cancel();
    }
    _subscriptions.clear();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Container();
  }
}
```

### CompositeSubscription Pattern

```dart
// Helper class สำหรับจัดการ subscriptions หลายตัว
class CompositeSubscription {
  final List<StreamSubscription> _subscriptions = [];
  bool _isDisposed = false;

  void add(StreamSubscription subscription) {
    if (_isDisposed) {
      subscription.cancel();
      return;
    }
    _subscriptions.add(subscription);
  }

  Future<void> dispose() async {
    _isDisposed = true;
    final futures = _subscriptions.map((sub) => sub.cancel()).toList();
    await Future.wait(futures);
    _subscriptions.clear();
  }

  int get length => _subscriptions.length;
}

// Mixin สำหรับ Widget ที่ใช้ Streams
mixin StreamSubscriptionMixin<T extends StatefulWidget> on State<T> {
  final CompositeSubscription _composite = CompositeSubscription();

  void addSubscription(StreamSubscription subscription) {
    _composite.add(subscription);
  }

  @override
  void dispose() {
    _composite.dispose();
    super.dispose();
  }
}

// การใช้งาน
class MyStreamWidget extends StatefulWidget {
  @override
  _MyStreamWidgetState createState() => _MyStreamWidgetState();
}

class _MyStreamWidgetState extends State<MyStreamWidget>
    with StreamSubscriptionMixin {
  @override
  void initState() {
    super.initState();

    // เพิ่ม subscriptions โดยใช้ addSubscription
    addSubscription(
      someStream.listen((data) {
        if (mounted) setState(() {});
      }),
    );

    addSubscription(
      anotherStream.listen((data) {
        if (mounted) setState(() {});
      }),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Container();
  }
}
```

### StreamController lifecycle

```dart
class DataService {
  // ใช้ StreamController สำหรับ broadcast stream
  final _dataController = StreamController<List<String>>.broadcast();

  Stream<List<String>> get dataStream => _dataController.stream;

  final List<String> _data = [];

  Future<void> addItem(String item) async {
    _data.add(item);
    _dataController.add(List.from(_data)); // ส่งข้อมูลใหม่
  }

  Future<void> removeItem(String item) async {
    _data.remove(item);
    _dataController.add(List.from(_data));
  }

  // สำคัญ! ต้อง close StreamController เมื่อไม่ใช้งาน
  void dispose() {
    _dataController.close();
  }
}
```

---

## 5. Image Memory Management

### การจัดการ Memory สำหรับรูปภาพ

```dart
// การโหลดรูปภาพแบบประหยัด Memory
class EfficientImageWidget extends StatelessWidget {
  final String imageUrl;
  final double width;
  final double height;

  const EfficientImageWidget({
    required this.imageUrl,
    required this.width,
    required this.height,
  });

  @override
  Widget build(BuildContext context) {
    return CachedNetworkImage(
      imageUrl: imageUrl,
      width: width,
      height: height,
      // ระบุ cache dimensions เพื่อลด Memory
      memCacheWidth: width.toInt(),
      memCacheHeight: height.toInt(),
      // จำกัด disk cache
      maxWidthDiskCache: (width * 2).toInt(), // 2x สำหรับ retina
      maxHeightDiskCache: (height * 2).toInt(),
      placeholder: (context, url) => ShimmerPlaceholder(
        width: width,
        height: height,
      ),
      errorWidget: (context, url, error) => Icon(Icons.error),
    );
  }
}
```

```dart
// การจัดการ Image Cache
class ImageCacheManager {
  static void configureImageCache() {
    // กำหนดขนาด cache
    PaintingBinding.instance.imageCache.maximumSize = 100; // จำนวน images
    PaintingBinding.instance.imageCache.maximumSizeBytes =
        50 * 1024 * 1024; // 50MB
  }

  static void clearImageCache() {
    PaintingBinding.instance.imageCache.clear();
    PaintingBinding.instance.imageCache.clearLiveImages();
  }

  static void evictImage(String url) {
    final provider = NetworkImage(url);
    provider.evict();
  }

  static ImageCacheStatus? getImageCacheStatus(String url) {
    final provider = NetworkImage(url);
    return PaintingBinding.instance.imageCache.statusForKey(provider);
  }
}

// ใช้งานใน main.dart
void main() {
  WidgetsFlutterBinding.ensureInitialized();
  ImageCacheManager.configureImageCache();
  runApp(MyApp());
}
```

```dart
// Widget สำหรับแสดงรูปขนาดใหญ่อย่างมีประสิทธิภาพ
class LargeImageViewer extends StatefulWidget {
  final List<String> imageUrls;

  const LargeImageViewer({required this.imageUrls});

  @override
  _LargeImageViewerState createState() => _LargeImageViewerState();
}

class _LargeImageViewerState extends State<LargeImageViewer> {
  late PageController _pageController;
  int _currentIndex = 0;

  @override
  void initState() {
    super.initState();
    _pageController = PageController();
  }

  @override
  void dispose() {
    _pageController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return PageView.builder(
      controller: _pageController,
      itemCount: widget.imageUrls.length,
      onPageChanged: (index) {
        setState(() => _currentIndex = index);

        // Evict รูปที่ห่างออกไป 2 หน้า เพื่อประหยัด Memory
        if (index > 2) {
          final url = widget.imageUrls[index - 2];
          NetworkImage(url).evict();
        }
      },
      itemBuilder: (context, index) {
        // แสดงเฉพาะรูปที่ใกล้เคียงกับหน้าปัจจุบัน
        final isNearby = (index - _currentIndex).abs() <= 1;

        return isNearby
            ? CachedNetworkImage(
                imageUrl: widget.imageUrls[index],
                fit: BoxFit.contain,
              )
            : Container(color: Colors.grey[200]); // placeholder สำหรับหน้าไกล
      },
    );
  }
}
```

### ResizeImage สำหรับลด Memory

```dart
// ใช้ ResizeImage เพื่อโหลดรูปในขนาดที่พอดี
class ResizedImageExample extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Image(
      image: ResizeImage(
        NetworkImage('https://example.com/large-image.jpg'),
        width: 200, // resize เป็น 200px
        height: 200,
      ),
      width: 100, // แสดงที่ 100px
      height: 100,
    );
  }
}

// Asset image ที่ resize แล้ว
class ResizedAssetImage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Image(
      image: ResizeImage(
        AssetImage('assets/large_photo.jpg'),
        width: 400,
      ),
    );
  }
}
```

---

## 6. Memory Profiling Tools

### ใช้ Flutter DevTools

```dart
// เพิ่ม debug info สำหรับ Memory Profiling
class MemoryDebugWrapper extends StatefulWidget {
  final Widget child;

  const MemoryDebugWrapper({required this.child});

  @override
  _MemoryDebugWrapperState createState() => _MemoryDebugWrapperState();
}

class _MemoryDebugWrapperState extends State<MemoryDebugWrapper> {
  Timer? _memoryTimer;

  @override
  void initState() {
    super.initState();

    if (kDebugMode) {
      _memoryTimer = Timer.periodic(Duration(seconds: 5), (_) {
        _logMemoryUsage();
      });
    }
  }

  void _logMemoryUsage() {
    final imageCache = PaintingBinding.instance.imageCache;
    debugPrint(
      'Memory: images=${imageCache.currentSize}/${imageCache.maximumSize} '
      'bytes=${imageCache.currentSizeBytes}/${imageCache.maximumSizeBytes}',
    );
  }

  @override
  void dispose() {
    _memoryTimer?.cancel();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => widget.child;
}
```

---

## Workshop: Memory Leak Detection

### สร้างแอปสำหรับตรวจจับและแก้ไข Memory Leaks

```dart
// main.dart
import 'package:flutter/material.dart';
import 'dart:async';

void main() {
  runApp(MemoryLeakDemoApp());
}

class MemoryLeakDemoApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Memory Leak Demo',
      theme: ThemeData(primarySwatch: Colors.blue),
      home: MemoryLeakDemoScreen(),
    );
  }
}

class MemoryLeakDemoScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Memory Leak Demo')),
      body: ListView(
        children: [
          _buildSection('Timer Leak', TimerLeakDemo()),
          _buildSection('Stream Leak', StreamLeakDemo()),
          _buildSection('Controller Leak', ControllerLeakDemo()),
          _buildSection('ถูกต้อง: Fixed Version', FixedDemo()),
        ],
      ),
    );
  }

  Widget _buildSection(String title, Widget demo) {
    return Card(
      margin: EdgeInsets.all(8),
      child: Column(
        children: [
          ListTile(
            title: Text(title, style: TextStyle(fontWeight: FontWeight.bold)),
          ),
          Padding(
            padding: EdgeInsets.all(8),
            child: demo,
          ),
        ],
      ),
    );
  }
}

// ❌ ตัวอย่าง Timer Leak
class TimerLeakDemo extends StatefulWidget {
  @override
  _TimerLeakDemoState createState() => _TimerLeakDemoState();
}

class _TimerLeakDemoState extends State<TimerLeakDemo> {
  int _count = 0;
  Timer? _timer; // ❌ จะรั่วถ้าไม่ dispose

  @override
  void initState() {
    super.initState();
    _timer = Timer.periodic(Duration(seconds: 1), (_) {
      if (mounted) {
        setState(() => _count++);
      }
    });
  }

  // ❌ ไม่มี dispose - Memory Leak!

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text('Count: $_count', style: TextStyle(color: Colors.red)),
        Text('⚠️ Timer ไม่ได้ถูก dispose!',
            style: TextStyle(color: Colors.red, fontSize: 12)),
      ],
    );
  }
}

// ❌ ตัวอย่าง Stream Leak
class StreamLeakDemo extends StatefulWidget {
  @override
  _StreamLeakDemoState createState() => _StreamLeakDemoState();
}

class _StreamLeakDemoState extends State<StreamLeakDemo> {
  String _status = 'รอ...';
  // ❌ ไม่เก็บ subscription ไว้ cancel

  @override
  void initState() {
    super.initState();
    // ❌ ไม่เก็บ subscription
    Stream.periodic(Duration(seconds: 2), (i) => 'Update $i').listen(
      (data) {
        if (mounted) {
          setState(() => _status = data);
        }
      },
    );
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text('Status: $_status'),
        Text('⚠️ Stream ไม่ได้ถูก cancel!',
            style: TextStyle(color: Colors.orange, fontSize: 12)),
      ],
    );
  }
}

// ❌ ตัวอย่าง Controller Leak
class ControllerLeakDemo extends StatefulWidget {
  @override
  _ControllerLeakDemoState createState() => _ControllerLeakDemoState();
}

class _ControllerLeakDemoState extends State<ControllerLeakDemo>
    with SingleTickerProviderStateMixin {
  late AnimationController _animController;
  late TextEditingController _textController;

  @override
  void initState() {
    super.initState();
    _animController = AnimationController(
      vsync: this,
      duration: Duration(seconds: 1),
    )..repeat(reverse: true);
    _textController = TextEditingController(text: 'test');
  }

  // ❌ ไม่มี dispose

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        FadeTransition(
          opacity: _animController,
          child: Text('กะพริบ'),
        ),
        TextField(controller: _textController),
        Text('⚠️ Controllers ไม่ได้ถูก dispose!',
            style: TextStyle(color: Colors.red, fontSize: 12)),
      ],
    );
  }
}

// ✅ ตัวอย่างที่ถูกต้อง
class FixedDemo extends StatefulWidget {
  @override
  _FixedDemoState createState() => _FixedDemoState();
}

class _FixedDemoState extends State<FixedDemo>
    with SingleTickerProviderStateMixin {
  late AnimationController _animController;
  late TextEditingController _textController;
  int _count = 0;
  Timer? _timer;
  StreamSubscription? _subscription;
  String _status = 'รอ...';

  @override
  void initState() {
    super.initState();

    _animController = AnimationController(
      vsync: this,
      duration: Duration(seconds: 1),
    )..repeat(reverse: true);

    _textController = TextEditingController(text: 'test');

    _timer = Timer.periodic(Duration(seconds: 1), (_) {
      if (mounted) setState(() => _count++);
    });

    _subscription = Stream.periodic(
      Duration(seconds: 2),
      (i) => 'Update $i',
    ).listen((data) {
      if (mounted) setState(() => _status = data);
    });
  }

  @override
  void dispose() {
    // ✅ Dispose ทุกอย่าง!
    _animController.dispose();
    _textController.dispose();
    _timer?.cancel();
    _subscription?.cancel();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        FadeTransition(
          opacity: _animController,
          child: Text('กะพริบอย่างถูกต้อง'),
        ),
        TextField(controller: _textController),
        Text('Count: $_count'),
        Text('Status: $_status'),
        Text('✅ ทุกอย่างถูก dispose อย่างถูกต้อง!',
            style: TextStyle(color: Colors.green, fontSize: 12)),
      ],
    );
  }
}
```

### Memory Monitor Widget

```dart
// Widget สำหรับ monitor memory ใน debug mode
class MemoryMonitor extends StatefulWidget {
  final Widget child;

  const MemoryMonitor({required this.child, Key? key}) : super(key: key);

  @override
  _MemoryMonitorState createState() => _MemoryMonitorState();
}

class _MemoryMonitorState extends State<MemoryMonitor> {
  Timer? _timer;
  String _memoryInfo = '';

  @override
  void initState() {
    super.initState();
    if (kDebugMode) {
      _timer = Timer.periodic(Duration(seconds: 3), (_) {
        setState(() {
          final cache = PaintingBinding.instance.imageCache;
          _memoryInfo =
              'Images: ${cache.currentSize} | '
              'Bytes: ${(cache.currentSizeBytes / 1024 / 1024).toStringAsFixed(2)}MB';
        });
      });
    }
  }

  @override
  void dispose() {
    _timer?.cancel();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Stack(
      children: [
        widget.child,
        if (kDebugMode && _memoryInfo.isNotEmpty)
          Positioned(
            bottom: 0,
            left: 0,
            right: 0,
            child: Container(
              color: Colors.black54,
              padding: EdgeInsets.all(4),
              child: Text(
                _memoryInfo,
                style: TextStyle(color: Colors.white, fontSize: 10),
                textAlign: TextAlign.center,
              ),
            ),
          ),
      ],
    );
  }
}
```

### สรุปแนวปฏิบัติที่ดีสำหรับ Memory Management

```dart
// Checklist สำหรับ Memory Management ใน Flutter
class MemoryManagementChecklist {
  /*
  ✅ ต้องทำ:
  1. dispose() ทุก AnimationController
  2. dispose() ทุก TextEditingController
  3. dispose() ทุก FocusNode
  4. dispose() ทุก ScrollController
  5. cancel() ทุก StreamSubscription
  6. cancel() ทุก Timer
  7. close() ทุก StreamController
  8. ตรวจสอบ mounted ก่อน setState()
  9. ใช้ ResizeImage สำหรับรูปขนาดใหญ่
  10. กำหนด Image Cache size ที่เหมาะสม

  ❌ ห้ามทำ:
  1. เพิ่ม listener โดยไม่ remove
  2. สร้าง Object ใน build() ที่ต้อง dispose
  3. เก็บ context ไว้ใน async operation โดยไม่ตรวจ mounted
  4. ลืม super.dispose() ใน dispose()
  5. ใช้ static field เก็บ Widget หรือ BuildContext
  */
}
```

---

## สรุป

การจัดการ Memory ที่ดีใน Flutter ประกอบด้วย:

1. **dispose()** - เรียกทุกครั้งเมื่อ Widget ถูกทำลาย
2. **StreamSubscription** - cancel() ทุกตัวใน dispose()
3. **WeakReference** - ใช้เมื่อต้องการ reference โดยไม่ block GC
4. **Image Cache** - กำหนดขนาดที่เหมาะสม
5. **Timer** - cancel() ทุกตัวใน dispose()

การ profile memory สม่ำเสมอด้วย Flutter DevTools จะช่วยตรวจจับ Memory Leaks ได้ก่อนที่จะส่งผลกระทบต่อผู้ใช้
