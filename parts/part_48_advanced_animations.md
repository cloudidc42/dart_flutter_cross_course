# Part 48: Advanced Animations

## บทนำ

Advanced Animations ใน Flutter ช่วยให้เราสร้างประสบการณ์ผู้ใช้ที่น่าประทับใจ ในบทนี้เราจะเรียนรู้ Explicit Animations แบบลึก, Staggered Animations, Lottie และ Rive animations

---

## 48.1 Explicit Animations แบบลึก

### AnimationController

```dart
import 'package:flutter/material.dart';

class ExplicitAnimationExample extends StatefulWidget {
  const ExplicitAnimationExample({super.key});
  
  @override
  State<ExplicitAnimationExample> createState() => 
      _ExplicitAnimationExampleState();
}

class _ExplicitAnimationExampleState extends State<ExplicitAnimationExample>
    with SingleTickerProviderStateMixin {
  
  late AnimationController _controller;
  late Animation<double> _scaleAnimation;
  late Animation<double> _opacityAnimation;
  late Animation<Offset> _slideAnimation;
  late Animation<Color?> _colorAnimation;
  
  @override
  void initState() {
    super.initState();
    
    _controller = AnimationController(
      duration: const Duration(milliseconds: 1500),
      vsync: this,
    );
    
    // Scale animation with custom curve
    _scaleAnimation = Tween<double>(begin: 0.0, end: 1.0).animate(
      CurvedAnimation(
        parent: _controller,
        curve: const Interval(0.0, 0.6, curve: Curves.elasticOut),
      ),
    );
    
    // Opacity animation
    _opacityAnimation = Tween<double>(begin: 0.0, end: 1.0).animate(
      CurvedAnimation(
        parent: _controller,
        curve: const Interval(0.0, 0.3, curve: Curves.easeIn),
      ),
    );
    
    // Slide animation
    _slideAnimation = Tween<Offset>(
      begin: const Offset(0, -0.5),
      end: Offset.zero,
    ).animate(
      CurvedAnimation(
        parent: _controller,
        curve: const Interval(0.2, 0.8, curve: Curves.easeOut),
      ),
    );
    
    // Color animation
    _colorAnimation = ColorTween(
      begin: Colors.blue,
      end: Colors.purple,
    ).animate(
      CurvedAnimation(
        parent: _controller,
        curve: const Interval(0.5, 1.0, curve: Curves.easeInOut),
      ),
    );
    
    // Start animation
    _controller.forward();
  }
  
  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _controller,
      builder: (context, child) {
        return Opacity(
          opacity: _opacityAnimation.value,
          child: SlideTransition(
            position: _slideAnimation,
            child: ScaleTransition(
              scale: _scaleAnimation,
              child: Container(
                width: 200,
                height: 200,
                decoration: BoxDecoration(
                  color: _colorAnimation.value,
                  borderRadius: BorderRadius.circular(20),
                  boxShadow: [
                    BoxShadow(
                      color: (_colorAnimation.value ?? Colors.blue)
                          .withOpacity(0.4),
                      blurRadius: 20,
                      spreadRadius: 5,
                    ),
                  ],
                ),
                child: const Icon(
                  Icons.star,
                  color: Colors.white,
                  size: 80,
                ),
              ),
            ),
          ),
        );
      },
    );
  }
  
  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}
```

### AnimationController Groups (Multiple Controllers)

```dart
class MultipleAnimations extends StatefulWidget {
  const MultipleAnimations({super.key});
  
  @override
  State<MultipleAnimations> createState() => _MultipleAnimationsState();
}

class _MultipleAnimationsState extends State<MultipleAnimations>
    with TickerProviderStateMixin { // ใช้ TickerProviderStateMixin สำหรับหลาย controllers
  
  late AnimationController _bounceController;
  late AnimationController _rotateController;
  late AnimationController _pulseController;
  
  late Animation<double> _bounceAnimation;
  late Animation<double> _rotateAnimation;
  late Animation<double> _pulseAnimation;
  
  @override
  void initState() {
    super.initState();
    
    // Bounce animation
    _bounceController = AnimationController(
      duration: const Duration(milliseconds: 800),
      vsync: this,
    );
    _bounceAnimation = Tween<double>(begin: 0, end: -30).animate(
      CurvedAnimation(
        parent: _bounceController,
        curve: Curves.bounceOut,
      ),
    );
    
    // Rotate animation (continuous)
    _rotateController = AnimationController(
      duration: const Duration(seconds: 3),
      vsync: this,
    )..repeat();
    _rotateAnimation = Tween<double>(begin: 0, end: 2 * pi).animate(
      _rotateController,
    );
    
    // Pulse animation (repeat with reverse)
    _pulseController = AnimationController(
      duration: const Duration(milliseconds: 1000),
      vsync: this,
    )..repeat(reverse: true);
    _pulseAnimation = Tween<double>(begin: 1.0, end: 1.2).animate(
      CurvedAnimation(
        parent: _pulseController,
        curve: Curves.easeInOut,
      ),
    );
    
    _bounceController.forward();
  }
  
  @override
  Widget build(BuildContext context) {
    return Row(
      mainAxisAlignment: MainAxisAlignment.spaceEvenly,
      children: [
        // Bounce
        AnimatedBuilder(
          animation: _bounceAnimation,
          builder: (context, child) => Transform.translate(
            offset: Offset(0, _bounceAnimation.value),
            child: child,
          ),
          child: const FlutterLogo(size: 60),
        ),
        
        // Rotate
        AnimatedBuilder(
          animation: _rotateAnimation,
          builder: (context, child) => Transform.rotate(
            angle: _rotateAnimation.value,
            child: child,
          ),
          child: const Icon(Icons.star, size: 60, color: Colors.amber),
        ),
        
        // Pulse
        AnimatedBuilder(
          animation: _pulseAnimation,
          builder: (context, child) => Transform.scale(
            scale: _pulseAnimation.value,
            child: child,
          ),
          child: const Icon(Icons.favorite, size: 60, color: Colors.red),
        ),
      ],
    );
  }
  
  @override
  void dispose() {
    _bounceController.dispose();
    _rotateController.dispose();
    _pulseController.dispose();
    super.dispose();
  }
}
```

---

## 48.2 Staggered Animations

```dart
class StaggeredAnimation extends StatefulWidget {
  const StaggeredAnimation({super.key});
  
  @override
  State<StaggeredAnimation> createState() => _StaggeredAnimationState();
}

class _StaggeredAnimationState extends State<StaggeredAnimation>
    with SingleTickerProviderStateMixin {
  
  late AnimationController _controller;
  late List<Animation<Offset>> _slideAnimations;
  late List<Animation<double>> _fadeAnimations;
  
  final List<String> _items = [
    'ขั้นตอนที่ 1: เริ่มต้น',
    'ขั้นตอนที่ 2: ดำเนินการ',
    'ขั้นตอนที่ 3: ตรวจสอบ',
    'ขั้นตอนที่ 4: สรุป',
    'ขั้นตอนที่ 5: เสร็จสิ้น',
  ];
  
  @override
  void initState() {
    super.initState();
    
    _controller = AnimationController(
      duration: const Duration(milliseconds: 1500),
      vsync: this,
    );
    
    // สร้าง animation สำหรับแต่ละ item โดยมี interval ต่างกัน
    _slideAnimations = List.generate(_items.length, (index) {
      final start = index * 0.15;
      final end = start + 0.4;
      
      return Tween<Offset>(
        begin: const Offset(-1.0, 0),
        end: Offset.zero,
      ).animate(
        CurvedAnimation(
          parent: _controller,
          curve: Interval(
            start.clamp(0.0, 1.0),
            end.clamp(0.0, 1.0),
            curve: Curves.easeOutCubic,
          ),
        ),
      );
    });
    
    _fadeAnimations = List.generate(_items.length, (index) {
      final start = index * 0.15;
      final end = start + 0.3;
      
      return Tween<double>(begin: 0.0, end: 1.0).animate(
        CurvedAnimation(
          parent: _controller,
          curve: Interval(
            start.clamp(0.0, 1.0),
            end.clamp(0.0, 1.0),
            curve: Curves.easeIn,
          ),
        ),
      );
    });
    
    _controller.forward();
  }
  
  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _controller,
      builder: (context, _) {
        return Column(
          children: List.generate(
            _items.length,
            (index) => Padding(
              padding: const EdgeInsets.symmetric(
                horizontal: 20,
                vertical: 8,
              ),
              child: FadeTransition(
                opacity: _fadeAnimations[index],
                child: SlideTransition(
                  position: _slideAnimations[index],
                  child: Card(
                    child: ListTile(
                      leading: CircleAvatar(
                        backgroundColor: Colors.blue,
                        child: Text(
                          '${index + 1}',
                          style: const TextStyle(color: Colors.white),
                        ),
                      ),
                      title: Text(_items[index]),
                    ),
                  ),
                ),
              ),
            ),
          ),
        );
      },
    );
  }
  
  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}
```

---

## 48.3 Lottie Animations

```yaml
# pubspec.yaml
dependencies:
  lottie: ^3.0.0
```

```dart
import 'package:lottie/lottie.dart';

class LottieExamples extends StatefulWidget {
  const LottieExamples({super.key});
  
  @override
  State<LottieExamples> createState() => _LottieExamplesState();
}

class _LottieExamplesState extends State<LottieExamples>
    with SingleTickerProviderStateMixin {
  
  late AnimationController _controller;
  bool _isPlaying = true;
  
  @override
  void initState() {
    super.initState();
    _controller = AnimationController(vsync: this);
  }
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // Basic Lottie from assets
        Lottie.asset(
          'assets/animations/loading.json',
          width: 200,
          height: 200,
          fit: BoxFit.fill,
        ),
        
        // Lottie from network
        Lottie.network(
          'https://assets.lottiefiles.com/packages/lf20_success.json',
          width: 200,
          height: 200,
        ),
        
        // Lottie พร้อม controller
        Lottie.asset(
          'assets/animations/heart.json',
          controller: _controller,
          onLoaded: (composition) {
            _controller
              ..duration = composition.duration
              ..forward();
          },
          width: 150,
          height: 150,
        ),
        
        // Controls
        Row(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            IconButton(
              onPressed: () {
                if (_controller.isAnimating) {
                  _controller.stop();
                } else {
                  _controller.forward();
                }
                setState(() => _isPlaying = !_isPlaying);
              },
              icon: Icon(_isPlaying ? Icons.pause : Icons.play_arrow),
            ),
            IconButton(
              onPressed: () => _controller.reset(),
              icon: const Icon(Icons.replay),
            ),
          ],
        ),
        
        // Lottie ที่ loop
        Lottie.asset(
          'assets/animations/stars.json',
          repeat: true,
          reverse: false,
          animate: true,
        ),
      ],
    );
  }
  
  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}
```

---

## 48.4 Rive Animations

```yaml
# pubspec.yaml
dependencies:
  rive: ^0.12.4
```

```dart
import 'package:rive/rive.dart';

class RiveExamples extends StatefulWidget {
  const RiveExamples({super.key});
  
  @override
  State<RiveExamples> createState() => _RiveExamplesState();
}

class _RiveExamplesState extends State<RiveExamples> {
  SMIBool? _isHovered;
  SMITrigger? _clickTrigger;
  StateMachineController? _controller;
  
  void _onRiveInit(Artboard artboard) {
    // เชื่อมต่อ State Machine
    final controller = StateMachineController.fromArtboard(
      artboard,
      'ButtonMachine', // ชื่อ State Machine ใน Rive
    );
    
    if (controller != null) {
      artboard.addController(controller);
      _isHovered = controller.findInput<bool>('isHovered') as SMIBool?;
      _clickTrigger = controller.findInput<bool>('click') as SMITrigger?;
      _controller = controller;
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // Basic Rive animation
        const SizedBox(
          width: 300,
          height: 200,
          child: RiveAnimation.asset(
            'assets/animations/loading_bear.riv',
            fit: BoxFit.contain,
          ),
        ),
        
        // Rive พร้อม State Machine
        MouseRegion(
          onEnter: (_) => _isHovered?.value = true,
          onExit: (_) => _isHovered?.value = false,
          child: GestureDetector(
            onTap: () => _clickTrigger?.fire(),
            child: SizedBox(
              width: 200,
              height: 100,
              child: RiveAnimation.asset(
                'assets/animations/button.riv',
                onInit: _onRiveInit,
              ),
            ),
          ),
        ),
        
        // Rive จาก network
        const SizedBox(
          width: 200,
          height: 200,
          child: RiveAnimation.network(
            'https://cdn.rive.app/animations/vehicles.riv',
            artboard: 'Truck',
            animations: ['idle', 'windshield_wipers'],
          ),
        ),
      ],
    );
  }
  
  @override
  void dispose() {
    _controller?.dispose();
    super.dispose();
  }
}
```

---

## 48.5 Workshop: Onboarding พร้อม Animations

```dart
// onboarding_page.dart
class OnboardingPage extends StatefulWidget {
  const OnboardingPage({super.key});
  
  @override
  State<OnboardingPage> createState() => _OnboardingPageState();
}

class _OnboardingPageState extends State<OnboardingPage>
    with TickerProviderStateMixin {
  
  final PageController _pageController = PageController();
  int _currentPage = 0;
  
  late AnimationController _backgroundController;
  late AnimationController _contentController;
  late Animation<Color?> _backgroundAnimation;
  
  final List<OnboardingData> _pages = [
    OnboardingData(
      title: 'ยินดีต้อนรับ',
      description: 'แอปของเราช่วยให้คุณจัดการชีวิตได้ง่ายขึ้น',
      icon: Icons.waving_hand,
      color: Colors.blue,
    ),
    OnboardingData(
      title: 'ติดตามเป้าหมาย',
      description: 'วางแผนและติดตามความก้าวหน้าของคุณทุกวัน',
      icon: Icons.track_changes,
      color: Colors.purple,
    ),
    OnboardingData(
      title: 'เชื่อมต่อกัน',
      description: 'แบ่งปันความสำเร็จกับเพื่อนและครอบครัว',
      icon: Icons.people,
      color: Colors.teal,
    ),
  ];
  
  @override
  void initState() {
    super.initState();
    
    _backgroundController = AnimationController(
      duration: const Duration(milliseconds: 500),
      vsync: this,
    );
    
    _contentController = AnimationController(
      duration: const Duration(milliseconds: 800),
      vsync: this,
    );
    
    _backgroundAnimation = ColorTween(
      begin: _pages[0].color,
      end: _pages[0].color,
    ).animate(_backgroundController);
    
    _contentController.forward();
  }
  
  void _onPageChanged(int page) {
    // Animate background color
    _backgroundAnimation = ColorTween(
      begin: _pages[_currentPage].color,
      end: _pages[page].color,
    ).animate(
      CurvedAnimation(
        parent: _backgroundController,
        curve: Curves.easeInOut,
      ),
    );
    
    setState(() => _currentPage = page);
    
    _backgroundController.forward(from: 0);
    _contentController.forward(from: 0);
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: AnimatedBuilder(
        animation: Listenable.merge([
          _backgroundController,
          _contentController,
        ]),
        builder: (context, _) {
          return Container(
            color: _backgroundAnimation.value ?? _pages[_currentPage].color,
            child: SafeArea(
              child: Column(
                children: [
                  // Skip button
                  Align(
                    alignment: Alignment.topRight,
                    child: Padding(
                      padding: const EdgeInsets.all(16),
                      child: TextButton(
                        onPressed: _finish,
                        child: const Text(
                          'ข้าม',
                          style: TextStyle(
                            color: Colors.white,
                            fontSize: 16,
                          ),
                        ),
                      ),
                    ),
                  ),
                  
                  // Pages
                  Expanded(
                    child: PageView.builder(
                      controller: _pageController,
                      onPageChanged: _onPageChanged,
                      itemCount: _pages.length,
                      itemBuilder: (context, index) {
                        return _OnboardingPageContent(
                          data: _pages[index],
                          animation: _contentController,
                          isActive: index == _currentPage,
                        );
                      },
                    ),
                  ),
                  
                  // Indicators
                  Row(
                    mainAxisAlignment: MainAxisAlignment.center,
                    children: List.generate(
                      _pages.length,
                      (index) => AnimatedContainer(
                        duration: const Duration(milliseconds: 300),
                        margin: const EdgeInsets.symmetric(horizontal: 4),
                        width: index == _currentPage ? 24 : 8,
                        height: 8,
                        decoration: BoxDecoration(
                          color: Colors.white.withOpacity(
                            index == _currentPage ? 1.0 : 0.4,
                          ),
                          borderRadius: BorderRadius.circular(4),
                        ),
                      ),
                    ),
                  ),
                  
                  const SizedBox(height: 32),
                  
                  // Next / Done button
                  Padding(
                    padding: const EdgeInsets.symmetric(horizontal: 40),
                    child: ElevatedButton(
                      onPressed: () {
                        if (_currentPage < _pages.length - 1) {
                          _pageController.nextPage(
                            duration: const Duration(milliseconds: 400),
                            curve: Curves.easeInOut,
                          );
                        } else {
                          _finish();
                        }
                      },
                      style: ElevatedButton.styleFrom(
                        backgroundColor: Colors.white,
                        foregroundColor: _pages[_currentPage].color,
                        minimumSize: const Size(double.infinity, 50),
                        shape: RoundedRectangleBorder(
                          borderRadius: BorderRadius.circular(25),
                        ),
                      ),
                      child: Text(
                        _currentPage < _pages.length - 1
                            ? 'ถัดไป'
                            : 'เริ่มต้นใช้งาน',
                        style: const TextStyle(
                          fontSize: 18,
                          fontWeight: FontWeight.bold,
                        ),
                      ),
                    ),
                  ),
                  
                  const SizedBox(height: 40),
                ],
              ),
            ),
          );
        },
      ),
    );
  }
  
  void _finish() {
    // บันทึกว่าดู onboarding แล้ว
    // SharedPreferences หรือ Hive
    Navigator.of(context).pushReplacementNamed('/home');
  }
  
  @override
  void dispose() {
    _pageController.dispose();
    _backgroundController.dispose();
    _contentController.dispose();
    super.dispose();
  }
}

class _OnboardingPageContent extends StatelessWidget {
  final OnboardingData data;
  final AnimationController animation;
  final bool isActive;
  
  const _OnboardingPageContent({
    required this.data,
    required this.animation,
    required this.isActive,
  });
  
  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: animation,
      builder: (context, _) {
        final scale = Tween<double>(begin: 0.5, end: 1.0).animate(
          CurvedAnimation(
            parent: animation,
            curve: const Interval(0.0, 0.6, curve: Curves.elasticOut),
          ),
        ).value;
        
        final opacity = Tween<double>(begin: 0.0, end: 1.0).animate(
          CurvedAnimation(
            parent: animation,
            curve: const Interval(0.0, 0.4, curve: Curves.easeIn),
          ),
        ).value;
        
        final slideY = Tween<double>(begin: 50, end: 0).animate(
          CurvedAnimation(
            parent: animation,
            curve: const Interval(0.2, 0.8, curve: Curves.easeOut),
          ),
        ).value;
        
        return Padding(
          padding: const EdgeInsets.all(40),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              Transform.scale(
                scale: scale,
                child: Opacity(
                  opacity: opacity,
                  child: Container(
                    width: 180,
                    height: 180,
                    decoration: BoxDecoration(
                      color: Colors.white.withOpacity(0.2),
                      shape: BoxShape.circle,
                    ),
                    child: Icon(
                      data.icon,
                      size: 100,
                      color: Colors.white,
                    ),
                  ),
                ),
              ),
              
              const SizedBox(height: 48),
              
              Transform.translate(
                offset: Offset(0, slideY),
                child: Opacity(
                  opacity: opacity,
                  child: Column(
                    children: [
                      Text(
                        data.title,
                        style: const TextStyle(
                          fontSize: 28,
                          fontWeight: FontWeight.bold,
                          color: Colors.white,
                        ),
                        textAlign: TextAlign.center,
                      ),
                      const SizedBox(height: 16),
                      Text(
                        data.description,
                        style: TextStyle(
                          fontSize: 16,
                          color: Colors.white.withOpacity(0.8),
                          height: 1.5,
                        ),
                        textAlign: TextAlign.center,
                      ),
                    ],
                  ),
                ),
              ),
            ],
          ),
        );
      },
    );
  }
}

class OnboardingData {
  final String title;
  final String description;
  final IconData icon;
  final Color color;
  
  const OnboardingData({
    required this.title,
    required this.description,
    required this.icon,
    required this.color,
  });
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- Explicit Animations แบบลึกด้วย AnimationController
- การใช้งาน Multiple AnimationControllers
- Staggered Animations สำหรับ list items
- Lottie Animations สำหรับ vector animations
- Rive Animations สำหรับ interactive animations
- Workshop: Onboarding แบบ animated

**แบบฝึกหัดเพิ่มเติม:**
1. สร้าง loading screen พร้อม animated logo
2. เพิ่ม shared element transition ระหว่างหน้า
3. สร้าง particle effect ด้วย CustomPainter + Animation
4. Implement physics-based animation ด้วย SpringSimulation
