# Part 90: Advanced Dart - Dart ขั้นสูง

## บทนำ

บทสุดท้ายนี้ครอบคลุม Dart features ขั้นสูงที่ช่วยให้เขียนโค้ดได้มีประสิทธิภาพสูงขึ้น ได้แก่ Isolates, FFI, code generation และ Dart Macros

## 1. Isolates สำหรับ Parallel Processing

### Isolate พื้นฐาน

```dart
// Dart รันบน single thread แต่ใช้ Isolate เพื่อ parallel processing
// แต่ละ Isolate มี memory ของตัวเอง (ไม่ share memory)

import 'dart:isolate';

// ===== วิธีที่ 1: compute() - ง่ายที่สุด =====
import 'package:flutter/foundation.dart';

Future<List<int>> findPrimes(int max) async {
  return await compute(_findPrimesInIsolate, max);
}

// ฟังก์ชันนี้รันใน Isolate แยก
List<int> _findPrimesInIsolate(int max) {
  final primes = <int>[];
  for (var i = 2; i <= max; i++) {
    if (_isPrime(i)) primes.add(i);
  }
  return primes;
}

bool _isPrime(int n) {
  if (n < 2) return false;
  for (var i = 2; i <= n ~/ 2; i++) {
    if (n % i == 0) return false;
  }
  return true;
}

// ===== วิธีที่ 2: Isolate.run() - Dart 2.19+ =====
Future<String> processLargeFile(String path) async {
  return await Isolate.run(() async {
    final file = File(path);
    final content = await file.readAsString();
    // Process content...
    return _processContent(content);
  });
}

// ===== วิธีที่ 3: Long-running Isolate ด้วย ReceivePort =====
class DataProcessingIsolate {
  late Isolate _isolate;
  late ReceivePort _receivePort;
  late SendPort _sendPort;
  final _resultController = StreamController<ProcessedData>.broadcast();

  Stream<ProcessedData> get results => _resultController.stream;

  Future<void> start() async {
    _receivePort = ReceivePort();
    _isolate = await Isolate.spawn(
      _isolateEntryPoint,
      _receivePort.sendPort,
    );

    // รอรับ SendPort จาก Isolate
    _sendPort = await _receivePort.first as SendPort;

    // ฟัง results
    _receivePort.listen((message) {
      if (message is ProcessedData) {
        _resultController.add(message);
      }
    });
  }

  void process(RawData data) {
    _sendPort.send(data);
  }

  void dispose() {
    _receivePort.close();
    _isolate.kill();
    _resultController.close();
  }
}

// ฟังก์ชันที่รันใน Isolate
void _isolateEntryPoint(SendPort mainSendPort) {
  final receivePort = ReceivePort();
  mainSendPort.send(receivePort.sendPort);  // ส่ง SendPort กลับ

  receivePort.listen((message) {
    if (message is RawData) {
      final result = _processData(message);  // heavy computation
      mainSendPort.send(result);
    }
  });
}
```

### Isolate Pool Pattern

```dart
// สร้าง pool ของ Isolates เพื่อ handle งานหลาย tasks พร้อมกัน
class IsolatePool {
  final int size;
  final List<_PoolWorker> _workers = [];
  int _nextWorker = 0;

  IsolatePool({required this.size});

  Future<void> initialize() async {
    for (var i = 0; i < size; i++) {
      final worker = _PoolWorker(id: i);
      await worker.start();
      _workers.add(worker);
    }
  }

  Future<T> run<T>(Future<T> Function() task) {
    final worker = _workers[_nextWorker % _workers.length];
    _nextWorker++;
    return worker.run(task);
  }

  void dispose() {
    for (final worker in _workers) {
      worker.dispose();
    }
  }
}
```

## 2. FFI (Foreign Function Interface)

### การเรียก C Code จาก Dart

```dart
// pubspec.yaml
// dependencies:
//   ffi: ^2.1.0

import 'dart:ffi';
import 'package:ffi/ffi.dart';

// ===== ตัวอย่าง: เรียก C math functions =====

// โหลด dynamic library
final DynamicLibrary _lib = Platform.isAndroid
    ? DynamicLibrary.open('libmy_library.so')
    : DynamicLibrary.process();

// นิยาม C function signatures
typedef NativeAddFunc = Int32 Function(Int32 a, Int32 b);
typedef DartAddFunc = int Function(int a, int b);

// Bind ฟังก์ชัน
final _add = _lib.lookupFunction<NativeAddFunc, DartAddFunc>('add');

// ใช้งาน
int addNumbers(int a, int b) => _add(a, b);

// ===== ตัวอย่าง: String Handling กับ FFI =====

typedef NativeProcessStringFunc = Pointer<Utf8> Function(Pointer<Utf8> input);
typedef DartProcessStringFunc = Pointer<Utf8> Function(Pointer<Utf8> input);

final _processString = _lib.lookupFunction<
    NativeProcessStringFunc,
    DartProcessStringFunc>('process_string');

String processString(String input) {
  final inputPtr = input.toNativeUtf8();
  try {
    final resultPtr = _processString(inputPtr);
    return resultPtr.toDartString();
  } finally {
    malloc.free(inputPtr);
  }
}

// ===== ตัวอย่าง: C Struct =====

// C struct:
// struct Point {
//   double x;
//   double y;
// };

final class CPoint extends Struct {
  @Double()
  external double x;

  @Double()
  external double y;
}

typedef NativeDistanceFunc = Double Function(
  Pointer<CPoint> p1,
  Pointer<CPoint> p2,
);
typedef DartDistanceFunc = double Function(
  Pointer<CPoint> p1,
  Pointer<CPoint> p2,
);

final _distance = _lib.lookupFunction<NativeDistanceFunc, DartDistanceFunc>(
  'calculate_distance',
);

double calculateDistance(double x1, double y1, double x2, double y2) {
  final p1 = calloc<CPoint>();
  final p2 = calloc<CPoint>();
  
  try {
    p1.ref.x = x1;
    p1.ref.y = y1;
    p2.ref.x = x2;
    p2.ref.y = y2;
    
    return _distance(p1, p2);
  } finally {
    calloc.free(p1);
    calloc.free(p2);
  }
}
```

### FFI ด้วย dart:ffi สำหรับ Performance-Critical Code

```dart
// ตัวอย่าง: เร่งความเร็วการประมวลผล image ด้วย C
class NativeImageProcessor {
  static final _lib = DynamicLibrary.open(_getLibraryPath());
  
  static String _getLibraryPath() {
    if (Platform.isAndroid) return 'libimage_processor.so';
    if (Platform.isIOS) return 'image_processor.framework/image_processor';
    if (Platform.isMacOS) return 'libimage_processor.dylib';
    throw UnsupportedError('Unsupported platform');
  }

  static final _grayscale = _lib.lookupFunction<
    Void Function(Pointer<Uint8> pixels, Int32 length),
    void Function(Pointer<Uint8> pixels, int length)
  >('convert_to_grayscale');

  static Uint8List convertToGrayscale(Uint8List imageData) {
    final pointer = malloc<Uint8>(imageData.length);
    try {
      final nativeArray = pointer.asTypedList(imageData.length);
      nativeArray.setAll(0, imageData);
      
      _grayscale(pointer, imageData.length);
      
      return Uint8List.fromList(nativeArray);
    } finally {
      malloc.free(pointer);
    }
  }
}
```

## 3. Code Generation ด้วย build_runner

### Freezed สำหรับ Immutable Models

```dart
// ต้องใช้ร่วมกับ build_runner
// flutter pub run build_runner build

// pubspec.yaml
// dev_dependencies:
//   build_runner: ^2.4.0
//   freezed: ^2.4.0
//   json_serializable: ^6.7.0
// dependencies:
//   freezed_annotation: ^2.4.0
//   json_annotation: ^4.8.0

import 'package:freezed_annotation/freezed_annotation.dart';

part 'user.freezed.dart';   // generated
part 'user.g.dart';         // generated (json)

@freezed
class User with _$User {
  const factory User({
    required String id,
    required String name,
    required String email,
    @Default(false) bool isPremium,
    String? avatarUrl,
  }) = _User;

  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
}

// ใช้งาน:
void exampleUsage() {
  const user = User(
    id: '1',
    name: 'John',
    email: 'john@example.com',
  );

  // copyWith
  final updated = user.copyWith(name: 'Jane', isPremium: true);

  // pattern matching
  final display = switch (user) {
    User(isPremium: true) => 'Premium: ${user.name}',
    User() => user.name,
  };

  // JSON
  final json = user.toJson();
  final fromJson = User.fromJson(json);
}
```

### Custom Code Generator

```dart
// lib/annotations/validated.dart
class Validated {
  final String? minLength;
  final int? min;
  final int? max;
  final String? pattern;
  const Validated({this.minLength, this.min, this.max, this.pattern});
}

// ใช้ annotation
class UserForm {
  @Validated(minLength: '3')
  final String username;
  
  @Validated(pattern: r'^[\w\.-]+@[\w\.-]+\.\w{2,}$')
  final String email;
  
  @Validated(min: 0, max: 150)
  final int age;
  
  const UserForm({
    required this.username,
    required this.email,
    required this.age,
  });
}

// Code generator จะสร้าง validate() method อัตโนมัติ
// lib/generators/validated_generator.dart
import 'package:build/build.dart';
import 'package:source_gen/source_gen.dart';
import 'package:analyzer/dart/element/element.dart';

class ValidatedGenerator extends GeneratorForAnnotation<Validated> {
  @override
  String generateForAnnotatedElement(
    Element element,
    ConstantReader annotation,
    BuildStep buildStep,
  ) {
    if (element is! ClassElement) return '';
    
    final className = element.name;
    final buffer = StringBuffer();
    
    buffer.writeln('extension ${className}Validation on $className {');
    buffer.writeln('  Map<String, String?> validate() {');
    buffer.writeln('    final errors = <String, String?>{};');
    
    for (final field in element.fields) {
      final validatedAnnotation = TypeChecker
          .fromRuntime(Validated)
          .firstAnnotationOf(field);
      
      if (validatedAnnotation == null) continue;
      
      final reader = ConstantReader(validatedAnnotation);
      final fieldName = field.name;
      
      // Generate validation logic
      final minLength = reader.peek('minLength')?.intValue;
      if (minLength != null) {
        buffer.writeln(
          "    if ($fieldName.length < $minLength) {"
          "      errors['$fieldName'] = '$fieldName must be at least $minLength characters';"
          "    }",
        );
      }
    }
    
    buffer.writeln('    return errors;');
    buffer.writeln('  }');
    buffer.writeln('}');
    
    return buffer.toString();
  }
}
```

## 4. Dart Macros (Dart 3.x)

### Macro พื้นฐาน

```dart
// Dart Macros ยังอยู่ใน experimental (Dart 3.x)
// ต้องเปิด experiment flag

// analysis_options.yaml:
// analyzer:
//   enable-experiment:
//     - macros

import 'dart:core';
import 'package:macros/macros.dart';

// ===== ตัวอย่าง: AutoToString Macro =====
macro class AutoToString implements ClassDeclarationsMacro {
  const AutoToString();

  @override
  Future<void> buildDeclarationsForClass(
    ClassDeclaration clazz,
    MemberDeclarationBuilder builder,
  ) async {
    final fields = await builder.fieldsOf(clazz);
    
    final fieldParts = fields
        .map((f) => '${f.identifier.name}: \$${f.identifier.name}')
        .join(', ');
    
    builder.declareInType(DeclarationCode.fromString(
      'String toString() => \'${clazz.identifier.name}($fieldParts)\';'
    ));
  }
}

// ใช้งาน macro:
@AutoToString()
class Point {
  final double x;
  final double y;
  const Point(this.x, this.y);
}

// Macro จะ generate toString() อัตโนมัติ:
// String toString() => 'Point(x: $x, y: $y)';

// ===== ตัวอย่าง: Observable Macro =====
macro class Observable implements ClassDeclarationsMacro {
  const Observable();

  @override
  Future<void> buildDeclarationsForClass(
    ClassDeclaration clazz,
    MemberDeclarationBuilder builder,
  ) async {
    final fields = await builder.fieldsOf(clazz);
    
    // สร้าง StreamController
    builder.declareInType(DeclarationCode.fromString(
      'final _controller = StreamController<void>.broadcast();'
      'Stream<void> get changes => _controller.stream;'
    ));
    
    // สร้าง setter ที่ notify สำหรับแต่ละ field
    for (final field in fields) {
      final name = field.identifier.name;
      builder.declareInType(DeclarationCode.fromString(
        'set $name(${field.type} value) {'
        '  _$name = value;'
        '  _controller.add(null);'
        '}'
      ));
    }
  }
}
```

## 5. Workshop: Parallel Data Processing

```dart
// workshop: parallel_processing/
// ประมวลผล large dataset ด้วย Isolates

import 'dart:isolate';
import 'dart:math';

class ParallelProcessor {
  final int workerCount;
  
  ParallelProcessor({this.workerCount = 4});

  Future<List<T>> processAll<T, R>(
    List<R> items,
    T Function(R item) processor,
  ) async {
    if (items.isEmpty) return [];
    
    // แบ่ง items เป็น chunks
    final chunks = _splitIntoChunks(items, workerCount);
    
    // ประมวลผลแต่ละ chunk ใน Isolate แยก
    final futures = chunks.map(
      (chunk) => Isolate.run(() => chunk.map(processor).toList()),
    );
    
    // รวมผลลัพธ์
    final results = await Future.wait(futures);
    return results.expand((r) => r).toList();
  }

  List<List<T>> _splitIntoChunks<T>(List<T> items, int count) {
    final chunkSize = (items.length / count).ceil();
    final chunks = <List<T>>[];
    
    for (var i = 0; i < items.length; i += chunkSize) {
      chunks.add(
        items.sublist(i, min(i + chunkSize, items.length)),
      );
    }
    
    return chunks;
  }
}

// ตัวอย่างการใช้งาน
class ImageBatchProcessor {
  final _processor = ParallelProcessor(workerCount: 4);

  Future<List<ProcessedImage>> processBatch(
    List<RawImage> images,
  ) async {
    return await _processor.processAll(
      images,
      _processImage,
    );
  }

  static ProcessedImage _processImage(RawImage raw) {
    // Heavy processing: resize, filter, compress
    return ProcessedImage(
      data: _applyFilters(raw.data),
      width: raw.width,
      height: raw.height,
    );
  }

  static Uint8List _applyFilters(Uint8List data) {
    // Simulate heavy processing
    return data;
  }
}

// ===== Benchmark: Sequential vs Parallel =====
Future<void> runBenchmark() async {
  final items = List.generate(10000, (i) => i);
  
  // Sequential
  final stopwatch = Stopwatch()..start();
  final sequential = items.map(_heavyComputation).toList();
  print('Sequential: ${stopwatch.elapsedMilliseconds}ms');
  
  // Parallel
  stopwatch.reset();
  final processor = ParallelProcessor(workerCount: 4);
  final parallel = await processor.processAll(items, _heavyComputation);
  print('Parallel: ${stopwatch.elapsedMilliseconds}ms');
  
  assert(sequential.length == parallel.length);
}

int _heavyComputation(int n) {
  // Simulate CPU-intensive work
  var result = 0;
  for (var i = 0; i < 1000; i++) {
    result += (n * i) % 100;
  }
  return result;
}
```

## 6. Advanced Dart Patterns

### Extension Methods

```dart
// Extensions ช่วยเพิ่ม methods ให้ existing classes
extension StringX on String {
  bool get isValidEmail =>
      RegExp(r'^[\w\.-]+@[\w\.-]+\.\w{2,}$').hasMatch(this);
  
  String get toTitleCase => split(' ')
      .map((word) => word.isEmpty
          ? word
          : '${word[0].toUpperCase()}${word.substring(1).toLowerCase()}')
      .join(' ');
  
  String truncate(int maxLength, {String suffix = '...'}) {
    if (length <= maxLength) return this;
    return '${substring(0, maxLength - suffix.length)}$suffix';
  }
}

extension ContextX on BuildContext {
  ThemeData get theme => Theme.of(this);
  TextTheme get textTheme => Theme.of(this).textTheme;
  ColorScheme get colorScheme => Theme.of(this).colorScheme;
  MediaQueryData get mediaQuery => MediaQuery.of(this);
  double get screenWidth => mediaQuery.size.width;
  double get screenHeight => mediaQuery.size.height;
  bool get isDark => theme.brightness == Brightness.dark;
  
  void showSnackBar(String message) {
    ScaffoldMessenger.of(this).showSnackBar(
      SnackBar(content: Text(message)),
    );
  }
}

extension ListX<T> on List<T> {
  List<T> unique() {
    final seen = <T>{};
    return where(seen.add).toList();
  }
  
  List<List<T>> chunked(int size) {
    final chunks = <List<T>>[];
    for (var i = 0; i < length; i += size) {
      chunks.add(sublist(i, min(i + size, length)));
    }
    return chunks;
  }
  
  T? get firstOrNull => isEmpty ? null : first;
  T? get lastOrNull => isEmpty ? null : last;
}
```

### Sealed Classes (Dart 3)

```dart
// Sealed classes สำหรับ exhaustive pattern matching
sealed class Shape {
  double get area;
  double get perimeter;
}

class Circle extends Shape {
  final double radius;
  Circle(this.radius);
  
  @override
  double get area => pi * radius * radius;
  
  @override
  double get perimeter => 2 * pi * radius;
}

class Rectangle extends Shape {
  final double width;
  final double height;
  Rectangle(this.width, this.height);
  
  @override
  double get area => width * height;
  
  @override
  double get perimeter => 2 * (width + height);
}

class Triangle extends Shape {
  final double a, b, c;
  Triangle(this.a, this.b, this.c);
  
  @override
  double get area {
    final s = (a + b + c) / 2;
    return sqrt(s * (s - a) * (s - b) * (s - c));
  }
  
  @override
  double get perimeter => a + b + c;
}

// Pattern matching ที่ exhaustive
String describeShape(Shape shape) => switch (shape) {
  Circle(radius: final r) => 'Circle with radius $r',
  Rectangle(width: final w, height: final h) => 'Rectangle ${w}x$h',
  Triangle(a: final a, b: final b, c: final c) => 'Triangle $a-$b-$c',
};
// Compiler บอกถ้าลืม case!
```

## สรุปหลักสูตร Flutter ทั้งหมด

```
Parts 1-20:   Dart & Flutter Fundamentals
Parts 21-40:  UI, Animations, State Management  
Parts 41-60:  Firebase, APIs, Local Storage
Parts 61-70:  Platform Features, Testing
Parts 71-80:  Advanced Flutter Techniques
Parts 81-90:  Architecture, Projects, Career

เส้นทางการเรียนรู้ต่อไป:
1. Flutter Web & Desktop (go beyond mobile)
2. Flutter Engine & Framework contributions
3. Dart compiler & VM internals
4. Custom Flutter embedders
5. Flutter for embedded systems
```

## แบบทดสอบ Final

1. สร้าง Isolate pool ที่มี work queue และ priority
2. สร้าง FFI wrapper สำหรับ image processing library
3. สร้าง custom code generator ด้วย source_gen
4. เขียน Dart macro ที่ generate JSON serialization
5. Benchmark parallel vs sequential processing บน real data
