# Part 76: In-App Purchases - การซื้อภายในแอป

## บทนำ

In-App Purchases (IAP) คือการขายสินค้าหรือบริการภายในแอป ทั้ง App Store และ Google Play มีระบบ IAP ของตัวเอง Flutter's `in_app_purchase` package ช่วยให้จัดการ IAP ได้บนทุกแพลตฟอร์ม

## 1. in_app_purchase Package

### การติดตั้ง

```yaml
# pubspec.yaml
dependencies:
  in_app_purchase: ^3.1.13
  in_app_purchase_android: ^0.3.6+3  # optional, สำหรับ Android specifics
  in_app_purchase_storekit: ^0.3.13+3  # optional, สำหรับ iOS specifics
```

### Android Configuration

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<uses-permission android:name="com.android.vending.BILLING" />
```

## 2. Product Types

### ประเภทของ Products

1. **Consumable** - สินค้าที่ใช้แล้วหมด (เช่น เหรียญ, ชีวิต)
2. **Non-consumable** - สินค้าที่ซื้อครั้งเดียวใช้ได้ตลอด (เช่น Remove Ads)
3. **Subscription** - การสมัครสมาชิกรายเดือน/รายปี

### Product Service

```dart
// lib/iap/product_service.dart
import 'dart:async';
import 'package:in_app_purchase/in_app_purchase.dart';

// Product IDs ต้องตรงกับที่กำหนดใน App Store/Play Store
class ProductIds {
  // Consumables
  static const String coins100 = 'coins_100';
  static const String coins500 = 'coins_500';
  static const String coins1000 = 'coins_1000';

  // Non-consumables
  static const String removeAds = 'remove_ads';
  static const String premiumTheme = 'premium_theme';

  // Subscriptions
  static const String premiumMonthly = 'premium_monthly';
  static const String premiumYearly = 'premium_yearly';

  static const Set<String> allProductIds = {
    coins100,
    coins500,
    coins1000,
    removeAds,
    premiumTheme,
    premiumMonthly,
    premiumYearly,
  };
}

class ProductService {
  final InAppPurchase _iap = InAppPurchase.instance;
  final _productsController =
      StreamController<List<ProductDetails>>.broadcast();

  List<ProductDetails> _products = [];
  bool _isAvailable = false;

  Stream<List<ProductDetails>> get productsStream => _productsController.stream;
  List<ProductDetails> get products => _products;
  bool get isAvailable => _isAvailable;

  Future<void> initialize() async {
    _isAvailable = await _iap.isAvailable();
    if (!_isAvailable) {
      print('In-app purchase not available');
      return;
    }

    await _loadProducts();
  }

  Future<void> _loadProducts() async {
    final response = await _iap.queryProductDetails(ProductIds.allProductIds);

    if (response.error != null) {
      print('Error loading products: ${response.error}');
      return;
    }

    if (response.notFoundIDs.isNotEmpty) {
      print('Products not found: ${response.notFoundIDs}');
    }

    _products = response.productDetails;
    _productsController.add(_products);
  }

  ProductDetails? getProduct(String productId) {
    return _products.firstWhere(
      (p) => p.id == productId,
      orElse: () => throw Exception('Product $productId not found'),
    );
  }

  void dispose() {
    _productsController.close();
  }
}
```

## 3. Purchase Flow

### Purchase Manager

```dart
// lib/iap/purchase_manager.dart
import 'dart:async';
import 'package:in_app_purchase/in_app_purchase.dart';
import 'product_service.dart';
import 'purchase_validator.dart';

enum PurchaseStatus {
  idle,
  processing,
  success,
  error,
  cancelled,
}

class PurchaseResult {
  final PurchaseStatus status;
  final String? productId;
  final String? errorMessage;

  const PurchaseResult({
    required this.status,
    this.productId,
    this.errorMessage,
  });
}

class PurchaseManager {
  final InAppPurchase _iap = InAppPurchase.instance;
  final PurchaseValidator _validator;

  StreamSubscription<List<PurchaseDetails>>? _purchaseSubscription;
  final _purchaseResultController =
      StreamController<PurchaseResult>.broadcast();

  Stream<PurchaseResult> get purchaseResultStream =>
      _purchaseResultController.stream;

  PurchaseManager({required PurchaseValidator validator})
      : _validator = validator;

  void initialize() {
    _purchaseSubscription =
        _iap.purchaseStream.listen(_handlePurchaseUpdate);
  }

  Future<void> _handlePurchaseUpdate(
    List<PurchaseDetails> purchaseDetailsList,
  ) async {
    for (final purchaseDetails in purchaseDetailsList) {
      if (purchaseDetails.status == PurchaseStatus2.pending) {
        // กำลังประมวลผล - แสดง loading
        _purchaseResultController.add(
          const PurchaseResult(status: PurchaseStatus.processing),
        );
      } else if (purchaseDetails.status == PurchaseStatus2.purchased ||
          purchaseDetails.status == PurchaseStatus2.restored) {
        // ซื้อสำเร็จหรือ restore - ตรวจสอบ receipt
        await _processPurchase(purchaseDetails);
      } else if (purchaseDetails.status == PurchaseStatus2.error) {
        // เกิดข้อผิดพลาด
        _purchaseResultController.add(
          PurchaseResult(
            status: PurchaseStatus.error,
            errorMessage: purchaseDetails.error?.message,
          ),
        );
      } else if (purchaseDetails.status == PurchaseStatus2.canceled) {
        _purchaseResultController.add(
          const PurchaseResult(status: PurchaseStatus.cancelled),
        );
      }

      // Complete the purchase เพื่อ acknowledge กับ store
      if (purchaseDetails.pendingCompletePurchase) {
        await _iap.completePurchase(purchaseDetails);
      }
    }
  }

  Future<void> _processPurchase(PurchaseDetails details) async {
    // ตรวจสอบ receipt กับ server
    final isValid = await _validator.validate(details);

    if (isValid) {
      // unlock feature ที่ซื้อ
      _purchaseResultController.add(
        PurchaseResult(
          status: PurchaseStatus.success,
          productId: details.productID,
        ),
      );
    } else {
      _purchaseResultController.add(
        const PurchaseResult(
          status: PurchaseStatus.error,
          errorMessage: 'Receipt validation failed',
        ),
      );
    }
  }

  // เริ่มกระบวนการซื้อ
  Future<void> buyProduct(ProductDetails productDetails) async {
    final purchaseParam = PurchaseParam(
      productDetails: productDetails,
    );

    final isConsumable = _isConsumable(productDetails.id);

    if (isConsumable) {
      await _iap.buyConsumable(purchaseParam: purchaseParam);
    } else {
      await _iap.buyNonConsumable(purchaseParam: purchaseParam);
    }
  }

  // Restore purchases (สำคัญมากสำหรับ iOS)
  Future<void> restorePurchases() async {
    await _iap.restorePurchases();
  }

  bool _isConsumable(String productId) {
    return productId.startsWith('coins_');
  }

  void dispose() {
    _purchaseSubscription?.cancel();
    _purchaseResultController.close();
  }
}

// Alias เพื่อหลีกเลี่ยง naming conflict
typedef PurchaseStatus2 = in_app_purchase.PurchaseStatus;
```

## 4. Receipt Validation

### Server-side Validation

```dart
// lib/iap/purchase_validator.dart
import 'package:in_app_purchase/in_app_purchase.dart';
import 'package:http/http.dart' as http;
import 'dart:convert';
import 'dart:io';

class PurchaseValidator {
  final String serverBaseUrl;
  final String userId;

  const PurchaseValidator({
    required this.serverBaseUrl,
    required this.userId,
  });

  Future<bool> validate(PurchaseDetails details) async {
    try {
      if (Platform.isAndroid) {
        return await _validateAndroid(details);
      } else if (Platform.isIOS) {
        return await _validateIOS(details);
      }
      return false;
    } catch (e) {
      print('Validation error: $e');
      return false;
    }
  }

  Future<bool> _validateAndroid(PurchaseDetails details) async {
    // ส่ง purchase token ไปตรวจสอบที่ server
    final response = await http.post(
      Uri.parse('$serverBaseUrl/api/iap/validate/android'),
      headers: {'Content-Type': 'application/json'},
      body: jsonEncode({
        'userId': userId,
        'productId': details.productID,
        'purchaseToken': details.verificationData.serverVerificationData,
        'packageName': 'com.example.app',
      }),
    );

    if (response.statusCode == 200) {
      final data = jsonDecode(response.body);
      return data['valid'] == true;
    }

    return false;
  }

  Future<bool> _validateIOS(PurchaseDetails details) async {
    // ส่ง receipt data ไปตรวจสอบที่ server
    final response = await http.post(
      Uri.parse('$serverBaseUrl/api/iap/validate/ios'),
      headers: {'Content-Type': 'application/json'},
      body: jsonEncode({
        'userId': userId,
        'productId': details.productID,
        'receiptData': details.verificationData.serverVerificationData,
      }),
    );

    if (response.statusCode == 200) {
      final data = jsonDecode(response.body);
      return data['valid'] == true;
    }

    return false;
  }
}
```

### Local Purchase Store

```dart
// lib/iap/purchase_store.dart
import 'package:shared_preferences/shared_preferences.dart';

class PurchaseStore {
  static const String _keyPrefix = 'iap_';
  static const String _coinsKey = 'coins';

  static Future<void> unlockFeature(String productId) async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setBool('$_keyPrefix$productId', true);
  }

  static Future<bool> isFeatureUnlocked(String productId) async {
    final prefs = await SharedPreferences.getInstance();
    return prefs.getBool('$_keyPrefix$productId') ?? false;
  }

  static Future<void> addCoins(String productId, int amount) async {
    final prefs = await SharedPreferences.getInstance();
    final current = prefs.getInt(_coinsKey) ?? 0;
    await prefs.setInt(_coinsKey, current + amount);
  }

  static Future<int> getCoins() async {
    final prefs = await SharedPreferences.getInstance();
    return prefs.getInt(_coinsKey) ?? 0;
  }

  static Future<void> spendCoins(int amount) async {
    final prefs = await SharedPreferences.getInstance();
    final current = prefs.getInt(_coinsKey) ?? 0;
    if (current < amount) throw Exception('ยอดเหรียญไม่เพียงพอ');
    await prefs.setInt(_coinsKey, current - amount);
  }
}
```

## 5. Workshop: Premium Features Unlock

### IAP Screen

```dart
// lib/screens/store_screen.dart
import 'package:flutter/material.dart';
import 'package:in_app_purchase/in_app_purchase.dart';
import '../iap/product_service.dart';
import '../iap/purchase_manager.dart';
import '../iap/purchase_store.dart';

class StoreScreen extends StatefulWidget {
  const StoreScreen({super.key});

  @override
  State<StoreScreen> createState() => _StoreScreenState();
}

class _StoreScreenState extends State<StoreScreen> {
  final _productService = ProductService();
  late PurchaseManager _purchaseManager;
  bool _isLoading = false;
  int _coins = 0;
  bool _hasRemovedAds = false;
  bool _hasPremiumTheme = false;
  bool _hasPremiumSub = false;

  @override
  void initState() {
    super.initState();
    _purchaseManager = PurchaseManager(
      validator: PurchaseValidator(
        serverBaseUrl: 'https://api.example.com',
        userId: 'user123',
      ),
    );
    _purchaseManager.initialize();

    _productService.initialize();
    _loadPurchaseStatus();

    // ฟัง purchase results
    _purchaseManager.purchaseResultStream.listen(_handlePurchaseResult);
  }

  Future<void> _loadPurchaseStatus() async {
    final coins = await PurchaseStore.getCoins();
    final removeAds = await PurchaseStore.isFeatureUnlocked(
      ProductIds.removeAds,
    );
    final premiumTheme = await PurchaseStore.isFeatureUnlocked(
      ProductIds.premiumTheme,
    );
    final premiumSub = await PurchaseStore.isFeatureUnlocked(
      ProductIds.premiumMonthly,
    );

    if (mounted) {
      setState(() {
        _coins = coins;
        _hasRemovedAds = removeAds;
        _hasPremiumTheme = premiumTheme;
        _hasPremiumSub = premiumSub;
      });
    }
  }

  void _handlePurchaseResult(PurchaseResult result) {
    if (result.status == PurchaseStatus.processing) {
      setState(() => _isLoading = true);
      return;
    }

    setState(() => _isLoading = false);

    if (result.status == PurchaseStatus.success) {
      _processPurchaseSuccess(result.productId!);
    } else if (result.status == PurchaseStatus.error) {
      _showError(result.errorMessage ?? 'เกิดข้อผิดพลาด');
    }
  }

  void _processPurchaseSuccess(String productId) {
    switch (productId) {
      case ProductIds.coins100:
        PurchaseStore.addCoins(productId, 100);
        _showSuccess('ได้รับ 100 เหรียญ!');
        break;
      case ProductIds.coins500:
        PurchaseStore.addCoins(productId, 500);
        _showSuccess('ได้รับ 500 เหรียญ!');
        break;
      case ProductIds.coins1000:
        PurchaseStore.addCoins(productId, 1000);
        _showSuccess('ได้รับ 1,000 เหรียญ!');
        break;
      case ProductIds.removeAds:
        PurchaseStore.unlockFeature(productId);
        setState(() => _hasRemovedAds = true);
        _showSuccess('ลบโฆษณาสำเร็จ!');
        break;
      case ProductIds.premiumTheme:
        PurchaseStore.unlockFeature(productId);
        setState(() => _hasPremiumTheme = true);
        _showSuccess('ได้รับธีม Premium!');
        break;
      case ProductIds.premiumMonthly:
      case ProductIds.premiumYearly:
        PurchaseStore.unlockFeature(productId);
        setState(() => _hasPremiumSub = true);
        _showSuccess('สมัครสมาชิก Premium สำเร็จ!');
        break;
    }

    _loadPurchaseStatus();
  }

  void _showSuccess(String message) {
    if (!mounted) return;
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(
        content: Text(message),
        backgroundColor: Colors.green,
      ),
    );
  }

  void _showError(String message) {
    if (!mounted) return;
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(
        content: Text(message),
        backgroundColor: Colors.red,
      ),
    );
  }

  Future<void> _purchaseProduct(ProductDetails product) async {
    await _purchaseManager.buyProduct(product);
  }

  @override
  void dispose() {
    _productService.dispose();
    _purchaseManager.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('ร้านค้า'),
        actions: [
          Padding(
            padding: const EdgeInsets.all(8),
            child: Chip(
              avatar: const Icon(Icons.monetization_on, size: 16),
              label: Text('$_coins'),
            ),
          ),
          IconButton(
            icon: const Icon(Icons.restore),
            onPressed: () => _purchaseManager.restorePurchases(),
            tooltip: 'กู้คืนการซื้อ',
          ),
        ],
      ),
      body: Stack(
        children: [
          StreamBuilder<List<ProductDetails>>(
            stream: _productService.productsStream,
            builder: (context, snapshot) {
              if (!snapshot.hasData) {
                return const Center(child: CircularProgressIndicator());
              }

              final products = snapshot.data!;
              if (products.isEmpty) {
                return const Center(
                  child: Text('ไม่พบสินค้า'),
                );
              }

              return ListView(
                padding: const EdgeInsets.all(16),
                children: [
                  _buildSection(
                    title: 'เหรียญ',
                    icon: Icons.monetization_on,
                    products: products
                        .where((p) => p.id.startsWith('coins_'))
                        .toList(),
                    type: 'consumable',
                  ),
                  const SizedBox(height: 24),
                  _buildSection(
                    title: 'ซื้อครั้งเดียว',
                    icon: Icons.star,
                    products: products
                        .where((p) =>
                            p.id == ProductIds.removeAds ||
                            p.id == ProductIds.premiumTheme)
                        .toList(),
                    type: 'non-consumable',
                  ),
                  const SizedBox(height: 24),
                  _buildSection(
                    title: 'สมาชิก Premium',
                    icon: Icons.workspace_premium,
                    products: products
                        .where((p) => p.id.startsWith('premium_'))
                        .toList(),
                    type: 'subscription',
                  ),
                ],
              );
            },
          ),

          if (_isLoading)
            Container(
              color: Colors.black54,
              child: const Center(
                child: Card(
                  child: Padding(
                    padding: EdgeInsets.all(24),
                    child: Column(
                      mainAxisSize: MainAxisSize.min,
                      children: [
                        CircularProgressIndicator(),
                        SizedBox(height: 16),
                        Text('กำลังดำเนินการ...'),
                      ],
                    ),
                  ),
                ),
              ),
            ),
        ],
      ),
    );
  }

  Widget _buildSection({
    required String title,
    required IconData icon,
    required List<ProductDetails> products,
    required String type,
  }) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Row(
          children: [
            Icon(icon, size: 20),
            const SizedBox(width: 8),
            Text(
              title,
              style: Theme.of(context).textTheme.titleLarge,
            ),
          ],
        ),
        const SizedBox(height: 12),
        ...products.map((product) => _buildProductCard(product, type)),
      ],
    );
  }

  Widget _buildProductCard(ProductDetails product, String type) {
    final isOwned = _isProductOwned(product.id);

    return Card(
      margin: const EdgeInsets.only(bottom: 8),
      child: ListTile(
        leading: CircleAvatar(
          child: Icon(_productIcon(product.id)),
        ),
        title: Text(product.title),
        subtitle: Text(product.description),
        trailing: isOwned
            ? const Chip(
                label: Text('ซื้อแล้ว'),
                backgroundColor: Colors.green,
                labelStyle: TextStyle(color: Colors.white),
              )
            : FilledButton(
                onPressed: () => _purchaseProduct(product),
                child: Text(product.price),
              ),
      ),
    );
  }

  bool _isProductOwned(String productId) {
    switch (productId) {
      case ProductIds.removeAds:
        return _hasRemovedAds;
      case ProductIds.premiumTheme:
        return _hasPremiumTheme;
      case ProductIds.premiumMonthly:
      case ProductIds.premiumYearly:
        return _hasPremiumSub;
      default:
        return false; // consumables ไม่มีสถานะ "owned"
    }
  }

  IconData _productIcon(String productId) {
    if (productId.startsWith('coins_')) return Icons.monetization_on;
    if (productId == ProductIds.removeAds) return Icons.block;
    if (productId == ProductIds.premiumTheme) return Icons.palette;
    return Icons.workspace_premium;
  }
}
```

## สรุป

In-App Purchases:
1. **ต้อง configure** ใน App Store Connect และ Google Play Console ก่อน
2. **Receipt validation** ควรทำบน server เพื่อความปลอดภัย
3. **Restore purchases** สำคัญสำหรับ iOS (App Store requirement)
4. **Test** ด้วย sandbox environment ก่อน production

## แบบทดสอบ

1. ความแตกต่างระหว่าง `buyConsumable()` และ `buyNonConsumable()` คืออะไร?
2. ทำไมต้อง `completePurchase()` หลังประมวลผลสำเร็จ?
3. สร้าง subscription management screen ที่แสดง expiry date
4. อธิบายว่า server-side receipt validation สำคัญกว่า client-side อย่างไร?
