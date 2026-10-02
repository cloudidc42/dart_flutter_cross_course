# Part 01: แนะนำ Dart และการติดตั้ง Development Environment

## 🎯 เป้าหมายของ Part นี้
- เข้าใจว่า Dart คืออะไร และทำไมต้องเรียน
- ติดตั้ง Dart SDK และ Flutter SDK
- ตั้งค่า IDE (VS Code / Android Studio)
- เขียนโปรแกรม Dart แรก
- เข้าใจโครงสร้างพื้นฐานของโปรแกรม Dart

---

## 1. Dart คืออะไร?

Dart เป็นภาษาโปรแกรมที่พัฒนาโดย Google ในปี 2011 ออกแบบมาเพื่อ:
- **Client-side development** - สร้าง UI สำหรับ Web, Mobile, Desktop
- **Server-side development** - Backend services
- **Strong typing** - ป้องกัน bugs ตั้งแต่ compile time
- **Null Safety** - หลีกเลี่ยง Null Pointer Exceptions

### คุณสมบัติเด่นของ Dart
```
✅ Strongly Typed
✅ Object-Oriented
✅ Garbage Collected
✅ AOT & JIT Compilation
✅ Null Safety by default
✅ Async/Await support
✅ Rich standard library
```

### Dart เทียบกับภาษาอื่น
| คุณสมบัติ | Dart | JavaScript | Python | Java |
|----------|------|-----------|--------|------|
| Type System | Static | Dynamic | Dynamic | Static |
| Null Safety | ✅ Built-in | ❌ | ❌ | Partial |
| Performance | Fast | Medium | Slow | Fast |
| Learning Curve | Easy | Easy | Easy | Medium |
| Mobile Dev | ✅ Flutter | React Native | Limited | ✅ Android |

---

## 2. Flutter คืออะไร?

Flutter เป็น UI Toolkit ที่สร้างด้วย Dart ช่วยให้สร้างแอปได้หลายแพลตฟอร์มจากโค้ดเดียว:

```
Flutter App
    │
    ├── iOS App
    ├── Android App
    ├── Web App
    ├── Windows App
    ├── macOS App
    └── Linux App
```

### ทำไมต้องใช้ Flutter?
1. **Write Once, Run Anywhere** - โค้ดเดียวทำงานได้ทุกที่
2. **Hot Reload** - เห็นผลการเปลี่ยนแปลงทันที
3. **Native Performance** - เร็วเท่า Native Apps
4. **Beautiful UI** - Material Design & Cupertino (iOS style)
5. **Large Community** - มี packages มากกว่า 30,000+

---

## 3. ติดตั้ง Development Environment

### 3.1 ติดตั้ง Flutter SDK

#### สำหรับ Windows:
```powershell
# 1. ดาวน์โหลด Flutter SDK จาก flutter.dev
# 2. แตกไฟล์ไปที่ C:\flutter
# 3. เพิ่ม PATH environment variable

# ใน PowerShell (Admin):
$env:PATH += ";C:\flutter\bin"

# หรือเพิ่มผ่าน System Environment Variables:
# Control Panel > System > Advanced > Environment Variables
# เพิ่ม C:\flutter\bin ใน PATH
```

#### สำหรับ macOS:
```bash
# ใช้ Homebrew (แนะนำ):
brew install flutter

# หรือดาวน์โหลด manually:
# 1. ดาวน์โหลดจาก flutter.dev
# 2. แตกไฟล์ไปที่ ~/flutter
# 3. เพิ่ม PATH ใน ~/.zshrc หรือ ~/.bashrc:
echo 'export PATH="$PATH:~/flutter/bin"' >> ~/.zshrc
source ~/.zshrc
```

#### สำหรับ Linux:
```bash
# Ubuntu/Debian:
sudo apt update
sudo apt install snapd
sudo snap install flutter --classic

# หรือ manual install:
wget https://storage.googleapis.com/flutter_infra_release/releases/stable/linux/flutter_linux_3.x.x-stable.tar.xz
tar xf flutter_linux_*.tar.xz
export PATH="$PATH:`pwd`/flutter/bin"
```

### 3.2 ตรวจสอบการติดตั้ง
```bash
# ตรวจสอบ Flutter version:
flutter --version

# ตรวจสอบ dependencies ทั้งหมด:
flutter doctor

# Output ที่ควรได้:
# Doctor summary (to see all details, run flutter doctor -v):
# [✓] Flutter (Channel stable, 3.x.x, ...)
# [✓] Android toolchain - develop for Android devices
# [✓] Xcode - develop for iOS and macOS
# [✓] Chrome - develop for the web
# [✓] Android Studio (version x.x)
# [✓] VS Code (version x.x)
```

### 3.3 ติดตั้ง Android Studio
```
1. ดาวน์โหลดจาก developer.android.com/studio
2. ติดตั้งตามขั้นตอน
3. เปิด Android Studio > SDK Manager
4. ติดตั้ง Android SDK
5. ติดตั้ง Flutter plugin:
   File > Settings > Plugins > ค้นหา "Flutter" > Install
```

### 3.4 ติดตั้ง VS Code (แนะนำสำหรับผู้เริ่มต้น)
```
1. ดาวน์โหลดจาก code.visualstudio.com
2. ติดตั้ง Extensions:
   - Flutter (Dart Code)
   - Dart (Dart Code)
   - Error Lens
   - Bracket Pair Colorizer
   - GitLens
```

---

## 4. ตั้งค่า Android Emulator

```bash
# สร้าง Virtual Device ใน Android Studio:
# Tools > AVD Manager > Create Virtual Device

# แนะนำ settings:
# - Device: Pixel 6
# - System Image: Android 13 (API 33)
# - RAM: 2048 MB
# - Storage: 2 GB

# เปิด Emulator จาก command line:
flutter emulators --launch <emulator_id>

# ดู emulators ที่มี:
flutter emulators
```

---

## 5. สร้างโปรเจค Flutter แรก

```bash
# สร้างโปรเจคใหม่:
flutter create my_first_app

# หรือพร้อม organization name:
flutter create --org com.example my_first_app

# เข้าไปในโปรเจค:
cd my_first_app

# รันแอป:
flutter run
```

### โครงสร้างโปรเจค Flutter
```
my_first_app/
├── android/          # Android-specific code
├── ios/              # iOS-specific code
├── lib/              # Dart source code (ส่วนที่เราแก้ไขหลัก)
│   └── main.dart     # Entry point
├── test/             # Unit tests
├── web/              # Web-specific code
├── windows/          # Windows-specific code
├── macos/            # macOS-specific code
├── linux/            # Linux-specific code
├── pubspec.yaml      # Dependencies & metadata
└── README.md
```

---

## 6. โปรแกรม Dart แรก

### 6.1 Dart Standalone (ไม่ใช้ Flutter)

สร้างไฟล์ `hello.dart`:
```dart
// hello.dart - โปรแกรม Dart แรก

void main() {
  // แสดงข้อความ
  print('สวัสดี Dart! Hello, World!');
  
  // ตัวแปรพื้นฐาน
  String name = 'Flutter Developer';
  int age = 25;
  double version = 3.0;
  bool isAwesome = true;
  
  // String interpolation
  print('ฉันคือ $name อายุ $age ปี');
  print('Flutter version: $version');
  print('Dart is awesome: $isAwesome');
  
  // การคำนวณ
  int a = 10;
  int b = 3;
  print('$a + $b = ${a + b}');
  print('$a - $b = ${a - b}');
  print('$a * $b = ${a * b}');
  print('$a / $b = ${a / b}');
  print('$a ~/ $b = ${a ~/ b}'); // Integer division
  print('$a % $b = ${a % b}');   // Modulo
}
```

รัน:
```bash
dart run hello.dart
```

Output:
```
สวัสดี Dart! Hello, World!
ฉันคือ Flutter Developer อายุ 25 ปี
Flutter version: 3.0
Dart is awesome: true
10 + 3 = 13
10 - 3 = 7
10 * 3 = 30
10 / 3 = 3.3333333333333335
10 ~/ 3 = 3
10 % 3 = 1
```

---

## 7. Flutter App แรก

แก้ไขไฟล์ `lib/main.dart`:
```dart
import 'package:flutter/material.dart';

// Entry point ของแอป
void main() {
  runApp(const MyApp());
}

// Root Widget ของแอป
class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'แอปแรกของฉัน',
      theme: ThemeData(
        // ใช้ Material 3 design
        useMaterial3: true,
        colorScheme: ColorScheme.fromSeed(
          seedColor: Colors.blue,
        ),
      ),
      home: const HomePage(),
    );
  }
}

// หน้าหลัก
class HomePage extends StatefulWidget {
  const HomePage({super.key});

  @override
  State<HomePage> createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  int _counter = 0; // นับจำนวนการกด

  // ฟังก์ชันเพิ่มค่า counter
  void _incrementCounter() {
    setState(() {
      _counter++;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        backgroundColor: Theme.of(context).colorScheme.inversePrimary,
        title: const Text('แอป Flutter แรก'),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            const Text(
              'กดปุ่มเพื่อนับ:',
              style: TextStyle(fontSize: 18),
            ),
            const SizedBox(height: 16),
            Text(
              '$_counter',
              style: Theme.of(context).textTheme.displayLarge,
            ),
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: _incrementCounter,
        tooltip: 'เพิ่ม',
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

---

## 8. ทำความเข้าใจ Flutter Hot Reload

Hot Reload คือคุณสมบัติที่ทำให้เห็นการเปลี่ยนแปลงทันที:

```bash
# รัน Flutter app:
flutter run

# คำสั่งระหว่าง run:
# r = Hot reload (เร็ว, รักษา state)
# R = Hot restart (ช้ากว่า, reset state)
# q = Quit
# h = Help
```

### ความแตกต่าง Hot Reload vs Hot Restart
```
Hot Reload:
- เร็วมาก (~1 วินาที)
- รักษา state ของแอป
- ใช้สำหรับ UI changes

Hot Restart:
- ช้ากว่า (~5 วินาที)
- Reset state ทั้งหมด
- ใช้สำหรับ logic changes
```

---

## 9. Flutter DevTools

```bash
# เปิด DevTools:
flutter pub global activate devtools
flutter pub global run devtools

# หรือกด 'v' ระหว่าง flutter run
# แล้วเปิด http://localhost:9100

# DevTools มีเครื่องมือ:
# - Widget Inspector
# - Performance Profiler
# - Memory Profiler
# - Network Monitor
# - Debugger
```

---

## 10. pubspec.yaml - ไฟล์สำคัญ

```yaml
# pubspec.yaml - ไฟล์กำหนด configuration และ dependencies

name: my_first_app
description: "แอปแรกของฉัน"

publish_to: 'none' # ไม่ publish ขึ้น pub.dev

version: 1.0.0+1 # version: major.minor.patch+build

environment:
  sdk: '>=3.0.0 <4.0.0' # Dart SDK version ที่รองรับ

dependencies:
  flutter:
    sdk: flutter
  
  # เพิ่ม packages ที่ต้องการ:
  http: ^1.1.0          # HTTP requests
  shared_preferences: ^2.2.0 # Local storage
  provider: ^6.1.0      # State management

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0

flutter:
  uses-material-design: true
  
  # เพิ่ม assets:
  assets:
    - assets/images/
    - assets/icons/
  
  # เพิ่ม fonts:
  fonts:
    - family: Sarabun
      fonts:
        - asset: assets/fonts/Sarabun-Regular.ttf
        - asset: assets/fonts/Sarabun-Bold.ttf
          weight: 700
```

### การติดตั้ง packages:
```bash
# เพิ่ม package ใน pubspec.yaml แล้วรัน:
flutter pub get

# หรือใช้ command line:
flutter pub add http
flutter pub add provider

# อัปเดต packages:
flutter pub upgrade

# ดู packages ที่ outdated:
flutter pub outdated
```

---

## 11. ทำความเข้าใจ Widget

ทุกอย่างใน Flutter คือ Widget:

```dart
// Widget ทั่วไปที่ใช้บ่อย:

// Text Widget
Text('สวัสดี', style: TextStyle(fontSize: 24))

// Container Widget (กล่อง)
Container(
  width: 200,
  height: 100,
  color: Colors.blue,
  child: Text('ข้อความ'),
)

// Row Widget (แนวนอน)
Row(
  children: [
    Icon(Icons.star),
    Text('5.0'),
  ],
)

// Column Widget (แนวตั้ง)
Column(
  children: [
    Text('บน'),
    Text('กลาง'),
    Text('ล่าง'),
  ],
)

// Button Widget
ElevatedButton(
  onPressed: () => print('กดปุ่ม!'),
  child: Text('กดฉัน'),
)
```

---

## 12. Workshop: สร้าง Profile Card App

ลองสร้างแอปแสดงข้อมูลส่วนตัว:

```dart
// lib/main.dart
import 'package:flutter/material.dart';

void main() {
  runApp(const ProfileApp());
}

class ProfileApp extends StatelessWidget {
  const ProfileApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Profile Card',
      theme: ThemeData(
        useMaterial3: true,
        colorScheme: ColorScheme.fromSeed(
          seedColor: const Color(0xFF6750A4),
        ),
      ),
      home: const ProfilePage(),
    );
  }
}

class ProfilePage extends StatelessWidget {
  const ProfilePage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: const Color(0xFF6750A4),
      body: Center(
        child: Container(
          width: 320,
          padding: const EdgeInsets.all(24),
          decoration: BoxDecoration(
            color: Colors.white,
            borderRadius: BorderRadius.circular(16),
            boxShadow: [
              BoxShadow(
                color: Colors.black.withOpacity(0.2),
                blurRadius: 20,
                offset: const Offset(0, 10),
              ),
            ],
          ),
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              // รูปโปรไฟล์
              CircleAvatar(
                radius: 60,
                backgroundColor: const Color(0xFF6750A4),
                child: const Icon(
                  Icons.person,
                  size: 60,
                  color: Colors.white,
                ),
              ),
              const SizedBox(height: 16),
              
              // ชื่อ
              const Text(
                'สมชาย ใจดี',
                style: TextStyle(
                  fontSize: 24,
                  fontWeight: FontWeight.bold,
                  color: Color(0xFF1C1B1F),
                ),
              ),
              const SizedBox(height: 4),
              
              // ตำแหน่ง
              const Text(
                'Flutter Developer',
                style: TextStyle(
                  fontSize: 16,
                  color: Color(0xFF6750A4),
                ),
              ),
              const SizedBox(height: 24),
              
              // ข้อมูลติดต่อ
              _buildInfoRow(Icons.email, 'somchai@example.com'),
              const SizedBox(height: 12),
              _buildInfoRow(Icons.phone, '+66 81-234-5678'),
              const SizedBox(height: 12),
              _buildInfoRow(Icons.location_on, 'กรุงเทพมหานคร, ไทย'),
              const SizedBox(height: 24),
              
              // ปุ่ม
              Row(
                children: [
                  Expanded(
                    child: OutlinedButton.icon(
                      onPressed: () {},
                      icon: const Icon(Icons.message),
                      label: const Text('ส่งข้อความ'),
                    ),
                  ),
                  const SizedBox(width: 12),
                  Expanded(
                    child: ElevatedButton.icon(
                      onPressed: () {},
                      icon: const Icon(Icons.person_add),
                      label: const Text('ติดตาม'),
                    ),
                  ),
                ],
              ),
            ],
          ),
        ),
      ),
    );
  }

  Widget _buildInfoRow(IconData icon, String text) {
    return Row(
      children: [
        Icon(icon, size: 20, color: const Color(0xFF6750A4)),
        const SizedBox(width: 12),
        Text(
          text,
          style: const TextStyle(
            fontSize: 14,
            color: Color(0xFF49454F),
          ),
        ),
      ],
    );
  }
}
```

---

## 13. ทดสอบและ Debug

### ใช้ print() สำหรับ debugging
```dart
void main() {
  String name = 'Dart';
  int count = 42;
  
  // print ธรรมดา
  print('สวัสดี $name');
  
  // debugPrint - ดีกว่าสำหรับ Flutter (ไม่ truncate output)
  debugPrint('Count: $count');
  
  // assert - สำหรับ debug mode เท่านั้น
  assert(count > 0, 'Count ต้องมากกว่า 0');
  
  // log จาก dart:developer
  import 'dart:developer';
  log('Debug message', name: 'MyApp');
}
```

### Breakpoints ใน VS Code
```
1. คลิกด้านซ้ายของบรรทัดที่ต้องการ pause
2. กด F5 เพื่อ run ใน debug mode
3. ใช้ Debug toolbar:
   - F10 = Step Over
   - F11 = Step Into
   - Shift+F11 = Step Out
   - F5 = Continue
```

---

## 14. Common Errors และวิธีแก้

### Error 1: SDK Not Found
```bash
# ปัญหา:
# 'flutter' is not recognized as an internal or external command

# แก้ไข:
# เพิ่ม flutter/bin ใน PATH environment variable
```

### Error 2: No devices found
```bash
# ปัญหา:
# No devices found when running flutter run

# แก้ไข:
flutter devices          # ดู devices ที่มี
flutter emulators        # ดู emulators ที่มี
flutter run -d chrome    # รันบน Chrome
flutter run -d web-server # รันเป็น web server
```

### Error 3: Dependency conflicts
```bash
# ปัญหา:
# Because package requires X >= 2.0.0, version solving failed

# แก้ไข:
flutter pub upgrade --major-versions  # อัปเกรด major versions
# หรือแก้ version ใน pubspec.yaml ด้วยตัวเอง
```

### Error 4: Build failed
```bash
# clean แล้ว build ใหม่:
flutter clean
flutter pub get
flutter run
```

---

## 15. สรุป Part 01

สิ่งที่เรียนรู้ใน Part นี้:
- ✅ Dart คืออะไรและมีคุณสมบัติอะไร
- ✅ Flutter คืออะไรและทำไมต้องใช้
- ✅ ติดตั้ง Flutter SDK และ IDE
- ✅ สร้างโปรเจค Flutter แรก
- ✅ เข้าใจโครงสร้างโปรเจค
- ✅ เขียนโปรแกรม Dart และ Flutter พื้นฐาน
- ✅ Hot Reload และ DevTools
- ✅ pubspec.yaml และการจัดการ packages
- ✅ Widget พื้นฐาน

---

## 📚 แหล่งเรียนรู้เพิ่มเติม
- [Flutter Official Docs](https://flutter.dev/docs)
- [Dart Official Docs](https://dart.dev/guides)
- [pub.dev](https://pub.dev) - Flutter packages
- [Flutter Cookbook](https://flutter.dev/cookbook)
- [DartPad](https://dartpad.dev) - ทดลองเขียน Dart online

---

## ➡️ Part ถัดไป
**Part 02: ตัวแปร, ประเภทข้อมูล และ Null Safety**

เราจะเรียนรู้เกี่ยวกับ:
- ประเภทข้อมูลพื้นฐานทั้งหมดใน Dart
- var, final, const ต่างกันอย่างไร
- Null Safety คืออะไรและทำงานอย่างไร
- Type inference
