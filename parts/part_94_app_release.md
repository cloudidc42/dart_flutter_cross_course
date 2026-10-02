# Part 94: App Store Optimization and Release Management

## 🎯 เป้าหมายของ Part นี้
- เตรียมแอปสำหรับ production
- App signing (Android/iOS)
- Google Play Store submission
- Apple App Store submission
- Version management
- Release checklist

---

## 1. Version Management

```yaml
# pubspec.yaml
version: 1.2.3+45
# format: major.minor.patch+buildNumber

# major: Breaking changes
# minor: New features (backward compatible)
# patch: Bug fixes
# buildNumber: Auto-incremented for store submissions
```

```bash
# ดูรุ่นปัจจุบัน:
cat pubspec.yaml | grep version

# ใน CI/CD - ตั้งค่า version automatically:
# flutter build apk --build-number=$CI_BUILD_NUMBER
```

### Semantic Versioning:
```dart
// version_manager.dart
class AppVersion {
  static const String version = '1.2.3';
  static const int buildNumber = 45;
  
  static String get fullVersion => '$version+$buildNumber';
  
  static bool get isProduction => version.endsWith('-beta') == false;
  
  // ตรวจสอบว่าต้องอัปเดตไหม:
  static bool requiresUpdate(String serverMinVersion) {
    final parts = version.split('.');
    final serverParts = serverMinVersion.split('.');
    
    for (int i = 0; i < 3; i++) {
      final current = int.parse(parts[i]);
      final server = int.parse(serverParts[i]);
      
      if (current < server) return true;
      if (current > server) return false;
    }
    return false;
  }
}
```

---

## 2. Android Release Build

### 2.1 สร้าง Keystore
```bash
# สร้าง keystore file (ทำครั้งเดียว!):
keytool -genkey -v -keystore ~/my-release-key.jks \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000 \
  -alias my-key-alias

# เก็บ keystore ไว้อย่างปลอดภัย! อย่า commit ขึ้น git!
```

### 2.2 ตั้งค่า key.properties
```properties
# android/key.properties (อย่า commit ไฟล์นี้!)
storePassword=your-store-password
keyPassword=your-key-password
keyAlias=my-key-alias
storeFile=/path/to/my-release-key.jks
```

### 2.3 build.gradle
```gradle
// android/app/build.gradle
def keystoreProperties = new Properties()
def keystorePropertiesFile = rootProject.file('key.properties')
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
}

android {
    // ...
    
    signingConfigs {
        release {
            keyAlias keystoreProperties['keyAlias']
            keyPassword keystoreProperties['keyPassword']
            storeFile keystoreProperties['storeFile'] ? file(keystoreProperties['storeFile']) : null
            storePassword keystoreProperties['storePassword']
        }
    }
    
    buildTypes {
        release {
            signingConfig signingConfigs.release
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}
```

### 2.4 สร้าง Release Build
```bash
# APK:
flutter build apk --release

# App Bundle (แนะนำ สำหรับ Play Store):
flutter build appbundle --release

# Output location:
# build/app/outputs/flutter-apk/app-release.apk
# build/app/outputs/bundle/release/app-release.aab

# Build พร้อม version:
flutter build appbundle --build-number=45 --build-name=1.2.3
```

---

## 3. iOS Release Build

### 3.1 ตั้งค่าใน Xcode
```
1. เปิด ios/Runner.xcworkspace ใน Xcode
2. เลือก Runner target
3. ตั้งค่า:
   - Bundle Identifier: com.yourcompany.yourapp
   - Version: 1.2.3
   - Build: 45
4. Signing & Capabilities:
   - Team: เลือก Apple Developer account
   - Automatically manage signing: ✅
```

### 3.2 สร้าง Release Build
```bash
# Build iosสำหรับ App Store:
flutter build ipa

# Output:
# build/ios/ipa/Runner.ipa

# หรือ archive ผ่าน Xcode:
# Product > Archive
# แล้ว Distribute App > App Store Connect
```

---

## 4. App Store Submission Checklist

### Google Play Store:
```
Pre-submission:
✅ App signing setup
✅ Target SDK 34+ (Google requirement)
✅ 64-bit support
✅ Privacy policy URL
✅ Content rating questionnaire
✅ Screenshot (phone, tablet)
✅ Feature graphic (1024x500)
✅ App icon (512x512)
✅ Short description (80 chars)
✅ Full description (4000 chars)
✅ Tags/categories
✅ Contact email

Technical:
✅ No crashes
✅ Handles permissions gracefully
✅ Works offline or shows proper error
✅ Tested on multiple Android versions
✅ ProGuard rules set
```

### Apple App Store:
```
Pre-submission:
✅ App Store Connect account
✅ Certificates and profiles
✅ Privacy policy
✅ App icon (1024x1024)
✅ Screenshots (6.5", 5.5", iPad Pro)
✅ App preview video (optional)
✅ Description (4000 chars)
✅ Keywords (100 chars)
✅ Support URL
✅ Marketing URL

Technical:
✅ NSAllowsArbitraryLoads removed or explained
✅ Privacy usage descriptions (NSCameraUsageDescription, etc.)
✅ App Transport Security configured
✅ Tested on device (not just simulator)
✅ No use of private APIs
```

---

## 5. CI/CD ด้วย GitHub Actions

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'  # trigger บน version tags เช่น v1.2.3

jobs:
  build-android:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.x'
          
      - name: Setup Java
        uses: actions/setup-java@v3
        with:
          distribution: 'zulu'
          java-version: '17'
          
      - name: Create key.properties
        run: |
          echo "storePassword=${{ secrets.STORE_PASSWORD }}" >> android/key.properties
          echo "keyPassword=${{ secrets.KEY_PASSWORD }}" >> android/key.properties
          echo "keyAlias=${{ secrets.KEY_ALIAS }}" >> android/key.properties
          echo "storeFile=../keystore.jks" >> android/key.properties
          
      - name: Decode Keystore
        run: |
          echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 --decode > android/keystore.jks
          
      - name: Get version from tag
        id: version
        run: |
          VERSION=${GITHUB_REF#refs/tags/v}
          BUILD=${GITHUB_RUN_NUMBER}
          echo "VERSION=$VERSION" >> $GITHUB_OUTPUT
          echo "BUILD=$BUILD" >> $GITHUB_OUTPUT
          
      - name: Build App Bundle
        run: |
          flutter pub get
          flutter build appbundle \
            --build-name=${{ steps.version.outputs.VERSION }} \
            --build-number=${{ steps.version.outputs.BUILD }}
            
      - name: Upload to Play Store
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.GOOGLE_PLAY_JSON }}
          packageName: com.yourcompany.yourapp
          releaseFiles: build/app/outputs/bundle/release/*.aab
          track: internal  # หรือ alpha, beta, production
          
  build-ios:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.x'
          
      - name: Install Certificates
        uses: apple-actions/import-codesign-certs@v2
        with:
          p12-file-base64: ${{ secrets.CERTIFICATES_P12 }}
          p12-password: ${{ secrets.CERTIFICATES_P12_PASSWORD }}
          
      - name: Build IPA
        run: flutter build ipa --export-options-plist=ios/ExportOptions.plist
        
      - name: Upload to TestFlight
        uses: apple-actions/upload-testflight-build@v1
        with:
          app-path: build/ios/ipa/Runner.ipa
          issuer-id: ${{ secrets.APPSTORE_ISSUER_ID }}
          api-key-id: ${{ secrets.APPSTORE_API_KEY_ID }}
          api-private-key: ${{ secrets.APPSTORE_API_PRIVATE_KEY }}
```

---

## 6. สรุป Part 94

สิ่งที่เรียนรู้:
- ✅ Version management (SemVer)
- ✅ Android signing และ release build
- ✅ iOS signing และ build
- ✅ Play Store submission checklist
- ✅ App Store submission checklist
- ✅ CI/CD automation ด้วย GitHub Actions

---

## ➡️ Part ถัดไป
**Part 95: Scalability Patterns**
