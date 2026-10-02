# Part 50: Flutter Web Basics

## บทนำ

Flutter Web ช่วยให้เราสร้างเว็บแอปพลิเคชันด้วย Flutter codebase เดียวกันกับมือถือ ในบทนี้จะครอบคลุมการพิจารณาพิเศษสำหรับ web, responsive layouts, SEO และการ deploy

---

## 50.1 Web-Specific Considerations

### ความแตกต่างจาก Mobile

```
Mobile                          Web
------                          ---
Touch gestures                  Mouse + keyboard + touch
Back button (hardware)          Browser back button
App bar navigation              URL-based navigation
Platform permissions            Browser permissions
Local storage limited           localStorage, cookies
Push notifications via FCM      Web Push API
```

### เปิดใช้งาน Flutter Web

```bash
# สร้างโปรเจกต์ใหม่พร้อม web support
flutter create --platforms=web mywebapp

# เพิ่ม web support ในโปรเจกต์ที่มีอยู่
flutter create --platforms=web .

# รันบน web
flutter run -d chrome

# Build สำหรับ production
flutter build web --release

# Build พร้อม base href (สำหรับ sub-path deployment)
flutter build web --base-href /myapp/
```

### web/index.html

```html
<!DOCTYPE html>
<html>
<head>
  <base href="$FLUTTER_BASE_HREF">
  <meta charset="UTF-8">
  <meta content="IE=Edge" http-equiv="X-UA-Compatible">
  <meta name="description" content="My Flutter Web App">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  
  <!-- Open Graph / Facebook -->
  <meta property="og:type" content="website">
  <meta property="og:title" content="My App">
  <meta property="og:description" content="My Flutter Web App">
  <meta property="og:image" content="/icons/Icon-512.png">
  
  <!-- Twitter -->
  <meta name="twitter:card" content="summary_large_image">
  
  <!-- PWA -->
  <link rel="manifest" href="manifest.json">
  <meta name="theme-color" content="#2196F3">
  
  <!-- Favicon -->
  <link rel="icon" type="image/png" href="favicon.png"/>
  
  <title>My Flutter Web App</title>
  
  <!-- Custom styles -->
  <style>
    body {
      background-color: #FFFFFF;
      margin: 0;
      padding: 0;
    }
    
    /* Loading screen */
    #loading {
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
      background: #2196F3;
    }
    
    .loading-spinner {
      width: 48px;
      height: 48px;
      border: 4px solid #ffffff40;
      border-top: 4px solid white;
      border-radius: 50%;
      animation: spin 1s linear infinite;
    }
    
    @keyframes spin {
      0% { transform: rotate(0deg); }
      100% { transform: rotate(360deg); }
    }
  </style>
</head>
<body>
  <div id="loading">
    <div class="loading-spinner"></div>
  </div>
  
  <script src="flutter_bootstrap.js" async></script>
  
  <script>
    window.addEventListener('flutter-first-frame', function() {
      document.getElementById('loading').remove();
    });
  </script>
</body>
</html>
```

---

## 50.2 Responsive Layouts

### Breakpoints

```dart
// lib/utils/responsive.dart
class Breakpoints {
  static const double mobile = 600;
  static const double tablet = 1024;
  static const double desktop = 1440;
}

enum DeviceType { mobile, tablet, desktop }

DeviceType getDeviceType(BuildContext context) {
  final width = MediaQuery.of(context).size.width;
  if (width < Breakpoints.mobile) return DeviceType.mobile;
  if (width < Breakpoints.tablet) return DeviceType.tablet;
  return DeviceType.desktop;
}
```

### Responsive Widget

```dart
// lib/widgets/responsive_layout.dart
class ResponsiveLayout extends StatelessWidget {
  final Widget mobile;
  final Widget? tablet;
  final Widget? desktop;
  
  const ResponsiveLayout({
    super.key,
    required this.mobile,
    this.tablet,
    this.desktop,
  });
  
  @override
  Widget build(BuildContext context) {
    final deviceType = getDeviceType(context);
    
    switch (deviceType) {
      case DeviceType.desktop:
        return desktop ?? tablet ?? mobile;
      case DeviceType.tablet:
        return tablet ?? mobile;
      case DeviceType.mobile:
        return mobile;
    }
  }
}

// ใช้งาน
class HomePage extends StatelessWidget {
  const HomePage({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: ResponsiveLayout(
        mobile: _MobileLayout(),
        tablet: _TabletLayout(),
        desktop: _DesktopLayout(),
      ),
    );
  }
}
```

### Responsive Navigation

```dart
// Adaptive Navigation - ใช้ Drawer บนมือถือ, Rail บน tablet, Sidebar บน desktop
class AdaptiveNavigation extends StatefulWidget {
  final int selectedIndex;
  final ValueChanged<int> onDestinationSelected;
  final Widget body;
  
  const AdaptiveNavigation({
    super.key,
    required this.selectedIndex,
    required this.onDestinationSelected,
    required this.body,
  });
  
  @override
  State<AdaptiveNavigation> createState() => _AdaptiveNavigationState();
}

class _AdaptiveNavigationState extends State<AdaptiveNavigation> {
  static const _destinations = [
    NavigationDestination(icon: Icon(Icons.home), label: 'หน้าหลัก'),
    NavigationDestination(icon: Icon(Icons.search), label: 'ค้นหา'),
    NavigationDestination(icon: Icon(Icons.shopping_cart), label: 'ตะกร้า'),
    NavigationDestination(icon: Icon(Icons.person), label: 'โปรไฟล์'),
  ];
  
  @override
  Widget build(BuildContext context) {
    final deviceType = getDeviceType(context);
    
    switch (deviceType) {
      case DeviceType.mobile:
        return Scaffold(
          body: widget.body,
          bottomNavigationBar: NavigationBar(
            selectedIndex: widget.selectedIndex,
            onDestinationSelected: widget.onDestinationSelected,
            destinations: _destinations,
          ),
        );
      
      case DeviceType.tablet:
        return Scaffold(
          body: Row(
            children: [
              NavigationRail(
                selectedIndex: widget.selectedIndex,
                onDestinationSelected: widget.onDestinationSelected,
                labelType: NavigationRailLabelType.selected,
                destinations: _destinations
                    .map((d) => NavigationRailDestination(
                          icon: d.icon,
                          label: Text(d.label),
                        ))
                    .toList(),
              ),
              const VerticalDivider(width: 1),
              Expanded(child: widget.body),
            ],
          ),
        );
      
      case DeviceType.desktop:
        return Scaffold(
          body: Row(
            children: [
              SizedBox(
                width: 250,
                child: _NavigationSidebar(
                  selectedIndex: widget.selectedIndex,
                  onDestinationSelected: widget.onDestinationSelected,
                  destinations: _destinations,
                ),
              ),
              const VerticalDivider(width: 1),
              Expanded(child: widget.body),
            ],
          ),
        );
    }
  }
}
```

### Fluid Layout

```dart
// Grid ที่ปรับจำนวน columns ตาม screen width
class ResponsiveGrid extends StatelessWidget {
  final List<Widget> children;
  
  const ResponsiveGrid({super.key, required this.children});
  
  int _getCrossAxisCount(double width) {
    if (width < 600) return 1;
    if (width < 900) return 2;
    if (width < 1200) return 3;
    return 4;
  }
  
  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        final crossAxisCount = _getCrossAxisCount(constraints.maxWidth);
        
        return GridView.builder(
          gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
            crossAxisCount: crossAxisCount,
            crossAxisSpacing: 16,
            mainAxisSpacing: 16,
            childAspectRatio: 1.5,
          ),
          itemCount: children.length,
          itemBuilder: (context, index) => children[index],
        );
      },
    );
  }
}
```

---

## 50.3 SEO Basics

### Meta Tags และ Title

```dart
// Web ต้องจัดการ SEO เพิ่มเติม
import 'dart:html' as html;

class SeoService {
  static void setTitle(String title) {
    html.document.title = title;
  }
  
  static void setMetaDescription(String description) {
    final meta = html.document.querySelector('meta[name="description"]');
    if (meta != null) {
      meta.setAttribute('content', description);
    } else {
      final newMeta = html.MetaElement()
        ..name = 'description'
        ..content = description;
      html.document.head?.append(newMeta);
    }
  }
  
  static void setCanonicalUrl(String url) {
    var link = html.document.querySelector('link[rel="canonical"]');
    if (link != null) {
      link.setAttribute('href', url);
    } else {
      final newLink = html.LinkElement()
        ..rel = 'canonical'
        ..href = url;
      html.document.head?.append(newLink);
    }
  }
  
  static void updateOpenGraph({
    required String title,
    required String description,
    String? imageUrl,
  }) {
    _setOgMeta('og:title', title);
    _setOgMeta('og:description', description);
    if (imageUrl != null) _setOgMeta('og:image', imageUrl);
  }
  
  static void _setOgMeta(String property, String content) {
    var meta = html.document.querySelector('meta[property="$property"]');
    if (meta != null) {
      meta.setAttribute('content', content);
    } else {
      final newMeta = html.MetaElement()
        ..setAttribute('property', property)
        ..content = content;
      html.document.head?.append(newMeta);
    }
  }
}
```

### URL-based Navigation สำหรับ SEO

```dart
// lib/router/app_router.dart
import 'package:go_router/go_router.dart';

final GoRouter appRouter = GoRouter(
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => const HomePage(),
    ),
    GoRoute(
      path: '/products',
      builder: (context, state) => const ProductsPage(),
    ),
    GoRoute(
      path: '/products/:id',
      builder: (context, state) {
        final productId = state.pathParameters['id']!;
        return ProductDetailPage(productId: productId);
      },
    ),
    GoRoute(
      path: '/about',
      builder: (context, state) => const AboutPage(),
    ),
  ],
  errorBuilder: (context, state) => const NotFoundPage(),
);

// ใน main.dart
MaterialApp.router(
  routerConfig: appRouter,
)
```

---

## 50.4 Web Deployment

### Build และ Deploy ไปยัง Firebase Hosting

```bash
# 1. Build
flutter build web --release

# 2. ติดตั้ง Firebase CLI
npm install -g firebase-tools

# 3. Login
firebase login

# 4. Init Firebase Hosting
firebase init hosting
# เลือก build/web เป็น public directory
# Y for single-page app rewrite

# 5. Deploy
firebase deploy --only hosting
```

### firebase.json Configuration

```json
{
  "hosting": {
    "public": "build/web",
    "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
    "rewrites": [
      {
        "source": "**",
        "destination": "/index.html"
      }
    ],
    "headers": [
      {
        "source": "**/*.@(eot|otf|ttf|ttc|woff|font.css)",
        "headers": [
          {
            "key": "Access-Control-Allow-Origin",
            "value": "*"
          }
        ]
      },
      {
        "source": "**/*.@(js|css)",
        "headers": [
          {
            "key": "Cache-Control",
            "value": "max-age=604800"
          }
        ]
      },
      {
        "source": "**",
        "headers": [
          {
            "key": "X-Frame-Options",
            "value": "SAMEORIGIN"
          }
        ]
      }
    ]
  }
}
```

### PWA Configuration

```json
// web/manifest.json
{
  "name": "My Flutter App",
  "short_name": "MyApp",
  "start_url": ".",
  "display": "standalone",
  "background_color": "#0175C2",
  "theme_color": "#0175C2",
  "description": "My Flutter Web Application",
  "orientation": "portrait-primary",
  "prefer_related_applications": false,
  "icons": [
    {
      "src": "icons/Icon-192.png",
      "sizes": "192x192",
      "type": "image/png"
    },
    {
      "src": "icons/Icon-512.png",
      "sizes": "512x512",
      "type": "image/png"
    },
    {
      "src": "icons/Icon-maskable-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "maskable"
    }
  ]
}
```

---

## 50.5 Workshop: Landing Page

```dart
// lib/pages/landing_page.dart
import 'package:flutter/material.dart';
import 'package:url_launcher/url_launcher.dart';

class LandingPage extends StatefulWidget {
  const LandingPage({super.key});
  
  @override
  State<LandingPage> createState() => _LandingPageState();
}

class _LandingPageState extends State<LandingPage> {
  final ScrollController _scrollController = ScrollController();
  bool _isScrolled = false;
  
  final GlobalKey _featuresKey = GlobalKey();
  final GlobalKey _pricingKey = GlobalKey();
  final GlobalKey _contactKey = GlobalKey();
  
  @override
  void initState() {
    super.initState();
    _scrollController.addListener(() {
      setState(() {
        _isScrolled = _scrollController.offset > 80;
      });
    });
  }
  
  void _scrollTo(GlobalKey key) {
    final context = key.currentContext;
    if (context != null) {
      Scrollable.ensureVisible(
        context,
        duration: const Duration(milliseconds: 600),
        curve: Curves.easeInOut,
      );
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        backgroundColor: _isScrolled
            ? Colors.white
            : Colors.transparent,
        elevation: _isScrolled ? 4 : 0,
        title: Row(
          children: [
            const FlutterLogo(size: 32),
            const SizedBox(width: 8),
            Text(
              'MyApp',
              style: TextStyle(
                color: _isScrolled ? Colors.black : Colors.white,
                fontWeight: FontWeight.bold,
                fontSize: 20,
              ),
            ),
          ],
        ),
        actions: [
          if (getDeviceType(context) != DeviceType.mobile) ...[
            _NavButton(
              label: 'ฟีเจอร์',
              isScrolled: _isScrolled,
              onTap: () => _scrollTo(_featuresKey),
            ),
            _NavButton(
              label: 'ราคา',
              isScrolled: _isScrolled,
              onTap: () => _scrollTo(_pricingKey),
            ),
            _NavButton(
              label: 'ติดต่อ',
              isScrolled: _isScrolled,
              onTap: () => _scrollTo(_contactKey),
            ),
          ],
          const SizedBox(width: 8),
          ElevatedButton(
            onPressed: () {},
            style: ElevatedButton.styleFrom(
              backgroundColor: Colors.blue,
              foregroundColor: Colors.white,
            ),
            child: const Text('ทดลองฟรี'),
          ),
          const SizedBox(width: 16),
        ],
      ),
      extendBodyBehindAppBar: true,
      body: SingleChildScrollView(
        controller: _scrollController,
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            // Hero Section
            _HeroSection(),
            
            // Features Section
            _FeaturesSection(key: _featuresKey),
            
            // Pricing Section
            _PricingSection(key: _pricingKey),
            
            // Testimonials
            const _TestimonialsSection(),
            
            // CTA Section
            const _CTASection(),
            
            // Footer
            _ContactSection(key: _contactKey),
          ],
        ),
      ),
    );
  }
  
  @override
  void dispose() {
    _scrollController.dispose();
    super.dispose();
  }
}

class _HeroSection extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final isDesktop = getDeviceType(context) == DeviceType.desktop;
    
    return Container(
      height: MediaQuery.of(context).size.height,
      decoration: const BoxDecoration(
        gradient: LinearGradient(
          begin: Alignment.topLeft,
          end: Alignment.bottomRight,
          colors: [Color(0xFF0D47A1), Color(0xFF1565C0), Color(0xFF1976D2)],
        ),
      ),
      child: Center(
        child: Padding(
          padding: const EdgeInsets.symmetric(horizontal: 40),
          child: isDesktop
              ? Row(
                  mainAxisAlignment: MainAxisAlignment.center,
                  children: [
                    Expanded(child: _HeroText()),
                    const SizedBox(width: 80),
                    Expanded(child: _HeroImage()),
                  ],
                )
              : Column(
                  mainAxisAlignment: MainAxisAlignment.center,
                  children: [
                    _HeroText(),
                    const SizedBox(height: 40),
                    _HeroImage(),
                  ],
                ),
        ),
      ),
    );
  }
}

class _HeroText extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      mainAxisSize: MainAxisSize.min,
      children: [
        const Text(
          'สร้างแอปสวยงาม\nได้เร็วกว่าเดิม',
          style: TextStyle(
            color: Colors.white,
            fontSize: 48,
            fontWeight: FontWeight.bold,
            height: 1.2,
          ),
        ),
        const SizedBox(height: 24),
        Text(
          'แพลตฟอร์มที่ช่วยให้นักพัฒนาสร้างแอปมือถือและเว็บ\nได้อย่างรวดเร็วและมีคุณภาพสูง',
          style: TextStyle(
            color: Colors.white.withOpacity(0.85),
            fontSize: 18,
            height: 1.6,
          ),
        ),
        const SizedBox(height: 40),
        Row(
          children: [
            ElevatedButton(
              onPressed: () {},
              style: ElevatedButton.styleFrom(
                backgroundColor: Colors.white,
                foregroundColor: Colors.blue[900],
                padding: const EdgeInsets.symmetric(
                  horizontal: 32,
                  vertical: 16,
                ),
                textStyle: const TextStyle(
                  fontSize: 16,
                  fontWeight: FontWeight.bold,
                ),
              ),
              child: const Text('เริ่มต้นฟรี'),
            ),
            const SizedBox(width: 16),
            OutlinedButton(
              onPressed: () {},
              style: OutlinedButton.styleFrom(
                foregroundColor: Colors.white,
                side: const BorderSide(color: Colors.white),
                padding: const EdgeInsets.symmetric(
                  horizontal: 32,
                  vertical: 16,
                ),
                textStyle: const TextStyle(fontSize: 16),
              ),
              child: const Text('ดูตัวอย่าง'),
            ),
          ],
        ),
      ],
    );
  }
}

class _HeroImage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Container(
      height: 400,
      decoration: BoxDecoration(
        borderRadius: BorderRadius.circular(20),
        boxShadow: [
          BoxShadow(
            color: Colors.black.withOpacity(0.3),
            blurRadius: 40,
            spreadRadius: 10,
          ),
        ],
      ),
      child: ClipRRect(
        borderRadius: BorderRadius.circular(20),
        child: Container(
          color: Colors.white.withOpacity(0.1),
          child: const Center(
            child: FlutterLogo(size: 200),
          ),
        ),
      ),
    );
  }
}

class _NavButton extends StatelessWidget {
  final String label;
  final bool isScrolled;
  final VoidCallback onTap;
  
  const _NavButton({
    required this.label,
    required this.isScrolled,
    required this.onTap,
  });
  
  @override
  Widget build(BuildContext context) {
    return TextButton(
      onPressed: onTap,
      child: Text(
        label,
        style: TextStyle(
          color: isScrolled ? Colors.black87 : Colors.white,
        ),
      ),
    );
  }
}

// Pricing Card
class _PricingCard extends StatelessWidget {
  final String plan;
  final String price;
  final List<String> features;
  final bool isPopular;
  
  const _PricingCard({
    required this.plan,
    required this.price,
    required this.features,
    this.isPopular = false,
  });
  
  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(24),
      decoration: BoxDecoration(
        color: isPopular ? Colors.blue : Colors.white,
        borderRadius: BorderRadius.circular(16),
        border: Border.all(
          color: isPopular ? Colors.blue : Colors.grey.shade200,
        ),
        boxShadow: isPopular
            ? [
                BoxShadow(
                  color: Colors.blue.withOpacity(0.3),
                  blurRadius: 20,
                  spreadRadius: 5,
                ),
              ]
            : null,
      ),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          if (isPopular)
            Container(
              padding: const EdgeInsets.symmetric(
                horizontal: 12,
                vertical: 4,
              ),
              decoration: BoxDecoration(
                color: Colors.white.withOpacity(0.2),
                borderRadius: BorderRadius.circular(20),
              ),
              child: const Text(
                'ยอดนิยม',
                style: TextStyle(color: Colors.white, fontSize: 12),
              ),
            ),
          const SizedBox(height: 16),
          Text(
            plan,
            style: TextStyle(
              fontSize: 20,
              fontWeight: FontWeight.bold,
              color: isPopular ? Colors.white : Colors.black87,
            ),
          ),
          const SizedBox(height: 8),
          RichText(
            text: TextSpan(
              children: [
                TextSpan(
                  text: price,
                  style: TextStyle(
                    fontSize: 36,
                    fontWeight: FontWeight.bold,
                    color: isPopular ? Colors.white : Colors.blue,
                  ),
                ),
                TextSpan(
                  text: '/เดือน',
                  style: TextStyle(
                    color: isPopular
                        ? Colors.white70
                        : Colors.grey,
                  ),
                ),
              ],
            ),
          ),
          const SizedBox(height: 24),
          ...features.map(
            (f) => Padding(
              padding: const EdgeInsets.only(bottom: 12),
              child: Row(
                children: [
                  Icon(
                    Icons.check_circle,
                    size: 20,
                    color: isPopular ? Colors.white : Colors.green,
                  ),
                  const SizedBox(width: 8),
                  Text(
                    f,
                    style: TextStyle(
                      color: isPopular ? Colors.white : Colors.black87,
                    ),
                  ),
                ],
              ),
            ),
          ),
          const SizedBox(height: 24),
          SizedBox(
            width: double.infinity,
            child: ElevatedButton(
              onPressed: () {},
              style: ElevatedButton.styleFrom(
                backgroundColor: isPopular ? Colors.white : Colors.blue,
                foregroundColor: isPopular ? Colors.blue : Colors.white,
                padding: const EdgeInsets.all(16),
              ),
              child: const Text('เลือกแผนนี้'),
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
- ความแตกต่างระหว่าง Flutter Mobile และ Flutter Web
- Responsive Layouts สำหรับ mobile, tablet และ desktop
- SEO พื้นฐานสำหรับ Flutter Web
- การ Deploy ไปยัง Firebase Hosting
- PWA Configuration
- Workshop: Landing Page แบบ responsive

**แบบฝึกหัดเพิ่มเติม:**
1. เพิ่ม web analytics (Google Analytics)
2. Implement lazy loading สำหรับรูปภาพ
3. สร้าง sitemap.xml อัตโนมัติ
4. เพิ่ม dark mode toggle
