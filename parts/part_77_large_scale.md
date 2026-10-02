# Part 77: Large-Scale App Architecture - สถาปัตยกรรมแอปขนาดใหญ่

## บทนำ

เมื่อแอปเติบโตขึ้น การจัดการโค้ดให้เป็นระเบียบและบำรุงรักษาได้ง่ายกลายเป็นความท้าทาย ส่วนนี้จะสอนการออกแบบสถาปัตยกรรมสำหรับแอปขนาดใหญ่ที่มีทีมพัฒนาหลายคน

## 1. Feature-first Folder Structure

### โครงสร้างที่แนะนำ

```
lib/
├── app/
│   ├── app.dart                # MaterialApp configuration
│   ├── router.dart             # App routing
│   └── di.dart                 # Dependency injection setup
├── core/
│   ├── constants/
│   │   ├── app_constants.dart
│   │   └── api_constants.dart
│   ├── errors/
│   │   ├── failures.dart
│   │   └── exceptions.dart
│   ├── extensions/
│   │   ├── string_extensions.dart
│   │   └── date_extensions.dart
│   ├── network/
│   │   ├── http_client.dart
│   │   └── network_info.dart
│   ├── storage/
│   │   ├── local_storage.dart
│   │   └── secure_storage.dart
│   └── theme/
│       ├── app_theme.dart
│       └── app_colors.dart
├── features/
│   ├── auth/
│   │   ├── data/
│   │   │   ├── datasources/
│   │   │   │   ├── auth_remote_data_source.dart
│   │   │   │   └── auth_local_data_source.dart
│   │   │   ├── models/
│   │   │   │   └── user_model.dart
│   │   │   └── repositories/
│   │   │       └── auth_repository_impl.dart
│   │   ├── domain/
│   │   │   ├── entities/
│   │   │   │   └── user.dart
│   │   │   ├── repositories/
│   │   │   │   └── auth_repository.dart
│   │   │   └── usecases/
│   │   │       ├── login_usecase.dart
│   │   │       └── logout_usecase.dart
│   │   └── presentation/
│   │       ├── bloc/
│   │       │   ├── auth_bloc.dart
│   │       │   ├── auth_event.dart
│   │       │   └── auth_state.dart
│   │       ├── pages/
│   │       │   ├── login_page.dart
│   │       │   └── register_page.dart
│   │       └── widgets/
│   │           ├── login_form.dart
│   │           └── social_login_buttons.dart
│   ├── products/
│   │   └── ...
│   └── cart/
│       └── ...
└── shared/
    ├── widgets/
    │   ├── buttons/
    │   ├── cards/
    │   └── loading/
    └── utils/
```

## 2. Module System

### Feature Module Interface

```dart
// lib/core/module/feature_module.dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';
import 'package:get_it/get_it.dart';

abstract class FeatureModule {
  // Module name (ใช้สำหรับ logging และ debugging)
  String get name;

  // Register dependencies ใน DI container
  Future<void> register(GetIt sl);

  // สร้าง routes สำหรับ module นี้
  List<RouteBase> get routes;

  // Initialize module (เรียกตอน app start)
  Future<void> initialize();

  // Dispose resources เมื่อ module ไม่ได้ใช้
  Future<void> dispose();
}
```

### Auth Feature Module

```dart
// lib/features/auth/auth_module.dart
import 'package:get_it/get_it.dart';
import 'package:go_router/go_router.dart';
import '../../core/module/feature_module.dart';
import 'data/datasources/auth_remote_data_source.dart';
import 'data/datasources/auth_local_data_source.dart';
import 'data/repositories/auth_repository_impl.dart';
import 'domain/repositories/auth_repository.dart';
import 'domain/usecases/login_usecase.dart';
import 'domain/usecases/logout_usecase.dart';
import 'presentation/bloc/auth_bloc.dart';
import 'presentation/pages/login_page.dart';

class AuthModule implements FeatureModule {
  @override
  String get name => 'auth';

  @override
  Future<void> register(GetIt sl) async {
    // Data sources
    sl.registerLazySingleton<AuthRemoteDataSource>(
      () => AuthRemoteDataSourceImpl(httpClient: sl()),
    );

    sl.registerLazySingleton<AuthLocalDataSource>(
      () => AuthLocalDataSourceImpl(secureStorage: sl()),
    );

    // Repository
    sl.registerLazySingleton<AuthRepository>(
      () => AuthRepositoryImpl(
        remoteDataSource: sl(),
        localDataSource: sl(),
      ),
    );

    // Use cases
    sl.registerLazySingleton(() => LoginUseCase(repository: sl()));
    sl.registerLazySingleton(() => LogoutUseCase(repository: sl()));

    // BLoC
    sl.registerFactory(
      () => AuthBloc(
        loginUseCase: sl(),
        logoutUseCase: sl(),
      ),
    );
  }

  @override
  List<RouteBase> get routes => [
        GoRoute(
          path: '/login',
          name: 'login',
          builder: (context, state) => const LoginPage(),
        ),
        GoRoute(
          path: '/register',
          name: 'register',
          builder: (context, state) => const RegisterPage(),
        ),
      ];

  @override
  Future<void> initialize() async {
    print('Auth module initialized');
  }

  @override
  Future<void> dispose() async {
    print('Auth module disposed');
  }
}
```

## 3. Dependency Injection Architecture

### DI Container Setup

```dart
// lib/app/di.dart
import 'package:get_it/get_it.dart';
import 'package:dio/dio.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:shared_preferences/shared_preferences.dart';
import '../core/network/http_client.dart';
import '../core/storage/local_storage.dart';
import '../core/storage/secure_storage.dart';
import '../features/auth/auth_module.dart';
import '../features/products/products_module.dart';
import '../features/cart/cart_module.dart';

final GetIt sl = GetIt.instance;

class DependencyInjection {
  static final List<FeatureModule> _modules = [
    AuthModule(),
    ProductsModule(),
    CartModule(),
  ];

  static Future<void> setup() async {
    await _registerCore();
    await _registerModules();
  }

  static Future<void> _registerCore() async {
    // External dependencies
    final sharedPreferences = await SharedPreferences.getInstance();
    sl.registerLazySingleton(() => sharedPreferences);
    sl.registerLazySingleton(() => const FlutterSecureStorage());

    // Network
    sl.registerLazySingleton<Dio>(() {
      final dio = Dio();
      dio.options.baseUrl = 'https://api.example.com';
      dio.options.connectTimeout = const Duration(seconds: 10);
      dio.options.receiveTimeout = const Duration(seconds: 30);
      dio.interceptors.add(LogInterceptor(
        requestBody: true,
        responseBody: true,
      ));
      return dio;
    });

    sl.registerLazySingleton<HttpClient>(
      () => DioHttpClient(dio: sl()),
    );

    // Storage
    sl.registerLazySingleton<LocalStorage>(
      () => SharedPreferencesStorage(prefs: sl()),
    );

    sl.registerLazySingleton<SecureStorage>(
      () => FlutterSecureStorageWrapper(storage: sl()),
    );
  }

  static Future<void> _registerModules() async {
    for (final module in _modules) {
      await module.register(sl);
      await module.initialize();
      print('Module ${module.name} registered');
    }
  }

  static List<RouteBase> get allRoutes =>
      _modules.expand((m) => m.routes).toList();
}
```

### Clean Architecture Layers

```dart
// lib/features/products/domain/entities/product.dart
import 'package:equatable/equatable.dart';

class Product extends Equatable {
  final String id;
  final String name;
  final String description;
  final double price;
  final String imageUrl;
  final int stock;
  final List<String> categories;

  const Product({
    required this.id,
    required this.name,
    required this.description,
    required this.price,
    required this.imageUrl,
    required this.stock,
    required this.categories,
  });

  bool get isAvailable => stock > 0;
  bool get isLowStock => stock > 0 && stock <= 5;

  @override
  List<Object?> get props => [id, name, price, stock];
}
```

```dart
// lib/features/products/domain/repositories/product_repository.dart
import 'package:dartz/dartz.dart';
import '../../../../core/errors/failures.dart';
import '../entities/product.dart';

abstract class ProductRepository {
  Future<Either<Failure, List<Product>>> getProducts({
    String? category,
    String? searchQuery,
    int page = 1,
    int limit = 20,
  });

  Future<Either<Failure, Product>> getProduct(String id);

  Future<Either<Failure, List<Product>>> getFeaturedProducts();

  Future<Either<Failure, void>> addToFavorites(String productId);

  Future<Either<Failure, void>> removeFromFavorites(String productId);
}
```

```dart
// lib/features/products/domain/usecases/get_products_usecase.dart
import 'package:dartz/dartz.dart';
import 'package:equatable/equatable.dart';
import '../../../../core/errors/failures.dart';
import '../entities/product.dart';
import '../repositories/product_repository.dart';

class GetProductsParams extends Equatable {
  final String? category;
  final String? searchQuery;
  final int page;
  final int limit;

  const GetProductsParams({
    this.category,
    this.searchQuery,
    this.page = 1,
    this.limit = 20,
  });

  @override
  List<Object?> get props => [category, searchQuery, page, limit];
}

class GetProductsUseCase {
  final ProductRepository _repository;

  const GetProductsUseCase({required ProductRepository repository})
      : _repository = repository;

  Future<Either<Failure, List<Product>>> call(GetProductsParams params) {
    return _repository.getProducts(
      category: params.category,
      searchQuery: params.searchQuery,
      page: params.page,
      limit: params.limit,
    );
  }
}
```

## 4. Code Generation

### Freezed สำหรับ Immutable Classes

```dart
// lib/features/products/presentation/bloc/product_state.dart
import 'package:freezed_annotation/freezed_annotation.dart';
import '../../domain/entities/product.dart';

part 'product_state.freezed.dart';

@freezed
class ProductState with _$ProductState {
  const factory ProductState.initial() = _Initial;
  
  const factory ProductState.loading() = _Loading;
  
  const factory ProductState.loaded({
    required List<Product> products,
    required bool hasMore,
    int page = 1,
  }) = _Loaded;
  
  const factory ProductState.error({
    required String message,
  }) = _Error;
}
```

### Build Runner Commands

```bash
# Generate code
flutter pub run build_runner build

# Watch for changes
flutter pub run build_runner watch

# Clean และ rebuild
flutter pub run build_runner build --delete-conflicting-outputs
```

### JSON Serialization

```dart
// lib/features/products/data/models/product_model.dart
import 'package:json_annotation/json_annotation.dart';
import '../../domain/entities/product.dart';

part 'product_model.g.dart';

@JsonSerializable()
class ProductModel extends Product {
  const ProductModel({
    required super.id,
    required super.name,
    required super.description,
    required super.price,
    @JsonKey(name: 'image_url') required super.imageUrl,
    required super.stock,
    required super.categories,
  });

  factory ProductModel.fromJson(Map<String, dynamic> json) =>
      _$ProductModelFromJson(json);

  Map<String, dynamic> toJson() => _$ProductModelToJson(this);

  factory ProductModel.fromEntity(Product product) => ProductModel(
        id: product.id,
        name: product.name,
        description: product.description,
        price: product.price,
        imageUrl: product.imageUrl,
        stock: product.stock,
        categories: product.categories,
      );
}
```

## 5. Workshop: E-Commerce App Structure

### App Entry Point

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:get_it/get_it.dart';
import 'app/di.dart';
import 'app/app.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // Setup dependency injection
  await DependencyInjection.setup();
  
  runApp(const ECommerceApp());
}
```

```dart
// lib/app/app.dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:go_router/go_router.dart';
import 'di.dart';
import 'router.dart';
import '../core/theme/app_theme.dart';
import '../features/auth/presentation/bloc/auth_bloc.dart';

class ECommerceApp extends StatelessWidget {
  const ECommerceApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MultiBlocProvider(
      providers: [
        BlocProvider<AuthBloc>(
          create: (_) => sl<AuthBloc>()..add(const AuthCheckStatusEvent()),
        ),
      ],
      child: MaterialApp.router(
        title: 'E-Commerce App',
        theme: AppTheme.light,
        darkTheme: AppTheme.dark,
        routerConfig: AppRouter.router,
      ),
    );
  }
}
```

### App Router

```dart
// lib/app/router.dart
import 'package:go_router/go_router.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import '../features/auth/presentation/bloc/auth_bloc.dart';
import '../features/auth/presentation/pages/login_page.dart';
import '../features/products/presentation/pages/product_list_page.dart';
import '../features/cart/presentation/pages/cart_page.dart';
import 'di.dart';

class AppRouter {
  static GoRouter get router => GoRouter(
        initialLocation: '/',
        redirect: (context, state) {
          final authBloc = sl<AuthBloc>();
          final isAuthenticated = authBloc.state.maybeWhen(
            authenticated: (_) => true,
            orElse: () => false,
          );

          final isLoginPage = state.matchedLocation == '/login';

          if (!isAuthenticated && !isLoginPage) return '/login';
          if (isAuthenticated && isLoginPage) return '/';

          return null;
        },
        routes: [
          GoRoute(
            path: '/',
            name: 'home',
            builder: (context, state) => const ProductListPage(),
            routes: [
              GoRoute(
                path: 'product/:id',
                name: 'product-detail',
                builder: (context, state) => ProductDetailPage(
                  productId: state.pathParameters['id']!,
                ),
              ),
            ],
          ),
          GoRoute(
            path: '/cart',
            name: 'cart',
            builder: (context, state) => const CartPage(),
          ),
          // เพิ่ม routes จาก modules
          ...DependencyInjection.allRoutes,
        ],
      );
}
```

## สรุป

Large-scale app architecture:
1. **Feature-first** แยก code ตาม business domain
2. **Clean Architecture** แยก concerns ชัดเจน
3. **DI** ทำให้ testing และ flexibility ดีขึ้น
4. **Code generation** ลด boilerplate

## แบบทดสอบ

1. อธิบาย Clean Architecture 3 layers และหน้าที่ของแต่ละ layer
2. ทำไม `Either<Failure, T>` จึงดีกว่าการ throw exceptions ใน use cases?
3. สร้าง `OrderModule` ตาม pattern ที่เรียนมา
4. อธิบาย trade-offs ระหว่าง feature-first และ layer-first folder structure
