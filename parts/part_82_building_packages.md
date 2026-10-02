# Part 82: Building Flutter Packages - สร้าง Flutter Packages

## บทนำ

การสร้าง Flutter package ช่วยให้เราแชร์โค้ดกับ community และทำให้โปรเจกต์ตัวเองเป็นระเบียบ ส่วนนี้จะสอนทุกอย่างตั้งแต่ออกแบบ API ไปจนถึง publish บน pub.dev

## 1. Package Structure

### สร้าง Package ใหม่

```bash
# สร้าง Dart package
flutter create --template=package my_awesome_package

# สร้าง Flutter plugin (มี native code)
flutter create --template=plugin my_awesome_plugin

# สร้าง Flutter plugin สำหรับแพลตฟอร์มเฉพาะ
flutter create --template=plugin \
  --platforms=android,ios,web \
  my_awesome_plugin
```

### โครงสร้าง Package

```
my_awesome_package/
├── lib/
│   ├── src/                    # Implementation (ซ่อนจาก consumer)
│   │   ├── models/
│   │   ├── utils/
│   │   └── widgets/
│   └── my_awesome_package.dart # Public API
├── test/
│   ├── src/
│   └── my_awesome_package_test.dart
├── example/                    # Example app
│   ├── lib/
│   │   └── main.dart
│   └── pubspec.yaml
├── CHANGELOG.md               # Version history
├── LICENSE                    # Open source license
├── README.md                  # Documentation
└── pubspec.yaml
```

### pubspec.yaml สำหรับ Package

```yaml
# pubspec.yaml
name: my_awesome_package
description: A Flutter package that does amazing things.
  Keep the description short (80 chars max for pub.dev score).
version: 1.0.0
homepage: https://github.com/yourusername/my_awesome_package
repository: https://github.com/yourusername/my_awesome_package
issue_tracker: https://github.com/yourusername/my_awesome_package/issues
documentation: https://pub.dev/documentation/my_awesome_package/latest/

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: '>=3.16.0'

dependencies:
  flutter:
    sdk: flutter

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0

flutter:
  # ถ้าม assets หรือ fonts
  # assets:
  #   - assets/
```

## 2. API Design

### หลักการออกแบบ API ที่ดี

```dart
// lib/src/widgets/cool_button.dart

/// A customizable button widget with multiple variants.
///
/// This widget provides a consistent button experience across your app
/// with support for different visual styles, sizes, and states.
///
/// ## Basic Usage
///
/// ```dart
/// CoolButton(
///   onPressed: () => print('Pressed!'),
///   child: const Text('Click me'),
/// )
/// ```
///
/// ## With Icon
///
/// ```dart
/// CoolButton.icon(
///   onPressed: () {},
///   icon: const Icon(Icons.add),
///   label: const Text('Add Item'),
/// )
/// ```
class CoolButton extends StatelessWidget {
  // ใช้ named parameters เสมอสำหรับ optional parameters
  // required parameters ก่อน, optional parameters หลัง
  
  final Widget child;
  final VoidCallback? onPressed;
  final CoolButtonVariant variant;
  final CoolButtonSize size;
  final bool isLoading;
  
  /// Creates a [CoolButton].
  ///
  /// The [child] argument must not be null.
  const CoolButton({
    super.key,
    required this.child,
    this.onPressed,
    this.variant = CoolButtonVariant.primary,
    this.size = CoolButtonSize.medium,
    this.isLoading = false,
  });
  
  /// Creates a [CoolButton] with an icon and label.
  ///
  /// The [icon] and [label] arguments must not be null.
  factory CoolButton.icon({
    Key? key,
    required Widget icon,
    required Widget label,
    VoidCallback? onPressed,
    CoolButtonVariant variant = CoolButtonVariant.primary,
    CoolButtonSize size = CoolButtonSize.medium,
    bool isLoading = false,
  }) {
    return CoolButton(
      key: key,
      onPressed: onPressed,
      variant: variant,
      size: size,
      isLoading: isLoading,
      child: Row(
        mainAxisSize: MainAxisSize.min,
        children: [icon, const SizedBox(width: 8), label],
      ),
    );
  }
  
  @override
  Widget build(BuildContext context) {
    // Implementation...
    return FilledButton(
      onPressed: isLoading ? null : onPressed,
      child: isLoading
          ? const SizedBox(
              width: 20,
              height: 20,
              child: CircularProgressIndicator(strokeWidth: 2),
            )
          : child,
    );
  }
  
  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.add(EnumProperty<CoolButtonVariant>('variant', variant));
    properties.add(EnumProperty<CoolButtonSize>('size', size));
    properties.add(DiagnosticsProperty<bool>('isLoading', isLoading));
  }
}

/// Visual variants for [CoolButton].
enum CoolButtonVariant {
  /// Primary action button, most visually prominent.
  primary,
  
  /// Secondary action button.
  secondary,
  
  /// Outlined button, less prominent.
  outlined,
  
  /// Text-only button, least prominent.
  text,
  
  /// Destructive action button (e.g., delete).
  destructive,
}

/// Size variants for [CoolButton].
enum CoolButtonSize {
  /// Small button for tight spaces.
  small,
  
  /// Default medium size.
  medium,
  
  /// Large button for prominent actions.
  large,
}
```

### Public API File

```dart
// lib/my_awesome_package.dart

/// A collection of awesome Flutter widgets and utilities.
///
/// ## Getting Started
///
/// Add to your `pubspec.yaml`:
///
/// ```yaml
/// dependencies:
///   my_awesome_package: ^1.0.0
/// ```
///
/// Then import:
///
/// ```dart
/// import 'package:my_awesome_package/my_awesome_package.dart';
/// ```
library my_awesome_package;

// Widgets
export 'src/widgets/cool_button.dart';
export 'src/widgets/cool_card.dart';
export 'src/widgets/cool_text_field.dart';

// Models
export 'src/models/cool_config.dart';

// Utils
export 'src/utils/cool_extensions.dart';

// แสดงเฉพาะสิ่งที่ต้องการ expose เท่านั้น
// ไม่ต้อง export implementation details
```

## 3. Documentation

### README.md Template

```markdown
# my_awesome_package

[![pub package](https://img.shields.io/pub/v/my_awesome_package.svg)](https://pub.dev/packages/my_awesome_package)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A Flutter package that provides awesome UI components with full customization.

## Features

- 🎨 Multiple visual variants (primary, secondary, outlined)
- 📱 Responsive on all platforms
- ♿ Full accessibility support
- 🌙 Dark mode ready
- 🧪 100% test coverage

## Getting Started

Add to `pubspec.yaml`:

```yaml
dependencies:
  my_awesome_package: ^1.0.0
```

## Usage

### CoolButton

```dart
import 'package:my_awesome_package/my_awesome_package.dart';

// Basic button
CoolButton(
  onPressed: () => print('Pressed!'),
  child: const Text('Click me'),
)

// With icon
CoolButton.icon(
  onPressed: () {},
  icon: const Icon(Icons.star),
  label: const Text('Favorite'),
  variant: CoolButtonVariant.primary,
)

// Loading state
CoolButton(
  onPressed: null,
  isLoading: true,
  child: const Text('Saving...'),
)
```

## Additional Information

- [API Documentation](https://pub.dev/documentation/my_awesome_package/latest/)
- [GitHub Repository](https://github.com/yourusername/my_awesome_package)
- [Changelog](CHANGELOG.md)

## Contributing

Contributions are welcome! Please read our [contributing guide](CONTRIBUTING.md).

## License

MIT License - see [LICENSE](LICENSE) file for details.
```

### CHANGELOG.md

```markdown
# Changelog

All notable changes to this project will be documented in this file.
This project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.1.0] - 2024-01-15

### Added
- New `CoolButton.icon()` factory constructor
- Support for `CoolButtonSize.large`
- Loading state animation

### Fixed
- Button text truncation on small screens

## [1.0.1] - 2024-01-10

### Fixed
- Fixed padding issue in `CoolCard` on iOS

## [1.0.0] - 2024-01-01

### Added
- Initial release
- `CoolButton` widget with primary, secondary, outlined variants
- `CoolCard` widget
- `CoolTextField` widget
```

## 4. Testing Packages

### Comprehensive Tests

```dart
// test/src/widgets/cool_button_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:my_awesome_package/my_awesome_package.dart';

void main() {
  group('CoolButton', () {
    Widget buildButton({
      VoidCallback? onPressed,
      CoolButtonVariant variant = CoolButtonVariant.primary,
      bool isLoading = false,
    }) {
      return MaterialApp(
        home: Scaffold(
          body: CoolButton(
            onPressed: onPressed,
            variant: variant,
            isLoading: isLoading,
            child: const Text('Test'),
          ),
        ),
      );
    }

    testWidgets('renders child widget', (tester) async {
      await tester.pumpWidget(buildButton());
      expect(find.text('Test'), findsOneWidget);
    });

    testWidgets('calls onPressed when tapped', (tester) async {
      bool pressed = false;
      await tester.pumpWidget(buildButton(onPressed: () => pressed = true));

      await tester.tap(find.byType(CoolButton));
      expect(pressed, isTrue);
    });

    testWidgets('is disabled when onPressed is null', (tester) async {
      await tester.pumpWidget(buildButton());
      
      final button = tester.widget<FilledButton>(find.byType(FilledButton));
      expect(button.onPressed, isNull);
    });

    testWidgets('shows loading indicator when isLoading is true', (tester) async {
      await tester.pumpWidget(buildButton(
        onPressed: () {},
        isLoading: true,
      ));

      expect(find.byType(CircularProgressIndicator), findsOneWidget);
      expect(find.text('Test'), findsNothing);
    });

    testWidgets('is disabled when loading', (tester) async {
      bool pressed = false;
      await tester.pumpWidget(buildButton(
        onPressed: () => pressed = true,
        isLoading: true,
      ));

      await tester.tap(find.byType(CoolButton));
      expect(pressed, isFalse);
    });

    group('factory constructors', () {
      testWidgets('CoolButton.icon renders icon and label', (tester) async {
        await tester.pumpWidget(
          MaterialApp(
            home: Scaffold(
              body: CoolButton.icon(
                onPressed: () {},
                icon: const Icon(Icons.star),
                label: const Text('Favorite'),
              ),
            ),
          ),
        );

        expect(find.byIcon(Icons.star), findsOneWidget);
        expect(find.text('Favorite'), findsOneWidget);
      });
    });

    group('semantics', () {
      testWidgets('has correct semantics', (tester) async {
        await tester.pumpWidget(buildButton(onPressed: () {}));

        expect(
          tester.getSemantics(find.byType(CoolButton)),
          matchesSemantics(
            isButton: true,
            isEnabled: true,
            hasEnabledState: true,
          ),
        );
      });
    });
  });
}
```

### Example App

```dart
// example/lib/main.dart
import 'package:flutter/material.dart';
import 'package:my_awesome_package/my_awesome_package.dart';

void main() {
  runApp(const ExampleApp());
}

class ExampleApp extends StatelessWidget {
  const ExampleApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'CoolButton Example',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
        useMaterial3: true,
      ),
      home: const HomePage(),
    );
  }
}

class HomePage extends StatefulWidget {
  const HomePage({super.key});

  @override
  State<HomePage> createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  bool _isLoading = false;

  Future<void> _simulateAction() async {
    setState(() => _isLoading = true);
    await Future.delayed(const Duration(seconds: 2));
    if (mounted) setState(() => _isLoading = false);
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('CoolButton Examples')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text('Variants', style: Theme.of(context).textTheme.titleLarge),
            const SizedBox(height: 16),
            
            CoolButton(
              onPressed: () {},
              variant: CoolButtonVariant.primary,
              child: const Text('Primary'),
            ),
            const SizedBox(height: 8),
            
            CoolButton(
              onPressed: () {},
              variant: CoolButtonVariant.secondary,
              child: const Text('Secondary'),
            ),
            const SizedBox(height: 8),
            
            CoolButton(
              onPressed: () {},
              variant: CoolButtonVariant.outlined,
              child: const Text('Outlined'),
            ),
            const SizedBox(height: 24),
            
            Text('With Icon', style: Theme.of(context).textTheme.titleLarge),
            const SizedBox(height: 16),
            
            CoolButton.icon(
              onPressed: () {},
              icon: const Icon(Icons.star),
              label: const Text('Favorite'),
            ),
            const SizedBox(height: 24),
            
            Text('Loading State', style: Theme.of(context).textTheme.titleLarge),
            const SizedBox(height: 16),
            
            CoolButton(
              onPressed: _isLoading ? null : _simulateAction,
              isLoading: _isLoading,
              child: Text(_isLoading ? 'Loading...' : 'Click to Load'),
            ),
          ],
        ),
      ),
    );
  }
}
```

## 5. Publishing to pub.dev

### ตรวจสอบก่อน Publish

```bash
# ตรวจสอบ package
dart pub publish --dry-run

# ดู pub score
dart pub score

# Format code
dart format .

# Run analyzer
dart analyze

# Run tests
flutter test
```

### pub.dev Score Checklist

```
✅ README.md มีเนื้อหาครบ
✅ CHANGELOG.md อัปเดตแล้ว
✅ LICENSE file มี
✅ API documentation ครบ (/// comments)
✅ Example app ทำงานได้
✅ Tests ผ่าน
✅ Analyzer ไม่มี warnings
✅ Format ถูกต้อง
✅ Version ตาม semver
```

### Publish

```bash
# Publish จริง
dart pub publish

# ยืนยันด้วย Google account
# Package จะ live บน pub.dev ภายใน 30 นาที
```

### Versioning Strategy

```
Semantic Versioning: MAJOR.MINOR.PATCH

MAJOR: breaking changes (API เปลี่ยน)
MINOR: new features (backward compatible)
PATCH: bug fixes (backward compatible)

ตัวอย่าง:
1.0.0 → 1.0.1 (bug fix)
1.0.0 → 1.1.0 (new feature)
1.0.0 → 2.0.0 (breaking change)
```

## สรุป

Building Flutter Packages:
1. **ออกแบบ API ให้ชัดเจน** - Intuitive, consistent naming
2. **Documentation** - README, API docs, Examples
3. **Tests** - 80%+ coverage
4. **pub.dev score** - ส่งผลต่อความน่าเชื่อถือ
5. **Semantic versioning** - ทำตามอย่างเคร่งครัด

## แบบทดสอบ

1. อธิบาย public API ของ package ต่างจาก implementation อย่างไร?
2. ทำไม `library` declaration สำคัญในไฟล์ main export?
3. สร้าง `CoolCard` widget ที่มี API ที่ดีพร้อม documentation ครบ
4. อธิบาย pub.dev score metrics 5 ข้อหลัก
