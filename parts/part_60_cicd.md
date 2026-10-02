# Part 60: CI/CD Pipeline สำหรับ Flutter

## CI/CD คืออะไร?

- **CI (Continuous Integration)**: รัน tests และ build อัตโนมัติทุกครั้งที่ push code
- **CD (Continuous Delivery/Deployment)**: Deploy แอปไปยัง store อัตโนมัติ

ประโยชน์:
- ตรวจจับ bug ได้เร็วขึ้น
- ลดความผิดพลาดจากการ deploy ด้วยมือ
- สร้าง feedback loop ที่รวดเร็ว

---

## 1. GitHub Actions สำหรับ Flutter

### โครงสร้าง GitHub Actions

```yaml
# .github/workflows/ci.yml
name: Flutter CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    name: Test & Analyze
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.0'
          channel: 'stable'
          cache: true
      
      - name: Install dependencies
        run: flutter pub get
      
      - name: Analyze code
        run: flutter analyze
      
      - name: Run tests
        run: flutter test --coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: coverage/lcov.info
```

### Build สำหรับ Android

```yaml
# .github/workflows/build-android.yml
name: Build Android

on:
  push:
    branches: [main]
  workflow_dispatch: # อนุญาตให้ trigger ด้วยมือ

jobs:
  build-android:
    name: Build Android APK/AAB
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'
      
      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.0'
          cache: true
      
      - name: Install dependencies
        run: flutter pub get
      
      - name: Run tests
        run: flutter test
      
      # Decode keystore จาก GitHub Secrets
      - name: Decode Keystore
        run: |
          echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 --decode > android/app/keystore.jks
      
      # สร้าง key.properties
      - name: Create key.properties
        run: |
          cat > android/key.properties << EOF
          storePassword=${{ secrets.STORE_PASSWORD }}
          keyPassword=${{ secrets.KEY_PASSWORD }}
          keyAlias=${{ secrets.KEY_ALIAS }}
          storeFile=keystore.jks
          EOF
      
      # Build APK สำหรับ testing
      - name: Build APK
        run: flutter build apk --release --split-per-abi
      
      # Build AAB สำหรับ Play Store
      - name: Build AAB
        run: flutter build appbundle --release
      
      # Upload artifacts
      - name: Upload APK
        uses: actions/upload-artifact@v4
        with:
          name: android-apk
          path: build/app/outputs/flutter-apk/*.apk
      
      - name: Upload AAB
        uses: actions/upload-artifact@v4
        with:
          name: android-aab
          path: build/app/outputs/bundle/release/*.aab
```

### Build สำหรับ iOS

```yaml
# .github/workflows/build-ios.yml
name: Build iOS

on:
  push:
    branches: [main]

jobs:
  build-ios:
    name: Build iOS IPA
    runs-on: macos-latest # ต้องใช้ macOS สำหรับ iOS build
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.0'
      
      - name: Install dependencies
        run: flutter pub get
      
      # ติดตั้ง certificates และ provisioning profiles
      - name: Install Apple Certificate
        env:
          BUILD_CERTIFICATE_BASE64: ${{ secrets.BUILD_CERTIFICATE_BASE64 }}
          P12_PASSWORD: ${{ secrets.P12_PASSWORD }}
          KEYCHAIN_PASSWORD: ${{ secrets.KEYCHAIN_PASSWORD }}
        run: |
          # สร้าง keychain ชั่วคราว
          security create-keychain -p "$KEYCHAIN_PASSWORD" build.keychain
          security set-keychain-settings -lut 21600 build.keychain
          security unlock-keychain -p "$KEYCHAIN_PASSWORD" build.keychain
          
          # Import certificate
          echo "$BUILD_CERTIFICATE_BASE64" | base64 --decode > certificate.p12
          security import certificate.p12 -P "$P12_PASSWORD" \
            -A -t cert -f pkcs12 -k build.keychain
          
          security list-keychain -d user -s build.keychain
      
      - name: Install Provisioning Profile
        env:
          BUILD_PROVISION_PROFILE_BASE64: ${{ secrets.BUILD_PROVISION_PROFILE_BASE64 }}
        run: |
          PP_PATH=$RUNNER_TEMP/build_pp.mobileprovision
          echo "$BUILD_PROVISION_PROFILE_BASE64" | base64 --decode > "$PP_PATH"
          mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
          cp "$PP_PATH" ~/Library/MobileDevice/Provisioning\ Profiles
      
      # Build iOS
      - name: Build iOS
        run: |
          flutter build ios --release --no-codesign
      
      # Archive and export IPA
      - name: Export IPA
        run: |
          xcodebuild -workspace ios/Runner.xcworkspace \
            -scheme Runner \
            -sdk iphoneos \
            -configuration Release \
            -archivePath build/Runner.xcarchive \
            archive
          
          xcodebuild -exportArchive \
            -archivePath build/Runner.xcarchive \
            -exportOptionsPlist ios/ExportOptions.plist \
            -exportPath build/ios-ipa
      
      - name: Upload IPA
        uses: actions/upload-artifact@v4
        with:
          name: ios-ipa
          path: build/ios-ipa/*.ipa
```

---

## 2. Fastlane

### Fastlane Setup

```ruby
# Gemfile
source "https://rubygems.org"
gem "fastlane"
gem "cocoapods"
```

```bash
# ติดตั้ง Fastlane
gem install fastlane
cd ios && fastlane init
cd android && fastlane init
```

### Android Fastfile

```ruby
# android/fastlane/Fastfile
default_platform(:android)

platform :android do
  
  # Run tests
  lane :test do
    gradle(task: "test")
  end
  
  # Build debug APK
  lane :build_debug do
    sh("flutter build apk --debug", dir: "../..")
  end
  
  # Build release AAB
  lane :build_release do
    sh("flutter build appbundle --release", dir: "../..")
  end
  
  # Deploy to Firebase App Distribution
  lane :distribute_internal do
    build_release
    
    firebase_app_distribution(
      app: ENV["FIREBASE_APP_ID_ANDROID"],
      firebase_cli_token: ENV["FIREBASE_TOKEN"],
      groups: "internal-testers",
      release_notes: "Build from CI: #{last_git_commit[:message]}"
    )
  end
  
  # Deploy to Play Store (internal track)
  lane :deploy_internal do
    build_release
    
    upload_to_play_store(
      track: "internal",
      aab: "../../build/app/outputs/bundle/release/app-release.aab",
      json_key: ENV["PLAY_STORE_JSON_KEY_PATH"]
    )
  end
  
  # Deploy to Play Store (production)
  lane :deploy_production do
    # Ensure we're on main branch
    ensure_git_branch(branch: "main")
    ensure_git_status_clean
    
    build_release
    
    upload_to_play_store(
      track: "production",
      aab: "../../build/app/outputs/bundle/release/app-release.aab",
      json_key: ENV["PLAY_STORE_JSON_KEY_PATH"],
      rollout: "0.1" # 10% rollout
    )
    
    # Tag release
    add_git_tag(
      tag: "v#{get_version_name}+#{get_version_code}"
    )
    push_git_tags
  end

  # Version management
  lane :bump_version do |options|
    # อัปเดต version ใน pubspec.yaml
    version_bump_podspec(
      path: "../../pubspec.yaml",
      version_number: options[:version]
    )
    
    git_commit(
      path: ["../../pubspec.yaml"],
      message: "Bump version to #{options[:version]}"
    )
  end
end
```

### iOS Fastfile

```ruby
# ios/fastlane/Fastfile
default_platform(:ios)

platform :ios do
  
  # Match สำหรับ code signing
  lane :setup_certificates do
    match(
      type: "appstore",
      app_identifier: "com.example.myapp",
      readonly: true
    )
  end
  
  # Run tests
  lane :test do
    scan(
      workspace: "Runner.xcworkspace",
      scheme: "Runner",
      devices: ["iPhone 15"]
    )
  end
  
  # Build และ distribute ด้วย TestFlight
  lane :beta do
    setup_certificates
    
    sh("flutter build ios --release --no-codesign", dir: "../..")
    
    build_app(
      workspace: "Runner.xcworkspace",
      scheme: "Runner",
      export_method: "app-store"
    )
    
    upload_to_testflight(
      api_key_path: ENV["APP_STORE_CONNECT_API_KEY_PATH"],
      distribute_external: false,
      notify_external_testers: false,
      changelog: last_git_commit[:message]
    )
  end
  
  # Deploy to App Store
  lane :release do
    ensure_git_branch(branch: "main")
    
    setup_certificates
    
    sh("flutter build ios --release --no-codesign", dir: "../..")
    
    build_app(
      workspace: "Runner.xcworkspace",
      scheme: "Runner",
      export_method: "app-store"
    )
    
    upload_to_app_store(
      api_key_path: ENV["APP_STORE_CONNECT_API_KEY_PATH"],
      submit_for_review: false, # submit manually
      force: true,
      metadata_path: "./metadata",
      screenshots_path: "./screenshots"
    )
    
    add_git_tag(
      tag: "ios/v#{get_version_number}+#{get_build_number}"
    )
    push_git_tags
  end
end
```

---

## 3. Code Signing

### Android Keystore

```bash
# สร้าง keystore
keytool -genkey -v \
  -keystore ~/upload-keystore.jks \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000 \
  -alias upload

# Convert เป็น base64 สำหรับ GitHub Secrets
base64 -i ~/upload-keystore.jks | pbcopy
```

```properties
# android/key.properties (ไม่ commit ไปยัง git!)
storePassword=your-store-password
keyPassword=your-key-password
keyAlias=upload
storeFile=../upload-keystore.jks
```

```groovy
// android/app/build.gradle
def keystoreProperties = new Properties()
def keystorePropertiesFile = rootProject.file('key.properties')
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
}

android {
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
            shrinkResources true
        }
    }
}
```

### iOS Code Signing ด้วย Fastlane Match

```ruby
# ios/fastlane/Matchfile
git_url("https://github.com/your-org/certificates")
app_identifier("com.example.myapp")
username("your@email.com")
type("appstore") # อาจเป็น "development", "adhoc", หรือ "appstore"
```

```bash
# สร้างและ sync certificates
fastlane match appstore
fastlane match development
```

---

## 4. Automated Testing in CI

### Test Configuration

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  unit-tests:
    name: Unit & Widget Tests
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.0'
      
      - run: flutter pub get
      
      - name: Run unit tests
        run: flutter test test/unit/ --coverage
      
      - name: Run widget tests
        run: flutter test test/widget/
      
      - name: Generate coverage report
        run: |
          sudo apt-get install -y lcov
          genhtml coverage/lcov.info -o coverage/html
      
      - name: Check coverage threshold
        run: |
          COVERAGE=$(lcov --summary coverage/lcov.info 2>&1 | grep "lines" | awk '{print $2}' | sed 's/%//')
          echo "Coverage: $COVERAGE%"
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage below 80%!"
            exit 1
          fi
  
  integration-tests:
    name: Integration Tests
    runs-on: macos-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.0'
      
      - run: flutter pub get
      
      # Start iOS Simulator
      - name: Start iOS Simulator
        run: |
          xcrun simctl boot "iPhone 15" || true
          xcrun simctl list devices
      
      - name: Run integration tests
        run: |
          flutter test integration_test/ \
            -d "iPhone 15" \
            --timeout 300
```

---

## 5. Deployment Automation

### Complete CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    tags:
      - 'v*' # trigger เมื่อ push tag เช่น v1.2.0

jobs:
  deploy-android:
    name: Deploy to Play Store
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'
      
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.0'
      
      - run: flutter pub get
      - run: flutter test
      
      - name: Setup keystore
        run: |
          echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 --decode > android/app/keystore.jks
          cat > android/key.properties << EOF
          storePassword=${{ secrets.STORE_PASSWORD }}
          keyPassword=${{ secrets.KEY_PASSWORD }}
          keyAlias=${{ secrets.KEY_ALIAS }}
          storeFile=keystore.jks
          EOF
      
      - name: Build AAB
        run: flutter build appbundle --release
      
      - name: Deploy to Play Store
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.PLAY_STORE_SERVICE_ACCOUNT_JSON }}
          packageName: com.example.myapp
          releaseFiles: build/app/outputs/bundle/release/*.aab
          track: internal
          status: completed
  
  deploy-ios:
    name: Deploy to TestFlight
    runs-on: macos-latest
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.0'
      
      - run: flutter pub get
      - run: flutter test
      
      - name: Install certificates
        env:
          BUILD_CERTIFICATE_BASE64: ${{ secrets.BUILD_CERTIFICATE_BASE64 }}
          P12_PASSWORD: ${{ secrets.P12_PASSWORD }}
          PROVISION_PROFILE_BASE64: ${{ secrets.PROVISION_PROFILE_BASE64 }}
          KEYCHAIN_PASSWORD: ${{ secrets.KEYCHAIN_PASSWORD }}
        run: |
          security create-keychain -p "$KEYCHAIN_PASSWORD" build.keychain
          security set-keychain-settings -lut 21600 build.keychain
          security unlock-keychain -p "$KEYCHAIN_PASSWORD" build.keychain
          
          echo "$BUILD_CERTIFICATE_BASE64" | base64 --decode > certificate.p12
          security import certificate.p12 -P "$P12_PASSWORD" \
            -A -t cert -f pkcs12 -k build.keychain
          security list-keychain -d user -s build.keychain
          
          echo "$PROVISION_PROFILE_BASE64" | base64 --decode > profile.mobileprovision
          mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
          cp profile.mobileprovision ~/Library/MobileDevice/Provisioning\ Profiles
      
      - name: Build and Upload to TestFlight
        env:
          APP_STORE_CONNECT_API_KEY_ID: ${{ secrets.APP_STORE_CONNECT_API_KEY_ID }}
          APP_STORE_CONNECT_API_KEY_ISSUER_ID: ${{ secrets.APP_STORE_CONNECT_API_KEY_ISSUER_ID }}
          APP_STORE_CONNECT_API_KEY_CONTENT: ${{ secrets.APP_STORE_CONNECT_API_KEY_CONTENT }}
        run: |
          flutter build ios --release --no-codesign
          
          cd ios
          xcodebuild -workspace Runner.xcworkspace \
            -scheme Runner \
            -sdk iphoneos \
            -configuration Release \
            -archivePath ../build/Runner.xcarchive \
            archive
          
          xcodebuild -exportArchive \
            -archivePath ../build/Runner.xcarchive \
            -exportOptionsPlist ExportOptions.plist \
            -exportPath ../build/ios-ipa
          
          xcrun altool --upload-app \
            -f ../build/ios-ipa/*.ipa \
            --apiKey "$APP_STORE_CONNECT_API_KEY_ID" \
            --apiIssuer "$APP_STORE_CONNECT_API_KEY_ISSUER_ID"
  
  create-github-release:
    name: Create GitHub Release
    runs-on: ubuntu-latest
    needs: [deploy-android, deploy-ios]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Download artifacts
        uses: actions/download-artifact@v4
      
      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          generate_release_notes: true
          files: |
            android-apk/*.apk
          body: |
            ## What's New
            ${{ github.event.head_commit.message }}
            
            ## Downloads
            - Android APK: See attached files
            - iOS: Available on TestFlight
```

---

## Workshop: GitHub Actions Workflow

### สร้าง Workflow สมบูรณ์

```yaml
# .github/workflows/flutter-complete.yml
name: Flutter Complete CI/CD

on:
  push:
    branches: [main, develop, 'feature/*', 'hotfix/*']
    tags: ['v*']
  pull_request:
    branches: [main, develop]

env:
  FLUTTER_VERSION: '3.24.0'

jobs:
  # ========= ANALYZE & TEST =========
  analyze:
    name: Analyze
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          cache: true
      - run: flutter pub get
      - name: Analyze
        run: flutter analyze --no-fatal-infos
      - name: Check formatting
        run: dart format --set-exit-if-changed .

  test:
    name: Test
    runs-on: ubuntu-latest
    needs: analyze
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          cache: true
      - run: flutter pub get
      - name: Run tests with coverage
        run: flutter test --coverage --reporter=github
      - uses: codecov/codecov-action@v3

  # ========= BUILD =========
  build-android:
    name: Build Android
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/tags/')
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          cache: true
      - run: flutter pub get
      - name: Setup signing
        run: |
          echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 --decode > android/app/keystore.jks
          printf 'storePassword=%s\nkeyPassword=%s\nkeyAlias=%s\nstoreFile=keystore.jks\n' \
            "${{ secrets.STORE_PASSWORD }}" \
            "${{ secrets.KEY_PASSWORD }}" \
            "${{ secrets.KEY_ALIAS }}" \
            > android/key.properties
      - run: flutter build appbundle --release
      - uses: actions/upload-artifact@v4
        with:
          name: android-aab
          path: build/app/outputs/bundle/release/*.aab
          retention-days: 30

  # ========= DEPLOY =========
  deploy-firebase:
    name: Deploy to Firebase App Distribution
    runs-on: ubuntu-latest
    needs: build-android
    if: github.ref == 'refs/heads/main'
    environment: staging
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with:
          name: android-aab
          path: build/
      
      - name: Deploy to Firebase
        uses: wzieba/Firebase-Distribution-Github-Action@v1
        with:
          appId: ${{ secrets.FIREBASE_APP_ID }}
          token: ${{ secrets.FIREBASE_TOKEN }}
          groups: internal-testers
          file: build/*.aab
          releaseNotes: "Branch: ${{ github.ref_name }} | Commit: ${{ github.sha }}"

  deploy-playstore:
    name: Deploy to Play Store
    runs-on: ubuntu-latest
    needs: build-android
    if: startsWith(github.ref, 'refs/tags/')
    environment: production
    
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: android-aab
          path: build/
      
      - uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.PLAY_STORE_JSON }}
          packageName: com.example.myapp
          releaseFiles: build/*.aab
          track: production
          userFraction: 0.1
          status: inProgress

  # ========= NOTIFY =========
  notify:
    name: Notify
    runs-on: ubuntu-latest
    needs: [deploy-firebase, deploy-playstore]
    if: always()
    
    steps:
      - name: Send Slack notification
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          fields: repo,message,commit,author,action,eventName,ref,workflow
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### .gitignore สำหรับ keys

```gitignore
# .gitignore
# Android signing
android/key.properties
android/app/keystore.jks
android/app/*.jks

# iOS signing
ios/fastlane/report.xml
*.mobileprovision
*.p12
*.cer

# Firebase
google-services.json
GoogleService-Info.plist

# Environment files
.env
.env.*
!.env.example
```

---

## สรุป

CI/CD Pipeline สำหรับ Flutter ประกอบด้วย:

1. **GitHub Actions** - รัน tests และ build อัตโนมัติ
2. **Fastlane** - จัดการ code signing และ deployment
3. **Code Signing** - keystore สำหรับ Android, certificates สำหรับ iOS
4. **Automated Testing** - unit, widget, และ integration tests
5. **Deployment** - ส่งไปยัง Firebase Distribution, Play Store, และ TestFlight
