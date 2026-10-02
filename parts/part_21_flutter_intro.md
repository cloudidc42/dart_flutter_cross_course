# Part 21: Flutter Introduction and Project Structure

## บทนำ

Flutter คือ UI framework แบบ open-source ที่พัฒนาโดย Google สำหรับสร้างแอปพลิเคชันแบบ cross-platform ที่ทำงานได้บน iOS, Android, Web, Desktop (Windows, macOS, Linux) จากโค้ดชุดเดียว

### ทำไมต้องเลือก Flutter?

- **Single Codebase**: เขียนโค้ดครั้งเดียว รันได้ทุก platform
- **High Performance**: ใช้ Dart language ที่ compile เป็น native code
- **Beautiful UI**: Widget system ที่ยืดหยุ่น สวยงาม
- **Hot Reload**: เห็นการเปลี่ยนแปลงทันทีโดยไม่ต้อง restart app
- **Rich Ecosystem**: Package และ plugin มากมาย

---

## Flutter Architecture

Flutter มีสถาปัตยกรรมที่ประกอบด้วย 3 tree หลัก:

### 1. Widget Tree

Widget Tree คือโครงสร้างของ UI ที่เราเขียน เป็น immutable (ไม่เปลี่ยนแปลงหลังสร้าง)

```
MaterialApp
└── Scaffold
    ├── AppBar
    │   └── Text("My App")
    └── Body
        └── Center
            └── Column
                ├── Text("Hello")
                └── ElevatedButton
                    └── Text("Click me")
```

### 2. Element Tree

Element Tree คือ "instance" ที่ Flutter สร้างจาก Widget ทำหน้าที่เป็น mediator ระหว่าง Widget Tree และ Render Tree

```dart
// Widget (blueprint/description)
class MyWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Container(color: Colors.blue);
  }
}

// Element (instance ที่ Flutter สร้างภายใน)
// StatelessElement, StatefulElement, etc.
```

Element จะ:
- Track ว่า Widget ไหน correspond กับ node ไหนใน tree
- ช่วยในการ reconcile เมื่อ Widget เปลี่ยน
- เก็บ State ของ StatefulWidget

### 3. Render Tree

Render Tree คือสิ่งที่วาดบนหน้าจอจริงๆ แต่ละ node (RenderObject) รู้จัก:
- ขนาดของตัวเอง (size/layout)
- ตำแหน่ง (position/paint)
- วิธีวาด (paint)

```
Flutter Framework
├── Widget Layer (เราเขียน)
├── Element Layer (Flutter จัดการ)
└── Render Layer (วาดบนหน้าจอ)

Flutter Engine (C++)
├── Skia/Impeller (Graphics)
├── Text rendering
└── Platform channels

Platform (iOS/Android/etc.)
```

### ทำไม Flutter เร็ว?

Flutter ไม่ใช้ native UI components ของแต่ละ platform แต่วาด UI เองทั้งหมดด้วย Skia/Impeller engine ทำให้:
- UI เหมือนกันทุก platform
- ควบคุม rendering ได้ 100%
- Target 60/120 fps

---

## Project Structure

เมื่อสร้าง Flutter project ใหม่ด้วย `flutter create my_app` จะได้โครงสร้างดังนี้:

```
my_app/
├── android/              # Android-specific code
├── ios/                  # iOS-specific code
├── web/                  # Web-specific code
├── macos/                # macOS-specific code
├── windows/              # Windows-specific code
├── linux/                # Linux-specific code
├── lib/                  # Dart source code (หลัก)
│   └── main.dart         # Entry point
├── test/                 # Unit & widget tests
│   └── widget_test.dart
├── assets/               # รูปภาพ, fonts (ต้องสร้างเอง)
├── pubspec.yaml          # Project configuration
├── pubspec.lock          # Lock file (auto-generated)
├── .gitignore
└── README.md
```

### โครงสร้าง lib/ ที่แนะนำ

สำหรับ project ขนาดกลาง-ใหญ่ ควรจัดระเบียบดังนี้:

```
lib/
├── main.dart
├── app.dart              # MaterialApp configuration
├── core/
│   ├── constants/        # Colors, sizes, strings
│   ├── theme/            # ThemeData
│   └── utils/            # Helper functions
├── features/
│   ├── home/
│   │   ├── screens/
│   │   ├── widgets/
│   │   └── models/
│   └── auth/
│       ├── screens/
│       └── widgets/
└── shared/
    ├── widgets/          # Reusable widgets
    └── models/           # Shared models
```

---

## main.dart Walkthrough

```dart
// import Flutter Material Design library
import 'package:flutter/material.dart';

// Entry point ของ Dart program
void main() {
  // runApp() เริ่มต้น Flutter framework
  // รับ Widget ที่จะเป็น root ของ Widget tree
  runApp(const MyApp());
}

// Root Widget ของแอพ
class MyApp extends StatelessWidget {
  // const constructor สำหรับ performance
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    // MaterialApp ให้ Material Design functionality
    return MaterialApp(
      title: 'My Flutter App',
      // ซ่อน "debug" banner ที่มุมขวาบน
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue),
        useMaterial3: true,
      ),
      home: const MyHomePage(title: 'Flutter Demo'),
    );
  }
}

class MyHomePage extends StatefulWidget {
  const MyHomePage({super.key, required this.title});
  
  final String title;

  @override
  State<MyHomePage> createState() => _MyHomePageState();
}

class _MyHomePageState extends State<MyHomePage> {
  int _counter = 0;

  void _incrementCounter() {
    // setState บอก Flutter ว่า state เปลี่ยน ให้ rebuild widget
    setState(() {
      _counter++;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        backgroundColor: Theme.of(context).colorScheme.inversePrimary,
        title: Text(widget.title),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            const Text('You have pushed the button this many times:'),
            Text(
              '$_counter',
              style: Theme.of(context).textTheme.headlineMedium,
            ),
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: _incrementCounter,
        tooltip: 'Increment',
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

---

## MaterialApp vs CupertinoApp

### MaterialApp

ใช้ Material Design (Google) - เหมาะสำหรับ Android และแอพทั่วไป

```dart
import 'package:flutter/material.dart';

class App extends StatelessWidget {
  const App({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'My App',
      debugShowCheckedModeBanner: false,
      theme: ThemeData.light(),
      darkTheme: ThemeData.dark(),
      themeMode: ThemeMode.system,
      initialRoute: '/',
      routes: {
        '/': (context) => const HomeScreen(),
        '/settings': (context) => const SettingsScreen(),
      },
      // หรือใช้ home แทน initialRoute
      // home: const HomeScreen(),
    );
  }
}
```

### CupertinoApp

ใช้ Cupertino Design (Apple) - เหมาะสำหรับ iOS-style app

```dart
import 'package:flutter/cupertino.dart';

class App extends StatelessWidget {
  const App({super.key});

  @override
  Widget build(BuildContext context) {
    return CupertinoApp(
      title: 'My iOS App',
      debugShowCheckedModeBanner: false,
      theme: const CupertinoThemeData(
        primaryColor: CupertinoColors.systemBlue,
        brightness: Brightness.light,
      ),
      home: const CupertinoPageScaffold(
        navigationBar: CupertinoNavigationBar(
          middle: Text('Home'),
        ),
        child: Center(
          child: Text('Hello iOS!'),
        ),
      ),
    );
  }
}
```

### ใช้ทั้งสองใน MaterialApp

```dart
// Scaffold สำหรับ Material
Scaffold(
  appBar: AppBar(title: Text('Material')),
  body: Column(
    children: [
      // Material widget
      ElevatedButton(onPressed: () {}, child: Text('Material Button')),
      // Cupertino widget ใน Material app (ต้องใช้ Material() wrapper)
      CupertinoButton(
        onPressed: () {},
        child: Text('iOS Button'),
      ),
    ],
  ),
)
```

---

## ThemeData

ThemeData กำหนดรูปแบบ (look and feel) ของแอพทั้งหมด

```dart
ThemeData customTheme = ThemeData(
  // Material 3
  useMaterial3: true,
  
  // Color Scheme (แนะนำใช้ ColorScheme.fromSeed)
  colorScheme: ColorScheme.fromSeed(
    seedColor: const Color(0xFF6750A4), // สีหลัก
    brightness: Brightness.light,
  ),
  
  // หรือกำหนด ColorScheme เองทั้งหมด
  // colorScheme: const ColorScheme(
  //   primary: Color(0xFF6750A4),
  //   onPrimary: Colors.white,
  //   secondary: Color(0xFF625B71),
  //   onSecondary: Colors.white,
  //   error: Color(0xFFB3261E),
  //   onError: Colors.white,
  //   background: Color(0xFFFFFBFE),
  //   onBackground: Color(0xFF1C1B1F),
  //   surface: Color(0xFFFFFBFE),
  //   onSurface: Color(0xFF1C1B1F),
  //   brightness: Brightness.light,
  // ),
  
  // Typography
  textTheme: const TextTheme(
    displayLarge: TextStyle(
      fontSize: 57,
      fontWeight: FontWeight.w400,
      letterSpacing: -0.25,
    ),
    headlineMedium: TextStyle(
      fontSize: 28,
      fontWeight: FontWeight.w400,
    ),
    bodyLarge: TextStyle(
      fontSize: 16,
      fontWeight: FontWeight.w400,
    ),
    labelLarge: TextStyle(
      fontSize: 14,
      fontWeight: FontWeight.w500,
    ),
  ),
  
  // Font Family
  fontFamily: 'Kanit', // ต้องเพิ่มใน pubspec.yaml
  
  // AppBar theme
  appBarTheme: const AppBarTheme(
    centerTitle: true,
    elevation: 0,
    backgroundColor: Colors.transparent,
  ),
  
  // Button themes
  elevatedButtonTheme: ElevatedButtonThemeData(
    style: ElevatedButton.styleFrom(
      padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 12),
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.circular(12),
      ),
    ),
  ),
  
  // Input decoration theme
  inputDecorationTheme: InputDecorationTheme(
    border: OutlineInputBorder(
      borderRadius: BorderRadius.circular(12),
    ),
    filled: true,
  ),
  
  // Card theme
  cardTheme: CardTheme(
    elevation: 2,
    shape: RoundedRectangleBorder(
      borderRadius: BorderRadius.circular(16),
    ),
  ),
);
```

### การใช้ Theme ใน Widget

```dart
// ดึง ThemeData
final theme = Theme.of(context);

// ใช้สี
Container(
  color: theme.colorScheme.primary,
  child: Text(
    'Hello',
    style: TextStyle(color: theme.colorScheme.onPrimary),
  ),
)

// ใช้ Text styles
Text(
  'Title',
  style: theme.textTheme.headlineMedium,
)

// Override theme เฉพาะส่วน
Theme(
  data: theme.copyWith(
    elevatedButtonTheme: ElevatedButtonThemeData(
      style: ElevatedButton.styleFrom(
        backgroundColor: Colors.red,
      ),
    ),
  ),
  child: ElevatedButton(
    onPressed: () {},
    child: const Text('Red Button'),
  ),
)
```

---

## pubspec.yaml Deep Dive

`pubspec.yaml` คือไฟล์ configuration หลักของ Flutter project

```yaml
name: my_flutter_app
description: A new Flutter project.

# Prevents accidental publishing to pub.dev
publish_to: 'none'

# Version (major.minor.patch+build)
version: 1.0.0+1

environment:
  sdk: '>=3.0.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter
  
  # UI
  cupertino_icons: ^1.0.6
  
  # State management
  provider: ^6.1.1
  # หรือ
  # riverpod: ^2.4.9
  # flutter_bloc: ^8.1.3
  
  # HTTP & API
  http: ^1.1.2
  dio: ^5.4.0
  
  # Local storage
  shared_preferences: ^2.2.2
  hive: ^2.2.3
  hive_flutter: ^1.1.0
  
  # Navigation
  go_router: ^13.0.0
  
  # JSON
  json_annotation: ^4.8.1
  
  # Images
  cached_network_image: ^3.3.1
  
  # Date/time
  intl: ^0.19.0
  
  # Utilities
  equatable: ^2.0.5
  uuid: ^4.2.2
  
  # Logging
  logger: ^2.0.2+1

dev_dependencies:
  flutter_test:
    sdk: flutter
  
  flutter_lints: ^3.0.0
  
  # Code generation
  build_runner: ^2.4.7
  json_serializable: ^6.7.1
  hive_generator: ^2.0.1

flutter:
  uses-material-design: true
  
  # Assets
  assets:
    - assets/images/
    - assets/icons/
    - assets/data/config.json
  
  # Custom fonts
  fonts:
    - family: Kanit
      fonts:
        - asset: assets/fonts/Kanit-Regular.ttf
        - asset: assets/fonts/Kanit-Bold.ttf
          weight: 700
        - asset: assets/fonts/Kanit-Light.ttf
          weight: 300
    
    - family: Prompt
      fonts:
        - asset: assets/fonts/Prompt-Regular.ttf
        - asset: assets/fonts/Prompt-Medium.ttf
          weight: 500
```

### การจัดการ Dependencies

```bash
# เพิ่ม package
flutter pub add package_name

# ลบ package
flutter pub remove package_name

# อัพเดท dependencies
flutter pub upgrade

# ดู dependencies ที่ outdated
flutter pub outdated

# ดาวน์โหลด dependencies
flutter pub get
```

### Version Constraints

```yaml
dependencies:
  # Exact version
  package: 1.2.3
  
  # Any version >= 1.2.3
  package: '>=1.2.3'
  
  # Compatible versions (1.x.x)
  package: ^1.2.3  # เท่ากับ >=1.2.3 <2.0.0
  
  # Patch versions only (1.2.x)
  package: '~1.2.3'  # เท่ากับ >=1.2.3 <1.3.0
  
  # Any version
  package: any
  
  # Local package
  my_package:
    path: ../my_package
  
  # Git package
  my_package:
    git:
      url: https://github.com/user/my_package.git
      ref: main  # branch, tag หรือ commit
```

---

## Workshop: Hello Flutter App with Custom Theme

เราจะสร้างแอพ Hello Flutter ที่มี custom theme สวยงาม

### สร้าง Project

```bash
flutter create hello_flutter
cd hello_flutter
```

### lib/core/theme/app_theme.dart

```dart
import 'package:flutter/material.dart';

class AppTheme {
  AppTheme._(); // Private constructor - ไม่ให้ instantiate
  
  // Brand colors
  static const Color primaryColor = Color(0xFF5C6BC0);
  static const Color secondaryColor = Color(0xFF42A5F5);
  static const Color accentColor = Color(0xFFFF7043);
  
  // Light theme
  static ThemeData get lightTheme {
    final colorScheme = ColorScheme.fromSeed(
      seedColor: primaryColor,
      brightness: Brightness.light,
    );
    
    return ThemeData(
      useMaterial3: true,
      colorScheme: colorScheme,
      fontFamily: 'Kanit',
      
      appBarTheme: AppBarTheme(
        backgroundColor: colorScheme.surface,
        foregroundColor: colorScheme.onSurface,
        elevation: 0,
        centerTitle: true,
        titleTextStyle: TextStyle(
          color: colorScheme.onSurface,
          fontSize: 20,
          fontWeight: FontWeight.w600,
          fontFamily: 'Kanit',
        ),
      ),
      
      elevatedButtonTheme: ElevatedButtonThemeData(
        style: ElevatedButton.styleFrom(
          backgroundColor: colorScheme.primary,
          foregroundColor: colorScheme.onPrimary,
          padding: const EdgeInsets.symmetric(
            horizontal: 32,
            vertical: 16,
          ),
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(16),
          ),
          textStyle: const TextStyle(
            fontSize: 16,
            fontWeight: FontWeight.w600,
            fontFamily: 'Kanit',
          ),
        ),
      ),
      
      cardTheme: CardTheme(
        elevation: 4,
        shadowColor: Colors.black26,
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(20),
        ),
        clipBehavior: Clip.antiAlias,
      ),
      
      textTheme: const TextTheme(
        displayLarge: TextStyle(
          fontWeight: FontWeight.bold,
          letterSpacing: -1,
        ),
        headlineLarge: TextStyle(
          fontWeight: FontWeight.bold,
        ),
        headlineMedium: TextStyle(
          fontWeight: FontWeight.w600,
        ),
        bodyLarge: TextStyle(
          fontSize: 16,
          height: 1.6,
        ),
        bodyMedium: TextStyle(
          fontSize: 14,
          height: 1.5,
        ),
      ),
    );
  }
  
  // Dark theme
  static ThemeData get darkTheme {
    final colorScheme = ColorScheme.fromSeed(
      seedColor: primaryColor,
      brightness: Brightness.dark,
    );
    
    return ThemeData(
      useMaterial3: true,
      colorScheme: colorScheme,
      fontFamily: 'Kanit',
      
      appBarTheme: AppBarTheme(
        backgroundColor: colorScheme.surface,
        foregroundColor: colorScheme.onSurface,
        elevation: 0,
        centerTitle: true,
      ),
      
      elevatedButtonTheme: ElevatedButtonThemeData(
        style: ElevatedButton.styleFrom(
          padding: const EdgeInsets.symmetric(
            horizontal: 32,
            vertical: 16,
          ),
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(16),
          ),
        ),
      ),
    );
  }
}
```

### lib/main.dart

```dart
import 'package:flutter/material.dart';
import 'core/theme/app_theme.dart';
import 'features/home/screens/home_screen.dart';

void main() {
  runApp(const HelloFlutterApp());
}

class HelloFlutterApp extends StatelessWidget {
  const HelloFlutterApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Hello Flutter',
      debugShowCheckedModeBanner: false,
      theme: AppTheme.lightTheme,
      darkTheme: AppTheme.darkTheme,
      themeMode: ThemeMode.system,
      home: const HomeScreen(),
    );
  }
}
```

### lib/features/home/screens/home_screen.dart

```dart
import 'package:flutter/material.dart';

class HomeScreen extends StatefulWidget {
  const HomeScreen({super.key});

  @override
  State<HomeScreen> createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> {
  int _greetingIndex = 0;
  
  final List<Map<String, String>> _greetings = [
    {'text': 'สวัสดี Flutter!', 'emoji': '👋'},
    {'text': 'Hello Flutter!', 'emoji': '🌟'},
    {'text': 'Bonjour Flutter!', 'emoji': '🥐'},
    {'text': 'こんにちは Flutter!', 'emoji': '🗾'},
    {'text': '你好 Flutter!', 'emoji': '🐉'},
  ];

  void _nextGreeting() {
    setState(() {
      _greetingIndex = (_greetingIndex + 1) % _greetings.length;
    });
  }

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final greeting = _greetings[_greetingIndex];
    
    return Scaffold(
      backgroundColor: theme.colorScheme.background,
      appBar: AppBar(
        title: const Text('Hello Flutter'),
        actions: [
          IconButton(
            icon: const Icon(Icons.info_outline),
            onPressed: () {
              showAboutDialog(
                context: context,
                applicationName: 'Hello Flutter',
                applicationVersion: '1.0.0',
                applicationLegalese: 'Flutter Course Part 21',
              );
            },
          ),
        ],
      ),
      body: SafeArea(
        child: Padding(
          padding: const EdgeInsets.all(24.0),
          child: Column(
            children: [
              // Header Card
              _buildHeaderCard(theme, greeting),
              
              const SizedBox(height: 32),
              
              // Features Grid
              Expanded(
                child: _buildFeaturesGrid(theme),
              ),
            ],
          ),
        ),
      ),
      floatingActionButton: FloatingActionButton.extended(
        onPressed: _nextGreeting,
        icon: const Icon(Icons.translate),
        label: const Text('เปลี่ยนภาษา'),
        backgroundColor: theme.colorScheme.primary,
        foregroundColor: theme.colorScheme.onPrimary,
      ),
    );
  }

  Widget _buildHeaderCard(ThemeData theme, Map<String, String> greeting) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(32.0),
        child: Column(
          children: [
            // Emoji
            Text(
              greeting['emoji']!,
              style: const TextStyle(fontSize: 80),
            ),
            
            const SizedBox(height: 16),
            
            // Greeting Text
            Text(
              greeting['text']!,
              style: theme.textTheme.headlineMedium?.copyWith(
                color: theme.colorScheme.primary,
                fontWeight: FontWeight.bold,
              ),
              textAlign: TextAlign.center,
            ),
            
            const SizedBox(height: 8),
            
            Text(
              'ยินดีต้อนรับสู่ Flutter Course',
              style: theme.textTheme.bodyLarge?.copyWith(
                color: theme.colorScheme.onSurfaceVariant,
              ),
              textAlign: TextAlign.center,
            ),
            
            const SizedBox(height: 24),
            
            // Flutter logo colors indicator
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                _buildColorDot(const Color(0xFF54C5F8)),
                _buildColorDot(const Color(0xFF01579B)),
                _buildColorDot(const Color(0xFF29B6F6)),
                _buildColorDot(const Color(0xFF0175C2)),
              ],
            ),
          ],
        ),
      ),
    );
  }

  Widget _buildColorDot(Color color) {
    return Container(
      width: 16,
      height: 16,
      margin: const EdgeInsets.symmetric(horizontal: 4),
      decoration: BoxDecoration(
        color: color,
        shape: BoxShape.circle,
      ),
    );
  }

  Widget _buildFeaturesGrid(ThemeData theme) {
    final features = [
      {
        'icon': Icons.speed,
        'title': 'เร็ว',
        'subtitle': '60fps native performance',
        'color': Colors.orange,
      },
      {
        'icon': Icons.devices,
        'title': 'Cross-platform',
        'subtitle': 'iOS, Android, Web, Desktop',
        'color': Colors.blue,
      },
      {
        'icon': Icons.palette,
        'title': 'สวยงาม',
        'subtitle': 'Custom UI widgets',
        'color': Colors.purple,
      },
      {
        'icon': Icons.bolt,
        'title': 'Hot Reload',
        'subtitle': 'เห็นผลทันที',
        'color': Colors.green,
      },
    ];
    
    return GridView.builder(
      gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
        crossAxisCount: 2,
        crossAxisSpacing: 16,
        mainAxisSpacing: 16,
        childAspectRatio: 1.1,
      ),
      itemCount: features.length,
      itemBuilder: (context, index) {
        final feature = features[index];
        return _buildFeatureCard(theme, feature);
      },
    );
  }

  Widget _buildFeatureCard(ThemeData theme, Map<String, dynamic> feature) {
    final color = feature['color'] as Color;
    
    return Card(
      child: InkWell(
        onTap: () {},
        child: Padding(
          padding: const EdgeInsets.all(16.0),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Container(
                padding: const EdgeInsets.all(10),
                decoration: BoxDecoration(
                  color: color.withOpacity(0.1),
                  borderRadius: BorderRadius.circular(12),
                ),
                child: Icon(
                  feature['icon'] as IconData,
                  color: color,
                  size: 28,
                ),
              ),
              
              const Spacer(),
              
              Text(
                feature['title'] as String,
                style: theme.textTheme.titleMedium?.copyWith(
                  fontWeight: FontWeight.bold,
                ),
              ),
              
              const SizedBox(height: 4),
              
              Text(
                feature['subtitle'] as String,
                style: theme.textTheme.bodySmall?.copyWith(
                  color: theme.colorScheme.onSurfaceVariant,
                ),
                maxLines: 2,
                overflow: TextOverflow.ellipsis,
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

### pubspec.yaml สำหรับ Workshop

```yaml
name: hello_flutter
description: Hello Flutter Workshop App

publish_to: 'none'

version: 1.0.0+1

environment:
  sdk: '>=3.0.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter
  cupertino_icons: ^1.0.6

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0

flutter:
  uses-material-design: true
  
  assets:
    - assets/images/
  
  fonts:
    - family: Kanit
      fonts:
        - asset: assets/fonts/Kanit-Regular.ttf
        - asset: assets/fonts/Kanit-Bold.ttf
          weight: 700
```

---

## คำสั่ง Flutter ที่ใช้บ่อย

```bash
# สร้าง project ใหม่
flutter create project_name
flutter create --org com.company project_name
flutter create --template=app project_name

# Run app
flutter run
flutter run -d chrome          # Web
flutter run -d ios             # iOS
flutter run -d android         # Android

# Build
flutter build apk              # Android APK
flutter build appbundle        # Android App Bundle
flutter build ios              # iOS
flutter build web              # Web
flutter build macos            # macOS

# Test
flutter test
flutter test test/unit_test.dart

# Analyze
flutter analyze

# Format code
dart format lib/

# Check devices
flutter devices

# Upgrade Flutter
flutter upgrade

# Clean build cache
flutter clean

# Doctor
flutter doctor
flutter doctor -v              # Verbose
```

---

## สรุปบทที่ 21

ในบทนี้เราได้เรียนรู้:

1. **Flutter Architecture**: Widget tree, Element tree, Render tree
2. **Project Structure**: การจัดระเบียบไฟล์ใน Flutter project
3. **main.dart**: Entry point และ widget หลัก
4. **MaterialApp vs CupertinoApp**: ความแตกต่างและการใช้งาน
5. **ThemeData**: การกำหนด theme ให้แอพ
6. **pubspec.yaml**: การจัดการ dependencies และ assets

บทต่อไปเราจะเจาะลึก Widget Tree และ Flutter Architecture ในแง่มุมต่างๆ เพิ่มเติม
