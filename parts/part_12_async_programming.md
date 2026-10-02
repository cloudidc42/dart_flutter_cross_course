# Part 12: Async Programming ใน Dart

## บทนำ

การเขียนโปรแกรมแบบ Asynchronous (async) ช่วยให้โปรแกรมสามารถทำงานหลายอย่างพร้อมกันโดยไม่บล็อกการทำงาน เช่น การโหลดข้อมูลจาก API ในขณะที่ UI ยังตอบสนองผู้ใช้ได้ Dart ใช้ `Future` และ `async/await` เป็นหัวใจของ async programming

## 12.1 Future และ async/await

### Future คืออะไร?

`Future<T>` แทนผลลัพธ์ที่จะเกิดขึ้นในอนาคต มี 3 สถานะ:
- **Uncompleted** - ยังไม่มีผลลัพธ์
- **Completed with value** - มีผลลัพธ์แล้ว
- **Completed with error** - เกิดข้อผิดพลาด

```dart
import 'dart:async';

// Future พื้นฐาน
Future<String> fetchData() {
  return Future.delayed(
    Duration(seconds: 2),
    () => 'ข้อมูลจาก server',
  );
}

// การใช้งานด้วย then/catchError
void usingThen() {
  print('เริ่มดึงข้อมูล...');
  fetchData()
      .then((data) => print('ได้รับข้อมูล: $data'))
      .catchError((error) => print('ข้อผิดพลาด: $error'))
      .whenComplete(() => print('เสร็จสิ้น'));
  print('โปรแกรมทำงานต่อได้ระหว่างรอข้อมูล');
}
```

### async/await

`async/await` ทำให้การเขียน async code อ่านง่ายเหมือน synchronous

```dart
// ไม่ใช้ async/await (ยากอ่าน)
Future<String> getUserNameOld(int id) {
  return fetchUser(id).then((user) {
    return fetchProfile(user.id).then((profile) {
      return '${user.name} - ${profile.bio}';
    });
  });
}

// ใช้ async/await (อ่านง่าย)
Future<String> getUserName(int id) async {
  final user = await fetchUser(id);
  final profile = await fetchProfile(user.id);
  return '${user.name} - ${profile.bio}';
}

// ฟังก์ชัน async ที่มี error handling
Future<void> loadAndDisplayUser(int id) async {
  try {
    print('กำลังโหลดข้อมูลผู้ใช้ ID: $id');
    final user = await fetchUser(id);
    final profile = await fetchProfile(user.id);
    print('ชื่อ: ${user.name}');
    print('Bio: ${profile.bio}');
  } on NotFoundException catch (e) {
    print('ไม่พบผู้ใช้: $e');
  } catch (e) {
    print('เกิดข้อผิดพลาด: $e');
  } finally {
    print('โหลดข้อมูลเสร็จสิ้น');
  }
}
```

### Simulated Data Layer

```dart
import 'dart:async';

// Models
class User {
  final int id;
  final String name;
  User(this.id, this.name);
  @override String toString() => 'User(id: $id, name: $name)';
}

class Profile {
  final int userId;
  final String bio;
  final int followers;
  Profile(this.userId, this.bio, this.followers);
  @override String toString() => 'Profile(userId: $userId, bio: $bio)';
}

class Post {
  final int id;
  final int userId;
  final String title;
  Post(this.id, this.userId, this.title);
  @override String toString() => 'Post(id: $id, title: $title)';
}

// Mock database
final _users = {
  1: User(1, 'สมชาย'),
  2: User(2, 'สมหญิง'),
  3: User(3, 'สมศักดิ์'),
};

final _profiles = {
  1: Profile(1, 'นักพัฒนา Flutter', 150),
  2: Profile(2, 'UI/UX Designer', 320),
  3: Profile(3, 'Backend Developer', 89),
};

final _posts = {
  1: [Post(1, 1, 'Hello Flutter'), Post(2, 1, 'Dart Tips')],
  2: [Post(3, 2, 'Design Systems')],
  3: [Post(4, 3, 'API Design'), Post(5, 3, 'REST vs GraphQL')],
};

// Async functions
Future<User> fetchUser(int id) async {
  await Future.delayed(Duration(milliseconds: 200));
  final user = _users[id];
  if (user == null) throw Exception('ไม่พบผู้ใช้ ID: $id');
  return user;
}

Future<Profile> fetchProfile(int userId) async {
  await Future.delayed(Duration(milliseconds: 150));
  final profile = _profiles[userId];
  if (profile == null) throw Exception('ไม่พบ Profile ของผู้ใช้ ID: $userId');
  return profile;
}

Future<List<Post>> fetchPosts(int userId) async {
  await Future.delayed(Duration(milliseconds: 250));
  return _posts[userId] ?? [];
}

// ตัวอย่างการใช้งาน
Future<void> demonstrateAsync() async {
  print('=== Async/Await Demo ===');

  // Sequential - รอทีละอัน
  final stopwatch = Stopwatch()..start();
  final user = await fetchUser(1);
  final profile = await fetchProfile(1);
  final posts = await fetchPosts(1);
  stopwatch.stop();

  print('Sequential: ${stopwatch.elapsedMilliseconds}ms');
  print('User: $user');
  print('Profile: $profile');
  print('Posts: $posts');
}
```

## 12.2 Future.wait และ Future.any

### Future.wait - รอทุก Future พร้อมกัน

```dart
Future<void> demonstrateFutureWait() async {
  print('=== Future.wait Demo ===');

  // Parallel fetch - ทำงานพร้อมกัน
  final stopwatch = Stopwatch()..start();

  final results = await Future.wait([
    fetchUser(1),
    fetchProfile(1),
    fetchPosts(1),
  ]);

  stopwatch.stop();
  print('Parallel: ${stopwatch.elapsedMilliseconds}ms');

  final user = results[0] as User;
  final profile = results[1] as Profile;
  final posts = results[2] as List<Post>;

  print('User: $user');
  print('Profile: $profile');
  print('Posts count: ${posts.length}');
}

// Future.wait กับ type-safe approach
Future<void> typeSafeFutureWait() async {
  // วิธีที่ดีกว่า: ใช้ record types (Dart 3+)
  final (user, profile, posts) = await (
    fetchUser(2),
    fetchProfile(2),
    fetchPosts(2),
  ).wait;

  print('User: $user');
  print('Profile: $profile');
  print('Posts: $posts');
}

// Future.wait กับ error handling
Future<void> futureWaitWithErrors() async {
  try {
    await Future.wait([
      fetchUser(1),
      fetchUser(999), // จะ fail
      fetchUser(2),
    ]);
  } catch (e) {
    print('หนึ่งใน Future ล้มเหลว: $e');
    // Future.wait จะ fail ทันทีที่มี Future หนึ่งตัว fail
  }

  // ถ้าต้องการดักจับทุก error
  final futures = [1, 2, 999].map((id) =>
    fetchUser(id).then<Object>((u) => u).catchError((e) => e)
  );
  final results = await Future.wait(futures);
  for (final result in results) {
    if (result is User) {
      print('สำเร็จ: $result');
    } else {
      print('ล้มเหลว: $result');
    }
  }
}
```

### Future.any - รับผลแรกที่เสร็จ

```dart
// ตัวอย่าง: Race condition - เอาผลแรกที่มาถึง
Future<String> fetchFromServer1() async {
  await Future.delayed(Duration(milliseconds: 300));
  return 'ข้อมูลจาก Server 1';
}

Future<String> fetchFromServer2() async {
  await Future.delayed(Duration(milliseconds: 500));
  return 'ข้อมูลจาก Server 2';
}

Future<String> fetchFromServer3() async {
  await Future.delayed(Duration(milliseconds: 100)); // เร็วสุด
  return 'ข้อมูลจาก Server 3';
}

Future<void> demonstrateFutureAny() async {
  print('=== Future.any Demo ===');

  final fastest = await Future.any([
    fetchFromServer1(),
    fetchFromServer2(),
    fetchFromServer3(), // จะชนะเพราะเร็วที่สุด
  ]);

  print('Server ที่เร็วที่สุดตอบกลับ: $fastest');
}

// Timeout pattern ด้วย Future.any
Future<T> withTimeout<T>(
  Future<T> future,
  Duration timeout,
  T defaultValue,
) async {
  return Future.any([
    future,
    Future.delayed(timeout).then((_) => defaultValue),
  ]);
}

Future<void> testTimeout() async {
  var result = await withTimeout(
    fetchFromServer1(),
    Duration(milliseconds: 200),
    'ค่า default (timeout)',
  );
  print('ผลลัพธ์: $result');
}
```

### Future.forEach และ Future.doWhile

```dart
Future<void> demonstrateFutureForEach() async {
  var ids = [1, 2, 3];

  print('=== Future.forEach (Sequential) ===');
  await Future.forEach(ids, (id) async {
    var user = await fetchUser(id);
    print('  โหลด: $user');
  });

  print('=== Parallel (Future.wait + map) ===');
  var users = await Future.wait(ids.map(fetchUser));
  users.forEach(print);
}

// Future.doWhile
Future<void> demonstrateDoWhile() async {
  var page = 1;
  var allData = <String>[];

  await Future.doWhile(() async {
    var data = await fetchPage(page);
    if (data.isEmpty) return false; // หยุด

    allData.addAll(data);
    page++;
    return page <= 3; // โหลดสูงสุด 3 หน้า
  });

  print('โหลดข้อมูลทั้งหมด ${allData.length} รายการ');
}

Future<List<String>> fetchPage(int page) async {
  await Future.delayed(Duration(milliseconds: 50));
  return List.generate(5, (i) => 'รายการหน้า$page-${i + 1}');
}
```

## 12.3 Completer

`Completer` ให้เราควบคุมการ complete ของ Future เองได้

```dart
import 'dart:async';

// พื้นฐาน Completer
Future<String> createFutureWithCompleter() {
  final completer = Completer<String>();

  // ทำงานบางอย่างแบบ async
  Future.delayed(Duration(seconds: 1), () {
    if (DateTime.now().second % 2 == 0) {
      completer.complete('สำเร็จ!');
    } else {
      completer.completeError(Exception('ล้มเหลว!'));
    }
  });

  return completer.future;
}

// ตัวอย่างการใช้งานจริง: Event-based API wrapper
class DialogController {
  Completer<bool>? _completer;

  Future<bool> showConfirmDialog(String message) {
    _completer = Completer<bool>();
    print('แสดง dialog: $message');
    return _completer!.future;
  }

  void confirm() {
    print('ผู้ใช้กด ยืนยัน');
    _completer?.complete(true);
    _completer = null;
  }

  void cancel() {
    print('ผู้ใช้กด ยกเลิก');
    _completer?.complete(false);
    _completer = null;
  }
}

Future<void> testDialogController() async {
  final controller = DialogController();

  // Simulate: แสดง dialog และรอผู้ใช้กด
  final futureResult = controller.showConfirmDialog('ต้องการลบข้อมูลหรือไม่?');

  // Simulate: ผู้ใช้กดยืนยันหลัง 1 วินาที
  Future.delayed(Duration(seconds: 1), () => controller.confirm());

  final result = await futureResult;
  print('ผลลัพธ์: ${result ? "ยืนยัน" : "ยกเลิก"}');
}

// Completer กับ timeout
Future<T> withCompleterTimeout<T>(
  void Function(Completer<T>) setup,
  Duration timeout,
) {
  final completer = Completer<T>();

  setup(completer);

  Future.delayed(timeout, () {
    if (!completer.isCompleted) {
      completer.completeError(
        TimeoutException('หมดเวลา', timeout),
      );
    }
  });

  return completer.future;
}

// Cache with Completer - ป้องกัน duplicate requests
class RequestCache {
  final Map<String, Completer<dynamic>> _pending = {};
  final Map<String, dynamic> _cache = {};

  Future<T> get<T>(String key, Future<T> Function() fetcher) async {
    // ถ้ามีใน cache แล้ว
    if (_cache.containsKey(key)) {
      return _cache[key] as T;
    }

    // ถ้ามี request ที่กำลังรออยู่
    if (_pending.containsKey(key)) {
      return (await _pending[key]!.future) as T;
    }

    // สร้าง request ใหม่
    final completer = Completer<T>();
    _pending[key] = completer;

    try {
      final result = await fetcher();
      _cache[key] = result;
      completer.complete(result);
      return result;
    } catch (e) {
      completer.completeError(e);
      rethrow;
    } finally {
      _pending.remove(key);
    }
  }
}

Future<void> testRequestCache() async {
  final cache = RequestCache();

  print('ส่ง 3 requests พร้อมกันสำหรับ user 1:');
  final futures = List.generate(3, (i) => cache.get(
    'user_1',
    () => fetchUser(1),
  ));

  final results = await Future.wait(futures);
  print('ผลลัพธ์ทั้ง 3: $results');
  print('(ทำ HTTP request จริงแค่ครั้งเดียว)');
}
```

## 12.4 Error Handling ใน Async

### async error handling patterns

```dart
// Pattern 1: try/catch ใน async
Future<User?> safeGetUser(int id) async {
  try {
    return await fetchUser(id);
  } on Exception catch (e) {
    print('ไม่สามารถโหลด user: $e');
    return null;
  }
}

// Pattern 2: catchError (ไม่แนะนำถ้าใช้ async/await แล้ว)
Future<User?> safeGetUserOld(int id) {
  return fetchUser(id).catchError((e) {
    print('Error: $e');
    return null;
  });
}

// Pattern 3: handleError
Future<User?> safeGetUserWithHandle(int id) async {
  return fetchUser(id).then(
    (user) => user,
    onError: (e) {
      print('Error: $e');
      return null;
    },
  );
}

// Async error ใน constructor - ใช้ factory
class DataService {
  final User _user;
  DataService._(this._user);

  static Future<DataService> create(int userId) async {
    final user = await fetchUser(userId); // async initialization
    return DataService._(user);
  }

  User get user => _user;
}

// Error propagation
Future<void> complexOperation() async {
  try {
    // ถ้า step ใดล้มเหลว จะ skip ขั้นตอนที่เหลือ
    final user = await fetchUser(1);
    final profile = await fetchProfile(user.id);
    final posts = await fetchPosts(user.id);

    print('ข้อมูลครบ: ${user.name}, ${profile.followers} followers, ${posts.length} posts');
  } catch (e) {
    // จัดการ error ในที่เดียว
    print('ล้มเหลว: $e');
  }
}

// Zone-based global error handling
void runWithErrorZone(void Function() body) {
  runZonedGuarded(
    body,
    (error, stackTrace) {
      print('Unhandled async error: $error');
      print(stackTrace);
    },
  );
}
```

### Async Validation

```dart
// Validate data asynchronously
class UserValidator {
  Future<ValidationResult> validate(Map<String, dynamic> data) async {
    final errors = <String, String>{};

    // ทำ validations พร้อมกัน
    await Future.wait([
      _validateUsername(data['username'] as String?, errors),
      _validateEmail(data['email'] as String?, errors),
    ]);

    return ValidationResult(
      isValid: errors.isEmpty,
      errors: errors,
    );
  }

  Future<void> _validateUsername(
    String? username,
    Map<String, String> errors,
  ) async {
    if (username == null || username.isEmpty) {
      errors['username'] = 'กรุณาใส่ชื่อผู้ใช้';
      return;
    }
    if (username.length < 3) {
      errors['username'] = 'ชื่อผู้ใช้ต้องมีอย่างน้อย 3 ตัวอักษร';
      return;
    }
    // Simulate DB check
    await Future.delayed(Duration(milliseconds: 100));
    if (username == 'admin') {
      errors['username'] = 'ชื่อผู้ใช้นี้ถูกใช้งานแล้ว';
    }
  }

  Future<void> _validateEmail(
    String? email,
    Map<String, String> errors,
  ) async {
    if (email == null || !email.contains('@')) {
      errors['email'] = 'รูปแบบ email ไม่ถูกต้อง';
      return;
    }
    // Simulate DNS check
    await Future.delayed(Duration(milliseconds: 50));
  }
}

class ValidationResult {
  final bool isValid;
  final Map<String, String> errors;
  ValidationResult({required this.isValid, required this.errors});
}
```

## 12.5 Isolates Introduction

Dart ทำงานบน single thread แต่สามารถสร้าง Isolate เพิ่มเพื่อทำงานหนักได้

```dart
import 'dart:isolate';

// การใช้ Isolate.run สำหรับ Dart 2.19+
Future<void> demonstrateIsolateRun() async {
  print('=== Isolate.run Demo ===');

  final stopwatch = Stopwatch()..start();

  // ทำงานหนักบน isolate อื่น ไม่บล็อก UI
  final result = await Isolate.run(() {
    return heavyComputation(50000000);
  });

  stopwatch.stop();
  print('ผลลัพธ์: $result');
  print('เวลา: ${stopwatch.elapsedMilliseconds}ms');
}

int heavyComputation(int iterations) {
  var sum = 0;
  for (var i = 0; i < iterations; i++) {
    sum += i;
  }
  return sum;
}

// Isolate แบบ low-level สำหรับงานที่ซับซ้อน
Future<void> demonstrateIsolateChannel() async {
  print('\n=== Isolate with SendPort/ReceivePort ===');

  final receivePort = ReceivePort();
  final isolate = await Isolate.spawn(
    isolateFunction,
    receivePort.sendPort,
  );

  await for (final message in receivePort) {
    if (message is String) {
      print('ได้รับจาก isolate: $message');
    } else if (message == null) {
      receivePort.close();
      isolate.kill();
      break;
    }
  }
}

void isolateFunction(SendPort sendPort) {
  sendPort.send('สวัสดีจาก Isolate!');
  sendPort.send('กำลังทำงาน...');

  // ทำงานหนัก
  var result = heavyComputation(1000000);
  sendPort.send('ผลลัพธ์: $result');
  sendPort.send(null); // บอกว่าเสร็จ
}

// Worker Isolate Pattern
class IsolateWorker {
  late Isolate _isolate;
  late SendPort _sendPort;
  late ReceivePort _receivePort;
  bool _isInitialized = false;

  Future<void> initialize() async {
    _receivePort = ReceivePort();
    _isolate = await Isolate.spawn(
      _workerMain,
      _receivePort.sendPort,
    );

    // รับ sendPort จาก worker
    _sendPort = await _receivePort.first;
    _isInitialized = true;
  }

  Future<T> compute<T>(Map<String, dynamic> task) async {
    if (!_isInitialized) await initialize();

    final responsePort = ReceivePort();
    _sendPort.send({
      'task': task,
      'responsePort': responsePort.sendPort,
    });

    final result = await responsePort.first;
    responsePort.close();

    if (result is Map && result.containsKey('error')) {
      throw Exception(result['error']);
    }
    return result as T;
  }

  void dispose() {
    _receivePort.close();
    _isolate.kill();
    _isInitialized = false;
  }

  static void _workerMain(SendPort mainSendPort) {
    final receivePort = ReceivePort();
    mainSendPort.send(receivePort.sendPort);

    receivePort.listen((message) {
      if (message is Map) {
        final task = message['task'] as Map<String, dynamic>;
        final responsePort = message['responsePort'] as SendPort;

        try {
          final result = _processTask(task);
          responsePort.send(result);
        } catch (e) {
          responsePort.send({'error': e.toString()});
        }
      }
    });
  }

  static dynamic _processTask(Map<String, dynamic> task) {
    switch (task['type']) {
      case 'sum':
        return heavyComputation(task['n'] as int);
      case 'sort':
        var list = List<int>.from(task['list'] as List);
        list.sort();
        return list;
      default:
        throw Exception('ไม่รู้จัก task type: ${task['type']}');
    }
  }
}
```

## 12.6 Workshop: Multi-fetch Data Loader

สร้าง data loader ที่โหลดข้อมูลหลายแหล่งพร้อมกัน พร้อม caching, retry, และ progress tracking

```dart
import 'dart:async';

// ===== Models =====

class UserProfile {
  final User user;
  final Profile profile;
  final List<Post> posts;
  final DateTime loadedAt;

  UserProfile({
    required this.user,
    required this.profile,
    required this.posts,
  }) : loadedAt = DateTime.now();

  @override
  String toString() =>
      'UserProfile(${user.name}, ${posts.length} posts, ${profile.followers} followers)';
}

// ===== Progress Tracking =====

enum LoadStatus { pending, loading, success, failure }

class LoadProgress {
  final String task;
  LoadStatus status;
  String? error;
  double progress;

  LoadProgress(this.task)
      : status = LoadStatus.pending,
        progress = 0.0;

  void start() {
    status = LoadStatus.loading;
    progress = 0.0;
  }

  void complete() {
    status = LoadStatus.success;
    progress = 1.0;
  }

  void fail(String errorMessage) {
    status = LoadStatus.failure;
    error = errorMessage;
  }

  @override
  String toString() {
    var statusStr = switch (status) {
      LoadStatus.pending => '⏳',
      LoadStatus.loading => '🔄',
      LoadStatus.success => '✅',
      LoadStatus.failure => '❌',
    };
    return '$statusStr $task${error != null ? " ($error)" : ""}';
  }
}

// ===== Cache Layer =====

class ProfileCache {
  final Map<int, UserProfile> _store = {};
  final Duration ttl;

  ProfileCache({this.ttl = const Duration(minutes: 5)});

  UserProfile? get(int userId) {
    final profile = _store[userId];
    if (profile == null) return null;

    // TTL check
    if (DateTime.now().difference(profile.loadedAt) > ttl) {
      _store.remove(userId);
      return null;
    }
    return profile;
  }

  void set(int userId, UserProfile profile) {
    _store[userId] = profile;
  }

  void invalidate(int userId) => _store.remove(userId);
  void clear() => _store.clear();
  int get size => _store.length;
}

// ===== Multi-fetch Data Loader =====

class DataLoader {
  final ProfileCache _cache;
  final int _maxConcurrent;
  final Map<int, LoadProgress> _progressMap = {};

  DataLoader({
    ProfileCache? cache,
    int maxConcurrent = 3,
  })  : _cache = cache ?? ProfileCache(),
        _maxConcurrent = maxConcurrent;

  // โหลด profile เดียว
  Future<UserProfile> loadProfile(
    int userId, {
    bool forceRefresh = false,
  }) async {
    // Check cache
    if (!forceRefresh) {
      final cached = _cache.get(userId);
      if (cached != null) {
        print('  📦 Cache hit สำหรับ user $userId');
        return cached;
      }
    }

    final progress = LoadProgress('User $userId');
    _progressMap[userId] = progress;
    progress.start();
    _printProgress();

    try {
      // โหลดข้อมูลแบบ parallel
      final (user, posts) = await (
        _fetchUserWithRetry(userId),
        fetchPosts(userId),
      ).wait;

      final profile = await fetchProfile(user.id);

      final userProfile = UserProfile(
        user: user,
        profile: profile,
        posts: posts,
      );

      _cache.set(userId, userProfile);
      progress.complete();
      _printProgress();

      return userProfile;
    } catch (e) {
      progress.fail(e.toString());
      _printProgress();
      rethrow;
    }
  }

  // โหลดหลาย profiles พร้อมกัน แต่จำกัดจำนวน concurrent
  Future<Map<int, Result<UserProfile>>> loadProfiles(
    List<int> userIds, {
    bool forceRefresh = false,
  }) async {
    final results = <int, Result<UserProfile>>{};
    final chunks = _chunk(userIds, _maxConcurrent);

    for (final chunk in chunks) {
      final chunkResults = await Future.wait(
        chunk.map((id) async {
          try {
            final profile = await loadProfile(id, forceRefresh: forceRefresh);
            return MapEntry(id, Success<UserProfile>(profile));
          } catch (e) {
            return MapEntry(
              id,
              Failure<UserProfile>(
                e is Exception ? e : Exception(e.toString()),
              ),
            );
          }
        }),
      );
      results.addEntries(chunkResults);
    }

    return results;
  }

  Future<User> _fetchUserWithRetry(int id, {int attempts = 0}) async {
    try {
      return await fetchUser(id);
    } catch (e) {
      if (attempts < 2) {
        await Future.delayed(Duration(milliseconds: 100 * (attempts + 1)));
        return _fetchUserWithRetry(id, attempts: attempts + 1);
      }
      rethrow;
    }
  }

  List<List<T>> _chunk<T>(List<T> list, int size) {
    final chunks = <List<T>>[];
    for (var i = 0; i < list.length; i += size) {
      chunks.add(list.sublist(i, (i + size).clamp(0, list.length)));
    }
    return chunks;
  }

  void _printProgress() {
    if (_progressMap.isEmpty) return;
    print('\nProgress:');
    _progressMap.values.forEach(print);
  }

  Map<String, dynamic> getStats() => {
    'cacheSize': _cache.size,
    'loadedTasks': _progressMap.length,
    'successful': _progressMap.values.where((p) => p.status == LoadStatus.success).length,
    'failed': _progressMap.values.where((p) => p.status == LoadStatus.failure).length,
  };
}

// ===== Result type (reused from Part 11) =====

sealed class Result<T> {}
class Success<T> extends Result<T> {
  final T value;
  Success(this.value);
  @override String toString() => 'Success($value)';
}
class Failure<T> extends Result<T> {
  final Exception exception;
  Failure(this.exception);
  @override String toString() => 'Failure($exception)';
}

extension on (Future, Future) {
  Future<(dynamic, dynamic)> get wait async {
    final results = await Future.wait([this.$1, this.$2]);
    return (results[0], results[1]);
  }
}

// ===== Workshop Main =====

Future<void> main() async {
  print('=== Workshop: Multi-fetch Data Loader ===\n');

  final loader = DataLoader(maxConcurrent: 3);

  // Test 1: โหลด profile เดียว
  print('1. โหลด user profile:');
  final profile1 = await loader.loadProfile(1);
  print('   $profile1\n');

  // Test 2: โหลดซ้ำจาก cache
  print('2. โหลดจาก cache:');
  final profile1Cached = await loader.loadProfile(1);
  print('   $profile1Cached\n');

  // Test 3: โหลดหลาย profiles
  print('3. โหลดหลาย profiles พร้อมกัน:');
  final results = await loader.loadProfiles([1, 2, 3]);

  print('\nผลลัพธ์:');
  results.forEach((id, result) {
    switch (result) {
      case Success(:final value):
        print('  User $id: $value');
      case Failure(:final exception):
        print('  User $id: ❌ $exception');
    }
  });

  // Test 4: Stats
  print('\nสถิติ:');
  loader.getStats().forEach((k, v) => print('  $k: $v'));

  // Test 5: Async composition
  print('\n4. Async composition - ดึงข้อมูลหลายขั้นตอน:');
  await composedDataLoad(loader);

  print('\n=== เสร็จสิ้น Workshop ===');
}

Future<void> composedDataLoad(DataLoader loader) async {
  // โหลด user 1 และ 2 พร้อมกัน แล้วหาว่าใครมี follower มากกว่า
  final [profile1, profile2] = await Future.wait([
    loader.loadProfile(1),
    loader.loadProfile(2),
  ]);

  final winner = profile1.profile.followers > profile2.profile.followers
      ? profile1
      : profile2;

  print('  ผู้ที่มี follower มากกว่า: ${winner.user.name}');
  print('  จำนวน: ${winner.profile.followers} followers');
  print('  Posts ล่าสุด:');
  winner.posts.take(2).forEach((p) => print('    - ${p.title}'));
}
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Future และ async/await** - การเขียน async code ที่อ่านง่าย
2. **Future.wait** - รอหลาย Future พร้อมกันเพื่อประหยัดเวลา
3. **Future.any** - รับผลแรกที่เสร็จ
4. **Completer** - ควบคุม Future lifecycle เอง
5. **Error handling ใน async** - try/catch ทำงานเหมือน sync code
6. **Isolates** - การทำงานหนักบน thread แยก

### Best Practices

- ใช้ `async/await` แทน `.then()` เพื่อให้อ่านง่าย
- รัน Future พร้อมกันด้วย `Future.wait` เมื่อ Future ไม่ขึ้นกัน
- ใช้ `Isolate.run()` สำหรับงานหนักที่ไม่ต้องการ IO
- อย่าลืม error handling ใน async function
- ใช้ timeout เพื่อป้องกัน Future ค้างไม่มีกำหนด
