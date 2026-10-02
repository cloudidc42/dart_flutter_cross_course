# Part 15: Extensions และ Callable Classes

## บทนำ

Extensions ใน Dart ช่วยให้เราเพิ่มฟังก์ชันการทำงานให้กับ class ที่มีอยู่แล้ว โดยไม่ต้องแก้ไข source code ของ class นั้น เหมาะมากสำหรับการ extend built-in types หรือ third-party libraries

## 15.1 Extension Methods

### สร้าง Extension พื้นฐาน

```dart
// สร้าง extension บน String
extension StringExtensions on String {
  // Method
  bool get isEmail => RegExp(
    r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$',
  ).hasMatch(this);

  // Method with parameters
  String capitalize() {
    if (isEmpty) return this;
    return '${this[0].toUpperCase()}${substring(1).toLowerCase()}';
  }

  String titleCase() {
    return split(' ')
        .map((word) => word.capitalize())
        .join(' ');
  }

  String truncate(int maxLength, {String ellipsis = '...'}) {
    if (length <= maxLength) return this;
    return '${substring(0, maxLength - ellipsis.length)}$ellipsis';
  }

  // Properties
  bool get isBlank => trim().isEmpty;
  bool get isNotBlank => !isBlank;
  bool get isNumeric => double.tryParse(this) != null;

  int get wordCount => trim().isEmpty ? 0 : trim().split(RegExp(r'\s+')).length;
}

void testStringExtensions() {
  print('=== String Extensions ===');

  var email = 'user@example.com';
  print(email.isEmail);  // true

  var name = 'hello world';
  print(name.capitalize());  // Hello world
  print(name.titleCase());   // Hello World

  var longText = 'นี่คือข้อความที่ยาวมาก ควรถูกตัดให้สั้นลง';
  print(longText.truncate(20));  // นี่คือข้อความที่ยาว...

  print('  hello  '.isBlank);  // false
  print('     '.isBlank);      // true

  print('Hello World'.wordCount);  // 2
}
```

### Extension บน int

```dart
extension IntExtensions on int {
  Duration get seconds => Duration(seconds: this);
  Duration get minutes => Duration(minutes: this);
  Duration get hours => Duration(hours: this);
  Duration get days => Duration(days: this);

  bool get isEven => this % 2 == 0;
  bool get isOdd => this % 2 != 0;
  bool get isPrime {
    if (this < 2) return false;
    for (var i = 2; i <= this ~/ 2; i++) {
      if (this % i == 0) return false;
    }
    return true;
  }

  int clamp(int min, int max) {
    if (this < min) return min;
    if (this > max) return max;
    return this;
  }

  String padLeft(int width, [String padding = '0']) {
    return toString().padLeft(width, padding);
  }

  Iterable<int> to(int end) sync* {
    final step = end >= this ? 1 : -1;
    for (var i = this; step > 0 ? i <= end : i >= end; i += step) {
      yield i;
    }
  }
}

void testIntExtensions() {
  print('\n=== Int Extensions ===');

  // Duration
  print(5.seconds);  // 0:00:05.000000
  print(2.hours);    // 2:00:00.000000

  // Future.delayed ใช้งานง่าย
  // await Future.delayed(3.seconds);

  // Math
  print(7.isPrime);  // true
  print(10.isPrime); // false
  print(105.clamp(0, 100)); // 100

  // Range
  print(1.to(5).toList()); // [1, 2, 3, 4, 5]
  print(5.to(1).toList()); // [5, 4, 3, 2, 1]
}
```

### Extension บน double

```dart
extension DoubleExtensions on double {
  double roundTo(int decimalPlaces) {
    final factor = 10.0.pow(decimalPlaces.toDouble());
    return (this * factor).round() / factor;
  }

  String toThaiCurrency({String symbol = '฿'}) {
    final formatted = toStringAsFixed(2)
        .replaceAllMapped(
          RegExp(r'(\d)(?=(\d{3})+(?!\d))'),
          (m) => '${m[1]},',
        );
    return '$symbol$formatted';
  }

  bool get isWholeNumber => this == truncate();

  double get celsius => (this - 32) * 5 / 9;
  double get fahrenheit => this * 9 / 5 + 32;
}

extension DoublePow on double {
  double pow(double exponent) {
    import 'dart:math' as math;
    return math.pow(this, exponent).toDouble();
  }
}

void testDoubleExtensions() {
  print('\n=== Double Extensions ===');
  print(3.14159.roundTo(2)); // 3.14
  print(1234567.89.toThaiCurrency()); // ฿1,234,567.89
  print(100.0.isWholeNumber); // true
  print(98.6.celsius); // 37.0 (Fahrenheit to Celsius)
}
```

## 15.2 Extension บน Built-in Types

### Extension บน List

```dart
extension ListExtensions<T> on List<T> {
  // ดึงค่าแบบปลอดภัย
  T? get firstOrNull => isEmpty ? null : first;
  T? get lastOrNull => isEmpty ? null : last;
  T? safeGet(int index) => (index >= 0 && index < length) ? this[index] : null;

  // แบ่ง list เป็น chunks
  List<List<T>> chunk(int size) {
    final chunks = <List<T>>[];
    for (var i = 0; i < length; i += size) {
      chunks.add(sublist(i, (i + size).clamp(0, length)));
    }
    return chunks;
  }

  // Random shuffle (in-place)
  List<T> shuffled() {
    import 'dart:math';
    final copy = [...this];
    final random = Random();
    for (var i = copy.length - 1; i > 0; i--) {
      final j = random.nextInt(i + 1);
      final temp = copy[i];
      copy[i] = copy[j];
      copy[j] = temp;
    }
    return copy;
  }

  // Group by key
  Map<K, List<T>> groupBy<K>(K Function(T) keySelector) {
    final map = <K, List<T>>{};
    for (final item in this) {
      final key = keySelector(item);
      map.putIfAbsent(key, () => []).add(item);
    }
    return map;
  }

  // Remove duplicates โดยใช้ key
  List<T> uniqueBy<K>(K Function(T) keySelector) {
    final seen = <K>{};
    return where((item) => seen.add(keySelector(item))).toList();
  }

  // Interleave กับ another list
  List<T> interleave(List<T> other) {
    final result = <T>[];
    final maxLen = length > other.length ? length : other.length;
    for (var i = 0; i < maxLen; i++) {
      if (i < length) result.add(this[i]);
      if (i < other.length) result.add(other[i]);
    }
    return result;
  }

  // Zip กับ another list
  List<(T, S)> zip<S>(List<S> other) {
    final minLen = length < other.length ? length : other.length;
    return List.generate(minLen, (i) => (this[i], other[i]));
  }
}

extension NumericList on List<num> {
  num get sum => fold(0, (a, b) => a + b);
  double get average => isEmpty ? 0.0 : sum / length;
  num get max => reduce((a, b) => a > b ? a : b);
  num get min => reduce((a, b) => a < b ? a : b);
  double get standardDeviation {
    if (isEmpty) return 0.0;
    final avg = average;
    final variance = map((x) => (x - avg) * (x - avg)).average;
    return import('dart:math').sqrt(variance);
  }
}

void testListExtensions() {
  print('\n=== List Extensions ===');

  var numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

  print('Chunks of 3: ${numbers.chunk(3)}');
  // [[1,2,3], [4,5,6], [7,8,9], [10]]

  print('firstOrNull: ${numbers.firstOrNull}');
  print('safeGet(15): ${numbers.safeGet(15)}'); // null

  var words = ['apple', 'banana', 'avocado', 'blueberry', 'cherry'];
  var grouped = words.groupBy((w) => w[0]);
  print('Grouped by first letter: $grouped');
  // {a: [apple, avocado], b: [banana, blueberry], c: [cherry]}

  var dups = [1, 2, 2, 3, 3, 3, 4];
  // uniqueBy ด้วย identity
  print('Unique: ${dups.uniqueBy((n) => n)}'); // [1, 2, 3, 4]

  var list1 = [1, 3, 5];
  var list2 = [2, 4, 6];
  print('Interleaved: ${list1.interleave(list2)}'); // [1, 2, 3, 4, 5, 6]
  print('Zipped: ${list1.zip(list2)}'); // [(1,2), (3,4), (5,6)]

  List<num> scores = [85, 92, 78, 95, 88];
  print('Sum: ${scores.sum}, Average: ${scores.average}');
  print('Max: ${scores.max}, Min: ${scores.min}');
}
```

### Extension บน Map

```dart
extension MapExtensions<K, V> on Map<K, V> {
  // Merge กับ map อื่น
  Map<K, V> merge(Map<K, V> other) => {...this, ...other};

  // Filter keys หรือ values
  Map<K, V> whereKey(bool Function(K) test) =>
      Map.fromEntries(entries.where((e) => test(e.key)));

  Map<K, V> whereValue(bool Function(V) test) =>
      Map.fromEntries(entries.where((e) => test(e.value)));

  // Transform values
  Map<K, R> mapValues<R>(R Function(V) transform) =>
      map((k, v) => MapEntry(k, transform(v)));

  // Invert (swap keys and values)
  Map<V, K> get inverted => map((k, v) => MapEntry(v, k));

  // Get or compute
  V getOrCompute(K key, V Function() compute) {
    if (containsKey(key)) return this[key] as V;
    final value = compute();
    this[key] = value;
    return value;
  }
}

void testMapExtensions() {
  print('\n=== Map Extensions ===');

  var scores = {'อลิส': 90, 'บ็อบ': 75, 'ชาร์ลี': 88, 'เดฟ': 60};

  // กรองเฉพาะคะแนน >= 80
  var passing = scores.whereValue((v) => v >= 80);
  print('ผ่าน: $passing');

  // แปลงคะแนนเป็น grade
  var grades = scores.mapValues((score) {
    if (score >= 90) return 'A';
    if (score >= 80) return 'B';
    if (score >= 70) return 'C';
    return 'F';
  });
  print('Grades: $grades');

  // Invert
  var gradeToName = grades.inverted;
  print('Inverted: $gradeToName');
}
```

## 15.3 Callable Classes

Callable class คือ class ที่สามารถเรียกได้เหมือน function โดยใช้ `call()` method

```dart
// Callable class พื้นฐาน
class Multiplier {
  final int factor;
  Multiplier(this.factor);

  // call() ทำให้ instance เรียกได้เหมือน function
  int call(int value) => value * factor;
}

void testCallable() {
  var triple = Multiplier(3);
  print(triple(5));  // 15
  print(triple(10)); // 30

  // ใช้เหมือน function
  var numbers = [1, 2, 3, 4, 5];
  var tripled = numbers.map(triple).toList();
  print(tripled); // [3, 6, 9, 12, 15]
}

// Validator callable
class Validator<T> {
  final bool Function(T) _test;
  final String message;

  Validator(this._test, this.message);

  bool call(T value) => _test(value);

  Validator<T> and(Validator<T> other) => Validator(
    (v) => this(v) && other(v),
    '$message และ ${other.message}',
  );

  Validator<T> or(Validator<T> other) => Validator(
    (v) => this(v) || other(v),
    '$message หรือ ${other.message}',
  );
}

// สร้าง validators
final isNotEmpty = Validator<String>(
  (s) => s.isNotEmpty,
  'ต้องไม่ว่างเปล่า',
);

final isLongEnough = Validator<String>(
  (s) => s.length >= 8,
  'ต้องมีอย่างน้อย 8 ตัวอักษร',
);

final hasUpperCase = Validator<String>(
  (s) => s.contains(RegExp(r'[A-Z]')),
  'ต้องมีตัวพิมพ์ใหญ่',
);

final hasNumber = Validator<String>(
  (s) => s.contains(RegExp(r'[0-9]')),
  'ต้องมีตัวเลข',
);

void testValidators() {
  print('\n=== Callable Validators ===');

  var passwordValidator = isNotEmpty
      .and(isLongEnough)
      .and(hasUpperCase)
      .and(hasNumber);

  var passwords = ['123', 'password', 'Password1', 'Secure123'];

  for (var pwd in passwords) {
    var valid = passwordValidator(pwd);
    print('$pwd: ${valid ? "✅" : "❌ ${passwordValidator.message}"}');
  }
}

// Memoize callable
class Memoize<T, R> {
  final R Function(T) _fn;
  final Map<T, R> _cache = {};

  Memoize(this._fn);

  R call(T input) {
    return _cache.putIfAbsent(input, () => _fn(input));
  }

  void clearCache() => _cache.clear();
  int get cacheSize => _cache.length;
}

int fibonacci(int n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

void testMemoize() {
  print('\n=== Memoize Callable ===');

  final memoFib = Memoize(fibonacci);

  final stopwatch = Stopwatch()..start();
  for (var i = 0; i < 10; i++) {
    memoFib(35);
  }
  stopwatch.stop();

  print('เรียก fibonacci(35) 10 ครั้ง: ${stopwatch.elapsedMilliseconds}ms');
  print('Cache size: ${memoFib.cacheSize}');
}

// Pipeline callable
class Pipeline<T> {
  final List<T Function(T)> _steps;

  Pipeline(this._steps);

  T call(T input) {
    return _steps.fold(input, (value, step) => step(value));
  }

  Pipeline<T> then(T Function(T) step) {
    return Pipeline([..._steps, step]);
  }
}

void testPipeline() {
  print('\n=== Pipeline Callable ===');

  var textPipeline = Pipeline<String>([
    (s) => s.trim(),
    (s) => s.toLowerCase(),
    (s) => s.replaceAll(RegExp(r'\s+'), '_'),
  ]);

  print(textPipeline('  Hello World  ')); // hello_world
  print(textPipeline('  Dart Programming  ')); // dart_programming
}
```

## 15.4 Extension Properties

```dart
// Extension กับ properties
extension DateTimeExtensions on DateTime {
  // Properties
  bool get isToday {
    final now = DateTime.now();
    return year == now.year && month == now.month && day == now.day;
  }

  bool get isYesterday {
    final yesterday = DateTime.now().subtract(Duration(days: 1));
    return year == yesterday.year &&
        month == yesterday.month &&
        day == yesterday.day;
  }

  bool get isTomorrow {
    final tomorrow = DateTime.now().add(Duration(days: 1));
    return year == tomorrow.year &&
        month == tomorrow.month &&
        day == tomorrow.day;
  }

  bool get isWeekend => weekday == DateTime.saturday || weekday == DateTime.sunday;
  bool get isWeekday => !isWeekend;

  String get thaiWeekday {
    const days = ['จันทร์', 'อังคาร', 'พุธ', 'พฤหัสบดี', 'ศุกร์', 'เสาร์', 'อาทิตย์'];
    return days[weekday - 1];
  }

  String get thaiMonth {
    const months = [
      'มกราคม', 'กุมภาพันธ์', 'มีนาคม', 'เมษายน',
      'พฤษภาคม', 'มิถุนายน', 'กรกฎาคม', 'สิงหาคม',
      'กันยายน', 'ตุลาคม', 'พฤศจิกายน', 'ธันวาคม',
    ];
    return months[month - 1];
  }

  int get thaiYear => year + 543; // พ.ศ.

  String get thaiDate => '$day $thaiMonth ${thaiYear}';
  String get thaiDateTime => '$thaiDate เวลา ${hour.padLeft(2)}:${minute.padLeft(2)} น.';

  // Methods
  DateTime startOfDay => DateTime(year, month, day);
  DateTime endOfDay => DateTime(year, month, day, 23, 59, 59, 999);

  DateTime addDays(int days) => add(Duration(days: days));
  DateTime subtractDays(int days) => subtract(Duration(days: days));

  String timeAgo() {
    final now = DateTime.now();
    final diff = now.difference(this);

    if (diff.inSeconds < 60) return 'เมื่อกี้';
    if (diff.inMinutes < 60) return '${diff.inMinutes} นาทีที่แล้ว';
    if (diff.inHours < 24) return '${diff.inHours} ชั่วโมงที่แล้ว';
    if (diff.inDays < 7) return '${diff.inDays} วันที่แล้ว';
    if (diff.inDays < 30) return '${diff.inDays ~/ 7} สัปดาห์ที่แล้ว';
    if (diff.inDays < 365) return '${diff.inDays ~/ 30} เดือนที่แล้ว';
    return '${diff.inDays ~/ 365} ปีที่แล้ว';
  }
}

extension IntTimeExtension on int {
  String padLeft(int width, [String pad = '0']) =>
      toString().padLeft(width, pad);
}

void testDateTimeExtensions() {
  print('\n=== DateTime Extensions ===');

  final now = DateTime.now();
  print('วันนี้: ${now.thaiDate}');
  print('วันและเวลา: ${now.thaiDateTime}');
  print('วันในสัปดาห์: ${now.thaiWeekday}');
  print('เป็น weekend: ${now.isWeekend}');

  final yesterday = now.subtractDays(1);
  print('เมื่อวาน: ${yesterday.timeAgo()}');

  final oldDate = now.subtract(Duration(days: 10));
  print('10 วันที่แล้ว: ${oldDate.timeAgo()}');
}

// Extension กับ Color (Flutter)
// extension ColorExtensions on Color {
//   Color withAlphaPercent(double percent) {
//     return withAlpha((255 * percent / 100).round());
//   }

//   Color darken(double amount) {
//     final hsl = HSLColor.fromColor(this);
//     return hsl.withLightness((hsl.lightness - amount).clamp(0.0, 1.0)).toColor();
//   }

//   Color lighten(double amount) {
//     final hsl = HSLColor.fromColor(this);
//     return hsl.withLightness((hsl.lightness + amount).clamp(0.0, 1.0)).toColor();
//   }

//   String get hex {
//     return '#${value.toRadixString(16).padLeft(8, '0').substring(2)}';
//   }
// }
```

## 15.5 Workshop: String and DateTime Extensions

สร้าง library ของ extensions ที่ครอบคลุม

```dart
// ===== String Utilities =====

extension StringUtils on String {
  // === Validation ===
  bool get isEmail =>
      RegExp(r'^[\w.+-]+@[\w-]+\.[a-z]{2,}$', caseSensitive: false)
          .hasMatch(this);

  bool get isUrl =>
      RegExp(r'^https?://[\w.-]+(?:\.[\w.-]+)+[\w\-._~:/?#\[\]@!$&\'()*+,;=.]+$')
          .hasMatch(this);

  bool get isThaiPhone =>
      RegExp(r'^0[689]\d{8}$').hasMatch(replaceAll(RegExp(r'[\s-]'), ''));

  bool get isThaiIdCard =>
      RegExp(r'^\d{13}$').hasMatch(replaceAll(RegExp(r'[-\s]'), ''));

  bool get isNumeric => RegExp(r'^\d+$').hasMatch(this);
  bool get isAlpha => RegExp(r'^[a-zA-Z]+$').hasMatch(this);
  bool get isAlphanumeric => RegExp(r'^[a-zA-Z0-9]+$').hasMatch(this);

  // === Transformation ===
  String get camelCase {
    final words = split(RegExp(r'[\s_-]+'));
    if (words.isEmpty) return this;
    return words[0].toLowerCase() +
        words.skip(1).map((w) => w.capitalize()).join();
  }

  String get snakeCase {
    return replaceAllMapped(
      RegExp(r'([a-z])([A-Z])'),
      (m) => '${m[1]}_${m[2]}',
    ).toLowerCase().replaceAll(RegExp(r'[\s-]+'), '_');
  }

  String get kebabCase => snakeCase.replaceAll('_', '-');

  String capitalize() {
    if (isEmpty) return this;
    return '${this[0].toUpperCase()}${substring(1)}';
  }

  String titleCase() =>
      split(' ').map((w) => w.capitalize()).join(' ');

  String truncate(int maxLength, {String suffix = '...'}) {
    if (length <= maxLength) return this;
    return substring(0, maxLength - suffix.length) + suffix;
  }

  // === Extraction ===
  List<String> get words =>
      trim().isEmpty ? [] : trim().split(RegExp(r'\s+'));

  List<String> get sentences =>
      split(RegExp(r'[.!?]+')).where((s) => s.trim().isNotEmpty).toList();

  String? get firstWord => words.isNotEmpty ? words.first : null;
  String? get lastWord => words.isNotEmpty ? words.last : null;

  // Extract numbers
  List<int> get extractedInts =>
      RegExp(r'\d+').allMatches(this).map((m) => int.parse(m.group(0)!)).toList();

  // === Formatting ===
  String repeat(int times) => List.filled(times, this).join();

  String surroundWith(String prefix, [String? suffix]) =>
      '$prefix$this${suffix ?? prefix}';

  String wrapIn(String char) => surroundWith(char);

  String get masked {
    if (length <= 4) return '*' * length;
    return '${'*' * (length - 4)}${substring(length - 4)}';
  }

  // === Conversion ===
  int? toIntOrNull() => int.tryParse(this);
  double? toDoubleOrNull() => double.tryParse(this);
  bool? toBoolOrNull() {
    if (toLowerCase() == 'true') return true;
    if (toLowerCase() == 'false') return false;
    return null;
  }

  DateTime? toDateTimeOrNull() => DateTime.tryParse(this);
}

// ===== DateTime Utilities =====

extension DateTimeUtils on DateTime {
  // === Comparison ===
  bool isSameDayAs(DateTime other) =>
      year == other.year && month == other.month && day == other.day;

  bool isBefore(DateTime other) => compareTo(other) < 0;
  bool isAfter(DateTime other) => compareTo(other) > 0;
  bool isBetween(DateTime start, DateTime end) =>
      isAfter(start) && isBefore(end);

  // === Formatting ===
  String format({
    bool includeTime = false,
    bool includeDayName = false,
    bool useThaiBuddhistYear = false,
  }) {
    const thaiMonths = [
      'ม.ค.', 'ก.พ.', 'มี.ค.', 'เม.ย.', 'พ.ค.', 'มิ.ย.',
      'ก.ค.', 'ส.ค.', 'ก.ย.', 'ต.ค.', 'พ.ย.', 'ธ.ค.',
    ];
    const thaiDays = ['จ.', 'อ.', 'พ.', 'พฤ.', 'ศ.', 'ส.', 'อา.'];

    var parts = <String>[];

    if (includeDayName) parts.add(thaiDays[weekday - 1]);

    parts.add('$day');
    parts.add(thaiMonths[this.month - 1]);

    final displayYear = useThaiBuddhistYear ? year + 543 : year;
    parts.add('$displayYear');

    var result = parts.join(' ');

    if (includeTime) {
      result += ' ${hour.toString().padLeft(2, '0')}:${minute.toString().padLeft(2, '0')}';
    }

    return result;
  }

  String get relative {
    final now = DateTime.now();
    final diff = now.difference(this);

    if (diff.inSeconds.abs() < 60) return 'เมื่อกี้';
    if (diff.inMinutes.abs() < 60) {
      return diff.isNegative
          ? 'อีก ${diff.inMinutes.abs()} นาที'
          : '${diff.inMinutes} นาทีที่แล้ว';
    }
    if (diff.inHours.abs() < 24) {
      return diff.isNegative
          ? 'อีก ${diff.inHours.abs()} ชั่วโมง'
          : '${diff.inHours} ชั่วโมงที่แล้ว';
    }
    if (diff.inDays.abs() < 7) {
      return diff.isNegative
          ? 'อีก ${diff.inDays.abs()} วัน'
          : '${diff.inDays} วันที่แล้ว';
    }

    return format(useThaiBuddhistYear: true);
  }

  // === Navigation ===
  DateTime get startOfWeek => subtract(Duration(days: weekday - 1));
  DateTime get endOfWeek => startOfWeek.add(Duration(days: 6));
  DateTime get startOfMonth => DateTime(year, month, 1);
  DateTime get endOfMonth => DateTime(year, month + 1, 0);

  List<DateTime> daysInMonth() {
    return List.generate(
      endOfMonth.day,
      (i) => DateTime(year, month, i + 1),
    );
  }

  // คำนวณอายุ
  int ageInYears() {
    final now = DateTime.now();
    var age = now.year - year;
    if (now.month < month || (now.month == month && now.day < day)) {
      age--;
    }
    return age;
  }
}

// ===== Workshop Tests =====

void main() {
  print('=== Workshop: String & DateTime Extensions ===\n');

  // String tests
  print('--- String Tests ---');

  var email = 'user@example.com';
  var badEmail = 'not-an-email';
  print('$email เป็น email: ${email.isEmail}');
  print('$badEmail เป็น email: ${badEmail.isEmail}');

  var phone = '0812345678';
  print('$phone เป็นเบอร์ไทย: ${phone.isThaiPhone}');

  var camel = 'helloWorldFromDart';
  print('camelCase → snake: ${camel.snakeCase}');
  print('camelCase → kebab: ${camel.kebabCase}');

  var snake = 'hello_world_dart';
  print('snake → camel: ${snake.camelCase}');

  var longText = 'นี่คือข้อความที่ยาวมากๆ ควรจะถูกตัดให้สั้นลง';
  print('truncate: ${longText.truncate(20)}');

  var creditCard = '1234567890123456';
  print('masked: ${creditCard.masked}');

  var text = 'ราคา 100 บาท ส่วนลด 20 บาท';
  print('extracted ints: ${text.extractedInts}');

  print('\n--- DateTime Tests ---');

  final now = DateTime.now();
  print('วันนี้: ${now.format(includeDayName: true, useThaiBuddhistYear: true)}');

  final yesterday = now.subtract(Duration(days: 1));
  print('เมื่อวาน: ${yesterday.relative}');

  final nextWeek = now.add(Duration(days: 7));
  print('อีก 7 วัน: ${nextWeek.relative}');

  final birthday = DateTime(1990, 5, 15);
  print('วันเกิด: ${birthday.format(useThaiBuddhistYear: true)}');
  print('อายุ: ${birthday.ageInYears()} ปี');

  print('\nวันในเดือนนี้:');
  final thisMonth = DateTime(now.year, now.month);
  print('วันเริ่มต้นสัปดาห์: ${thisMonth.startOfWeek.format()}');
  print('วันสิ้นสุดสัปดาห์: ${thisMonth.endOfWeek.format()}');
  print('วันแรกของเดือน: ${thisMonth.startOfMonth.format()}');
  print('วันสุดท้ายของเดือน: ${thisMonth.endOfMonth.format()}');

  print('\n=== เสร็จสิ้น Workshop ===');
}
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Extension methods** - เพิ่ม method ให้ class ที่มีอยู่
2. **Extension บน built-in types** - String, int, double, List, Map
3. **Callable classes** - class ที่เรียกได้เหมือน function ด้วย `call()`
4. **Extension properties** - เพิ่ม property ให้ class
5. **Workshop** - String และ DateTime extensions ที่ใช้งานได้จริง

### Best Practices

- ตั้งชื่อ extension ให้สื่อความหมาย (XxxExtensions หรือ XxxUtils)
- ไม่ควร extend class มากเกินไปจน confusing
- ใช้ extension สำหรับ convenience methods ไม่ใช่ core logic
- Callable classes เหมาะสำหรับ strategy pattern และ function objects
- Documentation สำหรับ extension สำคัญมาก
