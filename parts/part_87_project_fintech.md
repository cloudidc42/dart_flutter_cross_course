# Part 87: Real-World Project: Fintech App - โปรเจกต์จริง: แอป Fintech

## บทนำ

Fintech App มีความต้องการพิเศษด้านความปลอดภัย UX และประสิทธิภาพ ส่วนนี้จะสร้างแอป banking ที่มีฟีเจอร์ครบครัน พร้อม biometric authentication และ security features

## 1. Account Overview

### Account Model

```dart
// lib/features/accounts/domain/entities/account.dart
import 'package:equatable/equatable.dart';

class Account extends Equatable {
  final String id;
  final String accountNumber;
  final String accountName;
  final AccountType type;
  final String currency;
  final double balance;
  final double availableBalance;
  final bool isDefault;
  final String? cardLastFour;

  const Account({
    required this.id,
    required this.accountNumber,
    required this.accountName,
    required this.type,
    this.currency = 'THB',
    required this.balance,
    required this.availableBalance,
    this.isDefault = false,
    this.cardLastFour,
  });

  String get maskedAccountNumber {
    if (accountNumber.length <= 4) return accountNumber;
    return '****${accountNumber.substring(accountNumber.length - 4)}';
  }

  String get formattedBalance =>
      '฿${balance.toStringAsFixed(2).replaceAllMapped(
            RegExp(r'(\d)(?=(\d{3})+(?!\d))'),
            (match) => '${match[1]},',
          )}';

  @override
  List<Object?> get props => [id, balance];
}

enum AccountType { savings, current, credit, investment }
```

### Account Overview Screen

```dart
// lib/features/dashboard/presentation/screens/dashboard_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

class DashboardScreen extends StatelessWidget {
  const DashboardScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Theme.of(context).colorScheme.background,
      body: SafeArea(
        child: CustomScrollView(
          slivers: [
            SliverToBoxAdapter(
              child: _buildHeader(context),
            ),
            SliverToBoxAdapter(
              child: _buildAccountCard(context),
            ),
            SliverToBoxAdapter(
              child: _buildQuickActions(context),
            ),
            SliverToBoxAdapter(
              child: _buildRecentTransactions(context),
            ),
          ],
        ),
      ),
    );
  }

  Widget _buildHeader(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.all(20),
      child: Row(
        children: [
          CircleAvatar(
            radius: 24,
            backgroundImage: const NetworkImage('https://example.com/avatar.jpg'),
          ),
          const SizedBox(width: 12),
          Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              const Text(
                'สวัสดี,',
                style: TextStyle(color: Colors.grey),
              ),
              Text(
                'คุณสมชาย',
                style: Theme.of(context).textTheme.titleMedium?.copyWith(
                      fontWeight: FontWeight.bold,
                    ),
              ),
            ],
          ),
          const Spacer(),
          IconButton(
            icon: const Icon(Icons.notifications_outlined),
            onPressed: () => context.push('/notifications'),
          ),
        ],
      ),
    );
  }

  Widget _buildAccountCard(BuildContext context) {
    return Container(
      margin: const EdgeInsets.symmetric(horizontal: 20),
      padding: const EdgeInsets.all(24),
      decoration: BoxDecoration(
        gradient: const LinearGradient(
          begin: Alignment.topLeft,
          end: Alignment.bottomRight,
          colors: [Color(0xFF1E3A8A), Color(0xFF3B82F6)],
        ),
        borderRadius: BorderRadius.circular(20),
        boxShadow: [
          BoxShadow(
            color: Colors.blue.withOpacity(0.3),
            blurRadius: 15,
            offset: const Offset(0, 8),
          ),
        ],
      ),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Row(
            mainAxisAlignment: MainAxisAlignment.spaceBetween,
            children: [
              const Text(
                'บัญชีออมทรัพย์',
                style: TextStyle(color: Colors.white70, fontSize: 14),
              ),
              _buildBalanceToggle(),
            ],
          ),
          const SizedBox(height: 8),
          BlocBuilder<AccountBloc, AccountState>(
            builder: (context, state) {
              if (state is AccountLoaded) {
                return _buildBalanceDisplay(state.account);
              }
              return const Text(
                '฿---',
                style: TextStyle(
                  color: Colors.white,
                  fontSize: 32,
                  fontWeight: FontWeight.bold,
                ),
              );
            },
          ),
          const SizedBox(height: 16),
          Row(
            children: [
              const Icon(Icons.credit_card, color: Colors.white70, size: 16),
              const SizedBox(width: 8),
              BlocBuilder<AccountBloc, AccountState>(
                builder: (context, state) {
                  if (state is AccountLoaded) {
                    return Text(
                      state.account.maskedAccountNumber,
                      style: const TextStyle(
                        color: Colors.white70,
                        letterSpacing: 2,
                      ),
                    );
                  }
                  return const SizedBox();
                },
              ),
            ],
          ),
        ],
      ),
    );
  }

  Widget _buildBalanceToggle() {
    return BlocBuilder<AccountBloc, AccountState>(
      builder: (context, state) {
        return IconButton(
          icon: Icon(
            context.watch<DashboardCubit>().isBalanceVisible
                ? Icons.visibility_outlined
                : Icons.visibility_off_outlined,
            color: Colors.white70,
            size: 20,
          ),
          onPressed: () =>
              context.read<DashboardCubit>().toggleBalanceVisibility(),
        );
      },
    );
  }

  Widget _buildBalanceDisplay(Account account) {
    return BlocBuilder<DashboardCubit, DashboardState>(
      builder: (context, state) {
        return Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              state.isBalanceVisible ? account.formattedBalance : '฿ ••••••',
              style: const TextStyle(
                color: Colors.white,
                fontSize: 32,
                fontWeight: FontWeight.bold,
              ),
            ),
            if (state.isBalanceVisible)
              Text(
                'ยอดใช้ได้: ฿${account.availableBalance.toStringAsFixed(2)}',
                style: const TextStyle(color: Colors.white70, fontSize: 12),
              ),
          ],
        );
      },
    );
  }

  Widget _buildQuickActions(BuildContext context) {
    final actions = [
      _QuickAction(icon: Icons.send, label: 'โอนเงิน', route: '/transfer'),
      _QuickAction(icon: Icons.payment, label: 'ชำระบิล', route: '/pay-bill'),
      _QuickAction(icon: Icons.qr_code_scanner, label: 'สแกน QR', route: '/scan-qr'),
      _QuickAction(icon: Icons.more_horiz, label: 'เพิ่มเติม', route: '/more'),
    ];

    return Padding(
      padding: const EdgeInsets.all(20),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text('บริการด่วน', style: Theme.of(context).textTheme.titleMedium),
          const SizedBox(height: 16),
          Row(
            mainAxisAlignment: MainAxisAlignment.spaceAround,
            children: actions.map(_buildQuickActionItem).toList(),
          ),
        ],
      ),
    );
  }

  Widget _buildQuickActionItem(_QuickAction action) {
    return Builder(
      builder: (context) => InkWell(
        onTap: () => context.push(action.route),
        borderRadius: BorderRadius.circular(12),
        child: Padding(
          padding: const EdgeInsets.all(8),
          child: Column(
            children: [
              Container(
                padding: const EdgeInsets.all(12),
                decoration: BoxDecoration(
                  color: Theme.of(context)
                      .colorScheme
                      .primaryContainer,
                  borderRadius: BorderRadius.circular(12),
                ),
                child: Icon(
                  action.icon,
                  color: Theme.of(context).colorScheme.onPrimaryContainer,
                ),
              ),
              const SizedBox(height: 8),
              Text(action.label, style: const TextStyle(fontSize: 12)),
            ],
          ),
        ),
      ),
    );
  }

  Widget _buildRecentTransactions(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.all(20),
      child: Column(
        children: [
          Row(
            mainAxisAlignment: MainAxisAlignment.spaceBetween,
            children: [
              Text('รายการล่าสุด',
                  style: Theme.of(context).textTheme.titleMedium),
              TextButton(
                onPressed: () => context.push('/transactions'),
                child: const Text('ดูทั้งหมด'),
              ),
            ],
          ),
          BlocBuilder<TransactionBloc, TransactionState>(
            builder: (context, state) {
              if (state is TransactionsLoaded) {
                return Column(
                  children: state.transactions
                      .take(5)
                      .map((t) => TransactionListTile(transaction: t))
                      .toList(),
                );
              }
              return const CircularProgressIndicator();
            },
          ),
        ],
      ),
    );
  }
}

class _QuickAction {
  final IconData icon;
  final String label;
  final String route;

  const _QuickAction({
    required this.icon,
    required this.label,
    required this.route,
  });
}
```

## 2. Transaction History

### Transaction Model

```dart
// lib/features/transactions/domain/entities/transaction.dart
import 'package:equatable/equatable.dart';

class Transaction extends Equatable {
  final String id;
  final String description;
  final double amount;
  final TransactionType type;
  final TransactionStatus status;
  final DateTime date;
  final String? category;
  final String? merchantLogo;
  final String? reference;
  final double? balanceAfter;
  final String? note;

  const Transaction({
    required this.id,
    required this.description,
    required this.amount,
    required this.type,
    required this.status,
    required this.date,
    this.category,
    this.merchantLogo,
    this.reference,
    this.balanceAfter,
    this.note,
  });

  bool get isCredit => type == TransactionType.credit;
  bool get isDebit => type == TransactionType.debit;

  String get formattedAmount {
    final prefix = isCredit ? '+' : '-';
    return '$prefix฿${amount.toStringAsFixed(2).replaceAllMapped(
          RegExp(r'(\d)(?=(\d{3})+(?!\d))'),
          (match) => '${match[1]},',
        )}';
  }

  @override
  List<Object?> get props => [id];
}

enum TransactionType { credit, debit }
enum TransactionStatus { completed, pending, failed, cancelled }
```

## 3. Money Transfer

### Transfer Screen

```dart
// lib/features/transfer/presentation/screens/transfer_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:flutter/services.dart';

class TransferScreen extends StatefulWidget {
  const TransferScreen({super.key});

  @override
  State<TransferScreen> createState() => _TransferScreenState();
}

class _TransferScreenState extends State<TransferScreen> {
  final _formKey = GlobalKey<FormState>();
  final _recipientController = TextEditingController();
  final _amountController = TextEditingController();
  final _noteController = TextEditingController();

  @override
  void dispose() {
    _recipientController.dispose();
    _amountController.dispose();
    _noteController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('โอนเงิน')),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(20),
        child: Form(
          key: _formKey,
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              // Recipient
              Text('ผู้รับ', style: Theme.of(context).textTheme.titleSmall),
              const SizedBox(height: 8),
              TextFormField(
                controller: _recipientController,
                decoration: InputDecoration(
                  hintText: 'เลขบัญชี หรือ เบอร์โทร',
                  border: const OutlineInputBorder(),
                  suffixIcon: IconButton(
                    icon: const Icon(Icons.contacts),
                    onPressed: _selectFromContacts,
                  ),
                ),
                keyboardType: TextInputType.phone,
                inputFormatters: [FilteringTextInputFormatter.digitsOnly],
                validator: (value) {
                  if (value == null || value.isEmpty) {
                    return 'กรุณาระบุผู้รับ';
                  }
                  if (value.length != 10 && value.length != 13) {
                    return 'เลขบัญชีหรือเบอร์โทรไม่ถูกต้อง';
                  }
                  return null;
                },
              ),

              const SizedBox(height: 20),

              // Amount
              Text('จำนวนเงิน', style: Theme.of(context).textTheme.titleSmall),
              const SizedBox(height: 8),
              TextFormField(
                controller: _amountController,
                decoration: const InputDecoration(
                  hintText: '0.00',
                  border: OutlineInputBorder(),
                  prefixText: '฿ ',
                ),
                keyboardType: const TextInputType.numberWithOptions(
                  decimal: true,
                ),
                inputFormatters: [
                  FilteringTextInputFormatter.allow(RegExp(r'^\d+\.?\d{0,2}')),
                ],
                validator: (value) {
                  if (value == null || value.isEmpty) {
                    return 'กรุณาระบุจำนวนเงิน';
                  }
                  final amount = double.tryParse(value);
                  if (amount == null || amount <= 0) {
                    return 'จำนวนเงินไม่ถูกต้อง';
                  }
                  return null;
                },
              ),

              // Quick amounts
              const SizedBox(height: 8),
              Wrap(
                spacing: 8,
                children: [100, 500, 1000, 5000].map(
                  (amount) => ActionChip(
                    label: Text('฿$amount'),
                    onPressed: () =>
                        _amountController.text = amount.toString(),
                  ),
                ).toList(),
              ),

              const SizedBox(height: 20),

              // Note
              Text('หมายเหตุ (ไม่บังคับ)',
                  style: Theme.of(context).textTheme.titleSmall),
              const SizedBox(height: 8),
              TextFormField(
                controller: _noteController,
                decoration: const InputDecoration(
                  hintText: 'ระบุเหตุผล หรือ หมายเหตุ',
                  border: OutlineInputBorder(),
                ),
                maxLength: 50,
              ),

              const SizedBox(height: 32),

              SizedBox(
                width: double.infinity,
                child: FilledButton(
                  onPressed: _proceedTransfer,
                  child: const Text('ถัดไป'),
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }

  void _selectFromContacts() {
    // Pick from contacts
  }

  Future<void> _proceedTransfer() async {
    if (!_formKey.currentState!.validate()) return;

    // Verify recipient
    final recipient = await _verifyRecipient();
    if (recipient == null) return;

    if (!mounted) return;

    // Show confirmation
    final confirmed = await _showConfirmation(recipient);
    if (!confirmed) return;

    if (!mounted) return;

    // Biometric auth
    final authenticated = await _biometricAuth();
    if (!authenticated) return;

    if (!mounted) return;

    // Execute transfer
    context.read<TransferBloc>().add(
      ExecuteTransferEvent(
        recipientId: recipient.id,
        amount: double.parse(_amountController.text),
        note: _noteController.text,
      ),
    );
  }

  Future<RecipientInfo?> _verifyRecipient() async {
    // Verify and return recipient info
    return null; // placeholder
  }

  Future<bool> _showConfirmation(RecipientInfo recipient) async {
    return await showModalBottomSheet<bool>(
      context: context,
      shape: const RoundedRectangleBorder(
        borderRadius: BorderRadius.vertical(top: Radius.circular(20)),
      ),
      builder: (context) => TransferConfirmationSheet(
        recipient: recipient,
        amount: double.parse(_amountController.text),
        note: _noteController.text,
      ),
    ) ?? false;
  }

  Future<bool> _biometricAuth() async {
    final biometric = context.read<BiometricAuthService>();
    return await biometric.authenticate(
      reason: 'ยืนยันการโอนเงิน',
    );
  }
}
```

## 4. Security Features

### PIN Screen

```dart
// lib/features/security/presentation/screens/pin_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

class PinScreen extends StatefulWidget {
  final PinMode mode;
  final Function(String pin) onPinEntered;

  const PinScreen({
    super.key,
    required this.mode,
    required this.onPinEntered,
  });

  @override
  State<PinScreen> createState() => _PinScreenState();
}

enum PinMode { setup, verify, change }

class _PinScreenState extends State<PinScreen>
    with SingleTickerProviderStateMixin {
  String _pin = '';
  late AnimationController _shakeController;
  late Animation<double> _shakeAnimation;

  @override
  void initState() {
    super.initState();
    _shakeController = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 500),
    );
    _shakeAnimation = Tween<double>(begin: 0, end: 10)
        .chain(CurveTween(curve: Curves.elasticIn))
        .animate(_shakeController);
  }

  @override
  void dispose() {
    _shakeController.dispose();
    super.dispose();
  }

  void _addDigit(String digit) {
    if (_pin.length >= 6) return;

    HapticFeedback.selectionClick();
    setState(() => _pin += digit);

    if (_pin.length == 6) {
      widget.onPinEntered(_pin);
    }
  }

  void _deleteDigit() {
    if (_pin.isEmpty) return;
    HapticFeedback.lightImpact();
    setState(() => _pin = _pin.substring(0, _pin.length - 1));
  }

  void shakeError() {
    HapticFeedback.heavyImpact();
    _shakeController.forward(from: 0).then((_) {
      setState(() => _pin = '');
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Theme.of(context).colorScheme.background,
      appBar: AppBar(
        backgroundColor: Colors.transparent,
        elevation: 0,
      ),
      body: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          children: [
            const SizedBox(height: 40),

            // Title
            Text(
              _titleText(),
              style: Theme.of(context).textTheme.headlineSmall?.copyWith(
                    fontWeight: FontWeight.bold,
                  ),
            ),

            const SizedBox(height: 40),

            // PIN dots
            AnimatedBuilder(
              animation: _shakeAnimation,
              builder: (context, child) => Transform.translate(
                offset: Offset(
                  _shakeAnimation.value *
                      (_shakeController.value < 0.5 ? 1 : -1),
                  0,
                ),
                child: child,
              ),
              child: Row(
                mainAxisAlignment: MainAxisAlignment.center,
                children: List.generate(
                  6,
                  (index) => Container(
                    width: 16,
                    height: 16,
                    margin: const EdgeInsets.symmetric(horizontal: 8),
                    decoration: BoxDecoration(
                      shape: BoxShape.circle,
                      color: index < _pin.length
                          ? Theme.of(context).colorScheme.primary
                          : Colors.grey.shade300,
                    ),
                  ),
                ),
              ),
            ),

            const Spacer(),

            // Numpad
            _buildNumPad(),

            const SizedBox(height: 24),
          ],
        ),
      ),
    );
  }

  Widget _buildNumPad() {
    return Column(
      children: [
        for (var row in [
          ['1', '2', '3'],
          ['4', '5', '6'],
          ['7', '8', '9'],
          ['biometric', '0', 'delete'],
        ])
          Row(
            mainAxisAlignment: MainAxisAlignment.spaceEvenly,
            children: row.map((key) => _buildKey(key)).toList(),
          ),
      ],
    );
  }

  Widget _buildKey(String key) {
    if (key == 'delete') {
      return _NumpadKey(
        onTap: _deleteDigit,
        child: const Icon(Icons.backspace_outlined),
      );
    }

    if (key == 'biometric') {
      return _NumpadKey(
        onTap: _useBiometric,
        child: const Icon(Icons.fingerprint),
      );
    }

    return _NumpadKey(
      onTap: () => _addDigit(key),
      child: Text(
        key,
        style: const TextStyle(fontSize: 24, fontWeight: FontWeight.w300),
      ),
    );
  }

  String _titleText() {
    switch (widget.mode) {
      case PinMode.setup:
        return 'ตั้งรหัส PIN';
      case PinMode.verify:
        return 'ใส่รหัส PIN';
      case PinMode.change:
        return 'รหัส PIN ปัจจุบัน';
    }
  }

  void _useBiometric() async {
    final biometric = context.read<BiometricAuthService>();
    final success = await biometric.authenticate();
    if (success) widget.onPinEntered('BIOMETRIC');
  }
}

class _NumpadKey extends StatelessWidget {
  final VoidCallback onTap;
  final Widget child;

  const _NumpadKey({required this.onTap, required this.child});

  @override
  Widget build(BuildContext context) {
    return Material(
      color: Colors.transparent,
      child: InkWell(
        onTap: onTap,
        borderRadius: BorderRadius.circular(40),
        child: Container(
          width: 80,
          height: 80,
          alignment: Alignment.center,
          child: child,
        ),
      ),
    );
  }
}
```

## 5. Biometric Authentication

```dart
// lib/features/security/presentation/screens/biometric_setup_screen.dart
import 'package:flutter/material.dart';
import 'package:local_auth/local_auth.dart';

class BiometricSetupScreen extends StatefulWidget {
  const BiometricSetupScreen({super.key});

  @override
  State<BiometricSetupScreen> createState() => _BiometricSetupScreenState();
}

class _BiometricSetupScreenState extends State<BiometricSetupScreen> {
  final _localAuth = LocalAuthentication();
  List<BiometricType> _availableBiometrics = [];

  @override
  void initState() {
    super.initState();
    _checkBiometrics();
  }

  Future<void> _checkBiometrics() async {
    final biometrics = await _localAuth.getAvailableBiometrics();
    setState(() => _availableBiometrics = biometrics);
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('ตั้งค่า Biometric')),
      body: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.center,
          children: [
            const SizedBox(height: 40),
            Icon(
              _hasFaceID ? Icons.face : Icons.fingerprint,
              size: 80,
              color: Theme.of(context).colorScheme.primary,
            ),
            const SizedBox(height: 24),
            Text(
              _hasFaceID ? 'Face ID' : 'Touch ID / ลายนิ้วมือ',
              style: Theme.of(context).textTheme.headlineSmall,
            ),
            const SizedBox(height: 16),
            Text(
              'เปิดใช้งาน ${_hasFaceID ? "Face ID" : "ลายนิ้วมือ"} '
              'เพื่อเข้าสู่ระบบและยืนยันธุรกรรมได้อย่างรวดเร็วและปลอดภัย',
              textAlign: TextAlign.center,
              style: const TextStyle(color: Colors.grey),
            ),
            const Spacer(),
            SizedBox(
              width: double.infinity,
              child: FilledButton(
                onPressed: _enableBiometric,
                child: Text('เปิดใช้งาน ${_hasFaceID ? "Face ID" : "ลายนิ้วมือ"}'),
              ),
            ),
            const SizedBox(height: 12),
            TextButton(
              onPressed: () => Navigator.pop(context),
              child: const Text('ข้ามไปก่อน'),
            ),
          ],
        ),
      ),
    );
  }

  bool get _hasFaceID =>
      _availableBiometrics.contains(BiometricType.face);

  Future<void> _enableBiometric() async {
    final authenticated = await _localAuth.authenticate(
      localizedReason: 'ยืนยันเพื่อเปิดใช้งาน biometric',
    );

    if (!mounted) return;

    if (authenticated) {
      context.read<SecurityBloc>().add(const EnableBiometricEvent());
      Navigator.pop(context);
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('เปิดใช้งาน Biometric สำเร็จ')),
      );
    }
  }
}
```

## สรุป

Fintech App ครอบคลุม:
1. **Dashboard** - Account overview, balance display with toggle
2. **Transactions** - History with filtering and categories
3. **Transfer** - Step-by-step with confirmation
4. **Security** - PIN, biometric, session timeout

## แบบทดสอบ

1. เพิ่ม QR code payment (เป็นทั้ง QR generator และ scanner)
2. Implement transaction dispute/report feature
3. สร้าง budgeting/spending analysis screen
4. เพิ่ม multi-factor authentication ด้วย OTP
