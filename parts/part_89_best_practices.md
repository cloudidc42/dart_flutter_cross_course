# Part 89: Best Practices Summary - สรุป Best Practices

## บทนำ

บทนี้รวบรวม best practices ทั้งหมดสำหรับ Flutter development ตั้งแต่การจัดโค้ด ไปจนถึงการ release แอป เหมาะสำหรับใช้เป็น checklist

## 1. Code Organization Best Practices

### Folder Structure

```
lib/
├── core/
│   ├── constants/
│   │   ├── app_colors.dart
│   │   ├── app_strings.dart
│   │   └── app_dimensions.dart
│   ├── errors/
│   │   ├── exceptions.dart
│   │   └── failures.dart
│   ├── extensions/
│   │   ├── context_extension.dart
│   │   ├── string_extension.dart
│   │   └── date_extension.dart
│   ├── utils/
│   │   ├── validators.dart
│   │   └── formatters.dart
│   └── widgets/
│       ├── app_button.dart
│       ├── app_text_field.dart
│       └── loading_overlay.dart
├── features/
│   └── feature_name/
│       ├── data/
│       │   ├── datasources/
│       │   ├── models/
│       │   └── repositories/
│       ├── domain/
│       │   ├── entities/
│       │   ├── repositories/
│       │   └── usecases/
│       └── presentation/
│           ├── blocs/
│           ├── screens/
│           └── widgets/
└── main.dart
```

### Naming Conventions

```dart
// Classes: PascalCase
class UserProfileScreen extends StatelessWidget {}
class AuthBloc extends Bloc<AuthEvent, AuthState> {}

// Variables & functions: camelCase
final userName = 'John';
void fetchUserData() {}

// Constants: camelCase (with const)
const double defaultPadding = 16.0;
const String baseUrl = 'https://api.example.com';

// Private: underscore prefix
class _MyWidgetState extends State<MyWidget> {
  final _controller = TextEditingController();
  void _handleSubmit() {}
}

// Files: snake_case
// user_profile_screen.dart
// auth_bloc.dart
// product_repository.dart
```

### Widget Best Practices

```dart
// ✅ ดี: ใช้ const เสมอเมื่อทำได้
class MyWidget extends StatelessWidget {
  const MyWidget({super.key}); // ✅ const constructor
  
  @override
  Widget build(BuildContext context) {
    return const Text('Hello'); // ✅ const widget
  }
}

// ✅ ดี: แยก widget เล็ก ๆ
class ProductCard extends StatelessWidget {
  final Product product;
  const ProductCard({super.key, required this.product});
  
  @override
  Widget build(BuildContext context) {
    return Card(
      child: Column(
        children: [
          _ProductImage(url: product.imageUrl),     // ✅ แยก widget
          _ProductInfo(product: product),             // ✅ แยก widget
          _ProductActions(product: product),          // ✅ แยก widget
        ],
      ),
    );
  }
}

// ❌ ไม่ดี: widget ใหญ่ build() ที่ยาวมาก
class ProductCard extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Card(
      child: Column(
        children: [
          // 300 บรรทัดของ nested widgets...
        ],
      ),
    );
  }
}

// ✅ ดี: ใช้ named parameters แทน positional
// bad
AppButton(true, Colors.blue, 'Submit', () {});

// good
AppButton(
  isPrimary: true,
  color: Colors.blue,
  label: 'Submit',
  onPressed: () {},
);
```

## 2. Performance Checklist

### Widget Performance

```dart
// ✅ 1. ใช้ const constructor
const SizedBox(height: 16)
const Icon(Icons.home)

// ✅ 2. ใช้ RepaintBoundary สำหรับ widget ที่ animate บ่อย
RepaintBoundary(
  child: AnimatedWidget(...),
)

// ✅ 3. ใช้ ListView.builder แทน ListView
// ❌ ไม่ดี - สร้างทุก item พร้อมกัน
ListView(children: items.map(ItemWidget.new).toList())

// ✅ ดี - lazy loading
ListView.builder(
  itemCount: items.length,
  itemBuilder: (context, index) => ItemWidget(item: items[index]),
)

// ✅ 4. Cache Network Images
CachedNetworkImage(
  imageUrl: url,
  placeholder: (context, url) => const Shimmer(),
  errorWidget: (context, url, error) => const Icon(Icons.error),
)

// ✅ 5. ใช้ AutomaticKeepAliveClientMixin สำหรับ PageView
class MyPage extends StatefulWidget {
  @override
  State<MyPage> createState() => _MyPageState();
}

class _MyPageState extends State<MyPage>
    with AutomaticKeepAliveClientMixin {
  @override
  bool get wantKeepAlive => true;  // ✅ ไม่ rebuild เมื่อ swipe กลับมา
  
  @override
  Widget build(BuildContext context) {
    super.build(context);  // ✅ ต้องเรียก
    return const Text('Content');
  }
}

// ✅ 6. Avoid unnecessary rebuilds ใน BLoC
// ❌ ไม่ดี - rebuild ทุกครั้งที่ state เปลี่ยน
BlocBuilder<UserBloc, UserState>(
  builder: (context, state) => Text(state.user.name),
)

// ✅ ดี - rebuild เฉพาะเมื่อ name เปลี่ยน
BlocBuilder<UserBloc, UserState>(
  buildWhen: (previous, current) =>
      previous.user.name != current.user.name,
  builder: (context, state) => Text(state.user.name),
)
```

### Async/Isolate Performance

```dart
// ✅ ใช้ Isolate สำหรับงานหนัก
Future<List<ProcessedData>> processLargeDataset(
  List<RawData> rawData,
) async {
  return await compute(_processInIsolate, rawData);
}

List<ProcessedData> _processInIsolate(List<RawData> rawData) {
  return rawData.map((d) => _expensiveProcess(d)).toList();
}

// ✅ ใช้ Stream แทน polling
// ❌ ไม่ดี - polling
Timer.periodic(const Duration(seconds: 1), (timer) {
  checkForUpdates();
});

// ✅ ดี - stream
firestore.collection('updates').snapshots().listen((snapshot) {
  handleUpdates(snapshot);
});
```

## 3. Security Checklist

```dart
// ✅ 1. ไม่เก็บ sensitive data ใน SharedPreferences
// ❌ ไม่ดี
prefs.setString('token', 'jwt_token_here');

// ✅ ดี
const storage = FlutterSecureStorage();
await storage.write(key: 'token', value: 'jwt_token_here');

// ✅ 2. Certificate Pinning
class SecureHttpClient {
  static Dio createClient() {
    final dio = Dio();
    (dio.httpClientAdapter as IOHttpClientAdapter).createHttpClient = () {
      final client = HttpClient();
      client.badCertificateCallback = (cert, host, port) {
        // Verify fingerprint
        final fingerprint = sha256.convert(cert.der).toString();
        return trustedFingerprints.contains(fingerprint);
      };
      return client;
    };
    return dio;
  }
}

// ✅ 3. Root/Jailbreak Detection
class SecurityChecker {
  static Future<bool> isCompromised() async {
    if (Platform.isAndroid) {
      return await _checkRooted();
    } else if (Platform.isIOS) {
      return await _checkJailbroken();
    }
    return false;
  }

  static Future<bool> _checkRooted() async {
    // Check for su binary, test-keys, etc.
    const suspiciousPaths = [
      '/system/app/Superuser.apk',
      '/sbin/su',
      '/system/bin/su',
    ];
    for (final path in suspiciousPaths) {
      if (await File(path).exists()) return true;
    }
    return false;
  }
}

// ✅ 4. Obfuscate code ตอน release
// flutter build apk --obfuscate --split-debug-info=./debug-info

// ✅ 5. ตรวจสอบ input validation ทุกที่
class Validators {
  static String? validatePhone(String? value) {
    if (value == null || value.isEmpty) return 'Required';
    final clean = value.replaceAll(RegExp(r'[\s\-()]'), '');
    if (!RegExp(r'^(0[6-9]\d{8})$').hasMatch(clean)) {
      return 'Invalid phone number';
    }
    return null;
  }

  static String? validateAmount(String? value) {
    if (value == null || value.isEmpty) return 'Required';
    final amount = double.tryParse(value);
    if (amount == null) return 'Invalid amount';
    if (amount <= 0) return 'Amount must be positive';
    if (amount > 1000000) return 'Amount exceeds limit';
    return null;
  }
}
```

## 4. Testing Checklist

```dart
// Testing pyramid สำหรับ Flutter app:
// - 70-80% Unit tests (fast, cheap)
// - 15-20% Widget tests (medium)
// - 5-10% Integration/E2E tests (slow, expensive)

// ✅ Unit Test checklist:
// □ Test each Use Case
// □ Test each Repository
// □ Test each BLoC/Cubit
// □ Mock all external dependencies
// □ Test error scenarios
// □ Test edge cases (empty, null, max values)

// ✅ Widget Test checklist:
// □ Test rendering
// □ Test user interactions
// □ Test loading/error/empty states
// □ Test accessibility (semantics)
// □ Golden tests for complex UI

// ✅ Integration Test checklist:
// □ Test complete user flows
// □ Test real device behaviors
// □ Test platform permissions

// ตัวอย่าง comprehensive BLoC test
void main() {
  group('AuthBloc', () {
    late AuthBloc authBloc;
    late MockSignInUseCase mockSignIn;
    late MockSignOutUseCase mockSignOut;

    setUp(() {
      mockSignIn = MockSignInUseCase();
      mockSignOut = MockSignOutUseCase();
      authBloc = AuthBloc(
        signIn: mockSignIn,
        signOut: mockSignOut,
      );
    });

    tearDown(() => authBloc.close());

    // ✅ Test happy path
    blocTest<AuthBloc, AuthState>(
      'emits [Loading, Authenticated] when sign in succeeds',
      build: () {
        when(() => mockSignIn(any()))
            .thenAnswer((_) async => const Right(user));
        return authBloc;
      },
      act: (bloc) => bloc.add(
        const SignInEvent(email: 'test@test.com', password: 'password'),
      ),
      expect: () => [
        const AuthState.loading(),
        AuthState.authenticated(user: user),
      ],
    );

    // ✅ Test error path
    blocTest<AuthBloc, AuthState>(
      'emits [Loading, Error] when sign in fails',
      build: () {
        when(() => mockSignIn(any()))
            .thenAnswer((_) async => const Left(AuthFailure()));
        return authBloc;
      },
      act: (bloc) => bloc.add(
        const SignInEvent(email: 'test@test.com', password: 'wrong'),
      ),
      expect: () => [
        const AuthState.loading(),
        const AuthState.error(message: 'Authentication failed'),
      ],
    );
  });
}
```

## 5. Release Checklist

### Pre-Release Checklist

```
ANDROID
□ เพิ่ม version code และ version name ใน pubspec.yaml
□ สร้าง signed APK/AAB ด้วย keystore
□ ตรวจสอบ proguard rules
□ Test บน real device
□ ตรวจสอบ permissions ใน AndroidManifest.xml
□ เปิด R8/ProGuard สำหรับ release
□ ตรวจสอบ target SDK version

IOS
□ Bundle identifier ถูกต้อง
□ Provisioning profile และ certificate valid
□ ตรวจสอบ Info.plist (permissions, URLs)
□ Test บน physical device
□ Archive และ validate ด้วย Xcode
□ ตรวจสอบ App Store Connect metadata

FLUTTER
□ flutter pub outdated - อัพเดท dependencies
□ flutter analyze - 0 issues
□ flutter test - ผ่านทั้งหมด
□ flutter build apk --release - สำเร็จ
□ ตรวจสอบ app size (target < 15MB)

CONTENT
□ ตรวจสอบ app icon (ทุก resolution)
□ ตรวจสอบ splash screen
□ ตรวจสอบ store screenshots (ทุก device size)
□ เขียน release notes
□ ตรวจสอบ privacy policy URL

BACKEND
□ API endpoints ชี้ production
□ Firebase project ถูกต้อง (production)
□ Environment variables ถูกต้อง
□ Feature flags ตั้งค่า production

SECURITY
□ Disable debug logging
□ Enable certificate pinning
□ Enable obfuscation
□ Remove test/debug credentials
□ Verify API key restrictions
```

## 6. Complete Developer Checklist

### Daily Checklist

```
□ git pull origin main
□ flutter pub get (ถ้า pubspec เปลี่ยน)
□ flutter analyze
□ Run tests ก่อน commit
□ Write meaningful commit messages
□ Code review อย่างน้อย 1 PR ของเพื่อน
```

### Weekly Checklist

```
□ อัพเดท packages (ถ้า patch versions)
□ Review crash reports (Crashlytics/Sentry)
□ Check performance metrics (Firebase Performance)
□ อ่าน Flutter changelog
□ Document decisions ที่ทำในสัปดาห์นี้
```

### Monthly Checklist

```
□ Major package upgrades
□ Security vulnerability scan
□ Review test coverage report
□ Performance profiling session
□ Team retrospective
□ Update roadmap
```

## 7. Code Review Checklist

```dart
// เมื่อ review PR ตรวจสอบสิ่งเหล่านี้:

// ✅ Correctness
// □ Logic ถูกต้องหรือไม่?
// □ Edge cases ถูก handle หรือไม่?
// □ Error handling ครบหรือไม่?

// ✅ Performance
// □ มี unnecessary rebuilds หรือไม่?
// □ List ใช้ builder หรือไม่?
// □ Images ถูก cache หรือไม่?

// ✅ Code Quality
// □ ชื่อ variable/function สื่อความหมายหรือไม่?
// □ ความซับซ้อนเหมาะสมหรือไม่?
// □ DRY - มี code ซ้ำหรือไม่?

// ✅ Security
// □ User input ถูก validate หรือไม่?
// □ Sensitive data ถูก handle อย่างปลอดภัยหรือไม่?

// ✅ Testing
// □ มี test coverage สำหรับ new code หรือไม่?
// □ Tests มีความหมายหรือไม่?
```

## สรุป

Best practices ที่สำคัญที่สุด:
1. **Code Organization**: Feature-first, Clean Architecture
2. **Performance**: const, builder, cache, Isolates
3. **Security**: Secure storage, pin, obfuscation
4. **Testing**: 70/20/10 pyramid
5. **Release**: Checklist ก่อน deploy ทุกครั้ง

## แบบทดสอบ

1. Audit โปรเจกต์ปัจจุบันโดยใช้ checklist นี้
2. ตั้ง CI pipeline ที่ enforce best practices อัตโนมัติ
3. สร้าง Linting rules ที่ตรงกับ team standards
4. ทำ performance profiling session และ document findings
