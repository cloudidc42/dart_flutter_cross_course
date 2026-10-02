# Part 72: Custom Render Objects - สร้าง Render Objects เอง

## บทนำ

RenderObject คือชั้นล่างสุดของ Flutter's rendering pipeline ที่ทำหน้าที่วาด (paint) และ layout widget จริงๆ การเข้าใจ RenderObject ช่วยให้เราสร้าง widget ที่มีประสิทธิภาพสูงและมีความยืดหยุ่นสูงสุด

## 1. RenderObject Basics

### Flutter Rendering Pipeline

```
Widget Tree → Element Tree → RenderObject Tree
```

- **Widget**: Blueprint/Configuration (immutable)
- **Element**: Widget ที่ถูก instantiate และมี lifecycle
- **RenderObject**: ทำงานจริง - layout, paint, hit-test

### RenderObject Class Hierarchy

```
RenderObject
├── RenderBox (ใช้ BoxConstraints)
│   ├── RenderProxyBox
│   │   ├── RenderDecoratedBox
│   │   └── RenderPadding
│   └── RenderShiftedBox
│       └── RenderPositionedBox
└── RenderSliver (สำหรับ scrollable)
```

### สร้าง Simple RenderBox

```dart
// lib/render/render_colored_box.dart
import 'package:flutter/rendering.dart';

class RenderColoredBox extends RenderBox {
  Color _color;

  RenderColoredBox({required Color color}) : _color = color;

  // Getter และ Setter พร้อม markNeedsPaint
  Color get color => _color;
  set color(Color value) {
    if (_color == value) return;
    _color = value;
    markNeedsPaint(); // บอก Flutter ว่าต้อง repaint
  }

  @override
  void performLayout() {
    // กำหนด size ของ render object
    // ถ้า parent ให้ tight constraints ใช้ตามนั้น
    size = constraints.biggest;
  }

  @override
  void paint(PaintingContext context, Offset offset) {
    // วาดสี่เหลี่ยมสี
    context.canvas.drawRect(
      offset & size, // Rect จาก offset และ size
      Paint()..color = _color,
    );
  }
}
```

### Widget ที่ใช้ RenderObject

```dart
// lib/render/colored_box_widget.dart
import 'package:flutter/material.dart';
import 'package:flutter/rendering.dart';
import 'render_colored_box.dart';

class CustomColoredBox extends LeafRenderObjectWidget {
  final Color color;

  const CustomColoredBox({
    super.key,
    required this.color,
  });

  @override
  RenderColoredBox createRenderObject(BuildContext context) {
    return RenderColoredBox(color: color);
  }

  @override
  void updateRenderObject(
    BuildContext context,
    RenderColoredBox renderObject,
  ) {
    // อัปเดต RenderObject เมื่อ Widget เปลี่ยน
    renderObject.color = color;
  }
}
```

## 2. Custom Layout Algorithms

### สร้าง Custom Layout RenderObject

```dart
// lib/render/render_circular_layout.dart
import 'package:flutter/rendering.dart';
import 'dart:math' as math;

class CircularLayoutParentData extends ContainerBoxParentData<RenderBox> {
  double angle = 0;
}

class RenderCircularLayout extends RenderBox
    with
        ContainerRenderObjectMixin<RenderBox, CircularLayoutParentData>,
        RenderBoxContainerDefaultsMixin<RenderBox, CircularLayoutParentData> {
  
  double _radius;
  double _startAngle;

  RenderCircularLayout({
    double radius = 100,
    double startAngle = -math.pi / 2,
  })  : _radius = radius,
        _startAngle = startAngle;

  double get radius => _radius;
  set radius(double value) {
    if (_radius == value) return;
    _radius = value;
    markNeedsLayout();
  }

  double get startAngle => _startAngle;
  set startAngle(double value) {
    if (_startAngle == value) return;
    _startAngle = value;
    markNeedsLayout();
  }

  @override
  void setupParentData(RenderBox child) {
    if (child.parentData is! CircularLayoutParentData) {
      child.parentData = CircularLayoutParentData();
    }
  }

  @override
  void performLayout() {
    // คำนวณ size รวม
    final diameter = _radius * 2 + 60;
    size = constraints.constrain(Size(diameter, diameter));

    final childCount = this.childCount;
    if (childCount == 0) return;

    final angleStep = (2 * math.pi) / childCount;
    final center = Offset(size.width / 2, size.height / 2);

    RenderBox? child = firstChild;
    int index = 0;

    while (child != null) {
      final childParentData = child.parentData as CircularLayoutParentData;

      // Layout แต่ละ child
      child.layout(
        const BoxConstraints(),
        parentUsesSize: true,
      );

      // คำนวณ position บน circle
      final angle = _startAngle + angleStep * index;
      childParentData.angle = angle;

      final childCenter = Offset(
        center.dx + _radius * math.cos(angle),
        center.dy + _radius * math.sin(angle),
      );

      childParentData.offset = Offset(
        childCenter.dx - child.size.width / 2,
        childCenter.dy - child.size.height / 2,
      );

      child = childParentData.nextSibling;
      index++;
    }
  }

  @override
  void paint(PaintingContext context, Offset offset) {
    // วาด children
    defaultPaint(context, offset);
  }

  @override
  bool hitTestChildren(BoxHitTestResult result, {required Offset position}) {
    return defaultHitTestChildren(result, position: position);
  }
}
```

### Widget สำหรับ Circular Layout

```dart
// lib/render/circular_layout.dart
import 'package:flutter/material.dart';
import 'package:flutter/rendering.dart';
import 'render_circular_layout.dart';

class CircularLayout extends MultiChildRenderObjectWidget {
  final double radius;
  final double startAngle;

  const CircularLayout({
    super.key,
    super.children,
    this.radius = 100,
    this.startAngle = -3.14159 / 2, // -π/2 = top
  });

  @override
  RenderCircularLayout createRenderObject(BuildContext context) {
    return RenderCircularLayout(
      radius: radius,
      startAngle: startAngle,
    );
  }

  @override
  void updateRenderObject(
    BuildContext context,
    RenderCircularLayout renderObject,
  ) {
    renderObject
      ..radius = radius
      ..startAngle = startAngle;
  }
}
```

## 3. SingleChildRenderObjectWidget

### สร้าง Custom Clip Widget

```dart
// lib/render/custom_clip_widget.dart
import 'package:flutter/material.dart';
import 'package:flutter/rendering.dart';

// Custom RenderObject ที่ clip เป็นรูปดาว
class RenderStarClip extends RenderProxyBox {
  int _points;
  double _innerRadiusRatio;

  RenderStarClip({
    int points = 5,
    double innerRadiusRatio = 0.4,
  })  : _points = points,
        _innerRadiusRatio = innerRadiusRatio;

  int get points => _points;
  set points(int value) {
    if (_points == value) return;
    _points = value;
    markNeedsPaint();
  }

  double get innerRadiusRatio => _innerRadiusRatio;
  set innerRadiusRatio(double value) {
    if (_innerRadiusRatio == value) return;
    _innerRadiusRatio = value;
    markNeedsPaint();
  }

  Path _createStarPath() {
    final center = Offset(size.width / 2, size.height / 2);
    final outerRadius = size.shortestSide / 2;
    final innerRadius = outerRadius * _innerRadiusRatio;
    final angleStep = 3.14159 / _points;

    final path = Path();
    for (int i = 0; i < _points * 2; i++) {
      final radius = i.isEven ? outerRadius : innerRadius;
      final angle = angleStep * i - 3.14159 / 2;
      final x = center.dx + radius * cos(angle);
      final y = center.dy + radius * sin(angle);

      if (i == 0) {
        path.moveTo(x, y);
      } else {
        path.lineTo(x, y);
      }
    }
    path.close();
    return path;
  }

  double cos(double radians) => _cosine(radians);
  double sin(double radians) => _sine(radians);

  double _cosine(double r) {
    import 'dart:math' as math;
    return math.cos(r);
  }

  double _sine(double r) {
    import 'dart:math' as math;
    return math.sin(r);
  }

  @override
  void paint(PaintingContext context, Offset offset) {
    // Clip เป็นรูปดาวก่อนวาด child
    context.canvas.save();
    context.canvas.translate(offset.dx, offset.dy);
    context.canvas.clipPath(_createStarPath());
    context.canvas.translate(-offset.dx, -offset.dy);
    
    super.paint(context, offset);
    context.canvas.restore();
  }
}

class StarClip extends SingleChildRenderObjectWidget {
  final int points;
  final double innerRadiusRatio;

  const StarClip({
    super.key,
    super.child,
    this.points = 5,
    this.innerRadiusRatio = 0.4,
  });

  @override
  RenderBox createRenderObject(BuildContext context) {
    return _RenderStarClipFixed(
      points: points,
      innerRadiusRatio: innerRadiusRatio,
    );
  }

  @override
  void updateRenderObject(
    BuildContext context,
    covariant _RenderStarClipFixed renderObject,
  ) {
    renderObject
      ..points = points
      ..innerRadiusRatio = innerRadiusRatio;
  }
}

class _RenderStarClipFixed extends RenderProxyBox {
  int _points;
  double _innerRadiusRatio;

  _RenderStarClipFixed({
    required int points,
    required double innerRadiusRatio,
  })  : _points = points,
        _innerRadiusRatio = innerRadiusRatio;

  int get points => _points;
  set points(int value) {
    if (_points == value) return;
    _points = value;
    markNeedsPaint();
  }

  double get innerRadiusRatio => _innerRadiusRatio;
  set innerRadiusRatio(double value) {
    if (_innerRadiusRatio == value) return;
    _innerRadiusRatio = value;
    markNeedsPaint();
  }

  @override
  void paint(PaintingContext context, Offset offset) {
    import 'dart:math' as math;
    final center = Offset(
      offset.dx + size.width / 2,
      offset.dy + size.height / 2,
    );
    final outerRadius = size.shortestSide / 2;
    final innerRadius = outerRadius * _innerRadiusRatio;
    final angleStep = math.pi / _points;

    final path = Path();
    for (int i = 0; i < _points * 2; i++) {
      final radius = i.isEven ? outerRadius : innerRadius;
      final angle = angleStep * i - math.pi / 2;
      final x = center.dx + radius * math.cos(angle);
      final y = center.dy + radius * math.sin(angle);

      if (i == 0) {
        path.moveTo(x, y);
      } else {
        path.lineTo(x, y);
      }
    }
    path.close();

    context.canvas.save();
    context.canvas.clipPath(path);
    super.paint(context, offset);
    context.canvas.restore();
  }
}
```

## 4. Workshop: Custom Layout Widget

### สร้าง WaterfallLayout (Masonry Grid)

```dart
// lib/workshop/waterfall_layout.dart
import 'package:flutter/rendering.dart';
import 'package:flutter/material.dart';

class WaterfallParentData extends ContainerBoxParentData<RenderBox> {}

class RenderWaterfallLayout extends RenderBox
    with
        ContainerRenderObjectMixin<RenderBox, WaterfallParentData>,
        RenderBoxContainerDefaultsMixin<RenderBox, WaterfallParentData> {
  
  int _columnCount;
  double _columnSpacing;
  double _rowSpacing;

  RenderWaterfallLayout({
    int columnCount = 2,
    double columnSpacing = 8,
    double rowSpacing = 8,
  })  : _columnCount = columnCount,
        _columnSpacing = columnSpacing,
        _rowSpacing = rowSpacing;

  int get columnCount => _columnCount;
  set columnCount(int value) {
    if (_columnCount == value) return;
    _columnCount = value;
    markNeedsLayout();
  }

  double get columnSpacing => _columnSpacing;
  set columnSpacing(double value) {
    if (_columnSpacing == value) return;
    _columnSpacing = value;
    markNeedsLayout();
  }

  double get rowSpacing => _rowSpacing;
  set rowSpacing(double value) {
    if (_rowSpacing == value) return;
    _rowSpacing = value;
    markNeedsLayout();
  }

  @override
  void setupParentData(RenderBox child) {
    if (child.parentData is! WaterfallParentData) {
      child.parentData = WaterfallParentData();
    }
  }

  @override
  void performLayout() {
    if (childCount == 0) {
      size = constraints.smallest;
      return;
    }

    final totalSpacing = _columnSpacing * (_columnCount - 1);
    final columnWidth =
        (constraints.maxWidth - totalSpacing) / _columnCount;

    // Track ความสูงของแต่ละ column
    final columnHeights = List<double>.filled(_columnCount, 0);

    RenderBox? child = firstChild;
    while (child != null) {
      final childParentData = child.parentData as WaterfallParentData;

      // Layout child ด้วย column width
      child.layout(
        BoxConstraints(
          minWidth: columnWidth,
          maxWidth: columnWidth,
        ),
        parentUsesSize: true,
      );

      // หา column ที่สั้นที่สุด
      int shortestColumn = 0;
      double shortestHeight = columnHeights[0];
      for (int i = 1; i < _columnCount; i++) {
        if (columnHeights[i] < shortestHeight) {
          shortestHeight = columnHeights[i];
          shortestColumn = i;
        }
      }

      // วาง child ใน column ที่สั้นที่สุด
      final x = shortestColumn * (columnWidth + _columnSpacing);
      final y = columnHeights[shortestColumn];
      childParentData.offset = Offset(x, y);

      // อัปเดตความสูงของ column
      columnHeights[shortestColumn] += child.size.height + _rowSpacing;

      child = childParentData.nextSibling;
    }

    // คำนวณ total height
    final maxHeight = columnHeights.reduce(
      (a, b) => a > b ? a : b,
    );

    size = constraints.constrain(
      Size(constraints.maxWidth, maxHeight - _rowSpacing),
    );
  }

  @override
  void paint(PaintingContext context, Offset offset) {
    defaultPaint(context, offset);
  }

  @override
  bool hitTestChildren(BoxHitTestResult result, {required Offset position}) {
    return defaultHitTestChildren(result, position: position);
  }
}

// Widget wrapper
class WaterfallLayout extends MultiChildRenderObjectWidget {
  final int columnCount;
  final double columnSpacing;
  final double rowSpacing;

  const WaterfallLayout({
    super.key,
    super.children,
    this.columnCount = 2,
    this.columnSpacing = 8,
    this.rowSpacing = 8,
  });

  @override
  RenderWaterfallLayout createRenderObject(BuildContext context) {
    return RenderWaterfallLayout(
      columnCount: columnCount,
      columnSpacing: columnSpacing,
      rowSpacing: rowSpacing,
    );
  }

  @override
  void updateRenderObject(
    BuildContext context,
    RenderWaterfallLayout renderObject,
  ) {
    renderObject
      ..columnCount = columnCount
      ..columnSpacing = columnSpacing
      ..rowSpacing = rowSpacing;
  }
}
```

### Demo App สำหรับ WaterfallLayout

```dart
// lib/workshop/waterfall_demo.dart
import 'package:flutter/material.dart';
import 'dart:math' as math;
import 'waterfall_layout.dart';

class WaterfallDemo extends StatelessWidget {
  const WaterfallDemo({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Waterfall Layout'),
        actions: [
          IconButton(
            icon: const Icon(Icons.refresh),
            onPressed: () {},
          ),
        ],
      ),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(8),
        child: WaterfallLayout(
          columnCount: 2,
          columnSpacing: 8,
          rowSpacing: 8,
          children: _generateCards(),
        ),
      ),
    );
  }

  List<Widget> _generateCards() {
    final random = math.Random(42);
    final colors = [
      Colors.red[100]!,
      Colors.blue[100]!,
      Colors.green[100]!,
      Colors.orange[100]!,
      Colors.purple[100]!,
    ];

    return List.generate(20, (index) {
      final height = 100.0 + random.nextDouble() * 150;
      final color = colors[index % colors.length];

      return Card(
        color: color,
        child: SizedBox(
          height: height,
          child: Padding(
            padding: const EdgeInsets.all(12),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(
                  'Card ${index + 1}',
                  style: const TextStyle(
                    fontWeight: FontWeight.bold,
                    fontSize: 16,
                  ),
                ),
                const SizedBox(height: 8),
                Text(
                  'ความสูง: ${height.toStringAsFixed(0)}px',
                  style: const TextStyle(fontSize: 12),
                ),
              ],
            ),
          ),
        ),
      );
    });
  }
}
```

### Animated Custom Render Object

```dart
// lib/workshop/animated_progress_ring.dart
import 'package:flutter/material.dart';
import 'package:flutter/rendering.dart';
import 'dart:math' as math;

class RenderProgressRing extends RenderBox {
  double _progress;
  Color _foregroundColor;
  Color _backgroundColor;
  double _strokeWidth;

  RenderProgressRing({
    double progress = 0,
    Color foregroundColor = Colors.blue,
    Color backgroundColor = Colors.grey,
    double strokeWidth = 8,
  })  : _progress = progress.clamp(0, 1),
        _foregroundColor = foregroundColor,
        _backgroundColor = backgroundColor,
        _strokeWidth = strokeWidth;

  double get progress => _progress;
  set progress(double value) {
    final clamped = value.clamp(0.0, 1.0);
    if (_progress == clamped) return;
    _progress = clamped;
    markNeedsPaint();
  }

  Color get foregroundColor => _foregroundColor;
  set foregroundColor(Color value) {
    if (_foregroundColor == value) return;
    _foregroundColor = value;
    markNeedsPaint();
  }

  Color get backgroundColor => _backgroundColor;
  set backgroundColor(Color value) {
    if (_backgroundColor == value) return;
    _backgroundColor = value;
    markNeedsPaint();
  }

  double get strokeWidth => _strokeWidth;
  set strokeWidth(double value) {
    if (_strokeWidth == value) return;
    _strokeWidth = value;
    markNeedsPaint();
  }

  @override
  void performLayout() {
    size = constraints.constrain(const Size(100, 100));
  }

  @override
  void paint(PaintingContext context, Offset offset) {
    final canvas = context.canvas;
    final center = Offset(
      offset.dx + size.width / 2,
      offset.dy + size.height / 2,
    );
    final radius = (size.shortestSide - _strokeWidth) / 2;

    // Background arc
    canvas.drawArc(
      Rect.fromCircle(center: center, radius: radius),
      0,
      2 * math.pi,
      false,
      Paint()
        ..color = _backgroundColor
        ..style = PaintingStyle.stroke
        ..strokeWidth = _strokeWidth
        ..strokeCap = StrokeCap.round,
    );

    // Progress arc
    if (_progress > 0) {
      canvas.drawArc(
        Rect.fromCircle(center: center, radius: radius),
        -math.pi / 2, // เริ่มจากด้านบน
        2 * math.pi * _progress,
        false,
        Paint()
          ..color = _foregroundColor
          ..style = PaintingStyle.stroke
          ..strokeWidth = _strokeWidth
          ..strokeCap = StrokeCap.round,
      );
    }

    // วาดเปอร์เซ็นต์ตรงกลาง
    final textPainter = TextPainter(
      text: TextSpan(
        text: '${(_progress * 100).toInt()}%',
        style: TextStyle(
          color: _foregroundColor,
          fontSize: size.width * 0.2,
          fontWeight: FontWeight.bold,
        ),
      ),
      textDirection: TextDirection.ltr,
    );
    textPainter.layout();
    textPainter.paint(
      canvas,
      Offset(
        center.dx - textPainter.width / 2,
        center.dy - textPainter.height / 2,
      ),
    );
  }
}

class ProgressRing extends LeafRenderObjectWidget {
  final double progress;
  final Color foregroundColor;
  final Color backgroundColor;
  final double strokeWidth;

  const ProgressRing({
    super.key,
    required this.progress,
    this.foregroundColor = Colors.blue,
    this.backgroundColor = const Color(0xFFE0E0E0),
    this.strokeWidth = 8,
  });

  @override
  RenderProgressRing createRenderObject(BuildContext context) {
    return RenderProgressRing(
      progress: progress,
      foregroundColor: foregroundColor,
      backgroundColor: backgroundColor,
      strokeWidth: strokeWidth,
    );
  }

  @override
  void updateRenderObject(
    BuildContext context,
    RenderProgressRing renderObject,
  ) {
    renderObject
      ..progress = progress
      ..foregroundColor = foregroundColor
      ..backgroundColor = backgroundColor
      ..strokeWidth = strokeWidth;
  }
}

// Demo screen
class RenderObjectDemo extends StatefulWidget {
  const RenderObjectDemo({super.key});

  @override
  State<RenderObjectDemo> createState() => _RenderObjectDemoState();
}

class _RenderObjectDemoState extends State<RenderObjectDemo>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 3),
    )..repeat();
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Custom Render Objects')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            AnimatedBuilder(
              animation: _controller,
              builder: (context, child) {
                return SizedBox(
                  width: 150,
                  height: 150,
                  child: ProgressRing(
                    progress: _controller.value,
                    foregroundColor: Colors.blue,
                    strokeWidth: 12,
                  ),
                );
              },
            ),
            const SizedBox(height: 32),
            const Text(
              'Custom ProgressRing\nสร้างด้วย RenderObject',
              textAlign: TextAlign.center,
            ),
          ],
        ),
      ),
    );
  }
}
```

## สรุป

การสร้าง Custom RenderObjects ช่วยให้:
1. **ประสิทธิภาพสูงสุด** - ไม่มี widget tree overhead
2. **Control ทุกอย่าง** - layout, paint, hit-test
3. **Reusable** - สร้างครั้งเดียวใช้ได้ทุกที่
4. **Animate ได้** - ทำงานร่วมกับ AnimationController ได้ดี

## แบบทดสอบ

1. อธิบายความแตกต่างระหว่าง `markNeedsLayout()` และ `markNeedsPaint()`
2. ทำไม `RenderBox` จึง override `performLayout()` แต่ไม่ใช่ `layout()`?
3. สร้าง `RenderHexagonGrid` ที่จัด children เป็น hexagonal pattern
4. เมื่อไหร่ควรใช้ `CustomPaint` แทนที่จะสร้าง `RenderObject` เอง?
