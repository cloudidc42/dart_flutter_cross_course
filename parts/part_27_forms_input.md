# Part 27: Forms and User Input

## บทนำ

Forms เป็นส่วนสำคัญของแอพเกือบทุกตัว ตั้งแต่ Login จนถึง Registration, Settings และอื่นๆ Flutter มี widget ครอบคลุมทุก input type

---

## TextField

`TextField` คือ widget สำหรับรับข้อความจาก user

```dart
// TextField พื้นฐาน
TextField(
  decoration: const InputDecoration(
    labelText: 'ชื่อ',
    hintText: 'กรอกชื่อของคุณ',
    prefixIcon: Icon(Icons.person),
    border: OutlineInputBorder(),
  ),
  onChanged: (value) {
    print('Text changed: $value');
  },
)

// TextField พร้อม controller
final _controller = TextEditingController();

TextField(
  controller: _controller,
  decoration: const InputDecoration(
    labelText: 'อีเมล',
    prefixIcon: Icon(Icons.email),
    border: OutlineInputBorder(),
  ),
)

// อ่านค่า
print(_controller.text);

// ตั้งค่า
_controller.text = 'new value';

// Clear
_controller.clear();

// เลือกข้อความทั้งหมด
_controller.selection = TextSelection(
  baseOffset: 0,
  extentOffset: _controller.text.length,
);

// ต้อง dispose controller
@override
void dispose() {
  _controller.dispose();
  super.dispose();
}
```

### InputDecoration ครบถ้วน

```dart
TextField(
  decoration: InputDecoration(
    // Label
    labelText: 'รหัสผ่าน',
    labelStyle: const TextStyle(color: Colors.blue),
    floatingLabelStyle: const TextStyle(color: Colors.blue),
    floatingLabelBehavior: FloatingLabelBehavior.always,
    
    // Hint
    hintText: 'กรอกรหัสผ่านอย่างน้อย 8 ตัว',
    hintStyle: const TextStyle(color: Colors.grey),
    
    // Helper
    helperText: 'ใช้ตัวอักษรและตัวเลขผสมกัน',
    helperStyle: const TextStyle(fontSize: 12),
    
    // Error (แสดงเมื่อ validate fail)
    errorText: null, // ถ้าเป็น null จะไม่แสดง error
    errorStyle: const TextStyle(color: Colors.red),
    errorMaxLines: 2,
    
    // Counter
    counterText: '0/50',
    
    // Icons
    prefixIcon: const Icon(Icons.lock),
    suffixIcon: IconButton(
      icon: const Icon(Icons.visibility),
      onPressed: () {},
    ),
    prefix: const Text('฿'),
    suffix: const Text('.00'),
    prefixText: '+66',
    
    // Border styles
    border: OutlineInputBorder(
      borderRadius: BorderRadius.circular(12),
    ),
    enabledBorder: OutlineInputBorder(
      borderRadius: BorderRadius.circular(12),
      borderSide: const BorderSide(color: Colors.grey),
    ),
    focusedBorder: OutlineInputBorder(
      borderRadius: BorderRadius.circular(12),
      borderSide: const BorderSide(color: Colors.blue, width: 2),
    ),
    errorBorder: OutlineInputBorder(
      borderRadius: BorderRadius.circular(12),
      borderSide: const BorderSide(color: Colors.red),
    ),
    
    // Fill
    filled: true,
    fillColor: Colors.grey[100],
    
    // Content padding
    contentPadding: const EdgeInsets.symmetric(horizontal: 16, vertical: 16),
    
    // Dense
    isDense: true,
  ),
)
```

### Keyboard Types

```dart
// ประเภท keyboard
TextField(
  keyboardType: TextInputType.text,           // ข้อความทั่วไป (default)
  keyboardType: TextInputType.number,         // ตัวเลข
  keyboardType: TextInputType.phone,          // เบอร์โทร
  keyboardType: TextInputType.emailAddress,   // อีเมล
  keyboardType: TextInputType.url,            // URL
  keyboardType: TextInputType.datetime,       // วันที่/เวลา
  keyboardType: TextInputType.multiline,      // หลายบรรทัด
  keyboardType: const TextInputType.numberWithOptions(
    signed: true,    // รับค่าลบ
    decimal: true,   // รับทศนิยม
  ),
)

// Action button บน keyboard
TextField(
  textInputAction: TextInputAction.next,      // ปุ่ม "ถัดไป"
  textInputAction: TextInputAction.done,      // ปุ่ม "เสร็จ"
  textInputAction: TextInputAction.search,    // ปุ่ม "ค้นหา"
  textInputAction: TextInputAction.go,        // ปุ่ม "ไป"
  textInputAction: TextInputAction.send,      // ปุ่ม "ส่ง"
  onSubmitted: (value) {
    print('Submitted: $value');
  },
)
```

### Password Field

```dart
class PasswordField extends StatefulWidget {
  final TextEditingController controller;
  final String? labelText;
  
  const PasswordField({
    super.key,
    required this.controller,
    this.labelText,
  });

  @override
  State<PasswordField> createState() => _PasswordFieldState();
}

class _PasswordFieldState extends State<PasswordField> {
  bool _obscureText = true;

  @override
  Widget build(BuildContext context) {
    return TextField(
      controller: widget.controller,
      obscureText: _obscureText,
      decoration: InputDecoration(
        labelText: widget.labelText ?? 'รหัสผ่าน',
        prefixIcon: const Icon(Icons.lock_outline),
        suffixIcon: IconButton(
          icon: Icon(_obscureText ? Icons.visibility : Icons.visibility_off),
          onPressed: () => setState(() => _obscureText = !_obscureText),
        ),
        border: const OutlineInputBorder(),
      ),
    );
  }
}
```

---

## TextFormField และ Form

`TextFormField` เหมาะสำหรับใช้ใน `Form` widget เพราะมี validation built-in

```dart
// Form ต้องการ GlobalKey<FormState>
final _formKey = GlobalKey<FormState>();

Form(
  key: _formKey,
  child: Column(
    children: [
      TextFormField(
        decoration: const InputDecoration(
          labelText: 'อีเมล *',
          border: OutlineInputBorder(),
        ),
        keyboardType: TextInputType.emailAddress,
        // Validator
        validator: (value) {
          if (value == null || value.trim().isEmpty) {
            return 'กรุณากรอกอีเมล';
          }
          if (!value.contains('@')) {
            return 'อีเมลไม่ถูกต้อง';
          }
          return null; // null = ผ่าน validation
        },
        onSaved: (value) {
          // เรียกเมื่อ form.save()
          print('Email saved: $value');
        },
      ),
      
      ElevatedButton(
        onPressed: () {
          // Validate
          if (_formKey.currentState!.validate()) {
            // ผ่าน validation
            _formKey.currentState!.save();
            print('Form submitted!');
          }
        },
        child: const Text('ส่ง'),
      ),
    ],
  ),
)
```

---

## Validators

```dart
// Validators library
class Validators {
  static String? required(String? value, [String? fieldName]) {
    if (value == null || value.trim().isEmpty) {
      return 'กรุณากรอก${fieldName ?? 'ข้อมูล'}';
    }
    return null;
  }
  
  static String? email(String? value) {
    if (value == null || value.isEmpty) return null;
    final emailRegex = RegExp(r'^[^\s@]+@[^\s@]+\.[^\s@]+$');
    if (!emailRegex.hasMatch(value)) {
      return 'อีเมลไม่ถูกต้อง';
    }
    return null;
  }
  
  static String? phone(String? value) {
    if (value == null || value.isEmpty) return null;
    final phoneRegex = RegExp(r'^0[0-9]{9}$');
    if (!phoneRegex.hasMatch(value.replaceAll('-', ''))) {
      return 'เบอร์โทรไม่ถูกต้อง (ต้องการ 10 หลัก)';
    }
    return null;
  }
  
  static String? minLength(String? value, int min) {
    if (value == null || value.isEmpty) return null;
    if (value.length < min) {
      return 'ต้องมีอย่างน้อย $min ตัวอักษร';
    }
    return null;
  }
  
  static String? maxLength(String? value, int max) {
    if (value == null || value.isEmpty) return null;
    if (value.length > max) {
      return 'ต้องไม่เกิน $max ตัวอักษร';
    }
    return null;
  }
  
  static String? password(String? value) {
    if (value == null || value.isEmpty) return 'กรุณากรอกรหัสผ่าน';
    if (value.length < 8) return 'รหัสผ่านต้องมีอย่างน้อย 8 ตัว';
    if (!value.contains(RegExp(r'[A-Za-z]'))) {
      return 'ต้องมีตัวอักษร';
    }
    if (!value.contains(RegExp(r'[0-9]'))) {
      return 'ต้องมีตัวเลข';
    }
    return null;
  }
  
  // Combine multiple validators
  static String? Function(String?) combine(
      List<String? Function(String?)> validators) {
    return (value) {
      for (final validator in validators) {
        final result = validator(value);
        if (result != null) return result;
      }
      return null;
    };
  }
}

// ใช้งาน
TextFormField(
  validator: Validators.combine([
    (v) => Validators.required(v, 'อีเมล'),
    Validators.email,
  ]),
)
```

---

## Focus และ FocusNode

FocusNode ควบคุม keyboard focus

```dart
class FocusDemo extends StatefulWidget {
  const FocusDemo({super.key});

  @override
  State<FocusDemo> createState() => _FocusDemoState();
}

class _FocusDemoState extends State<FocusDemo> {
  final _nameFocus = FocusNode();
  final _emailFocus = FocusNode();
  final _passwordFocus = FocusNode();
  
  @override
  void initState() {
    super.initState();
    // Listen to focus changes
    _nameFocus.addListener(() {
      if (_nameFocus.hasFocus) {
        print('Name field focused');
      }
    });
  }
  
  @override
  void dispose() {
    _nameFocus.dispose();
    _emailFocus.dispose();
    _passwordFocus.dispose();
    super.dispose();
  }

  void _fieldFocusChange(BuildContext context, FocusNode current, FocusNode next) {
    current.unfocus();
    FocusScope.of(context).requestFocus(next);
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        TextFormField(
          focusNode: _nameFocus,
          decoration: const InputDecoration(labelText: 'ชื่อ'),
          textInputAction: TextInputAction.next,
          onFieldSubmitted: (_) => _fieldFocusChange(
            context, _nameFocus, _emailFocus,
          ),
        ),
        
        TextFormField(
          focusNode: _emailFocus,
          decoration: const InputDecoration(labelText: 'อีเมล'),
          keyboardType: TextInputType.emailAddress,
          textInputAction: TextInputAction.next,
          onFieldSubmitted: (_) => _fieldFocusChange(
            context, _emailFocus, _passwordFocus,
          ),
        ),
        
        TextFormField(
          focusNode: _passwordFocus,
          decoration: const InputDecoration(labelText: 'รหัสผ่าน'),
          obscureText: true,
          textInputAction: TextInputAction.done,
          onFieldSubmitted: (_) {
            _passwordFocus.unfocus();
            // Submit form
          },
        ),
        
        // Focus/unfocus programmatically
        Row(
          children: [
            ElevatedButton(
              onPressed: () => FocusScope.of(context).requestFocus(_nameFocus),
              child: const Text('Focus ชื่อ'),
            ),
            ElevatedButton(
              onPressed: () => FocusManager.instance.primaryFocus?.unfocus(),
              child: const Text('ปิด keyboard'),
            ),
          ],
        ),
      ],
    );
  }
}
```

---

## Checkbox, Switch, Radio

### Checkbox

```dart
bool _isChecked = false;

Checkbox(
  value: _isChecked,
  onChanged: (value) => setState(() => _isChecked = value ?? false),
)

// CheckboxListTile
CheckboxListTile(
  title: const Text('ยอมรับข้อกำหนดและเงื่อนไข'),
  subtitle: const Text('กรุณาอ่านก่อนยอมรับ'),
  value: _isChecked,
  onChanged: (value) => setState(() => _isChecked = value ?? false),
  controlAffinity: ListTileControlAffinity.leading, // checkbox ซ้าย
)
```

### Switch

```dart
bool _isSwitched = false;

Switch(
  value: _isSwitched,
  onChanged: (value) => setState(() => _isSwitched = value),
)

// SwitchListTile
SwitchListTile(
  title: const Text('รับการแจ้งเตือน'),
  subtitle: const Text('รับข่าวสารและโปรโมชั่น'),
  value: _isSwitched,
  onChanged: (value) => setState(() => _isSwitched = value),
  secondary: const Icon(Icons.notifications),
)
```

### Radio

```dart
String _selectedOption = 'a';

Column(
  children: [
    RadioListTile<String>(
      title: const Text('ตัวเลือก A'),
      value: 'a',
      groupValue: _selectedOption,
      onChanged: (value) => setState(() => _selectedOption = value!),
    ),
    RadioListTile<String>(
      title: const Text('ตัวเลือก B'),
      value: 'b',
      groupValue: _selectedOption,
      onChanged: (value) => setState(() => _selectedOption = value!),
    ),
    RadioListTile<String>(
      title: const Text('ตัวเลือก C'),
      value: 'c',
      groupValue: _selectedOption,
      onChanged: (value) => setState(() => _selectedOption = value!),
    ),
  ],
)
```

---

## Slider

```dart
double _sliderValue = 50;

Slider(
  value: _sliderValue,
  min: 0,
  max: 100,
  divisions: 10,   // แบ่ง 10 ช่อง
  label: _sliderValue.round().toString(),
  onChanged: (value) => setState(() => _sliderValue = value),
)

// RangeSlider
RangeValues _priceRange = const RangeValues(100, 500);

RangeSlider(
  values: _priceRange,
  min: 0,
  max: 1000,
  divisions: 20,
  labels: RangeLabels(
    '฿${_priceRange.start.toInt()}',
    '฿${_priceRange.end.toInt()}',
  ),
  onChanged: (values) => setState(() => _priceRange = values),
)
```

---

## DatePicker และ TimePicker

```dart
// Date Picker
DateTime? _selectedDate;

Future<void> _selectDate() async {
  final date = await showDatePicker(
    context: context,
    initialDate: _selectedDate ?? DateTime.now(),
    firstDate: DateTime(2020),
    lastDate: DateTime(2030),
    builder: (context, child) {
      return Theme(
        data: Theme.of(context).copyWith(
          colorScheme: ColorScheme.light(
            primary: Theme.of(context).colorScheme.primary,
          ),
        ),
        child: child!,
      );
    },
  );
  
  if (date != null) {
    setState(() => _selectedDate = date);
  }
}

// Time Picker
TimeOfDay? _selectedTime;

Future<void> _selectTime() async {
  final time = await showTimePicker(
    context: context,
    initialTime: _selectedTime ?? TimeOfDay.now(),
  );
  
  if (time != null) {
    setState(() => _selectedTime = time);
  }
}

// Date Range Picker
DateTimeRange? _dateRange;

Future<void> _selectDateRange() async {
  final range = await showDateRangePicker(
    context: context,
    firstDate: DateTime(2020),
    lastDate: DateTime(2030),
    initialDateRange: _dateRange,
  );
  
  if (range != null) {
    setState(() => _dateRange = range);
  }
}
```

---

## Workshop: Registration Form with Validation

### lib/screens/register_screen.dart

```dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

class RegistrationData {
  final String firstName;
  final String lastName;
  final String email;
  final String phone;
  final String password;
  final String birthDate;
  final String gender;
  final bool agreedToTerms;

  const RegistrationData({
    required this.firstName,
    required this.lastName,
    required this.email,
    required this.phone,
    required this.password,
    required this.birthDate,
    required this.gender,
    required this.agreedToTerms,
  });
}

class RegisterScreen extends StatefulWidget {
  const RegisterScreen({super.key});

  @override
  State<RegisterScreen> createState() => _RegisterScreenState();
}

class _RegisterScreenState extends State<RegisterScreen> {
  final _formKey = GlobalKey<FormState>();
  final _pageController = PageController();
  int _currentPage = 0;
  bool _isLoading = false;

  // Controllers
  final _firstNameController = TextEditingController();
  final _lastNameController = TextEditingController();
  final _emailController = TextEditingController();
  final _phoneController = TextEditingController();
  final _passwordController = TextEditingController();
  final _confirmPasswordController = TextEditingController();

  // FocusNodes
  final _lastNameFocus = FocusNode();
  final _emailFocus = FocusNode();
  final _phoneFocus = FocusNode();
  final _passwordFocus = FocusNode();
  final _confirmFocus = FocusNode();

  // State
  String _gender = 'male';
  DateTime? _birthDate;
  bool _agreedToTerms = false;
  bool _showPassword = false;
  bool _showConfirmPassword = false;

  @override
  void dispose() {
    _firstNameController.dispose();
    _lastNameController.dispose();
    _emailController.dispose();
    _phoneController.dispose();
    _passwordController.dispose();
    _confirmPasswordController.dispose();
    _lastNameFocus.dispose();
    _emailFocus.dispose();
    _phoneFocus.dispose();
    _passwordFocus.dispose();
    _confirmFocus.dispose();
    _pageController.dispose();
    super.dispose();
  }

  Future<void> _selectBirthDate() async {
    final date = await showDatePicker(
      context: context,
      initialDate: _birthDate ?? DateTime(2000),
      firstDate: DateTime(1950),
      lastDate: DateTime.now().subtract(const Duration(days: 365 * 13)),
    );
    if (date != null) setState(() => _birthDate = date);
  }

  void _nextPage() {
    if (_currentPage < 2) {
      _pageController.nextPage(
        duration: const Duration(milliseconds: 300),
        curve: Curves.easeInOut,
      );
    }
  }

  void _previousPage() {
    if (_currentPage > 0) {
      _pageController.previousPage(
        duration: const Duration(milliseconds: 300),
        curve: Curves.easeInOut,
      );
    }
  }

  Future<void> _submit() async {
    if (!_formKey.currentState!.validate()) return;
    if (!_agreedToTerms) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('กรุณายอมรับข้อกำหนดก่อน')),
      );
      return;
    }

    setState(() => _isLoading = true);

    // Simulate API call
    await Future.delayed(const Duration(seconds: 2));

    if (mounted) {
      setState(() => _isLoading = false);
      showDialog(
        context: context,
        builder: (context) => AlertDialog(
          icon: const Icon(Icons.check_circle, color: Colors.green, size: 64),
          title: const Text('สมัครสำเร็จ!'),
          content: Text(
            'ยินดีต้อนรับ ${_firstNameController.text} ${_lastNameController.text}!\n'
            'กรุณาตรวจสอบอีเมลเพื่อยืนยันบัญชี',
          ),
          actions: [
            ElevatedButton(
              onPressed: () {
                Navigator.of(context).pop();
                Navigator.of(context).pop(); // กลับหน้า login
              },
              child: const Text('เข้าสู่ระบบ'),
            ),
          ],
        ),
      );
    }
  }

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);

    return Scaffold(
      appBar: AppBar(
        title: const Text('สมัครสมาชิก'),
        leading: _currentPage > 0
            ? IconButton(
                icon: const Icon(Icons.arrow_back),
                onPressed: _previousPage,
              )
            : null,
      ),
      body: Form(
        key: _formKey,
        child: Column(
          children: [
            // Progress indicator
            _buildProgressIndicator(theme),

            // Pages
            Expanded(
              child: PageView(
                controller: _pageController,
                physics: const NeverScrollableScrollPhysics(),
                onPageChanged: (page) => setState(() => _currentPage = page),
                children: [
                  _buildPersonalInfoPage(theme),
                  _buildAccountPage(theme),
                  _buildConfirmPage(theme),
                ],
              ),
            ),

            // Navigation buttons
            _buildNavigationButtons(theme),
          ],
        ),
      ),
    );
  }

  Widget _buildProgressIndicator(ThemeData theme) {
    return Padding(
      padding: const EdgeInsets.all(16),
      child: Column(
        children: [
          Row(
            children: List.generate(3, (index) {
              final isCompleted = index < _currentPage;
              final isCurrent = index == _currentPage;
              return Expanded(
                child: Row(
                  children: [
                    CircleAvatar(
                      radius: 16,
                      backgroundColor: isCompleted || isCurrent
                          ? theme.colorScheme.primary
                          : theme.colorScheme.surfaceVariant,
                      child: isCompleted
                          ? const Icon(Icons.check, size: 16, color: Colors.white)
                          : Text(
                              '${index + 1}',
                              style: TextStyle(
                                color: isCurrent
                                    ? theme.colorScheme.onPrimary
                                    : theme.colorScheme.onSurfaceVariant,
                                fontSize: 12,
                                fontWeight: FontWeight.bold,
                              ),
                            ),
                    ),
                    if (index < 2)
                      Expanded(
                        child: Container(
                          height: 2,
                          color: index < _currentPage
                              ? theme.colorScheme.primary
                              : theme.colorScheme.surfaceVariant,
                        ),
                      ),
                  ],
                ),
              );
            }),
          ),
          const SizedBox(height: 8),
          Row(
            mainAxisAlignment: MainAxisAlignment.spaceBetween,
            children: const [
              Text('ข้อมูลส่วนตัว', style: TextStyle(fontSize: 11)),
              Text('บัญชี', style: TextStyle(fontSize: 11)),
              Text('ยืนยัน', style: TextStyle(fontSize: 11)),
            ],
          ),
        ],
      ),
    );
  }

  Widget _buildPersonalInfoPage(ThemeData theme) {
    return SingleChildScrollView(
      padding: const EdgeInsets.all(24),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text('ข้อมูลส่วนตัว', style: theme.textTheme.headlineSmall),
          const SizedBox(height: 8),
          Text('กรอกข้อมูลพื้นฐานของคุณ',
            style: TextStyle(color: theme.colorScheme.onSurfaceVariant)),
          const SizedBox(height: 24),

          // First Name
          TextFormField(
            controller: _firstNameController,
            decoration: const InputDecoration(
              labelText: 'ชื่อ *',
              prefixIcon: Icon(Icons.person_outline),
              border: OutlineInputBorder(),
            ),
            textInputAction: TextInputAction.next,
            onFieldSubmitted: (_) =>
                FocusScope.of(context).requestFocus(_lastNameFocus),
            validator: (v) =>
                v?.trim().isEmpty == true ? 'กรุณากรอกชื่อ' : null,
          ),
          const SizedBox(height: 16),

          // Last Name
          TextFormField(
            controller: _lastNameController,
            focusNode: _lastNameFocus,
            decoration: const InputDecoration(
              labelText: 'นามสกุล *',
              prefixIcon: Icon(Icons.person_outline),
              border: OutlineInputBorder(),
            ),
            textInputAction: TextInputAction.next,
            onFieldSubmitted: (_) =>
                FocusScope.of(context).requestFocus(_emailFocus),
            validator: (v) =>
                v?.trim().isEmpty == true ? 'กรุณากรอกนามสกุล' : null,
          ),
          const SizedBox(height: 16),

          // Birth Date
          InkWell(
            onTap: _selectBirthDate,
            child: InputDecorator(
              decoration: InputDecoration(
                labelText: 'วันเกิด',
                prefixIcon: const Icon(Icons.calendar_today),
                border: const OutlineInputBorder(),
                suffixIcon: _birthDate != null
                    ? IconButton(
                        icon: const Icon(Icons.clear),
                        onPressed: () => setState(() => _birthDate = null),
                      )
                    : null,
              ),
              child: Text(
                _birthDate != null
                    ? '${_birthDate!.day}/${_birthDate!.month}/${_birthDate!.year}'
                    : 'เลือกวันเกิด',
                style: TextStyle(
                  color: _birthDate != null
                      ? theme.colorScheme.onSurface
                      : theme.colorScheme.onSurfaceVariant,
                ),
              ),
            ),
          ),
          const SizedBox(height: 16),

          // Gender
          Text('เพศ', style: theme.textTheme.labelLarge),
          const SizedBox(height: 8),
          Row(
            children: [
              Expanded(
                child: RadioListTile<String>(
                  title: const Text('ชาย'),
                  value: 'male',
                  groupValue: _gender,
                  onChanged: (v) => setState(() => _gender = v!),
                  contentPadding: EdgeInsets.zero,
                ),
              ),
              Expanded(
                child: RadioListTile<String>(
                  title: const Text('หญิง'),
                  value: 'female',
                  groupValue: _gender,
                  onChanged: (v) => setState(() => _gender = v!),
                  contentPadding: EdgeInsets.zero,
                ),
              ),
              Expanded(
                child: RadioListTile<String>(
                  title: const Text('อื่นๆ'),
                  value: 'other',
                  groupValue: _gender,
                  onChanged: (v) => setState(() => _gender = v!),
                  contentPadding: EdgeInsets.zero,
                ),
              ),
            ],
          ),
        ],
      ),
    );
  }

  Widget _buildAccountPage(ThemeData theme) {
    return SingleChildScrollView(
      padding: const EdgeInsets.all(24),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text('ข้อมูลบัญชี', style: theme.textTheme.headlineSmall),
          const SizedBox(height: 8),
          Text('สร้างบัญชีเข้าระบบ',
            style: TextStyle(color: theme.colorScheme.onSurfaceVariant)),
          const SizedBox(height: 24),

          // Email
          TextFormField(
            controller: _emailController,
            focusNode: _emailFocus,
            decoration: const InputDecoration(
              labelText: 'อีเมล *',
              prefixIcon: Icon(Icons.email_outlined),
              border: OutlineInputBorder(),
              hintText: 'example@email.com',
            ),
            keyboardType: TextInputType.emailAddress,
            textInputAction: TextInputAction.next,
            onFieldSubmitted: (_) =>
                FocusScope.of(context).requestFocus(_phoneFocus),
            validator: (v) {
              if (v?.trim().isEmpty == true) return 'กรุณากรอกอีเมล';
              if (!RegExp(r'^[^\s@]+@[^\s@]+\.[^\s@]+$').hasMatch(v!)) {
                return 'อีเมลไม่ถูกต้อง';
              }
              return null;
            },
          ),
          const SizedBox(height: 16),

          // Phone
          TextFormField(
            controller: _phoneController,
            focusNode: _phoneFocus,
            decoration: const InputDecoration(
              labelText: 'เบอร์โทร',
              prefixIcon: Icon(Icons.phone_outlined),
              prefixText: '+66 ',
              border: OutlineInputBorder(),
              hintText: '081-234-5678',
            ),
            keyboardType: TextInputType.phone,
            inputFormatters: [
              FilteringTextInputFormatter.digitsOnly,
              LengthLimitingTextInputFormatter(10),
            ],
            textInputAction: TextInputAction.next,
            onFieldSubmitted: (_) =>
                FocusScope.of(context).requestFocus(_passwordFocus),
          ),
          const SizedBox(height: 16),

          // Password
          TextFormField(
            controller: _passwordController,
            focusNode: _passwordFocus,
            decoration: InputDecoration(
              labelText: 'รหัสผ่าน *',
              prefixIcon: const Icon(Icons.lock_outline),
              border: const OutlineInputBorder(),
              helperText: 'อย่างน้อย 8 ตัว ผสมตัวอักษรและตัวเลข',
              suffixIcon: IconButton(
                icon: Icon(_showPassword
                    ? Icons.visibility_off
                    : Icons.visibility),
                onPressed: () =>
                    setState(() => _showPassword = !_showPassword),
              ),
            ),
            obscureText: !_showPassword,
            textInputAction: TextInputAction.next,
            onFieldSubmitted: (_) =>
                FocusScope.of(context).requestFocus(_confirmFocus),
            validator: (v) {
              if (v == null || v.isEmpty) return 'กรุณากรอกรหัสผ่าน';
              if (v.length < 8) return 'ต้องมีอย่างน้อย 8 ตัว';
              if (!v.contains(RegExp(r'[A-Za-z]'))) return 'ต้องมีตัวอักษร';
              if (!v.contains(RegExp(r'[0-9]'))) return 'ต้องมีตัวเลข';
              return null;
            },
          ),
          const SizedBox(height: 16),

          // Confirm Password
          TextFormField(
            controller: _confirmPasswordController,
            focusNode: _confirmFocus,
            decoration: InputDecoration(
              labelText: 'ยืนยันรหัสผ่าน *',
              prefixIcon: const Icon(Icons.lock_outline),
              border: const OutlineInputBorder(),
              suffixIcon: IconButton(
                icon: Icon(_showConfirmPassword
                    ? Icons.visibility_off
                    : Icons.visibility),
                onPressed: () => setState(
                    () => _showConfirmPassword = !_showConfirmPassword),
              ),
            ),
            obscureText: !_showConfirmPassword,
            textInputAction: TextInputAction.done,
            validator: (v) {
              if (v != _passwordController.text) {
                return 'รหัสผ่านไม่ตรงกัน';
              }
              return null;
            },
          ),
        ],
      ),
    );
  }

  Widget _buildConfirmPage(ThemeData theme) {
    return SingleChildScrollView(
      padding: const EdgeInsets.all(24),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text('ยืนยันข้อมูล', style: theme.textTheme.headlineSmall),
          const SizedBox(height: 8),
          Text('ตรวจสอบข้อมูลก่อนสมัคร',
            style: TextStyle(color: theme.colorScheme.onSurfaceVariant)),
          const SizedBox(height: 24),

          // Summary card
          Card(
            child: Padding(
              padding: const EdgeInsets.all(16),
              child: Column(
                children: [
                  _summaryRow('ชื่อ-นามสกุล',
                    '${_firstNameController.text} ${_lastNameController.text}'),
                  _summaryRow('อีเมล', _emailController.text),
                  _summaryRow('เบอร์โทร',
                    _phoneController.text.isEmpty ? '-' : _phoneController.text),
                  _summaryRow('เพศ',
                    {'male': 'ชาย', 'female': 'หญิง', 'other': 'อื่นๆ'}[_gender] ?? ''),
                  _summaryRow('วันเกิด',
                    _birthDate != null
                        ? '${_birthDate!.day}/${_birthDate!.month}/${_birthDate!.year}'
                        : 'ไม่ระบุ'),
                ],
              ),
            ),
          ),

          const SizedBox(height: 16),

          // Terms
          CheckboxListTile(
            title: const Text('ฉันยอมรับ'),
            subtitle: RichText(
              text: TextSpan(
                style: DefaultTextStyle.of(context).style,
                children: [
                  const TextSpan(text: 'ยอมรับ '),
                  TextSpan(
                    text: 'ข้อกำหนดและเงื่อนไข',
                    style: TextStyle(
                      color: theme.colorScheme.primary,
                      decoration: TextDecoration.underline,
                    ),
                  ),
                  const TextSpan(text: ' และ '),
                  TextSpan(
                    text: 'นโยบายความเป็นส่วนตัว',
                    style: TextStyle(
                      color: theme.colorScheme.primary,
                      decoration: TextDecoration.underline,
                    ),
                  ),
                ],
              ),
            ),
            value: _agreedToTerms,
            onChanged: (v) => setState(() => _agreedToTerms = v ?? false),
            controlAffinity: ListTileControlAffinity.leading,
            contentPadding: EdgeInsets.zero,
          ),
        ],
      ),
    );
  }

  Widget _summaryRow(String label, String value) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 6),
      child: Row(
        children: [
          SizedBox(
            width: 120,
            child: Text(label,
              style: const TextStyle(color: Colors.grey)),
          ),
          Expanded(
            child: Text(value,
              style: const TextStyle(fontWeight: FontWeight.w500)),
          ),
        ],
      ),
    );
  }

  Widget _buildNavigationButtons(ThemeData theme) {
    return Padding(
      padding: const EdgeInsets.all(16),
      child: Row(
        children: [
          if (_currentPage > 0) ...[
            OutlinedButton(
              onPressed: _previousPage,
              child: const Text('ย้อนกลับ'),
            ),
            const SizedBox(width: 16),
          ],
          Expanded(
            child: _currentPage < 2
                ? ElevatedButton(
                    onPressed: _nextPage,
                    child: const Text('ถัดไป'),
                  )
                : ElevatedButton(
                    onPressed: _isLoading ? null : _submit,
                    child: _isLoading
                        ? const SizedBox(
                            width: 24,
                            height: 24,
                            child: CircularProgressIndicator(
                              strokeWidth: 2,
                              color: Colors.white,
                            ),
                          )
                        : const Text('สมัครสมาชิก'),
                  ),
          ),
        ],
      ),
    );
  }
}
```

---

## สรุปบทที่ 27

ในบทนี้เราได้เรียนรู้:

1. **TextField**: controllers, keyboard types, InputDecoration
2. **Form/TextFormField**: validation, save
3. **Validators**: การสร้าง validator ที่ reusable
4. **FocusNode**: การควบคุม keyboard focus
5. **Checkbox, Switch, Radio**: การรับ binary/choice input
6. **Slider**: การรับค่าตัวเลข
7. **DatePicker**: การเลือกวันที่
8. **Workshop**: Registration form หลายขั้นตอน

บทต่อไปเราจะเรียน Lists และ Grids
