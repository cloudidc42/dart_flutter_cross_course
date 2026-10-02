# Part 96: Real-World Project - E-Commerce App (Complete)

## 🎯 เป้าหมายของ Part นี้
- สร้าง E-Commerce app แบบสมบูรณ์
- รวมทุก concept ที่เรียนมา
- Clean architecture
- Firebase backend
- Payment integration

---

## 1. Project Overview

```
ShopApp - E-Commerce Flutter Application
├── Features:
│   ├── Authentication (Firebase Auth)
│   ├── Product Catalog (Grid/List view, Search, Filter)
│   ├── Product Detail (Images, Description, Reviews)
│   ├── Shopping Cart (Add/Remove, Quantity)
│   ├── Checkout (Shipping, Payment)
│   ├── Order History
│   ├── User Profile
│   └── Favorites/Wishlist
│
├── Tech Stack:
│   ├── State Management: Riverpod
│   ├── Navigation: GoRouter
│   ├── Backend: Firebase (Auth + Firestore + Storage)
│   ├── HTTP: Dio
│   ├── Local Storage: Hive
│   └── Architecture: Clean Architecture
```

---

## 2. Project Structure

```
lib/
├── core/
│   ├── constants/
│   │   ├── app_colors.dart
│   │   ├── app_text_styles.dart
│   │   └── app_strings.dart
│   ├── errors/
│   │   ├── exceptions.dart
│   │   └── failures.dart
│   ├── network/
│   │   └── dio_client.dart
│   ├── router/
│   │   └── app_router.dart
│   └── theme/
│       └── app_theme.dart
│
├── features/
│   ├── auth/
│   ├── products/
│   ├── cart/
│   ├── orders/
│   └── profile/
│
└── main.dart
```

---

## 3. Complete Implementation

### main.dart
```dart
import 'package:firebase_core/firebase_core.dart';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import 'core/router/app_router.dart';
import 'core/theme/app_theme.dart';
import 'firebase_options.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
  
  runApp(
    const ProviderScope(
      child: ShopApp(),
    ),
  );
}

class ShopApp extends ConsumerWidget {
  const ShopApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final router = ref.watch(appRouterProvider);
    
    return MaterialApp.router(
      title: 'ShopApp',
      theme: AppTheme.lightTheme,
      darkTheme: AppTheme.darkTheme,
      themeMode: ThemeMode.system,
      routerConfig: router,
      debugShowCheckedModeBanner: false,
    );
  }
}
```

### Theme
```dart
// core/theme/app_theme.dart
import 'package:flutter/material.dart';

class AppTheme {
  static final lightTheme = ThemeData(
    useMaterial3: true,
    colorScheme: ColorScheme.fromSeed(
      seedColor: const Color(0xFF6750A4),
      brightness: Brightness.light,
    ),
    cardTheme: CardTheme(
      elevation: 2,
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.circular(12),
      ),
    ),
    inputDecorationTheme: InputDecorationTheme(
      border: OutlineInputBorder(
        borderRadius: BorderRadius.circular(8),
      ),
      filled: true,
    ),
    elevatedButtonTheme: ElevatedButtonThemeData(
      style: ElevatedButton.styleFrom(
        minimumSize: const Size.fromHeight(48),
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(8),
        ),
      ),
    ),
  );
  
  static final darkTheme = ThemeData(
    useMaterial3: true,
    colorScheme: ColorScheme.fromSeed(
      seedColor: const Color(0xFF6750A4),
      brightness: Brightness.dark,
    ),
  );
}
```

### Router
```dart
// core/router/app_router.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';

final appRouterProvider = Provider<GoRouter>((ref) {
  final authState = ref.watch(authStateProvider);
  
  return GoRouter(
    initialLocation: '/',
    redirect: (context, state) {
      final isLoggedIn = authState.value != null;
      final isAuthRoute = state.matchedLocation.startsWith('/auth');
      
      if (!isLoggedIn && !isAuthRoute) {
        return '/auth/login';
      }
      if (isLoggedIn && isAuthRoute) {
        return '/';
      }
      return null;
    },
    routes: [
      GoRoute(
        path: '/',
        builder: (context, state) => const HomePage(),
      ),
      GoRoute(
        path: '/products',
        builder: (context, state) => const ProductListPage(),
        routes: [
          GoRoute(
            path: ':id',
            builder: (context, state) => ProductDetailPage(
              productId: state.pathParameters['id']!,
            ),
          ),
        ],
      ),
      GoRoute(
        path: '/cart',
        builder: (context, state) => const CartPage(),
      ),
      GoRoute(
        path: '/checkout',
        builder: (context, state) => const CheckoutPage(),
      ),
      GoRoute(
        path: '/orders',
        builder: (context, state) => const OrdersPage(),
        routes: [
          GoRoute(
            path: ':id',
            builder: (context, state) => OrderDetailPage(
              orderId: state.pathParameters['id']!,
            ),
          ),
        ],
      ),
      GoRoute(
        path: '/profile',
        builder: (context, state) => const ProfilePage(),
      ),
      GoRoute(
        path: '/favorites',
        builder: (context, state) => const FavoritesPage(),
      ),
      ShellRoute(
        builder: (context, state, child) => MainScaffold(child: child),
        routes: [
          // bottom nav routes
        ],
      ),
      GoRoute(
        path: '/auth',
        redirect: (context, state) => '/auth/login',
      ),
      GoRoute(
        path: '/auth/login',
        builder: (context, state) => const LoginPage(),
      ),
      GoRoute(
        path: '/auth/register',
        builder: (context, state) => const RegisterPage(),
      ),
    ],
  );
});
```

### Product Feature
```dart
// features/products/domain/entities/product.dart
class Product {
  final String id;
  final String name;
  final String description;
  final double price;
  final double? salePrice;
  final List<String> images;
  final String category;
  final double rating;
  final int reviewCount;
  final int stock;
  final Map<String, dynamic> specifications;
  
  const Product({
    required this.id,
    required this.name,
    required this.description,
    required this.price,
    this.salePrice,
    required this.images,
    required this.category,
    required this.rating,
    required this.reviewCount,
    required this.stock,
    this.specifications = const {},
  });
  
  bool get isOnSale => salePrice != null && salePrice! < price;
  bool get inStock => stock > 0;
  double get effectivePrice => salePrice ?? price;
  
  double get discountPercent => isOnSale 
      ? ((price - salePrice!) / price * 100).roundToDouble()
      : 0;
}

// features/products/presentation/pages/product_list_page.dart
class ProductListPage extends ConsumerWidget {
  const ProductListPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final productsAsync = ref.watch(productsProvider);
    final filterState = ref.watch(productFilterProvider);
    
    return Scaffold(
      appBar: AppBar(
        title: const Text('สินค้า'),
        actions: [
          IconButton(
            icon: const Icon(Icons.search),
            onPressed: () => showSearch(
              context: context,
              delegate: ProductSearchDelegate(ref),
            ),
          ),
          IconButton(
            icon: const Icon(Icons.filter_list),
            onPressed: () => _showFilterSheet(context, ref),
          ),
        ],
      ),
      body: Column(
        children: [
          // Category filter chips:
          const CategoryFilterBar(),
          
          // Products grid/list:
          Expanded(
            child: productsAsync.when(
              loading: () => const LoadingGrid(),
              error: (error, stack) => ErrorWidget(
                message: error.toString(),
                onRetry: () => ref.invalidate(productsProvider),
              ),
              data: (products) => products.isEmpty
                  ? const EmptyProductsWidget()
                  : ProductGrid(products: products),
            ),
          ),
        ],
      ),
    );
  }
  
  void _showFilterSheet(BuildContext context, WidgetRef ref) {
    showModalBottomSheet(
      context: context,
      isScrollControlled: true,
      builder: (context) => const ProductFilterSheet(),
    );
  }
}

// Product Card:
class ProductCard extends StatelessWidget {
  final Product product;
  
  const ProductCard({super.key, required this.product});

  @override
  Widget build(BuildContext context) {
    return Card(
      clipBehavior: Clip.antiAlias,
      child: InkWell(
        onTap: () => context.push('/products/${product.id}'),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // Product image with sale badge:
            AspectRatio(
              aspectRatio: 1,
              child: Stack(
                children: [
                  CachedNetworkImage(
                    imageUrl: product.images.first,
                    fit: BoxFit.cover,
                    width: double.infinity,
                    placeholder: (context, url) => const ShimmerBox(),
                    errorWidget: (context, url, error) => const Icon(Icons.broken_image),
                  ),
                  if (product.isOnSale)
                    Positioned(
                      top: 8,
                      left: 8,
                      child: Container(
                        padding: const EdgeInsets.symmetric(
                          horizontal: 8, vertical: 4,
                        ),
                        decoration: BoxDecoration(
                          color: Colors.red,
                          borderRadius: BorderRadius.circular(4),
                        ),
                        child: Text(
                          '-${product.discountPercent.toInt()}%',
                          style: const TextStyle(
                            color: Colors.white,
                            fontSize: 12,
                            fontWeight: FontWeight.bold,
                          ),
                        ),
                      ),
                    ),
                  Positioned(
                    top: 8,
                    right: 8,
                    child: FavoriteButton(productId: product.id),
                  ),
                ],
              ),
            ),
            
            Padding(
              padding: const EdgeInsets.all(8),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(
                    product.name,
                    maxLines: 2,
                    overflow: TextOverflow.ellipsis,
                    style: Theme.of(context).textTheme.bodyMedium,
                  ),
                  const SizedBox(height: 4),
                  Row(
                    children: [
                      const Icon(Icons.star, color: Colors.amber, size: 14),
                      Text(
                        '${product.rating} (${product.reviewCount})',
                        style: Theme.of(context).textTheme.bodySmall,
                      ),
                    ],
                  ),
                  const SizedBox(height: 4),
                  if (product.isOnSale) ...[
                    Text(
                      '฿${product.price.toStringAsFixed(0)}',
                      style: Theme.of(context).textTheme.bodySmall?.copyWith(
                        decoration: TextDecoration.lineThrough,
                        color: Colors.grey,
                      ),
                    ),
                    Text(
                      '฿${product.salePrice!.toStringAsFixed(0)}',
                      style: Theme.of(context).textTheme.titleMedium?.copyWith(
                        color: Colors.red,
                        fontWeight: FontWeight.bold,
                      ),
                    ),
                  ] else
                    Text(
                      '฿${product.price.toStringAsFixed(0)}',
                      style: Theme.of(context).textTheme.titleMedium?.copyWith(
                        fontWeight: FontWeight.bold,
                      ),
                    ),
                ],
              ),
            ),
            
            Padding(
              padding: const EdgeInsets.fromLTRB(8, 0, 8, 8),
              child: SizedBox(
                width: double.infinity,
                child: FilledButton.icon(
                  onPressed: product.inStock
                      ? () => _addToCart(context)
                      : null,
                  icon: const Icon(Icons.shopping_cart, size: 16),
                  label: Text(product.inStock ? 'เพิ่มลงตะกร้า' : 'หมดสต็อก'),
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }
  
  void _addToCart(BuildContext context) {
    // Add to cart logic
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(
        content: Text('เพิ่ม ${product.name} ลงตะกร้าแล้ว'),
        action: SnackBarAction(
          label: 'ดูตะกร้า',
          onPressed: () => context.push('/cart'),
        ),
      ),
    );
  }
}
```

### Cart Feature
```dart
// features/cart/presentation/pages/cart_page.dart
class CartPage extends ConsumerWidget {
  const CartPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final cart = ref.watch(cartProvider);
    
    if (cart.isEmpty) {
      return Scaffold(
        appBar: AppBar(title: const Text('ตะกร้าสินค้า')),
        body: const EmptyCartWidget(),
      );
    }
    
    return Scaffold(
      appBar: AppBar(
        title: Text('ตะกร้า (${cart.itemCount} ชิ้น)'),
        actions: [
          TextButton(
            onPressed: () => ref.read(cartProvider.notifier).clear(),
            child: const Text('ล้างตะกร้า'),
          ),
        ],
      ),
      body: Column(
        children: [
          Expanded(
            child: ListView.builder(
              itemCount: cart.items.length,
              itemBuilder: (context, index) {
                final item = cart.items[index];
                return CartItemTile(item: item);
              },
            ),
          ),
          
          // Order summary:
          Container(
            padding: const EdgeInsets.all(16),
            decoration: BoxDecoration(
              color: Theme.of(context).colorScheme.surface,
              boxShadow: [
                BoxShadow(
                  color: Colors.black.withOpacity(0.1),
                  blurRadius: 8,
                  offset: const Offset(0, -2),
                ),
              ],
            ),
            child: Column(
              children: [
                Row(
                  mainAxisAlignment: MainAxisAlignment.spaceBetween,
                  children: [
                    const Text('ยอดรวมสินค้า:'),
                    Text('฿${cart.subtotal.toStringAsFixed(0)}'),
                  ],
                ),
                if (cart.discount > 0) ...[
                  const SizedBox(height: 8),
                  Row(
                    mainAxisAlignment: MainAxisAlignment.spaceBetween,
                    children: [
                      const Text('ส่วนลด:'),
                      Text(
                        '-฿${cart.discount.toStringAsFixed(0)}',
                        style: const TextStyle(color: Colors.red),
                      ),
                    ],
                  ),
                ],
                const Divider(),
                Row(
                  mainAxisAlignment: MainAxisAlignment.spaceBetween,
                  children: [
                    const Text(
                      'ยอดรวมทั้งหมด:',
                      style: TextStyle(fontWeight: FontWeight.bold),
                    ),
                    Text(
                      '฿${cart.total.toStringAsFixed(0)}',
                      style: const TextStyle(
                        fontWeight: FontWeight.bold,
                        fontSize: 18,
                      ),
                    ),
                  ],
                ),
                const SizedBox(height: 16),
                SizedBox(
                  width: double.infinity,
                  child: FilledButton(
                    onPressed: () => context.push('/checkout'),
                    child: const Text('สั่งซื้อสินค้า'),
                  ),
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

## 4. pubspec.yaml สำหรับโปรเจค

```yaml
name: shop_app
description: E-Commerce App built with Flutter
version: 1.0.0+1

environment:
  sdk: '>=3.0.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter
  
  # State Management
  flutter_riverpod: ^2.4.0
  riverpod_annotation: ^2.3.0
  
  # Navigation
  go_router: ^12.0.0
  
  # Firebase
  firebase_core: ^2.24.0
  firebase_auth: ^4.14.0
  cloud_firestore: ^4.13.0
  firebase_storage: ^11.5.0
  
  # Network
  dio: ^5.3.0
  
  # Local Storage
  hive_flutter: ^1.1.0
  shared_preferences: ^2.2.0
  
  # UI
  cached_network_image: ^3.3.0
  shimmer: ^3.0.0
  
  # Utils
  intl: ^0.18.0
  either_dart: ^1.0.0
  equatable: ^2.0.5
  
  # Icons
  cupertino_icons: ^1.0.6

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0
  mockito: ^5.4.0
  build_runner: ^2.4.0
  riverpod_generator: ^2.3.0
  hive_generator: ^2.0.0
  json_serializable: ^6.7.0
```

---

## 5. สรุป Part 96

สิ่งที่เรียนรู้:
- ✅ Complete E-Commerce app architecture
- ✅ Feature-first folder structure
- ✅ Riverpod + GoRouter integration
- ✅ Firebase backend
- ✅ Product catalog with filtering/search
- ✅ Shopping cart with state management
- ✅ Material 3 UI components
- ✅ Image caching, loading states

---

## ➡️ Part ถัดไป
**Part 97: Real-World Project - Social Media App**
