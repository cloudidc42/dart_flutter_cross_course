# Part 81: Contributing to Flutter Open Source - การ Contribute ให้ Flutter OSS

## บทนำ

การ contribute ให้ Flutter open source ไม่เพียงแต่ช่วยพัฒนา ecosystem แต่ยังช่วยพัฒนา skills ของเราเองด้วย ส่วนนี้จะสอนวิธีเริ่มต้น contribute ให้ Flutter framework และ ecosystem

## 1. Flutter Repo Structure

### ทำความเข้าใจ Flutter Repositories

```
flutter/flutter          - Main Flutter framework
flutter/engine           - Flutter engine (C++, Dart, Skia)
flutter/packages         - Official packages (camera, path, etc.)
dart-lang/sdk            - Dart SDK
flutter/website          - flutter.dev website
flutter/devtools         - Flutter DevTools
```

### Clone Flutter Repo

```bash
# Clone Flutter framework
git clone https://github.com/flutter/flutter.git
cd flutter

# ดู repo structure
ls -la
```

```
flutter/
├── bin/                 # flutter CLI scripts
├── dev/                 # Developer tools
│   ├── bots/           # CI bots
│   ├── benchmarks/     # Performance benchmarks
│   └── tools/          # Developer tools
├── examples/           # Sample applications
├── packages/           # Framework packages
│   ├── flutter/        # Core Flutter package
│   ├── flutter_test/   # Testing utilities
│   ├── flutter_driver/ # Integration tests
│   └── integration_test/
└── tests/              # Tests
```

### Framework Package Structure

```
packages/flutter/lib/
├── src/
│   ├── animation/      # Animation framework
│   ├── cupertino/      # iOS-style widgets
│   ├── foundation/     # Core utilities
│   ├── gestures/       # Gesture recognition
│   ├── material/       # Material Design widgets
│   ├── painting/       # Drawing and painting
│   ├── physics/        # Physics simulations
│   ├── rendering/      # Render objects
│   ├── scheduler/      # Frame scheduling
│   ├── semantics/      # Accessibility
│   ├── services/       # Platform services
│   └── widgets/        # Core widgets
└── flutter.dart        # Main export file
```

## 2. Good First Issues

### หา Issues ที่เหมาะสม

```
Labels ที่ควรมองหา:
- "good first contribution" - เหมาะสำหรับ beginners
- "P2" หรือ "P3" - priority ปานกลาง ไม่ urgent
- "documentation" - แก้ docs ง่ายกว่า code
- "a: tests" - เพิ่ม/แก้ tests
```

### ตัวอย่างการค้นหา Issues

```
GitHub search:
https://github.com/flutter/flutter/issues?q=is:open+is:issue+label:"good first contribution"

Flutter issue tracker:
https://github.com/flutter/flutter/labels/good%20first%20contribution
```

### ประเภท Contributions ที่เริ่มง่าย

1. **Documentation fixes** - แก้ typo, เพิ่ม examples
2. **Test additions** - เพิ่ม test cases ที่ขาด
3. **Minor bug fixes** - Fix edge cases ที่ง่าย
4. **Translation** - แปล docs เป็นภาษาต่างๆ

## 3. PR Process

### ขั้นตอนการสร้าง PR

```bash
# 1. Fork repo บน GitHub

# 2. Clone fork ของเราเอง
git clone https://github.com/YOUR_USERNAME/flutter.git
cd flutter

# 3. เพิ่ม upstream remote
git remote add upstream https://github.com/flutter/flutter.git

# 4. สร้าง branch ใหม่
git checkout -b fix/button-tooltip-alignment

# 5. ทำการแก้ไข...

# 6. Run tests
flutter test packages/flutter/test/

# 7. Check formatting
dart format --set-exit-if-changed packages/flutter/lib/

# 8. Check analyzer
flutter analyze packages/flutter/

# 9. Commit
git add .
git commit -m "Fix: Button tooltip alignment on RTL layouts

Fixes #12345

The tooltip was misaligned when the layout direction
was set to RTL. This fix adjusts the tooltip position
calculation to account for directionality."

# 10. Push และสร้าง PR
git push origin fix/button-tooltip-alignment
```

### PR Template

```markdown
## Description
<!-- อธิบายว่าแก้อะไร และทำไม -->

Fixes the alignment issue with `Tooltip` widget in RTL layouts.
Previously, the tooltip would appear on the wrong side of the
anchor widget when `TextDirection.rtl` was set.

## Related Issues
Fixes #12345

## Type of Change
- [ ] Bug fix (non-breaking change that fixes an issue)
- [ ] New feature (non-breaking change that adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] Documentation update

## Testing
- Added unit tests in `packages/flutter/test/material/tooltip_test.dart`
- Added golden tests for RTL layout
- Manually tested on both iOS and Android

## Screenshots (if applicable)
<!-- เพิ่ม before/after screenshots ถ้ามี visual changes -->
```

### Code Review Process

```
1. สร้าง PR
2. CI/CD runs automatically (analyze, test, format)
3. Reviewer assign ให้ Flutter team member
4. Address feedback
5. LGTM (Looks Good To Me) จาก reviewer
6. Merge
```

## 4. Writing Flutter Framework Code

### Style Guide

```dart
// Flutter Framework Coding Style

// 1. ใช้ /// สำหรับ documentation comments
/// A widget that displays a tooltip message when long pressed.
///
/// Tooltips provide text labels which help explain the function of a
/// button or other user interface action.
///
/// {@tool dartpad}
/// Here is an example showing how to add a tooltip to a button:
///
/// ** See code in examples/api/lib/material/tooltip/tooltip.0.dart **
/// {@end-tool}
///
/// See also:
///  * [IconButton], which automatically provides tooltips.
class Tooltip extends StatefulWidget {
  // ...
}

// 2. ใช้ asserts สำหรับ validation
class SomeWidget extends StatelessWidget {
  final double size;
  
  const SomeWidget({
    super.key,
    required this.size,
  }) : assert(size > 0, 'size must be positive');
  
  @override
  Widget build(BuildContext context) => const SizedBox();
}

// 3. Override toString() สำหรับ debugging
@override
String toString() {
  return '${objectRuntimeType(this, 'Tooltip')}(message: $message)';
}

// 4. Implement debugFillProperties
@override
void debugFillProperties(DiagnosticPropertiesBuilder properties) {
  super.debugFillProperties(properties);
  properties.add(StringProperty('message', message));
  properties.add(DoubleProperty('height', height));
}
```

### Writing Tests

```dart
// test/material/my_widget_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  group('MyWidget', () {
    testWidgets('shows correct text', (tester) async {
      // Build widget
      await tester.pumpWidget(
        const MaterialApp(
          home: Scaffold(
            body: MyWidget(text: 'Hello'),
          ),
        ),
      );

      // Verify
      expect(find.text('Hello'), findsOneWidget);
    });

    testWidgets('handles tap', (tester) async {
      bool tapped = false;

      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: MyWidget(
              text: 'Tap me',
              onTap: () => tapped = true,
            ),
          ),
        ),
      );

      await tester.tap(find.byType(MyWidget));
      await tester.pump();

      expect(tapped, isTrue);
    });

    // Golden test สำหรับ visual regression
    testWidgets('matches golden', (tester) async {
      await tester.pumpWidget(
        const RepaintBoundary(
          child: MaterialApp(
            home: Scaffold(
              body: Center(
                child: MyWidget(text: 'Golden'),
              ),
            ),
          ),
        ),
      );

      await expectLater(
        find.byType(RepaintBoundary),
        matchesGoldenFile('goldens/my_widget.png'),
      );
    });
  });
}
```

## 5. Workshop: Fix a Flutter Issue

### ตัวอย่าง: แก้ Issue จริง

```dart
// สมมติ issue: "TextField hint text color ignores MaterialStateProperty"
// ปัญหา: hint color ไม่เปลี่ยนตาม state (focused, disabled, etc.)

// BEFORE (ปัญหา):
class _TextFieldState extends State<TextField> {
  // ...
  
  // เดิมใช้ color โดยตรงโดยไม่ check state
  Color? get _hintColor {
    return widget.decoration?.hintStyle?.color ?? 
           theme.hintColor; // ไม่ support MaterialStateProperty
  }
}

// AFTER (แก้ไข):
class _TextFieldState extends State<TextField> {
  // ...
  
  // เพิ่ม support สำหรับ MaterialStateProperty
  Color? get _hintColor {
    final hintStyle = widget.decoration?.hintStyle;
    
    // Check ว่า color เป็น MaterialStateProperty หรือไม่
    if (hintStyle?.color is MaterialStateProperty<Color?>) {
      final stateProperty =
          hintStyle!.color as MaterialStateProperty<Color?>;
      return stateProperty.resolve(_materialState);
    }
    
    return hintStyle?.color ?? theme.hintColor;
  }
  
  Set<MaterialState> get _materialState {
    return <MaterialState>{
      if (!widget.enabled) MaterialState.disabled,
      if (_effectiveFocusNode.hasFocus) MaterialState.focused,
      if (_isHovering) MaterialState.hovered,
      if (_hasError) MaterialState.error,
    };
  }
}
```

### เพิ่ม Tests สำหรับ Fix

```dart
// test/material/text_field_hint_color_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  group('TextField hint color', () {
    testWidgets(
      'hint color changes when focused',
      (tester) async {
        const normalColor = Color(0xFF888888);
        const focusedColor = Color(0xFF0000FF);

        await tester.pumpWidget(
          MaterialApp(
            home: Scaffold(
              body: TextField(
                decoration: InputDecoration(
                  hintText: 'Enter text',
                  hintStyle: TextStyle(
                    color: MaterialStateProperty.resolveWith((states) {
                      if (states.contains(MaterialState.focused)) {
                        return focusedColor;
                      }
                      return normalColor;
                    }),
                  ),
                ),
              ),
            ),
          ),
        );

        // ก่อน focus: ควรเป็น normalColor
        final hintText = tester.widget<Text>(find.text('Enter text'));
        expect(hintText.style?.color, equals(normalColor));

        // Tap เพื่อ focus
        await tester.tap(find.byType(TextField));
        await tester.pump();

        // หลัง focus: ควรเป็น focusedColor
        final focusedHintText = tester.widget<Text>(find.text('Enter text'));
        expect(focusedHintText.style?.color, equals(focusedColor));
      },
    );
  });
}
```

### Changelog Entry

```markdown
<!-- CHANGELOG.md -->
## [Unreleased]

### Fixed
- `TextField` hint text now correctly respects `MaterialStateProperty` 
  for the hint style color. (#12345)
```

## สรุป

Contributing to Flutter OSS:
1. เริ่มจาก "good first contribution" issues
2. อ่าน CONTRIBUTING.md ก่อนเสมอ
3. เขียน tests ที่ครอบคลุม
4. Follow Flutter's coding style
5. อดทนกับ review process

## แบบทดสอบ

1. อธิบายความแตกต่างระหว่าง flutter/flutter และ flutter/engine repo
2. ทำไม Flutter team ถึง require golden tests สำหรับ visual changes?
3. หา 3 "good first contribution" issues บน GitHub และอธิบายว่าจะแก้อย่างไร
4. เขียน `debugFillProperties` สำหรับ widget ที่มี 3 properties
