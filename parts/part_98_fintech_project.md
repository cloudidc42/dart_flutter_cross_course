# Part 98: Real-World Project - Fintech App

## 🎯 เป้าหมายของ Part นี้
- สร้าง Fintech app ระดับ production
- Security best practices
- Biometric authentication
- Transaction system
- แผนภูมิและสถิติ

---

## 1. App Overview

```
FintechApp - Digital Banking & Payment
├── Authentication (PIN + Biometric)
├── Account Overview (Balance, recent transactions)
├── Transaction History (filter, search, export)
├── Money Transfer
├── QR Code Payment
├── Bill Payment
├── Investment Portfolio
└── Financial Reports (Charts)
```

---

## 2. Security Layer

```dart
// core/security/auth_service.dart
import 'package:local_auth/local_auth.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class AuthService {
  final LocalAuthentication _localAuth = LocalAuthentication();
  final FlutterSecureStorage _secureStorage = const FlutterSecureStorage(
    aOptions: AndroidOptions(
      encryptedSharedPreferences: true,
    ),
    iOptions: IOSOptions(
      accessibility: KeychainAccessibility.first_unlock,
    ),
  );
  
  // ตรวจสอบ biometric availability:
  Future<bool> get isBiometricAvailable async {
    final isAvailable = await _localAuth.canCheckBiometrics;
    final isDeviceSupported = await _localAuth.isDeviceSupported();
    return isAvailable && isDeviceSupported;
  }
  
  // authenticate ด้วย biometric:
  Future<bool> authenticateWithBiometric() async {
    try {
      final biometrics = await _localAuth.getAvailableBiometrics();
      
      return await _localAuth.authenticate(
        localizedReason: 'ยืนยันตัวตนเพื่อเข้าถึงบัญชี',
        options: const AuthenticationOptions(
          biometricOnly: false,
          useErrorDialogs: true,
          stickyAuth: true,
        ),
      );
    } catch (e) {
      return false;
    }
  }
  
  // PIN management:
  Future<void> savePin(String pin) async {
    // Hash PIN ก่อนบันทึก:
    final hashedPin = _hashPin(pin);
    await _secureStorage.write(key: 'user_pin', value: hashedPin);
  }
  
  Future<bool> verifyPin(String pin) async {
    final stored = await _secureStorage.read(key: 'user_pin');
    if (stored == null) return false;
    return stored == _hashPin(pin);
  }
  
  String _hashPin(String pin) {
    // ในการใช้งานจริงใช้ bcrypt หรือ PBKDF2:
    final bytes = utf8.encode(pin + 'salt_value_here');
    final digest = sha256.convert(bytes);
    return digest.toString();
  }
  
  // Token management:
  Future<String?> getAccessToken() async {
    return _secureStorage.read(key: 'access_token');
  }
  
  Future<void> saveTokens({
    required String accessToken,
    required String refreshToken,
  }) async {
    await Future.wait([
      _secureStorage.write(key: 'access_token', value: accessToken),
      _secureStorage.write(key: 'refresh_token', value: refreshToken),
    ]);
  }
  
  Future<void> clearTokens() async {
    await Future.wait([
      _secureStorage.delete(key: 'access_token'),
      _secureStorage.delete(key: 'refresh_token'),
    ]);
  }
}

// PIN Entry Widget:
class PinEntryWidget extends StatefulWidget {
  final String title;
  final void Function(String pin) onCompleted;
  final int pinLength;
  
  const PinEntryWidget({
    super.key,
    required this.title,
    required this.onCompleted,
    this.pinLength = 6,
  });

  @override
  State<PinEntryWidget> createState() => _PinEntryWidgetState();
}

class _PinEntryWidgetState extends State<PinEntryWidget> {
  String _pin = '';
  bool _showError = false;
  
  void _addDigit(String digit) {
    if (_pin.length < widget.pinLength) {
      setState(() {
        _pin += digit;
        _showError = false;
      });
      
      if (_pin.length == widget.pinLength) {
        widget.onCompleted(_pin);
      }
    }
  }
  
  void _removeDigit() {
    if (_pin.isNotEmpty) {
      setState(() => _pin = _pin.substring(0, _pin.length - 1));
    }
  }
  
  void showError() {
    setState(() {
      _showError = true;
      _pin = '';
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: SafeArea(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text(widget.title, style: Theme.of(context).textTheme.headlineSmall),
            const SizedBox(height: 32),
            
            // PIN dots:
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: List.generate(
                widget.pinLength,
                (index) => AnimatedContainer(
                  duration: const Duration(milliseconds: 200),
                  margin: const EdgeInsets.symmetric(horizontal: 8),
                  width: 20,
                  height: 20,
                  decoration: BoxDecoration(
                    shape: BoxShape.circle,
                    color: index < _pin.length
                        ? (_showError ? Colors.red : Theme.of(context).primaryColor)
                        : Colors.grey.shade300,
                  ),
                ),
              ),
            ),
            
            if (_showError) ...[
              const SizedBox(height: 16),
              const Text(
                'PIN ไม่ถูกต้อง ลองใหม่อีกครั้ง',
                style: TextStyle(color: Colors.red),
              ),
            ],
            
            const SizedBox(height: 48),
            
            // Number pad:
            SizedBox(
              width: 280,
              child: GridView.builder(
                shrinkWrap: true,
                physics: const NeverScrollableScrollPhysics(),
                gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
                  crossAxisCount: 3,
                  childAspectRatio: 1.5,
                ),
                itemCount: 12,
                itemBuilder: (context, index) {
                  if (index == 9) return const SizedBox.shrink(); // empty
                  if (index == 10) return _DigitButton('0', _addDigit);
                  if (index == 11) {
                    return IconButton(
                      onPressed: _removeDigit,
                      icon: const Icon(Icons.backspace_outlined),
                    );
                  }
                  return _DigitButton('${index + 1}', _addDigit);
                },
              ),
            ),
            
            const SizedBox(height: 16),
            
            // Biometric button:
            TextButton.icon(
              onPressed: () async {
                final auth = AuthService();
                if (await auth.authenticateWithBiometric()) {
                  widget.onCompleted('biometric');
                }
              },
              icon: const Icon(Icons.fingerprint),
              label: const Text('ใช้ลายนิ้วมือ'),
            ),
          ],
        ),
      ),
    );
  }
}

class _DigitButton extends StatelessWidget {
  final String digit;
  final void Function(String) onTap;
  
  const _DigitButton(this.digit, this.onTap);

  @override
  Widget build(BuildContext context) {
    return InkWell(
      onTap: () => onTap(digit),
      borderRadius: BorderRadius.circular(40),
      child: Center(
        child: Text(
          digit,
          style: const TextStyle(fontSize: 24, fontWeight: FontWeight.w300),
        ),
      ),
    );
  }
}
```

---

## 3. Account Dashboard

```dart
class DashboardPage extends ConsumerWidget {
  const DashboardPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final account = ref.watch(accountProvider);
    final transactions = ref.watch(recentTransactionsProvider);
    
    return Scaffold(
      body: CustomScrollView(
        slivers: [
          // Balance Card:
          SliverToBoxAdapter(
            child: account.when(
              loading: () => const ShimmerBalanceCard(),
              error: (e, _) => ErrorCard(message: e.toString()),
              data: (account) => BalanceCard(account: account),
            ),
          ),
          
          // Quick Actions:
          const SliverToBoxAdapter(child: QuickActionsBar()),
          
          // Recent Transactions:
          SliverToBoxAdapter(
            child: Padding(
              padding: const EdgeInsets.all(16),
              child: Row(
                mainAxisAlignment: MainAxisAlignment.spaceBetween,
                children: [
                  const Text(
                    'ธุรกรรมล่าสุด',
                    style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold),
                  ),
                  TextButton(
                    onPressed: () => context.push('/transactions'),
                    child: const Text('ดูทั้งหมด'),
                  ),
                ],
              ),
            ),
          ),
          
          transactions.when(
            loading: () => const SliverToBoxAdapter(child: LoadingList()),
            error: (e, _) => SliverToBoxAdapter(child: ErrorCard(message: e.toString())),
            data: (txns) => SliverList(
              delegate: SliverChildBuilderDelegate(
                (context, index) => TransactionTile(transaction: txns[index]),
                childCount: txns.take(5).length,
              ),
            ),
          ),
        ],
      ),
    );
  }
}

// Balance Card:
class BalanceCard extends StatefulWidget {
  final Account account;
  
  const BalanceCard({super.key, required this.account});

  @override
  State<BalanceCard> createState() => _BalanceCardState();
}

class _BalanceCardState extends State<BalanceCard> {
  bool _isHidden = false;

  @override
  Widget build(BuildContext context) {
    return Container(
      margin: const EdgeInsets.all(16),
      padding: const EdgeInsets.all(24),
      decoration: BoxDecoration(
        gradient: const LinearGradient(
          colors: [Color(0xFF1A237E), Color(0xFF3949AB)],
          begin: Alignment.topLeft,
          end: Alignment.bottomRight,
        ),
        borderRadius: BorderRadius.circular(16),
        boxShadow: [
          BoxShadow(
            color: Colors.blue.withOpacity(0.3),
            blurRadius: 20,
            offset: const Offset(0, 10),
          ),
        ],
      ),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Row(
            mainAxisAlignment: MainAxisAlignment.spaceBetween,
            children: [
              Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  const Text(
                    'ยอดเงินรวม',
                    style: TextStyle(color: Colors.white70, fontSize: 14),
                  ),
                  const SizedBox(height: 4),
                  Row(
                    children: [
                      Text(
                        _isHidden
                            ? '฿ ••••••'
                            : '฿ ${NumberFormat('#,##0.00').format(widget.account.balance)}',
                        style: const TextStyle(
                          color: Colors.white,
                          fontSize: 28,
                          fontWeight: FontWeight.bold,
                        ),
                      ),
                      IconButton(
                        icon: Icon(
                          _isHidden ? Icons.visibility_off : Icons.visibility,
                          color: Colors.white70,
                          size: 20,
                        ),
                        onPressed: () => setState(() => _isHidden = !_isHidden),
                      ),
                    ],
                  ),
                ],
              ),
              const Icon(Icons.account_balance_wallet, color: Colors.white30, size: 48),
            ],
          ),
          
          const SizedBox(height: 24),
          
          Row(
            mainAxisAlignment: MainAxisAlignment.spaceBetween,
            children: [
              Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  const Text(
                    'รายรับเดือนนี้',
                    style: TextStyle(color: Colors.white70, fontSize: 12),
                  ),
                  Text(
                    '฿ ${NumberFormat('#,##0').format(widget.account.monthlyIncome)}',
                    style: const TextStyle(color: Colors.greenAccent, fontSize: 16),
                  ),
                ],
              ),
              Column(
                crossAxisAlignment: CrossAxisAlignment.end,
                children: [
                  const Text(
                    'รายจ่ายเดือนนี้',
                    style: TextStyle(color: Colors.white70, fontSize: 12),
                  ),
                  Text(
                    '฿ ${NumberFormat('#,##0').format(widget.account.monthlyExpense)}',
                    style: const TextStyle(color: Colors.redAccent, fontSize: 16),
                  ),
                ],
              ),
            ],
          ),
          
          const SizedBox(height: 16),
          
          Text(
            'บัญชี: ${_maskAccountNumber(widget.account.accountNumber)}',
            style: const TextStyle(color: Colors.white60, fontSize: 14),
          ),
        ],
      ),
    );
  }
  
  String _maskAccountNumber(String number) {
    if (number.length <= 4) return number;
    return '${'*' * (number.length - 4)}${number.substring(number.length - 4)}';
  }
}

// Transaction Tile:
class TransactionTile extends StatelessWidget {
  final Transaction transaction;
  
  const TransactionTile({super.key, required this.transaction});

  @override
  Widget build(BuildContext context) {
    final isIncome = transaction.type == TransactionType.income;
    
    return ListTile(
      leading: CircleAvatar(
        backgroundColor: (isIncome ? Colors.green : Colors.red).withOpacity(0.1),
        child: Icon(
          transaction.category.icon,
          color: isIncome ? Colors.green : Colors.red,
        ),
      ),
      title: Text(transaction.description),
      subtitle: Text(
        '${transaction.category.label} • ${DateFormat('d MMM').format(transaction.date)}',
        style: const TextStyle(fontSize: 12),
      ),
      trailing: Text(
        '${isIncome ? '+' : '-'}฿${NumberFormat('#,##0.00').format(transaction.amount)}',
        style: TextStyle(
          color: isIncome ? Colors.green : Colors.red,
          fontWeight: FontWeight.bold,
        ),
      ),
    );
  }
}
```

---

## 4. Transfer Money Screen

```dart
class TransferPage extends ConsumerStatefulWidget {
  const TransferPage({super.key});

  @override
  ConsumerState<TransferPage> createState() => _TransferPageState();
}

class _TransferPageState extends ConsumerState<TransferPage> {
  final _accountController = TextEditingController();
  final _amountController = TextEditingController();
  final _noteController = TextEditingController();
  
  String? _recipientName;
  bool _isVerifying = false;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('โอนเงิน')),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // Recipient:
            const Text('ผู้รับ', style: TextStyle(fontWeight: FontWeight.bold)),
            const SizedBox(height: 8),
            TextField(
              controller: _accountController,
              decoration: InputDecoration(
                hintText: 'เลขบัญชีหรือพร้อมเพย์',
                suffixIcon: _isVerifying
                    ? const SizedBox(
                        width: 20,
                        height: 20,
                        child: CircularProgressIndicator(strokeWidth: 2),
                      )
                    : IconButton(
                        icon: const Icon(Icons.search),
                        onPressed: _verifyRecipient,
                      ),
              ),
              keyboardType: TextInputType.number,
              onSubmitted: (_) => _verifyRecipient(),
            ),
            
            if (_recipientName != null) ...[
              const SizedBox(height: 8),
              Container(
                padding: const EdgeInsets.all(12),
                decoration: BoxDecoration(
                  color: Colors.green.shade50,
                  borderRadius: BorderRadius.circular(8),
                  border: Border.all(color: Colors.green.shade200),
                ),
                child: Row(
                  children: [
                    const Icon(Icons.check_circle, color: Colors.green),
                    const SizedBox(width: 8),
                    Text(_recipientName!),
                  ],
                ),
              ),
            ],
            
            const SizedBox(height: 24),
            
            // Amount:
            const Text('จำนวนเงิน', style: TextStyle(fontWeight: FontWeight.bold)),
            const SizedBox(height: 8),
            TextField(
              controller: _amountController,
              decoration: const InputDecoration(
                hintText: '0.00',
                prefixText: '฿ ',
                suffixText: 'THB',
              ),
              keyboardType: const TextInputType.numberWithOptions(decimal: true),
            ),
            
            // Quick amount buttons:
            Wrap(
              spacing: 8,
              children: [100, 500, 1000, 5000, 10000].map((amount) {
                return ActionChip(
                  label: Text('฿$amount'),
                  onPressed: () {
                    _amountController.text = amount.toString();
                  },
                );
              }).toList(),
            ),
            
            const SizedBox(height: 24),
            
            // Note:
            const Text('หมายเหตุ (ไม่บังคับ)'),
            const SizedBox(height: 8),
            TextField(
              controller: _noteController,
              decoration: const InputDecoration(hintText: 'เช่น ค่าอาหาร'),
              maxLength: 100,
            ),
            
            const SizedBox(height: 32),
            
            SizedBox(
              width: double.infinity,
              child: FilledButton(
                onPressed: _recipientName != null ? _confirmTransfer : null,
                child: const Text('ยืนยันการโอน'),
              ),
            ),
          ],
        ),
      ),
    );
  }
  
  Future<void> _verifyRecipient() async {
    setState(() => _isVerifying = true);
    
    await Future.delayed(const Duration(seconds: 1)); // API call
    
    setState(() {
      _isVerifying = false;
      _recipientName = 'สมชาย ใจดี'; // from API
    });
  }
  
  void _confirmTransfer() {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('ยืนยันการโอนเงิน'),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            _ConfirmRow('ผู้รับ:', _recipientName!),
            _ConfirmRow('บัญชี:', _accountController.text),
            _ConfirmRow('จำนวน:', '฿${_amountController.text}'),
            if (_noteController.text.isNotEmpty)
              _ConfirmRow('หมายเหตุ:', _noteController.text),
          ],
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('ยกเลิก'),
          ),
          FilledButton(
            onPressed: () {
              Navigator.pop(context);
              _processTransfer();
            },
            child: const Text('ยืนยัน'),
          ),
        ],
      ),
    );
  }
  
  Future<void> _processTransfer() async {
    // แสดง PIN/Biometric สำหรับยืนยัน:
    final confirmed = await Navigator.push<bool>(
      context,
      MaterialPageRoute(
        builder: (context) => PinEntryWidget(
          title: 'ยืนยัน PIN เพื่อโอนเงิน',
          onCompleted: (pin) async {
            final auth = AuthService();
            final valid = await auth.verifyPin(pin);
            if (context.mounted) {
              Navigator.pop(context, valid);
            }
          },
        ),
      ),
    );
    
    if (confirmed == true && mounted) {
      // Execute transfer
      await ref.read(transferProvider.notifier).transfer(
        toAccount: _accountController.text,
        amount: double.parse(_amountController.text),
        note: _noteController.text,
      );
      
      if (mounted) {
        context.pushReplacement('/transfer-success');
      }
    }
  }
  
  @override
  void dispose() {
    _accountController.dispose();
    _amountController.dispose();
    _noteController.dispose();
    super.dispose();
  }
}

class _ConfirmRow extends StatelessWidget {
  final String label;
  final String value;
  
  const _ConfirmRow(this.label, this.value);

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceBetween,
        children: [
          Text(label, style: const TextStyle(color: Colors.grey)),
          Text(value, style: const TextStyle(fontWeight: FontWeight.bold)),
        ],
      ),
    );
  }
}
```

---

## 5. สรุป Part 98

สิ่งที่เรียนรู้:
- ✅ Fintech app security architecture
- ✅ Biometric authentication
- ✅ PIN entry widget
- ✅ Secure storage สำหรับ tokens
- ✅ Account dashboard design
- ✅ Transfer money flow
- ✅ Transaction history

---

## ➡️ Part ถัดไป
**Part 99: Career Development และ Portfolio**
