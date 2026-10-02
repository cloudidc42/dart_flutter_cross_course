# Part 95: Scalability Patterns

## 🎯 เป้าหมายของ Part นี้
- Feature-first folder structure
- Dependency injection ด้วย get_it
- Service locator pattern
- Repository pattern เชิงลึก
- Event-driven architecture
- Pagination และ infinite scroll

---

## 1. Feature-First Architecture

```
lib/
├── core/                    # Shared across features
│   ├── errors/
│   │   ├── exceptions.dart
│   │   └── failures.dart
│   ├── network/
│   │   ├── api_client.dart
│   │   └── network_info.dart
│   ├── utils/
│   │   ├── date_utils.dart
│   │   └── format_utils.dart
│   └── widgets/             # Shared widgets
│       ├── loading_widget.dart
│       └── error_widget.dart
│
├── features/
│   ├── auth/
│   │   ├── data/
│   │   │   ├── datasources/
│   │   │   │   ├── auth_remote_datasource.dart
│   │   │   │   └── auth_local_datasource.dart
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
│   │           └── login_form.dart
│   │
│   ├── products/
│   │   ├── data/
│   │   ├── domain/
│   │   └── presentation/
│   │
│   └── cart/
│       ├── data/
│       ├── domain/
│       └── presentation/
│
└── main.dart
```

---

## 2. Dependency Injection ด้วย get_it

```dart
// pubspec.yaml:
// get_it: ^7.6.0
// injectable: ^2.3.0  (optional - code generation)

// core/injection_container.dart
import 'package:get_it/get_it.dart';
import 'package:http/http.dart' as http;

final sl = GetIt.instance; // Service Locator

Future<void> initDependencies() async {
  // ============ External ============
  sl.registerLazySingleton(() => http.Client());
  
  // ============ Core ============
  sl.registerLazySingleton<NetworkInfo>(
    () => NetworkInfoImpl(),
  );
  
  // ============ Features - Auth ============
  
  // Datasources
  sl.registerLazySingleton<AuthRemoteDataSource>(
    () => AuthRemoteDataSourceImpl(client: sl()),
  );
  sl.registerLazySingleton<AuthLocalDataSource>(
    () => AuthLocalDataSourceImpl(),
  );
  
  // Repository
  sl.registerLazySingleton<AuthRepository>(
    () => AuthRepositoryImpl(
      remoteDataSource: sl(),
      localDataSource: sl(),
      networkInfo: sl(),
    ),
  );
  
  // Use cases
  sl.registerLazySingleton(() => LoginUseCase(sl()));
  sl.registerLazySingleton(() => LogoutUseCase(sl()));
  
  // BLoC / ViewModel (factory - สร้างใหม่ทุกครั้ง)
  sl.registerFactory(
    () => AuthBloc(
      loginUseCase: sl(),
      logoutUseCase: sl(),
    ),
  );
  
  // ============ Features - Products ============
  sl.registerLazySingleton<ProductRepository>(
    () => ProductRepositoryImpl(
      remoteDataSource: sl(),
      localDataSource: sl(),
    ),
  );
  
  sl.registerLazySingleton(() => GetProductsUseCase(sl()));
  sl.registerLazySingleton(() => GetProductDetailUseCase(sl()));
  
  sl.registerFactory(
    () => ProductBloc(
      getProducts: sl(),
      getProductDetail: sl(),
    ),
  );
}

// main.dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await initDependencies(); // ติดตั้ง DI ก่อน runApp
  runApp(const MyApp());
}

// ใช้ใน widget:
class LoginPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (_) => sl<AuthBloc>(), // ดึงจาก service locator
      child: const LoginForm(),
    );
  }
}
```

---

## 3. Repository Pattern เชิงลึก

```dart
// domain/repositories/product_repository.dart
import 'package:either_dart/either.dart';

abstract class ProductRepository {
  Future<Either<Failure, List<Product>>> getProducts({
    int page = 1,
    int limit = 20,
    String? category,
    String? searchQuery,
  });
  
  Future<Either<Failure, Product>> getProductById(String id);
  
  Future<Either<Failure, void>> saveToFavorites(String productId);
  
  Stream<List<Product>> watchFavorites();
}

// data/repositories/product_repository_impl.dart
class ProductRepositoryImpl implements ProductRepository {
  final ProductRemoteDataSource _remote;
  final ProductLocalDataSource _local;
  final NetworkInfo _network;
  
  const ProductRepositoryImpl({
    required ProductRemoteDataSource remoteDataSource,
    required ProductLocalDataSource localDataSource,
    required NetworkInfo networkInfo,
  }) : _remote = remoteDataSource,
       _local = localDataSource,
       _network = networkInfo;
  
  @override
  Future<Either<Failure, List<Product>>> getProducts({
    int page = 1,
    int limit = 20,
    String? category,
    String? searchQuery,
  }) async {
    if (await _network.isConnected) {
      try {
        final models = await _remote.getProducts(
          page: page,
          limit: limit,
          category: category,
          searchQuery: searchQuery,
        );
        
        // Cache ใน local storage
        if (page == 1) {
          await _local.cacheProducts(models);
        }
        
        return Right(models.map((m) => m.toEntity()).toList());
      } on ServerException catch (e) {
        return Left(ServerFailure(e.message));
      }
    } else {
      // Offline: ใช้ cached data
      try {
        final cached = await _local.getCachedProducts();
        return Right(cached.map((m) => m.toEntity()).toList());
      } on CacheException {
        return const Left(CacheFailure('ไม่มีข้อมูล cache'));
      }
    }
  }
  
  @override
  Future<Either<Failure, Product>> getProductById(String id) async {
    try {
      // ลอง local ก่อน:
      final cached = await _local.getProductById(id);
      if (cached != null) {
        return Right(cached.toEntity());
      }
      
      // ถ้าไม่มี cache:
      if (await _network.isConnected) {
        final model = await _remote.getProductById(id);
        await _local.cacheProduct(model);
        return Right(model.toEntity());
      }
      
      return const Left(NetworkFailure('ไม่มีการเชื่อมต่อ'));
    } catch (e) {
      return Left(UnknownFailure(e.toString()));
    }
  }
  
  @override
  Future<Either<Failure, void>> saveToFavorites(String productId) async {
    try {
      await _local.saveFavorite(productId);
      return const Right(null);
    } on CacheException catch (e) {
      return Left(CacheFailure(e.message));
    }
  }
  
  @override
  Stream<List<Product>> watchFavorites() {
    return _local.watchFavorites().map(
      (models) => models.map((m) => m.toEntity()).toList(),
    );
  }
}
```

---

## 4. Pagination Pattern

```dart
// Pagination state:
class PaginationState<T> {
  final List<T> items;
  final bool isLoading;
  final bool hasMore;
  final String? error;
  final int currentPage;
  
  const PaginationState({
    this.items = const [],
    this.isLoading = false,
    this.hasMore = true,
    this.error,
    this.currentPage = 0,
  });
  
  bool get isEmpty => items.isEmpty && !isLoading;
  bool get canLoadMore => hasMore && !isLoading;
  
  PaginationState<T> copyWith({
    List<T>? items,
    bool? isLoading,
    bool? hasMore,
    String? error,
    int? currentPage,
  }) {
    return PaginationState<T>(
      items: items ?? this.items,
      isLoading: isLoading ?? this.isLoading,
      hasMore: hasMore ?? this.hasMore,
      error: error ?? this.error,
      currentPage: currentPage ?? this.currentPage,
    );
  }
}

// Pagination Notifier:
class ProductListNotifier extends StateNotifier<PaginationState<Product>> {
  final GetProductsUseCase _getProducts;
  static const _pageSize = 20;
  
  ProductListNotifier(this._getProducts) 
      : super(const PaginationState());
  
  Future<void> loadInitial() async {
    state = const PaginationState(isLoading: true);
    await _fetchPage(1, replace: true);
  }
  
  Future<void> loadMore() async {
    if (!state.canLoadMore) return;
    
    state = state.copyWith(isLoading: true);
    await _fetchPage(state.currentPage + 1, replace: false);
  }
  
  Future<void> refresh() async {
    await loadInitial();
  }
  
  Future<void> _fetchPage(int page, {required bool replace}) async {
    final result = await _getProducts(
      page: page,
      limit: _pageSize,
    );
    
    result.fold(
      (failure) => state = state.copyWith(
        isLoading: false,
        error: failure.message,
      ),
      (products) => state = state.copyWith(
        items: replace ? products : [...state.items, ...products],
        isLoading: false,
        hasMore: products.length >= _pageSize,
        currentPage: page,
        error: null,
      ),
    );
  }
}

// Flutter widget with infinite scroll:
class ProductListPage extends ConsumerWidget {
  const ProductListPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final paginationState = ref.watch(productListProvider);
    
    return Scaffold(
      body: RefreshIndicator(
        onRefresh: () => ref.read(productListProvider.notifier).refresh(),
        child: CustomScrollView(
          slivers: [
            // Error banner:
            if (paginationState.error != null)
              SliverToBoxAdapter(
                child: Container(
                  color: Colors.red.shade100,
                  padding: const EdgeInsets.all(8),
                  child: Text(paginationState.error!),
                ),
              ),
            
            // Products:
            SliverList(
              delegate: SliverChildBuilderDelegate(
                (context, index) {
                  // Last item - trigger load more
                  if (index == paginationState.items.length - 3) {
                    ref.read(productListProvider.notifier).loadMore();
                  }
                  
                  return ProductCard(
                    product: paginationState.items[index],
                  );
                },
                childCount: paginationState.items.length,
              ),
            ),
            
            // Loading indicator:
            if (paginationState.isLoading)
              const SliverToBoxAdapter(
                child: Padding(
                  padding: EdgeInsets.all(16),
                  child: Center(child: CircularProgressIndicator()),
                ),
              ),
            
            // End of list:
            if (!paginationState.hasMore && paginationState.items.isNotEmpty)
              const SliverToBoxAdapter(
                child: Padding(
                  padding: EdgeInsets.all(16),
                  child: Center(child: Text('โหลดครบแล้ว')),
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

## 5. Event-Driven Architecture

```dart
// Event bus สำหรับ cross-feature communication:
class AppEventBus {
  static final AppEventBus _instance = AppEventBus._();
  static AppEventBus get instance => _instance;
  AppEventBus._();
  
  final _controller = StreamController<AppEvent>.broadcast();
  Stream<AppEvent> get events => _controller.stream;
  
  void emit(AppEvent event) {
    _controller.add(event);
  }
  
  Stream<T> on<T extends AppEvent>() {
    return events.whereType<T>();
  }
  
  void dispose() {
    _controller.close();
  }
}

// Events:
abstract class AppEvent {}

class UserLoggedIn extends AppEvent {
  final String userId;
  UserLoggedIn(this.userId);
}

class UserLoggedOut extends AppEvent {}

class CartUpdated extends AppEvent {
  final int itemCount;
  CartUpdated(this.itemCount);
}

class NotificationReceived extends AppEvent {
  final String title;
  final String body;
  NotificationReceived(this.title, this.body);
}

// Usage:
class AuthViewModel extends ChangeNotifier {
  final AppEventBus _eventBus = AppEventBus.instance;
  
  Future<void> login(String email, String password) async {
    // ... authenticate
    _eventBus.emit(UserLoggedIn('user123'));
  }
  
  Future<void> logout() async {
    // ... clear session
    _eventBus.emit(UserLoggedOut());
  }
}

class CartViewModel extends ChangeNotifier {
  final AppEventBus _eventBus = AppEventBus.instance;
  late final StreamSubscription _sub;
  
  CartViewModel() {
    // ฟัง logout event เพื่อ clear cart:
    _sub = _eventBus.on<UserLoggedOut>().listen((_) {
      clear();
    });
  }
  
  @override
  void dispose() {
    _sub.cancel();
    super.dispose();
  }
  
  void addItem(Product p) {
    // ... add item
    _eventBus.emit(CartUpdated(itemCount));
  }
  
  void clear() {
    // ... clear
    notifyListeners();
  }
  
  int get itemCount => 0; // placeholder
}
```

---

## 6. สรุป Part 95

สิ่งที่เรียนรู้:
- ✅ Feature-first folder structure
- ✅ Dependency injection ด้วย get_it
- ✅ Repository pattern with Either (functional error handling)
- ✅ Pagination pattern
- ✅ Infinite scroll implementation
- ✅ Event-driven architecture ด้วย event bus

---

## ➡️ Part ถัดไป
**Part 96: Real-World Project - E-Commerce App (Complete)**
