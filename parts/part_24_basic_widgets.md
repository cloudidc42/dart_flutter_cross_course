# Part 24: Basic Widgets

## บทนำ

Widget พื้นฐานคือ building block ของทุก Flutter app ในบทนี้เราจะเรียนรู้ widget ที่ใช้บ่อยที่สุด ได้แก่ Text, Image, Icon, Container, SizedBox และ Spacer

---

## Text Widget

`Text` คือ widget สำหรับแสดงข้อความ เป็น widget ที่ใช้บ่อยมากที่สุดใน Flutter

### Text พื้นฐาน

```dart
// Text ธรรมดา
const Text('สวัสดีชาวโลก')

// Text พร้อม TextStyle
Text(
  'Hello Flutter',
  style: TextStyle(
    fontSize: 24,
    fontWeight: FontWeight.bold,
    color: Colors.blue,
    letterSpacing: 1.5,
    wordSpacing: 2.0,
    height: 1.5,        // line height multiplier
    fontStyle: FontStyle.italic,
    decoration: TextDecoration.underline,
    decorationColor: Colors.blue,
    decorationStyle: TextDecorationStyle.dashed,
  ),
)
```

### Text overflow และ maxLines

```dart
// ควบคุมการ overflow ของ Text
Text(
  'ข้อความยาวมากๆ ที่อาจเกินขนาดของ container และต้องถูก truncate',
  maxLines: 1,
  overflow: TextOverflow.ellipsis,  // '...' ตอนท้าย
)

Text(
  'ข้อความยาวมากๆ',
  maxLines: 2,
  overflow: TextOverflow.fade,      // fade ออก
)

Text(
  'ข้อความยาวมากๆ',
  overflow: TextOverflow.clip,      // ตัดออก
)

Text(
  'ข้อความยาวมากๆ',
  overflow: TextOverflow.visible,   // แสดงเกิน boundary
)

// softWrap - ขึ้นบรรทัดใหม่อัตโนมัติหรือไม่
Text(
  'ข้อความ',
  softWrap: false,  // ไม่ขึ้นบรรทัดใหม่
)
```

### textAlign

```dart
// การจัดตำแหน่งข้อความ
SizedBox(
  width: double.infinity,
  child: Column(
    children: [
      Text('Left', textAlign: TextAlign.left),
      Text('Center', textAlign: TextAlign.center),
      Text('Right', textAlign: TextAlign.right),
      Text('Justify', textAlign: TextAlign.justify),
      Text('Start', textAlign: TextAlign.start),  // ตามทิศทางการอ่าน
      Text('End', textAlign: TextAlign.end),
    ],
  ),
)
```

### TextStyle ที่น่าใช้

```dart
// ดึงจาก Theme (แนะนำ)
Text(
  'Headline',
  style: Theme.of(context).textTheme.headlineMedium,
)

// copyWith - แก้ไขบางค่า
Text(
  'Headline Bold',
  style: Theme.of(context).textTheme.headlineMedium?.copyWith(
    fontWeight: FontWeight.bold,
    color: Colors.red,
  ),
)

// TextStyle inherit
TextStyle baseStyle = const TextStyle(
  fontSize: 16,
  fontFamily: 'Kanit',
);

Text('Normal', style: baseStyle)
Text('Bold', style: baseStyle.copyWith(fontWeight: FontWeight.bold))
Text('Red', style: baseStyle.copyWith(color: Colors.red))
```

---

## RichText และ TextSpan

`RichText` ใช้แสดงข้อความที่มีหลาย style ใน string เดียว

```dart
// RichText พื้นฐาน
RichText(
  text: const TextSpan(
    style: TextStyle(color: Colors.black, fontSize: 16),
    children: [
      TextSpan(text: 'ราคา: '),
      TextSpan(
        text: '฿1,299',
        style: TextStyle(
          color: Colors.red,
          fontWeight: FontWeight.bold,
          fontSize: 20,
        ),
      ),
      TextSpan(
        text: ' (ลด 20%)',
        style: TextStyle(
          color: Colors.green,
          fontSize: 14,
        ),
      ),
    ],
  ),
)

// TextSpan พร้อม gesture
RichText(
  text: TextSpan(
    style: const TextStyle(color: Colors.black, fontSize: 16),
    children: [
      const TextSpan(text: 'คุณยอมรับ '),
      TextSpan(
        text: 'ข้อกำหนดและเงื่อนไข',
        style: const TextStyle(
          color: Colors.blue,
          decoration: TextDecoration.underline,
        ),
        recognizer: TapGestureRecognizer()
          ..onTap = () {
            // Handle tap
            print('Terms tapped');
          },
      ),
      const TextSpan(text: ' ของเรา'),
    ],
  ),
)
```

### SelectableText

```dart
// Text ที่ user สามารถเลือก copy ได้
SelectableText(
  'ข้อความที่ copy ได้',
  style: const TextStyle(fontSize: 16),
)

// SelectableText.rich
SelectableText.rich(
  TextSpan(
    children: [
      const TextSpan(text: 'Email: '),
      TextSpan(
        text: 'user@example.com',
        style: const TextStyle(color: Colors.blue),
      ),
    ],
  ),
)
```

---

## Image Widget

`Image` ใช้แสดงรูปภาพจากหลายแหล่ง

### Image.asset

```dart
// รูปจาก assets (ต้องประกาศใน pubspec.yaml)
Image.asset('assets/images/logo.png')

Image.asset(
  'assets/images/banner.jpg',
  width: 300,
  height: 200,
  fit: BoxFit.cover,   // วิธีวางรูป
)

// BoxFit options
// BoxFit.fill - ยืดเต็ม widget (อาจผิดสัดส่วน)
// BoxFit.contain - แสดงทั้งรูป ไม่ตัด
// BoxFit.cover - เต็ม widget ตัดถ้าต้อง
// BoxFit.fitWidth - เต็มความกว้าง
// BoxFit.fitHeight - เต็มความสูง
// BoxFit.none - ไม่ scale
// BoxFit.scaleDown - เล็กลงถ้าเกิน

// กำหนด color filter
Image.asset(
  'assets/images/icon.png',
  color: Colors.blue,
  colorBlendMode: BlendMode.srcIn,
)
```

### Image.network

```dart
// รูปจาก URL
Image.network('https://example.com/image.jpg')

Image.network(
  'https://picsum.photos/400/300',
  width: 400,
  height: 300,
  fit: BoxFit.cover,
  
  // Loading builder
  loadingBuilder: (context, child, loadingProgress) {
    if (loadingProgress == null) return child; // โหลดแล้ว
    
    return Container(
      width: 400,
      height: 300,
      color: Colors.grey[200],
      child: Center(
        child: CircularProgressIndicator(
          value: loadingProgress.expectedTotalBytes != null
              ? loadingProgress.cumulativeBytesLoaded /
                  loadingProgress.expectedTotalBytes!
              : null,
        ),
      ),
    );
  },
  
  // Error builder
  errorBuilder: (context, error, stackTrace) {
    return Container(
      width: 400,
      height: 300,
      color: Colors.grey[300],
      child: const Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          Icon(Icons.broken_image, size: 48, color: Colors.grey),
          Text('ไม่สามารถโหลดรูปได้'),
        ],
      ),
    );
  },
)
```

### Image.file

```dart
import 'dart:io';

// รูปจากไฟล์ในเครื่อง
Image.file(
  File('/path/to/image.jpg'),
  fit: BoxFit.cover,
)
```

### CircleAvatar

```dart
// รูปวงกลม (มักใช้กับ profile picture)
const CircleAvatar(
  radius: 40,
  backgroundImage: NetworkImage('https://example.com/avatar.jpg'),
)

// Circle avatar พร้อม fallback
CircleAvatar(
  radius: 40,
  backgroundColor: Colors.blue,
  child: user.avatarUrl != null
      ? null  // backgroundImage จะแสดง child บน background
      : Text(
          user.initials,
          style: const TextStyle(color: Colors.white, fontSize: 24),
        ),
  backgroundImage: user.avatarUrl != null
      ? NetworkImage(user.avatarUrl!)
      : null,
)
```

### FadeInImage

```dart
// แสดง placeholder ขณะโหลดรูป
FadeInImage(
  placeholder: const AssetImage('assets/images/placeholder.png'),
  image: const NetworkImage('https://example.com/image.jpg'),
  fit: BoxFit.cover,
  fadeInDuration: const Duration(milliseconds: 500),
)

// ใช้ memory placeholder (สีเดียว)
FadeInImage(
  placeholder: MemoryImage(kTransparentImage), // จาก package transparent_image
  image: const NetworkImage('https://example.com/image.jpg'),
)
```

### CachedNetworkImage (package)

```dart
// ต้องเพิ่ม cached_network_image ใน pubspec.yaml
import 'package:cached_network_image/cached_network_image.dart';

CachedNetworkImage(
  imageUrl: 'https://example.com/image.jpg',
  placeholder: (context, url) => const CircularProgressIndicator(),
  errorWidget: (context, url, error) => const Icon(Icons.error),
  fit: BoxFit.cover,
)
```

---

## Icon และ IconButton

### Icon

```dart
// Material Icons
const Icon(Icons.home)
const Icon(Icons.favorite, color: Colors.red)
const Icon(Icons.star, size: 48, color: Colors.amber)

// Outlined icons
const Icon(Icons.home_outlined)
const Icon(Icons.favorite_border)

// Cupertino Icons
import 'package:flutter/cupertino.dart';
const Icon(CupertinoIcons.heart)
const Icon(CupertinoIcons.star)
```

### IconButton

```dart
// IconButton - ปุ่มที่มีแค่ icon
IconButton(
  icon: const Icon(Icons.favorite_border),
  onPressed: () {
    print('Favorite pressed');
  },
  tooltip: 'เพิ่มในรายการโปรด',
  iconSize: 32,
  color: Colors.red,
)

// IconButton.filled (Material 3)
IconButton.filled(
  icon: const Icon(Icons.favorite),
  onPressed: () {},
)

// IconButton.outlined (Material 3)
IconButton.outlined(
  icon: const Icon(Icons.delete),
  onPressed: () {},
)

// ToggleButtons (หลาย icon button ที่ toggle กัน)
class ToggleDemo extends StatefulWidget {
  const ToggleDemo({super.key});

  @override
  State<ToggleDemo> createState() => _ToggleDemoState();
}

class _ToggleDemoState extends State<ToggleDemo> {
  final _selected = [false, false, false];

  @override
  Widget build(BuildContext context) {
    return ToggleButtons(
      isSelected: _selected,
      onPressed: (index) {
        setState(() => _selected[index] = !_selected[index]);
      },
      children: const [
        Icon(Icons.format_bold),
        Icon(Icons.format_italic),
        Icon(Icons.format_underline),
      ],
    );
  }
}
```

---

## Container Widget

`Container` คือ widget ที่รวม decoration, sizing, padding, margin ไว้ด้วยกัน

### Container พื้นฐาน

```dart
Container(
  width: 200,
  height: 100,
  color: Colors.blue,  // ถ้าไม่มี decoration
  child: const Text('Hello'),
)

// Container พร้อม decoration
Container(
  width: 200,
  height: 100,
  decoration: BoxDecoration(
    color: Colors.blue,          // background color
    borderRadius: BorderRadius.circular(16),
    border: Border.all(
      color: Colors.white,
      width: 2,
    ),
    boxShadow: [
      BoxShadow(
        color: Colors.black.withOpacity(0.2),
        blurRadius: 10,
        offset: const Offset(0, 4),
      ),
    ],
    gradient: const LinearGradient(
      colors: [Colors.blue, Colors.purple],
      begin: Alignment.topLeft,
      end: Alignment.bottomRight,
    ),
    image: const DecorationImage(
      image: NetworkImage('https://example.com/bg.jpg'),
      fit: BoxFit.cover,
    ),
  ),
  child: const Text('Hello'),
)
```

### BoxDecoration

```dart
// Gradient backgrounds
Container(
  decoration: const BoxDecoration(
    gradient: LinearGradient(
      colors: [Color(0xFF667eea), Color(0xFF764ba2)],
      begin: Alignment.topLeft,
      end: Alignment.bottomRight,
    ),
  ),
)

// Radial gradient
Container(
  decoration: const BoxDecoration(
    gradient: RadialGradient(
      colors: [Colors.yellow, Colors.orange, Colors.red],
      center: Alignment.center,
      radius: 0.8,
    ),
  ),
)

// Shape
Container(
  width: 80,
  height: 80,
  decoration: const BoxDecoration(
    color: Colors.blue,
    shape: BoxShape.circle,  // วงกลม
  ),
)

// Multiple box shadows
Container(
  decoration: BoxDecoration(
    color: Colors.white,
    borderRadius: BorderRadius.circular(16),
    boxShadow: [
      BoxShadow(
        color: Colors.grey.withOpacity(0.2),
        spreadRadius: 2,
        blurRadius: 10,
        offset: const Offset(0, 4),
      ),
      BoxShadow(
        color: Colors.grey.withOpacity(0.1),
        spreadRadius: 4,
        blurRadius: 20,
        offset: const Offset(0, 8),
      ),
    ],
  ),
)
```

### Padding และ Margin

```dart
// EdgeInsets
Container(
  padding: const EdgeInsets.all(16),              // ทุกด้าน
  padding: const EdgeInsets.symmetric(
    horizontal: 16, vertical: 8,                   // แกน
  ),
  padding: const EdgeInsets.only(
    left: 16, top: 8, right: 16, bottom: 8,        // แต่ละด้าน
  ),
  padding: const EdgeInsets.fromLTRB(16, 8, 16, 8), // L,T,R,B
  margin: const EdgeInsets.all(8),
)

// Padding widget (ถ้าต้องการแค่ padding ไม่ต้องการ decoration)
const Padding(
  padding: EdgeInsets.all(16),
  child: Text('Padded text'),
)
```

---

## SizedBox

`SizedBox` ใช้กำหนดขนาดหรือเพิ่ม spacing

```dart
// กำหนดขนาด
const SizedBox(width: 200, height: 100)

// แค่ spacing
const SizedBox(height: 16)  // vertical space
const SizedBox(width: 8)    // horizontal space

// บังคับ child ให้มีขนาดตาม SizedBox
SizedBox(
  width: 100,
  height: 100,
  child: ElevatedButton(  // button จะกว้าง 100, สูง 100
    onPressed: () {},
    child: const Text('100x100'),
  ),
)

// SizedBox.expand - เต็ม parent
const SizedBox.expand(
  child: Text('Full size'),
)

// SizedBox.shrink - 0x0
const SizedBox.shrink()  // ใช้แทน null หรือ empty widget
```

---

## Spacer

`Spacer` ใช้ใน Row/Column เพื่อดัน widget ออกจากกัน

```dart
// Row กับ Spacer
Row(
  children: [
    const Text('Left'),
    const Spacer(),         // ดัน Left กับ Right ออก
    const Text('Right'),
  ],
)

// Column กับ Spacer
Column(
  children: [
    const Text('Top'),
    const Spacer(),         // เต็มพื้นที่กลาง
    const Text('Bottom'),
  ],
)

// Spacer มี flex
Row(
  children: [
    const Text('A'),
    const Spacer(flex: 2), // กว้าง 2 เท่าของ Spacer flex:1
    const Text('B'),
    const Spacer(flex: 1),
    const Text('C'),
  ],
)
```

---

## Workshop: Product Card Widget

สร้าง Product Card ที่สวยงามและใช้งานได้จริง

### lib/models/product.dart

```dart
class Product {
  final String id;
  final String name;
  final String description;
  final double price;
  final double? originalPrice;
  final String imageUrl;
  final double rating;
  final int reviewCount;
  final String category;
  final bool isNew;
  final bool isBestseller;
  bool isFavorite;
  int stockCount;

  Product({
    required this.id,
    required this.name,
    required this.description,
    required this.price,
    this.originalPrice,
    required this.imageUrl,
    required this.rating,
    required this.reviewCount,
    required this.category,
    this.isNew = false,
    this.isBestseller = false,
    this.isFavorite = false,
    required this.stockCount,
  });

  double get discountPercent {
    if (originalPrice == null || originalPrice! <= price) return 0;
    return ((originalPrice! - price) / originalPrice! * 100).roundToDouble();
  }

  bool get hasDiscount => discountPercent > 0;
  bool get isInStock => stockCount > 0;
}
```

### lib/widgets/product_card.dart

```dart
import 'package:flutter/material.dart';
import '../models/product.dart';

class ProductCard extends StatefulWidget {
  final Product product;
  final VoidCallback? onTap;
  final void Function(Product)? onFavoriteToggle;
  final void Function(Product)? onAddToCart;

  const ProductCard({
    super.key,
    required this.product,
    this.onTap,
    this.onFavoriteToggle,
    this.onAddToCart,
  });

  @override
  State<ProductCard> createState() => _ProductCardState();
}

class _ProductCardState extends State<ProductCard> {
  bool _isHovered = false;

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final product = widget.product;

    return MouseRegion(
      onEnter: (_) => setState(() => _isHovered = true),
      onExit: (_) => setState(() => _isHovered = false),
      child: GestureDetector(
        onTap: widget.onTap,
        child: AnimatedContainer(
          duration: const Duration(milliseconds: 200),
          decoration: BoxDecoration(
            color: theme.colorScheme.surface,
            borderRadius: BorderRadius.circular(16),
            boxShadow: [
              BoxShadow(
                color: Colors.black.withOpacity(_isHovered ? 0.15 : 0.07),
                blurRadius: _isHovered ? 16 : 8,
                offset: Offset(0, _isHovered ? 6 : 3),
              ),
            ],
          ),
          child: ClipRRect(
            borderRadius: BorderRadius.circular(16),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                // Image section
                _buildImageSection(theme, product),

                // Info section
                _buildInfoSection(theme, product),
              ],
            ),
          ),
        ),
      ),
    );
  }

  Widget _buildImageSection(ThemeData theme, Product product) {
    return Stack(
      children: [
        // Product image
        AspectRatio(
          aspectRatio: 4 / 3,
          child: Image.network(
            product.imageUrl,
            fit: BoxFit.cover,
            loadingBuilder: (context, child, progress) {
              if (progress == null) return child;
              return Container(
                color: Colors.grey[200],
                child: const Center(child: CircularProgressIndicator()),
              );
            },
            errorBuilder: (_, __, ___) => Container(
              color: Colors.grey[200],
              child: const Icon(Icons.image_not_supported,
                size: 48, color: Colors.grey),
            ),
          ),
        ),

        // Badges (New, Bestseller, Discount)
        Positioned(
          top: 8,
          left: 8,
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              if (product.isNew)
                _buildBadge('ใหม่', Colors.blue),
              if (product.isBestseller)
                _buildBadge('ขายดี', Colors.orange),
              if (product.hasDiscount)
                _buildBadge('-${product.discountPercent.toInt()}%', Colors.red),
            ],
          ),
        ),

        // Favorite button
        Positioned(
          top: 8,
          right: 8,
          child: _buildFavoriteButton(theme, product),
        ),

        // Out of stock overlay
        if (!product.isInStock)
          Positioned.fill(
            child: Container(
              color: Colors.black.withOpacity(0.5),
              child: const Center(
                child: Text(
                  'สินค้าหมด',
                  style: TextStyle(
                    color: Colors.white,
                    fontSize: 18,
                    fontWeight: FontWeight.bold,
                  ),
                ),
              ),
            ),
          ),
      ],
    );
  }

  Widget _buildBadge(String text, Color color) {
    return Container(
      margin: const EdgeInsets.only(bottom: 4),
      padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
      decoration: BoxDecoration(
        color: color,
        borderRadius: BorderRadius.circular(8),
      ),
      child: Text(
        text,
        style: const TextStyle(
          color: Colors.white,
          fontSize: 11,
          fontWeight: FontWeight.bold,
        ),
      ),
    );
  }

  Widget _buildFavoriteButton(ThemeData theme, Product product) {
    return GestureDetector(
      onTap: () {
        setState(() => product.isFavorite = !product.isFavorite);
        widget.onFavoriteToggle?.call(product);
      },
      child: Container(
        padding: const EdgeInsets.all(8),
        decoration: BoxDecoration(
          color: Colors.white.withOpacity(0.9),
          shape: BoxShape.circle,
          boxShadow: [
            BoxShadow(
              color: Colors.black.withOpacity(0.1),
              blurRadius: 4,
            ),
          ],
        ),
        child: AnimatedSwitcher(
          duration: const Duration(milliseconds: 300),
          child: Icon(
            product.isFavorite ? Icons.favorite : Icons.favorite_border,
            key: ValueKey(product.isFavorite),
            size: 20,
            color: product.isFavorite ? Colors.red : Colors.grey,
          ),
        ),
      ),
    );
  }

  Widget _buildInfoSection(ThemeData theme, Product product) {
    return Padding(
      padding: const EdgeInsets.all(12),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          // Category
          Text(
            product.category,
            style: theme.textTheme.labelSmall?.copyWith(
              color: theme.colorScheme.primary,
              fontWeight: FontWeight.w600,
            ),
          ),

          const SizedBox(height: 4),

          // Name
          Text(
            product.name,
            style: theme.textTheme.titleSmall?.copyWith(
              fontWeight: FontWeight.bold,
            ),
            maxLines: 2,
            overflow: TextOverflow.ellipsis,
          ),

          const SizedBox(height: 8),

          // Rating
          _buildRating(theme, product),

          const SizedBox(height: 8),

          // Price
          _buildPrice(theme, product),

          const SizedBox(height: 12),

          // Add to cart button
          _buildAddToCartButton(theme, product),
        ],
      ),
    );
  }

  Widget _buildRating(ThemeData theme, Product product) {
    return Row(
      children: [
        // Stars
        ...List.generate(5, (index) {
          final starValue = index + 1;
          IconData icon;
          if (product.rating >= starValue) {
            icon = Icons.star;
          } else if (product.rating >= starValue - 0.5) {
            icon = Icons.star_half;
          } else {
            icon = Icons.star_border;
          }
          return Icon(icon, size: 14, color: Colors.amber[700]);
        }),

        const SizedBox(width: 4),

        Text(
          '${product.rating}',
          style: theme.textTheme.labelSmall?.copyWith(
            fontWeight: FontWeight.bold,
          ),
        ),

        const SizedBox(width: 4),

        Text(
          '(${product.reviewCount})',
          style: theme.textTheme.labelSmall?.copyWith(
            color: theme.colorScheme.onSurfaceVariant,
          ),
        ),
      ],
    );
  }

  Widget _buildPrice(ThemeData theme, Product product) {
    return Row(
      crossAxisAlignment: CrossAxisAlignment.end,
      children: [
        Text(
          '฿${product.price.toStringAsFixed(0)}',
          style: theme.textTheme.titleMedium?.copyWith(
            color: theme.colorScheme.primary,
            fontWeight: FontWeight.bold,
          ),
        ),

        if (product.hasDiscount) ...[
          const SizedBox(width: 8),
          Text(
            '฿${product.originalPrice!.toStringAsFixed(0)}',
            style: theme.textTheme.bodySmall?.copyWith(
              decoration: TextDecoration.lineThrough,
              color: theme.colorScheme.outline,
            ),
          ),
        ],
      ],
    );
  }

  Widget _buildAddToCartButton(ThemeData theme, Product product) {
    return SizedBox(
      width: double.infinity,
      child: ElevatedButton.icon(
        onPressed: product.isInStock
            ? () => widget.onAddToCart?.call(product)
            : null,
        icon: const Icon(Icons.shopping_cart_outlined, size: 18),
        label: Text(product.isInStock ? 'เพิ่มลงตะกร้า' : 'สินค้าหมด'),
        style: ElevatedButton.styleFrom(
          padding: const EdgeInsets.symmetric(vertical: 10),
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(10),
          ),
        ),
      ),
    );
  }
}
```

### lib/screens/product_catalog_screen.dart

```dart
import 'package:flutter/material.dart';
import '../models/product.dart';
import '../widgets/product_card.dart';

class ProductCatalogScreen extends StatefulWidget {
  const ProductCatalogScreen({super.key});

  @override
  State<ProductCatalogScreen> createState() => _ProductCatalogScreenState();
}

class _ProductCatalogScreenState extends State<ProductCatalogScreen> {
  int _cartCount = 0;
  
  final List<Product> _products = [
    Product(
      id: '1',
      name: 'Flutter Development Masterclass',
      description: 'เรียนรู้ Flutter จากศูนย์จนถึงระดับ Expert',
      price: 1299,
      originalPrice: 1999,
      imageUrl: 'https://picsum.photos/seed/flutter/400/300',
      rating: 4.8,
      reviewCount: 1250,
      category: 'คอร์สออนไลน์',
      isNew: true,
      stockCount: 999,
    ),
    Product(
      id: '2',
      name: 'Clean Architecture in Flutter',
      description: 'สร้าง Flutter app ด้วย Clean Architecture',
      price: 899,
      originalPrice: 1299,
      imageUrl: 'https://picsum.photos/seed/arch/400/300',
      rating: 4.6,
      reviewCount: 856,
      category: 'คอร์สออนไลน์',
      isBestseller: true,
      stockCount: 999,
    ),
    Product(
      id: '3',
      name: 'Dart Programming Guide',
      description: 'เรียน Dart ตั้งแต่พื้นฐาน',
      price: 599,
      imageUrl: 'https://picsum.photos/seed/dart/400/300',
      rating: 4.5,
      reviewCount: 423,
      category: 'หนังสือ',
      stockCount: 50,
    ),
    Product(
      id: '4',
      name: 'State Management with Riverpod',
      description: 'จัดการ state ด้วย Riverpod อย่างมืออาชีพ',
      price: 799,
      imageUrl: 'https://picsum.photos/seed/riverpod/400/300',
      rating: 4.9,
      reviewCount: 678,
      category: 'คอร์สออนไลน์',
      stockCount: 0,  // out of stock
    ),
  ];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('ร้านค้า Flutter'),
        actions: [
          Stack(
            clipBehavior: Clip.none,
            children: [
              IconButton(
                icon: const Icon(Icons.shopping_cart),
                onPressed: () {
                  ScaffoldMessenger.of(context).showSnackBar(
                    SnackBar(content: Text('ตะกร้า: $_cartCount รายการ')),
                  );
                },
              ),
              if (_cartCount > 0)
                Positioned(
                  right: 4,
                  top: 4,
                  child: Container(
                    padding: const EdgeInsets.all(4),
                    decoration: const BoxDecoration(
                      color: Colors.red,
                      shape: BoxShape.circle,
                    ),
                    child: Text(
                      '$_cartCount',
                      style: const TextStyle(
                        color: Colors.white,
                        fontSize: 10,
                        fontWeight: FontWeight.bold,
                      ),
                    ),
                  ),
                ),
            ],
          ),
        ],
      ),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: GridView.builder(
          gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
            crossAxisCount: 2,
            crossAxisSpacing: 12,
            mainAxisSpacing: 12,
            childAspectRatio: 0.65,
          ),
          itemCount: _products.length,
          itemBuilder: (context, index) {
            final product = _products[index];
            return ProductCard(
              product: product,
              onTap: () {
                ScaffoldMessenger.of(context).showSnackBar(
                  SnackBar(content: Text('เปิด: ${product.name}')),
                );
              },
              onFavoriteToggle: (p) {
                ScaffoldMessenger.of(context).showSnackBar(
                  SnackBar(
                    content: Text(p.isFavorite
                        ? 'เพิ่ม ${p.name} ในรายการโปรด'
                        : 'ลบ ${p.name} ออกจากรายการโปรด'),
                    duration: const Duration(seconds: 1),
                  ),
                );
              },
              onAddToCart: (p) {
                setState(() => _cartCount++);
                ScaffoldMessenger.of(context).showSnackBar(
                  SnackBar(
                    content: Text('เพิ่ม "${p.name}" ลงตะกร้าแล้ว'),
                    action: SnackBarAction(
                      label: 'ดูตะกร้า',
                      onPressed: () {},
                    ),
                  ),
                );
              },
            );
          },
        ),
      ),
    );
  }
}
```

### lib/main.dart

```dart
import 'package:flutter/material.dart';
import 'screens/product_catalog_screen.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Product Catalog',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
        useMaterial3: true,
      ),
      home: const ProductCatalogScreen(),
    );
  }
}
```

---

## สรุป Basic Widgets

| Widget | ใช้สำหรับ |
|--------|-----------|
| Text | แสดงข้อความ |
| RichText | ข้อความหลาย style |
| SelectableText | ข้อความที่ copy ได้ |
| Image.asset | รูปจาก assets |
| Image.network | รูปจาก URL |
| Image.file | รูปจากไฟล์ |
| CircleAvatar | รูปวงกลม |
| Icon | Material icons |
| IconButton | ปุ่ม icon |
| Container | Box ที่ customize ได้ |
| SizedBox | กำหนดขนาด/spacing |
| Spacer | ดัน widget ออกจากกัน |

## สรุปบทที่ 24

ในบทนี้เราได้เรียนรู้:

1. **Text**: style, overflow, maxLines, textAlign
2. **RichText**: แสดงข้อความหลาย style
3. **Image**: asset, network, file พร้อม error/loading handler
4. **Icon และ IconButton**: การใช้ icon ใน UI
5. **Container**: BoxDecoration, padding, margin, gradient
6. **SizedBox และ Spacer**: การจัดระยะ spacing

บทต่อไปเราจะเรียน Layout Widgets เช่น Row, Column, Stack และอื่นๆ
