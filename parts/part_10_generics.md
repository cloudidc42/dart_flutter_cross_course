# Part 10: Generics

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- เข้าใจว่า Generics คืออะไรและทำไมถึงสำคัญ
- สร้าง Generic classes และ Generic methods ได้
- ใช้ Type constraints (`extends`) เพื่อจำกัดประเภท
- ทำงานกับ Generic collections ได้อย่างมีประสิทธิภาพ
- เข้าใจ Covariance และ Contravariance
- สร้าง Generic Stack และ Queue เป็น workshop

---

## 10.1 ทำไมต้องใช้ Generics?

```dart
// ❌ ปัญหาโดยไม่ใช้ Generics - ต้องเขียนซ้ำหรือใช้ dynamic

class IntBox {
  int value;
  IntBox(this.value);
  int get() => value;
  void set(int v) => value = v;
}

class StringBox {
  String value;
  StringBox(this.value);
  String get() => value;
  void set(String v) => value = v;
}

// ใช้ dynamic - เสีย type safety
class DynamicBox {
  dynamic value;
  DynamicBox(this.value);
  dynamic get() => value;
  void set(dynamic v) => value = v;
}

// ✅ ใช้ Generics - เขียนครั้งเดียว ใช้ได้ทุก type
class Box<T> {
  T value;
  Box(this.value);
  T get() => value;
  void set(T v) => value = v;
  
  Box<T> copy() => Box<T>(value);
  
  // ทำงานกับ value
  Box<R> transform<R>(R Function(T) transform) {
    return Box<R>(transform(value));
  }
  
  @override
  String toString() => 'Box<${T.toString()}>($value)';
}

void main() {
  // Type safe - Dart รู้ว่าเป็น int
  Box<int> intBox = Box<int>(42);
  int num = intBox.get(); // ไม่ต้อง cast
  
  // Type safe - Dart รู้ว่าเป็น String  
  Box<String> strBox = Box<String>('Hello Thailand');
  String str = strBox.get(); // ไม่ต้อง cast
  
  // Type inference - Dart เดา type ให้ได้
  var doubleBox = Box(3.14); // Box<double>
  
  print(intBox);    // Box<int>(42)
  print(strBox);    // Box<String>(Hello Thailand)
  print(doubleBox); // Box<double>(3.14)
  
  // Transform
  Box<String> strFromInt = intBox.transform((n) => 'จำนวน: $n');
  print(strFromInt); // Box<String>(จำนวน: 42)
  
  // ❌ Compile error - type safe
  // intBox.set('hello'); // Error: String ไม่ใช่ int
}
```

---

## 10.2 Generic Classes

### Container Class

```dart
// Generic Container ที่ใช้ pattern ต่างๆ

class Optional<T> {
  final T? _value;
  
  const Optional._(this._value);
  
  factory Optional.of(T value) => Optional._(value);
  factory Optional.empty() => Optional._(null);
  
  bool get isPresent => _value != null;
  bool get isEmpty => _value == null;
  
  T get value {
    if (_value == null) throw StateError('Optional ว่างเปล่า');
    return _value;
  }
  
  T orElse(T defaultValue) => _value ?? defaultValue;
  T? orNull() => _value;
  
  Optional<R> map<R>(R Function(T) mapper) {
    if (_value == null) return Optional.empty();
    return Optional.of(mapper(_value));
  }
  
  Optional<R> flatMap<R>(Optional<R> Function(T) mapper) {
    if (_value == null) return Optional.empty();
    return mapper(_value);
  }
  
  void ifPresent(void Function(T) action) {
    if (_value != null) action(_value);
  }
  
  void ifPresentOrElse(void Function(T) action, void Function() emptyAction) {
    if (_value != null) {
      action(_value);
    } else {
      emptyAction();
    }
  }
  
  @override
  String toString() => isPresent ? 'Optional($value)' : 'Optional.empty()';
}

void main() {
  // ใช้งาน Optional
  Optional<String> name = Optional.of('สมชาย');
  Optional<String> empty = Optional.empty();
  
  print(name.isPresent); // true
  print(empty.isEmpty);  // true
  
  // orElse
  print(name.orElse('ไม่ระบุ'));  // สมชาย
  print(empty.orElse('ไม่ระบุ')); // ไม่ระบุ
  
  // map
  Optional<int> nameLength = name.map((s) => s.length);
  print(nameLength); // Optional(3)
  
  // ifPresent
  name.ifPresent((n) => print('สวัสดี $n'));
  empty.ifPresent((n) => print('ไม่แสดง')); // ไม่แสดง
  
  // chain
  Optional<String> result = Optional.of('  HELLO WORLD  ')
      .map((s) => s.trim())
      .map((s) => s.toLowerCase());
  print(result); // Optional(hello world)
  
  // ตัวอย่างจริง: ค้นหาใน database
  Map<String, String> userDb = {
    'U001': 'สมชาย',
    'U002': 'มานี',
  };
  
  Optional<String> findUser(String id) {
    String? user = userDb[id];
    return user != null ? Optional.of(user) : Optional.empty();
  }
  
  findUser('U001').ifPresentOrElse(
    (name) => print('พบผู้ใช้: $name'),
    () => print('ไม่พบผู้ใช้'),
  ); // พบผู้ใช้: สมชาย
  
  findUser('U999').ifPresentOrElse(
    (name) => print('พบผู้ใช้: $name'),
    () => print('ไม่พบผู้ใช้'),
  ); // ไม่พบผู้ใช้
}
```

### Result Type

```dart
// Result<S, E> - แทน try/catch ด้วย type

abstract class Result<S, E extends Exception> {
  const Result();
  
  bool get isSuccess;
  bool get isFailure => !isSuccess;
  
  factory Result.success(S value) = Success<S, E>;
  factory Result.failure(E error) = Failure<S, E>;
  
  // Fold - handle ทั้ง success และ failure
  R fold<R>({
    required R Function(S value) onSuccess,
    required R Function(E error) onFailure,
  });
  
  S getOrElse(S defaultValue);
  S? getOrNull();
  
  Result<R, E> map<R>(R Function(S) mapper);
  Result<S, E> mapError<F extends Exception>(F Function(E) mapper);
  
  void when({
    void Function(S value)? success,
    void Function(E error)? failure,
  });
}

class Success<S, E extends Exception> extends Result<S, E> {
  final S value;
  const Success(this.value);
  
  @override
  bool get isSuccess => true;
  
  @override
  R fold<R>({
    required R Function(S value) onSuccess,
    required R Function(E error) onFailure,
  }) => onSuccess(value);
  
  @override
  S getOrElse(S defaultValue) => value;
  
  @override
  S? getOrNull() => value;
  
  @override
  Result<R, E> map<R>(R Function(S) mapper) => Success(mapper(value));
  
  @override
  Result<S, E> mapError<F extends Exception>(F Function(E) mapper) => this;
  
  @override
  void when({void Function(S)? success, void Function(E)? failure}) {
    success?.call(value);
  }
  
  @override
  String toString() => 'Success($value)';
}

class Failure<S, E extends Exception> extends Result<S, E> {
  final E error;
  const Failure(this.error);
  
  @override
  bool get isSuccess => false;
  
  @override
  R fold<R>({
    required R Function(S value) onSuccess,
    required R Function(E error) onFailure,
  }) => onFailure(error);
  
  @override
  S getOrElse(S defaultValue) => defaultValue;
  
  @override
  S? getOrNull() => null;
  
  @override
  Result<R, E> map<R>(R Function(S) mapper) => Failure(error);
  
  @override
  Result<S, E> mapError<F extends Exception>(F Function(E) mapper) {
    throw UnimplementedError();
  }
  
  @override
  void when({void Function(S)? success, void Function(E)? failure}) {
    failure?.call(error);
  }
  
  @override
  String toString() => 'Failure($error)';
}

class AppException implements Exception {
  final String message;
  final String code;
  const AppException(this.message, {this.code = 'UNKNOWN'});
  
  @override
  String toString() => 'AppException[$code]: $message';
}

// ฟังก์ชันที่คืน Result แทน throw
Result<double, AppException> divide(double a, double b) {
  if (b == 0) {
    return Result.failure(AppException('หารด้วยศูนย์ไม่ได้', code: 'DIV_ZERO'));
  }
  return Result.success(a / b);
}

Result<int, AppException> parseAge(String input) {
  int? age = int.tryParse(input);
  if (age == null) {
    return Result.failure(AppException('อายุต้องเป็นตัวเลข', code: 'INVALID_AGE'));
  }
  if (age < 0 || age > 150) {
    return Result.failure(AppException('อายุต้องอยู่ระหว่าง 0-150', code: 'AGE_RANGE'));
  }
  return Result.success(age);
}

void main() {
  print('=== Result Type ===\n');
  
  // หาร
  Result<double, AppException> r1 = divide(10, 3);
  Result<double, AppException> r2 = divide(10, 0);
  
  r1.when(
    success: (v) => print('10 / 3 = ${v.toStringAsFixed(4)}'),
    failure: (e) => print('Error: $e'),
  ); // 10 / 3 = 3.3333
  
  r2.when(
    success: (v) => print('10 / 0 = $v'),
    failure: (e) => print('Error: $e'),
  ); // Error: AppException[DIV_ZERO]: หารด้วยศูนย์ไม่ได้
  
  // fold
  String message = divide(20, 5).fold(
    onSuccess: (v) => 'ผลลัพธ์: $v',
    onFailure: (e) => 'เกิดข้อผิดพลาด: ${e.message}',
  );
  print(message); // ผลลัพธ์: 4.0
  
  // parse
  Result<int, AppException> age1 = parseAge('25');
  Result<int, AppException> age2 = parseAge('abc');
  Result<int, AppException> age3 = parseAge('200');
  
  print(age1.getOrElse(0));  // 25
  print(age2.getOrElse(0));  // 0
  print(age3.getOrNull());   // null
  
  // chain map
  Result<String, AppException> ageGroup = parseAge('22').map((age) {
    if (age < 18) return 'ผู้เยาว์';
    if (age < 65) return 'ผู้ใหญ่';
    return 'ผู้สูงอายุ';
  });
  print(ageGroup); // Success(ผู้ใหญ่)
}
```

---

## 10.3 Generic Methods

```dart
// Generic methods - เมื่อ type ขึ้นอยู่กับ call site ไม่ใช่ class

class ListUtils {
  ListUtils._();
  
  // Generic method - T เป็น type parameter ของ method ไม่ใช่ class
  static T first<T>(List<T> list) {
    if (list.isEmpty) throw RangeError('List ว่างเปล่า');
    return list.first;
  }
  
  static T? firstOrNull<T>(List<T> list) {
    return list.isEmpty ? null : list.first;
  }
  
  static T? findFirst<T>(List<T> list, bool Function(T) predicate) {
    for (T item in list) {
      if (predicate(item)) return item;
    }
    return null;
  }
  
  // Generic method ที่มี 2 type parameters
  static Map<K, V> zipToMap<K, V>(List<K> keys, List<V> values) {
    Map<K, V> result = {};
    int length = keys.length < values.length ? keys.length : values.length;
    for (int i = 0; i < length; i++) {
      result[keys[i]] = values[i];
    }
    return result;
  }
  
  // Group by
  static Map<K, List<T>> groupBy<T, K>(List<T> list, K Function(T) keyFn) {
    Map<K, List<T>> result = {};
    for (T item in list) {
      K key = keyFn(item);
      result.putIfAbsent(key, () => []);
      result[key]!.add(item);
    }
    return result;
  }
  
  // Partition - แบ่งออกเป็น 2 กลุ่ม
  static (List<T>, List<T>) partition<T>(List<T> list, bool Function(T) predicate) {
    List<T> trueList = [];
    List<T> falseList = [];
    for (T item in list) {
      if (predicate(item)) {
        trueList.add(item);
      } else {
        falseList.add(item);
      }
    }
    return (trueList, falseList);
  }
  
  // Chunk - แบ่ง list เป็น chunks
  static List<List<T>> chunk<T>(List<T> list, int size) {
    List<List<T>> chunks = [];
    for (int i = 0; i < list.length; i += size) {
      int end = (i + size).clamp(0, list.length);
      chunks.add(list.sublist(i, end));
    }
    return chunks;
  }
  
  // Zip - รวม 2 lists เป็น list ของ pairs
  static List<(A, B)> zip<A, B>(List<A> listA, List<B> listB) {
    int length = listA.length < listB.length ? listA.length : listB.length;
    return List.generate(length, (i) => (listA[i], listB[i]));
  }
  
  // Flatten
  static List<T> flatten<T>(List<List<T>> nestedList) {
    return nestedList.expand((list) => list).toList();
  }
  
  // Distinct by
  static List<T> distinctBy<T, K>(List<T> list, K Function(T) keyFn) {
    Map<K, T> seen = {};
    for (T item in list) {
      K key = keyFn(item);
      if (!seen.containsKey(key)) {
        seen[key] = item;
      }
    }
    return seen.values.toList();
  }
  
  // Aggregate stats
  static Map<String, double> stats(List<num> numbers) {
    if (numbers.isEmpty) return {};
    double sum = numbers.fold(0, (a, b) => a + b);
    double mean = sum / numbers.length;
    List<num> sorted = [...numbers]..sort();
    double median = sorted.length.isOdd
        ? sorted[sorted.length ~/ 2].toDouble()
        : (sorted[sorted.length ~/ 2 - 1] + sorted[sorted.length ~/ 2]) / 2;
    
    return {
      'count': numbers.length.toDouble(),
      'sum': sum,
      'mean': mean,
      'min': sorted.first.toDouble(),
      'max': sorted.last.toDouble(),
      'median': median,
    };
  }
}

void main() {
  print('=== Generic Methods ===\n');
  
  List<int> numbers = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5];
  
  // findFirst
  int? firstBig = ListUtils.findFirst(numbers, (n) => n > 5);
  print('ตัวแรกที่มากกว่า 5: $firstBig'); // 9
  
  // zipToMap
  List<String> keys = ['ชื่อ', 'อายุ', 'เมือง'];
  List<dynamic> values = ['สมชาย', 25, 'กรุงเทพฯ'];
  Map<String, dynamic> person = ListUtils.zipToMap(keys, values);
  print('\nzipToMap: $person');
  
  // groupBy
  List<String> fruits = ['แอปเปิ้ล', 'กล้วย', 'องุ่น', 'มะม่วง', 'แก้วมังกร', 'กีวี'];
  Map<int, List<String>> byLength = ListUtils.groupBy(fruits, (f) => f.length);
  print('\nจัดกลุ่มตามความยาว:');
  byLength.forEach((len, items) => print('  $len: $items'));
  
  // partition
  var (evens, odds) = ListUtils.partition(numbers, (n) => n % 2 == 0);
  print('\nเลขคู่: $evens');
  print('เลขคี่: $odds');
  
  // chunk
  List<List<int>> chunks = ListUtils.chunk([1,2,3,4,5,6,7,8,9,10], 3);
  print('\nChunk ละ 3: $chunks');
  
  // zip
  List<String> names = ['สมชาย', 'มานี', 'วิชัย'];
  List<int> scores = [85, 92, 78];
  List<(String, int)> zipped = ListUtils.zip(names, scores);
  print('\nZip:');
  zipped.forEach((pair) => print('  ${pair.$1}: ${pair.$2}'));
  
  // stats
  List<double> testScores = [78, 82, 91, 65, 88, 76, 95, 71, 84, 90];
  Map<String, double> scoreStats = ListUtils.stats(testScores);
  print('\nสถิติคะแนน:');
  scoreStats.forEach((stat, value) => print('  $stat: ${value.toStringAsFixed(2)}'));
  
  // distinctBy
  List<Map<String, dynamic>> students = [
    {'id': 1, 'name': 'สมชาย', 'grade': 'A'},
    {'id': 2, 'name': 'มานี', 'grade': 'B'},
    {'id': 1, 'name': 'สมชาย ซ้ำ', 'grade': 'A'}, // id ซ้ำ
  ];
  List<Map<String, dynamic>> unique = ListUtils.distinctBy(
    students, 
    (s) => s['id'],
  );
  print('\nDistinct by id: ${unique.length} รายการ');
}
```

---

## 10.4 Type Constraints (extends)

```dart
// ใช้ extends เพื่อบอกว่า Generic type ต้องเป็น subtype ของอะไร

// Constraint - T ต้องเป็น Comparable
class SortedList<T extends Comparable<T>> {
  final List<T> _data = [];
  
  void add(T item) {
    // Binary search insertion
    int low = 0, high = _data.length;
    while (low < high) {
      int mid = (low + high) ~/ 2;
      if (_data[mid].compareTo(item) <= 0) {
        low = mid + 1;
      } else {
        high = mid;
      }
    }
    _data.insert(low, item);
  }
  
  void addAll(Iterable<T> items) => items.forEach(add);
  
  bool contains(T item) => _data.contains(item);
  
  T? min() => _data.isEmpty ? null : _data.first;
  T? max() => _data.isEmpty ? null : _data.last;
  
  List<T> range(T from, T to) {
    return _data.where((item) => 
        item.compareTo(from) >= 0 && item.compareTo(to) <= 0
    ).toList();
  }
  
  List<T> toList() => List.unmodifiable(_data);
  
  int get length => _data.length;
  
  @override
  String toString() => 'SortedList($_data)';
}

// Constraint - T ต้องมี id field
abstract class HasId {
  String get id;
}

class Repository<T extends HasId> {
  final Map<String, T> _store = {};
  
  void save(T entity) {
    _store[entity.id] = entity;
    print('บันทึก ${entity.runtimeType} id=${entity.id}');
  }
  
  T? findById(String id) => _store[id];
  
  List<T> findAll() => _store.values.toList();
  
  bool delete(String id) => _store.remove(id) != null;
  
  int get count => _store.length;
}

class User implements HasId {
  @override
  final String id;
  String name;
  String email;
  
  User({required this.id, required this.name, required this.email});
  
  @override
  String toString() => 'User($id: $name)';
}

class Product implements HasId {
  @override
  final String id;
  String name;
  double price;
  
  Product({required this.id, required this.name, required this.price});
  
  @override
  String toString() => 'Product($id: $name ฿$price)';
}

// Multiple constraints (Dart ไม่รองรับ A extends B, C แต่ทำได้ด้วย abstract class)
abstract class Storable extends HasId {
  DateTime get createdAt;
  DateTime? get updatedAt;
  bool get isActive;
}

class StorableRepository<T extends Storable> extends Repository<T> {
  List<T> findActive() => findAll().where((e) => e.isActive).toList();
  
  List<T> findByDateRange(DateTime from, DateTime to) {
    return findAll().where((e) => 
      e.createdAt.isAfter(from) && e.createdAt.isBefore(to)
    ).toList();
  }
}

// Bounded numeric type
double sum<T extends num>(List<T> numbers) {
  return numbers.fold(0, (acc, n) => acc + n);
}

T max<T extends Comparable<T>>(T a, T b) {
  return a.compareTo(b) >= 0 ? a : b;
}

T min<T extends Comparable<T>>(T a, T b) {
  return a.compareTo(b) <= 0 ? a : b;
}

T clamp<T extends Comparable<T>>(T value, T minimum, T maximum) {
  if (value.compareTo(minimum) < 0) return minimum;
  if (value.compareTo(maximum) > 0) return maximum;
  return value;
}

void main() {
  print('=== Type Constraints ===\n');
  
  // SortedList
  SortedList<int> sortedNums = SortedList<int>();
  sortedNums.addAll([5, 2, 8, 1, 9, 3, 7, 4, 6]);
  print('Sorted: $sortedNums');
  print('Min: ${sortedNums.min()}, Max: ${sortedNums.max()}');
  print('Range 3-7: ${sortedNums.range(3, 7)}');
  
  SortedList<String> sortedStrings = SortedList<String>();
  sortedStrings.addAll(['กล้วย', 'แอปเปิ้ล', 'ส้ม', 'มะม่วง', 'องุ่น']);
  print('\nSorted strings: $sortedStrings');
  
  // Repository
  print('\n=== Repository ===');
  Repository<User> userRepo = Repository<User>();
  Repository<Product> productRepo = Repository<Product>();
  
  userRepo.save(User(id: 'U001', name: 'สมชาย', email: 'somchai@test.com'));
  userRepo.save(User(id: 'U002', name: 'มานี', email: 'manee@test.com'));
  
  productRepo.save(Product(id: 'P001', name: 'กาแฟ', price: 60.0));
  productRepo.save(Product(id: 'P002', name: 'ชา', price: 45.0));
  
  print(userRepo.findById('U001'));
  print(productRepo.findAll());
  
  // Bounded generic functions
  print('\n=== Generic Functions ===');
  print('sum: ${sum([1, 2, 3, 4, 5])}');
  print('sum double: ${sum([1.5, 2.5, 3.5])}');
  print('max(5, 3): ${max(5, 3)}');
  print('max("ข", "ก"): ${max("ข", "ก")}');
  print('clamp(15, 0, 10): ${clamp(15, 0, 10)}');
  print('clamp(-5, 0, 10): ${clamp(-5, 0, 10)}');
}
```

---

## 10.5 Generic Collections

```dart
// Custom Generic Data Structures

class Pair<A, B> {
  final A first;
  final B second;
  
  const Pair(this.first, this.second);
  
  // swap - คืน Pair กลับ
  Pair<B, A> swap() => Pair(second, first);
  
  Pair<C, B> mapFirst<C>(C Function(A) mapper) => Pair(mapper(first), second);
  Pair<A, C> mapSecond<C>(C Function(B) mapper) => Pair(first, mapper(second));
  
  @override
  String toString() => '($first, $second)';
  
  @override
  bool operator ==(Object other) =>
      other is Pair<A, B> && first == other.first && second == other.second;
  
  @override
  int get hashCode => Object.hash(first, second);
}

class Triple<A, B, C> {
  final A first;
  final B second;
  final C third;
  
  const Triple(this.first, this.second, this.third);
  
  @override
  String toString() => '($first, $second, $third)';
}

// BiMap - Map ที่ lookup ได้ทั้ง key -> value และ value -> key
class BiMap<K, V> {
  final Map<K, V> _forward = {};
  final Map<V, K> _inverse = {};
  
  void put(K key, V value) {
    // ลบ mappings เก่าถ้ามี
    if (_forward.containsKey(key)) {
      _inverse.remove(_forward[key]);
    }
    if (_inverse.containsKey(value)) {
      _forward.remove(_inverse[value]);
    }
    _forward[key] = value;
    _inverse[value] = key;
  }
  
  V? getValue(K key) => _forward[key];
  K? getKey(V value) => _inverse[value];
  
  bool containsKey(K key) => _forward.containsKey(key);
  bool containsValue(V value) => _inverse.containsKey(value);
  
  void removeByKey(K key) {
    V? value = _forward.remove(key);
    if (value != null) _inverse.remove(value);
  }
  
  void removeByValue(V value) {
    K? key = _inverse.remove(value);
    if (key != null) _forward.remove(key);
  }
  
  int get length => _forward.length;
  
  @override
  String toString() => _forward.toString();
}

// Multimap - Map ที่ key ชี้ไปหลาย values
class Multimap<K, V> {
  final Map<K, List<V>> _data = {};
  
  void add(K key, V value) {
    _data.putIfAbsent(key, () => []);
    _data[key]!.add(value);
  }
  
  void addAll(K key, Iterable<V> values) {
    _data.putIfAbsent(key, () => []);
    _data[key]!.addAll(values);
  }
  
  List<V> get(K key) => _data[key] ?? [];
  
  bool containsKey(K key) => _data.containsKey(key);
  
  bool containsEntry(K key, V value) => get(key).contains(value);
  
  void remove(K key, V value) {
    _data[key]?.remove(value);
    if (_data[key]?.isEmpty ?? false) _data.remove(key);
  }
  
  Set<K> get keys => _data.keys.toSet();
  
  int get keyCount => _data.length;
  int get valueCount => _data.values.fold(0, (sum, list) => sum + list.length);
  
  @override
  String toString() => _data.toString();
}

void main() {
  print('=== Generic Collections ===\n');
  
  // Pair
  Pair<String, int> nameAge = Pair('สมชาย', 25);
  print('Pair: $nameAge');
  print('Swap: ${nameAge.swap()}');
  
  Pair<String, String> formatted = nameAge.mapSecond((age) => 'อายุ $age ปี');
  print('Mapped: $formatted');
  
  // Unzip เป็น 2 lists
  List<Pair<String, int>> pairs = [
    Pair('สมชาย', 85),
    Pair('มานี', 92),
    Pair('วิชัย', 78),
  ];
  
  List<String> names = pairs.map((p) => p.first).toList();
  List<int> scores = pairs.map((p) => p.second).toList();
  print('\nNames: $names');
  print('Scores: $scores');
  
  // BiMap
  print('\n=== BiMap ===');
  BiMap<String, String> countryToCapital = BiMap();
  countryToCapital.put('ไทย', 'กรุงเทพฯ');
  countryToCapital.put('ญี่ปุ่น', 'โตเกียว');
  countryToCapital.put('ฝรั่งเศส', 'ปารีส');
  
  print('เมืองหลวงของไทย: ${countryToCapital.getValue("ไทย")}');
  print('ประเทศที่มีเมืองหลวงโตเกียว: ${countryToCapital.getKey("โตเกียว")}');
  
  // Multimap
  print('\n=== Multimap ===');
  Multimap<String, String> hobbies = Multimap();
  hobbies.add('สมชาย', 'อ่านหนังสือ');
  hobbies.add('สมชาย', 'ฟังเพลง');
  hobbies.add('สมชาย', 'เล่นกีฬา');
  hobbies.add('มานี', 'ทำอาหาร');
  hobbies.add('มานี', 'อ่านหนังสือ');
  
  print('งานอดิเรกสมชาย: ${hobbies.get("สมชาย")}');
  print('งานอดิเรกมานี: ${hobbies.get("มานี")}');
  print('Keys: ${hobbies.keys}');
  print('Key count: ${hobbies.keyCount}, Value count: ${hobbies.valueCount}');
}
```

---

## 10.6 Covariance และ Contravariance

```dart
// Covariance - ใช้ out keyword (read-only position)
// ใน Dart, covariance ทำงานกับ generics โดย default

void main() {
  // Covariance - List<Dog> เป็น List<Animal> ได้ (ถ้า Animal เป็น covariant)
  
  // แต่ใน Dart, List ไม่ fully covariant (runtime check)
  List<String> strings = ['a', 'b', 'c'];
  List<Object> objects = strings; // Dart allow นี้ (covariant assignment)
  
  print(objects); // [a, b, c]
  
  // แต่ถ้า add object เข้าไปจะ runtime error
  try {
    objects.add(42); // Runtime Error! int ไม่ใช่ String
  } catch (e) {
    print('Runtime Error: $e');
  }
  
  // Type-safe covariance - ใช้ getter ที่ return covariant type
  
  // Contravariance example - function parameters
  void Function(Object) objectHandler = (obj) => print('Object: $obj');
  void Function(String) stringHandler = objectHandler; // OK - contravariant
  stringHandler('Hello'); // Object: Hello
}

// Covariant parameter - ใช้เมื่อต้องการ override parameter type
class Animal {
  void eat(Object food) {
    print('Animal กิน $food');
  }
}

class Cat extends Animal {
  // covariant - บอกว่า parameter ถูก override ด้วย subtype
  @override
  void eat(covariant String fishName) {
    print('แมวกิน $fishName');
  }
}

// Generic Variance Pattern ที่ใช้บ่อย
class Producer<out T> {
  // จำลอง covariant producer (read-only)
}

// ตัวอย่างจริง: Producer vs Consumer
abstract class Source<T> {
  T produce();
}

abstract class Sink<T> {
  void consume(T value);
}

class NumberSource implements Source<int> {
  int _current = 0;
  
  @override
  int produce() => _current++;
}

class PrintSink implements Sink<Object> {
  @override
  void consume(Object value) => print('ได้รับ: $value');
}

void processSource<T>(Source<T> source, Sink<T> sink, int count) {
  for (int i = 0; i < count; i++) {
    sink.consume(source.produce());
  }
}

void main2() {
  NumberSource numbers = NumberSource();
  Sink<Object> printer = PrintSink();
  
  // Sink<Object> ใช้ได้กับ Sink<int> เพราะ contravariant
  // (Object เป็น supertype ของ int)
  processSource<int>(numbers, printer, 5);
}
```

---

## 10.7 Workshop: Generic Stack และ Queue

```dart
// data_structures.dart
// Generic Stack และ Queue ที่ใช้งานได้จริง

// === Generic Stack ===
// LIFO - Last In First Out

class Stack<T> {
  final List<T> _elements = [];
  final int? maxSize;
  
  Stack({this.maxSize});
  
  // Push - เพิ่มที่ top
  bool push(T element) {
    if (maxSize != null && _elements.length >= maxSize!) {
      print('Stack เต็ม (max: $maxSize)');
      return false;
    }
    _elements.add(element);
    return true;
  }
  
  // Pop - ดึงออกจาก top
  T pop() {
    if (isEmpty) throw StackUnderflowException('Stack ว่างเปล่า');
    return _elements.removeLast();
  }
  
  // Peek - ดู top โดยไม่ลบ
  T peek() {
    if (isEmpty) throw StackUnderflowException('Stack ว่างเปล่า');
    return _elements.last;
  }
  
  T? peekOrNull() => isEmpty ? null : _elements.last;
  
  bool get isEmpty => _elements.isEmpty;
  bool get isNotEmpty => _elements.isNotEmpty;
  bool get isFull => maxSize != null && _elements.length >= maxSize!;
  
  int get size => _elements.length;
  
  void clear() => _elements.clear();
  
  // ดู elements ทั้งหมด (top เป็น last)
  List<T> toList() => List.unmodifiable(_elements);
  
  @override
  String toString() {
    if (isEmpty) return 'Stack(empty)';
    return 'Stack(top: ${_elements.last}, size: ${_elements.length})';
  }
}

class StackUnderflowException implements Exception {
  final String message;
  StackUnderflowException(this.message);
  
  @override
  String toString() => 'StackUnderflowException: $message';
}

// === Generic Queue ===
// FIFO - First In First Out

class Queue<T> {
  final List<T> _elements = [];
  final int? maxSize;
  
  Queue({this.maxSize});
  
  // Enqueue - เพิ่มที่ท้าย
  bool enqueue(T element) {
    if (maxSize != null && _elements.length >= maxSize!) {
      return false;
    }
    _elements.add(element);
    return true;
  }
  
  // Dequeue - ดึงออกจากหน้า
  T dequeue() {
    if (isEmpty) throw QueueEmptyException('Queue ว่างเปล่า');
    return _elements.removeAt(0);
  }
  
  // Peek front
  T front() {
    if (isEmpty) throw QueueEmptyException('Queue ว่างเปล่า');
    return _elements.first;
  }
  
  T? frontOrNull() => isEmpty ? null : _elements.first;
  
  bool get isEmpty => _elements.isEmpty;
  bool get isNotEmpty => _elements.isNotEmpty;
  
  int get size => _elements.length;
  
  void clear() => _elements.clear();
  
  List<T> toList() => List.unmodifiable(_elements);
  
  @override
  String toString() {
    if (isEmpty) return 'Queue(empty)';
    return 'Queue(front: ${_elements.first}, size: ${_elements.length})';
  }
}

class QueueEmptyException implements Exception {
  final String message;
  QueueEmptyException(this.message);
  
  @override
  String toString() => 'QueueEmptyException: $message';
}

// === Priority Queue ===
class PriorityQueue<T> {
  final List<(int priority, T value)> _heap = [];
  final bool isMinHeap; // true = min heap (ลำดับต่ำ = ออกก่อน)
  
  PriorityQueue({this.isMinHeap = false}); // false = max heap
  
  void enqueue(T value, int priority) {
    _heap.add((priority, value));
    _siftUp(_heap.length - 1);
  }
  
  T dequeue() {
    if (isEmpty) throw QueueEmptyException('PriorityQueue ว่างเปล่า');
    
    T result = _heap.first.$2;
    
    if (_heap.length == 1) {
      _heap.clear();
    } else {
      _heap[0] = _heap.last;
      _heap.removeLast();
      _siftDown(0);
    }
    
    return result;
  }
  
  T peek() {
    if (isEmpty) throw QueueEmptyException('PriorityQueue ว่างเปล่า');
    return _heap.first.$2;
  }
  
  bool _compare(int a, int b) {
    return isMinHeap ? a < b : a > b;
  }
  
  void _siftUp(int i) {
    while (i > 0) {
      int parent = (i - 1) ~/ 2;
      if (_compare(_heap[i].$1, _heap[parent].$1)) {
        var temp = _heap[i];
        _heap[i] = _heap[parent];
        _heap[parent] = temp;
        i = parent;
      } else {
        break;
      }
    }
  }
  
  void _siftDown(int i) {
    while (true) {
      int left = 2 * i + 1;
      int right = 2 * i + 2;
      int target = i;
      
      if (left < _heap.length && _compare(_heap[left].$1, _heap[target].$1)) {
        target = left;
      }
      if (right < _heap.length && _compare(_heap[right].$1, _heap[target].$1)) {
        target = right;
      }
      
      if (target != i) {
        var temp = _heap[i];
        _heap[i] = _heap[target];
        _heap[target] = temp;
        i = target;
      } else {
        break;
      }
    }
  }
  
  bool get isEmpty => _heap.isEmpty;
  bool get isNotEmpty => _heap.isNotEmpty;
  int get size => _heap.length;
}

// === LRU Cache ===
class LRUCache<K, V> {
  final int capacity;
  final Map<K, V> _cache = {};
  final List<K> _order = []; // Most recent at end
  
  LRUCache(this.capacity);
  
  V? get(K key) {
    if (!_cache.containsKey(key)) return null;
    
    // Move to end (most recent)
    _order.remove(key);
    _order.add(key);
    
    return _cache[key];
  }
  
  void put(K key, V value) {
    if (_cache.containsKey(key)) {
      _order.remove(key);
    } else if (_cache.length >= capacity) {
      // Evict least recently used (front of list)
      K lru = _order.removeAt(0);
      _cache.remove(lru);
      print('LRU Evict: $lru');
    }
    
    _cache[key] = value;
    _order.add(key);
  }
  
  bool containsKey(K key) => _cache.containsKey(key);
  
  int get size => _cache.length;
  
  @override
  String toString() {
    return 'LRUCache(size: $size/$capacity, order: $_order)';
  }
}

// === Applications ===

// 1. undo/redo ด้วย Stack
class UndoRedoManager<T> {
  final Stack<T> _undoStack = Stack();
  final Stack<T> _redoStack = Stack(maxSize: 50); // จำกัด 50 undo
  T _currentState;
  
  UndoRedoManager(this._currentState);
  
  T get state => _currentState;
  
  void executeAction(T newState) {
    _undoStack.push(_currentState);
    _redoStack.clear(); // ล้าง redo stack เมื่อมีการกระทำใหม่
    _currentState = newState;
    print('Action: $newState');
  }
  
  bool undo() {
    if (_undoStack.isEmpty) {
      print('ไม่มีอะไรให้ Undo');
      return false;
    }
    _redoStack.push(_currentState);
    _currentState = _undoStack.pop();
    print('Undo -> $_currentState');
    return true;
  }
  
  bool redo() {
    if (_redoStack.isEmpty) {
      print('ไม่มีอะไรให้ Redo');
      return false;
    }
    _undoStack.push(_currentState);
    _currentState = _redoStack.pop();
    print('Redo -> $_currentState');
    return true;
  }
  
  bool get canUndo => _undoStack.isNotEmpty;
  bool get canRedo => _redoStack.isNotEmpty;
}

// 2. Task Queue ด้วย Priority Queue
class Task {
  final String id;
  final String name;
  final String description;
  
  Task({required this.id, required this.name, required this.description});
  
  @override
  String toString() => 'Task[$id]: $name';
}

class TaskManager {
  final PriorityQueue<Task> _queue = PriorityQueue(isMinHeap: false); // High priority first
  final List<Task> _completed = [];
  
  void addTask(Task task, {int priority = 5}) {
    _queue.enqueue(task, priority);
    print('เพิ่มงาน: ${task.name} (priority: $priority)');
  }
  
  Task? processNext() {
    if (_queue.isEmpty) return null;
    Task task = _queue.dequeue();
    _completed.add(task);
    print('ดำเนินการ: ${task.name}');
    return task;
  }
  
  void processAll() {
    print('\n=== ดำเนินการทุกงาน ===');
    while (_queue.isNotEmpty) {
      processNext();
    }
    print('เสร็จสิ้น! ดำเนินการ ${_completed.length} งาน');
  }
  
  int get pendingCount => _queue.size;
  int get completedCount => _completed.length;
}

void main() {
  print('=== Generic Stack ===\n');
  
  Stack<int> calcStack = Stack<int>();
  
  // Push numbers
  calcStack.push(5);
  calcStack.push(3);
  calcStack.push(8);
  print(calcStack);
  
  // Pop and compute
  int a = calcStack.pop();
  int b = calcStack.pop();
  calcStack.push(a + b); // 8 + 3 = 11
  print('หลังบวก: ${calcStack.peek()}');
  
  // ตรวจสอบ bracket matching ด้วย Stack
  bool isBalanced(String input) {
    Stack<String> stack = Stack<String>();
    Map<String, String> matching = {')': '(', ']': '[', '}': '{'};
    
    for (int i = 0; i < input.length; i++) {
      String char = input[i];
      if ('([{'.contains(char)) {
        stack.push(char);
      } else if (')]}'.contains(char)) {
        if (stack.isEmpty || stack.pop() != matching[char]) {
          return false;
        }
      }
    }
    return stack.isEmpty;
  }
  
  print('\n=== Bracket Matching ===');
  print('"([]{})" balanced: ${isBalanced("([]{})")}');       // true
  print('"([)]" balanced: ${isBalanced("([)]")}');           // false
  print('"(((" balanced: ${isBalanced("(((")}');             // false
  print('"Dart {Flutter [OK]}" balanced: ${isBalanced("Dart {Flutter [OK]}")}'); // true
  
  print('\n=== Generic Queue ===\n');
  
  Queue<String> printQueue = Queue<String>(maxSize: 5);
  printQueue.enqueue('เอกสาร 1');
  printQueue.enqueue('เอกสาร 2');
  printQueue.enqueue('เอกสาร 3');
  
  print('Queue: ${printQueue.toList()}');
  print('ประมวลผล: ${printQueue.dequeue()}');
  print('Queue หลัง dequeue: ${printQueue.toList()}');
  
  print('\n=== Undo/Redo ===\n');
  
  UndoRedoManager<String> editor = UndoRedoManager('เอกสารว่าง');
  
  editor.executeAction('สวัสดีครับ');
  editor.executeAction('สวัสดีครับ ยินดีต้อนรับ');
  editor.executeAction('สวัสดีครับ ยินดีต้อนรับ สู่ Dart');
  
  print('\nState ปัจจุบัน: ${editor.state}');
  print('canUndo: ${editor.canUndo}, canRedo: ${editor.canRedo}');
  
  editor.undo();
  editor.undo();
  print('State: ${editor.state}');
  
  editor.redo();
  print('State: ${editor.state}');
  
  editor.executeAction('แก้ไขใหม่');
  print('State: ${editor.state}');
  print('canRedo (ควร false): ${editor.canRedo}');
  
  print('\n=== Priority Queue ===\n');
  
  TaskManager taskManager = TaskManager();
  
  taskManager.addTask(Task(id: 'T1', name: 'อัปเดต UI', description: 'เปลี่ยนสี button'), priority: 3);
  taskManager.addTask(Task(id: 'T2', name: 'แก้ Bug ร้ายแรง', description: 'App crash บน iOS'), priority: 10);
  taskManager.addTask(Task(id: 'T3', name: 'เพิ่ม Feature', description: 'Dark mode'), priority: 5);
  taskManager.addTask(Task(id: 'T4', name: 'Security Patch', description: 'XSS vulnerability'), priority: 9);
  taskManager.addTask(Task(id: 'T5', name: 'Refactor Code', description: 'Clean up old code'), priority: 2);
  
  print('\nรอดำเนินการ: ${taskManager.pendingCount} งาน');
  taskManager.processAll();
  
  print('\n=== LRU Cache ===\n');
  
  LRUCache<String, String> cache = LRUCache<String, String>(3);
  
  cache.put('user:001', 'สมชาย ใจดี');
  cache.put('user:002', 'มานี รักเรียน');
  cache.put('user:003', 'วิชัย เก่งกาจ');
  print(cache); // size 3
  
  cache.get('user:001'); // Access 001 - ทำให้ 002 เป็น LRU
  cache.put('user:004', 'สุดา สวยงาม'); // Evict LRU (002)
  print(cache);
  
  print('user:002 ยังอยู่? ${cache.containsKey("user:002")}'); // false
  print('user:001 ยังอยู่? ${cache.containsKey("user:001")}'); // true
}
```

---

## 10.8 Generics ใน Flutter

```dart
// ตัวอย่างการใช้ Generics ใน Flutter patterns (conceptual)

/*
// AsyncSnapshot จาก Flutter - ตัวอย่าง generic class จริงๆ
// class AsyncSnapshot<T> {
//   final T? data;
//   final Object? error;
//   ...
// }

// FutureBuilder<T> - Generic Widget
// FutureBuilder<User>(
//   future: fetchUser(userId),
//   builder: (context, AsyncSnapshot<User> snapshot) {
//     if (snapshot.hasData) {
//       User user = snapshot.data!;
//       return Text(user.name);
//     }
//     return CircularProgressIndicator();
//   },
// )
*/

// Generic State Management Pattern (Pure Dart)
abstract class AppState<T> {
  const AppState();
  
  bool get isLoading => this is LoadingState;
  bool get isSuccess => this is SuccessState;
  bool get isError => this is ErrorState;
}

class LoadingState<T> extends AppState<T> {
  const LoadingState();
  
  @override
  String toString() => 'LoadingState<$T>';
}

class SuccessState<T> extends AppState<T> {
  final T data;
  const SuccessState(this.data);
  
  @override
  String toString() => 'SuccessState<$T>($data)';
}

class ErrorState<T> extends AppState<T> {
  final String message;
  final Exception? exception;
  const ErrorState(this.message, {this.exception});
  
  @override
  String toString() => 'ErrorState<$T>($message)';
}

// Generic ViewModel/Bloc Pattern
class AsyncViewModel<T> {
  AppState<T> _state = const LoadingState();
  final List<void Function(AppState)> _listeners = [];
  
  AppState<T> get state => _state;
  
  void addListener(void Function(AppState<T>) listener) {
    _listeners.add(listener as void Function(AppState));
  }
  
  void _setState(AppState<T> newState) {
    _state = newState;
    for (var listener in [..._listeners]) {
      listener(newState);
    }
  }
  
  Future<void> load(Future<T> Function() fetcher) async {
    _setState(LoadingState<T>());
    
    try {
      T data = await fetcher();
      _setState(SuccessState<T>(data));
    } catch (e) {
      _setState(ErrorState<T>(
        e.toString(),
        exception: e is Exception ? e : null,
      ));
    }
  }
}

// Generic API Response Wrapper
class ApiResponse<T> {
  final int statusCode;
  final T? data;
  final String? message;
  final List<String> errors;
  
  ApiResponse({
    required this.statusCode,
    this.data,
    this.message,
    this.errors = const [],
  });
  
  bool get isSuccess => statusCode >= 200 && statusCode < 300;
  bool get hasData => data != null;
  bool get hasErrors => errors.isNotEmpty;
  
  ApiResponse<R> map<R>(R Function(T) mapper) {
    return ApiResponse<R>(
      statusCode: statusCode,
      data: data != null ? mapper(data as T) : null,
      message: message,
      errors: errors,
    );
  }
  
  factory ApiResponse.success(T data, {String? message}) {
    return ApiResponse(statusCode: 200, data: data, message: message);
  }
  
  factory ApiResponse.error(int statusCode, {String? message, List<String> errors = const []}) {
    return ApiResponse(statusCode: statusCode, message: message, errors: errors);
  }
  
  @override
  String toString() {
    return isSuccess 
        ? 'ApiResponse.success($statusCode, data: $data)'
        : 'ApiResponse.error($statusCode, errors: $errors)';
  }
}

void main() async {
  print('=== Generic State Management ===\n');
  
  AsyncViewModel<String> vm = AsyncViewModel<String>();
  
  vm.addListener((state) {
    print('State: $state');
    if (state is SuccessState<String>) {
      print('  Data: ${state.data}');
    } else if (state is ErrorState<String>) {
      print('  Error: ${state.message}');
    }
  });
  
  // จำลอง load สำเร็จ
  await vm.load(() async {
    await Future.delayed(Duration(milliseconds: 100));
    return 'ข้อมูลที่ดึงมาสำเร็จ';
  });
  
  print('สถานะปัจจุบัน: ${vm.state}');
  print();
  
  // จำลอง load ล้มเหลว
  await vm.load(() async {
    await Future.delayed(Duration(milliseconds: 50));
    throw Exception('Network error: Connection timeout');
  });
  
  print('\n=== Generic ApiResponse ===\n');
  
  // สำเร็จ
  ApiResponse<Map<String, dynamic>> successResponse = ApiResponse.success({
    'id': 'U001',
    'name': 'สมชาย',
    'email': 'somchai@example.com',
  });
  
  print(successResponse);
  
  // แปลง data
  ApiResponse<String> nameOnly = successResponse.map((data) => data['name'] as String);
  print('Name only: ${nameOnly.data}');
  
  // Error
  ApiResponse<String> errorResponse = ApiResponse.error(
    404,
    message: 'ไม่พบข้อมูล',
    errors: ['User not found', 'Invalid ID format'],
  );
  
  print(errorResponse);
  print('Has errors: ${errorResponse.hasErrors}');
}
```

---

## สรุป Part 10

ใน Part นี้เราได้เรียนรู้:

### Generics ทำอะไรได้บ้าง:
| ฟีเจอร์ | ตัวอย่าง | ประโยชน์ |
|--------|---------|---------|
| **Generic Class** | `Box<T>`, `Stack<T>` | Reuse code หลาย types |
| **Generic Method** | `findFirst<T>()` | Flexible functions |
| **Type Constraint** | `T extends Comparable` | Safety + capabilities |
| **Multiple Types** | `Pair<A, B>` | Complex type relationships |

### Patterns ที่ใช้บ่อย:
- **Optional\<T\>** - แทน null ด้วย type-safe container
- **Result\<S, E\>** - Error handling แบบ functional
- **Repository\<T\>** - Generic data access layer
- **AsyncViewModel\<T\>** - Type-safe state management

### Covariance:
- Dart List เป็น covariant โดย default (แต่มี runtime check)
- ใช้ `covariant` keyword เมื่อ override parameter type
- Function parameters เป็น contravariant ตามหลัก type theory

## ➡️ Part ถัดไป

**Part 11: Async Programming** - เราจะเรียนรู้ Future, async/await, Stream และการจัดการ concurrency ใน Dart
