# Part 57: Advanced Navigation with GoRouter

## GoRouter คืออะไร?

GoRouter เป็น package สำหรับจัดการ Navigation ใน Flutter ที่รองรับ:
- Declarative routing
- Deep linking
- URL-based navigation (สำหรับ Web)
- Route guards
- Nested navigation
- Named routes

```yaml
# pubspec.yaml
dependencies:
  go_router: ^13.0.0
```

---

## 1. GoRouter Setup

### การตั้งค่าพื้นฐาน

```dart
// router/app_router.dart
import 'package:go_router/go_router.dart';
import 'package:flutter/material.dart';

// กำหนด routes ทั้งหมดในที่เดียว
final GoRouter appRouter = GoRouter(
  initialLocation: '/',
  debugLogDiagnostics: true, // แสดง log ใน debug mode
  routes: [
    GoRoute(
      path: '/',
      name: 'home',
      builder: (context, state) => HomeScreen(),
    ),
    GoRoute(
      path: '/profile',
      name: 'profile',
      builder: (context, state) => ProfileScreen(),
    ),
    GoRoute(
      path: '/settings',
      name: 'settings',
      builder: (context, state) => SettingsScreen(),
    ),
  ],
  // Error page
  errorBuilder: (context, state) => ErrorScreen(error: state.error),
);

// main.dart
void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      title: 'GoRouter Demo',
      routerConfig: appRouter, // ใช้ routerConfig แทน router แบบเก่า
      theme: ThemeData(primarySwatch: Colors.blue),
    );
  }
}
```

### Navigation Methods

```dart
// วิธีต่างๆ ในการ navigate
class NavigationExamples extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // 1. Go - แทนที่ current location (ไม่สามารถกลับได้)
        ElevatedButton(
          onPressed: () => context.go('/profile'),
          child: Text('Go to Profile'),
        ),

        // 2. Push - เพิ่มใน navigation stack (กดกลับได้)
        ElevatedButton(
          onPressed: () => context.push('/settings'),
          child: Text('Push Settings'),
        ),

        // 3. Named route navigation
        ElevatedButton(
          onPressed: () => context.goNamed('profile'),
          child: Text('Go to Profile (Named)'),
        ),

        // 4. Replace - แทนที่ current route
        ElevatedButton(
          onPressed: () => context.pushReplacement('/new-page'),
          child: Text('Replace Current'),
        ),

        // 5. Pop - กลับหน้าก่อน
        ElevatedButton(
          onPressed: () {
            if (context.canPop()) {
              context.pop();
            }
          },
          child: Text('Go Back'),
        ),
      ],
    );
  }
}
```

---

## 2. Named Routes และ Paths

### กำหนด Named Routes

```dart
// constants/route_names.dart
class RouteNames {
  static const String home = 'home';
  static const String products = 'products';
  static const String productDetail = 'product-detail';
  static const String cart = 'cart';
  static const String checkout = 'checkout';
  static const String orderSuccess = 'order-success';
  static const String profile = 'profile';
  static const String login = 'login';
}

// router/app_router.dart
final GoRouter appRouter = GoRouter(
  initialLocation: '/',
  routes: [
    GoRoute(
      path: '/',
      name: RouteNames.home,
      builder: (context, state) => HomeScreen(),
    ),
    GoRoute(
      path: '/products',
      name: RouteNames.products,
      builder: (context, state) => ProductListScreen(),
      routes: [
        // Nested route
        GoRoute(
          path: ':productId', // path parameter
          name: RouteNames.productDetail,
          builder: (context, state) {
            final productId = state.pathParameters['productId']!;
            return ProductDetailScreen(productId: productId);
          },
        ),
      ],
    ),
    GoRoute(
      path: '/cart',
      name: RouteNames.cart,
      builder: (context, state) => CartScreen(),
    ),
    GoRoute(
      path: '/checkout',
      name: RouteNames.checkout,
      builder: (context, state) => CheckoutScreen(),
      routes: [
        GoRoute(
          path: 'success',
          name: RouteNames.orderSuccess,
          builder: (context, state) => OrderSuccessScreen(),
        ),
      ],
    ),
  ],
);

// การใช้ Named Routes
class ProductCard extends StatelessWidget {
  final String productId;
  final String productName;

  const ProductCard({required this.productId, required this.productName});

  @override
  Widget build(BuildContext context) {
    return Card(
      child: ListTile(
        title: Text(productName),
        onTap: () {
          // Navigate โดยใช้ named route
          context.goNamed(
            RouteNames.productDetail,
            pathParameters: {'productId': productId},
          );
        },
      ),
    );
  }
}
```

---

## 3. Route Parameters

### Path Parameters

```dart
// /:userId/posts/:postId
GoRoute(
  path: '/users/:userId/posts/:postId',
  name: 'post-detail',
  builder: (context, state) {
    final userId = state.pathParameters['userId']!;
    final postId = state.pathParameters['postId']!;

    return PostDetailScreen(
      userId: userId,
      postId: postId,
    );
  },
),
```

### Query Parameters

```dart
// /search?q=flutter&category=tutorial
GoRoute(
  path: '/search',
  name: 'search',
  builder: (context, state) {
    final query = state.uri.queryParameters['q'] ?? '';
    final category = state.uri.queryParameters['category'];

    return SearchScreen(
      query: query,
      category: category,
    );
  },
),

// Navigate พร้อม query parameters
class SearchButton extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ElevatedButton(
      onPressed: () {
        context.goNamed(
          'search',
          queryParameters: {
            'q': 'flutter tutorial',
            'category': 'mobile',
          },
        );
      },
      child: Text('ค้นหา'),
    );
  }
}
```

### Extra Parameters (Object passing)

```dart
// ส่ง object ผ่าน extra
GoRoute(
  path: '/product-preview',
  name: 'product-preview',
  builder: (context, state) {
    final product = state.extra as Product;
    return ProductPreviewScreen(product: product);
  },
),

// Navigate พร้อม extra object
void navigateToPreview(BuildContext context, Product product) {
  context.goNamed(
    'product-preview',
    extra: product, // ส่ง object
  );
}
```

---

## 4. Nested Navigation

### ShellRoute สำหรับ Bottom Navigation

```dart
// router/app_router.dart
final GoRouter appRouter = GoRouter(
  initialLocation: '/home',
  routes: [
    // Login route (ไม่มี bottom nav)
    GoRoute(
      path: '/login',
      builder: (context, state) => LoginScreen(),
    ),

    // Shell route สำหรับหน้าที่มี Bottom Navigation
    ShellRoute(
      builder: (context, state, child) {
        return MainScaffold(child: child); // Scaffold ที่มี Bottom Nav
      },
      routes: [
        GoRoute(
          path: '/home',
          name: 'home',
          builder: (context, state) => HomeTab(),
        ),
        GoRoute(
          path: '/explore',
          name: 'explore',
          builder: (context, state) => ExploreTab(),
          routes: [
            GoRoute(
              path: 'detail/:id',
              builder: (context, state) {
                final id = state.pathParameters['id']!;
                return ExploreDetailScreen(id: id);
              },
            ),
          ],
        ),
        GoRoute(
          path: '/notifications',
          name: 'notifications',
          builder: (context, state) => NotificationsTab(),
        ),
        GoRoute(
          path: '/profile',
          name: 'profile',
          builder: (context, state) => ProfileTab(),
        ),
      ],
    ),
  ],
);

// MainScaffold.dart
class MainScaffold extends StatelessWidget {
  final Widget child;

  const MainScaffold({required this.child});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: child,
      bottomNavigationBar: BottomNavigationBar(
        currentIndex: _getCurrentIndex(context),
        onTap: (index) => _onTabTapped(context, index),
        items: [
          BottomNavigationBarItem(icon: Icon(Icons.home), label: 'หน้าหลัก'),
          BottomNavigationBarItem(icon: Icon(Icons.explore), label: 'สำรวจ'),
          BottomNavigationBarItem(
              icon: Icon(Icons.notifications), label: 'แจ้งเตือน'),
          BottomNavigationBarItem(icon: Icon(Icons.person), label: 'โปรไฟล์'),
        ],
      ),
    );
  }

  int _getCurrentIndex(BuildContext context) {
    final location = GoRouterState.of(context).uri.path;
    if (location.startsWith('/explore')) return 1;
    if (location.startsWith('/notifications')) return 2;
    if (location.startsWith('/profile')) return 3;
    return 0; // home
  }

  void _onTabTapped(BuildContext context, int index) {
    switch (index) {
      case 0:
        context.go('/home');
        break;
      case 1:
        context.go('/explore');
        break;
      case 2:
        context.go('/notifications');
        break;
      case 3:
        context.go('/profile');
        break;
    }
  }
}
```

---

## 5. Route Guards (Redirects)

### Authentication Guard

```dart
// services/auth_service.dart
class AuthService extends ChangeNotifier {
  bool _isAuthenticated = false;
  User? _currentUser;

  bool get isAuthenticated => _isAuthenticated;
  User? get currentUser => _currentUser;

  Future<void> login(String email, String password) async {
    // ทำการ login
    _isAuthenticated = true;
    notifyListeners();
  }

  Future<void> logout() async {
    _isAuthenticated = false;
    _currentUser = null;
    notifyListeners();
  }
}

// router/app_router.dart
GoRouter createRouter(AuthService authService) {
  return GoRouter(
    initialLocation: '/home',
    refreshListenable: authService, // refresh เมื่อ auth state เปลี่ยน

    // Global redirect
    redirect: (context, state) {
      final isAuthenticated = authService.isAuthenticated;
      final isLoginPage = state.matchedLocation == '/login';

      // ถ้าไม่ได้ login และไม่ได้อยู่ที่หน้า login -> ไปหน้า login
      if (!isAuthenticated && !isLoginPage) {
        return '/login?redirect=${state.uri}';
      }

      // ถ้า login แล้วและอยู่ที่หน้า login -> ไปหน้าหลัก
      if (isAuthenticated && isLoginPage) {
        final redirectTo = state.uri.queryParameters['redirect'];
        return redirectTo ?? '/home';
      }

      return null; // ไม่ redirect
    },

    routes: [
      GoRoute(
        path: '/login',
        builder: (context, state) => LoginScreen(),
      ),
      ShellRoute(
        builder: (context, state, child) => MainScaffold(child: child),
        routes: [
          GoRoute(
            path: '/home',
            builder: (context, state) => HomeScreen(),
          ),
          GoRoute(
            path: '/admin',
            // Route-level redirect สำหรับ admin only
            redirect: (context, state) {
              final user = authService.currentUser;
              if (user?.role != 'admin') {
                return '/home'; // ไม่มีสิทธิ์ -> กลับหน้าหลัก
              }
              return null;
            },
            builder: (context, state) => AdminScreen(),
          ),
        ],
      ),
    ],
  );
}
```

### Role-based Access Control

```dart
// การตรวจสอบ role ด้วย redirect
String? roleBasedRedirect(
  BuildContext context,
  GoRouterState state,
  AuthService auth,
  List<String> requiredRoles,
) {
  final user = auth.currentUser;

  if (user == null) {
    return '/login?redirect=${state.uri}';
  }

  if (!requiredRoles.contains(user.role)) {
    return '/unauthorized';
  }

  return null;
}

// ใช้งาน
GoRoute(
  path: '/admin/users',
  redirect: (context, state) => roleBasedRedirect(
    context,
    state,
    authService,
    ['admin', 'superadmin'],
  ),
  builder: (context, state) => AdminUsersScreen(),
),
```

---

## 6. Deep Linking

### iOS Deep Link Configuration

```xml
<!-- ios/Runner/Info.plist -->
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleTypeRole</key>
    <string>Editor</string>
    <key>CFBundleURLName</key>
    <string>com.example.myapp</string>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>myapp</string>
    </array>
  </dict>
</array>
```

### Android Deep Link Configuration

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<activity ...>
  <intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="https"
          android:host="myapp.example.com" />
  </intent-filter>

  <!-- Custom scheme -->
  <intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="myapp" />
  </intent-filter>
</activity>
```

### GoRouter Deep Link Handling

```dart
// Deep link URLs จะถูก handle โดย GoRouter อัตโนมัติ
// https://myapp.example.com/products/123 -> /products/123
// myapp://products/123 -> /products/123

final GoRouter appRouter = GoRouter(
  initialLocation: '/',
  routes: [
    GoRoute(
      path: '/products/:id',
      builder: (context, state) {
        final productId = state.pathParameters['id']!;
        return ProductDetailScreen(productId: productId);
      },
    ),
    GoRoute(
      path: '/share',
      builder: (context, state) {
        // รับ share data จาก query params
        final type = state.uri.queryParameters['type'];
        final id = state.uri.queryParameters['id'];
        return ShareScreen(type: type, id: id);
      },
    ),
  ],
);
```

---

## Workshop: App ที่มี Navigation ซับซ้อน

### สร้าง E-commerce App พร้อม GoRouter

```dart
// main.dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';
import 'package:provider/provider.dart';

void main() {
  runApp(
    MultiProvider(
      providers: [
        ChangeNotifierProvider(create: (_) => AuthService()),
        ChangeNotifierProvider(create: (_) => CartProvider()),
      ],
      child: ECommerceApp(),
    ),
  );
}

class ECommerceApp extends StatefulWidget {
  @override
  _ECommerceAppState createState() => _ECommerceAppState();
}

class _ECommerceAppState extends State<ECommerceApp> {
  late GoRouter _router;

  @override
  void initState() {
    super.initState();
    final authService = Provider.of<AuthService>(context, listen: false);
    _router = _createRouter(authService);
  }

  GoRouter _createRouter(AuthService authService) {
    return GoRouter(
      initialLocation: '/home',
      refreshListenable: authService,
      redirect: (context, state) {
        final isAuth = authService.isAuthenticated;
        final isOnLoginPage = state.matchedLocation.startsWith('/auth');

        if (!isAuth && !isOnLoginPage) {
          // หน้าที่ต้อง login
          const protectedPaths = ['/checkout', '/orders', '/profile'];
          if (protectedPaths.any((p) => state.matchedLocation.startsWith(p))) {
            return '/auth/login?redirect=${Uri.encodeComponent(state.uri.toString())}';
          }
        }

        if (isAuth && isOnLoginPage) {
          return '/home';
        }

        return null;
      },
      routes: [
        // Auth routes
        GoRoute(
          path: '/auth/login',
          builder: (context, state) {
            final redirect = state.uri.queryParameters['redirect'];
            return LoginScreen(redirectTo: redirect);
          },
        ),
        GoRoute(
          path: '/auth/register',
          builder: (context, state) => RegisterScreen(),
        ),

        // Main app with shell
        ShellRoute(
          builder: (context, state, child) => AppShell(child: child),
          routes: [
            // Home
            GoRoute(
              path: '/home',
              builder: (context, state) => HomeScreen(),
            ),

            // Products
            GoRoute(
              path: '/products',
              builder: (context, state) {
                final categoryId = state.uri.queryParameters['category'];
                return ProductListScreen(categoryId: categoryId);
              },
              routes: [
                GoRoute(
                  path: ':productId',
                  builder: (context, state) {
                    final productId = state.pathParameters['productId']!;
                    return ProductDetailScreen(productId: productId);
                  },
                ),
              ],
            ),

            // Cart
            GoRoute(
              path: '/cart',
              builder: (context, state) => CartScreen(),
            ),

            // Checkout (protected)
            GoRoute(
              path: '/checkout',
              builder: (context, state) => CheckoutScreen(),
              routes: [
                GoRoute(
                  path: 'payment',
                  builder: (context, state) => PaymentScreen(),
                ),
                GoRoute(
                  path: 'success/:orderId',
                  builder: (context, state) {
                    final orderId = state.pathParameters['orderId']!;
                    return OrderSuccessScreen(orderId: orderId);
                  },
                ),
              ],
            ),

            // Orders (protected)
            GoRoute(
              path: '/orders',
              builder: (context, state) => OrderListScreen(),
              routes: [
                GoRoute(
                  path: ':orderId',
                  builder: (context, state) {
                    final orderId = state.pathParameters['orderId']!;
                    return OrderDetailScreen(orderId: orderId);
                  },
                ),
              ],
            ),

            // Profile (protected)
            GoRoute(
              path: '/profile',
              builder: (context, state) => ProfileScreen(),
              routes: [
                GoRoute(
                  path: 'edit',
                  builder: (context, state) => EditProfileScreen(),
                ),
                GoRoute(
                  path: 'addresses',
                  builder: (context, state) => AddressListScreen(),
                ),
              ],
            ),

            // Search
            GoRoute(
              path: '/search',
              builder: (context, state) {
                final query = state.uri.queryParameters['q'] ?? '';
                return SearchScreen(initialQuery: query);
              },
            ),
          ],
        ),

        // Standalone pages (ไม่มี shell)
        GoRoute(
          path: '/onboarding',
          builder: (context, state) => OnboardingScreen(),
        ),
      ],

      errorBuilder: (context, state) => ErrorScreen(error: state.error),
    );
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      routerConfig: _router,
      title: 'E-Commerce App',
      theme: AppTheme.lightTheme,
    );
  }
}

// AppShell.dart
class AppShell extends StatelessWidget {
  final Widget child;

  const AppShell({required this.child});

  @override
  Widget build(BuildContext context) {
    final currentPath = GoRouterState.of(context).matchedLocation;
    final currentIndex = _getTabIndex(currentPath);

    return Scaffold(
      body: child,
      bottomNavigationBar: NavigationBar(
        selectedIndex: currentIndex,
        onDestinationSelected: (index) => _navigateTo(context, index),
        destinations: [
          NavigationDestination(icon: Icon(Icons.home), label: 'หน้าหลัก'),
          NavigationDestination(
              icon: Icon(Icons.grid_view), label: 'สินค้า'),
          NavigationDestination(
              icon: _CartIcon(),
              label: 'ตะกร้า'),
          NavigationDestination(icon: Icon(Icons.person), label: 'โปรไฟล์'),
        ],
      ),
    );
  }

  int _getTabIndex(String path) {
    if (path.startsWith('/products') || path.startsWith('/search')) return 1;
    if (path.startsWith('/cart')) return 2;
    if (path.startsWith('/profile') || path.startsWith('/orders')) return 3;
    return 0;
  }

  void _navigateTo(BuildContext context, int index) {
    switch (index) {
      case 0:
        context.go('/home');
        break;
      case 1:
        context.go('/products');
        break;
      case 2:
        context.go('/cart');
        break;
      case 3:
        context.go('/profile');
        break;
    }
  }
}

// CartIcon with badge
class _CartIcon extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Consumer<CartProvider>(
      builder: (context, cart, _) {
        return Badge(
          label: Text('${cart.itemCount}'),
          isLabelVisible: cart.itemCount > 0,
          child: Icon(Icons.shopping_cart),
        );
      },
    );
  }
}

// HomeScreen.dart
class HomeScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('ร้านค้า'),
        actions: [
          IconButton(
            icon: Icon(Icons.search),
            onPressed: () => context.push('/search'),
          ),
        ],
      ),
      body: SingleChildScrollView(
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // Banner
            _BannerSection(),
            // Categories
            _CategorySection(),
            // Featured Products
            _FeaturedProducts(),
          ],
        ),
      ),
    );
  }
}

class _CategorySection extends StatelessWidget {
  final categories = [
    {'id': 'electronics', 'name': 'อิเล็กทรอนิกส์', 'icon': Icons.devices},
    {'id': 'clothing', 'name': 'เสื้อผ้า', 'icon': Icons.checkroom},
    {'id': 'books', 'name': 'หนังสือ', 'icon': Icons.book},
    {'id': 'sports', 'name': 'กีฬา', 'icon': Icons.sports_soccer},
  ];

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Padding(
          padding: EdgeInsets.all(16),
          child: Text('หมวดหมู่', style: TextStyle(
            fontSize: 18,
            fontWeight: FontWeight.bold,
          )),
        ),
        SizedBox(
          height: 100,
          child: ListView.builder(
            scrollDirection: Axis.horizontal,
            padding: EdgeInsets.symmetric(horizontal: 8),
            itemCount: categories.length,
            itemBuilder: (context, index) {
              final cat = categories[index];
              return GestureDetector(
                onTap: () => context.go(
                  '/products',
                  extra: {'category': cat['id']},
                ),
                child: Container(
                  width: 80,
                  margin: EdgeInsets.symmetric(horizontal: 8),
                  child: Column(
                    mainAxisAlignment: MainAxisAlignment.center,
                    children: [
                      Icon(cat['icon'] as IconData, size: 40),
                      SizedBox(height: 8),
                      Text(cat['name'] as String,
                          textAlign: TextAlign.center,
                          style: TextStyle(fontSize: 12)),
                    ],
                  ),
                ),
              );
            },
          ),
        ),
      ],
    );
  }
}
```

### ทดสอบ Navigation

```dart
// test/navigation_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:go_router/go_router.dart';

void main() {
  group('App Router Tests', () {
    testWidgets('Home route renders correctly', (tester) async {
      final router = GoRouter(
        routes: [
          GoRoute(
            path: '/',
            builder: (context, state) => HomeScreen(),
          ),
        ],
      );

      await tester.pumpWidget(
        MaterialApp.router(routerConfig: router),
      );

      expect(find.byType(HomeScreen), findsOneWidget);
    });

    testWidgets('Protected route redirects to login', (tester) async {
      final authService = MockAuthService(isAuthenticated: false);
      final router = createRouter(authService);

      await tester.pumpWidget(
        MaterialApp.router(routerConfig: router),
      );

      // Navigate to protected route
      router.go('/profile');
      await tester.pumpAndSettle();

      // Should redirect to login
      expect(find.byType(LoginScreen), findsOneWidget);
    });
  });
}
```

---

## สรุป

GoRouter ช่วยให้การจัดการ Navigation ใน Flutter ง่ายขึ้นมาก:

1. **Declarative routing** - กำหนด routes ทั้งหมดในที่เดียว
2. **Named routes** - navigate ได้ง่ายและ type-safe
3. **Route parameters** - รองรับ path/query/extra parameters
4. **Nested navigation** - ShellRoute สำหรับ Bottom Nav
5. **Route guards** - redirect สำหรับ authentication
6. **Deep linking** - รองรับ universal links และ custom scheme
