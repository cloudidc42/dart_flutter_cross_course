# Part 30: Basic Animations

## บทนำ

Animation คือสิ่งที่ทำให้ Flutter app มีชีวิตชีวา Flutter มีระบบ animation ที่ยืดหยุ่นมาก ตั้งแต่ implicit animations ที่ใช้งานง่าย ไปจนถึง explicit animations ที่ควบคุมได้ทุกรายละเอียด

---

## Animation ประเภทต่างๆ

```
Flutter Animations
├── Implicit Animations (ง่าย, อัตโนมัติ)
│   ├── AnimatedContainer
│   ├── AnimatedOpacity
│   ├── AnimatedSize
│   ├── AnimatedAlign
│   ├── AnimatedPositioned
│   ├── AnimatedPadding
│   ├── AnimatedDefaultTextStyle
│   ├── AnimatedSwitcher
│   └── TweenAnimationBuilder
│
└── Explicit Animations (ควบคุมเอง)
    ├── AnimationController
    ├── Tween + CurvedAnimation
    ├── AnimatedWidget
    ├── AnimatedBuilder
    └── CustomPainter
```

---

## AnimationController

AnimationController คือ "เครื่องยนต์" ของ animation ควบคุม duration, direction และ state

```dart
class AnimationDemo extends StatefulWidget {
  const AnimationDemo({super.key});

  @override
  State<AnimationDemo> createState() => _AnimationDemoState();
}

// ต้องใช้ TickerProviderStateMixin หรือ SingleTickerProviderStateMixin
class _AnimationDemoState extends State<AnimationDemo>
    with SingleTickerProviderStateMixin {
  
  late AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,              // TickerProvider
      duration: const Duration(milliseconds: 800),
      reverseDuration: const Duration(milliseconds: 400), // เร็วกว่าตอน forward
    );
    
    // Listener
    _controller.addListener(() {
      // เรียกทุก frame (0.0 to 1.0)
      print(_controller.value);
    });
    
    // Status listener
    _controller.addStatusListener((status) {
      switch (status) {
        case AnimationStatus.forward:
          print('กำลัง animate ไปข้างหน้า');
        case AnimationStatus.reverse:
          print('กำลัง animate ย้อนกลับ');
        case AnimationStatus.completed:
          print('animation เสร็จแล้ว (forward)');
        case AnimationStatus.dismissed:
          print('animation กลับต้นแล้ว (reverse)');
      }
    });
  }

  @override
  void dispose() {
    _controller.dispose(); // MUST dispose
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      mainAxisAlignment: MainAxisAlignment.center,
      children: [
        // Progress indicator
        AnimatedBuilder(
          animation: _controller,
          builder: (context, child) {
            return Text(
              'Value: ${_controller.value.toStringAsFixed(2)}',
              style: const TextStyle(fontSize: 20),
            );
          },
        ),
        
        const SizedBox(height: 24),
        
        // Control buttons
        Wrap(
          spacing: 8,
          children: [
            ElevatedButton(
              onPressed: () => _controller.forward(),
              child: const Text('Forward'),
            ),
            ElevatedButton(
              onPressed: () => _controller.reverse(),
              child: const Text('Reverse'),
            ),
            ElevatedButton(
              onPressed: () => _controller.reset(),
              child: const Text('Reset'),
            ),
            ElevatedButton(
              onPressed: () => _controller.repeat(),
              child: const Text('Repeat'),
            ),
            ElevatedButton(
              onPressed: () => _controller.repeat(reverse: true),
              child: const Text('Ping-Pong'),
            ),
            ElevatedButton(
              onPressed: () => _controller.stop(),
              child: const Text('Stop'),
            ),
          ],
        ),
      ],
    );
  }
}

// Multiple AnimationControllers
class _MultiControllerState extends State<MultiControllerDemo>
    with TickerProviderStateMixin { // TickerProvider สำหรับหลาย controller
  
  late AnimationController _fadeController;
  late AnimationController _slideController;
  late AnimationController _scaleController;
  
  @override
  void initState() {
    super.initState();
    _fadeController = AnimationController(vsync: this, duration: const Duration(milliseconds: 500));
    _slideController = AnimationController(vsync: this, duration: const Duration(milliseconds: 600));
    _scaleController = AnimationController(vsync: this, duration: const Duration(milliseconds: 400));
  }
  
  @override
  void dispose() {
    _fadeController.dispose();
    _slideController.dispose();
    _scaleController.dispose();
    super.dispose();
  }
}
```

---

## Tween และ CurvedAnimation

`Tween` แปลง animation value จาก 0.0-1.0 ไปเป็นค่าต่างๆ

### Tween พื้นฐาน

```dart
// Tween<double>
final _animation = Tween<double>(begin: 0, end: 1).animate(_controller);

// Color Tween
final _colorAnimation = ColorTween(
  begin: Colors.blue,
  end: Colors.red,
).animate(_controller);

// Offset Tween
final _slideAnimation = Tween<Offset>(
  begin: const Offset(-1, 0), // จากซ้าย
  end: Offset.zero,
).animate(_controller);

// Size Tween
final _sizeAnimation = Tween<Size>(
  begin: const Size(50, 50),
  end: const Size(200, 200),
).animate(_controller);

// Border radius
final _borderAnimation = Tween<BorderRadius?>(
  begin: BorderRadius.circular(0),
  end: BorderRadius.circular(50),
).animate(_controller);
```

### CurvedAnimation

```dart
// CurvedAnimation เพิ่ม easing ให้ animation
final _curvedAnimation = CurvedAnimation(
  parent: _controller,
  curve: Curves.easeInOut,
  reverseCurve: Curves.easeIn,
);

// ใช้กับ Tween
final _animation = Tween<double>(begin: 0, end: 1).animate(_curvedAnimation);

// Curves ยอดนิยม
Curves.linear          // ความเร็วคงที่
Curves.ease            // เร็วตอนกลาง
Curves.easeIn          // ช้าก่อน เร็วหลัง
Curves.easeOut         // เร็วก่อน ช้าหลัง
Curves.easeInOut       // ช้าต้น เร็วกลาง ช้าปลาย
Curves.bounceOut       // กระดอน
Curves.elasticOut      // ยืดหยุ่น
Curves.decelerate      // ลดความเร็ว
Curves.fastOutSlowIn   // Material Design standard
Curves.slowMiddle      // ช้าตรงกลาง
```

### Chained Animations (Intervals)

```dart
// Stagger animation - animate ทีละส่วน
class _StaggerState extends State<StaggerDemo>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  
  late Animation<double> _fadeAnimation;
  late Animation<Offset> _slideAnimation;
  late Animation<double> _scaleAnimation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 1200),
    );
    
    // แบ่ง timeline: fade 0-40%, slide 20-70%, scale 60-100%
    _fadeAnimation = Tween<double>(begin: 0, end: 1).animate(
      CurvedAnimation(
        parent: _controller,
        curve: const Interval(0.0, 0.4, curve: Curves.easeIn),
      ),
    );
    
    _slideAnimation = Tween<Offset>(
      begin: const Offset(0, 0.5),
      end: Offset.zero,
    ).animate(
      CurvedAnimation(
        parent: _controller,
        curve: const Interval(0.2, 0.7, curve: Curves.easeOutCubic),
      ),
    );
    
    _scaleAnimation = Tween<double>(begin: 0.8, end: 1.0).animate(
      CurvedAnimation(
        parent: _controller,
        curve: const Interval(0.6, 1.0, curve: Curves.elasticOut),
      ),
    );
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _controller,
      builder: (context, child) {
        return FadeTransition(
          opacity: _fadeAnimation,
          child: SlideTransition(
            position: _slideAnimation,
            child: ScaleTransition(
              scale: _scaleAnimation,
              child: child,
            ),
          ),
        );
      },
      child: const Text('Stagger Animation!', style: TextStyle(fontSize: 28)),
    );
  }
}
```

---

## AnimatedWidget

`AnimatedWidget` simplify การสร้าง widget ที่ animate เอง

```dart
// สร้าง animated widget แบบง่าย
class SpinningIcon extends AnimatedWidget {
  final IconData icon;
  final double size;
  final Color? color;
  
  const SpinningIcon({
    super.key,
    required super.listenable,
    required this.icon,
    this.size = 48,
    this.color,
  });

  @override
  Widget build(BuildContext context) {
    final animation = listenable as Animation<double>;
    
    return Transform.rotate(
      angle: animation.value * 2 * 3.14159, // full rotation
      child: Icon(icon, size: size, color: color),
    );
  }
}

// ใช้งาน
SpinningIcon(
  listenable: _controller,
  icon: Icons.refresh,
  size: 64,
  color: Theme.of(context).colorScheme.primary,
)

// ตัวอย่างอื่น: Pulsing Widget
class PulsingWidget extends AnimatedWidget {
  final Widget child;
  
  const PulsingWidget({
    super.key,
    required super.listenable,
    required this.child,
  });

  @override
  Widget build(BuildContext context) {
    final animation = listenable as Animation<double>;
    
    return Transform.scale(
      scale: animation.value,
      child: child,
    );
  }
}

// Setup
final _pulseAnimation = Tween<double>(begin: 0.95, end: 1.05).animate(
  CurvedAnimation(parent: _controller, curve: Curves.easeInOut),
);
_controller.repeat(reverse: true);

PulsingWidget(
  listenable: _pulseAnimation,
  child: const Icon(Icons.favorite, size: 64, color: Colors.red),
)
```

---

## AnimatedBuilder

`AnimatedBuilder` ใช้สร้าง animation แบบยืดหยุ่นกว่า AnimatedWidget

```dart
// AnimatedBuilder พื้นฐาน
AnimatedBuilder(
  animation: _controller,
  builder: (context, child) {
    return Transform.rotate(
      angle: _controller.value * 2 * 3.14159,
      child: child, // child ไม่ rebuild เมื่อ animation เปลี่ยน (performance)
    );
  },
  // child ส่วนที่ไม่เปลี่ยนตาม animation
  child: const Icon(Icons.star, size: 64),
)

// ตัวอย่าง: Animated progress ring
class AnimatedProgressRing extends StatelessWidget {
  final Animation<double> animation;
  final Color color;
  final double size;
  
  const AnimatedProgressRing({
    super.key,
    required this.animation,
    this.color = Colors.blue,
    this.size = 100,
  });

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: animation,
      builder: (context, child) {
        return SizedBox(
          width: size,
          height: size,
          child: CustomPaint(
            painter: _RingPainter(
              progress: animation.value,
              color: color,
            ),
            child: Center(
              child: Text(
                '${(animation.value * 100).toInt()}%',
                style: TextStyle(
                  fontSize: size * 0.25,
                  fontWeight: FontWeight.bold,
                ),
              ),
            ),
          ),
        );
      },
    );
  }
}

class _RingPainter extends CustomPainter {
  final double progress;
  final Color color;
  
  _RingPainter({required this.progress, required this.color});

  @override
  void paint(Canvas canvas, Size size) {
    final center = Offset(size.width / 2, size.height / 2);
    final radius = (size.width - 10) / 2;
    
    // Background ring
    canvas.drawCircle(
      center,
      radius,
      Paint()
        ..color = color.withOpacity(0.2)
        ..style = PaintingStyle.stroke
        ..strokeWidth = 8,
    );
    
    // Progress arc
    canvas.drawArc(
      Rect.fromCircle(center: center, radius: radius),
      -3.14159 / 2,
      progress * 2 * 3.14159,
      false,
      Paint()
        ..color = color
        ..style = PaintingStyle.stroke
        ..strokeWidth = 8
        ..strokeCap = StrokeCap.round,
    );
  }

  @override
  bool shouldRepaint(_RingPainter oldDelegate) {
    return oldDelegate.progress != progress;
  }
}
```

---

## Implicit Animated Widgets

Implicit animations ง่ายกว่ามาก - แค่เปลี่ยน value แล้วจะ animate เอง

### AnimatedContainer

```dart
class AnimatedContainerDemo extends StatefulWidget {
  const AnimatedContainerDemo({super.key});

  @override
  State<AnimatedContainerDemo> createState() => _AnimatedContainerDemoState();
}

class _AnimatedContainerDemoState extends State<AnimatedContainerDemo> {
  bool _isExpanded = false;
  bool _isBlue = true;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        AnimatedContainer(
          duration: const Duration(milliseconds: 500),
          curve: Curves.easeInOut,
          width: _isExpanded ? 300 : 100,
          height: _isExpanded ? 200 : 100,
          decoration: BoxDecoration(
            color: _isBlue ? Colors.blue : Colors.red,
            borderRadius: BorderRadius.circular(_isExpanded ? 24 : 50),
            boxShadow: [
              BoxShadow(
                color: (_isBlue ? Colors.blue : Colors.red).withOpacity(0.4),
                blurRadius: _isExpanded ? 20 : 5,
                spreadRadius: _isExpanded ? 4 : 0,
              ),
            ],
          ),
          child: Center(
            child: Icon(
              _isExpanded ? Icons.compress : Icons.expand,
              color: Colors.white,
              size: _isExpanded ? 48 : 24,
            ),
          ),
        ),
        
        const SizedBox(height: 24),
        
        Row(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            ElevatedButton(
              onPressed: () => setState(() => _isExpanded = !_isExpanded),
              child: Text(_isExpanded ? 'Collapse' : 'Expand'),
            ),
            const SizedBox(width: 16),
            ElevatedButton(
              onPressed: () => setState(() => _isBlue = !_isBlue),
              child: Text(_isBlue ? 'Make Red' : 'Make Blue'),
            ),
          ],
        ),
      ],
    );
  }
}
```

### AnimatedOpacity

```dart
AnimatedOpacity(
  opacity: _isVisible ? 1.0 : 0.0,
  duration: const Duration(milliseconds: 400),
  child: const Text('ข้อความที่ fade in/out'),
)
```

### AnimatedSwitcher

```dart
// Animate เมื่อ child เปลี่ยน
AnimatedSwitcher(
  duration: const Duration(milliseconds: 400),
  transitionBuilder: (child, animation) {
    return FadeTransition(
      opacity: animation,
      child: ScaleTransition(scale: animation, child: child),
    );
  },
  child: _isLoading
      ? const CircularProgressIndicator(key: ValueKey('loading'))
      : Icon(
          _isSuccess ? Icons.check_circle : Icons.error,
          key: ValueKey(_isSuccess),
          color: _isSuccess ? Colors.green : Colors.red,
          size: 64,
        ),
)
```

### TweenAnimationBuilder

```dart
// Animate ค่าใดๆ ด้วย Tween โดยไม่ต้องมี controller
TweenAnimationBuilder<double>(
  tween: Tween(begin: 0, end: _targetValue),
  duration: const Duration(milliseconds: 800),
  curve: Curves.easeOut,
  builder: (context, value, child) {
    return Column(
      children: [
        Text(
          '${value.toInt()}',
          style: const TextStyle(fontSize: 48, fontWeight: FontWeight.bold),
        ),
        child!,
      ],
    );
  },
  child: const Text('คะแนน'), // child ไม่ rebuild ตาม animation
)

// Color animation
TweenAnimationBuilder<Color?>(
  tween: ColorTween(
    begin: Colors.blue,
    end: _targetColor,
  ),
  duration: const Duration(milliseconds: 500),
  builder: (context, color, child) {
    return Container(color: color, child: child);
  },
  child: const Text('Animated color'),
)
```

---

## Hero Animations

Hero animations ทำให้ widget เคลื่อนที่ระหว่าง route ต่างๆ อย่างสวยงาม

```dart
// หน้าที่ 1: List screen
GridView.builder(
  itemBuilder: (context, index) {
    final product = products[index];
    return GestureDetector(
      onTap: () {
        Navigator.push(
          context,
          MaterialPageRoute(
            builder: (_) => ProductDetailScreen(product: product),
          ),
        );
      },
      child: Hero(
        tag: 'product-${product.id}', // unique tag
        child: Image.network(product.imageUrl, fit: BoxFit.cover),
      ),
    );
  },
)

// หน้าที่ 2: Detail screen
class ProductDetailScreen extends StatelessWidget {
  final Product product;
  
  const ProductDetailScreen({super.key, required this.product});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: CustomScrollView(
        slivers: [
          SliverAppBar(
            expandedHeight: 300,
            flexibleSpace: FlexibleSpaceBar(
              background: Hero(
                tag: 'product-${product.id}', // ต้องตรงกับหน้าแรก
                child: Image.network(product.imageUrl, fit: BoxFit.cover),
              ),
            ),
          ),
          SliverToBoxAdapter(
            child: Padding(
              padding: const EdgeInsets.all(16),
              child: Text(product.name, style: const TextStyle(fontSize: 24)),
            ),
          ),
        ],
      ),
    );
  }
}

// Hero กับ Text
// หน้า 1
Hero(
  tag: 'product-title-${product.id}',
  child: Material(
    color: Colors.transparent,
    child: Text(product.name, style: const TextStyle(fontSize: 16)),
  ),
)

// หน้า 2
Hero(
  tag: 'product-title-${product.id}',
  child: Material(
    color: Colors.transparent,
    child: Text(product.name, style: const TextStyle(fontSize: 32)),
  ),
)
```

---

## Workshop: Animated Product Card

สร้าง Product Card ที่มี animation สวยงาม

### lib/widgets/animated_product_card.dart

```dart
import 'package:flutter/material.dart';

class AnimatedProductCard extends StatefulWidget {
  final String id;
  final String name;
  final double price;
  final String imageUrl;
  final double rating;
  final VoidCallback? onTap;

  const AnimatedProductCard({
    super.key,
    required this.id,
    required this.name,
    required this.price,
    required this.imageUrl,
    required this.rating,
    this.onTap,
  });

  @override
  State<AnimatedProductCard> createState() => _AnimatedProductCardState();
}

class _AnimatedProductCardState extends State<AnimatedProductCard>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _scaleAnimation;
  late Animation<double> _elevationAnimation;
  late Animation<double> _favoriteAnimation;

  bool _isFavorite = false;
  bool _isPressed = false;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 300),
    );

    _scaleAnimation = Tween<double>(begin: 1.0, end: 0.97).animate(
      CurvedAnimation(parent: _controller, curve: Curves.easeInOut),
    );

    _elevationAnimation = Tween<double>(begin: 2, end: 8).animate(
      CurvedAnimation(parent: _controller, curve: Curves.easeOut),
    );

    _favoriteAnimation = Tween<double>(begin: 1.0, end: 1.3).animate(
      CurvedAnimation(
        parent: AnimationController(
          vsync: this,
          duration: const Duration(milliseconds: 200),
        )..forward(),
        curve: Curves.elasticOut,
      ),
    );
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  void _onTapDown(TapDownDetails _) {
    _controller.forward();
    setState(() => _isPressed = true);
  }

  void _onTapUp(TapUpDetails _) {
    _controller.reverse();
    setState(() => _isPressed = false);
    widget.onTap?.call();
  }

  void _onTapCancel() {
    _controller.reverse();
    setState(() => _isPressed = false);
  }

  void _toggleFavorite() {
    setState(() => _isFavorite = !_isFavorite);
  }

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);

    return AnimatedBuilder(
      animation: _controller,
      builder: (context, child) {
        return Transform.scale(
          scale: _scaleAnimation.value,
          child: child,
        );
      },
      child: GestureDetector(
        onTapDown: _onTapDown,
        onTapUp: _onTapUp,
        onTapCancel: _onTapCancel,
        child: AnimatedContainer(
          duration: const Duration(milliseconds: 200),
          decoration: BoxDecoration(
            color: theme.colorScheme.surface,
            borderRadius: BorderRadius.circular(20),
            boxShadow: [
              BoxShadow(
                color: Colors.black.withOpacity(_isPressed ? 0.15 : 0.08),
                blurRadius: _isPressed ? 16 : 8,
                offset: Offset(0, _isPressed ? 8 : 4),
              ),
            ],
          ),
          child: ClipRRect(
            borderRadius: BorderRadius.circular(20),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                // Image with Hero animation
                _buildImageSection(theme),

                // Info section
                _buildInfoSection(theme),
              ],
            ),
          ),
        ),
      ),
    );
  }

  Widget _buildImageSection(ThemeData theme) {
    return Stack(
      children: [
        // Hero image
        Hero(
          tag: 'product-${widget.id}',
          child: AspectRatio(
            aspectRatio: 1,
            child: Image.network(
              widget.imageUrl,
              fit: BoxFit.cover,
              loadingBuilder: (context, child, progress) {
                if (progress == null) return child;
                return Container(
                  color: Colors.grey[200],
                  child: Center(
                    child: CircularProgressIndicator(
                      value: progress.expectedTotalBytes != null
                          ? progress.cumulativeBytesLoaded /
                              progress.expectedTotalBytes!
                          : null,
                    ),
                  ),
                );
              },
            ),
          ),
        ),

        // Favorite button with animation
        Positioned(
          top: 8,
          right: 8,
          child: _AnimatedFavoriteButton(
            isFavorite: _isFavorite,
            onToggle: _toggleFavorite,
          ),
        ),
      ],
    );
  }

  Widget _buildInfoSection(ThemeData theme) {
    return Padding(
      padding: const EdgeInsets.all(12),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          // Product name with animation
          TweenAnimationBuilder<double>(
            tween: Tween(begin: 0, end: 1),
            duration: const Duration(milliseconds: 500),
            curve: Curves.easeOut,
            builder: (context, value, child) {
              return Opacity(
                opacity: value,
                child: Transform.translate(
                  offset: Offset(0, 20 * (1 - value)),
                  child: child,
                ),
              );
            },
            child: Text(
              widget.name,
              style: theme.textTheme.titleSmall?.copyWith(
                fontWeight: FontWeight.bold,
              ),
              maxLines: 2,
              overflow: TextOverflow.ellipsis,
            ),
          ),

          const SizedBox(height: 8),

          // Rating
          Row(
            children: [
              ...List.generate(5, (i) {
                return Icon(
                  i < widget.rating.floor()
                      ? Icons.star
                      : i < widget.rating
                          ? Icons.star_half
                          : Icons.star_border,
                  size: 14,
                  color: Colors.amber[700],
                );
              }),
              const SizedBox(width: 4),
              Text(
                widget.rating.toStringAsFixed(1),
                style: const TextStyle(fontSize: 11),
              ),
            ],
          ),

          const SizedBox(height: 8),

          // Price with counter animation
          _AnimatedPrice(price: widget.price),

          const SizedBox(height: 12),

          // Add to cart button
          _AnimatedCartButton(
            onPressed: () {
              ScaffoldMessenger.of(context).showSnackBar(
                SnackBar(
                  content: Text('เพิ่ม ${widget.name} ลงตะกร้า'),
                  duration: const Duration(seconds: 1),
                  behavior: SnackBarBehavior.floating,
                ),
              );
            },
          ),
        ],
      ),
    );
  }
}

// Animated Favorite Button
class _AnimatedFavoriteButton extends StatefulWidget {
  final bool isFavorite;
  final VoidCallback onToggle;

  const _AnimatedFavoriteButton({
    required this.isFavorite,
    required this.onToggle,
  });

  @override
  State<_AnimatedFavoriteButton> createState() =>
      _AnimatedFavoriteButtonState();
}

class _AnimatedFavoriteButtonState extends State<_AnimatedFavoriteButton>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _scaleAnimation;
  late Animation<Color?> _colorAnimation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 400),
    );

    _scaleAnimation = TweenSequence<double>([
      TweenSequenceItem(tween: Tween(begin: 1.0, end: 1.4), weight: 50),
      TweenSequenceItem(tween: Tween(begin: 1.4, end: 1.0), weight: 50),
    ]).animate(CurvedAnimation(parent: _controller, curve: Curves.easeInOut));

    _colorAnimation = ColorTween(
      begin: Colors.grey,
      end: Colors.red,
    ).animate(_controller);

    if (widget.isFavorite) _controller.value = 1.0;
  }

  @override
  void didUpdateWidget(_AnimatedFavoriteButton oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (widget.isFavorite != oldWidget.isFavorite) {
      if (widget.isFavorite) {
        _controller.forward(from: 0);
      } else {
        _controller.reverse();
      }
    }
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: widget.onToggle,
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
        child: AnimatedBuilder(
          animation: _controller,
          builder: (context, child) {
            return Transform.scale(
              scale: _scaleAnimation.value,
              child: Icon(
                widget.isFavorite ? Icons.favorite : Icons.favorite_border,
                color: _colorAnimation.value,
                size: 20,
              ),
            );
          },
        ),
      ),
    );
  }
}

// Animated Price Counter
class _AnimatedPrice extends StatefulWidget {
  final double price;

  const _AnimatedPrice({required this.price});

  @override
  State<_AnimatedPrice> createState() => _AnimatedPriceState();
}

class _AnimatedPriceState extends State<_AnimatedPrice>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _priceAnimation;
  double _prevPrice = 0;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 800),
    );
    _priceAnimation = Tween<double>(begin: 0, end: widget.price)
        .animate(CurvedAnimation(parent: _controller, curve: Curves.easeOut));
    _controller.forward();
  }

  @override
  void didUpdateWidget(_AnimatedPrice oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (oldWidget.price != widget.price) {
      _prevPrice = oldWidget.price;
      _priceAnimation = Tween<double>(begin: _prevPrice, end: widget.price)
          .animate(CurvedAnimation(parent: _controller, curve: Curves.easeOut));
      _controller.forward(from: 0);
    }
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _priceAnimation,
      builder: (context, child) {
        return Text(
          '฿${_priceAnimation.value.toStringAsFixed(0)}',
          style: TextStyle(
            fontSize: 18,
            fontWeight: FontWeight.bold,
            color: Theme.of(context).colorScheme.primary,
          ),
        );
      },
    );
  }
}

// Animated Cart Button
class _AnimatedCartButton extends StatefulWidget {
  final VoidCallback onPressed;

  const _AnimatedCartButton({required this.onPressed});

  @override
  State<_AnimatedCartButton> createState() => _AnimatedCartButtonState();
}

class _AnimatedCartButtonState extends State<_AnimatedCartButton>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  bool _isAdded = false;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 600),
    );
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  Future<void> _handlePress() async {
    widget.onPressed();
    setState(() => _isAdded = true);
    await _controller.forward();
    await Future.delayed(const Duration(milliseconds: 800));
    if (mounted) {
      await _controller.reverse();
      setState(() => _isAdded = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);

    return SizedBox(
      width: double.infinity,
      child: AnimatedContainer(
        duration: const Duration(milliseconds: 300),
        height: 40,
        decoration: BoxDecoration(
          color: _isAdded
              ? Colors.green
              : theme.colorScheme.primary,
          borderRadius: BorderRadius.circular(10),
        ),
        child: Material(
          color: Colors.transparent,
          child: InkWell(
            borderRadius: BorderRadius.circular(10),
            onTap: _isAdded ? null : _handlePress,
            child: Center(
              child: AnimatedSwitcher(
                duration: const Duration(milliseconds: 300),
                child: _isAdded
                    ? const Row(
                        key: ValueKey('added'),
                        mainAxisSize: MainAxisSize.min,
                        children: [
                          Icon(Icons.check, color: Colors.white, size: 18),
                          SizedBox(width: 4),
                          Text('เพิ่มแล้ว!',
                            style: TextStyle(color: Colors.white,
                              fontWeight: FontWeight.w600)),
                        ],
                      )
                    : const Row(
                        key: ValueKey('cart'),
                        mainAxisSize: MainAxisSize.min,
                        children: [
                          Icon(Icons.shopping_cart_outlined,
                            color: Colors.white, size: 18),
                          SizedBox(width: 4),
                          Text('เพิ่มลงตะกร้า',
                            style: TextStyle(color: Colors.white,
                              fontWeight: FontWeight.w600)),
                        ],
                      ),
              ),
            ),
          ),
        ),
      ),
    );
  }
}
```

### lib/screens/product_list_screen.dart

```dart
import 'package:flutter/material.dart';
import '../widgets/animated_product_card.dart';

class ProductListScreen extends StatelessWidget {
  const ProductListScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final products = [
      {'id': '1', 'name': 'Flutter Course', 'price': 1299.0,
        'imageUrl': 'https://picsum.photos/seed/flutter/400/400', 'rating': 4.8},
      {'id': '2', 'name': 'Dart Programming', 'price': 899.0,
        'imageUrl': 'https://picsum.photos/seed/dart/400/400', 'rating': 4.6},
      {'id': '3', 'name': 'Firebase Guide', 'price': 799.0,
        'imageUrl': 'https://picsum.photos/seed/firebase/400/400', 'rating': 4.7},
      {'id': '4', 'name': 'State Management', 'price': 999.0,
        'imageUrl': 'https://picsum.photos/seed/state/400/400', 'rating': 4.9},
    ];

    return Scaffold(
      appBar: AppBar(title: const Text('Animated Products')),
      body: GridView.builder(
        padding: const EdgeInsets.all(16),
        gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
          crossAxisCount: 2,
          crossAxisSpacing: 12,
          mainAxisSpacing: 12,
          childAspectRatio: 0.7,
        ),
        itemCount: products.length,
        itemBuilder: (context, index) {
          final p = products[index];
          // Stagger animation: แต่ละ card animate ทีละตัว
          return TweenAnimationBuilder<double>(
            tween: Tween(begin: 0, end: 1),
            duration: Duration(milliseconds: 400 + index * 100),
            curve: Curves.easeOutBack,
            builder: (context, value, child) {
              return Transform.scale(
                scale: value,
                child: Opacity(opacity: value, child: child),
              );
            },
            child: AnimatedProductCard(
              id: p['id'] as String,
              name: p['name'] as String,
              price: p['price'] as double,
              imageUrl: p['imageUrl'] as String,
              rating: p['rating'] as double,
              onTap: () {},
            ),
          );
        },
      ),
    );
  }
}
```

---

## สรุป Animation Types

| ประเภท | ตัวอย่าง | ความยาก |
|--------|----------|---------|
| Implicit | AnimatedContainer, AnimatedOpacity | ง่ายมาก |
| TweenAnimationBuilder | Animate ค่าใดๆ | ง่าย |
| AnimatedSwitcher | สลับ widget | ง่าย |
| Hero | ระหว่าง routes | ง่าย |
| AnimatedWidget | Custom animated widget | กลาง |
| AnimatedBuilder | Full control | กลาง |
| AnimationController | Timeline | ยาก |
| Stagger | หลาย animation ต่อเนื่อง | ยาก |

## สรุปบทที่ 30

ในบทนี้เราได้เรียนรู้:

1. **AnimationController**: vsync, duration, forward/reverse/repeat
2. **Tween และ CurvedAnimation**: value transformation และ easing
3. **AnimatedWidget**: สร้าง widget ที่ animate เอง
4. **AnimatedBuilder**: explicit animation แบบยืดหยุ่น
5. **Implicit Animations**: AnimatedContainer, AnimatedOpacity, AnimatedSwitcher
6. **Hero Animations**: transition ระหว่าง route
7. **Workshop**: Animated product card ที่สมบูรณ์

จบบทที่ 21-30 แล้ว! ในหลักสูตรต่อไปเราจะเรียน State Management, API Integration, Firebase และอีกมากมาย
