# Part 05: Functions และ Parameters

## 🎯 เป้าหมายของ Part นี้
- Function declaration ทุกรูปแบบ
- Required, Optional, Named parameters
- Default values
- Arrow functions (=>)
- Higher-order functions
- Closures และ Lexical scope
- Recursion

---

## 1. Function Basics

```dart
// ============ รูปแบบพื้นฐาน ============
// returnType functionName(parameters) { body }

// Function ที่คืนค่า:
int add(int a, int b) {
  return a + b;
}

// Function ที่ไม่คืนค่า:
void greet(String name) {
  print('สวัสดี $name!');
}

// Function ที่คืนค่า nullable:
String? findUser(int id) {
  if (id == 1) return 'Alice';
  return null; // ต้องคืนได้ null เพราะ return type เป็น String?
}

// Arrow function (สำหรับ single expression):
int multiply(int a, int b) => a * b;
void sayHello(String name) => print('Hello $name!');

void main() {
  print(add(3, 4));        // 7
  greet('Dart');           // สวัสดี Dart!
  print(findUser(1));      // Alice
  print(findUser(99));     // null
  print(multiply(5, 6));   // 30
  sayHello('Flutter');     // Hello Flutter!
  
  // ============ Function เป็น first-class citizen ============
  // สามารถ assign ให้ variable ได้:
  var addFunction = add;
  print(addFunction(10, 20)); // 30
  
  // Function type:
  int Function(int, int) operation = multiply;
  print(operation(4, 5)); // 20
}
```

---

## 2. Parameters Types

```dart
// ============ Required Positional Parameters ============
void displayInfo(String name, int age, String city) {
  print('$name, $age, $city');
}

// ============ Optional Positional Parameters ============
// ใช้ [] ครอบ
void greetOptional(String name, [String? title, String greeting = 'สวัสดี']) {
  String display = title != null ? '$title $name' : name;
  print('$greeting $display!');
}

// ============ Named Parameters ============
// ใช้ {} ครอบ
void createUser({
  required String name,  // required - บังคับส่ง
  required int age,      // required
  String? email,         // optional - อาจไม่ส่ง
  String role = 'user',  // optional with default
}) {
  print('User: $name, Age: $age, Email: ${email ?? "none"}, Role: $role');
}

void main() {
  // Required positional - ต้องส่งตามลำดับ:
  displayInfo('Alice', 30, 'Bangkok');
  
  // Optional positional:
  greetOptional('Bob');              // สวัสดี Bob!
  greetOptional('Bob', 'คุณ');      // สวัสดี คุณ Bob!
  greetOptional('Bob', 'Dr.', 'ยินดีต้อนรับ'); // ยินดีต้อนรับ Dr. Bob!
  
  // Named parameters - ส่งด้วยชื่อ ลำดับไม่สำคัญ:
  createUser(name: 'Charlie', age: 25);
  createUser(
    age: 30,
    name: 'Dave',
    email: 'dave@example.com',
    role: 'admin',
  );
  
  // ============ Mixing parameters ============
  // positional ก่อน optional ก่อน named (ห้ามผสม positional กับ named)
  void mixed(String required, [String optional = 'def']) {
    print('$required, $optional');
  }
  mixed('hello');          // hello, def
  mixed('hello', 'world'); // hello, world
}
```

---

## 3. Higher-Order Functions

```dart
void main() {
  // ============ Function เป็น parameter ============
  List<int> numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
  
  // Function ที่รับ function:
  List<int> filter(List<int> list, bool Function(int) predicate) {
    return list.where(predicate).toList();
  }
  
  List<int> evens = filter(numbers, (n) => n % 2 == 0);
  List<int> odds = filter(numbers, (n) => n % 2 != 0);
  List<int> bigNums = filter(numbers, (n) => n > 5);
  
  print(evens);   // [2, 4, 6, 8, 10]
  print(odds);    // [1, 3, 5, 7, 9]
  print(bigNums); // [6, 7, 8, 9, 10]
  
  // ============ Function คืน Function ============
  Function(int) makeMultiplier(int factor) {
    return (int n) => n * factor; // closure
  }
  
  var double_ = makeMultiplier(2);
  var triple = makeMultiplier(3);
  
  print(double_(5));  // 10
  print(triple(5));   // 15
  
  // ============ Callbacks ============
  void processData(List<int> data, void Function(int) callback) {
    for (int item in data) {
      callback(item);
    }
  }
  
  processData(numbers, (n) => print('Processing: $n'));
  
  // ============ Function composition ============
  int Function(int) compose(
    int Function(int) f,
    int Function(int) g,
  ) {
    return (int x) => f(g(x));
  }
  
  int addOne(int x) => x + 1;
  int square(int x) => x * x;
  
  var squareThenAdd = compose(addOne, square);
  var addThenSquare = compose(square, addOne);
  
  print(squareThenAdd(5)); // (5^2) + 1 = 26
  print(addThenSquare(5)); // (5+1)^2 = 36
}
```

---

## 4. Closures

```dart
void main() {
  // ============ Closure พื้นฐาน ============
  // Closure คือ function ที่ "จดจำ" environment ที่สร้างมัน
  
  Function makeCounter() {
    int count = 0; // variable ที่ closure จดจำ
    
    return () {
      count++;
      return count;
    };
  }
  
  var counter1 = makeCounter();
  var counter2 = makeCounter(); // independent state
  
  print(counter1()); // 1
  print(counter1()); // 2
  print(counter1()); // 3
  print(counter2()); // 1 (independent!)
  print(counter2()); // 2
  
  // ============ Closure ใน loops ============
  List<Function> funcs = [];
  
  // ⚠️ WRONG - ทุก function ใช้ i เดียวกัน:
  for (int i = 0; i < 3; i++) {
    funcs.add(() => print(i)); // i จะเป็น 3 ทุกตัว (ใน JS)
    // ใน Dart, loop variable ถูก capture correctly!
  }
  
  for (var f in funcs) {
    f(); // Dart: 0, 1, 2 ✅
  }
  
  // ============ Memoization ด้วย closure ============
  Function(int) memoize(int Function(int) fn) {
    Map<int, int> cache = {};
    
    return (int n) {
      if (cache.containsKey(n)) {
        print('Cache hit: $n');
        return cache[n]!;
      }
      int result = fn(n);
      cache[n] = result;
      return result;
    };
  }
  
  int expensiveCalc(int n) {
    // จำลองการคำนวณที่ช้า
    return n * n;
  }
  
  var memoized = memoize(expensiveCalc);
  print(memoized(5));  // คำนวณ: 25
  print(memoized(5));  // Cache hit: 25
  print(memoized(10)); // คำนวณ: 100
  print(memoized(10)); // Cache hit: 100
  
  // ============ Partial application ============
  int Function(int) partial(int Function(int, int) fn, int first) {
    return (int second) => fn(first, second);
  }
  
  int add(int a, int b) => a + b;
  var add5 = partial(add, 5);
  
  print(add5(3));  // 8
  print(add5(10)); // 15
}
```

---

## 5. Recursion

```dart
void main() {
  // ============ Factorial ============
  int factorial(int n) {
    if (n <= 1) return 1;      // base case
    return n * factorial(n - 1); // recursive case
  }
  
  print(factorial(5));  // 120
  print(factorial(10)); // 3628800
  
  // ============ Fibonacci ============
  int fibonacci(int n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
  }
  
  // ⚠️ ช้า! O(2^n) - ใช้ memoization แก้:
  Map<int, int> memo = {};
  int fibMemo(int n) {
    if (n <= 1) return n;
    return memo.putIfAbsent(n, () => fibMemo(n - 1) + fibMemo(n - 2));
  }
  
  for (int i = 0; i <= 10; i++) {
    print('fib($i) = ${fibMemo(i)}');
  }
  
  // ============ Tail recursion (Dart ไม่ optimize แต่ควรรู้) ============
  int factTail(int n, [int acc = 1]) {
    if (n <= 1) return acc;
    return factTail(n - 1, n * acc);
  }
  
  print(factTail(5)); // 120
  
  // ============ Tree traversal ============
  class TreeNode {
    int value;
    TreeNode? left;
    TreeNode? right;
    TreeNode(this.value, {this.left, this.right});
  }
  
  // สร้าง tree:
  //       1
  //      / \
  //     2   3
  //    / \
  //   4   5
  
  TreeNode tree = TreeNode(1,
    left: TreeNode(2,
      left: TreeNode(4),
      right: TreeNode(5),
    ),
    right: TreeNode(3),
  );
  
  void inOrder(TreeNode? node) {
    if (node == null) return;
    inOrder(node.left);
    print(node.value);
    inOrder(node.right);
  }
  
  print('In-order traversal:');
  inOrder(tree); // 4, 2, 5, 1, 3
  
  // ============ Recursive List operations ============
  int sumList(List<int> list) {
    if (list.isEmpty) return 0;
    return list.first + sumList(list.sublist(1));
  }
  
  print(sumList([1, 2, 3, 4, 5])); // 15
  
  // ============ Flatten nested list ============
  List flatten(List nested) {
    return nested.fold([], (List acc, element) {
      if (element is List) {
        return [...acc, ...flatten(element)];
      }
      return [...acc, element];
    });
  }
  
  print(flatten([1, [2, 3], [4, [5, 6]]])); // [1, 2, 3, 4, 5, 6]
}
```

---

## 6. Anonymous Functions และ Lambda

```dart
void main() {
  // ============ Anonymous function ============
  var printItem = (String item) {
    print('Item: $item');
  };
  
  printItem('Apple'); // Item: Apple
  
  // ============ Lambda (arrow) ============
  var square = (int n) => n * n;
  print(square(5)); // 25
  
  // ============ Immediately Invoked Function ============
  var result = () {
    int x = 10;
    int y = 20;
    return x + y;
  }(); // เรียกทันที
  
  print(result); // 30
  
  // ============ Function as map values ============
  Map<String, Function(double, double)> operations = {
    'บวก': (a, b) => a + b,
    'ลบ': (a, b) => a - b,
    'คูณ': (a, b) => a * b,
    'หาร': (a, b) => b != 0 ? a / b : double.infinity,
  };
  
  double a = 10, b = 3;
  operations.forEach((op, fn) {
    print('$a $op $b = ${fn(a, b)}');
  });
  
  // ============ Tear-offs ============
  // ใช้ method โดยตรงโดยไม่ต้อง wrap ใน lambda
  
  List<String> names = ['charlie', 'alice', 'bob'];
  
  // แบบ lambda:
  names.sort((a, b) => a.compareTo(b));
  
  // แบบ tear-off:
  names.sort(Comparable.compare); // เหมือนกัน แต่สั้นกว่า
  
  print(names); // [alice, bob, charlie]
  
  // Print ทุก element:
  names.forEach(print); // alice, bob, charlie (ไม่ต้อง (name) => print(name))
  
  // ============ typedef ============
  // สร้าง alias สำหรับ function type
  typedef Predicate<T> = bool Function(T);
  typedef Transform<T, R> = R Function(T);
  
  Predicate<int> isEven = (n) => n % 2 == 0;
  Transform<int, String> toStr = (n) => 'Number: $n';
  
  print(isEven(4));    // true
  print(toStr(42));    // Number: 42
  
  // ใช้ใน function signature:
  List<T> myFilter<T>(List<T> list, Predicate<T> pred) {
    return list.where(pred).toList();
  }
  
  print(myFilter([1, 2, 3, 4, 5], isEven)); // [2, 4]
}
```

---

## 7. Workshop: Functional Pipeline

```dart
// functional_pipeline.dart - Data processing pipeline

typedef Transformer<T> = T Function(T);
typedef Processor<T, R> = R Function(T);

class Pipeline<T> {
  final List<Transformer<T>> _transforms = [];
  
  Pipeline<T> pipe(Transformer<T> transform) {
    _transforms.add(transform);
    return this;
  }
  
  T process(T input) {
    return _transforms.fold(input, (acc, transform) => transform(acc));
  }
  
  List<T> processAll(List<T> inputs) {
    return inputs.map(process).toList();
  }
}

// String transformers
String trim(String s) => s.trim();
String toLowerCase(String s) => s.toLowerCase();
String capitalize(String s) {
  if (s.isEmpty) return s;
  return s[0].toUpperCase() + s.substring(1);
}
String Function(String, String) replace(String from, String to) {
  return (String s) => s.replaceAll(from, to);
}

// Number transformers
int Function(int) multiplyBy(int factor) => (int n) => n * factor;
int Function(int) addN(int n) => (int x) => x + n;
int clampTo100(int n) => n.clamp(0, 100);

void main() {
  // String pipeline:
  var nameProcessor = Pipeline<String>()
    .pipe(trim)
    .pipe(toLowerCase)
    .pipe(capitalize)
    .pipe(replace('_', ' '));
  
  List<String> dirtyNames = [
    '  alice  ',
    'BOB',
    'charlie_brown',
    '  DIANA PRINCE  ',
  ];
  
  print('=== Processed Names ===');
  for (String name in dirtyNames) {
    print('${name.trim()} => ${nameProcessor.process(name)}');
  }
  
  // Number pipeline:
  var scoreProcessor = Pipeline<int>()
    .pipe(multiplyBy(2))    // double the score
    .pipe(addN(10))          // add bonus
    .pipe(clampTo100);       // cap at 100
  
  List<int> rawScores = [30, 45, 55, 40, 48];
  List<int> processedScores = scoreProcessor.processAll(rawScores);
  
  print('\n=== Processed Scores ===');
  for (int i = 0; i < rawScores.length; i++) {
    print('${rawScores[i]} => ${processedScores[i]}');
  }
  
  // Higher-order composition:
  int sumOfSquares(List<int> numbers) {
    return numbers
        .map((n) => n * n)
        .reduce((a, b) => a + b);
  }
  
  double average(List<int> numbers) {
    if (numbers.isEmpty) return 0;
    return numbers.reduce((a, b) => a + b) / numbers.length;
  }
  
  List<int> data = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
  print('\nSum of squares: ${sumOfSquares(data)}');
  print('Average: ${average(data)}');
  
  // สร้าง custom aggregator:
  Map<String, dynamic> stats(List<int> nums) {
    if (nums.isEmpty) return {'min': null, 'max': null, 'avg': null};
    
    nums.sort();
    return {
      'min': nums.first,
      'max': nums.last,
      'sum': nums.reduce((a, b) => a + b),
      'avg': nums.reduce((a, b) => a + b) / nums.length,
      'count': nums.length,
    };
  }
  
  var result = stats(data);
  print('\nStats: $result');
}
```

---

## 8. สรุป Part 05

สิ่งที่เรียนรู้:
- ✅ Function declaration ทุกรูปแบบ
- ✅ Required, Optional positional, Named parameters
- ✅ Default parameter values
- ✅ Arrow functions (=>)
- ✅ Higher-order functions - รับและคืน functions
- ✅ Closures และ lexical scoping
- ✅ Recursion และ memoization
- ✅ Anonymous functions และ tear-offs
- ✅ typedef สำหรับ function types
- ✅ Functional programming patterns

---

## ➡️ Part ถัดไป
**Part 06: Collections - List, Map, Set**

เราจะเรียนรู้:
- List ทุก method อย่างละเอียด
- Map และการจัดการข้อมูล key-value
- Set และ Set operations
- Iterable methods ขั้นสูง
- Immutable collections
