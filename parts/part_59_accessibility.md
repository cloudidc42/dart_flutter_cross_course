# Part 59: Accessibility ใน Flutter

## Accessibility คืออะไร?

Accessibility (a11y) คือการทำให้แอปสามารถใช้งานได้โดยทุกคน รวมถึงผู้ที่มีความบกพร่อง:
- ผู้พิการทางสายตา (ใช้ Screen Reader)
- ผู้ที่มีปัญหาการมองเห็นสี
- ผู้ที่มีปัญหาการเคลื่อนไหว (ใช้ Keyboard navigation)
- ผู้สูงอายุ

---

## 1. Semantics Widget

### การใช้งาน Semantics พื้นฐาน

```dart
// Semantics Widget เพิ่มข้อมูลสำหรับ Screen Readers
class BasicSemanticsExample extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // เพิ่ม label สำหรับ icon ที่ไม่มีข้อความ
        Semantics(
          label: 'ปุ่มค้นหา',
          button: true,
          child: IconButton(
            icon: Icon(Icons.search),
            onPressed: () {},
          ),
        ),

        // กำหนด hint สำหรับ screen reader
        Semantics(
          label: 'กดเพื่อดูรายละเอียดสินค้า',
          hint: 'เปิดหน้ารายละเอียด',
          button: true,
          child: GestureDetector(
            onTap: () {},
            child: Card(
              child: Padding(
                padding: EdgeInsets.all(16),
                child: Text('iPhone 15 Pro'),
              ),
            ),
          ),
        ),

        // Image ต้องมี label เสมอ
        Semantics(
          label: 'รูปภาพโปรไฟล์ของ สมชาย',
          child: CircleAvatar(
            radius: 40,
            backgroundImage: NetworkImage('https://example.com/avatar.jpg'),
          ),
        ),

        // ซ่อน Widget จาก Screen Reader (decorative)
        Semantics(
          excludeSemantics: true,
          child: Icon(Icons.star, color: Colors.amber),
        ),
      ],
    );
  }
}
```

### MergeSemantics

```dart
// รวม semantics ของ Widget ย่อยเข้าด้วยกัน
class MergeSemanticsExample extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MergeSemantics(
      // Screen reader จะอ่านทั้งหมดเป็นข้อความเดียว
      child: Row(
        children: [
          Icon(Icons.favorite, color: Colors.red),
          SizedBox(width: 8),
          Text('ชื่นชอบ'),
          Spacer(),
          Text('234'),
        ],
      ),
    );
  }
}
```

### Semantics สำหรับ Custom Widgets

```dart
// Rating widget ที่มี proper semantics
class AccessibleRatingWidget extends StatelessWidget {
  final double rating;
  final int maxRating;
  final Function(int)? onRatingChanged;

  const AccessibleRatingWidget({
    required this.rating,
    this.maxRating = 5,
    this.onRatingChanged,
  });

  @override
  Widget build(BuildContext context) {
    return Semantics(
      // แสดงค่าปัจจุบันและ range
      value: '${rating.toStringAsFixed(1)} จาก $maxRating ดาว',
      label: 'คะแนน',
      // ถ้าแก้ไขได้ ให้เพิ่ม slider semantics
      slider: onRatingChanged != null,
      child: Row(
        mainAxisSize: MainAxisSize.min,
        children: List.generate(maxRating, (index) {
          final starValue = index + 1;
          final isFilled = starValue <= rating;

          return Semantics(
            excludeSemantics: true, // ซ่อนดาวแต่ละดวงจาก SR
            child: GestureDetector(
              onTap: onRatingChanged != null
                  ? () => onRatingChanged!(starValue)
                  : null,
              child: Icon(
                isFilled ? Icons.star : Icons.star_border,
                color: Colors.amber,
                size: 24,
              ),
            ),
          );
        }),
      ),
    );
  }
}
```

---

## 2. Screen Reader Support

### TalkBack (Android) และ VoiceOver (iOS)

```dart
// Widget ที่รองรับ Screen Reader อย่างสมบูรณ์
class ScreenReaderFriendlyForm extends StatefulWidget {
  @override
  _ScreenReaderFriendlyFormState createState() =>
      _ScreenReaderFriendlyFormState();
}

class _ScreenReaderFriendlyFormState
    extends State<ScreenReaderFriendlyForm> {
  final _nameController = TextEditingController();
  final _emailController = TextEditingController();
  bool _isLoading = false;
  String? _successMessage;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Semantics(
          header: true, // บอก SR ว่าเป็น heading
          child: Text('แบบฟอร์มลงทะเบียน'),
        ),
      ),
      body: Padding(
        padding: EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            // Live region สำหรับ dynamic content
            if (_successMessage != null)
              Semantics(
                liveRegion: true, // SR จะประกาศเมื่อข้อความเปลี่ยน
                child: Container(
                  padding: EdgeInsets.all(12),
                  decoration: BoxDecoration(
                    color: Colors.green.shade100,
                    borderRadius: BorderRadius.circular(8),
                  ),
                  child: Text(
                    _successMessage!,
                    style: TextStyle(color: Colors.green.shade800),
                  ),
                ),
              ),

            // Form fields
            _buildTextField(
              controller: _nameController,
              label: 'ชื่อ-นามสกุล',
              hint: 'กรอกชื่อและนามสกุลของคุณ',
              icon: Icons.person,
            ),

            SizedBox(height: 16),

            _buildTextField(
              controller: _emailController,
              label: 'อีเมล',
              hint: 'กรอกอีเมลของคุณ',
              icon: Icons.email,
              keyboardType: TextInputType.emailAddress,
            ),

            SizedBox(height: 24),

            // Button ที่มี loading state ที่ชัดเจน
            Semantics(
              button: true,
              enabled: !_isLoading,
              label: _isLoading ? 'กำลังดำเนินการ...' : 'ลงทะเบียน',
              child: ElevatedButton(
                onPressed: _isLoading ? null : _handleSubmit,
                child: _isLoading
                    ? SizedBox(
                        height: 20,
                        width: 20,
                        child: CircularProgressIndicator(
                          strokeWidth: 2,
                          color: Colors.white,
                        ),
                      )
                    : Text('ลงทะเบียน'),
              ),
            ),
          ],
        ),
      ),
    );
  }

  Widget _buildTextField({
    required TextEditingController controller,
    required String label,
    required String hint,
    required IconData icon,
    TextInputType? keyboardType,
  }) {
    return TextField(
      controller: controller,
      keyboardType: keyboardType,
      decoration: InputDecoration(
        labelText: label,
        hintText: hint,
        prefixIcon: Icon(icon),
        border: OutlineInputBorder(),
      ),
    );
  }

  void _handleSubmit() async {
    setState(() => _isLoading = true);
    await Future.delayed(Duration(seconds: 2));
    setState(() {
      _isLoading = false;
      _successMessage = 'ลงทะเบียนสำเร็จ! ยินดีต้อนรับ';
    });
  }

  @override
  void dispose() {
    _nameController.dispose();
    _emailController.dispose();
    super.dispose();
  }
}
```

### Focus Management

```dart
// การจัดการ Focus สำหรับ keyboard navigation
class FocusManagementExample extends StatefulWidget {
  @override
  _FocusManagementExampleState createState() =>
      _FocusManagementExampleState();
}

class _FocusManagementExampleState extends State<FocusManagementExample> {
  final _firstNameFocus = FocusNode();
  final _lastNameFocus = FocusNode();
  final _emailFocus = FocusNode();
  final _phoneFocus = FocusNode();

  @override
  void dispose() {
    _firstNameFocus.dispose();
    _lastNameFocus.dispose();
    _emailFocus.dispose();
    _phoneFocus.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        TextField(
          focusNode: _firstNameFocus,
          decoration: InputDecoration(labelText: 'ชื่อ'),
          textInputAction: TextInputAction.next, // Next button
          onSubmitted: (_) {
            // ย้าย focus ไป field ถัดไป
            FocusScope.of(context).requestFocus(_lastNameFocus);
          },
        ),

        TextField(
          focusNode: _lastNameFocus,
          decoration: InputDecoration(labelText: 'นามสกุล'),
          textInputAction: TextInputAction.next,
          onSubmitted: (_) {
            FocusScope.of(context).requestFocus(_emailFocus);
          },
        ),

        TextField(
          focusNode: _emailFocus,
          decoration: InputDecoration(labelText: 'อีเมล'),
          keyboardType: TextInputType.emailAddress,
          textInputAction: TextInputAction.next,
          onSubmitted: (_) {
            FocusScope.of(context).requestFocus(_phoneFocus);
          },
        ),

        TextField(
          focusNode: _phoneFocus,
          decoration: InputDecoration(labelText: 'เบอร์โทรศัพท์'),
          keyboardType: TextInputType.phone,
          textInputAction: TextInputAction.done, // Done button
          onSubmitted: (_) {
            _phoneFocus.unfocus(); // ปิด keyboard
          },
        ),
      ],
    );
  }
}
```

---

## 3. Color Contrast

### WCAG Color Contrast Guidelines

```dart
// ตรวจสอบ color contrast ratio
class ColorContrastChecker {
  // คำนวณ relative luminance
  static double _getLuminance(Color color) {
    final r = color.red / 255;
    final g = color.green / 255;
    final b = color.blue / 255;

    double toLinear(double c) {
      return c <= 0.03928 ? c / 12.92 : ((c + 0.055) / 1.055) * 2.4;
    }

    return 0.2126 * toLinear(r) +
        0.7152 * toLinear(g) +
        0.0722 * toLinear(b);
  }

  // คำนวณ contrast ratio
  static double getContrastRatio(Color foreground, Color background) {
    final fgLum = _getLuminance(foreground);
    final bgLum = _getLuminance(background);

    final lighter = fgLum > bgLum ? fgLum : bgLum;
    final darker = fgLum > bgLum ? bgLum : fgLum;

    return (lighter + 0.05) / (darker + 0.05);
  }

  // ตรวจสอบว่าผ่าน WCAG AA (4.5:1 สำหรับ text ปกติ, 3:1 สำหรับ large text)
  static bool passesWCAG_AA(Color foreground, Color background,
      {bool isLargeText = false}) {
    final ratio = getContrastRatio(foreground, background);
    return isLargeText ? ratio >= 3.0 : ratio >= 4.5;
  }

  // ตรวจสอบว่าผ่าน WCAG AAA (7:1 สำหรับ text ปกติ, 4.5:1 สำหรับ large text)
  static bool passesWCAG_AAA(Color foreground, Color background,
      {bool isLargeText = false}) {
    final ratio = getContrastRatio(foreground, background);
    return isLargeText ? ratio >= 4.5 : ratio >= 7.0;
  }
}

// ตัวอย่างการใช้งาน
class AccessibleColorWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // สีที่ผ่าน WCAG AA
    const textColor = Color(0xFF1A1A1A); // Dark gray
    const backgroundColor = Color(0xFFFFFFFF); // White
    const primaryColor = Color(0xFF1565C0); // Dark blue

    // ตรวจสอบ contrast
    final ratio = ColorContrastChecker.getContrastRatio(
      textColor,
      backgroundColor,
    );
    // ratio = 16.1:1 - ผ่าน WCAG AAA ✅

    return Container(
      color: backgroundColor,
      padding: EdgeInsets.all(16),
      child: Column(
        children: [
          Text(
            'ข้อความที่อ่านง่าย',
            style: TextStyle(
              color: textColor,
              fontSize: 16,
            ),
          ),
          ElevatedButton(
            style: ElevatedButton.styleFrom(
              backgroundColor: primaryColor,
              foregroundColor: Colors.white,
            ),
            onPressed: () {},
            child: Text('ปุ่มที่มองเห็นชัด'),
          ),
        ],
      ),
    );
  }
}
```

### Theme สำหรับ Accessibility

```dart
// Theme ที่คำนึงถึง accessibility
ThemeData createAccessibleTheme() {
  return ThemeData(
    // สีหลักที่มี contrast สูง
    colorScheme: ColorScheme.fromSeed(
      seedColor: Color(0xFF1565C0),
      brightness: Brightness.light,
    ).copyWith(
      // Ensure sufficient contrast
      onPrimary: Colors.white,
      onSecondary: Colors.white,
      onError: Colors.white,
    ),

    // Text ขนาดที่อ่านง่าย
    textTheme: TextTheme(
      bodyLarge: TextStyle(fontSize: 16, height: 1.5),
      bodyMedium: TextStyle(fontSize: 14, height: 1.5),
      titleLarge: TextStyle(fontSize: 22, fontWeight: FontWeight.bold),
    ),

    // เพิ่ม focus indicator ที่ชัดเจน
    focusColor: Color(0xFF1565C0).withOpacity(0.3),
    hoverColor: Color(0xFF1565C0).withOpacity(0.1),

    // Input decoration
    inputDecorationTheme: InputDecorationTheme(
      border: OutlineInputBorder(
        borderSide: BorderSide(width: 1.5),
      ),
      focusedBorder: OutlineInputBorder(
        borderSide: BorderSide(color: Color(0xFF1565C0), width: 2.5),
      ),
      errorBorder: OutlineInputBorder(
        borderSide: BorderSide(color: Colors.red.shade700, width: 2),
      ),
    ),
  );
}
```

---

## 4. Touch Target Sizes

### ขนาด Touch Target ที่เหมาะสม

```dart
// WCAG กำหนดขนาด touch target ขั้นต่ำ 44x44 pt
class AccessibleTouchTarget extends StatelessWidget {
  final Widget child;
  final VoidCallback? onTap;
  final double minSize;

  const AccessibleTouchTarget({
    required this.child,
    this.onTap,
    this.minSize = 44, // 44 pt ตาม Apple HIG และ WCAG
  });

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: onTap,
      behavior: HitTestBehavior.opaque,
      child: ConstrainedBox(
        constraints: BoxConstraints(
          minWidth: minSize,
          minHeight: minSize,
        ),
        child: Center(child: child),
      ),
    );
  }
}

// ตัวอย่างการใช้งาน
class TouchTargetExamples extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // ❌ Touch target เล็กเกินไป
        Row(
          children: [
            Text('❌ เล็กเกินไป: '),
            GestureDetector(
              onTap: () {},
              child: Icon(Icons.close, size: 16), // เล็กเกินไป!
            ),
          ],
        ),

        // ✅ Touch target ขนาดเหมาะสม
        Row(
          children: [
            Text('✅ ขนาดพอดี: '),
            AccessibleTouchTarget(
              onTap: () {},
              child: Icon(Icons.close, size: 16),
            ),
          ],
        ),

        // IconButton มีขนาดเหมาะสมอยู่แล้ว (48x48)
        IconButton(
          icon: Icon(Icons.delete),
          onPressed: () {},
          // ขนาดเริ่มต้น 48x48 - ผ่าน WCAG
        ),

        // Custom close button ที่มีขนาดพอดี
        CloseButton(onPressed: () {}),
      ],
    );
  }
}
```

### ListTile ที่ปรับ touch target

```dart
class AccessibleListTile extends StatelessWidget {
  final String title;
  final String? subtitle;
  final IconData icon;
  final VoidCallback? onTap;

  const AccessibleListTile({
    required this.title,
    this.subtitle,
    required this.icon,
    this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    return Semantics(
      button: onTap != null,
      label: subtitle != null ? '$title, $subtitle' : title,
      child: InkWell(
        onTap: onTap,
        child: Padding(
          // Padding เพิ่ม touch area
          padding: EdgeInsets.symmetric(horizontal: 16, vertical: 12),
          child: Row(
            children: [
              SizedBox(
                width: 48,
                height: 48,
                child: Center(child: Icon(icon, size: 24)),
              ),
              SizedBox(width: 16),
              Expanded(
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(title,
                        style: TextStyle(
                            fontSize: 16, fontWeight: FontWeight.w500)),
                    if (subtitle != null)
                      Text(subtitle!,
                          style:
                              TextStyle(fontSize: 14, color: Colors.grey[600])),
                  ],
                ),
              ),
              if (onTap != null)
                SizedBox(
                  width: 48,
                  height: 48,
                  child: Center(
                    child: Icon(Icons.chevron_right, color: Colors.grey),
                  ),
                ),
            ],
          ),
        ),
      ),
    );
  }
}
```

---

## Workshop: Accessible Form

### สร้าง Form ที่ Accessible อย่างสมบูรณ์

```dart
// accessible_form.dart
import 'package:flutter/material.dart';
import 'package:flutter/semantics.dart';

class AccessibleRegistrationForm extends StatefulWidget {
  @override
  _AccessibleRegistrationFormState createState() =>
      _AccessibleRegistrationFormState();
}

class _AccessibleRegistrationFormState
    extends State<AccessibleRegistrationForm> {
  final _formKey = GlobalKey<FormState>();

  final _firstNameController = TextEditingController();
  final _lastNameController = TextEditingController();
  final _emailController = TextEditingController();
  final _phoneController = TextEditingController();
  final _passwordController = TextEditingController();
  final _confirmPasswordController = TextEditingController();

  final _firstNameFocus = FocusNode();
  final _lastNameFocus = FocusNode();
  final _emailFocus = FocusNode();
  final _phoneFocus = FocusNode();
  final _passwordFocus = FocusNode();
  final _confirmPasswordFocus = FocusNode();

  bool _passwordVisible = false;
  bool _confirmPasswordVisible = false;
  bool _agreeToTerms = false;
  bool _isSubmitting = false;
  String? _submitError;
  bool _submitSuccess = false;

  @override
  void dispose() {
    _firstNameController.dispose();
    _lastNameController.dispose();
    _emailController.dispose();
    _phoneController.dispose();
    _passwordController.dispose();
    _confirmPasswordController.dispose();
    _firstNameFocus.dispose();
    _lastNameFocus.dispose();
    _emailFocus.dispose();
    _phoneFocus.dispose();
    _passwordFocus.dispose();
    _confirmPasswordFocus.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Semantics(
          header: true,
          child: Text('สมัครสมาชิก'),
        ),
      ),
      body: SingleChildScrollView(
        padding: EdgeInsets.all(24),
        child: Form(
          key: _formKey,
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.stretch,
            children: [
              // Success/Error messages (live region)
              if (_submitSuccess)
                Semantics(
                  liveRegion: true,
                  child: _AlertBanner(
                    type: AlertType.success,
                    message: 'สมัครสมาชิกสำเร็จ! กรุณาตรวจสอบอีเมลของคุณ',
                  ),
                ),

              if (_submitError != null)
                Semantics(
                  liveRegion: true,
                  child: _AlertBanner(
                    type: AlertType.error,
                    message: _submitError!,
                  ),
                ),

              SizedBox(height: 16),

              // Section header
              Semantics(
                header: true,
                child: Text(
                  'ข้อมูลส่วนตัว',
                  style: Theme.of(context).textTheme.titleMedium?.copyWith(
                        fontWeight: FontWeight.bold,
                      ),
                ),
              ),
              SizedBox(height: 16),

              // First & Last name (side by side)
              Row(
                children: [
                  Expanded(
                    child: _AccessibleTextField(
                      controller: _firstNameController,
                      focusNode: _firstNameFocus,
                      label: 'ชื่อ',
                      hint: 'กรอกชื่อของคุณ',
                      icon: Icons.person_outline,
                      nextFocus: _lastNameFocus,
                      validator: (v) => v?.isEmpty == true ? 'กรุณากรอกชื่อ' : null,
                    ),
                  ),
                  SizedBox(width: 12),
                  Expanded(
                    child: _AccessibleTextField(
                      controller: _lastNameController,
                      focusNode: _lastNameFocus,
                      label: 'นามสกุล',
                      hint: 'กรอกนามสกุลของคุณ',
                      nextFocus: _emailFocus,
                      validator: (v) =>
                          v?.isEmpty == true ? 'กรุณากรอกนามสกุล' : null,
                    ),
                  ),
                ],
              ),
              SizedBox(height: 16),

              // Email
              _AccessibleTextField(
                controller: _emailController,
                focusNode: _emailFocus,
                label: 'อีเมล',
                hint: 'example@email.com',
                icon: Icons.email_outlined,
                keyboardType: TextInputType.emailAddress,
                nextFocus: _phoneFocus,
                validator: (v) {
                  if (v?.isEmpty == true) return 'กรุณากรอกอีเมล';
                  if (!v!.contains('@')) return 'รูปแบบอีเมลไม่ถูกต้อง';
                  return null;
                },
              ),
              SizedBox(height: 16),

              // Phone
              _AccessibleTextField(
                controller: _phoneController,
                focusNode: _phoneFocus,
                label: 'เบอร์โทรศัพท์',
                hint: '08X-XXX-XXXX',
                icon: Icons.phone_outlined,
                keyboardType: TextInputType.phone,
                nextFocus: _passwordFocus,
                validator: (v) {
                  if (v?.isEmpty == true) return 'กรุณากรอกเบอร์โทรศัพท์';
                  if (v!.length < 9) return 'เบอร์โทรศัพท์ไม่ถูกต้อง';
                  return null;
                },
              ),
              SizedBox(height: 24),

              // Password section
              Semantics(
                header: true,
                child: Text(
                  'รหัสผ่าน',
                  style: Theme.of(context).textTheme.titleMedium?.copyWith(
                        fontWeight: FontWeight.bold,
                      ),
                ),
              ),
              SizedBox(height: 16),

              // Password
              _AccessiblePasswordField(
                controller: _passwordController,
                focusNode: _passwordFocus,
                label: 'รหัสผ่าน',
                hint: 'อย่างน้อย 8 ตัวอักษร',
                isVisible: _passwordVisible,
                onToggleVisibility: () =>
                    setState(() => _passwordVisible = !_passwordVisible),
                nextFocus: _confirmPasswordFocus,
                validator: (v) {
                  if (v?.isEmpty == true) return 'กรุณากรอกรหัสผ่าน';
                  if (v!.length < 8) return 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร';
                  return null;
                },
              ),
              SizedBox(height: 16),

              // Confirm Password
              _AccessiblePasswordField(
                controller: _confirmPasswordController,
                focusNode: _confirmPasswordFocus,
                label: 'ยืนยันรหัสผ่าน',
                hint: 'กรอกรหัสผ่านอีกครั้ง',
                isVisible: _confirmPasswordVisible,
                onToggleVisibility: () => setState(
                    () => _confirmPasswordVisible = !_confirmPasswordVisible),
                isLastField: true,
                validator: (v) {
                  if (v != _passwordController.text) {
                    return 'รหัสผ่านไม่ตรงกัน';
                  }
                  return null;
                },
              ),
              SizedBox(height: 24),

              // Terms checkbox
              Semantics(
                label: 'ยอมรับเงื่อนไขการใช้งาน',
                toggled: _agreeToTerms,
                child: Row(
                  children: [
                    SizedBox(
                      width: 48,
                      height: 48,
                      child: Checkbox(
                        value: _agreeToTerms,
                        onChanged: (v) =>
                            setState(() => _agreeToTerms = v ?? false),
                      ),
                    ),
                    Expanded(
                      child: GestureDetector(
                        onTap: () =>
                            setState(() => _agreeToTerms = !_agreeToTerms),
                        child: Text.rich(
                          TextSpan(
                            text: 'ฉันยอมรับ ',
                            children: [
                              TextSpan(
                                text: 'เงื่อนไขการใช้งาน',
                                style: TextStyle(
                                  color: Colors.blue,
                                  decoration: TextDecoration.underline,
                                ),
                              ),
                              TextSpan(text: ' และ '),
                              TextSpan(
                                text: 'นโยบายความเป็นส่วนตัว',
                                style: TextStyle(
                                  color: Colors.blue,
                                  decoration: TextDecoration.underline,
                                ),
                              ),
                            ],
                          ),
                        ),
                      ),
                    ),
                  ],
                ),
              ),
              SizedBox(height: 24),

              // Submit button
              Semantics(
                button: true,
                enabled: !_isSubmitting && _agreeToTerms,
                label: _isSubmitting
                    ? 'กำลังสมัครสมาชิก...'
                    : 'สมัครสมาชิก',
                child: ElevatedButton(
                  onPressed:
                      !_isSubmitting && _agreeToTerms ? _handleSubmit : null,
                  style: ElevatedButton.styleFrom(
                    minimumSize: Size(double.infinity, 52), // touch target
                    shape: RoundedRectangleBorder(
                      borderRadius: BorderRadius.circular(8),
                    ),
                  ),
                  child: _isSubmitting
                      ? SizedBox(
                          height: 24,
                          width: 24,
                          child: CircularProgressIndicator(
                            strokeWidth: 2,
                            color: Colors.white,
                          ),
                        )
                      : Text('สมัครสมาชิก', style: TextStyle(fontSize: 16)),
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }

  Future<void> _handleSubmit() async {
    if (!_formKey.currentState!.validate()) return;
    if (!_agreeToTerms) {
      setState(() => _submitError = 'กรุณายอมรับเงื่อนไขการใช้งาน');
      return;
    }

    setState(() {
      _isSubmitting = true;
      _submitError = null;
    });

    try {
      await Future.delayed(Duration(seconds: 2));
      setState(() {
        _isSubmitting = false;
        _submitSuccess = true;
      });
    } catch (e) {
      setState(() {
        _isSubmitting = false;
        _submitError = 'เกิดข้อผิดพลาด กรุณาลองใหม่อีกครั้ง';
      });
    }
  }
}

// Helper Widgets
enum AlertType { success, error, warning }

class _AlertBanner extends StatelessWidget {
  final AlertType type;
  final String message;

  const _AlertBanner({required this.type, required this.message});

  @override
  Widget build(BuildContext context) {
    Color bgColor;
    Color textColor;
    IconData icon;

    switch (type) {
      case AlertType.success:
        bgColor = Colors.green.shade50;
        textColor = Colors.green.shade800;
        icon = Icons.check_circle;
        break;
      case AlertType.error:
        bgColor = Colors.red.shade50;
        textColor = Colors.red.shade800;
        icon = Icons.error;
        break;
      case AlertType.warning:
        bgColor = Colors.orange.shade50;
        textColor = Colors.orange.shade800;
        icon = Icons.warning;
        break;
    }

    return Container(
      padding: EdgeInsets.all(12),
      decoration: BoxDecoration(
        color: bgColor,
        borderRadius: BorderRadius.circular(8),
        border: Border.all(color: textColor.withOpacity(0.3)),
      ),
      child: Row(
        children: [
          Icon(icon, color: textColor),
          SizedBox(width: 12),
          Expanded(
            child: Text(message, style: TextStyle(color: textColor)),
          ),
        ],
      ),
    );
  }
}

class _AccessibleTextField extends StatelessWidget {
  final TextEditingController controller;
  final FocusNode focusNode;
  final String label;
  final String hint;
  final IconData? icon;
  final TextInputType? keyboardType;
  final FocusNode? nextFocus;
  final String? Function(String?)? validator;

  const _AccessibleTextField({
    required this.controller,
    required this.focusNode,
    required this.label,
    required this.hint,
    this.icon,
    this.keyboardType,
    this.nextFocus,
    this.validator,
  });

  @override
  Widget build(BuildContext context) {
    return TextFormField(
      controller: controller,
      focusNode: focusNode,
      keyboardType: keyboardType,
      textInputAction:
          nextFocus != null ? TextInputAction.next : TextInputAction.done,
      onFieldSubmitted: (_) {
        if (nextFocus != null) {
          FocusScope.of(context).requestFocus(nextFocus);
        }
      },
      decoration: InputDecoration(
        labelText: label,
        hintText: hint,
        prefixIcon: icon != null ? Icon(icon) : null,
        border: OutlineInputBorder(borderRadius: BorderRadius.circular(8)),
        focusedBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(8),
          borderSide: BorderSide(
            color: Theme.of(context).primaryColor,
            width: 2,
          ),
        ),
      ),
      validator: validator,
    );
  }
}

class _AccessiblePasswordField extends StatelessWidget {
  final TextEditingController controller;
  final FocusNode focusNode;
  final String label;
  final String hint;
  final bool isVisible;
  final VoidCallback onToggleVisibility;
  final FocusNode? nextFocus;
  final bool isLastField;
  final String? Function(String?)? validator;

  const _AccessiblePasswordField({
    required this.controller,
    required this.focusNode,
    required this.label,
    required this.hint,
    required this.isVisible,
    required this.onToggleVisibility,
    this.nextFocus,
    this.isLastField = false,
    this.validator,
  });

  @override
  Widget build(BuildContext context) {
    return TextFormField(
      controller: controller,
      focusNode: focusNode,
      obscureText: !isVisible,
      textInputAction:
          isLastField ? TextInputAction.done : TextInputAction.next,
      onFieldSubmitted: (_) {
        if (nextFocus != null) {
          FocusScope.of(context).requestFocus(nextFocus);
        }
      },
      decoration: InputDecoration(
        labelText: label,
        hintText: hint,
        prefixIcon: Icon(Icons.lock_outline),
        suffixIcon: Semantics(
          label: isVisible ? 'ซ่อนรหัสผ่าน' : 'แสดงรหัสผ่าน',
          button: true,
          child: IconButton(
            icon: Icon(isVisible ? Icons.visibility_off : Icons.visibility),
            onPressed: onToggleVisibility,
          ),
        ),
        border: OutlineInputBorder(borderRadius: BorderRadius.circular(8)),
      ),
      validator: validator,
    );
  }
}
```

---

## สรุป

Accessibility ที่ดีใน Flutter ประกอบด้วย:

1. **Semantics** - ให้ข้อมูลสำหรับ Screen Readers
2. **Screen Reader** - รองรับ TalkBack และ VoiceOver
3. **Color Contrast** - ผ่าน WCAG AA (4.5:1)
4. **Touch Targets** - อย่างน้อย 44x44 pt
5. **Focus Management** - keyboard navigation ที่ถูกต้อง
6. **Live Regions** - แจ้ง Screen Reader เมื่อ content เปลี่ยน
