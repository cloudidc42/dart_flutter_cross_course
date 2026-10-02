# Part 06: Collections - List, Map, Set

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้งาน List, Map, และ Set ได้อย่างชำนาญ
- เรียกใช้ collection methods ต่างๆ เช่น map, where, reduce, fold ได้
- เข้าใจ immutable collections และเมื่อไหร่ควรใช้
- สร้าง patterns การจัดการข้อมูลด้วย collections
- สร้าง Contact Book โดยใช้ collections จริง

---

## 6.1 List - รายการข้อมูล

List คือโครงสร้างข้อมูลแบบ ordered ที่เก็บข้อมูลตามลำดับ และสามารถมีข้อมูลซ้ำกันได้

### การสร้าง List

```dart
void main() {
  // วิธีที่ 1: สร้างด้วย literal
  List<String> fruits = ['แอปเปิ้ล', 'กล้วย', 'ส้ม'];
  
  // วิธีที่ 2: สร้างด้วย constructor
  List<int> numbers = List<int>.empty(growable: true);
  
  // วิธีที่ 3: สร้าง List ที่มีขนาดคงที่
  List<String> fixedList = List<String>.filled(5, 'ว่าง');
  print(fixedList); // [ว่าง, ว่าง, ว่าง, ว่าง, ว่าง]
  
  // วิธีที่ 4: สร้างจาก Iterable
  List<int> generated = List<int>.generate(5, (index) => index * 2);
  print(generated); // [0, 2, 4, 6, 8]
  
  // วิธีที่ 5: var (type inference)
  var cities = ['กรุงเทพฯ', 'เชียงใหม่', 'ภูเก็ต', 'ขอนแก่น'];
  print(cities);
}
```

### การเข้าถึงข้อมูลใน List

```dart
void main() {
  List<String> provinces = ['กรุงเทพฯ', 'เชียงใหม่', 'ภูเก็ต', 'ขอนแก่น', 'สงขลา'];
  
  // เข้าถึงด้วย index (เริ่มที่ 0)
  print(provinces[0]);         // กรุงเทพฯ
  print(provinces.first);      // กรุงเทพฯ
  print(provinces.last);       // สงขลา
  
  // ขนาดของ List
  print(provinces.length);     // 5
  
  // ตรวจสอบว่าว่างหรือไม่
  print(provinces.isEmpty);    // false
  print(provinces.isNotEmpty); // true
  
  // การ index แบบ sublist
  List<String> centralRegion = provinces.sublist(0, 2);
  print(centralRegion); // [กรุงเทพฯ, เชียงใหม่]
  
  // ค้นหา index ของข้อมูล
  int index = provinces.indexOf('ภูเก็ต');
  print(index); // 2
  
  // ตรวจสอบว่ามีข้อมูลอยู่หรือไม่
  print(provinces.contains('ภูเก็ต')); // true
  print(provinces.contains('นครราชสีมา')); // false
}
```

---

## 6.2 List Methods - การเพิ่ม/ลบข้อมูล

```dart
void main() {
  List<String> shoppingList = ['ข้าว', 'น้ำ'];
  
  // === การเพิ่มข้อมูล ===
  
  // เพิ่มที่ท้าย
  shoppingList.add('นม');
  print(shoppingList); // [ข้าว, น้ำ, นม]
  
  // เพิ่มหลายรายการ
  shoppingList.addAll(['ไข่', 'ผัก', 'เนื้อ']);
  print(shoppingList); // [ข้าว, น้ำ, นม, ไข่, ผัก, เนื้อ]
  
  // แทรกที่ตำแหน่งที่กำหนด
  shoppingList.insert(1, 'น้ำตาล');
  print(shoppingList); // [ข้าว, น้ำตาล, น้ำ, นม, ไข่, ผัก, เนื้อ]
  
  // แทรกหลายรายการ
  shoppingList.insertAll(0, ['ซอสถั่วเหลือง', 'น้ำปลา']);
  print(shoppingList);
  
  // === การลบข้อมูล ===
  
  // ลบด้วยค่า (ลบตัวแรกที่พบ)
  shoppingList.remove('น้ำ');
  
  // ลบด้วย index
  shoppingList.removeAt(0);
  
  // ลบตัวสุดท้าย
  shoppingList.removeLast();
  
  // ลบช่วง
  shoppingList.removeRange(0, 2);
  
  // ลบตามเงื่อนไข
  shoppingList.removeWhere((item) => item.length > 3);
  
  // ล้างทั้งหมด
  // shoppingList.clear();
  
  print('รายการซื้อของที่เหลือ: $shoppingList');
}
```

### การแก้ไขข้อมูล

```dart
void main() {
  List<String> menu = ['ข้าวผัด', 'ต้มยำ', 'ผัดไทย', 'ข้าวมันไก่'];
  
  // แทนที่ด้วย index
  menu[1] = 'ต้มยำกุ้ง';
  print(menu); // [ข้าวผัด, ต้มยำกุ้ง, ผัดไทย, ข้าวมันไก่]
  
  // fillRange - เติมช่วงด้วยค่าเดียว
  menu.fillRange(2, 4, 'ของวันนี้หมดแล้ว');
  print(menu); // [ข้าวผัด, ต้มยำกุ้ง, ของวันนี้หมดแล้ว, ของวันนี้หมดแล้ว]
  
  // replaceRange - แทนที่ช่วงด้วย Iterable ใหม่
  List<String> newMenu = ['ข้าวผัด', 'ต้มยำกุ้ง', 'ผัดซีอิ๊ว', 'ราดหน้า'];
  newMenu.replaceRange(2, 4, ['แกงเขียวหวาน', 'พะแนง']);
  print(newMenu); // [ข้าวผัด, ต้มยำกุ้ง, แกงเขียวหวาน, พะแนง]
  
  // setRange - กำหนดค่าในช่วง
  List<int> scores = [0, 0, 0, 0, 0];
  scores.setRange(1, 4, [85, 90, 78]);
  print(scores); // [0, 85, 90, 78, 0]
}
```

---

## 6.3 List Functional Methods

นี่คือส่วนที่ทรงพลังที่สุดของ List - functional programming style

### map() - แปลงข้อมูลทุกตัว

```dart
void main() {
  List<int> prices = [100, 200, 350, 500, 750];
  
  // เพิ่ม VAT 7%
  List<double> pricesWithVat = prices.map((price) => price * 1.07).toList();
  print(pricesWithVat); // [107.0, 214.0, 374.5, 535.0, 802.5]
  
  // แปลงเป็น String
  List<String> priceStrings = prices.map((p) => '฿$p').toList();
  print(priceStrings); // [฿100, ฿200, ฿350, ฿500, ฿750]
  
  // แปลง object
  List<Map<String, dynamic>> products = [
    {'name': 'กาแฟ', 'price': 60},
    {'name': 'ชา', 'price': 45},
    {'name': 'น้ำส้ม', 'price': 55},
  ];
  
  List<String> productNames = products.map((p) => p['name'] as String).toList();
  print(productNames); // [กาแฟ, ชา, น้ำส้ม]
  
  // Method chaining
  List<String> expensiveProducts = products
      .where((p) => (p['price'] as int) > 50)
      .map((p) => '${p['name']} - ฿${p['price']}')
      .toList();
  print(expensiveProducts); // [กาแฟ - ฿60, น้ำส้ม - ฿55]
}
```

### where() - กรองข้อมูล

```dart
void main() {
  List<int> ages = [15, 22, 17, 30, 16, 25, 18, 14];
  
  // กรองผู้ใหญ่ (18+)
  List<int> adults = ages.where((age) => age >= 18).toList();
  print(adults); // [22, 30, 25, 18]
  
  // กรองเด็ก (< 18)
  List<int> minors = ages.where((age) => age < 18).toList();
  print(minors); // [15, 17, 16, 14]
  
  // ตัวอย่างกับ String
  List<String> names = ['สมชาย', 'สมหญิง', 'วิชัย', 'สมศรี', 'ณัฐพล', 'สมาน'];
  
  // กรองชื่อที่ขึ้นต้นด้วย 'สม'
  List<String> somNames = names.where((name) => name.startsWith('สม')).toList();
  print(somNames); // [สมชาย, สมหญิง, สมศรี, สมาน]
  
  // กรองชื่อที่มีความยาวมากกว่า 3 ตัวอักษร
  List<String> longNames = names.where((name) => name.length > 3).toList();
  print(longNames);
  
  // ใช้ whereType() สำหรับกรอง type
  List<dynamic> mixedList = [1, 'hello', 2.5, 'world', 42, true];
  List<String> strings = mixedList.whereType<String>().toList();
  print(strings); // [hello, world]
  
  List<int> integers = mixedList.whereType<int>().toList();
  print(integers); // [1, 42]
}
```

### reduce() และ fold() - รวมข้อมูล

```dart
void main() {
  List<int> scores = [85, 92, 78, 96, 88, 74, 90];
  
  // reduce - รวมค่าทั้งหมด (ต้องมีอย่างน้อย 1 element)
  int total = scores.reduce((sum, score) => sum + score);
  print('คะแนนรวม: $total'); // คะแนนรวม: 603
  
  // หาค่าสูงสุด
  int maxScore = scores.reduce((max, score) => score > max ? score : max);
  print('คะแนนสูงสุด: $maxScore'); // คะแนนสูงสุด: 96
  
  // หาค่าต่ำสุด
  int minScore = scores.reduce((min, score) => score < min ? score : min);
  print('คะแนนต่ำสุด: $minScore'); // คะแนนต่ำสุด: 74
  
  // fold - เหมือน reduce แต่มี initial value (ปลอดภัยกว่าสำหรับ empty list)
  int sum = scores.fold(0, (acc, score) => acc + score);
  print('ผลรวม: $sum'); // ผลรวม: 603
  
  double average = scores.fold(0, (acc, score) => acc + score) / scores.length;
  print('คะแนนเฉลี่ย: ${average.toStringAsFixed(2)}'); // คะแนนเฉลี่ย: 86.14
  
  // fold กับ empty list (ปลอดภัย)
  List<int> emptyList = [];
  int safeSum = emptyList.fold(0, (acc, val) => acc + val);
  print('ผลรวม empty list: $safeSum'); // ผลรวม empty list: 0
  
  // fold เพื่อสร้าง Map
  List<String> fruits = ['แอปเปิ้ล', 'กล้วย', 'แอปเปิ้ล', 'ส้ม', 'กล้วย', 'กล้วย'];
  Map<String, int> fruitCount = fruits.fold({}, (map, fruit) {
    map[fruit] = (map[fruit] ?? 0) + 1;
    return map;
  });
  print(fruitCount); // {แอปเปิ้ล: 2, กล้วย: 3, ส้ม: 1}
}
```

### expand() - แผ่ข้อมูล nested

```dart
void main() {
  // expand() แผ่ nested lists ให้เป็น flat list
  List<List<int>> nested = [[1, 2, 3], [4, 5], [6, 7, 8, 9]];
  List<int> flat = nested.expand((list) => list).toList();
  print(flat); // [1, 2, 3, 4, 5, 6, 7, 8, 9]
  
  // ตัวอย่างจริง: แผ่รายการสินค้าจากหลายหมวด
  List<Map<String, dynamic>> categories = [
    {'name': 'ผลไม้', 'items': ['แอปเปิ้ล', 'กล้วย', 'ส้ม']},
    {'name': 'ผัก', 'items': ['แครอท', 'บร็อคโคลี่']},
    {'name': 'เนื้อสัตว์', 'items': ['ไก่', 'หมู', 'เนื้อ', 'ปลา']},
  ];
  
  List<String> allItems = categories
      .expand((cat) => (cat['items'] as List<String>))
      .toList();
  print(allItems);
  // [แอปเปิ้ล, กล้วย, ส้ม, แครอท, บร็อคโคลี่, ไก่, หมู, เนื้อ, ปลา]
  
  // สร้าง range จาก ranges
  List<int> numbers = [1, 2, 3, 4, 5];
  List<int> repeated = numbers.expand((n) => List.filled(n, n)).toList();
  print(repeated); // [1, 2, 2, 3, 3, 3, 4, 4, 4, 4, 5, 5, 5, 5, 5]
}
```

### sort() - การเรียงลำดับ

```dart
void main() {
  // เรียงตัวเลข
  List<int> numbers = [42, 17, 86, 3, 55, 29];
  numbers.sort(); // ascending by default
  print(numbers); // [3, 17, 29, 42, 55, 86]
  
  // เรียงจากมากไปน้อย
  numbers.sort((a, b) => b.compareTo(a));
  print(numbers); // [86, 55, 42, 29, 17, 3]
  
  // เรียง String
  List<String> names = ['สมชาย', 'อรัญ', 'กิตติ', 'ณัฐ', 'ประยุทธ'];
  names.sort();
  print(names); // เรียงตาม Unicode
  
  // เรียง object ซับซ้อน
  List<Map<String, dynamic>> students = [
    {'name': 'สมชาย', 'score': 85},
    {'name': 'มานี', 'score': 92},
    {'name': 'วิชัย', 'score': 78},
    {'name': 'สุดา', 'score': 96},
    {'name': 'ประยุทธ', 'score': 88},
  ];
  
  // เรียงตามคะแนนจากมากไปน้อย
  students.sort((a, b) => (b['score'] as int).compareTo(a['score'] as int));
  print('จัดอันดับตามคะแนน:');
  for (int i = 0; i < students.length; i++) {
    print('  อันดับ ${i+1}: ${students[i]['name']} - ${students[i]['score']} คะแนน');
  }
  
  // เรียงตามชื่อ
  students.sort((a, b) => (a['name'] as String).compareTo(b['name'] as String));
  
  // ใช้ sorted (ไม่แก้ไข list เดิม) - ต้อง import package:collection
  // หรือทำด้วย spread operator
  List<Map<String, dynamic>> sortedStudents = [...students]
    ..sort((a, b) => (a['score'] as int).compareTo(b['score'] as int));
}
```

---

## 6.4 List Utility Methods

```dart
void main() {
  List<int> numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
  
  // any() - ตรวจสอบว่ามีสมาชิกที่ตรงเงื่อนไขหรือไม่
  bool hasEven = numbers.any((n) => n % 2 == 0);
  print('มีเลขคู่: $hasEven'); // true
  
  bool hasNegative = numbers.any((n) => n < 0);
  print('มีเลขลบ: $hasNegative'); // false
  
  // every() - ตรวจสอบว่าทุกสมาชิกตรงเงื่อนไขหรือไม่
  bool allPositive = numbers.every((n) => n > 0);
  print('ทุกตัวเป็นบวก: $allPositive'); // true
  
  bool allEven = numbers.every((n) => n % 2 == 0);
  print('ทุกตัวเป็นเลขคู่: $allEven'); // false
  
  // take() - เอาแค่ N ตัวแรก
  List<int> firstFive = numbers.take(5).toList();
  print('5 ตัวแรก: $firstFive'); // [1, 2, 3, 4, 5]
  
  // skip() - ข้าม N ตัวแรก
  List<int> skipThree = numbers.skip(3).toList();
  print('ข้าม 3 ตัวแรก: $skipThree'); // [4, 5, 6, 7, 8, 9, 10]
  
  // takeWhile() - เอาตราบเท่าที่เงื่อนไขเป็นจริง
  List<int> lessThanSix = numbers.takeWhile((n) => n < 6).toList();
  print('น้อยกว่า 6: $lessThanSix'); // [1, 2, 3, 4, 5]
  
  // skipWhile() - ข้ามตราบเท่าที่เงื่อนไขเป็นจริง
  List<int> afterFive = numbers.skipWhile((n) => n <= 5).toList();
  print('หลังจาก 5: $afterFive'); // [6, 7, 8, 9, 10]
  
  // firstWhere() - หาตัวแรกที่ตรงเงื่อนไข
  int firstEven = numbers.firstWhere((n) => n % 2 == 0);
  print('เลขคู่ตัวแรก: $firstEven'); // 2
  
  // lastWhere() - หาตัวสุดท้ายที่ตรงเงื่อนไข
  int lastEven = numbers.lastWhere((n) => n % 2 == 0);
  print('เลขคู่ตัวสุดท้าย: $lastEven'); // 10
  
  // singleWhere() - หาตัวเดียวที่ตรงเงื่อนไข (error ถ้าพบมากกว่า 1)
  int seven = numbers.singleWhere((n) => n == 7);
  print('เลข 7: $seven');
  
  // indexWhere() - หา index ของตัวที่ตรงเงื่อนไข
  int index = numbers.indexWhere((n) => n > 7);
  print('index แรกที่มากกว่า 7: $index'); // 7 (ค่า 8 อยู่ที่ index 7)
  
  // join() - รวมเป็น String
  String joined = numbers.join(', ');
  print('รวมกัน: $joined'); // 1, 2, 3, 4, 5, 6, 7, 8, 9, 10
  
  // toSet() - แปลงเป็น Set (ลบซ้ำ)
  List<int> withDups = [1, 2, 2, 3, 3, 3, 4];
  Set<int> unique = withDups.toSet();
  print('ไม่ซ้ำ: $unique'); // {1, 2, 3, 4}
  
  // reversed - กลับลำดับ
  List<int> reversed = numbers.reversed.toList();
  print('กลับลำดับ: $reversed');
}
```

---

## 6.5 Map - คู่ข้อมูล Key-Value

Map เก็บข้อมูลเป็นคู่ key-value โดย key ต้องไม่ซ้ำกัน

### การสร้างและเข้าถึง Map

```dart
void main() {
  // การสร้าง Map
  Map<String, int> scores = {
    'สมชาย': 85,
    'มานี': 92,
    'วิชัย': 78,
    'สุดา': 96,
  };
  
  // เข้าถึงด้วย key
  print(scores['สมชาย']); // 85
  print(scores['ไม่มี']); // null (ไม่ error)
  
  // เพิ่มหรือแก้ไข
  scores['ประยุทธ'] = 88;  // เพิ่มใหม่
  scores['สมชาย'] = 90;   // แก้ไขของเดิม
  
  // ตรวจสอบ
  print(scores.containsKey('มานี'));    // true
  print(scores.containsValue(92));     // true
  print(scores.length);                // 5
  print(scores.isEmpty);               // false
  
  // ดู keys และ values
  print(scores.keys.toList());    // [สมชาย, มานี, วิชัย, สุดา, ประยุทธ]
  print(scores.values.toList()); // [90, 92, 78, 96, 88]
  
  // ลบ
  scores.remove('วิชัย');
  print(scores);
  
  // ล้างทั้งหมด
  // scores.clear();
}
```

### Map Methods ที่ทรงพลัง

```dart
void main() {
  Map<String, int> inventory = {
    'ข้าว': 50,
    'น้ำมัน': 20,
    'น้ำตาล': 30,
    'เกลือ': 15,
  };
  
  // putIfAbsent - เพิ่มเฉพาะถ้า key ยังไม่มี
  inventory.putIfAbsent('พริก', () => 25);
  print(inventory['พริก']); // 25
  
  inventory.putIfAbsent('ข้าว', () => 999); // ไม่แก้ไข เพราะมีอยู่แล้ว
  print(inventory['ข้าว']); // 50 (ยังเป็น 50)
  
  // update - อัปเดตค่าที่มีอยู่
  inventory.update('ข้าว', (value) => value + 10);
  print(inventory['ข้าว']); // 60
  
  // update พร้อม ifAbsent - เพิ่มถ้าไม่มี key นั้น
  inventory.update('ซอส', (value) => value + 5, ifAbsent: () => 5);
  print(inventory['ซอส']); // 5
  
  // updateAll - อัปเดตทุก value
  inventory.updateAll((key, value) => value * 2);
  print(inventory);
  
  // map - แปลงทั้ง key และ value
  Map<String, String> inventoryStatus = inventory.map(
    (key, value) => MapEntry(key, value > 50 ? 'มาก' : 'น้อย'),
  );
  print(inventoryStatus);
  
  // forEach
  print('\nรายการสินค้าคงคลัง:');
  inventory.forEach((product, qty) {
    print('  $product: $qty ชิ้น');
  });
  
  // entries - IterableMapEntry
  for (MapEntry<String, int> entry in inventory.entries) {
    print('${entry.key} => ${entry.value}');
  }
}
```

### การกรองและแปลง Map

```dart
void main() {
  Map<String, double> productPrices = {
    'กาแฟ': 60.0,
    'ชา': 45.0,
    'น้ำส้ม': 55.0,
    'น้ำเปล่า': 15.0,
    'โกโก้': 65.0,
    'ชามะนาว': 50.0,
  };
  
  // กรองด้วย where (ใช้ entries)
  Map<String, double> expensiveItems = Map.fromEntries(
    productPrices.entries.where((entry) => entry.value >= 50),
  );
  print('สินค้าราคา 50+ บาท:');
  expensiveItems.forEach((name, price) => print('  $name: ฿$price'));
  
  // แปลงราคาเพิ่ม VAT 7%
  Map<String, double> pricesWithVat = productPrices.map(
    (name, price) => MapEntry(name, price * 1.07),
  );
  
  // รวมสองMap
  Map<String, double> morePrices = {
    'มะพร้าว': 40.0,
    'แตงโม': 35.0,
  };
  
  // วิธีที่ 1: addAll
  Map<String, double> allPrices = {...productPrices};
  allPrices.addAll(morePrices);
  
  // วิธีที่ 2: spread operator
  Map<String, double> combined = {
    ...productPrices,
    ...morePrices,
  };
  
  print('\nทั้งหมด: ${combined.length} รายการ');
  
  // สร้าง Map จาก List
  List<String> items = ['แอปเปิ้ล', 'กล้วย', 'ส้ม'];
  List<int> qtys = [10, 25, 15];
  
  Map<String, int> fruitInventory = Map.fromIterables(items, qtys);
  print(fruitInventory); // {แอปเปิ้ล: 10, กล้วย: 25, ส้ม: 15}
  
  // สร้างจาก List ด้วย fold
  List<Map<String, dynamic>> products = [
    {'name': 'ข้าวผัด', 'price': 80},
    {'name': 'ต้มยำ', 'price': 120},
    {'name': 'ผัดไทย', 'price': 90},
  ];
  
  Map<String, int> priceMap = products.fold({}, (map, product) {
    map[product['name'] as String] = product['price'] as int;
    return map;
  });
  print(priceMap);
}
```

---

## 6.6 Set - ชุดข้อมูลไม่ซ้ำ

Set เก็บข้อมูลที่ไม่ซ้ำกัน และไม่มีลำดับที่แน่นอน

### การสร้างและใช้งาน Set

```dart
void main() {
  // สร้าง Set
  Set<String> uniqueTags = {'dart', 'flutter', 'mobile', 'cross-platform'};
  
  // เพิ่มข้อมูล
  uniqueTags.add('ios');
  uniqueTags.add('android');
  uniqueTags.add('dart'); // ไม่เพิ่ม เพราะมีอยู่แล้ว
  
  print(uniqueTags);
  print('จำนวน tags: ${uniqueTags.length}'); // ไม่รวม 'dart' ซ้ำ
  
  // ตรวจสอบ
  print(uniqueTags.contains('flutter')); // true
  print(uniqueTags.contains('web'));     // false
  
  // ลบ
  uniqueTags.remove('ios');
  
  // เพิ่มหลายรายการ
  uniqueTags.addAll(['web', 'desktop', 'embedded']);
  
  // แปลงระหว่าง List และ Set
  List<String> tagList = uniqueTags.toList();
  Set<String> tagSet = tagList.toSet(); // ลบซ้ำ
  
  // สร้าง Set จาก List ที่มีซ้ำ
  List<int> withDups = [1, 2, 2, 3, 3, 3, 4, 4, 4, 4];
  Set<int> unique = Set.from(withDups);
  print('ไม่ซ้ำ: $unique'); // {1, 2, 3, 4}
  
  // กลับเป็น List ที่ไม่มีซ้ำ
  List<int> uniqueList = withDups.toSet().toList();
  print('List ไม่ซ้ำ: $uniqueList');
}
```

### Set Operations - Union, Intersection, Difference

```dart
void main() {
  Set<String> mathStudents = {'สมชาย', 'มานี', 'วิชัย', 'สุดา', 'ประยุทธ'};
  Set<String> scienceStudents = {'มานี', 'สุดา', 'กิตติ', 'นิตยา', 'สมชาย'};
  
  // Union - รวมทั้งสอง Set (ไม่ซ้ำ)
  Set<String> allStudents = mathStudents.union(scienceStudents);
  print('นักเรียนทั้งหมด: $allStudents');
  // {สมชาย, มานี, วิชัย, สุดา, ประยุทธ, กิตติ, นิตยา}
  
  // Intersection - ข้อมูลที่อยู่ในทั้งสอง Set
  Set<String> bothSubjects = mathStudents.intersection(scienceStudents);
  print('เรียนทั้งสองวิชา: $bothSubjects');
  // {สมชาย, มานี, สุดา}
  
  // Difference - ข้อมูลที่อยู่ใน Set แรกแต่ไม่อยู่ใน Set ที่สอง
  Set<String> mathOnly = mathStudents.difference(scienceStudents);
  print('เรียนแค่คณิตฯ: $mathOnly');
  // {วิชัย, ประยุทธ}
  
  Set<String> scienceOnly = scienceStudents.difference(mathStudents);
  print('เรียนแค่วิทยาศาสตร์: $scienceOnly');
  // {กิตติ, นิตยา}
  
  // containsAll - ตรวจสอบว่ามีทุก element หรือไม่
  print(mathStudents.containsAll({'สมชาย', 'มานี'})); // true
  print(mathStudents.containsAll({'สมชาย', 'กิตติ'})); // false
  
  // ตัวอย่างการใช้จริง: หาสิ่งที่ต้องซื้อเพิ่ม
  Set<String> haveIngredients = {'แป้ง', 'น้ำตาล', 'เนย', 'วานิลลา'};
  Set<String> needIngredients = {'แป้ง', 'น้ำตาล', 'ไข่', 'นม', 'เนย'};
  
  Set<String> needToBuy = needIngredients.difference(haveIngredients);
  print('ต้องซื้อ: $needToBuy'); // {ไข่, นม}
}
```

---

## 6.7 Immutable Collections

```dart
void main() {
  // const List - ไม่สามารถแก้ไขได้เลย
  const List<String> weekdays = ['จันทร์', 'อังคาร', 'พุธ', 'พฤหัส', 'ศุกร์', 'เสาร์', 'อาทิตย์'];
  
  // weekdays.add('วันพิเศษ'); // Error! ไม่สามารถแก้ไข const
  print(weekdays);
  
  // const Map
  const Map<String, String> countryCode = {
    'ไทย': 'TH',
    'ญี่ปุ่น': 'JP',
    'สหรัฐฯ': 'US',
    'อังกฤษ': 'GB',
  };
  
  // countryCode['เกาหลี'] = 'KR'; // Error!
  
  // const Set
  const Set<int> primeNumbers = {2, 3, 5, 7, 11, 13, 17, 19, 23, 29};
  
  // List.unmodifiable - สร้างจาก List ที่มีอยู่
  List<String> mutableList = ['a', 'b', 'c'];
  List<String> immutableList = List.unmodifiable(mutableList);
  
  // immutableList.add('d'); // Error! UnsupportedError
  print(immutableList);
  
  // Map.unmodifiable
  Map<String, int> mutableMap = {'one': 1, 'two': 2};
  Map<String, int> immutableMap = Map.unmodifiable(mutableMap);
  
  // immutableMap['three'] = 3; // Error!
  
  // Set.unmodifiable
  Set<String> mutableSet = {'a', 'b', 'c'};
  Set<String> immutableSet = Set.unmodifiable(mutableSet);
  
  // เมื่อไหร่ควรใช้ immutable collections?
  // 1. ข้อมูลคงที่ที่ไม่ควรเปลี่ยน (config, constants)
  // 2. ส่งผ่าน function โดยไม่ต้องการให้แก้ไข
  // 3. เพื่อ thread safety
  // 4. เพื่อ performance (Dart สามารถ optimize ได้)
}
```

### Spread Operator และ Collection If/For

```dart
void main() {
  // Spread operator
  List<int> list1 = [1, 2, 3];
  List<int> list2 = [4, 5, 6];
  List<int> combined = [...list1, ...list2, 7, 8];
  print(combined); // [1, 2, 3, 4, 5, 6, 7, 8]
  
  // Null-aware spread
  List<int>? maybeNull = null;
  List<int> safe = [...list1, ...?maybeNull, ...list2];
  print(safe); // [1, 2, 3, 4, 5, 6]
  
  // Collection if
  bool isAdmin = true;
  List<String> menuItems = [
    'หน้าแรก',
    'สินค้า',
    'เกี่ยวกับเรา',
    if (isAdmin) 'จัดการระบบ',
    if (isAdmin) 'รายงาน',
    'ติดต่อ',
  ];
  print(menuItems);
  
  // Collection for
  List<int> indices = [1, 2, 3, 4, 5];
  List<String> items = [
    for (int i in indices) 'สินค้าที่ $i',
  ];
  print(items);
  
  // ผสม if และ for
  List<int> allNumbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
  List<String> evenLabels = [
    for (int n in allNumbers)
      if (n % 2 == 0) 'เลขคู่: $n',
  ];
  print(evenLabels);
  
  // Map spread
  Map<String, int> map1 = {'a': 1, 'b': 2};
  Map<String, int> map2 = {'c': 3, 'd': 4};
  Map<String, int> mergedMap = {...map1, ...map2};
  print(mergedMap);
}
```

---

## 6.8 Collection Patterns ที่ใช้บ่อย

### Pattern 1: Grouping

```dart
void main() {
  List<Map<String, dynamic>> orders = [
    {'id': 1, 'customer': 'สมชาย', 'status': 'pending', 'amount': 250},
    {'id': 2, 'customer': 'มานี', 'status': 'completed', 'amount': 450},
    {'id': 3, 'customer': 'วิชัย', 'status': 'pending', 'amount': 180},
    {'id': 4, 'customer': 'สุดา', 'status': 'cancelled', 'amount': 320},
    {'id': 5, 'customer': 'ประยุทธ', 'status': 'completed', 'amount': 560},
    {'id': 6, 'customer': 'กิตติ', 'status': 'pending', 'amount': 120},
  ];
  
  // จัดกลุ่มตาม status
  Map<String, List<Map<String, dynamic>>> grouped = {};
  for (var order in orders) {
    String status = order['status'] as String;
    grouped.putIfAbsent(status, () => []);
    grouped[status]!.add(order);
  }
  
  print('ออเดอร์รอดำเนินการ: ${grouped['pending']?.length} รายการ');
  print('ออเดอร์เสร็จสิ้น: ${grouped['completed']?.length} รายการ');
  print('ออเดอร์ยกเลิก: ${grouped['cancelled']?.length} รายการ');
  
  // คำนวณยอดรวมแต่ละกลุ่ม
  grouped.forEach((status, orderList) {
    int total = orderList.fold(0, (sum, o) => sum + (o['amount'] as int));
    print('$status: ฿$total');
  });
}
```

### Pattern 2: Pagination

```dart
void main() {
  List<String> allProducts = List.generate(100, (i) => 'สินค้า ${i + 1}');
  
  int pageSize = 10;
  int currentPage = 3; // หน้าที่ 3
  
  List<String> getPage(List<String> items, int page, int size) {
    int start = (page - 1) * size;
    int end = start + size;
    if (start >= items.length) return [];
    return items.sublist(start, end.clamp(0, items.length));
  }
  
  List<String> page3 = getPage(allProducts, currentPage, pageSize);
  print('หน้า $currentPage: $page3');
  // [สินค้า 21, สินค้า 22, ..., สินค้า 30]
  
  int totalPages = (allProducts.length / pageSize).ceil();
  print('ทั้งหมด: $totalPages หน้า');
}
```

### Pattern 3: Frequency Count

```dart
void main() {
  List<String> votes = [
    'สมชาย', 'มานี', 'สมชาย', 'วิชัย', 'มานี', 'สมชาย',
    'สุดา', 'มานี', 'วิชัย', 'สมชาย', 'กิตติ', 'มานี',
  ];
  
  // นับความถี่
  Map<String, int> frequency = {};
  for (String vote in votes) {
    frequency[vote] = (frequency[vote] ?? 0) + 1;
  }
  
  // เรียงตามคะแนน
  List<MapEntry<String, int>> sorted = frequency.entries.toList()
    ..sort((a, b) => b.value.compareTo(a.value));
  
  print('ผลการโหวต:');
  for (int i = 0; i < sorted.length; i++) {
    print('  อันดับ ${i+1}: ${sorted[i].key} - ${sorted[i].value} คะแนน');
  }
}
```

### Pattern 4: Unique Values และ Duplicates

```dart
void main() {
  List<String> emails = [
    'user@example.com',
    'admin@example.com',
    'user@example.com',
    'test@test.com',
    'admin@example.com',
    'new@example.com',
  ];
  
  // หา duplicates
  Set<String> seen = {};
  List<String> duplicates = [];
  
  for (String email in emails) {
    if (!seen.add(email)) {
      duplicates.add(email);
    }
  }
  
  print('Email ซ้ำ: $duplicates');
  
  // ลบซ้ำและรักษาลำดับ
  List<String> unique = emails.toSet().toList();
  print('Email ไม่ซ้ำ: $unique');
  
  // รักษาลำดับดั้งเดิม
  List<String> uniqueOrdered = [];
  Set<String> addedSet = {};
  for (String email in emails) {
    if (addedSet.add(email)) {
      uniqueOrdered.add(email);
    }
  }
  print('Email ไม่ซ้ำ (รักษาลำดับ): $uniqueOrdered');
}
```

---

## 6.9 Workshop: สมุดรายชื่อผู้ติดต่อ (Contact Book)

```dart
// contact_book.dart
// สมุดรายชื่อผู้ติดต่อใช้ Collections อย่างเต็มที่

class Contact {
  final String id;
  String name;
  String phone;
  String email;
  List<String> tags;
  bool isFavorite;
  
  Contact({
    required this.id,
    required this.name,
    required this.phone,
    required this.email,
    List<String>? tags,
    this.isFavorite = false,
  }) : tags = tags ?? [];
  
  // แสดงผลสวยงาม
  @override
  String toString() {
    return 'Contact(id: $id, name: $name, phone: $phone, email: $email, tags: $tags, favorite: $isFavorite)';
  }
  
  // สร้าง copy with changes
  Contact copyWith({
    String? name,
    String? phone,
    String? email,
    List<String>? tags,
    bool? isFavorite,
  }) {
    return Contact(
      id: id,
      name: name ?? this.name,
      phone: phone ?? this.phone,
      email: email ?? this.email,
      tags: tags ?? this.tags,
      isFavorite: isFavorite ?? this.isFavorite,
    );
  }
}

class ContactBook {
  // เก็บ contacts โดยใช้ Map<id, Contact> เพื่อ O(1) lookup
  final Map<String, Contact> _contacts = {};
  
  // Track tags ทั้งหมดด้วย Set
  final Set<String> _allTags = {};
  
  int _nextId = 1;
  
  // สร้าง ID ใหม่
  String _generateId() => 'C${_nextId++}'.padLeft(5, '0');
  
  // === เพิ่ม/แก้ไข/ลบ ===
  
  Contact addContact({
    required String name,
    required String phone,
    required String email,
    List<String>? tags,
  }) {
    String id = _generateId();
    Contact contact = Contact(
      id: id,
      name: name,
      phone: phone,
      email: email,
      tags: tags ?? [],
    );
    
    _contacts[id] = contact;
    _allTags.addAll(contact.tags);
    
    print('เพิ่มผู้ติดต่อ: ${contact.name} (ID: ${contact.id})');
    return contact;
  }
  
  bool updateContact(String id, {
    String? name,
    String? phone,
    String? email,
    List<String>? tags,
  }) {
    if (!_contacts.containsKey(id)) {
      print('ไม่พบผู้ติดต่อ ID: $id');
      return false;
    }
    
    Contact updated = _contacts[id]!.copyWith(
      name: name,
      phone: phone,
      email: email,
      tags: tags,
    );
    
    _contacts[id] = updated;
    
    // อัปเดต tags
    if (tags != null) {
      _allTags.addAll(tags);
    }
    
    return true;
  }
  
  bool deleteContact(String id) {
    if (_contacts.remove(id) != null) {
      print('ลบผู้ติดต่อ ID: $id แล้ว');
      return true;
    }
    return false;
  }
  
  void toggleFavorite(String id) {
    if (_contacts.containsKey(id)) {
      Contact contact = _contacts[id]!;
      _contacts[id] = contact.copyWith(isFavorite: !contact.isFavorite);
    }
  }
  
  // === ค้นหา ===
  
  Contact? findById(String id) => _contacts[id];
  
  List<Contact> searchByName(String query) {
    String lowerQuery = query.toLowerCase();
    return _contacts.values
        .where((c) => c.name.toLowerCase().contains(lowerQuery))
        .toList();
  }
  
  List<Contact> searchByPhone(String phone) {
    return _contacts.values
        .where((c) => c.phone.contains(phone))
        .toList();
  }
  
  List<Contact> searchByTag(String tag) {
    return _contacts.values
        .where((c) => c.tags.contains(tag))
        .toList();
  }
  
  // === การแสดงผล ===
  
  List<Contact> getAll({String? sortBy}) {
    List<Contact> contacts = _contacts.values.toList();
    
    switch (sortBy) {
      case 'name':
        contacts.sort((a, b) => a.name.compareTo(b.name));
        break;
      case 'id':
        contacts.sort((a, b) => a.id.compareTo(b.id));
        break;
    }
    
    return contacts;
  }
  
  List<Contact> getFavorites() {
    return _contacts.values.where((c) => c.isFavorite).toList()
      ..sort((a, b) => a.name.compareTo(b.name));
  }
  
  // === สถิติ ===
  
  Map<String, int> getTagStats() {
    Map<String, int> stats = {};
    for (Contact contact in _contacts.values) {
      for (String tag in contact.tags) {
        stats[tag] = (stats[tag] ?? 0) + 1;
      }
    }
    // เรียงตามความถี่
    return Map.fromEntries(
      stats.entries.toList()..sort((a, b) => b.value.compareTo(a.value))
    );
  }
  
  Map<String, List<Contact>> groupByFirstLetter() {
    Map<String, List<Contact>> grouped = {};
    for (Contact contact in _contacts.values) {
      String letter = contact.name[0].toUpperCase();
      grouped.putIfAbsent(letter, () => []);
      grouped[letter]!.add(contact);
    }
    // เรียงตามตัวอักษร
    return Map.fromEntries(
      grouped.entries.toList()..sort((a, b) => a.key.compareTo(b.key))
    );
  }
  
  // === แสดงผล ===
  
  void printAll() {
    print('\n=== สมุดรายชื่อผู้ติดต่อทั้งหมด (${_contacts.length} คน) ===');
    List<Contact> sorted = getAll(sortBy: 'name');
    for (Contact c in sorted) {
      String favIcon = c.isFavorite ? '⭐ ' : '   ';
      String tagsStr = c.tags.isEmpty ? '' : ' [${c.tags.join(', ')}]';
      print('$favIcon${c.name}$tagsStr');
      print('     📞 ${c.phone} | ✉️  ${c.email}');
    }
  }
  
  void printStats() {
    print('\n=== สถิติ ===');
    print('ผู้ติดต่อทั้งหมด: ${_contacts.length} คน');
    print('รายการโปรด: ${getFavorites().length} คน');
    print('Tags ทั้งหมด: ${_allTags.length} tags');
    
    print('\nTop Tags:');
    getTagStats().entries.take(5).forEach((entry) {
      print('  ${entry.key}: ${entry.value} คน');
    });
  }
  
  int get length => _contacts.length;
}

void main() {
  print('=== ระบบสมุดรายชื่อผู้ติดต่อ ===\n');
  
  ContactBook book = ContactBook();
  
  // เพิ่มผู้ติดต่อ
  Contact c1 = book.addContact(
    name: 'สมชาย ใจดี',
    phone: '081-234-5678',
    email: 'somchai@example.com',
    tags: ['ครอบครัว', 'เพื่อน'],
  );
  
  Contact c2 = book.addContact(
    name: 'มานี รักเรียน',
    phone: '082-345-6789',
    email: 'manee@example.com',
    tags: ['เพื่อน', 'เพื่อนร่วมงาน'],
  );
  
  book.addContact(
    name: 'วิชัย เก่งกาจ',
    phone: '083-456-7890',
    email: 'wichai@example.com',
    tags: ['เพื่อนร่วมงาน'],
  );
  
  book.addContact(
    name: 'สุดา สวยงาม',
    phone: '084-567-8901',
    email: 'suda@example.com',
    tags: ['ครอบครัว'],
  );
  
  book.addContact(
    name: 'ประยุทธ ขยันทำ',
    phone: '085-678-9012',
    email: 'prayuth@example.com',
    tags: ['เพื่อนร่วมงาน', 'เพื่อน'],
  );
  
  book.addContact(
    name: 'กิตติ นามสกุล',
    phone: '086-789-0123',
    email: 'kitti@example.com',
    tags: ['เพื่อน'],
  );
  
  // แสดงรายการทั้งหมด
  book.printAll();
  
  // ตั้งรายการโปรด
  book.toggleFavorite(c1.id);
  book.toggleFavorite(c2.id);
  
  // ค้นหา
  print('\n=== ค้นหา "สม" ===');
  List<Contact> results = book.searchByName('สม');
  for (Contact c in results) {
    print('  ${c.name} - ${c.phone}');
  }
  
  print('\n=== ค้นหาด้วย Tag "เพื่อน" ===');
  List<Contact> friendContacts = book.searchByTag('เพื่อน');
  for (Contact c in friendContacts) {
    print('  ${c.name}');
  }
  
  print('\n=== รายการโปรด ===');
  for (Contact c in book.getFavorites()) {
    print('  ⭐ ${c.name}');
  }
  
  print('\n=== จัดกลุ่มตามตัวอักษร ===');
  Map<String, List<Contact>> byLetter = book.groupByFirstLetter();
  byLetter.forEach((letter, contacts) {
    print('[$letter]');
    for (Contact c in contacts) {
      print('  ${c.name}');
    }
  });
  
  // แสดงสถิติ
  book.printStats();
  
  // อัปเดตข้อมูล
  print('\n=== อัปเดตข้อมูล ===');
  book.updateContact(c1.id, phone: '091-111-2222', tags: ['ครอบครัว', 'เพื่อน', 'VIP']);
  
  Contact? updated = book.findById(c1.id);
  print('อัปเดตแล้ว: ${updated?.name} - ${updated?.phone} - ${updated?.tags}');
  
  // ลบ
  print('\n=== ลบผู้ติดต่อ ===');
  book.deleteContact(c2.id);
  print('ผู้ติดต่อคงเหลือ: ${book.length} คน');
}
```

---

## 6.10 LinkedList และ Queue (จาก dart:collection)

```dart
import 'dart:collection';

void main() {
  // Queue - FIFO (First In, First Out)
  Queue<String> queue = Queue<String>();
  
  // เพิ่มที่ท้าย
  queue.add('งานแรก');
  queue.add('งานสอง');
  queue.add('งานสาม');
  
  // เพิ่มที่หน้า
  queue.addFirst('งานด่วน');
  
  print('Queue: $queue');
  
  // ดึงออกจากหน้า
  String firstJob = queue.removeFirst();
  print('ดำเนินการ: $firstJob');
  print('Queue ที่เหลือ: $queue');
  
  // ดู element แรกโดยไม่ลบ
  print('งานถัดไป: ${queue.first}');
  
  // HashMap - Map ที่เน้น performance
  HashMap<String, int> fastMap = HashMap<String, int>();
  fastMap['key1'] = 100;
  fastMap['key2'] = 200;
  print(fastMap);
  
  // LinkedHashMap - Map ที่รักษาลำดับการแทรก
  LinkedHashMap<String, int> orderedMap = LinkedHashMap<String, int>();
  orderedMap['ที่สาม'] = 3;
  orderedMap['ที่หนึ่ง'] = 1;
  orderedMap['ที่สอง'] = 2;
  print(orderedMap); // รักษาลำดับ: ที่สาม, ที่หนึ่ง, ที่สอง
  
  // HashSet - Set ที่เน้น performance
  HashSet<String> fastSet = HashSet<String>();
  fastSet.add('a');
  fastSet.add('b');
  fastSet.add('a'); // ไม่เพิ่ม
  print(fastSet);
  
  // SplayTreeMap - Map ที่เรียงลำดับ key
  SplayTreeMap<int, String> sortedMap = SplayTreeMap<int, String>();
  sortedMap[5] = 'five';
  sortedMap[1] = 'one';
  sortedMap[3] = 'three';
  sortedMap[2] = 'two';
  sortedMap[4] = 'four';
  print(sortedMap); // {1: one, 2: two, 3: three, 4: four, 5: five}
}
```

---

## 6.11 Performance Tips

```dart
void main() {
  // ✅ ใช้ toList() เฉพาะเมื่อจำเป็น
  // Lazy evaluation - ดีกว่าสำหรับข้อมูลมาก
  Iterable<int> lazyResult = [1, 2, 3, 4, 5]
      .where((n) => n > 2)
      .map((n) => n * 2);
  // ยังไม่คำนวณ!
  
  int first = lazyResult.first; // คำนวณแค่ตัวแรก
  print(first); // 6
  
  // ✅ ใช้ any() แทน where().isNotEmpty
  List<int> numbers = [1, 2, 3, 4, 5];
  
  // ❌ ช้า - scan ทั้ง list
  bool slow = numbers.where((n) => n > 3).isNotEmpty;
  
  // ✅ เร็ว - หยุดเมื่อพบตัวแรก
  bool fast = numbers.any((n) => n > 3);
  
  // ✅ ใช้ Set สำหรับ lookup บ่อยๆ
  List<String> bannedUsers = ['user1', 'user2', 'user3'];
  Set<String> bannedSet = bannedUsers.toSet(); // convert ครั้งเดียว
  
  // O(n) - ช้า
  bool isBannedSlow = bannedUsers.contains('user2');
  
  // O(1) - เร็ว
  bool isBannedFast = bannedSet.contains('user2');
  
  // ✅ ใช้ Map<id, Object> สำหรับ lookup ด้วย id
  // ✅ Pre-size list เมื่อรู้ขนาดล่วงหน้า
  List<int> presized = List<int>.filled(1000, 0);
  
  // ✅ growable: false สำหรับ list ที่ไม่เปลี่ยนขนาด
  List<String> fixed = List<String>.filled(5, 'item', growable: false);
}
```

---

## สรุป Part 06

ใน Part นี้เราได้เรียนรู้:

| Collection | ลักษณะเด่น | ใช้เมื่อ |
|------------|-----------|---------|
| **List** | Ordered, ซ้ำได้ | เก็บลำดับ, index สำคัญ |
| **Map** | Key-Value pairs, key unique | lookup ด้วย key |
| **Set** | Unordered, ไม่ซ้ำ | ตรวจสอบการมีอยู่, union/intersection |

### Functional Methods สำคัญ:
- **map()** - แปลงข้อมูล
- **where()** - กรองข้อมูล  
- **reduce()/fold()** - รวมข้อมูล
- **expand()** - แผ่ข้อมูล nested
- **sort()** - เรียงลำดับ

### Best Practices:
1. ใช้ `const` สำหรับ collections ที่ไม่เปลี่ยน
2. ใช้ Set สำหรับ lookup ที่ต้องการ performance
3. ใช้ Lazy evaluation (ไม่ต้อง `.toList()` ทุกครั้ง)
4. ใช้ `any()` แทน `where().isNotEmpty`

## ➡️ Part ถัดไป

**Part 07: OOP - Classes and Objects** - เราจะเรียนรู้การสร้าง classes, constructors, encapsulation และ patterns OOP ใน Dart อย่างละเอียด
