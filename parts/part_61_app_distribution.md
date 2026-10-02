# Part 61: App Distribution

## ภาพรวมการกระจาย App

หลังจาก build แอปแล้ว ขั้นตอนต่อไปคือการกระจายให้ผู้ใช้ มีหลายช่องทาง:

1. **Google Play Store** - Android
2. **Apple App Store** - iOS
3. **TestFlight** - Beta testing สำหรับ iOS
4. **Firebase App Distribution** - Internal/External testing

---

## 1. Google Play Store Publishing

### เตรียม App สำหรับ Play Store

```yaml
# pubspec.yaml - version format: version: 1.2.3+45
# 1.2.3 = version name (แสดงให้ผู้ใช้เห็น)
# 45 = version code (ต้องเพิ่มขึ้นทุก release)
version: 1.0.0+1
```

```groovy
// android/app/build.gradle
android {
    defaultConfig {
        applicationId "com.example.myapp"
        minSdkVersion 21
        targetSdkVersion 34
        versionCode flutterVersionCode.toInteger()
        versionName flutterVersionName
    }
    
    buildTypes {
        release {
            // ProGuard สำหรับ shrinking และ obfuscation
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android.txt'),
                         'proguard-rules.pro'
            
            signingConfig signingConfigs.release
        }
    }
}
```

### สร้าง Release Build

```bash
# Build AAB (แนะนำสำหรับ Play Store)
flutter build appbundle --release

# Build APK (สำหรับ direct distribution)
flutter build apk --release

# Build APK แยกตาม ABI (ขนาดเล็กกว่า)
flutter build apk --split-per-abi --release
```

### Play Store Listing

```
สิ่งที่ต้องเตรียมสำหรับ Play Store:
1. App Title (สูงสุด 50 ตัวอักษร)
2. Short Description (สูงสุด 80 ตัวอักษร)
3. Full Description (สูงสุด 4,000 ตัวอักษร)
4. App Icon (512x512 px, PNG)
5. Feature Graphic (1024x500 px)
6. Screenshots:
   - Phone: อย่างน้อย 2 รูป (สูงสุด 8)
   - Tablet 7": อย่างน้อย 1 รูป (ถ้า support)
   - Tablet 10": อย่างน้อย 1 รูป (ถ้า support)
7. Privacy Policy URL
8. Content Rating
9. Target Audience
```

### Play Console API สำหรับ Automation

```dart
// lib/services/play_store_service.dart
// ใช้ googleapis package สำหรับ upload อัตโนมัติ

// pubspec.yaml
// dependencies:
//   googleapis: ^12.0.0
//   googleapis_auth: ^1.4.0

import 'package:googleapis/androidpublisher/v3.dart';
import 'package:googleapis_auth/auth_io.dart';

class PlayStoreService {
  static const _packageName = 'com.example.myapp';
  
  Future<void> uploadAAB({
    required String aabPath,
    required String serviceAccountKeyPath,
    String track = 'internal',
  }) async {
    // Load service account credentials
    final jsonCredentials = await File(serviceAccountKeyPath).readAsString();
    final credentials = ServiceAccountCredentials.fromJson(jsonCredentials);
    
    // Authenticate
    final client = await clientViaServiceAccount(
      credentials,
      [AndroidPublisherApi.androidpublisherScope],
    );
    
    final api = AndroidPublisherApi(client);
    
    // Create edit
    final edit = await api.edits.insert(
      AppEdit(),
      _packageName,
    );
    final editId = edit.id!;
    
    // Upload AAB
    final aabBytes = await File(aabPath).readAsBytes();
    await api.edits.bundles.upload(
      _packageName,
      editId,
      uploadMedia: Media(
        Stream.fromIterable([aabBytes]),
        aabBytes.length,
      ),
    );
    
    // Assign to track
    await api.edits.tracks.update(
      Track(
        track: track,
        releases: [
          TrackRelease(
            status: 'completed',
            name: 'Release ${DateTime.now()}',
          ),
        ],
      ),
      _packageName,
      editId,
      track,
    );
    
    // Commit edit
    await api.edits.commit(_packageName, editId);
    
    client.close();
    print('Successfully uploaded to Play Store track: $track');
  }
}
```

---

## 2. Apple App Store Publishing

### เตรียม App สำหรับ App Store

```bash
# Build iOS release
flutter build ios --release

# หรือ build IPA ด้วย xcodebuild
xcodebuild -workspace ios/Runner.xcworkspace \
  -scheme Runner \
  -configuration Release \
  -archivePath build/Runner.xcarchive \
  archive

xcodebuild -exportArchive \
  -archivePath build/Runner.xcarchive \
  -exportOptionsPlist ios/ExportOptions.plist \
  -exportPath build/ios-release
```

### ExportOptions.plist

```xml
<!-- ios/ExportOptions.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" 
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>method</key>
  <string>app-store</string>
  <key>teamID</key>
  <string>YOUR_TEAM_ID</string>
  <key>uploadBitcode</key>
  <false/>
  <key>uploadSymbols</key>
  <true/>
  <key>signingStyle</key>
  <string>automatic</string>
</dict>
</plist>
```

### App Store Connect API

```dart
// ใช้ App Store Connect API สำหรับ automation
// ต้องใช้ API Key จาก App Store Connect

class AppStoreConnectService {
  final String keyId;
  final String issuerId;
  final String privateKeyPath;
  
  AppStoreConnectService({
    required this.keyId,
    required this.issuerId,
    required this.privateKeyPath,
  });
  
  Future<String> _generateJWT() async {
    // สร้าง JWT token สำหรับ authentication
    final privateKey = await File(privateKeyPath).readAsString();
    // Use dart_jsonwebtoken package
    // final jwt = JWT({'iss': issuerId, 'iat': ..., 'exp': ...});
    // return jwt.sign(ECPrivateKey(privateKey), algorithm: JWTAlgorithm.ES256);
    return 'generated_jwt_token';
  }
  
  Future<List<Map<String, dynamic>>> getApps() async {
    final jwt = await _generateJWT();
    final response = await http.get(
      Uri.parse('https://api.appstoreconnect.apple.com/v1/apps'),
      headers: {'Authorization': 'Bearer $jwt'},
    );
    
    if (response.statusCode == 200) {
      final data = jsonDecode(response.body);
      return List<Map<String, dynamic>>.from(data['data']);
    }
    throw Exception('Failed to get apps');
  }
}
```

---

## 3. TestFlight

### TestFlight Distribution

```bash
# Upload ไปยัง TestFlight ด้วย xcrun altool
xcrun altool --upload-app \
  -f build/ios-release/MyApp.ipa \
  -t ios \
  --apiKey YOUR_API_KEY_ID \
  --apiIssuer YOUR_ISSUER_ID

# หรือใช้ Transporter app บน Mac
```

### Fastlane TestFlight

```ruby
# ios/fastlane/Fastfile
lane :beta do
  # ตรวจสอบ branch
  ensure_git_branch(branch: /^(main|develop|release.*)$/)
  
  # Match certificates
  match(type: "appstore")
  
  # Build
  sh("flutter build ios --release --no-codesign", dir: "../..")
  
  build_app(
    workspace: "Runner.xcworkspace",
    scheme: "Runner",
    export_method: "app-store"
  )
  
  # Upload to TestFlight
  upload_to_testflight(
    api_key_path: "fastlane/app_store_connect_api_key.json",
    
    # กำหนด testers
    groups: ["Internal Team", "Beta Testers"],
    distribute_external: true,
    
    # Release notes
    changelog: lane_context[SharedValues::FL_CHANGELOG] || "Bug fixes and improvements",
    
    # ไม่ notify ถ้าเป็น internal
    notify_external_testers: true,
    
    # Wait สำหรับ processing
    wait_processing_interval: 30
  )
  
  # แจ้งผ่าน Slack
  slack(
    message: "New beta version uploaded to TestFlight!",
    slack_url: ENV["SLACK_WEBHOOK_URL"],
    success: true
  )
end
```

### จัดการ TestFlight Groups

```ruby
# เพิ่ม testers ด้วย Fastlane
lane :add_testers do
  # เพิ่ม tester คนเดียว
  add_test_flight_tester(
    api_key_path: "fastlane/app_store_connect_api_key.json",
    email: "tester@example.com",
    first_name: "John",
    last_name: "Doe"
  )
  
  # เพิ่มจาก CSV file
  # CSV format: email,first_name,last_name
  testers = CSV.read("testers.csv")
  testers.each do |row|
    add_test_flight_tester(
      api_key_path: "fastlane/app_store_connect_api_key.json",
      email: row[0],
      first_name: row[1],
      last_name: row[2]
    )
  end
end
```

---

## 4. Firebase App Distribution

### Setup Firebase App Distribution

```bash
# ติดตั้ง Firebase CLI
npm install -g firebase-tools

# Login
firebase login

# Initialize project
firebase init appdistribution
```

### การ Upload ด้วย Firebase CLI

```bash
# Upload APK
firebase appdistribution:distribute build/app/outputs/apk/release/app-release.apk \
  --app YOUR_FIREBASE_APP_ID \
  --groups "internal-testers,qa-team" \
  --release-notes "New features: Dark mode, Push notifications"

# Upload IPA
firebase appdistribution:distribute build/ios-ipa/MyApp.ipa \
  --app YOUR_FIREBASE_APP_ID \
  --groups "ios-testers" \
  --release-notes "Bug fixes"
```

### Firebase Distribution ใน Flutter

```dart
// pubspec.yaml
// dev_dependencies:
//   firebase_app_distribution: ^0.3.0

// เพิ่ม in-app update prompt สำหรับ testers
import 'package:firebase_app_distribution/firebase_app_distribution.dart';

class UpdateService {
  Future<void> checkForUpdate() async {
    try {
      final newRelease = await FirebaseAppDistribution.instance
          .checkForNewRelease();
      
      if (newRelease != null) {
        _showUpdateDialog(newRelease);
      }
    } on FirebaseAppDistributionException catch (e) {
      if (e.code == FirebaseAppDistributionExceptionCode.notSignedIn) {
        // Tester ยังไม่ได้ sign in
        await FirebaseAppDistribution.instance.signInTester();
      }
    }
  }
  
  void _showUpdateDialog(AppDistributionRelease release) {
    showDialog(
      context: navigatorKey.currentContext!,
      builder: (context) => AlertDialog(
        title: Text('มีการอัปเดตใหม่!'),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text('เวอร์ชัน ${release.displayVersion}'),
            SizedBox(height: 8),
            if (release.releaseNotes != null)
              Text(release.releaseNotes!),
          ],
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: Text('ภายหลัง'),
          ),
          ElevatedButton(
            onPressed: () async {
              Navigator.pop(context);
              await FirebaseAppDistribution.instance.updateIfNewReleaseAvailable();
            },
            child: Text('อัปเดตเลย'),
          ),
        ],
      ),
    );
  }
}
```

---

## 5. Version Management

### Semantic Versioning

```
MAJOR.MINOR.PATCH+BUILD_NUMBER

1.0.0+1  - Initial release
1.0.1+2  - Bug fix (PATCH)
1.1.0+3  - New feature (MINOR)
2.0.0+4  - Breaking change (MAJOR)
```

### Script จัดการ Version

```bash
#!/bin/bash
# scripts/bump_version.sh

# ใช้: ./scripts/bump_version.sh [major|minor|patch]

PUBSPEC="pubspec.yaml"
BUMP_TYPE=${1:-patch}

# อ่าน current version
CURRENT_VERSION=$(grep "^version:" $PUBSPEC | awk '{print $2}')
VERSION_NAME=$(echo $CURRENT_VERSION | cut -d'+' -f1)
BUILD_NUMBER=$(echo $CURRENT_VERSION | cut -d'+' -f2)

# แยก components
IFS='.' read -ra PARTS <<< "$VERSION_NAME"
MAJOR=${PARTS[0]}
MINOR=${PARTS[1]}
PATCH=${PARTS[2]}

# เพิ่ม version ตาม type
case $BUMP_TYPE in
  major)
    MAJOR=$((MAJOR + 1))
    MINOR=0
    PATCH=0
    ;;
  minor)
    MINOR=$((MINOR + 1))
    PATCH=0
    ;;
  patch)
    PATCH=$((PATCH + 1))
    ;;
esac

# เพิ่ม build number เสมอ
NEW_BUILD=$((BUILD_NUMBER + 1))
NEW_VERSION="$MAJOR.$MINOR.$PATCH+$NEW_BUILD"

# อัปเดต pubspec.yaml
sed -i "s/^version: .*/version: $NEW_VERSION/" $PUBSPEC

echo "Version bumped: $CURRENT_VERSION -> $NEW_VERSION"

# Commit
git add $PUBSPEC
git commit -m "Bump version to $NEW_VERSION"
git tag "v$MAJOR.$MINOR.$PATCH"
```

### CHANGELOG.md Management

```bash
#!/bin/bash
# scripts/generate_changelog.sh

CHANGELOG="CHANGELOG.md"
VERSION=$(grep "^version:" pubspec.yaml | awk '{print $2}' | cut -d'+' -f1)
DATE=$(date +%Y-%m-%d)

# สร้าง changelog entry จาก git commits
LAST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "")
if [ -n "$LAST_TAG" ]; then
  COMMITS=$(git log $LAST_TAG..HEAD --pretty=format:"- %s" --no-merges)
else
  COMMITS=$(git log --pretty=format:"- %s" --no-merges -20)
fi

# เพิ่มเข้าต้นไฟล์
NEW_ENTRY="## [$VERSION] - $DATE\n\n$COMMITS\n\n"

# Prepend to CHANGELOG
echo -e "$NEW_ENTRY$(cat $CHANGELOG 2>/dev/null)" > $CHANGELOG

echo "Changelog updated for version $VERSION"
```

---

## Workshop: Release Preparation Checklist

### Release Checklist Script

```dart
// lib/tools/release_checklist.dart
// Script สำหรับตรวจสอบความพร้อมก่อน release

class ReleaseChecklist {
  final List<ChecklistItem> items = [];
  
  ReleaseChecklist() {
    _addItems();
  }
  
  void _addItems() {
    // Code Quality
    items.addAll([
      ChecklistItem(
        category: 'Code Quality',
        title: 'รัน flutter analyze',
        command: 'flutter analyze',
        autoCheck: true,
      ),
      ChecklistItem(
        category: 'Code Quality',
        title: 'รัน tests ทั้งหมด',
        command: 'flutter test',
        autoCheck: true,
      ),
      ChecklistItem(
        category: 'Code Quality',
        title: 'ตรวจสอบ test coverage >= 80%',
        command: 'flutter test --coverage',
        autoCheck: false,
      ),
    ]);
    
    // Version
    items.addAll([
      ChecklistItem(
        category: 'Version',
        title: 'อัปเดต version ใน pubspec.yaml',
        autoCheck: false,
      ),
      ChecklistItem(
        category: 'Version',
        title: 'อัปเดต CHANGELOG.md',
        autoCheck: false,
      ),
    ]);
    
    // Android
    items.addAll([
      ChecklistItem(
        category: 'Android',
        title: 'Build Android AAB สำเร็จ',
        command: 'flutter build appbundle --release',
        autoCheck: true,
      ),
      ChecklistItem(
        category: 'Android',
        title: 'ตรวจสอบ APK size ไม่เกิน 50MB',
        autoCheck: false,
      ),
      ChecklistItem(
        category: 'Android',
        title: 'ทดสอบบน Android device จริง',
        autoCheck: false,
      ),
    ]);
    
    // iOS
    items.addAll([
      ChecklistItem(
        category: 'iOS',
        title: 'Build iOS สำเร็จ',
        command: 'flutter build ios --release',
        autoCheck: true,
      ),
      ChecklistItem(
        category: 'iOS',
        title: 'ทดสอบบน iPhone จริง',
        autoCheck: false,
      ),
      ChecklistItem(
        category: 'iOS',
        title: 'ผ่าน TestFlight testing',
        autoCheck: false,
      ),
    ]);
    
    // App Store Listing
    items.addAll([
      ChecklistItem(
        category: 'App Store Listing',
        title: 'อัปเดต What\'s New',
        autoCheck: false,
      ),
      ChecklistItem(
        category: 'App Store Listing',
        title: 'Screenshots อัปเดตแล้ว',
        autoCheck: false,
      ),
      ChecklistItem(
        category: 'App Store Listing',
        title: 'Privacy Policy อัปเดตแล้ว',
        autoCheck: false,
      ),
    ]);
  }
  
  Future<void> runAutoChecks() async {
    for (final item in items.where((i) => i.autoCheck)) {
      print('🔄 Checking: ${item.title}');
      try {
        final result = await Process.run(
          'bash',
          ['-c', item.command!],
        );
        item.passed = result.exitCode == 0;
        item.output = result.stdout.toString();
        
        if (item.passed) {
          print('  ✅ Passed');
        } else {
          print('  ❌ Failed: ${result.stderr}');
        }
      } catch (e) {
        item.passed = false;
        print('  ❌ Error: $e');
      }
    }
  }
  
  void printReport() {
    print('\n📋 Release Checklist Report\n');
    
    final categories = items.map((i) => i.category).toSet();
    
    for (final category in categories) {
      print('=== $category ===');
      final categoryItems = items.where((i) => i.category == category);
      
      for (final item in categoryItems) {
        final status = item.passed == null
            ? '⬜'
            : item.passed!
                ? '✅'
                : '❌';
        print('  $status ${item.title}');
      }
      print('');
    }
    
    final total = items.length;
    final passed = items.where((i) => i.passed == true).length;
    final pending = items.where((i) => i.passed == null).length;
    final failed = items.where((i) => i.passed == false).length;
    
    print('Summary: $passed/$total passed, $pending pending, $failed failed');
    
    if (failed > 0) {
      print('\n❌ ยังไม่พร้อม Release กรุณาแก้ไขปัญหาที่ failed ก่อน');
    } else if (pending > 0) {
      print('\n⚠️  กรุณาตรวจสอบ items ที่ pending ด้วยตัวเอง');
    } else {
      print('\n🎉 พร้อม Release!');
    }
  }
}

class ChecklistItem {
  final String category;
  final String title;
  final String? command;
  final bool autoCheck;
  bool? passed;
  String? output;
  
  ChecklistItem({
    required this.category,
    required this.title,
    this.command,
    required this.autoCheck,
  });
}
```

### Release Notes Template

```markdown
## What's New in Version X.Y.Z

### New Features
- ✨ [Feature 1 description]
- ✨ [Feature 2 description]

### Improvements
- 🚀 [Improvement 1]
- 🚀 [Improvement 2]

### Bug Fixes
- 🐛 Fixed [issue 1]
- 🐛 Fixed [issue 2]

### Breaking Changes (if any)
- ⚠️ [Breaking change description]

### Requirements
- Minimum iOS: 15.0
- Minimum Android: API 21 (Android 5.0)
```

---

## สรุป

App Distribution ประกอบด้วย:

1. **Google Play Store** - AAB + signing + listing
2. **Apple App Store** - IPA + certificates + metadata
3. **TestFlight** - Beta distribution สำหรับ iOS
4. **Firebase App Distribution** - Cross-platform beta testing
5. **Version Management** - Semantic versioning + CHANGELOG
6. **Release Checklist** - ตรวจสอบความพร้อมก่อน release
