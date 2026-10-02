# Part 29: Themes and Styling

## บทนำ

Themes คือระบบ styling ใน Flutter ที่ช่วยให้แอพมีรูปแบบที่สอดคล้องกันตลอด Material 3 ยิ่งทำให้การ customize theme ง่ายขึ้น

---

## ThemeData Customization

```dart
// Theme แบบครบถ้วน
ThemeData buildTheme(ColorScheme colorScheme) {
  return ThemeData(
    useMaterial3: true,
    colorScheme: colorScheme,
    
    // === Typography ===
    fontFamily: 'Kanit',
    textTheme: _buildTextTheme(),
    
    // === AppBar ===
    appBarTheme: AppBarTheme(
      backgroundColor: colorScheme.surface,
      foregroundColor: colorScheme.onSurface,
      elevation: 0,
      scrolledUnderElevation: 1,
      centerTitle: true,
      titleTextStyle: TextStyle(
        color: colorScheme.onSurface,
        fontSize: 20,
        fontWeight: FontWeight.w600,
      ),
      iconTheme: IconThemeData(color: colorScheme.onSurface),
    ),
    
    // === Bottom Navigation ===
    navigationBarTheme: NavigationBarThemeData(
      backgroundColor: colorScheme.surface,
      indicatorColor: colorScheme.secondaryContainer,
      iconTheme: MaterialStateProperty.resolveWith((states) {
        if (states.contains(MaterialState.selected)) {
          return IconThemeData(color: colorScheme.onSecondaryContainer);
        }
        return IconThemeData(color: colorScheme.onSurfaceVariant);
      }),
      labelTextStyle: MaterialStateProperty.resolveWith((states) {
        if (states.contains(MaterialState.selected)) {
          return TextStyle(
            color: colorScheme.onSurface,
            fontWeight: FontWeight.w600,
          );
        }
        return TextStyle(color: colorScheme.onSurfaceVariant);
      }),
    ),
    
    // === Buttons ===
    elevatedButtonTheme: ElevatedButtonThemeData(
      style: ElevatedButton.styleFrom(
        backgroundColor: colorScheme.primary,
        foregroundColor: colorScheme.onPrimary,
        padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 14),
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(14),
        ),
        textStyle: const TextStyle(
          fontSize: 16,
          fontWeight: FontWeight.w600,
        ),
      ),
    ),
    
    outlinedButtonTheme: OutlinedButtonThemeData(
      style: OutlinedButton.styleFrom(
        padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 14),
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(14),
        ),
        side: BorderSide(color: colorScheme.outline),
      ),
    ),
    
    textButtonTheme: TextButtonThemeData(
      style: TextButton.styleFrom(
        padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 10),
      ),
    ),
    
    filledButtonTheme: FilledButtonThemeData(
      style: FilledButton.styleFrom(
        padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 14),
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(14),
        ),
      ),
    ),
    
    // === Card ===
    cardTheme: CardTheme(
      elevation: 0,
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.circular(16),
        side: BorderSide(
          color: colorScheme.outlineVariant.withOpacity(0.5),
        ),
      ),
      clipBehavior: Clip.antiAlias,
    ),
    
    // === Input ===
    inputDecorationTheme: InputDecorationTheme(
      filled: true,
      fillColor: colorScheme.surfaceVariant.withOpacity(0.5),
      border: OutlineInputBorder(
        borderRadius: BorderRadius.circular(12),
        borderSide: BorderSide.none,
      ),
      enabledBorder: OutlineInputBorder(
        borderRadius: BorderRadius.circular(12),
        borderSide: BorderSide(color: colorScheme.outline.withOpacity(0.3)),
      ),
      focusedBorder: OutlineInputBorder(
        borderRadius: BorderRadius.circular(12),
        borderSide: BorderSide(color: colorScheme.primary, width: 2),
      ),
      errorBorder: OutlineInputBorder(
        borderRadius: BorderRadius.circular(12),
        borderSide: BorderSide(color: colorScheme.error),
      ),
      contentPadding: const EdgeInsets.symmetric(horizontal: 16, vertical: 16),
      labelStyle: TextStyle(color: colorScheme.onSurfaceVariant),
      hintStyle: TextStyle(color: colorScheme.onSurfaceVariant.withOpacity(0.7)),
    ),
    
    // === Chip ===
    chipTheme: ChipThemeData(
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.circular(8),
      ),
      side: BorderSide(color: colorScheme.outline.withOpacity(0.5)),
    ),
    
    // === Divider ===
    dividerTheme: DividerThemeData(
      color: colorScheme.outlineVariant,
      space: 1,
    ),
    
    // === Dialog ===
    dialogTheme: DialogTheme(
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.circular(24),
      ),
      elevation: 4,
    ),
    
    // === BottomSheet ===
    bottomSheetTheme: const BottomSheetThemeData(
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.vertical(top: Radius.circular(24)),
      ),
      clipBehavior: Clip.antiAlias,
      showDragHandle: true,
    ),
    
    // === SnackBar ===
    snackBarTheme: SnackBarThemeData(
      behavior: SnackBarBehavior.floating,
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.circular(12),
      ),
    ),
    
    // === FloatingActionButton ===
    floatingActionButtonTheme: FloatingActionButtonThemeData(
      backgroundColor: colorScheme.primaryContainer,
      foregroundColor: colorScheme.onPrimaryContainer,
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.circular(16),
      ),
    ),
  );
}

TextTheme _buildTextTheme() {
  return const TextTheme(
    displayLarge: TextStyle(fontSize: 57, fontWeight: FontWeight.w400, letterSpacing: -0.25),
    displayMedium: TextStyle(fontSize: 45, fontWeight: FontWeight.w400),
    displaySmall: TextStyle(fontSize: 36, fontWeight: FontWeight.w400),
    headlineLarge: TextStyle(fontSize: 32, fontWeight: FontWeight.w600),
    headlineMedium: TextStyle(fontSize: 28, fontWeight: FontWeight.w600),
    headlineSmall: TextStyle(fontSize: 24, fontWeight: FontWeight.w600),
    titleLarge: TextStyle(fontSize: 22, fontWeight: FontWeight.w500),
    titleMedium: TextStyle(fontSize: 16, fontWeight: FontWeight.w600),
    titleSmall: TextStyle(fontSize: 14, fontWeight: FontWeight.w600),
    bodyLarge: TextStyle(fontSize: 16, fontWeight: FontWeight.w400, height: 1.6),
    bodyMedium: TextStyle(fontSize: 14, fontWeight: FontWeight.w400, height: 1.5),
    bodySmall: TextStyle(fontSize: 12, fontWeight: FontWeight.w400, height: 1.4),
    labelLarge: TextStyle(fontSize: 14, fontWeight: FontWeight.w500, letterSpacing: 0.1),
    labelMedium: TextStyle(fontSize: 12, fontWeight: FontWeight.w500, letterSpacing: 0.5),
    labelSmall: TextStyle(fontSize: 11, fontWeight: FontWeight.w500, letterSpacing: 0.5),
  );
}
```

---

## ColorScheme (Material 3)

ColorScheme ใน Material 3 มีสีที่เชื่อมโยงกัน 25+ สี

```dart
// วิธีที่ 1: fromSeed - ง่ายที่สุด
final colorScheme = ColorScheme.fromSeed(
  seedColor: const Color(0xFF6750A4),
  brightness: Brightness.light,
);

// วิธีที่ 2: กำหนดเอง
const colorScheme = ColorScheme(
  // Primary
  primary: Color(0xFF6750A4),           // ปุ่ม, active state
  onPrimary: Color(0xFFFFFFFF),         // text บน primary
  primaryContainer: Color(0xFFEADDFF),  // background ของ container
  onPrimaryContainer: Color(0xFF21005D),// text บน primaryContainer
  
  // Secondary
  secondary: Color(0xFF625B71),
  onSecondary: Color(0xFFFFFFFF),
  secondaryContainer: Color(0xFFE8DEF8),
  onSecondaryContainer: Color(0xFF1D192B),
  
  // Tertiary
  tertiary: Color(0xFF7D5260),
  onTertiary: Color(0xFFFFFFFF),
  tertiaryContainer: Color(0xFFFFD8E4),
  onTertiaryContainer: Color(0xFF31111D),
  
  // Error
  error: Color(0xFFB3261E),
  onError: Color(0xFFFFFFFF),
  errorContainer: Color(0xFFF9DEDC),
  onErrorContainer: Color(0xFF410E0B),
  
  // Background
  background: Color(0xFFFFFBFE),
  onBackground: Color(0xFF1C1B1F),
  
  // Surface
  surface: Color(0xFFFFFBFE),
  onSurface: Color(0xFF1C1B1F),
  surfaceVariant: Color(0xFFE7E0EC),
  onSurfaceVariant: Color(0xFF49454F),
  
  // Others
  outline: Color(0xFF79747E),
  outlineVariant: Color(0xFFCAC4D0),
  shadow: Color(0xFF000000),
  scrim: Color(0xFF000000),
  inverseSurface: Color(0xFF313033),
  onInverseSurface: Color(0xFFF4EFF4),
  inversePrimary: Color(0xFFD0BCFF),
  
  brightness: Brightness.light,
);

// ใช้งานสี
final cs = Theme.of(context).colorScheme;

Container(color: cs.primaryContainer)
Text('Hello', style: TextStyle(color: cs.onPrimaryContainer))
```

---

## Dark Mode

```dart
// MaterialApp กำหนด light/dark theme
MaterialApp(
  theme: AppTheme.lightTheme,
  darkTheme: AppTheme.darkTheme,
  themeMode: ThemeMode.system, // หรือ light/dark
)

// สร้าง dark theme
class AppTheme {
  static ThemeData get lightTheme => buildTheme(
    ColorScheme.fromSeed(
      seedColor: const Color(0xFF6750A4),
      brightness: Brightness.light,
    ),
  );
  
  static ThemeData get darkTheme => buildTheme(
    ColorScheme.fromSeed(
      seedColor: const Color(0xFF6750A4),
      brightness: Brightness.dark,
    ),
  );
}

// Toggle dark mode จาก app
class ThemeNotifier extends ChangeNotifier {
  ThemeMode _themeMode = ThemeMode.system;
  
  ThemeMode get themeMode => _themeMode;
  
  bool get isDark => _themeMode == ThemeMode.dark;
  
  void setThemeMode(ThemeMode mode) {
    _themeMode = mode;
    notifyListeners();
  }
  
  void toggle() {
    _themeMode = isDark ? ThemeMode.light : ThemeMode.dark;
    notifyListeners();
  }
}
```

---

## Custom Fonts

### เพิ่ม Font ใน pubspec.yaml

```yaml
flutter:
  fonts:
    - family: Kanit
      fonts:
        - asset: assets/fonts/Kanit-Thin.ttf
          weight: 100
        - asset: assets/fonts/Kanit-Light.ttf
          weight: 300
        - asset: assets/fonts/Kanit-Regular.ttf
          weight: 400
        - asset: assets/fonts/Kanit-Medium.ttf
          weight: 500
        - asset: assets/fonts/Kanit-SemiBold.ttf
          weight: 600
        - asset: assets/fonts/Kanit-Bold.ttf
          weight: 700
        - asset: assets/fonts/Kanit-ExtraBold.ttf
          weight: 800
        - asset: assets/fonts/Kanit-Italic.ttf
          style: italic
    
    - family: Prompt
      fonts:
        - asset: assets/fonts/Prompt-Regular.ttf
        - asset: assets/fonts/Prompt-Medium.ttf
          weight: 500
        - asset: assets/fonts/Prompt-Bold.ttf
          weight: 700
```

### ใช้ Google Fonts

```dart
// pubspec.yaml: google_fonts: ^6.1.0
import 'package:google_fonts/google_fonts.dart';

// ใช้ font โดยตรง
Text(
  'Hello',
  style: GoogleFonts.kanit(
    fontSize: 24,
    fontWeight: FontWeight.bold,
  ),
)

// ใน ThemeData
ThemeData(
  textTheme: GoogleFonts.promptTextTheme(
    Theme.of(context).textTheme,
  ),
)

// Combine fonts
ThemeData(
  textTheme: TextTheme(
    displayLarge: GoogleFonts.sarabun(
      fontSize: 57,
      fontWeight: FontWeight.w300,
    ),
    bodyMedium: GoogleFonts.notoSansThai(
      fontSize: 14,
    ),
  ),
)
```

---

## Material 3 Components

Material 3 มี component ใหม่ที่ Flutter รองรับ

```dart
// Buttons
ElevatedButton(onPressed: () {}, child: const Text('Elevated'))
FilledButton(onPressed: () {}, child: const Text('Filled'))
FilledButton.tonal(onPressed: () {}, child: const Text('Tonal'))
OutlinedButton(onPressed: () {}, child: const Text('Outlined'))
TextButton(onPressed: () {}, child: const Text('Text'))

// FAB
FloatingActionButton(onPressed: () {}, child: const Icon(Icons.add))
FloatingActionButton.small(onPressed: () {}, child: const Icon(Icons.add))
FloatingActionButton.large(onPressed: () {}, child: const Icon(Icons.add))
FloatingActionButton.extended(
  onPressed: () {},
  icon: const Icon(Icons.add),
  label: const Text('Add'),
)

// IconButton
IconButton(icon: const Icon(Icons.settings), onPressed: () {})
IconButton.filled(icon: const Icon(Icons.settings), onPressed: () {})
IconButton.filledTonal(icon: const Icon(Icons.settings), onPressed: () {})
IconButton.outlined(icon: const Icon(Icons.settings), onPressed: () {})

// Cards (M3)
Card(child: ...) // elevated (default)
Card.filled(child: ...) // filled
Card.outlined(child: ...) // outlined

// NavigationBar (M3)
NavigationBar(
  destinations: [
    NavigationDestination(icon: Icon(Icons.home), label: 'หน้าหลัก'),
    NavigationDestination(icon: Icon(Icons.search), label: 'ค้นหา'),
  ],
  selectedIndex: 0,
  onDestinationSelected: (index) {},
)

// NavigationDrawer (M3)
NavigationDrawer(
  children: [
    const DrawerHeader(child: Text('Menu')),
    NavigationDrawerDestination(
      icon: const Icon(Icons.home),
      label: const Text('หน้าหลัก'),
    ),
  ],
)

// SegmentedButton (M3)
SegmentedButton<String>(
  segments: const [
    ButtonSegment(value: 'day', label: Text('วัน')),
    ButtonSegment(value: 'week', label: Text('สัปดาห์')),
    ButtonSegment(value: 'month', label: Text('เดือน')),
  ],
  selected: {'day'},
  onSelectionChanged: (selection) {},
)

// Badge (M3)
Badge(
  label: const Text('3'),
  child: const Icon(Icons.notifications),
)

// Chips
FilterChip(label: Text('Flutter'), selected: true, onSelected: (_) {})
ActionChip(label: Text('Share'), onPressed: () {})
AssistChip(label: Text('Suggestion'), onPressed: () {})
InputChip(label: Text('Tag'), onDeleted: () {})
ChoiceChip(label: Text('Option'), selected: false, onSelected: (_) {})
```

---

## Workshop: App with Light/Dark Theme Toggle

### lib/core/theme/app_theme.dart

```dart
import 'package:flutter/material.dart';

class AppColors {
  // Brand colors
  static const Color brandPrimary = Color(0xFF5B4FCF);
  static const Color brandSecondary = Color(0xFF00B4D8);
  static const Color brandAccent = Color(0xFFFF6B6B);
}

class AppTheme {
  AppTheme._();

  static ThemeData get lightTheme {
    final cs = ColorScheme.fromSeed(
      seedColor: AppColors.brandPrimary,
      brightness: Brightness.light,
    );
    return _buildTheme(cs);
  }

  static ThemeData get darkTheme {
    final cs = ColorScheme.fromSeed(
      seedColor: AppColors.brandPrimary,
      brightness: Brightness.dark,
    );
    return _buildTheme(cs);
  }

  static ThemeData _buildTheme(ColorScheme cs) {
    return ThemeData(
      useMaterial3: true,
      colorScheme: cs,
      fontFamily: 'Kanit',
      appBarTheme: AppBarTheme(
        backgroundColor: cs.surface,
        foregroundColor: cs.onSurface,
        elevation: 0,
        centerTitle: false,
      ),
      cardTheme: CardTheme(
        clipBehavior: Clip.antiAlias,
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(16),
        ),
      ),
      elevatedButtonTheme: ElevatedButtonThemeData(
        style: ElevatedButton.styleFrom(
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(12),
          ),
          padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 14),
        ),
      ),
      inputDecorationTheme: InputDecorationTheme(
        border: OutlineInputBorder(
          borderRadius: BorderRadius.circular(12),
        ),
        filled: true,
      ),
    );
  }
}
```

### lib/core/providers/theme_provider.dart

```dart
import 'package:flutter/material.dart';

class ThemeProvider extends ChangeNotifier {
  ThemeMode _themeMode;

  ThemeProvider({ThemeMode initialMode = ThemeMode.system})
      : _themeMode = initialMode;

  ThemeMode get themeMode => _themeMode;

  bool get isLight => _themeMode == ThemeMode.light;
  bool get isDark => _themeMode == ThemeMode.dark;
  bool get isSystem => _themeMode == ThemeMode.system;

  String get themeName {
    switch (_themeMode) {
      case ThemeMode.light:
        return 'Light';
      case ThemeMode.dark:
        return 'Dark';
      case ThemeMode.system:
        return 'System';
    }
  }

  void setThemeMode(ThemeMode mode) {
    if (_themeMode != mode) {
      _themeMode = mode;
      notifyListeners();
    }
  }

  void toggleTheme(bool isCurrentlyDark) {
    setThemeMode(isCurrentlyDark ? ThemeMode.light : ThemeMode.dark);
  }
}
```

### lib/main.dart

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'core/theme/app_theme.dart';
import 'core/providers/theme_provider.dart';
import 'screens/home_screen.dart';

void main() {
  runApp(
    ChangeNotifierProvider(
      create: (_) => ThemeProvider(),
      child: const MyApp(),
    ),
  );
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    final themeProvider = context.watch<ThemeProvider>();

    return MaterialApp(
      title: 'Theme Demo',
      debugShowCheckedModeBanner: false,
      theme: AppTheme.lightTheme,
      darkTheme: AppTheme.darkTheme,
      themeMode: themeProvider.themeMode,
      home: const HomeScreen(),
    );
  }
}
```

### lib/screens/home_screen.dart

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../core/providers/theme_provider.dart';

class HomeScreen extends StatelessWidget {
  const HomeScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final isDark = theme.brightness == Brightness.dark;
    final themeProvider = context.watch<ThemeProvider>();

    return Scaffold(
      appBar: AppBar(
        title: const Text('Theme Demo'),
        actions: [
          // Dark mode toggle
          IconButton(
            icon: AnimatedSwitcher(
              duration: const Duration(milliseconds: 300),
              child: Icon(
                isDark ? Icons.dark_mode : Icons.light_mode,
                key: ValueKey(isDark),
              ),
            ),
            onPressed: () => themeProvider.toggleTheme(isDark),
          ),
        ],
      ),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // Theme mode selector
            _buildThemeModeCard(context, themeProvider),
            const SizedBox(height: 24),

            // Color swatches
            _buildColorSwatches(context),
            const SizedBox(height: 24),

            // Typography showcase
            _buildTypographyShowcase(context),
            const SizedBox(height: 24),

            // Components showcase
            _buildComponentsShowcase(context),
          ],
        ),
      ),
    );
  }

  Widget _buildThemeModeCard(BuildContext context, ThemeProvider provider) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text('Theme Mode',
              style: Theme.of(context).textTheme.titleMedium),
            const SizedBox(height: 12),
            SegmentedButton<ThemeMode>(
              segments: const [
                ButtonSegment(
                  value: ThemeMode.light,
                  icon: Icon(Icons.light_mode),
                  label: Text('Light'),
                ),
                ButtonSegment(
                  value: ThemeMode.system,
                  icon: Icon(Icons.brightness_auto),
                  label: Text('System'),
                ),
                ButtonSegment(
                  value: ThemeMode.dark,
                  icon: Icon(Icons.dark_mode),
                  label: Text('Dark'),
                ),
              ],
              selected: {provider.themeMode},
              onSelectionChanged: (modes) =>
                  provider.setThemeMode(modes.first),
            ),
          ],
        ),
      ),
    );
  }

  Widget _buildColorSwatches(BuildContext context) {
    final cs = Theme.of(context).colorScheme;

    final colors = [
      ('Primary', cs.primary, cs.onPrimary),
      ('Secondary', cs.secondary, cs.onSecondary),
      ('Tertiary', cs.tertiary, cs.onTertiary),
      ('Error', cs.error, cs.onError),
      ('Surface', cs.surface, cs.onSurface),
      ('Container', cs.primaryContainer, cs.onPrimaryContainer),
    ];

    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text('Color Scheme',
              style: Theme.of(context).textTheme.titleMedium),
            const SizedBox(height: 12),
            GridView.count(
              crossAxisCount: 3,
              shrinkWrap: true,
              physics: const NeverScrollableScrollPhysics(),
              crossAxisSpacing: 8,
              mainAxisSpacing: 8,
              childAspectRatio: 2,
              children: colors.map((color) {
                final (name, bg, fg) = color;
                return Container(
                  decoration: BoxDecoration(
                    color: bg,
                    borderRadius: BorderRadius.circular(8),
                  ),
                  child: Center(
                    child: Text(
                      name,
                      style: TextStyle(
                        color: fg,
                        fontSize: 11,
                        fontWeight: FontWeight.w500,
                      ),
                    ),
                  ),
                );
              }).toList(),
            ),
          ],
        ),
      ),
    );
  }

  Widget _buildTypographyShowcase(BuildContext context) {
    final textTheme = Theme.of(context).textTheme;

    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text('Typography', style: textTheme.titleMedium),
            const Divider(),
            ...[
              ('Display Large', textTheme.displayLarge),
              ('Headline Medium', textTheme.headlineMedium),
              ('Title Large', textTheme.titleLarge),
              ('Body Large', textTheme.bodyLarge),
              ('Label Medium', textTheme.labelMedium),
            ].map((pair) {
              final (name, style) = pair;
              return Padding(
                padding: const EdgeInsets.symmetric(vertical: 4),
                child: Row(
                  children: [
                    SizedBox(
                      width: 140,
                      child: Text(
                        name,
                        style: const TextStyle(fontSize: 10, color: Colors.grey),
                      ),
                    ),
                    Expanded(
                      child: Text('ข้อความตัวอย่าง', style: style?.copyWith(
                        fontSize: (style.fontSize ?? 14).clamp(12, 24),
                      )),
                    ),
                  ],
                ),
              );
            }),
          ],
        ),
      ),
    );
  }

  Widget _buildComponentsShowcase(BuildContext context) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text('Components', style: Theme.of(context).textTheme.titleMedium),
            const Divider(),

            // Buttons
            Text('Buttons', style: Theme.of(context).textTheme.labelLarge),
            const SizedBox(height: 8),
            Wrap(
              spacing: 8,
              runSpacing: 8,
              children: [
                ElevatedButton(onPressed: () {}, child: const Text('Elevated')),
                FilledButton(onPressed: () {}, child: const Text('Filled')),
                FilledButton.tonal(onPressed: () {}, child: const Text('Tonal')),
                OutlinedButton(onPressed: () {}, child: const Text('Outlined')),
                TextButton(onPressed: () {}, child: const Text('Text')),
              ],
            ),

            const SizedBox(height: 16),

            // Chips
            Text('Chips', style: Theme.of(context).textTheme.labelLarge),
            const SizedBox(height: 8),
            Wrap(
              spacing: 8,
              children: [
                FilterChip(label: const Text('Filter'), onSelected: (_) {}, selected: true),
                ActionChip(label: const Text('Action'), onPressed: () {}),
                InputChip(label: const Text('Input'), onDeleted: () {}),
              ],
            ),

            const SizedBox(height: 16),

            // Cards
            Text('Cards', style: Theme.of(context).textTheme.labelLarge),
            const SizedBox(height: 8),
            Row(
              children: [
                Expanded(
                  child: Card(
                    child: Padding(
                      padding: const EdgeInsets.all(12),
                      child: Text('Elevated', style: Theme.of(context).textTheme.bodyMedium),
                    ),
                  ),
                ),
                const SizedBox(width: 8),
                Expanded(
                  child: Card.filled(
                    child: Padding(
                      padding: const EdgeInsets.all(12),
                      child: Text('Filled', style: Theme.of(context).textTheme.bodyMedium),
                    ),
                  ),
                ),
                const SizedBox(width: 8),
                Expanded(
                  child: Card.outlined(
                    child: Padding(
                      padding: const EdgeInsets.all(12),
                      child: Text('Outlined', style: Theme.of(context).textTheme.bodyMedium),
                    ),
                  ),
                ),
              ],
            ),

            const SizedBox(height: 16),

            // TextField
            Text('Text Field', style: Theme.of(context).textTheme.labelLarge),
            const SizedBox(height: 8),
            const TextField(
              decoration: InputDecoration(
                labelText: 'ตัวอย่าง TextField',
                prefixIcon: Icon(Icons.search),
                hintText: 'พิมพ์อะไรก็ได้...',
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

## Theme Extension

Custom theme extension สำหรับเพิ่ม color/style พิเศษ

```dart
// กำหนด extension
@immutable
class AppColorExtension extends ThemeExtension<AppColorExtension> {
  final Color? success;
  final Color? warning;
  final Color? info;
  final Color? onSuccess;
  final Color? onWarning;
  final Color? onInfo;

  const AppColorExtension({
    required this.success,
    required this.warning,
    required this.info,
    required this.onSuccess,
    required this.onWarning,
    required this.onInfo,
  });

  @override
  ThemeExtension<AppColorExtension> copyWith({
    Color? success,
    Color? warning,
    Color? info,
    Color? onSuccess,
    Color? onWarning,
    Color? onInfo,
  }) {
    return AppColorExtension(
      success: success ?? this.success,
      warning: warning ?? this.warning,
      info: info ?? this.info,
      onSuccess: onSuccess ?? this.onSuccess,
      onWarning: onWarning ?? this.onWarning,
      onInfo: onInfo ?? this.onInfo,
    );
  }

  @override
  ThemeExtension<AppColorExtension> lerp(
      ThemeExtension<AppColorExtension>? other, double t) {
    if (other is! AppColorExtension) return this;
    return AppColorExtension(
      success: Color.lerp(success, other.success, t),
      warning: Color.lerp(warning, other.warning, t),
      info: Color.lerp(info, other.info, t),
      onSuccess: Color.lerp(onSuccess, other.onSuccess, t),
      onWarning: Color.lerp(onWarning, other.onWarning, t),
      onInfo: Color.lerp(onInfo, other.onInfo, t),
    );
  }
}

// เพิ่มใน ThemeData
ThemeData(
  extensions: [
    const AppColorExtension(
      success: Color(0xFF4CAF50),
      warning: Color(0xFFFF9800),
      info: Color(0xFF2196F3),
      onSuccess: Colors.white,
      onWarning: Colors.white,
      onInfo: Colors.white,
    ),
  ],
)

// ใช้งาน
final appColors = Theme.of(context).extension<AppColorExtension>()!;
Container(color: appColors.success)
```

---

## สรุปบทที่ 29

ในบทนี้เราได้เรียนรู้:

1. **ThemeData**: การ customize ทุก component
2. **ColorScheme**: Material 3 color system
3. **Dark Mode**: ThemeMode, toggle theme
4. **Custom Fonts**: Google Fonts, local fonts
5. **Material 3 Components**: Button, Card, Badge, etc.
6. **Theme Extension**: custom colors
7. **Workshop**: App with light/dark theme toggle

บทต่อไปเราจะเรียน Animations
