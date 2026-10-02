# Part 78: Monorepo Setup - การตั้งค่า Monorepo

## บทนำ

Monorepo คือการเก็บโค้ดหลายๆ project ไว้ใน repository เดียว ช่วยให้ทีมแชร์โค้ดได้ง่าย จัดการ dependency ร่วมกัน และทำ CI/CD ได้สะดวก `melos` เป็น tool ยอดนิยมสำหรับจัดการ Dart/Flutter monorepo

## 1. Melos Package

### ติดตั้ง Melos

```bash
# ติดตั้งแบบ global
dart pub global activate melos

# หรือเพิ่มใน dev dependencies
```

```yaml
# pubspec.yaml (root)
name: my_monorepo
environment:
  sdk: '>=3.0.0 <4.0.0'

dev_dependencies:
  melos: ^6.1.0
```

### melos.yaml Configuration

```yaml
# melos.yaml (root ของ monorepo)
name: my_monorepo

packages:
  - apps/*
  - packages/*

sdkPath: auto

command:
  version:
    # Conventional commits สำหรับ automatic versioning
    conventional_commits: true
    
  bootstrap:
    # Dependency ที่ใช้ร่วมกัน
    environment:
      sdk: '>=3.0.0 <4.0.0'
      flutter: '>=3.16.0'
    dependencies:
      flutter_bloc: ^8.1.3
      go_router: ^13.0.0
      get_it: ^7.6.4
      dartz: ^0.10.1
    dev_dependencies:
      flutter_test:
        sdk: flutter
      bloc_test: ^9.1.5
      mocktail: ^1.0.1

scripts:
  # Lint tất cả packages
  lint:
    run: melos exec -- flutter analyze
    description: Analyze tất cả packages

  # Test ทุก packages
  test:
    run: melos exec -- flutter test
    description: Run tests ทุก packages
    packageFilters:
      dirExists: test

  # Build ทุก apps
  build:
    run: melos exec -- flutter build apk --release
    description: Build ทุก apps
    packageFilters:
      flutter: true
      dirExists: android

  # Format code
  format:
    run: melos exec -- dart format .
    description: Format code ทุก packages

  # Generate code
  generate:
    run: |
      melos exec -- flutter pub run build_runner build \
        --delete-conflicting-outputs
    description: Generate code ทุก packages
    packageFilters:
      dependsOn: "build_runner"

  # Clean
  clean:
    run: |
      melos exec -- flutter clean
    description: Clean ทุก packages
```

## 2. Multiple Packages in One Repo

### โครงสร้าง Monorepo

```
my_monorepo/
├── melos.yaml
├── pubspec.yaml
├── apps/
│   ├── app_customer/         # แอปสำหรับลูกค้า
│   │   ├── lib/
│   │   ├── pubspec.yaml
│   │   └── ...
│   ├── app_admin/            # แอปสำหรับ admin
│   │   ├── lib/
│   │   ├── pubspec.yaml
│   │   └── ...
│   └── app_driver/           # แอปสำหรับ delivery driver
│       ├── lib/
│       ├── pubspec.yaml
│       └── ...
├── packages/
│   ├── core/                 # Core utilities
│   │   ├── lib/
│   │   └── pubspec.yaml
│   ├── ui_kit/               # Shared UI components
│   │   ├── lib/
│   │   └── pubspec.yaml
│   ├── auth/                 # Authentication package
│   │   ├── lib/
│   │   └── pubspec.yaml
│   ├── api_client/           # HTTP client
│   │   ├── lib/
│   │   └── pubspec.yaml
│   └── analytics/            # Analytics
│       ├── lib/
│       └── pubspec.yaml
└── tools/
    ├── scripts/
    └── ci/
```

## 3. Shared Packages

### Core Package

```yaml
# packages/core/pubspec.yaml
name: core
description: Core utilities and shared code

environment:
  sdk: '>=3.0.0 <4.0.0'

dependencies:
  equatable: ^2.0.5
  dartz: ^0.10.1
  logger: ^2.1.0

dev_dependencies:
  test: ^1.24.0
  mocktail: ^1.0.1
```

```dart
// packages/core/lib/core.dart
library core;

export 'src/errors/failures.dart';
export 'src/errors/exceptions.dart';
export 'src/network/network_info.dart';
export 'src/storage/local_storage.dart';
export 'src/utils/either_extensions.dart';
export 'src/utils/logger.dart';
```

```dart
// packages/core/lib/src/errors/failures.dart
import 'package:equatable/equatable.dart';

abstract class Failure extends Equatable {
  final String message;
  final int? code;

  const Failure({
    required this.message,
    this.code,
  });

  @override
  List<Object?> get props => [message, code];
}

class ServerFailure extends Failure {
  const ServerFailure({
    required super.message,
    super.code,
  });
}

class NetworkFailure extends Failure {
  const NetworkFailure({
    super.message = 'ไม่มีการเชื่อมต่ออินเทอร์เน็ต',
    super.code,
  });
}

class CacheFailure extends Failure {
  const CacheFailure({
    super.message = 'เกิดข้อผิดพลาดในการเข้าถึงข้อมูล cache',
    super.code,
  });
}

class AuthFailure extends Failure {
  const AuthFailure({
    super.message = 'กรุณาเข้าสู่ระบบอีกครั้ง',
    super.code,
  });
}

class ValidationFailure extends Failure {
  final Map<String, String>? fieldErrors;

  const ValidationFailure({
    required super.message,
    this.fieldErrors,
    super.code,
  });

  @override
  List<Object?> get props => [...super.props, fieldErrors];
}
```

### UI Kit Package

```yaml
# packages/ui_kit/pubspec.yaml
name: ui_kit
description: Shared UI components for all apps

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: '>=3.16.0'

dependencies:
  flutter:
    sdk: flutter
  cached_network_image: ^3.3.1
  shimmer: ^3.0.0

dev_dependencies:
  flutter_test:
    sdk: flutter
```

```dart
// packages/ui_kit/lib/ui_kit.dart
library ui_kit;

export 'src/buttons/app_button.dart';
export 'src/cards/product_card.dart';
export 'src/loading/shimmer_loading.dart';
export 'src/inputs/app_text_field.dart';
export 'src/theme/app_theme.dart';
export 'src/theme/app_colors.dart';
export 'src/theme/app_text_styles.dart';
```

```dart
// packages/ui_kit/lib/src/buttons/app_button.dart
import 'package:flutter/material.dart';

enum AppButtonVariant {
  primary,
  secondary,
  outlined,
  text,
  destructive,
}

enum AppButtonSize {
  small,
  medium,
  large,
}

class AppButton extends StatelessWidget {
  final String label;
  final VoidCallback? onPressed;
  final AppButtonVariant variant;
  final AppButtonSize size;
  final Widget? icon;
  final bool isLoading;
  final bool isFullWidth;

  const AppButton({
    super.key,
    required this.label,
    this.onPressed,
    this.variant = AppButtonVariant.primary,
    this.size = AppButtonSize.medium,
    this.icon,
    this.isLoading = false,
    this.isFullWidth = false,
  });

  @override
  Widget build(BuildContext context) {
    final child = isLoading
        ? SizedBox(
            width: 20,
            height: 20,
            child: CircularProgressIndicator(
              strokeWidth: 2,
              color: _foregroundColor(context),
            ),
          )
        : icon != null
            ? Row(
                mainAxisSize: MainAxisSize.min,
                children: [
                  icon!,
                  const SizedBox(width: 8),
                  Text(label),
                ],
              )
            : Text(label);

    Widget button;

    switch (variant) {
      case AppButtonVariant.primary:
        button = FilledButton(
          onPressed: isLoading ? null : onPressed,
          style: _buttonStyle(context),
          child: child,
        );
        break;
      case AppButtonVariant.secondary:
        button = FilledButton.tonal(
          onPressed: isLoading ? null : onPressed,
          style: _buttonStyle(context),
          child: child,
        );
        break;
      case AppButtonVariant.outlined:
        button = OutlinedButton(
          onPressed: isLoading ? null : onPressed,
          style: _buttonStyle(context),
          child: child,
        );
        break;
      case AppButtonVariant.text:
        button = TextButton(
          onPressed: isLoading ? null : onPressed,
          style: _buttonStyle(context),
          child: child,
        );
        break;
      case AppButtonVariant.destructive:
        button = FilledButton(
          onPressed: isLoading ? null : onPressed,
          style: _buttonStyle(context).copyWith(
            backgroundColor: WidgetStatePropertyAll(
              Theme.of(context).colorScheme.error,
            ),
          ),
          child: child,
        );
        break;
    }

    if (isFullWidth) {
      return SizedBox(
        width: double.infinity,
        child: button,
      );
    }

    return button;
  }

  ButtonStyle _buttonStyle(BuildContext context) {
    double paddingH;
    double paddingV;
    double fontSize;

    switch (size) {
      case AppButtonSize.small:
        paddingH = 12;
        paddingV = 6;
        fontSize = 12;
        break;
      case AppButtonSize.medium:
        paddingH = 20;
        paddingV = 10;
        fontSize = 14;
        break;
      case AppButtonSize.large:
        paddingH = 28;
        paddingV = 14;
        fontSize = 16;
        break;
    }

    return ButtonStyle(
      padding: WidgetStatePropertyAll(
        EdgeInsets.symmetric(horizontal: paddingH, vertical: paddingV),
      ),
      textStyle: WidgetStatePropertyAll(
        TextStyle(fontSize: fontSize, fontWeight: FontWeight.w600),
      ),
    );
  }

  Color _foregroundColor(BuildContext context) {
    switch (variant) {
      case AppButtonVariant.primary:
      case AppButtonVariant.destructive:
        return Theme.of(context).colorScheme.onPrimary;
      case AppButtonVariant.secondary:
        return Theme.of(context).colorScheme.onSecondaryContainer;
      case AppButtonVariant.outlined:
      case AppButtonVariant.text:
        return Theme.of(context).colorScheme.primary;
    }
  }
}
```

### API Client Package

```yaml
# packages/api_client/pubspec.yaml
name: api_client
description: HTTP client for all apps

dependencies:
  core:
    path: ../core
  dio: ^5.4.0
  pretty_dio_logger: ^1.3.1
  retrofit: ^4.1.0
  json_annotation: ^4.8.1

dev_dependencies:
  retrofit_generator: ^8.1.0
  json_serializable: ^6.7.1
  build_runner: ^2.4.8
```

```dart
// packages/api_client/lib/src/api_client.dart
import 'package:dio/dio.dart';
import 'package:core/core.dart';
import 'package:dartz/dartz.dart';

class ApiClient {
  final Dio _dio;

  ApiClient({required String baseUrl, String? authToken}) : _dio = Dio() {
    _dio.options = BaseOptions(
      baseUrl: baseUrl,
      connectTimeout: const Duration(seconds: 10),
      receiveTimeout: const Duration(seconds: 30),
      headers: {
        'Content-Type': 'application/json',
        if (authToken != null) 'Authorization': 'Bearer $authToken',
      },
    );

    _dio.interceptors.addAll([
      _AuthInterceptor(),
      _ErrorInterceptor(),
      if (const bool.fromEnvironment('DEBUG'))
        LogInterceptor(
          requestBody: true,
          responseBody: true,
        ),
    ]);
  }

  Future<Either<Failure, T>> get<T>(
    String path, {
    Map<String, dynamic>? queryParameters,
    T Function(dynamic)? fromJson,
  }) async {
    try {
      final response = await _dio.get(
        path,
        queryParameters: queryParameters,
      );

      final data = fromJson != null ? fromJson(response.data) : response.data;
      return Right(data as T);
    } on DioException catch (e) {
      return Left(_handleDioError(e));
    }
  }

  Future<Either<Failure, T>> post<T>(
    String path, {
    dynamic data,
    T Function(dynamic)? fromJson,
  }) async {
    try {
      final response = await _dio.post(path, data: data);
      final result = fromJson != null ? fromJson(response.data) : response.data;
      return Right(result as T);
    } on DioException catch (e) {
      return Left(_handleDioError(e));
    }
  }

  Failure _handleDioError(DioException e) {
    switch (e.type) {
      case DioExceptionType.connectionTimeout:
      case DioExceptionType.receiveTimeout:
        return const NetworkFailure(message: 'หมดเวลาการเชื่อมต่อ');
      case DioExceptionType.connectionError:
        return const NetworkFailure();
      case DioExceptionType.badResponse:
        final statusCode = e.response?.statusCode;
        if (statusCode == 401) {
          return const AuthFailure();
        }
        if (statusCode == 422) {
          final errors = e.response?.data['errors'] as Map<String, dynamic>?;
          return ValidationFailure(
            message: e.response?.data['message'] ?? 'ข้อมูลไม่ถูกต้อง',
            fieldErrors: errors?.map(
              (key, value) => MapEntry(key, value.toString()),
            ),
          );
        }
        return ServerFailure(
          message: e.response?.data['message'] ?? 'เกิดข้อผิดพลาดบน server',
          code: statusCode,
        );
      default:
        return ServerFailure(message: e.message ?? 'เกิดข้อผิดพลาดที่ไม่ทราบสาเหตุ');
    }
  }
}

class _AuthInterceptor extends Interceptor {
  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    if (err.response?.statusCode == 401) {
      // Auto refresh token หรือ redirect ไป login
    }
    handler.next(err);
  }
}

class _ErrorInterceptor extends Interceptor {
  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    // Log errors
    AppLogger.error('API Error: ${err.message}', err.error);
    handler.next(err);
  }
}
```

## 4. Workshop: Monorepo with 3 Apps

### Customer App

```yaml
# apps/app_customer/pubspec.yaml
name: app_customer
description: Customer mobile app

dependencies:
  flutter:
    sdk: flutter
  core:
    path: ../../packages/core
  ui_kit:
    path: ../../packages/ui_kit
  api_client:
    path: ../../packages/api_client
  auth:
    path: ../../packages/auth
  flutter_bloc: ^8.1.3
  go_router: ^13.0.0
```

### Melos Commands

```bash
# Bootstrap ทุก packages (install dependencies)
melos bootstrap

# หรือ melos bs

# Run script ทุก packages
melos run lint
melos run test
melos run generate

# Run command เฉพาะ package
melos exec --scope=app_customer -- flutter run

# List packages
melos list

# Show dependency graph
melos list --graph
```

### CI/CD Integration

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.19.0'
          
      - name: Install Melos
        run: dart pub global activate melos
        
      - name: Bootstrap
        run: melos bootstrap
        
      - name: Lint
        run: melos run lint
        
      - name: Test
        run: melos run test
        
      - name: Build Android
        run: melos exec --scope=app_customer -- flutter build apk --release
```

## สรุป

Monorepo ด้วย Melos:
1. **แชร์โค้ด** ได้ง่ายระหว่าง apps
2. **จัดการ dependencies** ร่วมกัน
3. **Atomic changes** แก้ชั้นเดียวกระทบทุก apps
4. **Consistent tooling** ผ่าน melos scripts

## แบบทดสอบ

1. อธิบายข้อดีและข้อเสียของ monorepo เทียบกับ polyrepo
2. `melos exec` ต่างจาก `melos run` อย่างไร?
3. สร้าง `notification` package ที่ใช้ร่วมกันระหว่าง 3 apps
4. ทำไม path dependency (`path: ../package`) จึงดีกว่า version dependency ใน monorepo?
