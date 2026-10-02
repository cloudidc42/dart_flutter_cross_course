# Part 47: Custom Painters

## บทนำ

Custom Painters ช่วยให้เราวาดรูปทรงซับซ้อนบน canvas ได้โดยตรง เหมาะสำหรับกราฟ ชาร์ต วิดเจ็ตที่มีรูปทรงพิเศษ หรืออื่นๆ ที่ไม่สามารถทำได้ด้วย widget มาตรฐาน

---

## 47.1 CustomPainter และ CustomPaint

### โครงสร้างพื้นฐาน

```dart
import 'package:flutter/material.dart';

// สร้าง CustomPainter class
class MyPainter extends CustomPainter {
  @override
  void paint(Canvas canvas, Size size) {
    // วาดด้วย Canvas API ที่นี่
    final paint = Paint()
      ..color = Colors.blue
      ..strokeWidth = 2
      ..style = PaintingStyle.fill;
    
    canvas.drawCircle(
      Offset(size.width / 2, size.height / 2),
      50,
      paint,
    );
  }
  
  @override
  bool shouldRepaint(MyPainter oldDelegate) => false;
}

// ใช้งานใน Widget
class DrawingPage extends StatelessWidget {
  const DrawingPage({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Custom Painter')),
      body: Center(
        child: CustomPaint(
          size: const Size(300, 300),
          painter: MyPainter(),
        ),
      ),
    );
  }
}
```

### shouldRepaint

```dart
class AnimatedPainter extends CustomPainter {
  final double progress;
  final Color color;
  
  AnimatedPainter({required this.progress, required this.color});
  
  @override
  void paint(Canvas canvas, Size size) {
    // วาดตาม progress
  }
  
  @override
  bool shouldRepaint(AnimatedPainter oldDelegate) {
    // Repaint เฉพาะเมื่อค่าเปลี่ยน
    return oldDelegate.progress != progress || 
           oldDelegate.color != color;
  }
}
```

---

## 47.2 Paint Object

### ตั้งค่า Paint

```dart
void configurePaint() {
  // Fill style
  final fillPaint = Paint()
    ..color = Colors.blue
    ..style = PaintingStyle.fill;
  
  // Stroke style
  final strokePaint = Paint()
    ..color = Colors.red
    ..style = PaintingStyle.stroke
    ..strokeWidth = 3
    ..strokeCap = StrokeCap.round    // ปลายกลม
    ..strokeJoin = StrokeJoin.round  // มุมกลม
    ..strokeMiterLimit = 4.0;
  
  // Anti-aliasing
  final smoothPaint = Paint()
    ..color = Colors.green
    ..isAntiAlias = true;
  
  // Opacity
  final opaquePaint = Paint()
    ..color = Colors.purple.withOpacity(0.5);
  
  // Blend mode
  final blendPaint = Paint()
    ..color = Colors.orange
    ..blendMode = BlendMode.multiply;
}
```

---

## 47.3 Canvas API

### วาดรูปทรงพื้นฐาน

```dart
class ShapesPainter extends CustomPainter {
  @override
  void paint(Canvas canvas, Size size) {
    final paint = Paint()
      ..color = Colors.blue
      ..style = PaintingStyle.fill;
    
    // drawLine - เส้นตรง
    canvas.drawLine(
      const Offset(10, 10),
      const Offset(100, 100),
      Paint()
        ..color = Colors.red
        ..strokeWidth = 3
        ..strokeCap = StrokeCap.round,
    );
    
    // drawRect - สี่เหลี่ยม
    canvas.drawRect(
      const Rect.fromLTWH(20, 50, 100, 60),
      Paint()..color = Colors.blue,
    );
    
    // drawRRect - สี่เหลี่ยมมนมุม
    canvas.drawRRect(
      RRect.fromRectAndRadius(
        const Rect.fromLTWH(140, 50, 100, 60),
        const Radius.circular(12),
      ),
      Paint()..color = Colors.green,
    );
    
    // drawCircle - วงกลม
    canvas.drawCircle(
      const Offset(60, 200),
      40,
      Paint()..color = Colors.orange,
    );
    
    // drawOval - วงรี
    canvas.drawOval(
      const Rect.fromLTWH(120, 160, 120, 80),
      Paint()..color = Colors.purple,
    );
    
    // drawArc - ส่วนโค้ง
    canvas.drawArc(
      const Rect.fromLTWH(20, 280, 100, 100),
      -pi / 2,  // startAngle (top)
      pi * 1.5, // sweepAngle
      false,    // useCenter
      Paint()
        ..color = Colors.teal
        ..style = PaintingStyle.stroke
        ..strokeWidth = 4,
    );
  }
  
  @override
  bool shouldRepaint(covariant CustomPainter oldDelegate) => false;
}
```

### drawPath - เส้นทางซับซ้อน

```dart
class PathPainter extends CustomPainter {
  @override
  void paint(Canvas canvas, Size size) {
    final paint = Paint()
      ..color = Colors.blue
      ..style = PaintingStyle.fill;
    
    // สร้าง Path
    final path = Path();
    
    // วาด Star
    _drawStar(canvas, Offset(size.width / 2, size.height / 2), 80, paint);
    
    // วาด Arrow
    _drawArrow(canvas, Offset(50, 200), Offset(200, 200), 
        Paint()..color = Colors.red..strokeWidth = 3);
    
    // Custom shape
    final heartPath = _createHeartPath(
      Offset(size.width / 2, size.height * 0.7),
      60,
    );
    canvas.drawPath(
      heartPath,
      Paint()..color = Colors.pink,
    );
  }
  
  void _drawStar(Canvas canvas, Offset center, double radius, Paint paint) {
    final path = Path();
    const int points = 5;
    
    for (int i = 0; i < points * 2; i++) {
      final r = i.isEven ? radius : radius * 0.4;
      final angle = (i * pi / points) - pi / 2;
      final x = center.dx + r * cos(angle);
      final y = center.dy + r * sin(angle);
      
      if (i == 0) {
        path.moveTo(x, y);
      } else {
        path.lineTo(x, y);
      }
    }
    path.close();
    canvas.drawPath(path, paint);
  }
  
  void _drawArrow(Canvas canvas, Offset start, Offset end, Paint paint) {
    canvas.drawLine(start, end, paint);
    
    // หัวลูกศร
    final angle = atan2(end.dy - start.dy, end.dx - start.dx);
    const arrowSize = 15.0;
    
    final arrowPath = Path()
      ..moveTo(end.dx, end.dy)
      ..lineTo(
        end.dx - arrowSize * cos(angle - pi / 6),
        end.dy - arrowSize * sin(angle - pi / 6),
      )
      ..lineTo(
        end.dx - arrowSize * cos(angle + pi / 6),
        end.dy - arrowSize * sin(angle + pi / 6),
      )
      ..close();
    
    canvas.drawPath(arrowPath, paint..style = PaintingStyle.fill);
  }
  
  Path _createHeartPath(Offset center, double size) {
    final path = Path();
    final x = center.dx;
    final y = center.dy;
    
    path.moveTo(x, y + size * 0.35);
    
    path.cubicTo(
      x, y, x - size, y, x - size, y - size * 0.35,
    );
    path.cubicTo(
      x - size, y - size * 0.7, x, y - size * 0.7, x, y - size * 0.35,
    );
    path.cubicTo(
      x, y - size * 0.7, x + size, y - size * 0.7, x + size, y - size * 0.35,
    );
    path.cubicTo(
      x + size, y, x, y, x, y + size * 0.35,
    );
    
    return path;
  }
  
  @override
  bool shouldRepaint(covariant CustomPainter oldDelegate) => false;
}
```

---

## 47.4 Gradients

```dart
class GradientPainter extends CustomPainter {
  @override
  void paint(Canvas canvas, Size size) {
    // Linear Gradient
    final linearGradient = LinearGradient(
      begin: Alignment.topLeft,
      end: Alignment.bottomRight,
      colors: [Colors.blue, Colors.purple, Colors.pink],
    );
    
    final linearPaint = Paint()
      ..shader = linearGradient.createShader(
        Rect.fromLTWH(0, 0, size.width, size.height / 3),
      );
    
    canvas.drawRect(
      Rect.fromLTWH(0, 0, size.width, size.height / 3),
      linearPaint,
    );
    
    // Radial Gradient
    final radialGradient = RadialGradient(
      center: Alignment.center,
      radius: 0.7,
      colors: [Colors.yellow, Colors.orange, Colors.red],
    );
    
    final radialPaint = Paint()
      ..shader = radialGradient.createShader(
        Rect.fromLTWH(0, size.height / 3, size.width, size.height / 3),
      );
    
    canvas.drawRect(
      Rect.fromLTWH(0, size.height / 3, size.width, size.height / 3),
      radialPaint,
    );
    
    // Sweep Gradient
    final sweepGradient = SweepGradient(
      colors: [Colors.red, Colors.yellow, Colors.green, Colors.blue, Colors.red],
      stops: [0.0, 0.25, 0.5, 0.75, 1.0],
    );
    
    final sweepPaint = Paint()
      ..shader = sweepGradient.createShader(
        Rect.fromLTWH(
          size.width / 4,
          size.height * 2 / 3,
          size.width / 2,
          size.height / 3,
        ),
      );
    
    canvas.drawCircle(
      Offset(size.width / 2, size.height * 5 / 6),
      size.width / 4,
      sweepPaint,
    );
  }
  
  @override
  bool shouldRepaint(covariant CustomPainter oldDelegate) => false;
}
```

---

## 47.5 Workshop: Charts และ Graphs Painter

### Pie Chart

```dart
class PieChartPainter extends CustomPainter {
  final List<PieSlice> slices;
  final int? selectedIndex;
  
  PieChartPainter({
    required this.slices,
    this.selectedIndex,
  });
  
  @override
  void paint(Canvas canvas, Size size) {
    final center = Offset(size.width / 2, size.height / 2);
    final radius = min(size.width, size.height) / 2 - 20;
    
    double startAngle = -pi / 2;
    final total = slices.fold<double>(0, (sum, s) => sum + s.value);
    
    for (int i = 0; i < slices.length; i++) {
      final slice = slices[i];
      final sweepAngle = (slice.value / total) * 2 * pi;
      final isSelected = i == selectedIndex;
      
      // ขยาย slice ที่เลือก
      var sliceCenter = center;
      if (isSelected) {
        final midAngle = startAngle + sweepAngle / 2;
        sliceCenter = Offset(
          center.dx + cos(midAngle) * 15,
          center.dy + sin(midAngle) * 15,
        );
      }
      
      final paint = Paint()
        ..color = slice.color
        ..style = PaintingStyle.fill;
      
      canvas.drawArc(
        Rect.fromCircle(
          center: sliceCenter,
          radius: isSelected ? radius + 10 : radius,
        ),
        startAngle,
        sweepAngle,
        true,
        paint,
      );
      
      // Stroke สีขาวระหว่าง slices
      canvas.drawArc(
        Rect.fromCircle(center: sliceCenter, radius: radius),
        startAngle,
        sweepAngle,
        true,
        Paint()
          ..color = Colors.white
          ..style = PaintingStyle.stroke
          ..strokeWidth = 2,
      );
      
      // Label เปอร์เซ็นต์
      final midAngle = startAngle + sweepAngle / 2;
      final labelRadius = radius * 0.7;
      final labelOffset = Offset(
        sliceCenter.dx + cos(midAngle) * labelRadius,
        sliceCenter.dy + sin(midAngle) * labelRadius,
      );
      
      _drawText(
        canvas: canvas,
        text: '${(slice.value / total * 100).toStringAsFixed(0)}%',
        offset: labelOffset,
        fontSize: 12,
        color: Colors.white,
        bold: true,
      );
      
      startAngle += sweepAngle;
    }
  }
  
  void _drawText({
    required Canvas canvas,
    required String text,
    required Offset offset,
    required double fontSize,
    required Color color,
    bool bold = false,
  }) {
    final textPainter = TextPainter(
      text: TextSpan(
        text: text,
        style: TextStyle(
          fontSize: fontSize,
          color: color,
          fontWeight: bold ? FontWeight.bold : FontWeight.normal,
        ),
      ),
      textDirection: TextDirection.ltr,
    )..layout();
    
    textPainter.paint(
      canvas,
      offset - Offset(textPainter.width / 2, textPainter.height / 2),
    );
  }
  
  @override
  bool shouldRepaint(PieChartPainter oldDelegate) {
    return oldDelegate.slices != slices || 
           oldDelegate.selectedIndex != selectedIndex;
  }
}

class PieSlice {
  final String label;
  final double value;
  final Color color;
  
  const PieSlice({
    required this.label,
    required this.value,
    required this.color,
  });
}
```

### Bar Chart

```dart
class BarChartPainter extends CustomPainter {
  final List<BarData> bars;
  final String? title;
  
  BarChartPainter({required this.bars, this.title});
  
  @override
  void paint(Canvas canvas, Size size) {
    const padding = EdgeInsets.fromLTRB(50, 30, 20, 50);
    
    final chartWidth = size.width - padding.left - padding.right;
    final chartHeight = size.height - padding.top - padding.bottom;
    
    final chartRect = Rect.fromLTWH(
      padding.left,
      padding.top,
      chartWidth,
      chartHeight,
    );
    
    final maxValue = bars.map((b) => b.value).reduce(max);
    
    // วาด Grid Lines
    _drawGridLines(canvas, chartRect, maxValue);
    
    // วาด Axes
    _drawAxes(canvas, chartRect);
    
    // วาด Bars
    _drawBars(canvas, chartRect, maxValue);
    
    // วาด Title
    if (title != null) {
      _drawText(
        canvas: canvas,
        text: title!,
        offset: Offset(size.width / 2, 10),
        fontSize: 14,
        color: Colors.black87,
        bold: true,
      );
    }
  }
  
  void _drawGridLines(Canvas canvas, Rect chartRect, double maxValue) {
    const lineCount = 5;
    final linePaint = Paint()
      ..color = Colors.grey.withOpacity(0.3)
      ..strokeWidth = 1;
    
    for (int i = 0; i <= lineCount; i++) {
      final y = chartRect.bottom - (i / lineCount) * chartRect.height;
      
      canvas.drawLine(
        Offset(chartRect.left, y),
        Offset(chartRect.right, y),
        linePaint,
      );
      
      // Label ค่าบน Y axis
      final value = (i / lineCount * maxValue).round();
      _drawText(
        canvas: canvas,
        text: value.toString(),
        offset: Offset(chartRect.left - 8, y),
        fontSize: 10,
        color: Colors.grey,
        textAlign: TextAlign.right,
      );
    }
  }
  
  void _drawAxes(Canvas canvas, Rect chartRect) {
    final axisPaint = Paint()
      ..color = Colors.black54
      ..strokeWidth = 2
      ..strokeCap = StrokeCap.round;
    
    // Y axis
    canvas.drawLine(
      Offset(chartRect.left, chartRect.top),
      Offset(chartRect.left, chartRect.bottom),
      axisPaint,
    );
    
    // X axis
    canvas.drawLine(
      Offset(chartRect.left, chartRect.bottom),
      Offset(chartRect.right, chartRect.bottom),
      axisPaint,
    );
  }
  
  void _drawBars(Canvas canvas, Rect chartRect, double maxValue) {
    final barWidth = chartRect.width / bars.length;
    const barPadding = 8.0;
    
    for (int i = 0; i < bars.length; i++) {
      final bar = bars[i];
      final barHeight = (bar.value / maxValue) * chartRect.height;
      
      final barLeft = chartRect.left + i * barWidth + barPadding;
      final barRight = barLeft + barWidth - barPadding * 2;
      final barTop = chartRect.bottom - barHeight;
      
      // วาด bar พร้อม gradient
      final gradient = LinearGradient(
        begin: Alignment.topCenter,
        end: Alignment.bottomCenter,
        colors: [bar.color.withOpacity(0.8), bar.color],
      );
      
      final barRect = RRect.fromRectAndCorners(
        Rect.fromLTRB(barLeft, barTop, barRight, chartRect.bottom),
        topLeft: const Radius.circular(4),
        topRight: const Radius.circular(4),
      );
      
      canvas.drawRRect(
        barRect,
        Paint()..shader = gradient.createShader(barRect.outerRect),
      );
      
      // ค่าบนบาร์
      _drawText(
        canvas: canvas,
        text: bar.value.round().toString(),
        offset: Offset((barLeft + barRight) / 2, barTop - 14),
        fontSize: 11,
        color: Colors.black87,
        bold: true,
      );
      
      // Label ด้านล่าง
      _drawText(
        canvas: canvas,
        text: bar.label,
        offset: Offset(
          (barLeft + barRight) / 2,
          chartRect.bottom + 14,
        ),
        fontSize: 10,
        color: Colors.black54,
      );
    }
  }
  
  void _drawText({
    required Canvas canvas,
    required String text,
    required Offset offset,
    required double fontSize,
    required Color color,
    bool bold = false,
    TextAlign textAlign = TextAlign.center,
  }) {
    final textPainter = TextPainter(
      text: TextSpan(
        text: text,
        style: TextStyle(
          fontSize: fontSize,
          color: color,
          fontWeight: bold ? FontWeight.bold : FontWeight.normal,
        ),
      ),
      textDirection: TextDirection.ltr,
      textAlign: textAlign,
    )..layout();
    
    Offset position;
    switch (textAlign) {
      case TextAlign.right:
        position = Offset(
          offset.dx - textPainter.width,
          offset.dy - textPainter.height / 2,
        );
        break;
      default:
        position = Offset(
          offset.dx - textPainter.width / 2,
          offset.dy - textPainter.height / 2,
        );
    }
    
    textPainter.paint(canvas, position);
  }
  
  @override
  bool shouldRepaint(BarChartPainter oldDelegate) {
    return oldDelegate.bars != bars;
  }
}

class BarData {
  final String label;
  final double value;
  final Color color;
  
  const BarData({
    required this.label,
    required this.value,
    required this.color,
  });
}
```

### Line Chart

```dart
class LineChartPainter extends CustomPainter {
  final List<LinePoint> points;
  final Color lineColor;
  final bool showArea;
  final bool showDots;
  
  LineChartPainter({
    required this.points,
    this.lineColor = Colors.blue,
    this.showArea = true,
    this.showDots = true,
  });
  
  @override
  void paint(Canvas canvas, Size size) {
    if (points.isEmpty) return;
    
    const padding = EdgeInsets.fromLTRB(50, 20, 20, 50);
    
    final chartWidth = size.width - padding.left - padding.right;
    final chartHeight = size.height - padding.top - padding.bottom;
    
    final maxValue = points.map((p) => p.value).reduce(max);
    final minValue = points.map((p) => p.value).reduce(min);
    final valueRange = maxValue - minValue;
    
    Offset toCanvas(int index, double value) {
      final x = padding.left + (index / (points.length - 1)) * chartWidth;
      final y = padding.top + chartHeight - 
          ((value - minValue) / valueRange) * chartHeight;
      return Offset(x, y);
    }
    
    // สร้าง Path สำหรับเส้น
    final linePath = Path();
    for (int i = 0; i < points.length; i++) {
      final pos = toCanvas(i, points[i].value);
      if (i == 0) {
        linePath.moveTo(pos.dx, pos.dy);
      } else {
        // Smooth curve
        final prev = toCanvas(i - 1, points[i - 1].value);
        final cp1x = prev.dx + (pos.dx - prev.dx) / 3;
        final cp2x = pos.dx - (pos.dx - prev.dx) / 3;
        linePath.cubicTo(cp1x, prev.dy, cp2x, pos.dy, pos.dx, pos.dy);
      }
    }
    
    // วาด Area ใต้เส้น
    if (showArea) {
      final areaPath = Path.from(linePath)
        ..lineTo(padding.left + chartWidth, padding.top + chartHeight)
        ..lineTo(padding.left, padding.top + chartHeight)
        ..close();
      
      final areaGradient = LinearGradient(
        begin: Alignment.topCenter,
        end: Alignment.bottomCenter,
        colors: [
          lineColor.withOpacity(0.3),
          lineColor.withOpacity(0.0),
        ],
      );
      
      canvas.drawPath(
        areaPath,
        Paint()..shader = areaGradient.createShader(
          Rect.fromLTWH(padding.left, padding.top, chartWidth, chartHeight),
        ),
      );
    }
    
    // วาดเส้น
    canvas.drawPath(
      linePath,
      Paint()
        ..color = lineColor
        ..style = PaintingStyle.stroke
        ..strokeWidth = 2.5
        ..strokeCap = StrokeCap.round
        ..strokeJoin = StrokeJoin.round,
    );
    
    // วาด Dots
    if (showDots) {
      for (int i = 0; i < points.length; i++) {
        final pos = toCanvas(i, points[i].value);
        
        canvas.drawCircle(pos, 4, Paint()..color = Colors.white);
        canvas.drawCircle(
          pos, 3,
          Paint()..color = lineColor,
        );
      }
    }
  }
  
  @override
  bool shouldRepaint(LineChartPainter oldDelegate) {
    return oldDelegate.points != points;
  }
}

class LinePoint {
  final String label;
  final double value;
  
  const LinePoint({required this.label, required this.value});
}
```

### ใช้งาน Charts

```dart
class ChartsPage extends StatelessWidget {
  const ChartsPage({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Charts และ Graphs')),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          // Pie Chart
          const Text(
            'ส่วนแบ่งการตลาด',
            style: TextStyle(fontSize: 16, fontWeight: FontWeight.bold),
          ),
          const SizedBox(height: 8),
          SizedBox(
            height: 250,
            child: CustomPaint(
              painter: PieChartPainter(
                slices: [
                  PieSlice(label: 'Android', value: 72, color: Colors.green),
                  PieSlice(label: 'iOS', value: 27, color: Colors.blue),
                  PieSlice(label: 'Other', value: 1, color: Colors.grey),
                ],
              ),
            ),
          ),
          
          const SizedBox(height: 32),
          
          // Bar Chart
          const Text(
            'ยอดขายรายเดือน',
            style: TextStyle(fontSize: 16, fontWeight: FontWeight.bold),
          ),
          const SizedBox(height: 8),
          SizedBox(
            height: 250,
            child: CustomPaint(
              painter: BarChartPainter(
                bars: [
                  BarData(label: 'ม.ค.', value: 1200, color: Colors.blue),
                  BarData(label: 'ก.พ.', value: 1800, color: Colors.blue),
                  BarData(label: 'มี.ค.', value: 1500, color: Colors.blue),
                  BarData(label: 'เม.ย.', value: 2200, color: Colors.orange),
                  BarData(label: 'พ.ค.', value: 1900, color: Colors.blue),
                  BarData(label: 'มิ.ย.', value: 2500, color: Colors.blue),
                ],
              ),
            ),
          ),
          
          const SizedBox(height: 32),
          
          // Line Chart
          const Text(
            'อุณหภูมิ 7 วัน',
            style: TextStyle(fontSize: 16, fontWeight: FontWeight.bold),
          ),
          const SizedBox(height: 8),
          SizedBox(
            height: 200,
            child: CustomPaint(
              size: const Size(double.infinity, 200),
              painter: LineChartPainter(
                points: [
                  const LinePoint(label: 'จ', value: 28),
                  const LinePoint(label: 'อ', value: 31),
                  const LinePoint(label: 'พ', value: 29),
                  const LinePoint(label: 'พฤ', value: 33),
                  const LinePoint(label: 'ศ', value: 32),
                  const LinePoint(label: 'ส', value: 30),
                  const LinePoint(label: 'อา', value: 27),
                ],
                lineColor: Colors.orange,
                showArea: true,
              ),
            ),
          ),
        ],
      ),
    );
  }
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- โครงสร้างของ CustomPainter และ CustomPaint
- Canvas API: drawLine, drawRect, drawCircle, drawPath
- Paint object configuration
- Gradients: Linear, Radial, Sweep
- Workshop: Pie Chart, Bar Chart, Line Chart

**แบบฝึกหัดเพิ่มเติม:**
1. สร้าง Donut Chart (วงกลมแบบโดนัท)
2. Animate Bar Chart เมื่อ page แสดง
3. สร้าง Radar/Spider Chart
4. เพิ่ม touch interaction บน Charts
