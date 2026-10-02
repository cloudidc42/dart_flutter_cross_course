# Part 80: Advanced Performance Profiling - การ Profiling ขั้นสูง

## บทนำ

Performance profiling คือกระบวนการวิเคราะห์หาปัญหาด้านประสิทธิภาพในแอป Flutter ก่อนปรับปรุงได้ ต้องรู้ก่อนว่าปัญหาอยู่ที่ไหน DevTools ของ Flutter มีเครื่องมือครบครันสำหรับงานนี้

## 1. DevTools Deep Dive

### เปิด DevTools

```bash
# เปิดจาก terminal
flutter pub global activate devtools
flutter devtools

# หรือ run app ในโหมด profile
flutter run --profile

# เปิด DevTools จาก URL ที่แสดงใน terminal
```

### DevTools Panels

1. **Flutter Inspector** - ดู widget tree, layout issues
2. **Performance** - Frame rendering analysis
3. **CPU Profiler** - CPU usage analysis
4. **Memory** - Memory allocation profiling
5. **Network** - HTTP request monitoring
6. **Logging** - App logs

## 2. CPU Profiling

### ทำความเข้าใจ CPU Profile

```
CPU Profile แสดง:
- Stack frames (ฟังก์ชันที่กำลัง execute)
- Call tree (ความสัมพันธ์ระหว่างฟังก์ชัน)
- Time spent ในแต่ละฟังก์ชัน
- Bottom-up view (ฟังก์ชันที่ใช้เวลามากที่สุด)
```

### Code สำหรับ Profile

```dart
// lib/performance/expensive_operations.dart
import 'dart:isolate';

// ตัวอย่างโค้ดที่ใช้ CPU มาก
class ExpensiveOperations {
  // BAD: รัน expensive computation บน main thread
  static List<int> computePrimesSync(int limit) {
    final sieve = List<bool>.filled(limit + 1, true);
    sieve[0] = false;
    sieve[1] = false;

    for (int i = 2; i * i <= limit; i++) {
      if (sieve[i]) {
        for (int j = i * i; j <= limit; j += i) {
          sieve[j] = false;
        }
      }
    }

    return [
      for (int i = 2; i <= limit; i++)
        if (sieve[i]) i
    ];
  }

  // GOOD: รัน expensive computation บน Isolate
  static Future<List<int>> computePrimesAsync(int limit) async {
    return Isolate.run(() => computePrimesSync(limit));
  }

  // GOOD: ใช้ compute() สำหรับ simple tasks
  static Future<List<int>> computePrimesWithCompute(int limit) async {
    return compute(_computePrimesInIsolate, limit);
  }
}

// Top-level function สำหรับ compute()
List<int> _computePrimesInIsolate(int limit) {
  return ExpensiveOperations.computePrimesSync(limit);
}
```

### Timeline Events

```dart
// lib/performance/timeline_helper.dart
import 'package:flutter/foundation.dart';

class TimelineHelper {
  static T measure<T>(String name, T Function() action) {
    if (!kProfileMode) return action();

    // เพิ่ม timeline event สำหรับ custom profiling
    Timeline.startSync(name);
    try {
      return action();
    } finally {
      Timeline.finishSync();
    }
  }

  static Future<T> measureAsync<T>(
    String name,
    Future<T> Function() action,
  ) async {
    if (!kProfileMode) return action();

    final flow = Flow.begin();
    Timeline.startSync(name, flow: flow);
    try {
      final result = await action();
      return result;
    } finally {
      Timeline.finishSync();
      Timeline.startSync('$name.end', flow: Flow.end(flow.id));
      Timeline.finishSync();
    }
  }
}
```

## 3. Memory Profiling

### Memory Leak Detection

```dart
// lib/performance/memory_tracker.dart
import 'package:flutter/foundation.dart';

// Tracker สำหรับตรวจสอบ memory leaks
class MemoryTracker {
  static final Map<String, int> _liveObjects = {};

  static void trackAllocation(String className) {
    if (!kDebugMode) return;
    _liveObjects[className] = (_liveObjects[className] ?? 0) + 1;
    debugPrint('Allocated: $className (${_liveObjects[className]} live)');
  }

  static void trackDeallocation(String className) {
    if (!kDebugMode) return;
    _liveObjects[className] = (_liveObjects[className] ?? 1) - 1;
    if (_liveObjects[className] == 0) {
      _liveObjects.remove(className);
    }
  }

  static void printLiveObjects() {
    if (!kDebugMode) return;
    debugPrint('=== Live Objects ===');
    _liveObjects.forEach((key, value) {
      debugPrint('  $key: $value');
    });
    debugPrint('====================');
  }
}

// Mixin สำหรับ track objects
mixin TrackedObject {
  void initTracking() {
    MemoryTracker.trackAllocation(runtimeType.toString());
  }

  void disposeTracking() {
    MemoryTracker.trackDeallocation(runtimeType.toString());
  }
}
```

### Common Memory Issues

```dart
// lib/performance/memory_issues.dart
import 'dart:async';
import 'package:flutter/material.dart';

// BAD: StreamController ที่ไม่ได้ close
class BadWidget extends StatefulWidget {
  const BadWidget({super.key});

  @override
  State<BadWidget> createState() => _BadWidgetState();
}

class _BadWidgetState extends State<BadWidget> {
  // MEMORY LEAK! ไม่ได้ close
  final _controller = StreamController<int>();
  late StreamSubscription _subscription;

  @override
  void initState() {
    super.initState();
    // MEMORY LEAK! ไม่ได้ cancel
    _subscription = _controller.stream.listen((event) {});
  }

  // ลืม override dispose!

  @override
  Widget build(BuildContext context) => const SizedBox();
}

// GOOD: Proper cleanup
class GoodWidget extends StatefulWidget {
  const GoodWidget({super.key});

  @override
  State<GoodWidget> createState() => _GoodWidgetState();
}

class _GoodWidgetState extends State<GoodWidget> {
  StreamController<int>? _controller;
  StreamSubscription? _subscription;

  @override
  void initState() {
    super.initState();
    _controller = StreamController<int>();
    _subscription = _controller!.stream.listen((event) {});
  }

  @override
  void dispose() {
    _subscription?.cancel();
    _controller?.close();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => const SizedBox();
}
```

### Image Memory Optimization

```dart
// lib/performance/image_optimizer.dart
import 'package:flutter/material.dart';
import 'package:cached_network_image/cached_network_image.dart';

class OptimizedImage extends StatelessWidget {
  final String url;
  final double? width;
  final double? height;
  final BoxFit fit;

  const OptimizedImage({
    super.key,
    required this.url,
    this.width,
    this.height,
    this.fit = BoxFit.cover,
  });

  @override
  Widget build(BuildContext context) {
    // คำนวณ physical pixel size สำหรับ cache
    final devicePixelRatio = MediaQuery.devicePixelRatioOf(context);
    final cacheWidth =
        width != null ? (width! * devicePixelRatio).toInt() : null;
    final cacheHeight =
        height != null ? (height! * devicePixelRatio).toInt() : null;

    return CachedNetworkImage(
      imageUrl: url,
      width: width,
      height: height,
      fit: fit,
      // Resize image ตาม display size เพื่อประหยัด memory
      memCacheWidth: cacheWidth,
      memCacheHeight: cacheHeight,
      // Placeholder ขณะโหลด
      placeholder: (context, url) => Container(
        width: width,
        height: height,
        color: Colors.grey[200],
        child: const Center(child: CircularProgressIndicator()),
      ),
      errorWidget: (context, url, error) => Container(
        width: width,
        height: height,
        color: Colors.grey[200],
        child: const Icon(Icons.error),
      ),
    );
  }
}
```

## 4. Network Profiling

### Network Monitor

```dart
// lib/performance/network_monitor.dart
import 'dart:developer';
import 'package:dio/dio.dart';

class NetworkMonitorInterceptor extends Interceptor {
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    final startTime = DateTime.now();
    options.extra['start_time'] = startTime;

    // Log ใน timeline
    Timeline.startSync(
      'HTTP ${options.method} ${options.uri.path}',
      arguments: {
        'url': options.uri.toString(),
        'method': options.method,
      },
    );

    handler.next(options);
  }

  @override
  void onResponse(Response response, ResponseInterceptorHandler handler) {
    final startTime = response.requestOptions.extra['start_time'] as DateTime?;
    final duration = startTime != null
        ? DateTime.now().difference(startTime)
        : Duration.zero;

    Timeline.finishSync();

    // Log performance warning ถ้าช้าเกิน 2 วินาที
    if (duration.inMilliseconds > 2000) {
      debugPrint(
        '⚠️ SLOW REQUEST: ${response.requestOptions.method} '
        '${response.requestOptions.uri.path} '
        '(${duration.inMilliseconds}ms)',
      );
    }

    handler.next(response);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    Timeline.finishSync();
    handler.next(err);
  }
}
```

## 5. Frame Rendering Analysis

### Jank Detection

```dart
// lib/performance/jank_detector.dart
import 'package:flutter/scheduler.dart';

class JankDetector {
  static const int _targetFPS = 60;
  static const double _targetFrameTime =
      1000.0 / _targetFPS; // ~16.67ms

  late final Ticker _ticker;
  int _frameCount = 0;
  Duration? _lastFrameTime;
  final List<double> _frameTimes = [];

  void start(TickerProvider vsync) {
    _ticker = vsync.createTicker((elapsed) {
      _processFrame(elapsed);
    });
    _ticker.start();
  }

  void _processFrame(Duration elapsed) {
    if (_lastFrameTime != null) {
      final frameTime =
          elapsed.inMicroseconds - _lastFrameTime!.inMicroseconds;
      final frameTimeMs = frameTime / 1000.0;

      _frameTimes.add(frameTimeMs);
      _frameCount++;

      if (frameTimeMs > _targetFrameTime * 2) {
        debugPrint(
          '⚠️ JANK DETECTED: Frame took ${frameTimeMs.toStringAsFixed(1)}ms '
          '(target: ${_targetFrameTime.toStringAsFixed(1)}ms)',
        );
      }
    }

    _lastFrameTime = elapsed;
  }

  double get averageFPS {
    if (_frameTimes.isEmpty) return 0;
    final avgFrameTime =
        _frameTimes.reduce((a, b) => a + b) / _frameTimes.length;
    return 1000.0 / avgFrameTime;
  }

  void stop() {
    _ticker.stop();
    _ticker.dispose();
  }
}
```

### Widget Rebuild Tracker

```dart
// lib/performance/rebuild_tracker.dart
import 'package:flutter/material.dart';

// Mixin สำหรับ track widget rebuilds
mixin RebuildTracker<T extends StatefulWidget> on State<T> {
  int _rebuildCount = 0;

  @override
  Widget build(BuildContext context) {
    _rebuildCount++;
    if (_rebuildCount > 1) {
      debugPrint(
        '🔄 REBUILD #$_rebuildCount: ${widget.runtimeType}',
      );
    }
    return buildTracked(context);
  }

  Widget buildTracked(BuildContext context);
}

// Widget ที่แสดง rebuild count
class RebuildCounter extends StatefulWidget {
  final Widget child;

  const RebuildCounter({super.key, required this.child});

  @override
  State<RebuildCounter> createState() => _RebuildCounterState();
}

class _RebuildCounterState extends State<RebuildCounter> {
  int _count = 0;

  @override
  Widget build(BuildContext context) {
    _count++;
    return Stack(
      children: [
        widget.child,
        if (const bool.fromEnvironment('SHOW_REBUILD_COUNT'))
          Positioned(
            top: 0,
            right: 0,
            child: Container(
              color: Colors.red.withOpacity(0.8),
              padding: const EdgeInsets.all(2),
              child: Text(
                '$_count',
                style: const TextStyle(
                  color: Colors.white,
                  fontSize: 10,
                ),
              ),
            ),
          ),
      ],
    );
  }
}
```

## 6. Workshop: Profile and Optimize Real App

### Performance Test Screen

```dart
// lib/screens/performance_demo.dart
import 'package:flutter/material.dart';
import '../performance/expensive_operations.dart';
import '../performance/timeline_helper.dart';

class PerformanceDemo extends StatefulWidget {
  const PerformanceDemo({super.key});

  @override
  State<PerformanceDemo> createState() => _PerformanceDemoState();
}

class _PerformanceDemoState extends State<PerformanceDemo> {
  List<int>? _primes;
  bool _isLoading = false;
  Duration? _computeTime;

  Future<void> _computeOnMainThread() async {
    setState(() {
      _isLoading = true;
      _primes = null;
    });

    final stopwatch = Stopwatch()..start();
    // BAD: blocks UI
    final primes = ExpensiveOperations.computePrimesSync(1000000);
    stopwatch.stop();

    setState(() {
      _primes = primes;
      _isLoading = false;
      _computeTime = stopwatch.elapsed;
    });
  }

  Future<void> _computeInIsolate() async {
    setState(() {
      _isLoading = true;
      _primes = null;
    });

    final stopwatch = Stopwatch()..start();
    // GOOD: doesn't block UI
    final primes = await TimelineHelper.measureAsync(
      'ComputePrimes',
      () => ExpensiveOperations.computePrimesAsync(1000000),
    );
    stopwatch.stop();

    setState(() {
      _primes = primes;
      _isLoading = false;
      _computeTime = stopwatch.elapsed;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Performance Demo'),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            // Animated indicator เพื่อแสดงว่า UI ยัง responsive
            const AnimatedLoadingBar(),
            const SizedBox(height: 24),

            Row(
              children: [
                Expanded(
                  child: ElevatedButton(
                    onPressed: _isLoading ? null : _computeOnMainThread,
                    style: ElevatedButton.styleFrom(
                      backgroundColor: Colors.red[100],
                    ),
                    child: const Column(
                      children: [
                        Icon(Icons.warning, color: Colors.red),
                        Text('Main Thread\n(Blocks UI)',
                            textAlign: TextAlign.center),
                      ],
                    ),
                  ),
                ),
                const SizedBox(width: 16),
                Expanded(
                  child: ElevatedButton(
                    onPressed: _isLoading ? null : _computeInIsolate,
                    style: ElevatedButton.styleFrom(
                      backgroundColor: Colors.green[100],
                    ),
                    child: const Column(
                      children: [
                        Icon(Icons.check_circle, color: Colors.green),
                        Text('Isolate\n(Non-blocking)',
                            textAlign: TextAlign.center),
                      ],
                    ),
                  ),
                ),
              ],
            ),

            const SizedBox(height: 24),

            if (_isLoading)
              const CircularProgressIndicator()
            else if (_primes != null) ...[
              Text(
                'พบ ${_primes!.length} จำนวนเฉพาะ',
                style: Theme.of(context).textTheme.headlineSmall,
              ),
              if (_computeTime != null)
                Text(
                  'ใช้เวลา: ${_computeTime!.inMilliseconds}ms',
                  style: Theme.of(context).textTheme.bodyLarge,
                ),
            ],
          ],
        ),
      ),
    );
  }
}

// Animated indicator ที่แสดงว่า UI ยัง responsive
class AnimatedLoadingBar extends StatefulWidget {
  const AnimatedLoadingBar({super.key});

  @override
  State<AnimatedLoadingBar> createState() => _AnimatedLoadingBarState();
}

class _AnimatedLoadingBarState extends State<AnimatedLoadingBar>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 2),
    )..repeat();
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        const Text('UI Responsiveness:'),
        const SizedBox(height: 8),
        AnimatedBuilder(
          animation: _controller,
          builder: (context, _) {
            return LinearProgressIndicator(
              value: _controller.value,
              backgroundColor: Colors.grey[200],
            );
          },
        ),
      ],
    );
  }
}
```

### Performance Checklist Widget

```dart
// lib/screens/performance_checklist.dart
import 'package:flutter/material.dart';

class PerformanceChecklistScreen extends StatefulWidget {
  const PerformanceChecklistScreen({super.key});

  @override
  State<PerformanceChecklistScreen> createState() =>
      _PerformanceChecklistScreenState();
}

class _PerformanceChecklistScreenState
    extends State<PerformanceChecklistScreen> {
  final List<ChecklistItem> _items = [
    ChecklistItem(
      title: 'ใช้ const widgets',
      description: 'เพิ่ม const keyword กับ stateless widgets',
    ),
    ChecklistItem(
      title: 'ย้าย computation ออกจาก build()',
      description: 'คำนวณค่าใน initState() หรือ provider',
    ),
    ChecklistItem(
      title: 'ใช้ ListView.builder แทน Column',
      description: 'สำหรับ list ขนาดใหญ่ต้องใช้ lazy loading',
    ),
    ChecklistItem(
      title: 'Resize images ตาม display size',
      description: 'ใช้ cacheWidth/cacheHeight เพื่อประหยัด memory',
    ),
    ChecklistItem(
      title: 'ใช้ RepaintBoundary',
      description: 'แยก widgets ที่ animate บ่อยออกจาก parent',
    ),
    ChecklistItem(
      title: 'Isolate สำหรับ heavy computation',
      description: 'ใช้ Isolate.run() หรือ compute()',
    ),
    ChecklistItem(
      title: 'Cancel subscriptions ใน dispose()',
      description: 'ป้องกัน memory leaks จาก streams',
    ),
    ChecklistItem(
      title: 'ใช้ StreamBuilder/FutureBuilder อย่างถูกต้อง',
      description: 'ระวัง unnecessary rebuilds',
    ),
  ];

  @override
  Widget build(BuildContext context) {
    final checkedCount = _items.where((i) => i.isChecked).length;

    return Scaffold(
      appBar: AppBar(
        title: const Text('Performance Checklist'),
        actions: [
          Chip(
            label: Text('$checkedCount/${_items.length}'),
            backgroundColor: checkedCount == _items.length
                ? Colors.green[100]
                : Colors.grey[200],
          ),
          const SizedBox(width: 8),
        ],
      ),
      body: ListView.builder(
        itemCount: _items.length,
        itemBuilder: (context, index) {
          final item = _items[index];
          return CheckboxListTile(
            value: item.isChecked,
            onChanged: (value) {
              setState(() => item.isChecked = value ?? false);
            },
            title: Text(
              item.title,
              style: TextStyle(
                decoration:
                    item.isChecked ? TextDecoration.lineThrough : null,
                color: item.isChecked ? Colors.grey : null,
              ),
            ),
            subtitle: Text(item.description),
          );
        },
      ),
    );
  }
}

class ChecklistItem {
  final String title;
  final String description;
  bool isChecked;

  ChecklistItem({
    required this.title,
    required this.description,
    this.isChecked = false,
  });
}
```

## สรุป

Advanced Profiling:
1. **Profile Mode** สำคัญ - อย่า profile ใน debug mode
2. **DevTools** มีเครื่องมือครบ - CPU, Memory, Network, Frames
3. **Timeline events** ช่วย correlate code กับ performance
4. **Isolates** แก้ปัญหา CPU-heavy tasks

## แบบทดสอบ

1. ทำไมต้อง profile ใน `--profile` mode แทน `--debug`?
2. อธิบาย "jank" คืออะไรและเกิดจากอะไรได้บ้าง
3. ใช้ DevTools หา memory leak ใน widget ที่ไม่ dispose StreamController
4. เมื่อไหร่ควรใช้ `RepaintBoundary` และมีผลกระทบอะไรบ้าง?
