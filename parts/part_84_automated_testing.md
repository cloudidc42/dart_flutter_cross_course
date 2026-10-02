# Part 84: Automated Testing Strategy - กลยุทธ์การทดสอบอัตโนมัติ

## บทนำ

การทดสอบที่ดีคือการลงทุนระยะยาว ส่วนนี้จะสอนกลยุทธ์การทดสอบแบบครบวงจร ตั้งแต่ unit tests ไปถึง E2E tests และการรวม CI/CD

## 1. Testing Pyramid

```
        /\
       /  \
      / E2E \        - จำนวนน้อย แต่ครอบคลุม critical flows
     /--------\
    / Integration\   - ปานกลาง ทดสอบ features ทั้งหมด
   /-------------\
  /   Unit Tests  \  - จำนวนมาก เร็ว ราคาถูก
 /-----------------\
```

### หลักการ Test Coverage

```
Unit Tests: 70-80% ของ tests ทั้งหมด
Integration Tests: 15-20%
E2E Tests: 5-10%
```

## 2. Unit Test Strategies

### Testing BLoC

```dart
// test/features/auth/presentation/bloc/auth_bloc_test.dart
import 'package:bloc_test/bloc_test.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:dartz/dartz.dart';

import 'package:myapp/core/errors/failures.dart';
import 'package:myapp/features/auth/domain/entities/user.dart';
import 'package:myapp/features/auth/domain/usecases/login_usecase.dart';
import 'package:myapp/features/auth/presentation/bloc/auth_bloc.dart';

class MockLoginUseCase extends Mock implements LoginUseCase {}

void main() {
  late AuthBloc authBloc;
  late MockLoginUseCase mockLoginUseCase;

  setUp(() {
    mockLoginUseCase = MockLoginUseCase();
    authBloc = AuthBloc(loginUseCase: mockLoginUseCase);
  });

  tearDown(() {
    authBloc.close();
  });

  const testUser = User(
    id: '1',
    email: 'test@example.com',
    name: 'Test User',
  );

  const testCredentials = LoginCredentials(
    email: 'test@example.com',
    password: 'password123',
  );

  group('AuthBloc', () {
    test('initial state is AuthInitial', () {
      expect(authBloc.state, const AuthInitial());
    });

    blocTest<AuthBloc, AuthState>(
      'emits [AuthLoading, AuthAuthenticated] when login succeeds',
      build: () {
        when(() => mockLoginUseCase(testCredentials))
            .thenAnswer((_) async => const Right(testUser));
        return authBloc;
      },
      act: (bloc) => bloc.add(AuthLoginRequested(credentials: testCredentials)),
      expect: () => [
        const AuthLoading(),
        AuthAuthenticated(user: testUser),
      ],
      verify: (_) {
        verify(() => mockLoginUseCase(testCredentials)).called(1);
      },
    );

    blocTest<AuthBloc, AuthState>(
      'emits [AuthLoading, AuthError] when login fails',
      build: () {
        when(() => mockLoginUseCase(testCredentials))
            .thenAnswer((_) async => const Left(
                  ServerFailure(message: 'Invalid credentials'),
                ));
        return authBloc;
      },
      act: (bloc) => bloc.add(AuthLoginRequested(credentials: testCredentials)),
      expect: () => [
        const AuthLoading(),
        const AuthError(message: 'Invalid credentials'),
      ],
    );
  });
}
```

### Testing Repository

```dart
// test/features/products/data/repositories/product_repository_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:dartz/dartz.dart';

import 'package:myapp/core/errors/exceptions.dart';
import 'package:myapp/core/errors/failures.dart';
import 'package:myapp/core/network/network_info.dart';
import 'package:myapp/features/products/data/datasources/product_remote_datasource.dart';
import 'package:myapp/features/products/data/datasources/product_local_datasource.dart';
import 'package:myapp/features/products/data/models/product_model.dart';
import 'package:myapp/features/products/data/repositories/product_repository_impl.dart';
import 'package:myapp/features/products/domain/entities/product.dart';

class MockRemoteDataSource extends Mock implements ProductRemoteDataSource {}
class MockLocalDataSource extends Mock implements ProductLocalDataSource {}
class MockNetworkInfo extends Mock implements NetworkInfo {}

void main() {
  late ProductRepositoryImpl repository;
  late MockRemoteDataSource mockRemoteDataSource;
  late MockLocalDataSource mockLocalDataSource;
  late MockNetworkInfo mockNetworkInfo;

  setUp(() {
    mockRemoteDataSource = MockRemoteDataSource();
    mockLocalDataSource = MockLocalDataSource();
    mockNetworkInfo = MockNetworkInfo();
    repository = ProductRepositoryImpl(
      remoteDataSource: mockRemoteDataSource,
      localDataSource: mockLocalDataSource,
      networkInfo: mockNetworkInfo,
    );
  });

  const tProductModel = ProductModel(
    id: '1',
    name: 'Test Product',
    description: 'Test Description',
    price: 99.0,
    imageUrl: 'https://example.com/image.jpg',
    stock: 10,
    categories: ['electronics'],
  );

  const tProduct = Product(
    id: '1',
    name: 'Test Product',
    description: 'Test Description',
    price: 99.0,
    imageUrl: 'https://example.com/image.jpg',
    stock: 10,
    categories: ['electronics'],
  );

  group('getProducts', () {
    group('when device is online', () {
      setUp(() {
        when(() => mockNetworkInfo.isConnected).thenAnswer((_) async => true);
      });

      test('returns remote data and caches it', () async {
        when(() => mockRemoteDataSource.getProducts())
            .thenAnswer((_) async => [tProductModel]);
        when(() => mockLocalDataSource.cacheProducts(any()))
            .thenAnswer((_) async => {});

        final result = await repository.getProducts();

        verify(() => mockRemoteDataSource.getProducts()).called(1);
        verify(() => mockLocalDataSource.cacheProducts([tProductModel]))
            .called(1);
        expect(result, equals(Right<Failure, List<Product>>([tProduct])));
      });

      test('returns ServerFailure on server error', () async {
        when(() => mockRemoteDataSource.getProducts())
            .thenThrow(const ServerException(message: 'Server error'));

        final result = await repository.getProducts();

        expect(result, const Left(ServerFailure(message: 'Server error')));
      });
    });

    group('when device is offline', () {
      setUp(() {
        when(() => mockNetworkInfo.isConnected).thenAnswer((_) async => false);
      });

      test('returns cached data', () async {
        when(() => mockLocalDataSource.getCachedProducts())
            .thenAnswer((_) async => [tProductModel]);

        final result = await repository.getProducts();

        verify(() => mockLocalDataSource.getCachedProducts()).called(1);
        verifyNever(() => mockRemoteDataSource.getProducts());
        expect(result, equals(Right<Failure, List<Product>>([tProduct])));
      });

      test('returns CacheFailure when no cache', () async {
        when(() => mockLocalDataSource.getCachedProducts())
            .thenThrow(const CacheException());

        final result = await repository.getProducts();

        expect(result, const Left(CacheFailure()));
      });
    });
  });
}
```

### Testing Widgets

```dart
// test/features/products/presentation/widgets/product_card_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:network_image_mock/network_image_mock.dart';

import 'package:myapp/features/products/domain/entities/product.dart';
import 'package:myapp/features/products/presentation/widgets/product_card.dart';

void main() {
  const testProduct = Product(
    id: '1',
    name: 'iPhone 15 Pro',
    description: 'Latest iPhone',
    price: 39900.0,
    imageUrl: 'https://example.com/iphone.jpg',
    stock: 5,
    categories: ['electronics'],
  );

  Widget buildProductCard({
    Product product = testProduct,
    VoidCallback? onTap,
    VoidCallback? onAddToCart,
  }) {
    return MaterialApp(
      home: Scaffold(
        body: ProductCard(
          product: product,
          onTap: onTap,
          onAddToCart: onAddToCart,
        ),
      ),
    );
  }

  group('ProductCard', () {
    testWidgets('shows product name and price', (tester) async {
      await mockNetworkImagesFor(() async {
        await tester.pumpWidget(buildProductCard());

        expect(find.text('iPhone 15 Pro'), findsOneWidget);
        expect(find.text('฿39,900.00'), findsOneWidget);
      });
    });

    testWidgets('shows low stock warning', (tester) async {
      const lowStockProduct = Product(
        id: '1',
        name: 'Test',
        description: '',
        price: 100,
        imageUrl: '',
        stock: 3,
        categories: [],
      );

      await mockNetworkImagesFor(() async {
        await tester.pumpWidget(buildProductCard(product: lowStockProduct));
        expect(find.text('เหลือน้อย'), findsOneWidget);
      });
    });

    testWidgets('calls onTap when tapped', (tester) async {
      bool tapped = false;

      await mockNetworkImagesFor(() async {
        await tester.pumpWidget(
          buildProductCard(onTap: () => tapped = true),
        );

        await tester.tap(find.byType(ProductCard));
        expect(tapped, isTrue);
      });
    });

    testWidgets('disables add to cart when out of stock', (tester) async {
      const outOfStockProduct = Product(
        id: '1',
        name: 'Test',
        description: '',
        price: 100,
        imageUrl: '',
        stock: 0,
        categories: [],
      );

      bool addedToCart = false;

      await mockNetworkImagesFor(() async {
        await tester.pumpWidget(
          buildProductCard(
            product: outOfStockProduct,
            onAddToCart: () => addedToCart = true,
          ),
        );

        final addButton = tester.widget<FilledButton>(
          find.byType(FilledButton),
        );
        expect(addButton.onPressed, isNull);
        expect(addedToCart, isFalse);
      });
    });

    testWidgets('matches golden file', (tester) async {
      await mockNetworkImagesFor(() async {
        await tester.pumpWidget(
          buildProductCard(),
        );

        await expectLater(
          find.byType(ProductCard),
          matchesGoldenFile('goldens/product_card.png'),
        );
      });
    });
  });
}
```

## 3. Integration Test Strategies

### Integration Test Setup

```dart
// integration_test/app_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:myapp/main_development.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  group('Full App Test', () {
    testWidgets('complete purchase flow', (tester) async {
      // Start app
      app.main();
      await tester.pumpAndSettle();

      // Login
      await _performLogin(tester);

      // Browse products
      await _browseProducts(tester);

      // Add to cart
      await _addToCart(tester);

      // Checkout
      await _checkout(tester);
    });
  });
}

Future<void> _performLogin(WidgetTester tester) async {
  // Find and fill email field
  await tester.enterText(
    find.byKey(const Key('email_field')),
    'test@example.com',
  );

  // Fill password
  await tester.enterText(
    find.byKey(const Key('password_field')),
    'password123',
  );

  // Tap login button
  await tester.tap(find.byKey(const Key('login_button')));
  await tester.pumpAndSettle(const Duration(seconds: 3));

  // Verify logged in
  expect(find.byKey(const Key('home_screen')), findsOneWidget);
}

Future<void> _browseProducts(WidgetTester tester) async {
  // ค้นหาสินค้า
  await tester.tap(find.byKey(const Key('search_button')));
  await tester.pumpAndSettle();

  await tester.enterText(
    find.byKey(const Key('search_field')),
    'iPhone',
  );
  await tester.testTextInput.receiveAction(TextInputAction.search);
  await tester.pumpAndSettle(const Duration(seconds: 2));

  expect(find.byKey(const Key('search_results')), findsOneWidget);
}

Future<void> _addToCart(WidgetTester tester) async {
  // Tap บน product แรก
  await tester.tap(find.byKey(const Key('product_item_0')));
  await tester.pumpAndSettle();

  // Add to cart
  await tester.tap(find.byKey(const Key('add_to_cart_button')));
  await tester.pumpAndSettle();

  // Verify added
  expect(find.byKey(const Key('cart_badge')), findsOneWidget);
}

Future<void> _checkout(WidgetTester tester) async {
  // ไปที่ cart
  await tester.tap(find.byKey(const Key('cart_tab')));
  await tester.pumpAndSettle();

  // Verify cart has items
  expect(find.byKey(const Key('cart_item_0')), findsOneWidget);

  // Checkout
  await tester.tap(find.byKey(const Key('checkout_button')));
  await tester.pumpAndSettle();

  expect(find.byKey(const Key('checkout_screen')), findsOneWidget);
}
```

## 4. E2E Testing with Patrol

### Patrol Setup

```yaml
# pubspec.yaml
dev_dependencies:
  patrol: ^3.5.2
  patrol_cli: ^2.3.2
```

```bash
# ติดตั้ง patrol cli
dart pub global activate patrol_cli

# Run patrol tests
patrol test --target integration_test/patrol_test.dart
```

### Patrol Test

```dart
// integration_test/patrol_test.dart
import 'package:patrol/patrol.dart';
import 'package:myapp/main_development.dart' as app;

void main() {
  patrolTest(
    'purchase flow with permissions',
    config: const PatrolTesterConfig(
      visibleTimeout: Duration(seconds: 5),
    ),
    ($) async {
      app.main();
      await $.pumpAndSettle();

      // Test with real device interactions
      await $.tap(find.text('Login'));
      await $.enterText(find.byKey(const Key('email')), 'test@example.com');
      await $.enterText(find.byKey(const Key('password')), 'pass123');
      await $.tap(find.text('Sign In'));
      await $.pumpAndSettle();

      // Handle permission dialogs (Patrol ทำได้)
      await $.native.grantPermissionWhenInUse();

      // Verify navigation
      expect($('Home'), findsOneWidget);
    },
  );

  patrolTest(
    'handles notification permission',
    ($) async {
      app.main();
      await $.pumpAndSettle();

      // Trigger notification permission request
      await $.tap(find.text('Enable Notifications'));

      // Patrol จัดการ native dialog ได้
      await $.native.grantPermissionWhenInUse();

      // Verify permission granted
      expect($('Notifications enabled'), findsOneWidget);
    },
  );
}
```

## 5. CI Test Automation

### GitHub Actions

```yaml
# .github/workflows/test.yml
name: Test

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  unit_test:
    name: Unit & Widget Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.19.0'
          cache: true
          
      - name: Install dependencies
        run: flutter pub get
        
      - name: Run tests with coverage
        run: flutter test --coverage
        
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: coverage/lcov.info
          
      - name: Check coverage threshold
        run: |
          COVERAGE=$(lcov --summary coverage/lcov.info | grep "lines" | awk '{print $2}' | tr -d '%')
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage $COVERAGE% is below threshold 80%"
            exit 1
          fi

  integration_test_android:
    name: Integration Tests (Android)
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.19.0'
          
      - name: Start Android Emulator
        uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 33
          script: |
            flutter test integration_test/ \
              --flavor development \
              -d emulator-5554
              
  golden_test:
    name: Golden Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: subosito/flutter-action@v2
      
      - name: Run golden tests
        run: flutter test --tags golden
        
      - name: Upload golden failures
        if: failure()
        uses: actions/upload-artifact@v3
        with:
          name: golden-failures
          path: test/goldens/failures/
```

## 6. Workshop: Full Test Suite

### Test Organization

```
test/
├── unit/
│   ├── core/
│   │   ├── network/
│   │   └── storage/
│   └── features/
│       ├── auth/
│       │   ├── data/
│       │   ├── domain/
│       │   └── presentation/
│       └── products/
├── widget/
│   ├── common/
│   └── features/
├── golden/
│   └── goldens/
└── helpers/
    ├── mock_factories.dart
    └── test_helpers.dart
```

### Test Helpers

```dart
// test/helpers/test_helpers.dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:get_it/get_it.dart';

import 'package:myapp/features/auth/presentation/bloc/auth_bloc.dart';

// Helper สร้าง test widget ที่มี providers ครบ
Widget buildTestWidget({
  required Widget child,
  AuthBloc? authBloc,
  ThemeData? theme,
}) {
  return MultiBlocProvider(
    providers: [
      if (authBloc != null)
        BlocProvider<AuthBloc>.value(value: authBloc),
    ],
    child: MaterialApp(
      theme: theme ?? ThemeData.light(),
      home: child,
    ),
  );
}

// Mock factory
class MockFactories {
  static AuthBloc createMockAuthBloc() {
    final bloc = MockAuthBloc();
    when(() => bloc.state).thenReturn(const AuthInitial());
    when(() => bloc.stream).thenAnswer(
      (_) => Stream.fromIterable([const AuthInitial()]),
    );
    return bloc;
  }
}

class MockAuthBloc extends MockBloc<AuthEvent, AuthState> implements AuthBloc {}
```

## สรุป

Testing Strategy:
1. **Testing Pyramid** - มาก unit tests, น้อย E2E
2. **BLoC Testing** ด้วย `bloc_test`
3. **Widget Testing** ครอบคลุม visual states
4. **Patrol** สำหรับ E2E ที่จัดการ native dialogs
5. **CI automation** บน GitHub Actions

## แบบทดสอบ

1. อธิบาย Testing Pyramid และทำไมจึงมี shape เป็น pyramid
2. ทำไม `mocktail` ถึงดีกว่า `mockito` สำหรับ null-safe Dart?
3. สร้าง test สำหรับ `CartBloc` ที่ครอบคลุม add, remove, clear items
4. เขียน CI workflow ที่ fail เมื่อ test coverage ต่ำกว่า 70%
