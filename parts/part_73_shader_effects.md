# Part 73: Shader Effects - เอฟเฟกต์ด้วย Shaders

## บทนำ

Fragment Shaders ใน Flutter ช่วยให้เราสร้างเอฟเฟกต์กราฟิกที่ซับซ้อนได้โดยตรงบน GPU ตั้งแต่ Flutter 3.7 เป็นต้นมา เราสามารถเขียน GLSL shaders และใช้งานกับ Flutter widgets ได้

## 1. Flutter Shaders

### ทำความเข้าใจ Shader Pipeline

```
CPU                   GPU
Widget → CustomPaint → Fragment Shader → Pixel Color
```

Fragment Shader ทำงานบน GPU และรันสำหรับทุก pixel บนหน้าจอ ทำให้มีประสิทธิภาพสูงมาก

### การตั้งค่า pubspec.yaml

```yaml
# pubspec.yaml
flutter:
  shaders:
    - shaders/gradient.frag
    - shaders/glassmorphism.frag
    - shaders/ripple.frag
```

## 2. Fragment Shaders (GLSL)

### Shader พื้นฐาน

```glsl
// shaders/gradient.frag
#include <flutter/runtime_effect.glsl>

uniform vec2 uSize;
uniform float uTime;

out vec4 fragColor;

void main() {
  // FlutterFragCoord() คือ position ของ pixel ใน canvas space
  vec2 fragCoord = FlutterFragCoord().xy;
  
  // Normalize coordinates (0 ถึง 1)
  vec2 uv = fragCoord / uSize;
  
  // Simple gradient สีแดง → น้ำเงิน
  vec3 color = mix(
    vec3(1.0, 0.0, 0.0),  // สีแดง
    vec3(0.0, 0.0, 1.0),  // สีน้ำเงิน
    uv.x
  );
  
  fragColor = vec4(color, 1.0);
}
```

### Animated Shader

```glsl
// shaders/wave.frag
#include <flutter/runtime_effect.glsl>

uniform vec2 uSize;
uniform float uTime;
uniform vec4 uColor;

out vec4 fragColor;

void main() {
  vec2 fragCoord = FlutterFragCoord().xy;
  vec2 uv = fragCoord / uSize;
  
  // คลื่น sine
  float wave = sin(uv.x * 10.0 + uTime * 3.0) * 0.05;
  float y = uv.y + wave;
  
  // สร้างสีตาม wave
  float brightness = smoothstep(0.5 - 0.1, 0.5 + 0.1, y);
  
  vec3 color1 = uColor.rgb;
  vec3 color2 = vec3(1.0);
  vec3 finalColor = mix(color1, color2, brightness);
  
  fragColor = vec4(finalColor, uColor.a);
}
```

### ใช้ Shader ใน Flutter

```dart
// lib/shaders/gradient_widget.dart
import 'dart:ui' as ui;
import 'package:flutter/material.dart';

class GradientShaderWidget extends StatelessWidget {
  const GradientShaderWidget({super.key});

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<ui.FragmentShader>(
      future: _loadShader(),
      builder: (context, snapshot) {
        if (!snapshot.hasData) {
          return const CircularProgressIndicator();
        }

        return CustomPaint(
          painter: _GradientPainter(shader: snapshot.data!),
          child: const SizedBox(
            width: double.infinity,
            height: 200,
          ),
        );
      },
    );
  }

  Future<ui.FragmentShader> _loadShader() async {
    final program = await ui.FragmentProgram.fromAsset('shaders/gradient.frag');
    return program.fragmentShader();
  }
}

class _GradientPainter extends CustomPainter {
  final ui.FragmentShader shader;

  _GradientPainter({required this.shader});

  @override
  void paint(Canvas canvas, Size size) {
    // Set shader uniforms
    shader.setFloat(0, size.width);  // uSize.x
    shader.setFloat(1, size.height); // uSize.y

    // วาดด้วย shader
    canvas.drawRect(
      Rect.fromLTWH(0, 0, size.width, size.height),
      Paint()..shader = shader,
    );
  }

  @override
  bool shouldRepaint(covariant _GradientPainter oldDelegate) => false;
}
```

### Animated Shader Widget

```dart
// lib/shaders/animated_wave.dart
import 'dart:ui' as ui;
import 'package:flutter/material.dart';
import 'package:flutter/scheduler.dart';

class AnimatedWaveWidget extends StatefulWidget {
  final Color color;
  final double height;

  const AnimatedWaveWidget({
    super.key,
    this.color = Colors.blue,
    this.height = 200,
  });

  @override
  State<AnimatedWaveWidget> createState() => _AnimatedWaveWidgetState();
}

class _AnimatedWaveWidgetState extends State<AnimatedWaveWidget> {
  ui.FragmentShader? _shader;
  late Ticker _ticker;
  double _time = 0;

  @override
  void initState() {
    super.initState();
    _loadShader();
    _ticker = Ticker((elapsed) {
      setState(() {
        _time = elapsed.inMilliseconds / 1000.0;
      });
    });
    _ticker.start();
  }

  Future<void> _loadShader() async {
    final program = await ui.FragmentProgram.fromAsset('shaders/wave.frag');
    if (mounted) {
      setState(() {
        _shader = program.fragmentShader();
      });
    }
  }

  @override
  void dispose() {
    _ticker.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    if (_shader == null) {
      return SizedBox(height: widget.height);
    }

    return CustomPaint(
      painter: _WavePainter(
        shader: _shader!,
        time: _time,
        color: widget.color,
      ),
      size: Size(double.infinity, widget.height),
    );
  }
}

class _WavePainter extends CustomPainter {
  final ui.FragmentShader shader;
  final double time;
  final Color color;

  _WavePainter({
    required this.shader,
    required this.time,
    required this.color,
  });

  @override
  void paint(Canvas canvas, Size size) {
    shader.setFloat(0, size.width);
    shader.setFloat(1, size.height);
    shader.setFloat(2, time);
    shader.setFloat(3, color.red / 255.0);
    shader.setFloat(4, color.green / 255.0);
    shader.setFloat(5, color.blue / 255.0);
    shader.setFloat(6, color.alpha / 255.0);

    canvas.drawRect(
      Rect.fromLTWH(0, 0, size.width, size.height),
      Paint()..shader = shader,
    );
  }

  @override
  bool shouldRepaint(covariant _WavePainter oldDelegate) =>
      oldDelegate.time != time;
}
```

## 3. BackdropFilter

### Blur Effect

```dart
// lib/effects/blur_effect.dart
import 'dart:ui';
import 'package:flutter/material.dart';

class BlurCard extends StatelessWidget {
  final Widget child;
  final double blurAmount;
  final Color? backgroundColor;
  final BorderRadius borderRadius;

  const BlurCard({
    super.key,
    required this.child,
    this.blurAmount = 10,
    this.backgroundColor,
    this.borderRadius = const BorderRadius.all(Radius.circular(16)),
  });

  @override
  Widget build(BuildContext context) {
    return ClipRRect(
      borderRadius: borderRadius,
      child: BackdropFilter(
        filter: ImageFilter.blur(
          sigmaX: blurAmount,
          sigmaY: blurAmount,
        ),
        child: Container(
          decoration: BoxDecoration(
            color: backgroundColor ??
                Colors.white.withOpacity(0.2),
            borderRadius: borderRadius,
            border: Border.all(
              color: Colors.white.withOpacity(0.3),
            ),
          ),
          child: child,
        ),
      ),
    );
  }
}
```

## 4. Workshop: Glassmorphism Effect

### Glassmorphism Shader

```glsl
// shaders/glassmorphism.frag
#include <flutter/runtime_effect.glsl>

uniform vec2 uSize;
uniform sampler2D uTexture;
uniform float uBlur;
uniform vec4 uColor;

out vec4 fragColor;

vec4 blur(sampler2D tex, vec2 uv, vec2 texelSize, float radius) {
  vec4 color = vec4(0.0);
  float total = 0.0;
  
  for (float x = -radius; x <= radius; x++) {
    for (float y = -radius; y <= radius; y++) {
      vec2 offset = vec2(x, y) * texelSize;
      float weight = 1.0 / (length(vec2(x, y)) + 1.0);
      color += texture(tex, uv + offset) * weight;
      total += weight;
    }
  }
  
  return color / total;
}

void main() {
  vec2 fragCoord = FlutterFragCoord().xy;
  vec2 uv = fragCoord / uSize;
  
  vec2 texelSize = 1.0 / uSize;
  
  // Blur the background
  vec4 blurred = blur(uTexture, uv, texelSize, uBlur);
  
  // Mix with glass color
  vec4 glass = uColor;
  
  fragColor = mix(blurred, glass, glass.a);
  fragColor.a = 1.0;
}
```

### Glassmorphism Widget

```dart
// lib/effects/glassmorphism.dart
import 'dart:ui';
import 'package:flutter/material.dart';

class GlassmorphismContainer extends StatelessWidget {
  final Widget child;
  final double blurAmount;
  final double opacity;
  final Color tintColor;
  final BorderRadius borderRadius;
  final double borderOpacity;
  final List<BoxShadow>? shadows;

  const GlassmorphismContainer({
    super.key,
    required this.child,
    this.blurAmount = 15,
    this.opacity = 0.15,
    this.tintColor = Colors.white,
    this.borderRadius = const BorderRadius.all(Radius.circular(20)),
    this.borderOpacity = 0.3,
    this.shadows,
  });

  @override
  Widget build(BuildContext context) {
    return Container(
      decoration: BoxDecoration(
        borderRadius: borderRadius,
        boxShadow: shadows ??
            [
              BoxShadow(
                color: Colors.black.withOpacity(0.1),
                blurRadius: 20,
                spreadRadius: 0,
                offset: const Offset(0, 8),
              ),
            ],
      ),
      child: ClipRRect(
        borderRadius: borderRadius,
        child: BackdropFilter(
          filter: ImageFilter.blur(
            sigmaX: blurAmount,
            sigmaY: blurAmount,
          ),
          child: Container(
            decoration: BoxDecoration(
              gradient: LinearGradient(
                begin: Alignment.topLeft,
                end: Alignment.bottomRight,
                colors: [
                  tintColor.withOpacity(opacity + 0.1),
                  tintColor.withOpacity(opacity),
                ],
              ),
              borderRadius: borderRadius,
              border: Border.all(
                color: tintColor.withOpacity(borderOpacity),
                width: 1.5,
              ),
            ),
            child: child,
          ),
        ),
      ),
    );
  }
}
```

### Glassmorphism Demo Screen

```dart
// lib/effects/glassmorphism_demo.dart
import 'package:flutter/material.dart';
import 'glassmorphism.dart';

class GlassmorphismDemo extends StatefulWidget {
  const GlassmorphismDemo({super.key});

  @override
  State<GlassmorphismDemo> createState() => _GlassmorphismDemoState();
}

class _GlassmorphismDemoState extends State<GlassmorphismDemo> {
  double _blurAmount = 15;
  double _opacity = 0.15;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Stack(
        children: [
          // พื้นหลังสวยงาม
          Container(
            decoration: const BoxDecoration(
              gradient: LinearGradient(
                begin: Alignment.topLeft,
                end: Alignment.bottomRight,
                colors: [
                  Color(0xFF6B73FF),
                  Color(0xFF000DFF),
                  Color(0xFF9B59B6),
                ],
              ),
            ),
          ),

          // วงกลมสีสันเป็น decoration
          Positioned(
            top: 100,
            left: 50,
            child: Container(
              width: 200,
              height: 200,
              decoration: BoxDecoration(
                shape: BoxShape.circle,
                color: Colors.purple.withOpacity(0.5),
              ),
            ),
          ),
          Positioned(
            top: 200,
            right: 30,
            child: Container(
              width: 150,
              height: 150,
              decoration: BoxDecoration(
                shape: BoxShape.circle,
                color: Colors.pink.withOpacity(0.4),
              ),
            ),
          ),

          // Glass card
          Center(
            child: Padding(
              padding: const EdgeInsets.all(24),
              child: GlassmorphismContainer(
                blurAmount: _blurAmount,
                opacity: _opacity,
                child: Padding(
                  padding: const EdgeInsets.all(24),
                  child: Column(
                    mainAxisSize: MainAxisSize.min,
                    children: [
                      const Icon(
                        Icons.auto_awesome,
                        size: 48,
                        color: Colors.white,
                      ),
                      const SizedBox(height: 16),
                      const Text(
                        'Glassmorphism',
                        style: TextStyle(
                          color: Colors.white,
                          fontSize: 24,
                          fontWeight: FontWeight.bold,
                        ),
                      ),
                      const SizedBox(height: 8),
                      const Text(
                        'เอฟเฟกต์กระจกฝ้าที่สวยงาม\nด้วย BackdropFilter และ GLSL',
                        textAlign: TextAlign.center,
                        style: TextStyle(
                          color: Colors.white70,
                          height: 1.5,
                        ),
                      ),
                      const SizedBox(height: 24),

                      // Blur slider
                      Row(
                        children: [
                          const Text(
                            'Blur: ',
                            style: TextStyle(color: Colors.white),
                          ),
                          Expanded(
                            child: Slider(
                              value: _blurAmount,
                              min: 0,
                              max: 30,
                              activeColor: Colors.white,
                              inactiveColor: Colors.white30,
                              onChanged: (value) {
                                setState(() => _blurAmount = value);
                              },
                            ),
                          ),
                          Text(
                            _blurAmount.toStringAsFixed(0),
                            style: const TextStyle(color: Colors.white),
                          ),
                        ],
                      ),

                      // Opacity slider
                      Row(
                        children: [
                          const Text(
                            'Opacity: ',
                            style: TextStyle(color: Colors.white),
                          ),
                          Expanded(
                            child: Slider(
                              value: _opacity,
                              min: 0,
                              max: 0.5,
                              activeColor: Colors.white,
                              inactiveColor: Colors.white30,
                              onChanged: (value) {
                                setState(() => _opacity = value);
                              },
                            ),
                          ),
                          Text(
                            _opacity.toStringAsFixed(2),
                            style: const TextStyle(color: Colors.white),
                          ),
                        ],
                      ),
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
}
```

### Frosted Glass App Bar

```dart
// lib/effects/frosted_app_bar.dart
import 'dart:ui';
import 'package:flutter/material.dart';

class FrostedAppBar extends StatelessWidget implements PreferredSizeWidget {
  final String title;
  final List<Widget>? actions;

  const FrostedAppBar({
    super.key,
    required this.title,
    this.actions,
  });

  @override
  Size get preferredSize => const Size.fromHeight(kToolbarHeight);

  @override
  Widget build(BuildContext context) {
    return ClipRect(
      child: BackdropFilter(
        filter: ImageFilter.blur(sigmaX: 10, sigmaY: 10),
        child: AppBar(
          title: Text(title),
          backgroundColor: Colors.white.withOpacity(0.1),
          elevation: 0,
          actions: actions,
        ),
      ),
    );
  }
}
```

### Image Distortion Shader

```glsl
// shaders/ripple.frag
#include <flutter/runtime_effect.glsl>

uniform vec2 uSize;
uniform sampler2D uTexture;
uniform float uTime;
uniform vec2 uTouchPoint;

out vec4 fragColor;

void main() {
  vec2 fragCoord = FlutterFragCoord().xy;
  vec2 uv = fragCoord / uSize;
  
  // คำนวณ distance จาก touch point
  float dist = distance(uv, uTouchPoint);
  
  // สร้าง ripple effect
  float ripple = sin(dist * 50.0 - uTime * 10.0) * 0.01;
  float fade = max(0.0, 1.0 - dist * 5.0);
  
  // Offset UV ตาม ripple
  vec2 offset = normalize(uv - uTouchPoint) * ripple * fade;
  vec2 distortedUV = uv + offset;
  
  fragColor = texture(uTexture, distortedUV);
}
```

### Particle System ด้วย Shader

```dart
// lib/effects/particle_widget.dart
import 'package:flutter/material.dart';
import 'dart:math' as math;

class Particle {
  Offset position;
  Offset velocity;
  double size;
  Color color;
  double life;
  double maxLife;

  Particle({
    required this.position,
    required this.velocity,
    required this.size,
    required this.color,
    required this.maxLife,
  }) : life = maxLife;

  bool get isDead => life <= 0;
  double get alpha => (life / maxLife).clamp(0, 1);

  void update(double dt) {
    position += velocity * dt;
    velocity = velocity.scale(0.99, 0.99); // friction
    life -= dt;
  }
}

class ParticleWidget extends StatefulWidget {
  final Color color;
  const ParticleWidget({super.key, this.color = Colors.blue});

  @override
  State<ParticleWidget> createState() => _ParticleWidgetState();
}

class _ParticleWidgetState extends State<ParticleWidget>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  final List<Particle> _particles = [];
  final _random = math.Random();
  DateTime? _lastTime;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 1),
    )..repeat();

    _controller.addListener(_update);
  }

  void _update() {
    final now = DateTime.now();
    final dt = _lastTime != null
        ? now.difference(_lastTime!).inMilliseconds / 1000.0
        : 0.016;
    _lastTime = now;

    // เพิ่ม particle ใหม่
    if (_particles.length < 100) {
      _particles.add(Particle(
        position: const Offset(200, 400),
        velocity: Offset(
          (_random.nextDouble() - 0.5) * 200,
          -_random.nextDouble() * 300 - 100,
        ),
        size: _random.nextDouble() * 8 + 2,
        color: widget.color,
        maxLife: _random.nextDouble() * 2 + 0.5,
      ));
    }

    // อัปเดต particles
    for (final p in _particles) {
      p.update(dt);
    }

    // ลบ particles ที่ตายแล้ว
    _particles.removeWhere((p) => p.isDead);

    if (mounted) setState(() {});
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return CustomPaint(
      painter: _ParticlePainter(particles: _particles),
      size: Size.infinite,
    );
  }
}

class _ParticlePainter extends CustomPainter {
  final List<Particle> particles;

  _ParticlePainter({required this.particles});

  @override
  void paint(Canvas canvas, Size size) {
    for (final p in particles) {
      canvas.drawCircle(
        p.position,
        p.size,
        Paint()..color = p.color.withOpacity(p.alpha),
      );
    }
  }

  @override
  bool shouldRepaint(covariant _ParticlePainter oldDelegate) => true;
}
```

## สรุป

Shader Effects ใน Flutter ช่วยให้:
1. **ประสิทธิภาพสูง** - รันบน GPU โดยตรง
2. **เอฟเฟกต์ที่ซับซ้อน** - ทำได้ด้วย GLSL
3. **Animated** - ส่ง time uniform ได้
4. **Flexible** - ทำงานร่วมกับ Flutter rendering

## แบบทดสอบ

1. อธิบายว่า Fragment Shader คืออะไรและทำงานอย่างไรกับ GPU
2. `BackdropFilter` แตกต่างจาก `ImageFilter` อย่างไร?
3. เขียน GLSL shader ที่สร้าง vignette effect (มืดรอบขอบ)
4. ทำไม `shouldRepaint` ควร return `true` สำหรับ animated painters?
