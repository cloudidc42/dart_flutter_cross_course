# Part 55: Performance Optimization

## บทนำ

Performance เป็นส่วนสำคัญของ UX ที่ดี Flutter มี tools และเทคนิคหลายอย่างช่วยให้แอปทำงานได้ smooth 60fps หรือ 120fps การเข้าใจว่า Flutter render อย่างไรช่วยให้เราเขียน code ที่มี performance ดีขึ้น

---

## 55.1 เข้าใจ Flutter Rendering

### Widget Tree, Element Tree, Render Tree

```
Widget Tree    →    Element Tree    →    Render Tree
(Blueprints)       (Lifecycle)          (Actual drawing)

Flutter rebuild Widget tree บ่อยมาก
แต่จะ rebuild Render Tree เฉพาะส่วนที่จำเป็น

setState() → rebuild widgets → diff → update render objects
```

### Frame Budget

```
60 FPS = 1 frame ทุก 16.67ms
120 FPS = 1 frame ทุก 8.33ms

ถ้า build() ใช้เวลานาน → jank (กระตุก)
```

---

## 55.2 const Widgets

### ประหยัด memory และเวลา rebuild

```dart
// ❌ สร้าง Widget ใหม่ทุกครั้งที่ build()
Widget build(BuildContext context) {
  return Column(
    children: [
      Text('ชื่อ:'),                          // rebuild ทุกครั้ง
      Icon(Icons.person, color: Colors.blue), // rebuild ทุกครั้ง
      SizedBox(height: 16),                   // rebuild ทุกครั้ง
    ],
  );
}

// ✅ const ทำให้ Widget เป็น singleton - ไม่ rebuild
Widget build(BuildContext context) {
  return const Column(
    children: [
      Text('ชื่อ:'),
      Icon(Icons.person, color: Colors.blue),
      SizedBox(height: 16),
    ],
  );
}

// ✅ ใช้ const สำหรับ static content
class StaticCard extends StatelessWidget {
  const StaticCard({super.key}); // const constructor
  
  @override
  Widget build(BuildContext context) {
    return const Card(
      child: Padding(
        padding: EdgeInsets.all(16),
        child: Column(
          children: [
            Icon(Icons.star, color: Colors.amber, size: 48),
            SizedBox(height: 8),
            Text(
              'Premium Member',
              style: TextStyle(
                fontWeight: FontWeight.bold,
                fontSize: 18,
              ),
            ),
          ],
        ),
      ),
    );
  }
}

// ✅ const เฉพาะส่วนที่ไม่เปลี่ยน
class UserCard extends StatelessWidget {
  final String userName;
  final int score;
  
  const UserCard({
    super.key,
    required this.userName,
    required this.score,
  });
  
  @override
  Widget build(BuildContext context) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16), // const ได้
        child: Column(
          children: [
            const Icon(Icons.person, size: 48), // const ได้
            Text(userName), // ไม่ const ได้
            Text('$score คะแนน'), // ไม่ const ได้
          ],
        ),
      ),
    );
  }
}
```

---

## 55.3 Widget Rebuild Optimization

### setState อย่างชาญฉลาด

```dart
// ❌ setState ทำให้ทั้งหน้า rebuild
class BadPage extends StatefulWidget {
  const BadPage({super.key});
  
  @override
  State<BadPage> createState() => _BadPageState();
}

class _BadPageState extends State<BadPage> {
  int _counter = 0;
  
  @override
  Widget build(BuildContext context) {
    print('BuildPage rebuild!'); // rebuild ทั้งหน้า
    return Scaffold(
      body: Column(
        children: [
          // Widget ที่ไม่เปลี่ยนก็ rebuild ไปด้วย
          const ExpensiveWidget(),
          const AnotherExpensiveWidget(),
          // เฉพาะ counter นี้ที่เปลี่ยน
          Text('Count: $_counter'),
          ElevatedButton(
            onPressed: () => setState(() => _counter++),
            child: const Text('+'),
          ),
        ],
      ),
    );
  }
}

// ✅ แยก stateful widget เฉพาะส่วนที่เปลี่ยน
class GoodPage extends StatelessWidget {
  const GoodPage({super.key});
  
  @override
  Widget build(BuildContext context) {
    return const Scaffold(
      body: Column(
        children: [
          ExpensiveWidget(),        // ไม่ rebuild
          AnotherExpensiveWidget(), // ไม่ rebuild
          CounterWidget(),          // rebuild เฉพาะ counter
        ],
      ),
    );
  }
}

class CounterWidget extends StatefulWidget {
  const CounterWidget({super.key});
  
  @override
  State<CounterWidget> createState() => _CounterWidgetState();
}

class _CounterWidgetState extends State<CounterWidget> {
  int _counter = 0;
  
  @override
  Widget build(BuildContext context) {
    print('CounterWidget rebuild!'); // rebuild เฉพาะส่วนนี้
    return Column(
      children: [
        Text('Count: $_counter'),
        ElevatedButton(
          onPressed: () => setState(() => _counter++),
          child: const Text('+'),
        ),
      ],
    );
  }
}
```

### ValueNotifier + ValueListenableBuilder

```dart
// ✅ ValueNotifier - rebuild เฉพาะ builder
class CounterWithValueNotifier extends StatelessWidget {
  final ValueNotifier<int> _counter = ValueNotifier(0);
  
  CounterWithValueNotifier({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        const Text('Static header'), // ไม่ rebuild เลย
        
        // Rebuild เฉพาะ ValueListenableBuilder
        ValueListenableBuilder<int>(
          valueListenable: _counter,
          builder: (context, value, child) {
            return Text('Count: $value');
          },
        ),
        
        ElevatedButton(
          onPressed: () => _counter.value++,
          child: const Text('+'),
        ),
      ],
    );
  }
}
```

### Selector จาก Provider

```dart
// ✅ Consumer/Selector rebuild เฉพาะส่วนที่ต้องการ
class OptimizedPage extends StatelessWidget {
  const OptimizedPage({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        // ดึงเฉพาะ title ไม่ rebuild เมื่อ counter เปลี่ยน
        title: Selector<AppState, String>(
          selector: (_, state) => state.pageTitle,
          builder: (_, title, __) => Text(title),
        ),
      ),
      body: Column(
        children: [
          // ดึงเฉพาะ counter ไม่ rebuild เมื่อ title เปลี่ยน
          Selector<AppState, int>(
            selector: (_, state) => state.counter,
            builder: (_, counter, __) => Text('Count: $counter'),
          ),
        ],
      ),
    );
  }
}
```

---

## 55.4 RepaintBoundary

```dart
// ใช้ RepaintBoundary เพื่อแยก repaint layer
class AnimatedBackground extends StatefulWidget {
  const AnimatedBackground({super.key});
  
  @override
  State<AnimatedBackground> createState() => _AnimatedBackgroundState();
}

class _AnimatedBackgroundState extends State<AnimatedBackground>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  
  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 3),
    )..repeat();
  }
  
  @override
  Widget build(BuildContext context) {
    return Stack(
      children: [
        // Animation ที่ repaint บ่อย - แยก layer
        RepaintBoundary(
          child: AnimatedBuilder(
            animation: _controller,
            builder: (context, _) => CustomPaint(
              painter: BackgroundPainter(progress: _controller.value),
              child: Container(),
            ),
          ),
        ),
        
        // Static content - ไม่ repaint เมื่อ animation เล่น
        const Column(
          children: [
            Text('หัวข้อหน้า', style: TextStyle(fontSize: 24)),
            SizedBox(height: 16),
            Text('เนื้อหาที่ไม่เปลี่ยนแปลง'),
          ],
        ),
      ],
    );
  }
  
  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}

// ใช้ RepaintBoundary กับ list items ที่ complex
class ComplexListItem extends StatelessWidget {
  final Item item;
  
  const ComplexListItem({super.key, required this.item});
  
  @override
  Widget build(BuildContext context) {
    return RepaintBoundary(
      child: Card(
        child: Row(
          children: [
            // Complex image
            CachedNetworkImage(
              imageUrl: item.imageUrl,
              width: 80,
              height: 80,
              fit: BoxFit.cover,
            ),
            Expanded(
              child: Padding(
                padding: const EdgeInsets.all(8),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(item.title),
                    Text(item.description),
                    Text('฿${item.price}'),
                  ],
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## 55.5 ListView.builder vs ListView

```dart
// ❌ ListView ธรรมดา - สร้าง Widget ทั้งหมดพร้อมกัน
class BadList extends StatelessWidget {
  final List<Item> items;
  
  const BadList({super.key, required this.items});
  
  @override
  Widget build(BuildContext context) {
    return ListView(
      // สร้าง Widget ทั้ง 1000 ตัวพร้อมกัน!
      children: items.map((item) => ItemWidget(item: item)).toList(),
    );
  }
}

// ✅ ListView.builder - สร้าง Widget เฉพาะที่แสดง (lazy)
class GoodList extends StatelessWidget {
  final List<Item> items;
  
  const GoodList({super.key, required this.items});
  
  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: items.length,
      // กำหนด height เพื่อให้ estimate position ได้ดีขึ้น
      itemExtent: 80, // Optional แต่ช่วย performance
      itemBuilder: (context, index) {
        return ItemWidget(item: items[index]);
      },
    );
  }
}

// ✅ ListView.separated - มี separator
class ListWithSeparator extends StatelessWidget {
  final List<Item> items;
  
  const ListWithSeparator({super.key, required this.items});
  
  @override
  Widget build(BuildContext context) {
    return ListView.separated(
      itemCount: items.length,
      separatorBuilder: (context, index) => const Divider(height: 1),
      itemBuilder: (context, index) => ItemWidget(item: items[index]),
    );
  }
}

// ✅ CustomScrollView สำหรับ complex layouts
class ComplexScrollView extends StatelessWidget {
  const ComplexScrollView({super.key});
  
  @override
  Widget build(BuildContext context) {
    return CustomScrollView(
      slivers: [
        // Header
        const SliverAppBar(
          expandedHeight: 200,
          pinned: true,
          flexibleSpace: FlexibleSpaceBar(
            title: Text('ร้านค้า'),
          ),
        ),
        
        // Category chips (horizontal)
        SliverToBoxAdapter(
          child: SizedBox(
            height: 50,
            child: ListView.builder(
              scrollDirection: Axis.horizontal,
              itemCount: 10,
              itemBuilder: (context, index) => Chip(
                label: Text('หมวด $index'),
              ),
            ),
          ),
        ),
        
        // Product grid
        SliverGrid(
          gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
            crossAxisCount: 2,
            mainAxisSpacing: 8,
            crossAxisSpacing: 8,
            childAspectRatio: 0.75,
          ),
          delegate: SliverChildBuilderDelegate(
            (context, index) => ProductCard(index: index),
            childCount: 100,
          ),
        ),
      ],
    );
  }
}
```

---

## 55.6 Image Caching

```dart
// ✅ ใช้ cached_network_image
// pubspec.yaml: cached_network_image: ^3.3.1

class OptimizedImage extends StatelessWidget {
  final String imageUrl;
  final double width;
  final double height;
  
  const OptimizedImage({
    super.key,
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
      fit: BoxFit.cover,
      // Placeholder
      placeholder: (context, url) => Container(
        color: Colors.grey[200],
        child: const Center(
          child: CircularProgressIndicator(strokeWidth: 2),
        ),
      ),
      // Error fallback
      errorWidget: (context, url, error) => Container(
        color: Colors.grey[200],
        child: const Icon(Icons.broken_image, color: Colors.grey),
      ),
      // Cache settings
      memCacheWidth: width.toInt() * 2, // 2x สำหรับ retina
      memCacheHeight: height.toInt() * 2,
      maxWidthDiskCache: 800,
      maxHeightDiskCache: 800,
    );
  }
}

// Preload images
class ImagePreloader {
  static Future<void> preloadImages(
    BuildContext context,
    List<String> imageUrls,
  ) async {
    await Future.wait(
      imageUrls.map(
        (url) => precacheImage(CachedNetworkImageProvider(url), context),
      ),
    );
  }
}
```

---

## 55.7 Compute Isolates

```dart
import 'dart:isolate';
import 'package:flutter/foundation.dart';

// ❌ Heavy computation บน main thread - blocks UI
void processDataOnMainThread(List<int> data) {
  // จะทำให้แอป freeze!
  final result = data
      .where((n) => _isPrime(n))
      .toList();
  print(result.length);
}

// ✅ ใช้ compute() สำหรับ background processing
Future<List<int>> findPrimesInBackground(List<int> numbers) async {
  // compute() รัน function ใน isolate แยก
  return await compute(_findPrimes, numbers);
}

// Function ต้องเป็น top-level หรือ static
List<int> _findPrimes(List<int> numbers) {
  return numbers.where((n) => _isPrime(n)).toList();
}

bool _isPrime(int n) {
  if (n < 2) return false;
  for (int i = 2; i <= sqrt(n); i++) {
    if (n % i == 0) return false;
  }
  return true;
}

// ใช้งานใน Widget
class PrimeCalculatorPage extends StatefulWidget {
  const PrimeCalculatorPage({super.key});
  
  @override
  State<PrimeCalculatorPage> createState() => _PrimeCalculatorPageState();
}

class _PrimeCalculatorPageState extends State<PrimeCalculatorPage> {
  List<int>? _primes;
  bool _isCalculating = false;
  
  Future<void> _calculate() async {
    setState(() => _isCalculating = true);
    
    final numbers = List.generate(100000, (i) => i + 2);
    final primes = await findPrimesInBackground(numbers);
    
    setState(() {
      _primes = primes;
      _isCalculating = false;
    });
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Prime Calculator')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            if (_isCalculating)
              const CircularProgressIndicator()
            else if (_primes != null)
              Text('พบ ${_primes!.length} จำนวนเฉพาะ')
            else
              const Text('กด Calculate เพื่อเริ่ม'),
            const SizedBox(height: 16),
            ElevatedButton(
              onPressed: _isCalculating ? null : _calculate,
              child: const Text('Calculate'),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## 55.8 Workshop: Optimize a Slow List

### Before (Unoptimized)

```dart
class SlowProductList extends StatefulWidget {
  const SlowProductList({super.key});
  
  @override
  State<SlowProductList> createState() => _SlowProductListState();
}

class _SlowProductListState extends State<SlowProductList> {
  List<Product> _products = [];
  bool _showFavorites = false;
  
  @override
  void initState() {
    super.initState();
    _loadProducts();
  }
  
  Future<void> _loadProducts() async {
    // Simulate API call
    await Future.delayed(const Duration(seconds: 1));
    setState(() {
      _products = List.generate(1000, (i) => Product.mock(i));
    });
  }
  
  @override
  Widget build(BuildContext context) {
    // ❌ แต่ละ build สร้าง filtered list ใหม่
    final displayed = _showFavorites
        ? _products.where((p) => p.isFavorite).toList()
        : _products;
    
    return Scaffold(
      appBar: AppBar(
        title: const Text('สินค้า'),
        // ❌ Rebuilds ทั้งหน้าเมื่อ toggle
        actions: [
          IconButton(
            icon: Icon(_showFavorites ? Icons.favorite : Icons.favorite_border),
            onPressed: () => setState(() => _showFavorites = !_showFavorites),
          ),
        ],
      ),
      // ❌ ListView ไม่ใช่ builder
      body: ListView(
        children: displayed.map((product) => ProductCard(
          product: product,
          // ❌ Inline callback สร้างใหม่ทุก build
          onFavorite: () {
            setState(() {
              product.isFavorite = !product.isFavorite;
            });
          },
        )).toList(),
      ),
    );
  }
}
```

### After (Optimized)

```dart
class FastProductList extends StatelessWidget {
  const FastProductList({super.key});
  
  @override
  Widget build(BuildContext context) {
    return const Scaffold(
      body: _ProductListBody(),
    );
  }
}

class _ProductListBody extends StatefulWidget {
  const _ProductListBody();
  
  @override
  State<_ProductListBody> createState() => _ProductListBodyState();
}

class _ProductListBodyState extends State<_ProductListBody> {
  List<Product> _products = [];
  bool _showFavorites = false;
  
  // Cache filtered result
  List<Product>? _cachedFiltered;
  bool? _lastShowFavorites;
  
  @override
  void initState() {
    super.initState();
    _loadProducts();
  }
  
  Future<void> _loadProducts() async {
    // Load ใน isolate
    final products = await compute(
      _generateProducts,
      1000,
    );
    
    if (mounted) {
      setState(() {
        _products = products;
        _invalidateCache();
      });
    }
  }
  
  static List<Product> _generateProducts(int count) {
    return List.generate(count, (i) => Product.mock(i));
  }
  
  void _invalidateCache() {
    _cachedFiltered = null;
    _lastShowFavorites = null;
  }
  
  List<Product> get _displayedProducts {
    // ใช้ cache ถ้า filter ไม่เปลี่ยน
    if (_cachedFiltered != null && _lastShowFavorites == _showFavorites) {
      return _cachedFiltered!;
    }
    
    _cachedFiltered = _showFavorites
        ? _products.where((p) => p.isFavorite).toList()
        : _products;
    _lastShowFavorites = _showFavorites;
    
    return _cachedFiltered!;
  }
  
  void _toggleFavorite(String productId) {
    final index = _products.indexWhere((p) => p.id == productId);
    if (index == -1) return;
    
    setState(() {
      _products[index] = _products[index].copyWith(
        isFavorite: !_products[index].isFavorite,
      );
      _invalidateCache();
    });
  }
  
  @override
  Widget build(BuildContext context) {
    final products = _displayedProducts;
    
    return CustomScrollView(
      slivers: [
        // ✅ SliverAppBar แยกออกจาก list
        SliverAppBar(
          title: const Text('สินค้า'),
          pinned: true,
          actions: [
            // ✅ แยก widget ที่ toggle เพื่อไม่ rebuild ทั้งหมด
            _FavoriteToggleButton(
              showFavorites: _showFavorites,
              onToggle: () => setState(() {
                _showFavorites = !_showFavorites;
                _invalidateCache();
              }),
            ),
          ],
        ),
        
        if (products.isEmpty)
          const SliverFillRemaining(
            child: Center(child: Text('ไม่มีสินค้า')),
          )
        else
          // ✅ SliverList แทน ListView
          SliverList(
            delegate: SliverChildBuilderDelegate(
              (context, index) {
                final product = products[index];
                // ✅ RepaintBoundary แยก layer
                return RepaintBoundary(
                  key: ValueKey(product.id),
                  child: _ProductListItem(
                    product: product,
                    onFavorite: _toggleFavorite,
                  ),
                );
              },
              childCount: products.length,
              // ✅ ให้ scroll estimate ที่ถูกต้อง
              addAutomaticKeepAlives: false,
              addRepaintBoundaries: true,
            ),
          ),
      ],
    );
  }
}

// ✅ แยก widget ที่ toggle
class _FavoriteToggleButton extends StatelessWidget {
  final bool showFavorites;
  final VoidCallback onToggle;
  
  const _FavoriteToggleButton({
    required this.showFavorites,
    required this.onToggle,
  });
  
  @override
  Widget build(BuildContext context) {
    return IconButton(
      icon: Icon(showFavorites ? Icons.favorite : Icons.favorite_border),
      onPressed: onToggle,
    );
  }
}

// ✅ ใช้ const constructor และ equatable
class _ProductListItem extends StatelessWidget {
  final Product product;
  final void Function(String) onFavorite;
  
  const _ProductListItem({
    required this.product,
    required this.onFavorite,
  });
  
  @override
  Widget build(BuildContext context) {
    return ListTile(
      // ✅ Cached image
      leading: CachedNetworkImage(
        imageUrl: product.imageUrl,
        width: 60,
        height: 60,
        fit: BoxFit.cover,
        memCacheWidth: 120,
        memCacheHeight: 120,
      ),
      title: Text(product.name),
      subtitle: Text('฿${product.price}'),
      trailing: IconButton(
        icon: Icon(
          product.isFavorite ? Icons.favorite : Icons.favorite_border,
          color: product.isFavorite ? Colors.red : null,
        ),
        onPressed: () => onFavorite(product.id),
      ),
    );
  }
}
```

---

## 55.9 DevTools Profiling

### เปิด Flutter DevTools

```bash
# รัน app ใน debug mode
flutter run

# เปิด DevTools
flutter pub global activate devtools
flutter pub global run devtools

# หรือใน VS Code: Cmd+Shift+P -> "Open DevTools"
```

### Performance Overlay

```dart
// แสดง performance overlay ใน debug mode
MaterialApp(
  showPerformanceOverlay: true, // แสดง GPU/CPU bars
  debugShowMaterialGrid: false,
  // ...
)
```

### Checklist สำหรับ Performance

```
Performance Checklist
├── ✅ ใช้ const widgets ทุกที่ที่เป็นไปได้
├── ✅ ใช้ ListView.builder แทน ListView
├── ✅ ใช้ RepaintBoundary สำหรับ animations
├── ✅ ใช้ cached_network_image
├── ✅ ย้าย heavy computation ไป isolate
├── ✅ หลีกเลี่ยง rebuild ที่ไม่จำเป็น
├── ✅ ใช้ Keys อย่างถูกต้อง
├── ✅ Profile ก่อน optimize (ไม่ guess)
└── ✅ ตรวจสอบด้วย DevTools
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- Widget rebuild optimization
- const widgets ประหยัด memory
- RepaintBoundary แยก repaint layers
- ListView.builder สำหรับ lazy loading
- Image caching ด้วย CachedNetworkImage
- Compute isolates สำหรับ heavy computation
- DevTools profiling
- Workshop: Optimize Slow List

**แบบฝึกหัดเพิ่มเติม:**
1. Profile แอปด้วย DevTools แล้วหา bottlenecks
2. Implement infinite scroll พร้อม pagination
3. เพิ่ม image compression ก่อน upload
4. ใช้ Flutter's Skia Shader warm-up ลด jank ครั้งแรก
