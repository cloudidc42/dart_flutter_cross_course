# Part 13: Streams ใน Dart

## บทนำ

Stream คือลำดับของ events ที่เกิดขึ้นตามเวลา เหมือน "ท่อ" ที่ข้อมูลไหลผ่าน แตกต่างจาก Future ที่ให้ค่าเดียว Stream สามารถให้ค่าหลายครั้งได้ ตัวอย่างในชีวิตจริง: ข้อมูล GPS, ข้อมูลตลาดหุ้น, การพิมพ์ในช่องค้นหา

## 13.1 Stream Basics

### สร้าง Stream อย่างง่าย

```dart
import 'dart:async';

// Stream จาก Iterable
Stream<int> countStream(int max) async* {
  for (var i = 1; i <= max; i++) {
    await Future.delayed(Duration(milliseconds: 500));
    yield i; // ส่งค่าออกไปทีละครั้ง
  }
}

// Stream จาก Future
Stream<String> messageStream() async* {
  yield 'สวัสดี';
  await Future.delayed(Duration(seconds: 1));
  yield 'จาก Stream';
  await Future.delayed(Duration(seconds: 1));
  yield 'ลาก่อน!';
}

// การใช้งาน Stream
Future<void> listenToStream() async {
  print('=== Stream Basics ===');

  // วิธีที่ 1: await for loop
  await for (final count in countStream(5)) {
    print('นับ: $count');
  }

  // วิธีที่ 2: listen
  final subscription = messageStream().listen(
    (msg) => print('ข้อความ: $msg'),
    onError: (e) => print('Error: $e'),
    onDone: () => print('Stream สิ้นสุด'),
    cancelOnError: false,
  );

  // รอให้ stream เสร็จ
  await subscription.asFuture();
}

// Stream types
void streamTypes() {
  // Single-subscription stream (อ่านได้ครั้งเดียว)
  var singleStream = Stream.fromIterable([1, 2, 3]);

  // Broadcast stream (อ่านได้หลายคน)
  var broadcastStream = StreamController<int>.broadcast().stream;

  print('Single: $singleStream');
  print('Broadcast: $broadcastStream');
}

// Stream properties
Stream<int> demonstrateStreamProperties() {
  return Stream.periodic(Duration(seconds: 1), (i) => i)
      .take(10); // เอาแค่ 10 ค่าแรก
}
```

### Stream Constructors

```dart
void streamConstructors() {
  // Stream.value - emit ค่าเดียวแล้วจบ
  var single = Stream.value(42);

  // Stream.error - emit error แล้วจบ
  var errorStream = Stream<int>.error(Exception('เกิดข้อผิดพลาด'));

  // Stream.empty - ไม่มีค่า จบทันที
  var empty = Stream<int>.empty();

  // Stream.fromIterable
  var fromList = Stream.fromIterable([1, 2, 3, 4, 5]);

  // Stream.fromFuture - แปลง Future เป็น Stream
  var fromFuture = Stream.fromFuture(
    Future.delayed(Duration(seconds: 1), () => 'ค่าจาก Future'),
  );

  // Stream.fromFutures - รวมหลาย Future เป็น Stream
  var fromFutures = Stream.fromFutures([
    Future.delayed(Duration(seconds: 3), () => 'สาม'),
    Future.delayed(Duration(seconds: 1), () => 'หนึ่ง'),
    Future.delayed(Duration(seconds: 2), () => 'สอง'),
  ]);
  // จะ emit ตามลำดับที่เสร็จก่อน

  // Stream.periodic - ส่งค่าตามเวลา
  var periodic = Stream.periodic(
    Duration(seconds: 1),
    (count) => 'tick $count',
  ).take(5);
}
```

### StreamSubscription

```dart
Future<void> manageSubscription() async {
  print('=== StreamSubscription Management ===');

  final stream = Stream.periodic(Duration(milliseconds: 500), (i) => i).take(10);

  final subscription = stream.listen(
    (value) => print('ค่า: $value'),
    onError: (e) => print('Error: $e'),
    onDone: () => print('เสร็จ'),
  );

  // หยุดชั่วคราว
  await Future.delayed(Duration(seconds: 2));
  subscription.pause();
  print('หยุดชั่วคราว...');

  // กลับมาทำงาน
  await Future.delayed(Duration(seconds: 1));
  subscription.resume();
  print('กลับมาทำงาน...');

  // รอให้เสร็จ
  await subscription.asFuture();

  // หรือยกเลิก
  // await subscription.cancel();
}
```

## 13.2 StreamController

StreamController ให้เราสร้างและควบคุม stream เองได้

```dart
// Single-subscription StreamController
StreamController<String> createController() {
  final controller = StreamController<String>(
    onListen: () => print('มีคนเริ่ม listen'),
    onPause: () => print('Stream ถูก pause'),
    onResume: () => print('Stream ถูก resume'),
    onCancel: () => print('การ listen ถูกยกเลิก'),
  );
  return controller;
}

Future<void> demonstrateStreamController() async {
  print('=== StreamController Demo ===');

  final controller = StreamController<String>();

  // Listener
  final subscription = controller.stream.listen(
    (msg) => print('ได้รับ: $msg'),
    onError: (e) => print('Error: $e'),
    onDone: () => print('Controller ปิดแล้ว'),
  );

  // ส่งข้อมูล
  controller.add('ข้อความที่ 1');
  controller.add('ข้อความที่ 2');
  controller.addError(Exception('ข้อผิดพลาดทดสอบ'));
  controller.add('ข้อความที่ 3');

  // ปิด controller
  await controller.close();
  await subscription.asFuture().catchError((_) {});

  print('StreamController.isClosed: ${controller.isClosed}');
}

// Broadcast StreamController
class EventBus {
  final _controller = StreamController<AppEvent>.broadcast();

  Stream<T> on<T extends AppEvent>() {
    return _controller.stream.whereType<T>();
  }

  void emit(AppEvent event) {
    if (!_controller.isClosed) {
      _controller.add(event);
    }
  }

  void dispose() => _controller.close();
}

abstract class AppEvent {
  final DateTime timestamp = DateTime.now();
}

class UserLoggedIn extends AppEvent {
  final String userId;
  UserLoggedIn(this.userId);
}

class DataRefreshed extends AppEvent {
  final String dataType;
  DataRefreshed(this.dataType);
}

class ErrorOccurred extends AppEvent {
  final String message;
  ErrorOccurred(this.message);
}

Future<void> testEventBus() async {
  final bus = EventBus();

  // Multiple listeners
  bus.on<UserLoggedIn>().listen((e) =>
    print('ผู้ใช้ login: ${e.userId}'));

  bus.on<DataRefreshed>().listen((e) =>
    print('ข้อมูลถูก refresh: ${e.dataType}'));

  bus.on<ErrorOccurred>().listen((e) =>
    print('เกิดข้อผิดพลาด: ${e.message}'));

  // Emit events
  bus.emit(UserLoggedIn('user_123'));
  bus.emit(DataRefreshed('products'));
  bus.emit(ErrorOccurred('Connection timeout'));
  bus.emit(UserLoggedIn('user_456'));

  await Future.delayed(Duration(milliseconds: 100));
  bus.dispose();
}

// StreamController แบบ Sink
class DataBuffer<T> {
  final _controller = StreamController<T>();

  StreamSink<T> get sink => _controller.sink;
  Stream<T> get stream => _controller.stream;

  void add(T data) => sink.add(data);
  void addError(Object error) => sink.addError(error);
  Future<void> close() => _controller.close();
}
```

## 13.3 Stream Transformations

### Built-in transformations

```dart
Future<void> streamTransformations() async {
  print('=== Stream Transformations ===');

  var numbers = Stream.fromIterable(List.generate(10, (i) => i + 1));

  // map - แปลงแต่ละค่า
  var doubled = numbers.map((n) => n * 2);
  print('doubled: ${await doubled.toList()}');

  // where - กรองค่า
  var evenNumbers = Stream.fromIterable(List.generate(10, (i) => i + 1))
      .where((n) => n.isEven);
  print('even: ${await evenNumbers.toList()}');

  // take - เอาแค่ n ค่าแรก
  var firstThree = Stream.fromIterable(List.generate(100, (i) => i))
      .take(3);
  print('first 3: ${await firstThree.toList()}');

  // skip - ข้าม n ค่าแรก
  var skipTwo = Stream.fromIterable([1, 2, 3, 4, 5])
      .skip(2);
  print('skip 2: ${await skipTwo.toList()}');

  // takeWhile / skipWhile
  var untilFive = Stream.fromIterable(List.generate(10, (i) => i + 1))
      .takeWhile((n) => n <= 5);
  print('until 5: ${await untilFive.toList()}');

  // expand - หนึ่งค่าเป็นหลายค่า
  var expanded = Stream.fromIterable([1, 2, 3])
      .expand((n) => [n, n * 10]);
  print('expanded: ${await expanded.toList()}');

  // distinct - กรองค่าซ้ำ
  var distinct = Stream.fromIterable([1, 1, 2, 2, 3, 1, 2])
      .distinct();
  print('distinct: ${await distinct.toList()}');
}

// asyncMap - แปลงค่าแบบ async
Future<void> asyncMap() async {
  var userIds = Stream.fromIterable([1, 2, 3]);

  var userNames = userIds.asyncMap((id) async {
    await Future.delayed(Duration(milliseconds: 100));
    return 'User $id';
  });

  await for (final name in userNames) {
    print('Name: $name');
  }
}

// asyncExpand - flat map แบบ async
Future<void> asyncExpand() async {
  var pages = Stream.fromIterable([1, 2]);

  var allItems = pages.asyncExpand((page) async* {
    await Future.delayed(Duration(milliseconds: 100));
    yield 'หน้า$page-รายการ1';
    yield 'หน้า$page-รายการ2';
  });

  await for (final item in allItems) {
    print(item);
  }
}

// handleError
Future<void> handleStreamErrors() async {
  var stream = Stream<int>.periodic(
    Duration(milliseconds: 100),
    (i) {
      if (i == 3) throw Exception('Error at $i');
      return i;
    },
  ).take(6);

  stream
      .handleError(
        (e) => print('จัดการ error: $e'),
        test: (e) => e is Exception,
      )
      .listen(
        (v) => print('ค่า: $v'),
        onDone: () => print('เสร็จ'),
      );

  await Future.delayed(Duration(seconds: 1));
}
```

### Custom StreamTransformer

```dart
// StreamTransformer แบบ custom
StreamTransformer<int, String> numberFormatter() {
  return StreamTransformer.fromHandlers(
    handleData: (data, sink) {
      if (data.isEven) {
        sink.add('เลขคู่: $data');
      } else {
        sink.add('เลขคี่: $data');
      }
    },
    handleError: (error, stackTrace, sink) {
      sink.add('Error: $error');
    },
    handleDone: (sink) {
      sink.add('Stream สิ้นสุด');
      sink.close();
    },
  );
}

// Debounce transformer - สำหรับ search
StreamTransformer<T, T> debounce<T>(Duration duration) {
  return StreamTransformer.fromHandlers(
    handleData: (data, sink) {
      // Implementation ต้องการ Timer จัดการ
      sink.add(data); // simplified
    },
  );
}

// Rate limiter
class RateLimiter<T> extends StreamTransformerBase<T, T> {
  final int maxPerSecond;

  RateLimiter(this.maxPerSecond);

  @override
  Stream<T> bind(Stream<T> stream) async* {
    var count = 0;
    var windowStart = DateTime.now();

    await for (final event in stream) {
      final now = DateTime.now();
      if (now.difference(windowStart).inSeconds >= 1) {
        count = 0;
        windowStart = now;
      }

      if (count < maxPerSecond) {
        count++;
        yield event;
      } else {
        // ข้ามไป
        print('Rate limit: ข้ามข้อมูล $event');
      }
    }
  }
}

Future<void> testRateLimiter() async {
  var fastStream = Stream.periodic(
    Duration(milliseconds: 100),
    (i) => i,
  ).take(20);

  await for (final value in fastStream.transform(RateLimiter(5))) {
    print('ค่า: $value');
  }
}
```

## 13.4 StreamBuilder ใน Flutter

```dart
import 'package:flutter/material.dart';
import 'dart:async';

// Stream ข้อมูล
class StockPrice {
  final String symbol;
  final double price;
  final double change;
  StockPrice(this.symbol, this.price, this.change);
}

class StockService {
  static Stream<StockPrice> getPriceStream(String symbol) {
    var basePrice = 100.0;
    return Stream.periodic(Duration(seconds: 1), (i) {
      var change = (i % 5 - 2) * 0.5;
      basePrice += change;
      return StockPrice(symbol, basePrice, change);
    });
  }
}

// Flutter Widget
class StockWidget extends StatelessWidget {
  final String symbol;
  const StockWidget({super.key, required this.symbol});

  @override
  Widget build(BuildContext context) {
    return StreamBuilder<StockPrice>(
      stream: StockService.getPriceStream(symbol),
      builder: (context, snapshot) {
        // ตรวจสอบสถานะของ snapshot
        if (snapshot.connectionState == ConnectionState.waiting) {
          return const CircularProgressIndicator();
        }

        if (snapshot.hasError) {
          return Text('ข้อผิดพลาด: ${snapshot.error}');
        }

        if (!snapshot.hasData) {
          return const Text('ไม่มีข้อมูล');
        }

        final stock = snapshot.data!;
        final isPositive = stock.change >= 0;

        return Card(
          child: ListTile(
            title: Text(stock.symbol),
            subtitle: Text('ราคา: \$${stock.price.toStringAsFixed(2)}'),
            trailing: Text(
              '${isPositive ? '+' : ''}${stock.change.toStringAsFixed(2)}',
              style: TextStyle(
                color: isPositive ? Colors.green : Colors.red,
                fontWeight: FontWeight.bold,
              ),
            ),
          ),
        );
      },
    );
  }
}

// ConnectionState values
class ConnectionStateDemo extends StatelessWidget {
  final Stream<int> stream;
  const ConnectionStateDemo({super.key, required this.stream});

  @override
  Widget build(BuildContext context) {
    return StreamBuilder<int>(
      stream: stream,
      builder: (context, snapshot) {
        return switch (snapshot.connectionState) {
          ConnectionState.none => const Text('ไม่มี stream'),
          ConnectionState.waiting => const Text('กำลังรอข้อมูล...'),
          ConnectionState.active => Text('ค่าล่าสุด: ${snapshot.data}'),
          ConnectionState.done => Text('Stream สิ้นสุด: ${snapshot.data}'),
        };
      },
    );
  }
}

// Multiple StreamBuilders
class MultiStreamWidget extends StatelessWidget {
  const MultiStreamWidget({super.key});

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        StreamBuilder<int>(
          stream: _temperatureStream(),
          builder: (context, snap) => Text(
            'อุณหภูมิ: ${snap.data ?? '--'}°C',
          ),
        ),
        StreamBuilder<double>(
          stream: _humidityStream(),
          builder: (context, snap) => Text(
            'ความชื้น: ${snap.data?.toStringAsFixed(1) ?? '--'}%',
          ),
        ),
      ],
    );
  }

  Stream<int> _temperatureStream() =>
      Stream.periodic(Duration(seconds: 2), (i) => 25 + i % 5);

  Stream<double> _humidityStream() =>
      Stream.periodic(Duration(seconds: 3), (i) => 60.0 + i * 0.5);
}
```

## 13.5 Broadcast Streams

```dart
// Single-subscription vs Broadcast
void streamTypes() {
  // Single-subscription: ฟังได้คนเดียว
  var single = Stream.fromIterable([1, 2, 3]);
  single.listen(print); // OK
  // single.listen(print); // Error: Stream already has a subscriber

  // Broadcast: ฟังได้หลายคน
  var broadcast = Stream.fromIterable([1, 2, 3]).asBroadcastStream();
  broadcast.listen((v) => print('Listener 1: $v'));
  broadcast.listen((v) => print('Listener 2: $v'));
}

// Broadcast StreamController
class NotificationService {
  final _controller = StreamController<Notification>.broadcast();

  Stream<Notification> get notifications => _controller.stream;

  Stream<T> notificationsOfType<T extends Notification>() {
    return _controller.stream.whereType<T>();
  }

  void push(Notification notification) {
    _controller.add(notification);
  }

  void dispose() => _controller.close();
}

abstract class Notification {
  final String id;
  final String message;
  final DateTime time;
  Notification(this.id, this.message) : time = DateTime.now();
}

class PushNotification extends Notification {
  final String title;
  PushNotification(String id, this.title, String message)
      : super(id, message);
}

class SystemNotification extends Notification {
  final int severity;
  SystemNotification(String id, String message, {this.severity = 1})
      : super(id, message);
}

Future<void> testNotifications() async {
  final service = NotificationService();

  // หลาย listeners
  service.notifications.listen((n) =>
    print('[ALL] ${n.message}'));

  service.notificationsOfType<PushNotification>().listen((n) =>
    print('[PUSH] ${n.title}: ${n.message}'));

  service.notificationsOfType<SystemNotification>().listen((n) =>
    print('[SYSTEM] (severity ${n.severity}) ${n.message}'));

  // ส่ง notifications
  service.push(PushNotification('1', 'ข่าวสาร', 'มีโพสต์ใหม่'));
  service.push(SystemNotification('2', 'CPU สูง', severity: 3));
  service.push(PushNotification('3', 'ข้อความ', 'มีข้อความใหม่'));

  await Future.delayed(Duration(milliseconds: 100));
  service.dispose();
}

// State management with BroadcastStream
class AppState {
  int count = 0;
  String status = 'idle';

  AppState copyWith({int? count, String? status}) {
    return AppState()
      ..count = count ?? this.count
      ..status = status ?? this.status;
  }
}

class StateStore {
  AppState _state = AppState();
  final _controller = StreamController<AppState>.broadcast();

  Stream<AppState> get stream => _controller.stream;
  AppState get current => _state;

  void update(AppState Function(AppState) updater) {
    _state = updater(_state);
    _controller.add(_state);
  }

  void dispose() => _controller.close();
}
```

## 13.6 Workshop: Real-time Counter with Stream

สร้าง stopwatch แบบ real-time ด้วย Stream

```dart
import 'dart:async';

// ===== Models =====

class TimerState {
  final Duration elapsed;
  final bool isRunning;
  final List<Duration> laps;

  TimerState({
    required this.elapsed,
    required this.isRunning,
    required this.laps,
  });

  TimerState copyWith({
    Duration? elapsed,
    bool? isRunning,
    List<Duration>? laps,
  }) => TimerState(
    elapsed: elapsed ?? this.elapsed,
    isRunning: isRunning ?? this.isRunning,
    laps: laps ?? this.laps,
  );

  String get formattedTime {
    final ms = elapsed.inMilliseconds;
    final minutes = (ms ~/ 60000).toString().padLeft(2, '0');
    final seconds = ((ms ~/ 1000) % 60).toString().padLeft(2, '0');
    final centiseconds = ((ms ~/ 10) % 100).toString().padLeft(2, '0');
    return '$minutes:$seconds.$centiseconds';
  }
}

// ===== Stopwatch Stream =====

class StreamStopwatch {
  final _stateController = StreamController<TimerState>.broadcast();

  TimerState _state = TimerState(
    elapsed: Duration.zero,
    isRunning: false,
    laps: [],
  );

  Timer? _timer;
  DateTime? _startTime;
  Duration _accumulated = Duration.zero;

  Stream<TimerState> get state => _stateController.stream;
  TimerState get current => _state;

  void start() {
    if (_state.isRunning) return;

    _startTime = DateTime.now();
    _timer = Timer.periodic(Duration(milliseconds: 10), (_) {
      _updateElapsed();
    });

    _updateState(_state.copyWith(isRunning: true));
  }

  void pause() {
    if (!_state.isRunning) return;

    _accumulated = _state.elapsed;
    _timer?.cancel();
    _timer = null;

    _updateState(_state.copyWith(isRunning: false));
  }

  void reset() {
    _timer?.cancel();
    _timer = null;
    _accumulated = Duration.zero;
    _startTime = null;

    _updateState(TimerState(
      elapsed: Duration.zero,
      isRunning: false,
      laps: [],
    ));
  }

  void lap() {
    if (!_state.isRunning) return;

    var newLaps = [..._state.laps, _state.elapsed];
    _updateState(_state.copyWith(laps: newLaps));
    print('  บันทึก Lap ${newLaps.length}: ${_formatDuration(_state.elapsed)}');
  }

  void _updateElapsed() {
    if (_startTime == null) return;
    final elapsed = _accumulated + DateTime.now().difference(_startTime!);
    _updateState(_state.copyWith(elapsed: elapsed));
  }

  void _updateState(TimerState newState) {
    _state = newState;
    _stateController.add(_state);
  }

  String _formatDuration(Duration d) {
    final ms = d.inMilliseconds;
    final min = (ms ~/ 60000).toString().padLeft(2, '0');
    final sec = ((ms ~/ 1000) % 60).toString().padLeft(2, '0');
    final cs = ((ms ~/ 10) % 100).toString().padLeft(2, '0');
    return '$min:$sec.$cs';
  }

  void dispose() {
    _timer?.cancel();
    _stateController.close();
  }
}

// ===== Stream Processing Utilities =====

extension StreamUtils<T> on Stream<T> {
  // Throttle: emit ได้ไม่เกิน 1 ครั้งต่อ duration
  Stream<T> throttle(Duration duration) {
    DateTime? lastEmit;
    return where((value) {
      final now = DateTime.now();
      if (lastEmit == null || now.difference(lastEmit!) >= duration) {
        lastEmit = now;
        return true;
      }
      return false;
    });
  }

  // Buffer: รวม events เป็น batch
  Stream<List<T>> buffer(Duration duration) async* {
    var batch = <T>[];
    var timer = Timer(duration, () {});

    await for (final value in this) {
      batch.add(value);
      if (!timer.isActive) {
        yield batch;
        batch = [];
        timer = Timer(duration, () {});
      }
    }

    if (batch.isNotEmpty) yield batch;
  }

  // Scan: สะสมค่า (เหมือน reduce แต่ emit ทุกค่า)
  Stream<S> scan<S>(S initial, S Function(S, T) combine) async* {
    var accumulated = initial;
    yield accumulated;
    await for (final value in this) {
      accumulated = combine(accumulated, value);
      yield accumulated;
    }
  }
}

// ===== Real-time Analytics =====

class ClickAnalytics {
  final _clicks = StreamController<ClickEvent>.broadcast();

  Stream<ClickEvent> get clickStream => _clicks.stream;

  // Clicks per second
  Stream<int> get clicksPerSecond => clickStream
      .buffer(Duration(seconds: 1))
      .map((batch) => batch.length);

  // Running total
  Stream<int> get totalClicks => clickStream
      .scan<int>(0, (total, _) => total + 1)
      .skip(1); // skip initial 0

  // Recent clicks (last 5)
  Stream<List<ClickEvent>> get recentClicks => clickStream
      .scan<List<ClickEvent>>(
        [],
        (recent, click) => [...recent.takeLast(4), click],
      )
      .skip(1);

  void click(String buttonId) {
    _clicks.add(ClickEvent(buttonId));
  }

  void dispose() => _clicks.close();
}

class ClickEvent {
  final String buttonId;
  final DateTime time;
  ClickEvent(this.buttonId) : time = DateTime.now();
}

extension<T> on List<T> {
  List<T> takeLast(int n) => length <= n ? this : sublist(length - n);
}

// ===== Workshop Main =====

Future<void> main() async {
  print('=== Workshop: Real-time Counter with Stream ===\n');

  // Demo 1: StreamStopwatch
  print('1. Stopwatch Demo:');
  final stopwatch = StreamStopwatch();

  // Subscribe ด้วย throttle เพื่อแสดงแค่ทุก 500ms
  final displaySubscription = stopwatch.state
      .throttle(Duration(milliseconds: 500))
      .listen((state) {
        print('  ⏱ ${state.formattedTime} ${state.isRunning ? "(เดิน)" : "(หยุด)"}');
      });

  stopwatch.start();
  await Future.delayed(Duration(seconds: 2));

  stopwatch.lap();
  await Future.delayed(Duration(seconds: 1));

  stopwatch.lap();
  stopwatch.pause();

  await Future.delayed(Duration(milliseconds: 600));

  stopwatch.start();
  await Future.delayed(Duration(seconds: 1));

  stopwatch.pause();

  final finalState = stopwatch.current;
  print('\n  สรุป:');
  print('  เวลาทั้งหมด: ${finalState.formattedTime}');
  print('  Laps: ${finalState.laps.length}');

  displaySubscription.cancel();
  stopwatch.dispose();

  // Demo 2: Click Analytics
  print('\n2. Click Analytics Demo:');
  final analytics = ClickAnalytics();

  analytics.totalClicks.listen((total) =>
    print('  คลิกทั้งหมด: $total'));

  analytics.clicksPerSecond.listen((cps) =>
    print('  คลิก/วินาที: $cps'));

  // Simulate clicks
  for (var i = 0; i < 5; i++) {
    analytics.click('button_$i');
    await Future.delayed(Duration(milliseconds: 200));
  }

  await Future.delayed(Duration(seconds: 1));

  for (var i = 0; i < 3; i++) {
    analytics.click('ok_button');
    await Future.delayed(Duration(milliseconds: 100));
  }

  await Future.delayed(Duration(seconds: 2));
  analytics.dispose();

  // Demo 3: Stream transformation pipeline
  print('\n3. Stream Transformation Pipeline:');
  var dataStream = Stream.fromIterable(
    List.generate(20, (i) => i),
  );

  var processed = dataStream
      .where((n) => n % 2 == 0)        // เฉพาะเลขคู่
      .map((n) => n * n)               // ยกกำลังสอง
      .takeWhile((n) => n < 100)       // เอาแค่ < 100
      .scan<int>(0, (sum, n) => sum + n); // running sum

  print('  Running sum ของกำลังสองของเลขคู่:');
  await for (final sum in processed) {
    print('  $sum');
  }

  print('\n=== เสร็จสิ้น Workshop ===');
}
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Stream basics** - การสร้างและฟัง stream
2. **StreamController** - ควบคุม stream lifecycle
3. **Stream transformations** - map, where, expand, asyncMap
4. **StreamBuilder** - ใช้ stream ใน Flutter UI
5. **Broadcast streams** - stream ที่ฟังได้หลายคน
6. **Workshop** - สร้าง real-time stopwatch และ analytics

### เมื่อไหร่ควรใช้ Stream vs Future

| สถานการณ์ | Future | Stream |
|-----------|--------|--------|
| ดึงข้อมูลจาก API ครั้งเดียว | ✅ | - |
| Real-time updates | - | ✅ |
| ข้อมูลจาก WebSocket | - | ✅ |
| User input events | - | ✅ |
| File read (สั้น) | ✅ | - |
| File read (ยาว) | - | ✅ |

### Best Practices

- ปิด StreamController เมื่อไม่ใช้งาน
- ใช้ Broadcast stream เมื่อต้องการหลาย listeners
- ใช้ `await for` แทน `.listen()` เมื่อทำได้
- อย่าลืม cancel subscription เพื่อป้องกัน memory leak
- ใช้ Stream transformations แทนการเก็บ state เอง
