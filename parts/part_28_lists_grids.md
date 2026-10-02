# Part 28: Lists and Grids

## บทนำ

Lists และ Grids เป็น widget ที่ใช้บ่อยมากใน Flutter สำหรับแสดงข้อมูลจำนวนมาก Flutter มี ListView และ GridView ที่มีประสิทธิภาพสูง

---

## ListView

### ListView พื้นฐาน

```dart
// ListView ธรรมดา - เหมาะกับ list ที่มีจำนวนน้อย
ListView(
  children: const [
    ListTile(title: Text('Item 1')),
    ListTile(title: Text('Item 2')),
    ListTile(title: Text('Item 3')),
    Divider(),
    ListTile(title: Text('Item 4')),
  ],
)
```

### ListView.builder

```dart
// ListView.builder - เหมาะกับ list ขนาดใหญ่ (lazy loading)
ListView.builder(
  itemCount: items.length,
  itemBuilder: (context, index) {
    return ListTile(
      key: ValueKey(items[index].id),
      title: Text(items[index].name),
    );
  },
)

// ListView.builder พร้อม padding
ListView.builder(
  padding: const EdgeInsets.all(8),
  itemCount: items.length,
  itemBuilder: (context, index) => _buildItem(items[index]),
)
```

### ListView.separated

```dart
// ListView พร้อม separator ระหว่าง items
ListView.separated(
  itemCount: items.length,
  separatorBuilder: (context, index) => const Divider(height: 1),
  itemBuilder: (context, index) {
    return ListTile(
      title: Text(items[index].title),
    );
  },
)

// Custom separator
ListView.separated(
  itemCount: items.length,
  separatorBuilder: (context, index) {
    if (index % 5 == 4) {
      // โฆษณาทุก 5 รายการ
      return const _AdBanner();
    }
    return const SizedBox(height: 8);
  },
  itemBuilder: (context, index) => ItemCard(item: items[index]),
)
```

### ListView แนวนอน

```dart
// แนวนอน
SizedBox(
  height: 120,
  child: ListView.builder(
    scrollDirection: Axis.horizontal,
    padding: const EdgeInsets.symmetric(horizontal: 16),
    itemCount: categories.length,
    itemBuilder: (context, index) {
      return Container(
        margin: const EdgeInsets.only(right: 12),
        child: CategoryChip(category: categories[index]),
      );
    },
  ),
)
```

---

## ListTile

ListTile เป็น widget ที่ออกแบบมาสำหรับ list items

```dart
// ListTile พื้นฐาน
ListTile(
  leading: const CircleAvatar(child: Icon(Icons.person)),
  title: const Text('John Doe'),
  subtitle: const Text('Flutter Developer'),
  trailing: const Text('9:41 AM'),
  onTap: () {},
  onLongPress: () {},
)

// ListTile แบบขยาย (3 lines)
ListTile(
  isThreeLine: true,
  leading: const Icon(Icons.email),
  title: const Text('Flutter Newsletter'),
  subtitle: const Text(
    'New Flutter 3.x features:\n'
    'Impeller, Material 3, and more!',
  ),
  trailing: const Icon(Icons.chevron_right),
)

// CheckboxListTile
CheckboxListTile(
  value: isSelected,
  onChanged: (v) => setState(() => isSelected = v ?? false),
  title: const Text('เลือกรายการนี้'),
  secondary: const Icon(Icons.bookmark),
)

// SwitchListTile
SwitchListTile(
  value: isEnabled,
  onChanged: (v) => setState(() => isEnabled = v),
  title: const Text('เปิดการแจ้งเตือน'),
  secondary: const Icon(Icons.notifications),
)

// RadioListTile
RadioListTile<String>(
  value: 'option_a',
  groupValue: selectedOption,
  onChanged: (v) => setState(() => selectedOption = v!),
  title: const Text('ตัวเลือก A'),
)

// ExpansionTile
ExpansionTile(
  leading: const Icon(Icons.category),
  title: const Text('หมวดหมู่ย่อย'),
  children: [
    ListTile(
      leading: const SizedBox(width: 40),
      title: const Text('รายการย่อย 1'),
    ),
    ListTile(
      leading: const SizedBox(width: 40),
      title: const Text('รายการย่อย 2'),
    ),
  ],
)
```

---

## GridView

### GridView.count

```dart
// กำหนดจำนวน column
GridView.count(
  crossAxisCount: 2,
  crossAxisSpacing: 16,
  mainAxisSpacing: 16,
  padding: const EdgeInsets.all(16),
  children: items.map((item) => ItemCard(item: item)).toList(),
)

// กำหนด aspect ratio
GridView.count(
  crossAxisCount: 3,
  childAspectRatio: 1.5,  // width/height ratio
  children: [...],
)
```

### GridView.extent

```dart
// กำหนดขนาด max item
GridView.extent(
  maxCrossAxisExtent: 200,  // item กว้างสูงสุด 200
  crossAxisSpacing: 8,
  mainAxisSpacing: 8,
  children: [...],
)
```

### GridView.builder

```dart
// Lazy loading สำหรับ grid ขนาดใหญ่
GridView.builder(
  gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount: 2,
    crossAxisSpacing: 12,
    mainAxisSpacing: 12,
    childAspectRatio: 0.75,
  ),
  itemCount: products.length,
  itemBuilder: (context, index) {
    return ProductCard(product: products[index]);
  },
)

// Adaptive column count
GridView.builder(
  gridDelegate: SliverGridDelegateWithMaxCrossAxisExtent(
    maxCrossAxisExtent: 200,
    childAspectRatio: 0.75,
    crossAxisSpacing: 12,
    mainAxisSpacing: 12,
  ),
  itemCount: products.length,
  itemBuilder: (context, index) => ProductCard(product: products[index]),
)
```

---

## CustomScrollView และ Slivers

Slivers ช่วยสร้าง complex scroll effects

```dart
CustomScrollView(
  slivers: [
    // Collapsible AppBar
    SliverAppBar(
      expandedHeight: 200,
      pinned: true,
      floating: false,
      flexibleSpace: FlexibleSpaceBar(
        title: const Text('Products'),
        background: Image.network(
          'https://picsum.photos/seed/banner/800/400',
          fit: BoxFit.cover,
        ),
      ),
    ),
    
    // Persistent header (category chips)
    SliverPersistentHeader(
      pinned: true,
      delegate: _CategoryHeaderDelegate(
        categories: ['ทั้งหมด', 'อิเล็กทรอนิกส์', 'เสื้อผ้า', 'อาหาร'],
      ),
    ),
    
    // Padding wrapper
    SliverPadding(
      padding: const EdgeInsets.all(16),
      sliver: SliverGrid(
        gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
          crossAxisCount: 2,
          crossAxisSpacing: 12,
          mainAxisSpacing: 12,
          childAspectRatio: 0.75,
        ),
        delegate: SliverChildBuilderDelegate(
          (context, index) => ProductCard(product: products[index]),
          childCount: products.length,
        ),
      ),
    ),
    
    // List ต่อจาก grid
    SliverList(
      delegate: SliverChildBuilderDelegate(
        (context, index) => ListTile(
          title: Text('Related item $index'),
        ),
        childCount: 5,
      ),
    ),
    
    // ช่องว่างล่างสุด
    const SliverToBoxAdapter(
      child: SizedBox(height: 80), // พื้นที่สำหรับ FAB
    ),
  ],
)
```

### SliverPersistentHeader

```dart
class _CategoryHeaderDelegate extends SliverPersistentHeaderDelegate {
  final List<String> categories;
  final double height;
  
  _CategoryHeaderDelegate({
    required this.categories,
    this.height = 60,
  });

  @override
  double get minExtent => height;

  @override
  double get maxExtent => height;

  @override
  bool shouldRebuild(_CategoryHeaderDelegate oldDelegate) {
    return oldDelegate.categories != categories;
  }

  @override
  Widget build(
    BuildContext context,
    double shrinkOffset,
    bool overlapsContent,
  ) {
    return Container(
      color: Theme.of(context).colorScheme.surface,
      child: ListView.builder(
        scrollDirection: Axis.horizontal,
        padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 10),
        itemCount: categories.length,
        itemBuilder: (context, index) {
          return Padding(
            padding: const EdgeInsets.only(right: 8),
            child: FilterChip(
              label: Text(categories[index]),
              onSelected: (_) {},
              selected: index == 0,
            ),
          );
        },
      ),
    );
  }
}
```

---

## Pull to Refresh

```dart
// RefreshIndicator สำหรับ pull-to-refresh
class RefreshableList extends StatefulWidget {
  const RefreshableList({super.key});

  @override
  State<RefreshableList> createState() => _RefreshableListState();
}

class _RefreshableListState extends State<RefreshableList> {
  List<String> _items = List.generate(20, (i) => 'Item $i');

  Future<void> _onRefresh() async {
    // Simulate API call
    await Future.delayed(const Duration(seconds: 1));
    setState(() {
      _items = List.generate(20, (i) => 'Refreshed Item $i');
    });
  }

  @override
  Widget build(BuildContext context) {
    return RefreshIndicator(
      onRefresh: _onRefresh,
      color: Theme.of(context).colorScheme.primary,
      backgroundColor: Theme.of(context).colorScheme.surface,
      child: ListView.builder(
        // ต้องมี physics เพื่อให้ pull ได้เมื่อ items น้อย
        physics: const AlwaysScrollableScrollPhysics(),
        itemCount: _items.length,
        itemBuilder: (context, index) => ListTile(
          title: Text(_items[index]),
        ),
      ),
    );
  }
}
```

---

## Infinite Scroll (Pagination)

```dart
class InfiniteScrollList extends StatefulWidget {
  const InfiniteScrollList({super.key});

  @override
  State<InfiniteScrollList> createState() => _InfiniteScrollListState();
}

class _InfiniteScrollListState extends State<InfiniteScrollList> {
  final List<String> _items = [];
  final ScrollController _scrollController = ScrollController();
  bool _isLoading = false;
  bool _hasMore = true;
  int _page = 1;
  static const int _pageSize = 20;

  @override
  void initState() {
    super.initState();
    _loadMore();
    _scrollController.addListener(_onScroll);
  }

  @override
  void dispose() {
    _scrollController.removeListener(_onScroll);
    _scrollController.dispose();
    super.dispose();
  }

  void _onScroll() {
    if (_isNearBottom() && !_isLoading && _hasMore) {
      _loadMore();
    }
  }

  bool _isNearBottom() {
    if (!_scrollController.hasClients) return false;
    final maxScroll = _scrollController.position.maxScrollExtent;
    final current = _scrollController.offset;
    return current >= maxScroll - 200; // load เมื่อเหลือ 200px
  }

  Future<void> _loadMore() async {
    if (_isLoading) return;
    
    setState(() => _isLoading = true);
    
    // Simulate API
    await Future.delayed(const Duration(seconds: 1));
    
    final newItems = List.generate(
      _pageSize,
      (i) => 'Item ${(_page - 1) * _pageSize + i + 1}',
    );
    
    if (mounted) {
      setState(() {
        _items.addAll(newItems);
        _page++;
        _isLoading = false;
        _hasMore = _page <= 5; // หมด page 5
      });
    }
  }

  Future<void> _refresh() async {
    setState(() {
      _items.clear();
      _page = 1;
      _hasMore = true;
    });
    await _loadMore();
  }

  @override
  Widget build(BuildContext context) {
    return RefreshIndicator(
      onRefresh: _refresh,
      child: ListView.builder(
        controller: _scrollController,
        physics: const AlwaysScrollableScrollPhysics(),
        itemCount: _items.length + (_hasMore ? 1 : 0),
        itemBuilder: (context, index) {
          // Last item = loading indicator หรือ end message
          if (index == _items.length) {
            return _buildBottomWidget();
          }
          
          return ListTile(
            key: ValueKey(_items[index]),
            leading: CircleAvatar(child: Text('${index + 1}')),
            title: Text(_items[index]),
            subtitle: Text('Page: ${(index ~/ _pageSize) + 1}'),
          );
        },
      ),
    );
  }

  Widget _buildBottomWidget() {
    if (_isLoading) {
      return const Padding(
        padding: EdgeInsets.all(16),
        child: Center(child: CircularProgressIndicator()),
      );
    }
    
    if (!_hasMore) {
      return Padding(
        padding: const EdgeInsets.all(16),
        child: Center(
          child: Text(
            '✓ โหลดครบทุกรายการแล้ว (${_items.length} รายการ)',
            style: TextStyle(color: Colors.grey[600]),
          ),
        ),
      );
    }
    
    return const SizedBox.shrink();
  }
}
```

---

## Workshop: Product Catalog with Grid

### lib/models/product.dart

```dart
class Product {
  final String id;
  final String name;
  final double price;
  final double? originalPrice;
  final String imageUrl;
  final double rating;
  final int reviewCount;
  final String category;
  bool isFavorite;

  Product({
    required this.id,
    required this.name,
    required this.price,
    this.originalPrice,
    required this.imageUrl,
    required this.rating,
    required this.reviewCount,
    required this.category,
    this.isFavorite = false,
  });

  double get discountPercent {
    if (originalPrice == null) return 0;
    return ((originalPrice! - price) / originalPrice! * 100);
  }
}
```

### lib/screens/catalog_screen.dart

```dart
import 'package:flutter/material.dart';
import '../models/product.dart';

class CatalogScreen extends StatefulWidget {
  const CatalogScreen({super.key});

  @override
  State<CatalogScreen> createState() => _CatalogScreenState();
}

class _CatalogScreenState extends State<CatalogScreen> {
  String _selectedCategory = 'ทั้งหมด';
  String _sortBy = 'ล่าสุด';
  bool _isGridView = true;
  String _searchQuery = '';
  final _searchController = TextEditingController();

  final List<String> _categories = [
    'ทั้งหมด', 'อิเล็กทรอนิกส์', 'แฟชั่น', 'อาหาร', 'กีฬา'
  ];

  final List<String> _sortOptions = [
    'ล่าสุด', 'ราคาต่ำ-สูง', 'ราคาสูง-ต่ำ', 'คะแนนสูงสุด'
  ];

  // Sample products
  final List<Product> _allProducts = [
    Product(id: '1', name: 'iPhone 15 Pro', price: 44900, originalPrice: 49900,
        imageUrl: 'https://picsum.photos/seed/iphone/400/400',
        rating: 4.8, reviewCount: 1234, category: 'อิเล็กทรอนิกส์'),
    Product(id: '2', name: 'AirPods Pro', price: 8990,
        imageUrl: 'https://picsum.photos/seed/airpods/400/400',
        rating: 4.7, reviewCount: 856, category: 'อิเล็กทรอนิกส์'),
    Product(id: '3', name: 'เสื้อยืด Premium', price: 590, originalPrice: 890,
        imageUrl: 'https://picsum.photos/seed/shirt/400/400',
        rating: 4.3, reviewCount: 432, category: 'แฟชั่น'),
    Product(id: '4', name: 'กางเกงวิ่ง', price: 1290,
        imageUrl: 'https://picsum.photos/seed/pants/400/400',
        rating: 4.5, reviewCount: 298, category: 'กีฬา'),
    Product(id: '5', name: 'กาแฟ Premium', price: 399,
        imageUrl: 'https://picsum.photos/seed/coffee/400/400',
        rating: 4.6, reviewCount: 567, category: 'อาหาร'),
    Product(id: '6', name: 'MacBook Air M2', price: 42900, originalPrice: 46900,
        imageUrl: 'https://picsum.photos/seed/macbook/400/400',
        rating: 4.9, reviewCount: 2345, category: 'อิเล็กทรอนิกส์'),
  ];

  List<Product> get _filteredProducts {
    var products = _allProducts.where((p) {
      final matchCategory = _selectedCategory == 'ทั้งหมด' ||
          p.category == _selectedCategory;
      final matchSearch = _searchQuery.isEmpty ||
          p.name.toLowerCase().contains(_searchQuery.toLowerCase());
      return matchCategory && matchSearch;
    }).toList();

    switch (_sortBy) {
      case 'ราคาต่ำ-สูง':
        products.sort((a, b) => a.price.compareTo(b.price));
      case 'ราคาสูง-ต่ำ':
        products.sort((a, b) => b.price.compareTo(a.price));
      case 'คะแนนสูงสุด':
        products.sort((a, b) => b.rating.compareTo(a.rating));
    }

    return products;
  }

  @override
  void dispose() {
    _searchController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final products = _filteredProducts;

    return Scaffold(
      body: CustomScrollView(
        slivers: [
          // AppBar
          SliverAppBar(
            floating: true,
            snap: true,
            title: const Text('สินค้าทั้งหมด'),
            bottom: PreferredSize(
              preferredSize: const Size.fromHeight(56),
              child: Padding(
                padding: const EdgeInsets.fromLTRB(16, 0, 16, 8),
                child: TextField(
                  controller: _searchController,
                  decoration: InputDecoration(
                    hintText: 'ค้นหาสินค้า...',
                    prefixIcon: const Icon(Icons.search),
                    suffixIcon: _searchQuery.isNotEmpty
                        ? IconButton(
                            icon: const Icon(Icons.clear),
                            onPressed: () {
                              _searchController.clear();
                              setState(() => _searchQuery = '');
                            },
                          )
                        : null,
                    border: OutlineInputBorder(
                      borderRadius: BorderRadius.circular(24),
                    ),
                    filled: true,
                    fillColor: theme.colorScheme.surface,
                    isDense: true,
                    contentPadding: const EdgeInsets.symmetric(vertical: 8),
                  ),
                  onChanged: (v) => setState(() => _searchQuery = v),
                ),
              ),
            ),
          ),

          // Category filter
          SliverPersistentHeader(
            pinned: true,
            delegate: _FilterBarDelegate(
              categories: _categories,
              selectedCategory: _selectedCategory,
              sortBy: _sortBy,
              sortOptions: _sortOptions,
              isGridView: _isGridView,
              onCategoryChanged: (c) => setState(() => _selectedCategory = c),
              onSortChanged: (s) => setState(() => _sortBy = s),
              onViewToggle: () => setState(() => _isGridView = !_isGridView),
            ),
          ),

          // Results count
          SliverToBoxAdapter(
            child: Padding(
              padding: const EdgeInsets.fromLTRB(16, 12, 16, 4),
              child: Text(
                '${products.length} รายการ',
                style: theme.textTheme.bodySmall?.copyWith(
                  color: theme.colorScheme.onSurfaceVariant,
                ),
              ),
            ),
          ),

          // Products
          if (products.isEmpty)
            SliverFillRemaining(
              child: Center(
                child: Column(
                  mainAxisAlignment: MainAxisAlignment.center,
                  children: [
                    Icon(Icons.search_off, size: 64, color: Colors.grey[400]),
                    const SizedBox(height: 16),
                    Text('ไม่พบสินค้า',
                      style: TextStyle(color: Colors.grey[600], fontSize: 18)),
                  ],
                ),
              ),
            )
          else if (_isGridView)
            SliverPadding(
              padding: const EdgeInsets.fromLTRB(16, 8, 16, 80),
              sliver: SliverGrid(
                gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
                  crossAxisCount: 2,
                  crossAxisSpacing: 12,
                  mainAxisSpacing: 12,
                  childAspectRatio: 0.68,
                ),
                delegate: SliverChildBuilderDelegate(
                  (context, index) => _buildGridItem(products[index]),
                  childCount: products.length,
                ),
              ),
            )
          else
            SliverPadding(
              padding: const EdgeInsets.fromLTRB(16, 8, 16, 80),
              sliver: SliverList(
                delegate: SliverChildBuilderDelegate(
                  (context, index) => _buildListItem(products[index]),
                  childCount: products.length,
                ),
              ),
            ),
        ],
      ),
    );
  }

  Widget _buildGridItem(Product product) {
    return Card(
      clipBehavior: Clip.antiAlias,
      child: InkWell(
        onTap: () {},
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // Image
            Expanded(
              flex: 5,
              child: Stack(
                fit: StackFit.expand,
                children: [
                  Image.network(product.imageUrl, fit: BoxFit.cover),
                  if (product.discountPercent > 0)
                    Positioned(
                      top: 8,
                      left: 8,
                      child: Container(
                        padding: const EdgeInsets.symmetric(
                            horizontal: 6, vertical: 3),
                        decoration: BoxDecoration(
                          color: Colors.red,
                          borderRadius: BorderRadius.circular(4),
                        ),
                        child: Text(
                          '-${product.discountPercent.toInt()}%',
                          style: const TextStyle(
                              color: Colors.white,
                              fontSize: 10,
                              fontWeight: FontWeight.bold),
                        ),
                      ),
                    ),
                  Positioned(
                    top: 4,
                    right: 4,
                    child: GestureDetector(
                      onTap: () => setState(() =>
                          product.isFavorite = !product.isFavorite),
                      child: Container(
                        padding: const EdgeInsets.all(6),
                        decoration: BoxDecoration(
                          color: Colors.white.withOpacity(0.9),
                          shape: BoxShape.circle,
                        ),
                        child: Icon(
                          product.isFavorite
                              ? Icons.favorite
                              : Icons.favorite_border,
                          size: 16,
                          color: product.isFavorite ? Colors.red : Colors.grey,
                        ),
                      ),
                    ),
                  ),
                ],
              ),
            ),

            // Info
            Expanded(
              flex: 4,
              child: Padding(
                padding: const EdgeInsets.all(8),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(product.name,
                      maxLines: 2,
                      overflow: TextOverflow.ellipsis,
                      style: const TextStyle(
                          fontSize: 12, fontWeight: FontWeight.w500)),
                    const Spacer(),
                    Row(
                      children: [
                        Icon(Icons.star, size: 12, color: Colors.amber[700]),
                        Text(' ${product.rating}',
                          style: const TextStyle(fontSize: 11)),
                        Text(' (${product.reviewCount})',
                          style: const TextStyle(fontSize: 10, color: Colors.grey)),
                      ],
                    ),
                    const SizedBox(height: 4),
                    Text(
                      '฿${product.price.toStringAsFixed(0)}',
                      style: TextStyle(
                        fontSize: 14,
                        fontWeight: FontWeight.bold,
                        color: Theme.of(context).colorScheme.primary,
                      ),
                    ),
                    if (product.originalPrice != null)
                      Text(
                        '฿${product.originalPrice!.toStringAsFixed(0)}',
                        style: const TextStyle(
                          fontSize: 11,
                          decoration: TextDecoration.lineThrough,
                          color: Colors.grey,
                        ),
                      ),
                  ],
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }

  Widget _buildListItem(Product product) {
    return Card(
      margin: const EdgeInsets.only(bottom: 8),
      child: InkWell(
        onTap: () {},
        child: Padding(
          padding: const EdgeInsets.all(12),
          child: Row(
            children: [
              // Image
              ClipRRect(
                borderRadius: BorderRadius.circular(8),
                child: Image.network(
                  product.imageUrl,
                  width: 80,
                  height: 80,
                  fit: BoxFit.cover,
                ),
              ),
              const SizedBox(width: 12),

              // Info
              Expanded(
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(product.name,
                      style: const TextStyle(fontWeight: FontWeight.w600)),
                    const SizedBox(height: 4),
                    Row(
                      children: [
                        Icon(Icons.star, size: 14, color: Colors.amber[700]),
                        Text(' ${product.rating} (${product.reviewCount})',
                          style: const TextStyle(fontSize: 12, color: Colors.grey)),
                      ],
                    ),
                    const SizedBox(height: 8),
                    Row(
                      children: [
                        Text(
                          '฿${product.price.toStringAsFixed(0)}',
                          style: TextStyle(
                            fontWeight: FontWeight.bold,
                            color: Theme.of(context).colorScheme.primary,
                          ),
                        ),
                        if (product.originalPrice != null) ...[
                          const SizedBox(width: 8),
                          Text(
                            '฿${product.originalPrice!.toStringAsFixed(0)}',
                            style: const TextStyle(
                              decoration: TextDecoration.lineThrough,
                              color: Colors.grey,
                              fontSize: 12,
                            ),
                          ),
                        ],
                      ],
                    ),
                  ],
                ),
              ),

              // Actions
              Column(
                children: [
                  IconButton(
                    icon: Icon(
                      product.isFavorite ? Icons.favorite : Icons.favorite_border,
                      color: product.isFavorite ? Colors.red : null,
                    ),
                    onPressed: () => setState(() =>
                        product.isFavorite = !product.isFavorite),
                  ),
                  IconButton(
                    icon: const Icon(Icons.add_shopping_cart),
                    onPressed: () {},
                  ),
                ],
              ),
            ],
          ),
        ),
      ),
    );
  }
}

// Filter bar delegate
class _FilterBarDelegate extends SliverPersistentHeaderDelegate {
  final List<String> categories;
  final String selectedCategory;
  final String sortBy;
  final List<String> sortOptions;
  final bool isGridView;
  final void Function(String) onCategoryChanged;
  final void Function(String) onSortChanged;
  final VoidCallback onViewToggle;

  _FilterBarDelegate({
    required this.categories,
    required this.selectedCategory,
    required this.sortBy,
    required this.sortOptions,
    required this.isGridView,
    required this.onCategoryChanged,
    required this.onSortChanged,
    required this.onViewToggle,
  });

  @override
  double get minExtent => 110;

  @override
  double get maxExtent => 110;

  @override
  bool shouldRebuild(_FilterBarDelegate old) => true;

  @override
  Widget build(context, shrinkOffset, overlapsContent) {
    return Container(
      color: Theme.of(context).colorScheme.surface,
      child: Column(
        children: [
          // Categories
          SizedBox(
            height: 50,
            child: ListView.builder(
              scrollDirection: Axis.horizontal,
              padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
              itemCount: categories.length,
              itemBuilder: (context, index) {
                final cat = categories[index];
                return Padding(
                  padding: const EdgeInsets.only(right: 8),
                  child: FilterChip(
                    label: Text(cat),
                    selected: cat == selectedCategory,
                    onSelected: (_) => onCategoryChanged(cat),
                  ),
                );
              },
            ),
          ),

          // Sort and view toggle
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 16),
            child: Row(
              children: [
                const Icon(Icons.sort, size: 18),
                const SizedBox(width: 4),
                DropdownButton<String>(
                  value: sortBy,
                  underline: const SizedBox(),
                  isDense: true,
                  items: sortOptions
                      .map((s) => DropdownMenuItem(value: s, child: Text(s)))
                      .toList(),
                  onChanged: (v) => onSortChanged(v!),
                ),
                const Spacer(),
                IconButton(
                  icon: Icon(isGridView ? Icons.list : Icons.grid_view),
                  onPressed: onViewToggle,
                  iconSize: 20,
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}
```

---

## สรุปบทที่ 28

ในบทนี้เราได้เรียนรู้:

1. **ListView**: builder, separated, แนวนอน
2. **ListTile**: การสร้าง list item มาตรฐาน
3. **GridView**: count, extent, builder
4. **CustomScrollView/Slivers**: complex scroll
5. **Pull to Refresh**: RefreshIndicator
6. **Infinite Scroll**: pagination ด้วย ScrollController
7. **Workshop**: Product catalog พร้อม filter และ view toggle

บทต่อไปเราจะเรียน Themes และ Styling
