# Part 58: Internationalization (i18n) ใน Flutter

## i18n คืออะไร?

Internationalization (i18n) คือกระบวนการทำให้แอปรองรับหลายภาษาและวัฒนธรรม เช่น:
- ภาษาไทย/อังกฤษ/ญี่ปุ่น
- รูปแบบวันที่และเวลา
- สกุลเงิน
- การอ่านจากขวาไปซ้าย (RTL)

```yaml
# pubspec.yaml
dependencies:
  flutter_localizations:
    sdk: flutter
  intl: ^0.19.0

flutter:
  generate: true # จำเป็นสำหรับ code generation
```

---

## 1. flutter_localizations Setup

### l10n.yaml Configuration

```yaml
# l10n.yaml (ไฟล์นี้อยู่ใน root ของ project)
arb-dir: lib/l10n
template-arb-file: app_en.arb
output-localization-file: app_localizations.dart
output-class: AppLocalizations
preferred-supported-locales:
  - en
  - th
```

### main.dart Setup

```dart
import 'package:flutter/material.dart';
import 'package:flutter_localizations/flutter_localizations.dart';
import 'package:flutter_gen/gen_l10n/app_localizations.dart';

void main() {
  runApp(MyApp());
}

class MyApp extends StatefulWidget {
  @override
  _MyAppState createState() => _MyAppState();
}

class _MyAppState extends State<MyApp> {
  Locale _locale = const Locale('th'); // ภาษาเริ่มต้น

  void _changeLocale(Locale newLocale) {
    setState(() {
      _locale = newLocale;
    });
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'My App',
      locale: _locale,
      
      // ต้องมี localizationsDelegates
      localizationsDelegates: const [
        AppLocalizations.delegate,
        GlobalMaterialLocalizations.delegate,
        GlobalWidgetsLocalizations.delegate,
        GlobalCupertinoLocalizations.delegate,
      ],
      
      // ภาษาที่รองรับ
      supportedLocales: const [
        Locale('en'), // English
        Locale('th'), // Thai
        Locale('ja'), // Japanese
      ],
      
      // Fallback locale ถ้าภาษาที่ต้องการไม่มี
      localeResolutionCallback: (deviceLocale, supportedLocales) {
        for (final locale in supportedLocales) {
          if (locale.languageCode == deviceLocale?.languageCode) {
            return locale;
          }
        }
        return const Locale('en'); // fallback เป็นภาษาอังกฤษ
      },
      
      home: HomeScreen(onLocaleChange: _changeLocale),
    );
  }
}
```

---

## 2. ARB Files

### app_en.arb (English)

```json
{
  "@@locale": "en",
  
  "appTitle": "My Shopping App",
  "@appTitle": {
    "description": "The title of the application"
  },
  
  "welcomeMessage": "Welcome, {name}!",
  "@welcomeMessage": {
    "description": "Welcome message with user's name",
    "placeholders": {
      "name": {
        "type": "String",
        "example": "John"
      }
    }
  },
  
  "itemCount": "{count, plural, =0{No items} =1{1 item} other{{count} items}}",
  "@itemCount": {
    "description": "Number of items in cart",
    "placeholders": {
      "count": {
        "type": "int"
      }
    }
  },
  
  "loginButton": "Log In",
  "logoutButton": "Log Out",
  "registerButton": "Register",
  
  "emailLabel": "Email Address",
  "passwordLabel": "Password",
  
  "errorRequired": "This field is required",
  "errorInvalidEmail": "Invalid email format",
  "errorPasswordTooShort": "Password must be at least {minLength} characters",
  "@errorPasswordTooShort": {
    "placeholders": {
      "minLength": {
        "type": "int"
      }
    }
  },
  
  "productPrice": "{price}",
  "@productPrice": {
    "placeholders": {
      "price": {
        "type": "double",
        "format": "currency",
        "optionalParameters": {
          "symbol": "$",
          "decimalDigits": 2
        }
      }
    }
  },
  
  "dateFormat": "{date}",
  "@dateFormat": {
    "placeholders": {
      "date": {
        "type": "DateTime",
        "format": "yMMMd"
      }
    }
  },
  
  "gender": "{gender, select, male{Mr.} female{Ms.} other{}}",
  "@gender": {
    "placeholders": {
      "gender": {
        "type": "String"
      }
    }
  }
}
```

### app_th.arb (Thai)

```json
{
  "@@locale": "th",
  
  "appTitle": "แอปช้อปปิ้งของฉัน",
  
  "welcomeMessage": "ยินดีต้อนรับ, {name}!",
  "@welcomeMessage": {
    "placeholders": {
      "name": {
        "type": "String"
      }
    }
  },
  
  "itemCount": "{count, plural, =0{ไม่มีสินค้า} =1{1 ชิ้น} other{{count} ชิ้น}}",
  "@itemCount": {
    "placeholders": {
      "count": {
        "type": "int"
      }
    }
  },
  
  "loginButton": "เข้าสู่ระบบ",
  "logoutButton": "ออกจากระบบ",
  "registerButton": "ลงทะเบียน",
  
  "emailLabel": "อีเมล",
  "passwordLabel": "รหัสผ่าน",
  
  "errorRequired": "กรุณากรอกข้อมูล",
  "errorInvalidEmail": "รูปแบบอีเมลไม่ถูกต้อง",
  "errorPasswordTooShort": "รหัสผ่านต้องมีอย่างน้อย {minLength} ตัวอักษร",
  "@errorPasswordTooShort": {
    "placeholders": {
      "minLength": {
        "type": "int"
      }
    }
  },
  
  "productPrice": "{price}",
  "@productPrice": {
    "placeholders": {
      "price": {
        "type": "double",
        "format": "currency",
        "optionalParameters": {
          "symbol": "฿",
          "decimalDigits": 2
        }
      }
    }
  },
  
  "dateFormat": "{date}",
  "@dateFormat": {
    "placeholders": {
      "date": {
        "type": "DateTime",
        "format": "yMMMd",
        "optionalParameters": {
          "locale": "th"
        }
      }
    }
  },
  
  "gender": "{gender, select, male{นาย} female{นาง/นางสาว} other{}}"
}
```

---

## 3. intl Package

### การจัดรูปแบบตัวเลขและวันที่

```dart
import 'package:intl/intl.dart';

class FormattingExamples {
  // จัดรูปแบบสกุลเงิน
  static String formatCurrency(double amount, String locale) {
    final format = NumberFormat.currency(
      locale: locale,
      symbol: locale == 'th' ? '฿' : '\$',
    );
    return format.format(amount);
  }

  // จัดรูปแบบวันที่
  static String formatDate(DateTime date, String locale) {
    final format = DateFormat.yMMMd(locale);
    return format.format(date);
  }

  // จัดรูปแบบวันที่และเวลา
  static String formatDateTime(DateTime dateTime, String locale) {
    final format = DateFormat('d MMM y, HH:mm', locale);
    return format.format(dateTime);
  }

  // จัดรูปแบบตัวเลข
  static String formatNumber(double number, String locale) {
    final format = NumberFormat.decimalPattern(locale);
    return format.format(number);
  }

  // แสดงเวลาสัมพัทธ์ (เช่น "3 ชั่วโมงที่แล้ว")
  static String formatRelativeTime(DateTime time, String locale) {
    final now = DateTime.now();
    final diff = now.difference(time);

    if (diff.inSeconds < 60) {
      return locale == 'th' ? 'เมื่อกี้' : 'Just now';
    } else if (diff.inMinutes < 60) {
      return locale == 'th'
          ? '${diff.inMinutes} นาทีที่แล้ว'
          : '${diff.inMinutes} minutes ago';
    } else if (diff.inHours < 24) {
      return locale == 'th'
          ? '${diff.inHours} ชั่วโมงที่แล้ว'
          : '${diff.inHours} hours ago';
    } else {
      return formatDate(time, locale);
    }
  }
}

// ตัวอย่างการใช้งาน
class FormattingDemo extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final locale = Localizations.localeOf(context).toString();
    final now = DateTime.now();
    final price = 1234567.89;

    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text('ราคา: ${FormattingExamples.formatCurrency(price, locale)}'),
        Text('วันที่: ${FormattingExamples.formatDate(now, locale)}'),
        Text('เวลา: ${FormattingExamples.formatDateTime(now, locale)}'),
        Text('ตัวเลข: ${FormattingExamples.formatNumber(price, locale)}'),
      ],
    );
  }
}
```

### การใช้งาน AppLocalizations

```dart
// Widget ที่ใช้ localized strings
class LocalizedWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final l10n = AppLocalizations.of(context)!;

    return Scaffold(
      appBar: AppBar(
        title: Text(l10n.appTitle),
      ),
      body: Column(
        children: [
          // Simple string
          Text(l10n.loginButton),

          // String with parameter
          Text(l10n.welcomeMessage('สมชาย')),

          // Plural
          Text(l10n.itemCount(5)),
          Text(l10n.itemCount(1)),
          Text(l10n.itemCount(0)),

          // Currency formatting
          Text(l10n.productPrice(299.99)),

          // Date formatting
          Text(l10n.dateFormat(DateTime.now())),

          // Gender
          Text(l10n.gender('male')),
          Text(l10n.gender('female')),
        ],
      ),
    );
  }
}
```

---

## 4. Plural and Gender Forms

### Plural Rules

```json
// ภาษาอังกฤษมี 2 รูปแบบ: singular และ plural
{
  "messageCount": "{count, plural, =0{No messages} =1{1 message} other{{count} messages}}"
}

// ภาษาไทยไม่มี plural form แต่ยังใช้ได้
{
  "messageCount": "{count, plural, =0{ไม่มีข้อความ} other{{count} ข้อความ}}"
}
```

```dart
// การใช้ plural ใน code
class PluralExample extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final l10n = AppLocalizations.of(context)!;

    return ListView(
      children: [
        Text(l10n.itemCount(0)),   // ไม่มีสินค้า
        Text(l10n.itemCount(1)),   // 1 ชิ้น
        Text(l10n.itemCount(5)),   // 5 ชิ้น
        Text(l10n.itemCount(100)), // 100 ชิ้น
      ],
    );
  }
}
```

### Gender Forms

```dart
// ARB file
// "greeting": "{gender, select, male{สวัสดีครับ} female{สวัสดีค่ะ} other{สวัสดี}}"

class GenderGreeting extends StatelessWidget {
  final String gender; // 'male', 'female', หรือ other

  const GenderGreeting({required this.gender});

  @override
  Widget build(BuildContext context) {
    final l10n = AppLocalizations.of(context)!;
    return Text(l10n.greeting(gender));
  }
}
```

---

## 5. RTL Support

### การรองรับ Right-to-Left (RTL)

```dart
// Flutter รองรับ RTL อัตโนมัติสำหรับภาษาอาหรับ, ฮีบรู, เปอร์เซีย

// app_ar.arb (Arabic)
// {
//   "@@locale": "ar",
//   "appTitle": "تطبيق التسوق الخاص بي",
//   "loginButton": "تسجيل الدخول"
// }

// เพิ่ม Arabic locale
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      supportedLocales: const [
        Locale('en'),
        Locale('th'),
        Locale('ar'), // Arabic (RTL)
      ],
      localizationsDelegates: const [
        AppLocalizations.delegate,
        GlobalMaterialLocalizations.delegate,
        GlobalWidgetsLocalizations.delegate,
        GlobalCupertinoLocalizations.delegate,
      ],
      home: HomeScreen(),
    );
  }
}

// Widget ที่รองรับ RTL
class RTLAwareWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final isRTL = Directionality.of(context) == TextDirection.rtl;

    return Padding(
      padding: EdgeInsetsDirectional.fromSTEB(16, 8, 8, 8),
      // DirectionalEdgeInsets ทำงานได้ทั้ง LTR และ RTL
      child: Row(
        children: [
          Icon(Icons.arrow_back_ios),
          Text('กลับ'),
        ],
      ),
    );
  }
}

// การกำหนด direction สำหรับบาง Widget
class DirectionalText extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Directionality(
      textDirection: TextDirection.ltr, // บังคับ LTR สำหรับ code/URL
      child: Text('https://example.com/product/123'),
    );
  }
}
```

### Adaptive Layout สำหรับ RTL

```dart
// Layout ที่ปรับตัวตาม text direction
class AdaptiveLayout extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        // leading icon ปรับเองอัตโนมัติ
        leading: BackButton(),
        title: Text(AppLocalizations.of(context)!.appTitle),
        actions: [
          // Actions อยู่ด้านขวา (LTR) หรือซ้าย (RTL) อัตโนมัติ
          IconButton(icon: Icon(Icons.search), onPressed: () {}),
          IconButton(icon: Icon(Icons.more_vert), onPressed: () {}),
        ],
      ),
      body: ListView(
        children: [
          // ListTile ปรับ leading/trailing อัตโนมัติ
          ListTile(
            leading: Icon(Icons.person),
            title: Text('ชื่อผู้ใช้'),
            trailing: Icon(Icons.chevron_right),
          ),

          // Padding แบบ directional
          Padding(
            padding: EdgeInsetsDirectional.only(start: 16, end: 8),
            child: Text('ข้อความ'),
          ),

          // Align ด้วย directional alignment
          Align(
            alignment: AlignmentDirectional.centerStart,
            child: Text('ชิดซ้าย (หรือขวาสำหรับ RTL)'),
          ),
        ],
      ),
    );
  }
}
```

---

## 6. Language Switcher

### การเปลี่ยนภาษาในแอป

```dart
// providers/locale_provider.dart
class LocaleProvider extends ChangeNotifier {
  Locale _locale = const Locale('th');

  Locale get locale => _locale;

  void setLocale(Locale newLocale) {
    if (_locale == newLocale) return;
    _locale = newLocale;
    notifyListeners();
  }

  String get currentLanguageName {
    switch (_locale.languageCode) {
      case 'th':
        return 'ภาษาไทย';
      case 'en':
        return 'English';
      case 'ja':
        return '日本語';
      default:
        return 'Unknown';
    }
  }
}

// Language Selector Widget
class LanguageSelectorWidget extends StatelessWidget {
  final List<Map<String, dynamic>> languages = [
    {'locale': Locale('th'), 'name': 'ภาษาไทย', 'flag': '🇹🇭'},
    {'locale': Locale('en'), 'name': 'English', 'flag': '🇬🇧'},
    {'locale': Locale('ja'), 'name': '日本語', 'flag': '🇯🇵'},
  ];

  @override
  Widget build(BuildContext context) {
    return Consumer<LocaleProvider>(
      builder: (context, provider, _) {
        return Column(
          children: [
            Text(
              'เลือกภาษา',
              style: Theme.of(context).textTheme.titleLarge,
            ),
            SizedBox(height: 16),
            ...languages.map((lang) {
              final locale = lang['locale'] as Locale;
              final isSelected = provider.locale == locale;

              return ListTile(
                leading: Text(lang['flag'] as String,
                    style: TextStyle(fontSize: 24)),
                title: Text(lang['name'] as String),
                trailing: isSelected
                    ? Icon(Icons.check, color: Colors.green)
                    : null,
                selected: isSelected,
                onTap: () => provider.setLocale(locale),
              );
            }).toList(),
          ],
        );
      },
    );
  }
}
```

---

## Workshop: Multi-language App (Thai/English)

### สร้างแอปที่รองรับทั้งภาษาไทยและอังกฤษ

```
project_structure/
├── lib/
│   ├── l10n/
│   │   ├── app_en.arb
│   │   └── app_th.arb
│   ├── providers/
│   │   └── locale_provider.dart
│   ├── screens/
│   │   ├── home_screen.dart
│   │   ├── settings_screen.dart
│   │   └── login_screen.dart
│   └── main.dart
└── l10n.yaml
```

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'package:flutter_gen/gen_l10n/app_localizations.dart';
import 'package:flutter_localizations/flutter_localizations.dart';

void main() {
  runApp(
    ChangeNotifierProvider(
      create: (_) => LocaleProvider(),
      child: MultiLanguageApp(),
    ),
  );
}

class MultiLanguageApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Consumer<LocaleProvider>(
      builder: (context, localeProvider, _) {
        return MaterialApp(
          locale: localeProvider.locale,
          localizationsDelegates: const [
            AppLocalizations.delegate,
            GlobalMaterialLocalizations.delegate,
            GlobalWidgetsLocalizations.delegate,
            GlobalCupertinoLocalizations.delegate,
          ],
          supportedLocales: const [
            Locale('th'),
            Locale('en'),
          ],
          theme: ThemeData(
            primarySwatch: Colors.blue,
            // Font ที่รองรับภาษาไทย
            fontFamily: 'Sarabun',
          ),
          home: HomeScreen(),
        );
      },
    );
  }
}
```

```json
// lib/l10n/app_th.arb (Thai ARB file สมบูรณ์)
{
  "@@locale": "th",
  
  "appTitle": "แอปตัวอย่าง",
  "@appTitle": {
    "description": "ชื่อแอป"
  },
  
  "homeTitle": "หน้าหลัก",
  "settingsTitle": "การตั้งค่า",
  "profileTitle": "โปรไฟล์",
  
  "welcomeUser": "ยินดีต้อนรับ คุณ{name}",
  "@welcomeUser": {
    "placeholders": {
      "name": {"type": "String"}
    }
  },
  
  "loginTitle": "เข้าสู่ระบบ",
  "emailHint": "กรอกอีเมลของคุณ",
  "passwordHint": "กรอกรหัสผ่าน",
  "loginButton": "เข้าสู่ระบบ",
  "registerLink": "ยังไม่มีบัญชี? สมัครสมาชิก",
  
  "errorEmptyEmail": "กรุณากรอกอีเมล",
  "errorInvalidEmail": "รูปแบบอีเมลไม่ถูกต้อง",
  "errorEmptyPassword": "กรุณากรอกรหัสผ่าน",
  "errorShortPassword": "รหัสผ่านต้องมีอย่างน้อย {count} ตัวอักษร",
  "@errorShortPassword": {
    "placeholders": {
      "count": {"type": "int"}
    }
  },
  
  "unreadMessages": "{count, plural, =0{ไม่มีข้อความใหม่} =1{1 ข้อความใหม่} other{{count} ข้อความใหม่}}",
  "@unreadMessages": {
    "placeholders": {
      "count": {"type": "int"}
    }
  },
  
  "languageLabel": "ภาษา",
  "thaiLanguage": "ภาษาไทย",
  "englishLanguage": "English",
  
  "notificationsLabel": "การแจ้งเตือน",
  "enableNotifications": "เปิดการแจ้งเตือน",
  
  "lastSeen": "เข้าใช้งานล่าสุด {date}",
  "@lastSeen": {
    "placeholders": {
      "date": {
        "type": "DateTime",
        "format": "yMMMd"
      }
    }
  }
}
```

```json
// lib/l10n/app_en.arb (English ARB file)
{
  "@@locale": "en",
  
  "appTitle": "Sample App",
  "homeTitle": "Home",
  "settingsTitle": "Settings",
  "profileTitle": "Profile",
  
  "welcomeUser": "Welcome, {name}",
  "@welcomeUser": {
    "placeholders": {
      "name": {"type": "String"}
    }
  },
  
  "loginTitle": "Sign In",
  "emailHint": "Enter your email",
  "passwordHint": "Enter your password",
  "loginButton": "Sign In",
  "registerLink": "Don't have an account? Register",
  
  "errorEmptyEmail": "Please enter your email",
  "errorInvalidEmail": "Invalid email format",
  "errorEmptyPassword": "Please enter your password",
  "errorShortPassword": "Password must be at least {count} characters",
  "@errorShortPassword": {
    "placeholders": {
      "count": {"type": "int"}
    }
  },
  
  "unreadMessages": "{count, plural, =0{No new messages} =1{1 new message} other{{count} new messages}}",
  "@unreadMessages": {
    "placeholders": {
      "count": {"type": "int"}
    }
  },
  
  "languageLabel": "Language",
  "thaiLanguage": "ภาษาไทย",
  "englishLanguage": "English",
  
  "notificationsLabel": "Notifications",
  "enableNotifications": "Enable Notifications",
  
  "lastSeen": "Last seen {date}",
  "@lastSeen": {
    "placeholders": {
      "date": {
        "type": "DateTime",
        "format": "yMMMd"
      }
    }
  }
}
```

```dart
// lib/screens/login_screen.dart
class LoginScreen extends StatefulWidget {
  @override
  _LoginScreenState createState() => _LoginScreenState();
}

class _LoginScreenState extends State<LoginScreen> {
  final _formKey = GlobalKey<FormState>();
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();
  bool _isPasswordVisible = false;

  @override
  void dispose() {
    _emailController.dispose();
    _passwordController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final l10n = AppLocalizations.of(context)!;

    return Scaffold(
      appBar: AppBar(
        title: Text(l10n.loginTitle),
        actions: [
          // Language Switcher
          PopupMenuButton<Locale>(
            icon: Icon(Icons.language),
            onSelected: (locale) {
              context.read<LocaleProvider>().setLocale(locale);
            },
            itemBuilder: (context) => [
              PopupMenuItem(
                value: Locale('th'),
                child: Text('🇹🇭 ${l10n.thaiLanguage}'),
              ),
              PopupMenuItem(
                value: Locale('en'),
                child: Text('🇬🇧 ${l10n.englishLanguage}'),
              ),
            ],
          ),
        ],
      ),
      body: Padding(
        padding: EdgeInsets.all(24),
        child: Form(
          key: _formKey,
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.stretch,
            children: [
              SizedBox(height: 48),
              Text(
                l10n.loginTitle,
                style: Theme.of(context).textTheme.headlineMedium,
                textAlign: TextAlign.center,
              ),
              SizedBox(height: 32),

              // Email field
              TextFormField(
                controller: _emailController,
                keyboardType: TextInputType.emailAddress,
                decoration: InputDecoration(
                  labelText: l10n.emailHint,
                  prefixIcon: Icon(Icons.email),
                  border: OutlineInputBorder(),
                ),
                validator: (value) {
                  if (value == null || value.isEmpty) {
                    return l10n.errorEmptyEmail;
                  }
                  if (!value.contains('@')) {
                    return l10n.errorInvalidEmail;
                  }
                  return null;
                },
              ),
              SizedBox(height: 16),

              // Password field
              TextFormField(
                controller: _passwordController,
                obscureText: !_isPasswordVisible,
                decoration: InputDecoration(
                  labelText: l10n.passwordHint,
                  prefixIcon: Icon(Icons.lock),
                  border: OutlineInputBorder(),
                  suffixIcon: IconButton(
                    icon: Icon(
                      _isPasswordVisible
                          ? Icons.visibility_off
                          : Icons.visibility,
                    ),
                    onPressed: () {
                      setState(
                          () => _isPasswordVisible = !_isPasswordVisible);
                    },
                  ),
                ),
                validator: (value) {
                  if (value == null || value.isEmpty) {
                    return l10n.errorEmptyPassword;
                  }
                  if (value.length < 8) {
                    return l10n.errorShortPassword(8);
                  }
                  return null;
                },
              ),
              SizedBox(height: 24),

              // Login button
              ElevatedButton(
                onPressed: _handleLogin,
                style: ElevatedButton.styleFrom(
                  padding: EdgeInsets.symmetric(vertical: 16),
                ),
                child: Text(l10n.loginButton, style: TextStyle(fontSize: 16)),
              ),
              SizedBox(height: 16),

              // Register link
              TextButton(
                onPressed: () {},
                child: Text(l10n.registerLink),
              ),
            ],
          ),
        ),
      ),
    );
  }

  void _handleLogin() {
    if (_formKey.currentState!.validate()) {
      // Handle login
    }
  }
}
```

```dart
// lib/screens/settings_screen.dart
class SettingsScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final l10n = AppLocalizations.of(context)!;
    final localeProvider = context.watch<LocaleProvider>();

    return Scaffold(
      appBar: AppBar(title: Text(l10n.settingsTitle)),
      body: ListView(
        children: [
          // Language section
          ListTile(
            leading: Icon(Icons.language),
            title: Text(l10n.languageLabel),
            subtitle: Text(localeProvider.currentLanguageName),
            trailing: Icon(Icons.chevron_right),
            onTap: () {
              showModalBottomSheet(
                context: context,
                builder: (_) => LanguageSelectorSheet(),
              );
            },
          ),

          // Notifications section
          SwitchListTile(
            secondary: Icon(Icons.notifications),
            title: Text(l10n.notificationsLabel),
            subtitle: Text(l10n.enableNotifications),
            value: true,
            onChanged: (value) {},
          ),
        ],
      ),
    );
  }
}

// Bottom sheet สำหรับเลือกภาษา
class LanguageSelectorSheet extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final l10n = AppLocalizations.of(context)!;
    final localeProvider = context.watch<LocaleProvider>();

    return Container(
      padding: EdgeInsets.all(16),
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          Text(l10n.languageLabel,
              style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
          SizedBox(height: 16),
          _LanguageOption(
            flag: '🇹🇭',
            name: l10n.thaiLanguage,
            locale: Locale('th'),
            isSelected: localeProvider.locale.languageCode == 'th',
          ),
          _LanguageOption(
            flag: '🇬🇧',
            name: l10n.englishLanguage,
            locale: Locale('en'),
            isSelected: localeProvider.locale.languageCode == 'en',
          ),
          SizedBox(height: 16),
        ],
      ),
    );
  }
}

class _LanguageOption extends StatelessWidget {
  final String flag;
  final String name;
  final Locale locale;
  final bool isSelected;

  const _LanguageOption({
    required this.flag,
    required this.name,
    required this.locale,
    required this.isSelected,
  });

  @override
  Widget build(BuildContext context) {
    return ListTile(
      leading: Text(flag, style: TextStyle(fontSize: 28)),
      title: Text(name),
      trailing: isSelected
          ? Icon(Icons.check_circle, color: Colors.green)
          : Icon(Icons.circle_outlined),
      onTap: () {
        context.read<LocaleProvider>().setLocale(locale);
        Navigator.pop(context);
      },
    );
  }
}
```

---

## สรุป

การทำ i18n ใน Flutter ประกอบด้วย:

1. **ARB Files** - เก็บข้อความในแต่ละภาษา
2. **Code Generation** - สร้าง type-safe localization classes
3. **intl Package** - จัดรูปแบบตัวเลข วันที่ สกุลเงิน
4. **Plural/Gender** - รองรับรูปแบบที่ต่างกันตามจำนวนและเพศ
5. **RTL Support** - รองรับภาษาที่อ่านจากขวาไปซ้าย
6. **LocaleProvider** - จัดการการเปลี่ยนภาษาใน runtime
