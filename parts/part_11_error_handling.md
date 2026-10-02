# Part 11: Error Handling ใน Dart

## บทนำ

การจัดการข้อผิดพลาด (Error Handling) เป็นทักษะสำคัญในการเขียนโปรแกรมที่ดี โปรแกรมในโลกจริงต้องพร้อมรับมือกับสถานการณ์ที่ไม่คาดคิด เช่น ไฟล์ไม่พบ, เครือข่ายล้มเหลว, หรือข้อมูลที่ไม่ถูกต้อง Dart มีระบบ error handling ที่ทรงพลังและยืดหยุ่น

## 11.1 try/catch/finally

### พื้นฐาน try/catch

```dart
void main() {
  try {
    // โค้ดที่อาจเกิดข้อผิดพลาด
    int result = 10 ~/ 0; // หารด้วยศูนย์
    print('Result: $result');
  } catch (e) {
    // จัดการข้อผิดพลาด
    print('เกิดข้อผิดพลาด: $e');
  }
}
```

### try/catch/finally

`finally` จะทำงานเสมอ ไม่ว่าจะเกิด exception หรือไม่

```dart
import 'dart:io';

void readFile(String path) {
  File? file;
  try {
    file = File(path);
    String content = file.readAsStringSync();
    print('เนื้อหาไฟล์: $content');
  } catch (e) {
    print('ไม่สามารถอ่านไฟล์ได้: $e');
  } finally {
    print('การทำงานเสร็จสิ้น (finally block)');
    // ล้างทรัพยากรที่ใช้งาน
  }
}

void main() {
  readFile('existing_file.txt');
  readFile('nonexistent_file.txt');
}
```

### การใช้ on keyword เพื่อจัดการ exception เฉพาะประเภท

```dart
void parseNumber(String input) {
  try {
    int number = int.parse(input);
    print('ตัวเลข: $number');
  } on FormatException catch (e) {
    print('รูปแบบข้อมูลไม่ถูกต้อง: ${e.message}');
  } on RangeError catch (e) {
    print('ค่าอยู่นอกช่วงที่กำหนด: $e');
  } catch (e) {
    print('เกิดข้อผิดพลาดที่ไม่ทราบประเภท: $e');
  }
}

void main() {
  parseNumber('42');      // ตัวเลข: 42
  parseNumber('abc');     // รูปแบบข้อมูลไม่ถูกต้อง
  parseNumber('99999999999999999999'); // อาจเกิด exception
}
```

### การดึง Stack Trace

```dart
void riskyOperation() {
  try {
    throw Exception('Something went wrong');
  } catch (e, stackTrace) {
    print('Error: $e');
    print('Stack trace:');
    print(stackTrace);
  }
}
```

## 11.2 ประเภทของ Exception และ Error

### ลำดับชั้นของ Exception

```
Object
  ├── Error (ข้อผิดพลาดร้ายแรงที่ไม่ควร catch)
  │   ├── AssertionError
  │   ├── TypeError
  │   ├── RangeError
  │   ├── ArgumentError
  │   ├── StateError
  │   ├── UnsupportedError
  │   ├── UnimplementedError
  │   ├── OutOfMemoryError
  │   ├── StackOverflowError
  │   └── NullThrownError
  └── Exception (ข้อผิดพลาดที่คาดว่าจะเกิดขึ้นได้)
      ├── FormatException
      ├── IOException
      │   ├── FileSystemException
      │   └── HttpException
      ├── TimeoutException
      └── ...
```

### ความแตกต่างระหว่าง Error และ Exception

```dart
void demonstrateErrors() {
  // Error: ไม่ควร catch เพราะแสดงว่าโปรแกรมมีบัก
  try {
    List<int> list = [1, 2, 3];
    print(list[10]); // RangeError
  } on RangeError catch (e) {
    print('RangeError: $e'); // ควรแก้ bug แทนที่จะ catch
  }

  // Exception: คาดว่าจะเกิดและสามารถจัดการได้
  try {
    int.parse('not a number'); // FormatException
  } on FormatException catch (e) {
    print('FormatException: ${e.message}');
  }
}
```

### Exception ที่พบบ่อย

```dart
void commonExceptions() {
  // 1. FormatException
  try {
    double.parse('invalid');
  } on FormatException catch (e) {
    print('FormatException: ${e.message}');
  }

  // 2. RangeError
  try {
    var list = [1, 2, 3];
    list[5]; // เข้าถึง index ที่ไม่มี
  } on RangeError catch (e) {
    print('RangeError: $e');
  }

  // 3. ArgumentError
  try {
    throw ArgumentError.notNull('username');
  } on ArgumentError catch (e) {
    print('ArgumentError: $e');
  }

  // 4. StateError
  try {
    var iterator = <int>[].iterator;
    iterator.current; // เรียกก่อน moveNext()
  } on StateError catch (e) {
    print('StateError: $e');
  }
}
```

## 11.3 Custom Exceptions

### สร้าง Custom Exception พื้นฐาน

```dart
// Exception สำหรับการตรวจสอบข้อมูล
class ValidationException implements Exception {
  final String message;
  final String field;

  ValidationException(this.field, this.message);

  @override
  String toString() => 'ValidationException: $field - $message';
}

// Exception สำหรับการยืนยันตัวตน
class AuthException implements Exception {
  final String message;
  final int statusCode;

  AuthException(this.message, {this.statusCode = 401});

  @override
  String toString() => 'AuthException($statusCode): $message';
}

// Exception สำหรับการเชื่อมต่อเครือข่าย
class NetworkException implements Exception {
  final String message;
  final String? url;
  final int? statusCode;

  NetworkException(
    this.message, {
    this.url,
    this.statusCode,
  });

  @override
  String toString() {
    var parts = ['NetworkException: $message'];
    if (url != null) parts.add('URL: $url');
    if (statusCode != null) parts.add('Status: $statusCode');
    return parts.join(', ');
  }
}

void testCustomExceptions() {
  try {
    throw ValidationException('email', 'รูปแบบ email ไม่ถูกต้อง');
  } on ValidationException catch (e) {
    print(e);
  }

  try {
    throw AuthException('Token หมดอายุ', statusCode: 401);
  } on AuthException catch (e) {
    print(e);
  }

  try {
    throw NetworkException(
      'ไม่สามารถเชื่อมต่อได้',
      url: 'https://api.example.com/users',
      statusCode: 503,
    );
  } on NetworkException catch (e) {
    print(e);
  }
}
```

### Custom Exception แบบลำดับชั้น

```dart
// Base exception
abstract class AppException implements Exception {
  final String message;
  final DateTime timestamp;
  final String? details;

  AppException(this.message, {this.details})
      : timestamp = DateTime.now();

  @override
  String toString() {
    var result = '${runtimeType}: $message';
    if (details != null) result += '\nDetails: $details';
    return result;
  }
}

// Database exceptions
class DatabaseException extends AppException {
  final String query;

  DatabaseException(String message, {required this.query, String? details})
      : super(message, details: details);
}

class RecordNotFoundException extends DatabaseException {
  final String recordId;

  RecordNotFoundException(String table, this.recordId)
      : super(
          'ไม่พบข้อมูลใน $table',
          query: 'SELECT * FROM $table WHERE id = $recordId',
        );
}

// Business logic exceptions
class BusinessException extends AppException {
  final String code;

  BusinessException(String message, {required this.code, String? details})
      : super(message, details: details);
}

class InsufficientFundsException extends BusinessException {
  final double required;
  final double available;

  InsufficientFundsException({
    required this.required,
    required this.available,
  }) : super(
          'ยอดเงินไม่เพียงพอ',
          code: 'INSUFFICIENT_FUNDS',
          details: 'ต้องการ: $required, มีอยู่: $available',
        );
}

void testHierarchicalExceptions() {
  try {
    throw RecordNotFoundException('users', '12345');
  } on RecordNotFoundException catch (e) {
    print('ไม่พบผู้ใช้: ${e.recordId}');
    print('Query: ${e.query}');
  } on DatabaseException catch (e) {
    print('Database error: $e');
  } on AppException catch (e) {
    print('App error: $e');
  }

  try {
    throw InsufficientFundsException(required: 1000.0, available: 250.0);
  } on InsufficientFundsException catch (e) {
    print('${e.message}: ${e.details}');
  }
}
```

## 11.4 throw keyword

### การ throw exception

```dart
// throw Exception object
void validateAge(int age) {
  if (age < 0) {
    throw ArgumentError('อายุต้องเป็นตัวเลขบวก: $age');
  }
  if (age > 150) {
    throw ArgumentError('อายุไม่สมเหตุสมผล: $age');
  }
}

// throw string (ไม่แนะนำ แต่ทำได้)
void oldStyleThrow() {
  throw 'Something went wrong'; // ไม่แนะนำ
}

// rethrow
void processData(String data) {
  try {
    validateAge(int.parse(data));
  } on FormatException {
    // จัดการ FormatException แล้ว rethrow ใหม่ในรูป custom exception
    throw ValidationException('age', 'ต้องเป็นตัวเลขเท่านั้น');
  }
}

// ตัวอย่างการ rethrow
void riskyWithLogging() {
  try {
    throw StateError('Invalid state');
  } catch (e) {
    print('บันทึก log: $e');
    rethrow; // โยน exception เดิมต่อไป
  }
}
```

### Guard clauses pattern

```dart
class UserService {
  void createUser(String? name, String? email, int? age) {
    // Guard clauses แทนที่ nested if
    if (name == null || name.isEmpty) {
      throw ValidationException('name', 'ชื่อต้องไม่ว่างเปล่า');
    }
    if (email == null || !email.contains('@')) {
      throw ValidationException('email', 'รูปแบบ email ไม่ถูกต้อง');
    }
    if (age == null || age < 18) {
      throw ValidationException('age', 'อายุต้องมากกว่า 18 ปี');
    }

    print('สร้างผู้ใช้: $name ($email), อายุ $age ปี');
  }
}

void testGuardClauses() {
  var service = UserService();

  try {
    service.createUser('', 'test@example.com', 25);
  } on ValidationException catch (e) {
    print('ข้อผิดพลาด: $e');
  }

  try {
    service.createUser('สมชาย', 'invalid-email', 25);
  } on ValidationException catch (e) {
    print('ข้อผิดพลาด: $e');
  }

  try {
    service.createUser('สมชาย', 'somchai@example.com', 15);
  } on ValidationException catch (e) {
    print('ข้อผิดพลาด: $e');
  }

  service.createUser('สมชาย', 'somchai@example.com', 25);
}
```

## 11.5 Stack Traces

### การใช้งาน StackTrace

```dart
import 'dart:core';

void functionC() {
  throw Exception('เกิดข้อผิดพลาดใน C');
}

void functionB() {
  functionC();
}

void functionA() {
  functionB();
}

void demonstrateStackTrace() {
  try {
    functionA();
  } catch (e, stackTrace) {
    print('Exception: $e');
    print('\nStack Trace:');
    print(stackTrace);

    // ดึง stack trace เป็น string
    String traceString = stackTrace.toString();
    List<String> lines = traceString.split('\n');

    print('\nFirst 5 lines of stack trace:');
    for (var i = 0; i < lines.length && i < 5; i++) {
      print('  ${lines[i]}');
    }
  }
}
```

### Custom Error Logger

```dart
class ErrorLogger {
  static final List<ErrorRecord> _logs = [];

  static void log(
    Object error,
    StackTrace stackTrace, {
    String? context,
    Map<String, dynamic>? extra,
  }) {
    var record = ErrorRecord(
      error: error,
      stackTrace: stackTrace,
      context: context,
      extra: extra,
      timestamp: DateTime.now(),
    );
    _logs.add(record);
    _printLog(record);
  }

  static void _printLog(ErrorRecord record) {
    print('═══════════════════════════════════');
    print('เวลา: ${record.timestamp}');
    if (record.context != null) print('Context: ${record.context}');
    print('Error: ${record.error}');
    if (record.extra != null) print('Extra: ${record.extra}');
    print('Stack Trace (5 lines):');
    var lines = record.stackTrace.toString().split('\n');
    for (var i = 0; i < lines.length && i < 5; i++) {
      if (lines[i].isNotEmpty) print('  ${lines[i]}');
    }
    print('═══════════════════════════════════');
  }

  static List<ErrorRecord> get logs => List.unmodifiable(_logs);
  static int get count => _logs.length;
}

class ErrorRecord {
  final Object error;
  final StackTrace stackTrace;
  final String? context;
  final Map<String, dynamic>? extra;
  final DateTime timestamp;

  ErrorRecord({
    required this.error,
    required this.stackTrace,
    this.context,
    this.extra,
    required this.timestamp,
  });
}

void testErrorLogger() {
  try {
    int.parse('not a number');
  } catch (e, st) {
    ErrorLogger.log(e, st, context: 'parseUserInput');
  }

  print('\nบันทึก error ทั้งหมด: ${ErrorLogger.count} รายการ');
}
```

## 11.6 Error Handling Patterns

### Result Pattern

Pattern นี้ใช้แทน exception เพื่อทำให้ code อ่านง่ายขึ้น

```dart
// Result type
sealed class Result<T> {
  const Result();
}

class Success<T> extends Result<T> {
  final T value;
  const Success(this.value);

  @override
  String toString() => 'Success($value)';
}

class Failure<T> extends Result<T> {
  final Exception exception;
  final StackTrace? stackTrace;

  const Failure(this.exception, [this.stackTrace]);

  @override
  String toString() => 'Failure($exception)';
}

// Extension methods สำหรับ Result
extension ResultExtension<T> on Result<T> {
  bool get isSuccess => this is Success<T>;
  bool get isFailure => this is Failure<T>;

  T? get valueOrNull => isSuccess ? (this as Success<T>).value : null;

  T getOrElse(T defaultValue) {
    return isSuccess ? (this as Success<T>).value : defaultValue;
  }

  T getOrThrow() {
    if (this is Success<T>) return (this as Success<T>).value;
    throw (this as Failure<T>).exception;
  }

  Result<R> map<R>(R Function(T) transform) {
    if (this is Success<T>) {
      try {
        return Success(transform((this as Success<T>).value));
      } catch (e) {
        return Failure(e is Exception ? e : Exception(e.toString()));
      }
    }
    return Failure((this as Failure<T>).exception);
  }
}

// ตัวอย่างการใช้งาน
Result<int> parseIntSafely(String input) {
  try {
    return Success(int.parse(input));
  } on FormatException catch (e) {
    return Failure(e);
  }
}

Result<double> calculateSquareRoot(int number) {
  if (number < 0) {
    return Failure(ArgumentError('ไม่สามารถหารากที่สองของจำนวนลบได้'));
  }
  import 'dart:math' as math;
  return Success(math.sqrt(number.toDouble()));
}

void testResultPattern() {
  // แบบปกติ
  var result = parseIntSafely('42');
  switch (result) {
    case Success(:final value):
      print('แปลงสำเร็จ: $value');
    case Failure(:final exception):
      print('แปลงไม่สำเร็จ: $exception');
  }

  // แบบ chain
  var result2 = parseIntSafely('16')
      .map((n) => n * 2);

  print(result2.valueOrNull); // 32

  // แบบใช้ getOrElse
  var value = parseIntSafely('abc').getOrElse(0);
  print('ค่าที่ได้: $value'); // 0
}
```

### Retry Pattern

```dart
import 'dart:async';
import 'dart:math' as math;

Future<T> retry<T>(
  Future<T> Function() operation, {
  int maxAttempts = 3,
  Duration initialDelay = const Duration(seconds: 1),
  double backoffFactor = 2.0,
  bool Function(Exception)? retryIf,
}) async {
  int attempt = 0;
  Duration delay = initialDelay;

  while (true) {
    try {
      attempt++;
      return await operation();
    } on Exception catch (e) {
      if (attempt >= maxAttempts) rethrow;
      if (retryIf != null && !retryIf(e)) rethrow;

      print('ลองใหม่ครั้งที่ $attempt/$maxAttempts หลังจาก ${delay.inSeconds}s');
      await Future.delayed(delay);
      delay = Duration(
        milliseconds: (delay.inMilliseconds * backoffFactor).round(),
      );
    }
  }
}

// Simulated network call
int _callCount = 0;

Future<String> unreliableNetworkCall() async {
  _callCount++;
  await Future.delayed(Duration(milliseconds: 100));

  if (_callCount < 3) {
    throw NetworkException('ไม่สามารถเชื่อมต่อได้ (ครั้งที่ $_callCount)');
  }
  return 'ข้อมูลจาก server';
}

Future<void> testRetryPattern() async {
  _callCount = 0;
  try {
    var result = await retry(
      unreliableNetworkCall,
      maxAttempts: 5,
      initialDelay: Duration(milliseconds: 100),
      retryIf: (e) => e is NetworkException,
    );
    print('สำเร็จ: $result');
  } on NetworkException catch (e) {
    print('ล้มเหลวหลังลองหลายครั้ง: $e');
  }
}
```

### Circuit Breaker Pattern

```dart
enum CircuitState { closed, open, halfOpen }

class CircuitBreaker {
  final int failureThreshold;
  final Duration timeout;

  CircuitState _state = CircuitState.closed;
  int _failureCount = 0;
  DateTime? _lastFailureTime;

  CircuitBreaker({
    this.failureThreshold = 5,
    this.timeout = const Duration(seconds: 60),
  });

  Future<T> execute<T>(Future<T> Function() operation) async {
    if (_state == CircuitState.open) {
      if (_shouldTryReset()) {
        _state = CircuitState.halfOpen;
      } else {
        throw StateError('Circuit breaker เปิดอยู่ - ไม่อนุญาตให้เรียก');
      }
    }

    try {
      T result = await operation();
      _onSuccess();
      return result;
    } catch (e) {
      _onFailure();
      rethrow;
    }
  }

  void _onSuccess() {
    _failureCount = 0;
    _state = CircuitState.closed;
  }

  void _onFailure() {
    _failureCount++;
    _lastFailureTime = DateTime.now();

    if (_failureCount >= failureThreshold) {
      _state = CircuitState.open;
      print('Circuit breaker เปิด - หยุดรับ request ชั่วคราว');
    }
  }

  bool _shouldTryReset() {
    if (_lastFailureTime == null) return false;
    return DateTime.now().difference(_lastFailureTime!) >= timeout;
  }

  CircuitState get state => _state;
  int get failureCount => _failureCount;
}
```

## 11.7 Workshop: API Error Handling

สร้างระบบ HTTP client ที่มีการจัดการ error อย่างสมบูรณ์

```dart
import 'dart:async';
import 'dart:convert';

// ===== Exception Types =====

abstract class ApiException implements Exception {
  final String message;
  final int? statusCode;
  final String? endpoint;

  ApiException(this.message, {this.statusCode, this.endpoint});

  @override
  String toString() {
    var parts = ['${runtimeType}: $message'];
    if (statusCode != null) parts.add('(HTTP $statusCode)');
    if (endpoint != null) parts.add('at $endpoint');
    return parts.join(' ');
  }
}

class NotFoundApiException extends ApiException {
  NotFoundApiException(String endpoint)
      : super('ไม่พบทรัพยากร', statusCode: 404, endpoint: endpoint);
}

class UnauthorizedException extends ApiException {
  UnauthorizedException({String? endpoint})
      : super('ไม่มีสิทธิ์เข้าถึง', statusCode: 401, endpoint: endpoint);
}

class ServerErrorException extends ApiException {
  ServerErrorException({int statusCode = 500, String? endpoint})
      : super('เซิร์ฟเวอร์ผิดพลาด', statusCode: statusCode, endpoint: endpoint);
}

class NetworkTimeoutException extends ApiException {
  NetworkTimeoutException({String? endpoint})
      : super('หมดเวลาการเชื่อมต่อ', endpoint: endpoint);
}

class ParseException extends ApiException {
  ParseException(String message) : super('ไม่สามารถอ่านข้อมูลได้: $message');
}

// ===== Models =====

class User {
  final int id;
  final String name;
  final String email;

  User({required this.id, required this.name, required this.email});

  factory User.fromJson(Map<String, dynamic> json) {
    try {
      return User(
        id: json['id'] as int,
        name: json['name'] as String,
        email: json['email'] as String,
      );
    } catch (e) {
      throw ParseException('ข้อมูล User ไม่ถูกต้อง: $e');
    }
  }

  @override
  String toString() => 'User(id: $id, name: $name, email: $email)';
}

// ===== Simulated HTTP Client =====

class MockHttpResponse {
  final int statusCode;
  final String body;

  MockHttpResponse(this.statusCode, this.body);
}

class ApiClient {
  final String baseUrl;
  final Duration timeout;
  final int maxRetries;
  final CircuitBreaker _circuitBreaker;

  static final Map<String, MockHttpResponse> _mockData = {
    '/users/1': MockHttpResponse(200, '{"id":1,"name":"สมชาย","email":"somchai@example.com"}'),
    '/users/2': MockHttpResponse(200, '{"id":2,"name":"สมหญิง","email":"somying@example.com"}'),
    '/users/999': MockHttpResponse(404, '{"error":"User not found"}'),
    '/admin/secret': MockHttpResponse(401, '{"error":"Unauthorized"}'),
    '/server-error': MockHttpResponse(500, '{"error":"Internal server error"}'),
    '/bad-json': MockHttpResponse(200, 'not valid json {{{'),
  };

  ApiClient({
    required this.baseUrl,
    this.timeout = const Duration(seconds: 30),
    this.maxRetries = 3,
  }) : _circuitBreaker = CircuitBreaker(
         failureThreshold: 3,
         timeout: Duration(seconds: 30),
       );

  Future<MockHttpResponse> _makeRequest(String endpoint) async {
    // Simulate network delay
    await Future.delayed(Duration(milliseconds: 50));

    var response = _mockData[endpoint];
    if (response == null) {
      throw NetworkTimeoutException(endpoint: '$baseUrl$endpoint');
    }
    return response;
  }

  Future<T> get<T>(
    String endpoint,
    T Function(Map<String, dynamic>) parser,
  ) async {
    return _circuitBreaker.execute(() async {
      MockHttpResponse response;

      try {
        response = await _makeRequest(endpoint)
            .timeout(timeout, onTimeout: () {
          throw NetworkTimeoutException(endpoint: '$baseUrl$endpoint');
        });
      } on NetworkTimeoutException {
        rethrow;
      } catch (e) {
        throw ApiException('เชื่อมต่อไม่ได้: $e',
            endpoint: '$baseUrl$endpoint');
      }

      return _handleResponse(response, endpoint, parser);
    });
  }

  T _handleResponse<T>(
    MockHttpResponse response,
    String endpoint,
    T Function(Map<String, dynamic>) parser,
  ) {
    switch (response.statusCode) {
      case 200:
        return _parseResponse(response.body, parser);
      case 401:
      case 403:
        throw UnauthorizedException(endpoint: '$baseUrl$endpoint');
      case 404:
        throw NotFoundApiException('$baseUrl$endpoint');
      case >= 500:
        throw ServerErrorException(
          statusCode: response.statusCode,
          endpoint: '$baseUrl$endpoint',
        );
      default:
        throw ApiException(
          'HTTP ${response.statusCode}',
          statusCode: response.statusCode,
          endpoint: '$baseUrl$endpoint',
        );
    }
  }

  T _parseResponse<T>(
    String body,
    T Function(Map<String, dynamic>) parser,
  ) {
    try {
      final json = jsonDecode(body) as Map<String, dynamic>;
      return parser(json);
    } on FormatException catch (e) {
      throw ParseException(e.message);
    } on ParseException {
      rethrow;
    } catch (e) {
      throw ParseException(e.toString());
    }
  }

  Future<User> getUser(int id) async {
    return get('/users/$id', User.fromJson);
  }
}

// ===== Service Layer =====

class UserService {
  final ApiClient _client;

  UserService(this._client);

  Future<Result<User>> fetchUser(int id) async {
    try {
      final user = await _client.getUser(id);
      return Success(user);
    } on NotFoundApiException catch (e) {
      return Failure(e);
    } on UnauthorizedException catch (e) {
      return Failure(e);
    } on NetworkTimeoutException catch (e) {
      return Failure(e);
    } on ApiException catch (e) {
      return Failure(e);
    }
  }

  Future<List<Result<User>>> fetchMultipleUsers(List<int> ids) async {
    final futures = ids.map((id) => fetchUser(id));
    return Future.wait(futures);
  }
}

// ===== Main Workshop =====

Future<void> main() async {
  print('=== Workshop: API Error Handling ===\n');

  final client = ApiClient(baseUrl: 'https://api.example.com');
  final service = UserService(client);

  // Test 1: Successful fetch
  print('1. ดึงข้อมูลผู้ใช้ที่มีอยู่:');
  var result1 = await service.fetchUser(1);
  switch (result1) {
    case Success(:final value):
      print('   สำเร็จ: $value');
    case Failure(:final exception):
      print('   ล้มเหลว: $exception');
  }

  // Test 2: Not found
  print('\n2. ดึงข้อมูลผู้ใช้ที่ไม่มีอยู่:');
  var result2 = await service.fetchUser(999);
  switch (result2) {
    case Success(:final value):
      print('   สำเร็จ: $value');
    case Failure(:final exception):
      print('   ล้มเหลว: $exception');
  }

  // Test 3: Multiple users
  print('\n3. ดึงข้อมูลหลายผู้ใช้:');
  var results = await service.fetchMultipleUsers([1, 2, 999]);
  for (var i = 0; i < results.length; i++) {
    switch (results[i]) {
      case Success(:final value):
        print('   ผู้ใช้ ${i + 1}: $value');
      case Failure(:final exception):
        print('   ผู้ใช้ ${i + 1}: ข้อผิดพลาด - $exception');
    }
  }

  print('\n=== เสร็จสิ้น Workshop ===');
}
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **try/catch/finally** - โครงสร้างพื้นฐานการจัดการ error
2. **Exception types** - ความแตกต่างระหว่าง Error และ Exception
3. **Custom exceptions** - การสร้าง exception เองเพื่อความชัดเจน
4. **throw/rethrow** - การโยน exception และส่งต่อ
5. **Stack traces** - การดึงและบันทึกข้อมูล stack trace
6. **Error patterns** - Result pattern, Retry pattern, Circuit Breaker

### Best Practices

- ใช้ Exception สำหรับสถานการณ์ที่คาดเดาได้ (เครือข่ายล้มเหลว, ข้อมูลผิดรูปแบบ)
- ใช้ Error สำหรับบัคในโปรแกรม (ไม่ควร catch)
- สร้าง custom exception ที่มีข้อมูลเพียงพอในการ debug
- ใช้ Result pattern เมื่อต้องการทำให้ code อ่านง่ายขึ้น
- บันทึก log ทุกครั้งที่เกิด exception ที่ไม่คาดคิด
- ไม่ควร catch exception แล้วไม่ทำอะไรเลย (silent failure)
