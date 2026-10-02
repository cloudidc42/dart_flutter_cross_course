# Part 70: Flutter Web Advanced

## Flutter Web Overview

Flutter Web ช่วยให้คุณ deploy แอป Flutter บน browser ได้ มี 2 rendering modes:
- **HTML renderer**: ขนาดเล็ก, ดี สำหรับ text-heavy apps
- **CanvasKit renderer**: ประสิทธิภาพสูง, รองรับ graphics ซับซ้อน

```bash
# สร้าง Flutter Web app
flutter create my_web_app

# Run บน web
flutter run -d chrome

# Build สำหรับ production
flutter build web --release
```

---

## 1. Web Rendering Modes

### HTML Renderer vs CanvasKit

```bash
# HTML renderer (ค่าเริ่มต้นสำหรับ mobile web)
flutter run -d chrome --web-renderer html
flutter build web --web-renderer html

# CanvasKit renderer (ค่าเริ่มต้นสำหรับ desktop web)
flutter run -d chrome --web-renderer canvaskit
flutter build web --web-renderer canvaskit

# Auto (เลือกอัตโนมัติตาม device)
flutter build web --web-renderer auto
```

### เลือก Renderer ใน Runtime

```dart
// index.html - เลือก renderer
// CanvasKit สำหรับทุก device
// <script>
//   window.flutterWebRenderer = "canvaskit";
// </script>

// หรือตั้งค่าใน main.dart
void main() {
  // ตรวจสอบว่าอยู่บน web
  if (kIsWeb) {
    // Web-specific initialization
  }
  runApp(MyApp());
}
```

### Renderer Considerations

```dart
// ความแตกต่างระหว่าง HTML และ CanvasKit
class RendererExample extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // Custom painting - ทำงานได้ดีกับ CanvasKit
        CustomPaint(
          painter: ComplexChartPainter(),
          size: Size(400, 300),
        ),

        // Text rendering - HTML renderer ดีกว่าสำหรับ text selection
        SelectableText('เลือกข้อความนี้ได้'),

        // Filters/Blur - ต้องการ CanvasKit
        BackdropFilter(
          filter: ImageFilter.blur(sigmaX: 10, sigmaY: 10),
          child: Container(
            color: Colors.white.withOpacity(0.3),
            padding: EdgeInsets.all(16),
            child: Text('Frosted Glass Effect'),
          ),
        ),
      ],
    );
  }
}
```

---

## 2. SEO Optimization

### Meta Tags และ HTML Setup

```html
<!-- web/index.html -->
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  
  <!-- SEO Meta Tags -->
  <title>My Flutter App - แอปที่ยอดเยี่ยม</title>
  <meta name="description" content="แอปพลิเคชันที่สร้างด้วย Flutter สำหรับ...">
  <meta name="keywords" content="flutter, app, thai, mobile">
  
  <!-- Open Graph (Social Media) -->
  <meta property="og:title" content="My Flutter App">
  <meta property="og:description" content="แอปพลิเคชันที่ยอดเยี่ยม">
  <meta property="og:image" content="https://myapp.com/og-image.png">
  <meta property="og:url" content="https://myapp.com">
  <meta property="og:type" content="website">
  
  <!-- Twitter Card -->
  <meta name="twitter:card" content="summary_large_image">
  <meta name="twitter:title" content="My Flutter App">
  
  <!-- Canonical URL -->
  <link rel="canonical" href="https://myapp.com">
  
  <!-- Favicon -->
  <link rel="icon" type="image/png" href="favicon.png">
  
  <!-- Preload critical fonts -->
  <link rel="preload" href="fonts/Sarabun-Regular.ttf" as="font" crossorigin>
</head>
<body>
  <!-- Flutter app loads here -->
  <script src="flutter.js" defer></script>
  <script>
    window.addEventListener('load', function(ev) {
      _flutter.loader.load({
        onEntrypointLoaded: async function(engineInitializer) {
          let appRunner = await engineInitializer.initializeEngine();
          await appRunner.runApp();
        }
      });
    });
  </script>
</body>
</html>
```

### Dynamic Meta Tags

```dart
// services/seo_service.dart
import 'dart:html' as html;

class SeoService {
  static void updateMetaTags({
    required String title,
    required String description,
    String? imageUrl,
    String? url,
  }) {
    // อัปเดต title
    html.document.title = title;
    
    // อัปเดต meta tags
    _updateMeta('description', description);
    _updateMeta('og:title', title);
    _updateMeta('og:description', description);
    
    if (imageUrl != null) {
      _updateMeta('og:image', imageUrl);
      _updateMeta('twitter:image', imageUrl);
    }
    
    if (url != null) {
      _updateMeta('og:url', url);
      _updateCanonical(url);
    }
  }
  
  static void _updateMeta(String name, String content) {
    // ค้นหา meta tag ที่มีอยู่
    var element = html.document.querySelector(
      'meta[name="$name"], meta[property="$name"]'
    );
    
    if (element == null) {
      // สร้างใหม่ถ้าไม่มี
      element = html.MetaElement();
      if (name.startsWith('og:')) {
        element.setAttribute('property', name);
      } else {
        element.setAttribute('name', name);
      }
      html.document.head!.append(element);
    }
    
    element.setAttribute('content', content);
  }
  
  static void _updateCanonical(String url) {
    var link = html.document.querySelector('link[rel="canonical"]');
    if (link == null) {
      link = html.LinkElement();
      link.setAttribute('rel', 'canonical');
      html.document.head!.append(link);
    }
    link.setAttribute('href', url);
  }
}

// ใช้งานใน Screen
class ProductDetailScreen extends StatefulWidget {
  final String productId;
  
  const ProductDetailScreen({required this.productId});
  
  @override
  _ProductDetailScreenState createState() => _ProductDetailScreenState();
}

class _ProductDetailScreenState extends State<ProductDetailScreen> {
  Product? _product;
  
  @override
  void initState() {
    super.initState();
    _loadProduct();
  }
  
  Future<void> _loadProduct() async {
    final product = await ProductService().getProduct(widget.productId);
    setState(() => _product = product);
    
    // อัปเดต SEO meta tags
    if (kIsWeb && product != null) {
      SeoService.updateMetaTags(
        title: '${product.name} - My Shop',
        description: product.description,
        imageUrl: product.imageUrl,
        url: 'https://myshop.com/products/${product.id}',
      );
    }
  }
  
  @override
  Widget build(BuildContext context) {
    if (_product == null) return CircularProgressIndicator();
    return ProductDetail(product: _product!);
  }
}
```

---

## 3. PWA Setup

### Web App Manifest

```json
// web/manifest.json
{
  "name": "My Flutter App",
  "short_name": "MyApp",
  "description": "แอปที่สร้างด้วย Flutter",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#2196F3",
  "orientation": "portrait-primary",
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
  ],
  "categories": ["productivity", "utilities"],
  "lang": "th",
  "screenshots": [
    {
      "src": "screenshots/app-screenshot.png",
      "sizes": "1280x720",
      "type": "image/png"
    }
  ]
}
```

### Service Worker

```javascript
// web/flutter_service_worker.js (customize)
const CACHE_VERSION = 'v1.2.3';
const CACHE_NAME = `my-flutter-app-${CACHE_VERSION}`;

const STATIC_CACHE_URLS = [
  '/',
  '/index.html',
  '/main.dart.js',
  '/flutter.js',
  '/manifest.json',
  '/icons/Icon-192.png',
  '/icons/Icon-512.png',
  '/fonts/Sarabun-Regular.ttf',
];

// Install
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => {
      return cache.addAll(STATIC_CACHE_URLS);
    })
  );
});

// Activate - clean old caches
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((cacheNames) => {
      return Promise.all(
        cacheNames
          .filter((name) => name !== CACHE_NAME)
          .map((name) => caches.delete(name))
      );
    })
  );
});

// Fetch
self.addEventListener('fetch', (event) => {
  // Network first for API calls
  if (event.request.url.includes('/api/')) {
    event.respondWith(
      fetch(event.request)
        .catch(() => caches.match(event.request))
    );
    return;
  }
  
  // Cache first for static assets
  event.respondWith(
    caches.match(event.request).then((cached) => {
      return cached || fetch(event.request).then((response) => {
        const clone = response.clone();
        caches.open(CACHE_NAME).then((cache) => {
          cache.put(event.request, clone);
        });
        return response;
      });
    })
  );
});
```

### PWA Install Prompt

```dart
// widgets/pwa_install_prompt.dart
import 'dart:html' as html;
import 'dart:js' as js;

class PWAInstallPrompt extends StatefulWidget {
  @override
  _PWAInstallPromptState createState() => _PWAInstallPromptState();
}

class _PWAInstallPromptState extends State<PWAInstallPrompt> {
  bool _canInstall = false;
  dynamic _installPrompt;

  @override
  void initState() {
    super.initState();
    
    if (kIsWeb) {
      // Listen for beforeinstallprompt event
      html.window.addEventListener('beforeinstallprompt', (event) {
        event.preventDefault();
        setState(() {
          _canInstall = true;
          _installPrompt = event;
        });
      });
    }
  }

  void _handleInstall() {
    if (_installPrompt != null) {
      js.JsObject.fromBrowserObject(_installPrompt).callMethod('prompt');
      setState(() {
        _canInstall = false;
        _installPrompt = null;
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    if (!_canInstall) return SizedBox.shrink();
    
    return Card(
      margin: EdgeInsets.all(16),
      child: Padding(
        padding: EdgeInsets.all(16),
        child: Row(
          children: [
            Icon(Icons.install_mobile, color: Colors.blue),
            SizedBox(width: 12),
            Expanded(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text('ติดตั้งแอปบนอุปกรณ์ของคุณ',
                      style: TextStyle(fontWeight: FontWeight.bold)),
                  Text('เข้าถึงได้ง่ายขึ้นโดยไม่ต้องเปิด browser',
                      style: TextStyle(fontSize: 12, color: Colors.grey[600])),
                ],
              ),
            ),
            TextButton(
              onPressed: () => setState(() => _canInstall = false),
              child: Text('ภายหลัง'),
            ),
            ElevatedButton(
              onPressed: _handleInstall,
              child: Text('ติดตั้ง'),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## 4. URL Strategy

### Path URL Strategy

```dart
// main.dart - ใช้ path-based URLs (ไม่มี #)
import 'package:flutter_web_plugins/url_strategy.dart';

void main() {
  // ใช้ path URL strategy แทน hash strategy
  // path: https://myapp.com/profile (ดีกว่า)
  // hash: https://myapp.com/#/profile
  usePathUrlStrategy();
  
  runApp(MyApp());
}
```

### History API

```dart
// Web navigation ด้วย GoRouter + URL Strategy
final GoRouter _router = GoRouter(
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => HomeScreen(),
    ),
    GoRoute(
      path: '/products',
      builder: (context, state) => ProductsScreen(),
    ),
    GoRoute(
      path: '/products/:id',
      builder: (context, state) {
        final id = state.pathParameters['id']!;
        return ProductDetailScreen(productId: id);
      },
    ),
    GoRoute(
      path: '/blog/:slug',
      builder: (context, state) {
        final slug = state.pathParameters['slug']!;
        return BlogPostScreen(slug: slug);
      },
    ),
  ],
);
```

---

## 5. Web Plugins

### Web-specific Implementations

```dart
// lib/web_utils.dart
import 'dart:html' as html;

class WebUtils {
  // Copy to clipboard
  static Future<void> copyToClipboard(String text) async {
    await html.window.navigator.clipboard?.writeText(text);
  }

  // Download file
  static void downloadFile({
    required String content,
    required String fileName,
    String mimeType = 'text/plain',
  }) {
    final bytes = utf8.encode(content);
    final blob = html.Blob([bytes], mimeType);
    final url = html.Url.createObjectUrlFromBlob(blob);
    
    final anchor = html.AnchorElement(href: url)
      ..setAttribute('download', fileName)
      ..click();
    
    html.Url.revokeObjectUrl(url);
  }

  // Open URL in new tab
  static void openInNewTab(String url) {
    html.window.open(url, '_blank');
  }

  // Get URL parameters
  static Map<String, String> getUrlParams() {
    return Uri.parse(html.window.location.href).queryParameters;
  }

  // Set URL without navigation
  static void updateUrl(String path) {
    html.window.history.pushState(null, '', path);
  }

  // Detect if mobile browser
  static bool get isMobileBrowser {
    final userAgent = html.window.navigator.userAgent.toLowerCase();
    return userAgent.contains('mobile') || userAgent.contains('android');
  }

  // Share via Web Share API
  static Future<bool> shareContent({
    required String title,
    required String text,
    String? url,
  }) async {
    if (html.window.navigator.share == null) return false;
    
    try {
      await html.window.navigator.share({
        'title': title,
        'text': text,
        if (url != null) 'url': url,
      });
      return true;
    } catch (e) {
      return false;
    }
  }
}
```

---

## Workshop: Web Portfolio Site

```dart
// Portfolio website สมบูรณ์ด้วย Flutter Web

// main.dart
void main() {
  usePathUrlStrategy();
  runApp(PortfolioApp());
}

class PortfolioApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      title: 'Portfolio - ชื่อของฉัน',
      routerConfig: _router,
      theme: ThemeData(
        colorScheme: ColorScheme.dark(
          primary: Color(0xFF64FFDA), // Teal accent
          background: Color(0xFF0A192F),
        ),
        useMaterial3: true,
      ),
    );
  }
}

final _router = GoRouter(
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => PortfolioScreen(),
    ),
    GoRoute(
      path: '/projects/:id',
      builder: (context, state) => ProjectDetailScreen(
        id: state.pathParameters['id']!,
      ),
    ),
  ],
);

// screens/portfolio_screen.dart
class PortfolioScreen extends StatefulWidget {
  @override
  _PortfolioScreenState createState() => _PortfolioScreenState();
}

class _PortfolioScreenState extends State<PortfolioScreen> {
  final _scrollController = ScrollController();
  final _heroKey = GlobalKey();
  final _aboutKey = GlobalKey();
  final _projectsKey = GlobalKey();
  final _contactKey = GlobalKey();

  @override
  void initState() {
    super.initState();
    // Update SEO on page load
    if (kIsWeb) {
      SeoService.updateMetaTags(
        title: 'Portfolio - สมชาย ดี | Flutter Developer',
        description: 'Portfolio ของ Flutter Developer ผู้เชี่ยวชาญ Mobile และ Web',
        imageUrl: 'https://myportfolio.com/og-image.jpg',
        url: 'https://myportfolio.com',
      );
    }
  }

  @override
  void dispose() {
    _scrollController.dispose();
    super.dispose();
  }

  void _scrollTo(GlobalKey key) {
    final context = key.currentContext;
    if (context != null) {
      Scrollable.ensureVisible(
        context,
        duration: Duration(milliseconds: 500),
        curve: Curves.easeInOut,
      );
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Color(0xFF0A192F),
      body: Stack(
        children: [
          // Scrollable content
          SingleChildScrollView(
            controller: _scrollController,
            child: Column(
              children: [
                // Navigation space
                SizedBox(height: 80),

                // Hero section
                HeroSection(key: _heroKey),

                // About section
                AboutSection(key: _aboutKey),

                // Projects section
                ProjectsSection(key: _projectsKey),

                // Contact section
                ContactSection(key: _contactKey),

                // Footer
                FooterWidget(),
              ],
            ),
          ),

          // Fixed navbar
          Positioned(
            top: 0,
            left: 0,
            right: 0,
            child: PortfolioNavbar(
              onNavTap: (section) {
                switch (section) {
                  case 'home':
                    _scrollTo(_heroKey);
                    break;
                  case 'about':
                    _scrollTo(_aboutKey);
                    break;
                  case 'projects':
                    _scrollTo(_projectsKey);
                    break;
                  case 'contact':
                    _scrollTo(_contactKey);
                    break;
                }
              },
            ),
          ),
        ],
      ),
    );
  }
}

// sections/hero_section.dart
class HeroSection extends StatefulWidget {
  const HeroSection({Key? key}) : super(key: key);

  @override
  _HeroSectionState createState() => _HeroSectionState();
}

class _HeroSectionState extends State<HeroSection>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _fadeAnimation;
  late Animation<Offset> _slideAnimation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: Duration(milliseconds: 1500),
    );

    _fadeAnimation = Tween<double>(begin: 0, end: 1).animate(
      CurvedAnimation(parent: _controller, curve: Curves.easeOut),
    );

    _slideAnimation = Tween<Offset>(
      begin: Offset(0, 0.3),
      end: Offset.zero,
    ).animate(CurvedAnimation(
      parent: _controller,
      curve: Curves.easeOut,
    ));

    _controller.forward();
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Container(
      height: MediaQuery.of(context).size.height - 80,
      padding: EdgeInsets.symmetric(horizontal: 64, vertical: 40),
      child: Center(
        child: ConstrainedBox(
          constraints: BoxConstraints(maxWidth: 900),
          child: FadeTransition(
            opacity: _fadeAnimation,
            child: SlideTransition(
              position: _slideAnimation,
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(
                    'สวัสดี ฉันชื่อ',
                    style: TextStyle(
                      color: Color(0xFF64FFDA),
                      fontSize: 16,
                      fontFamily: 'Fira Code',
                    ),
                  ),
                  SizedBox(height: 16),
                  Text(
                    'สมชาย ดี',
                    style: TextStyle(
                      color: Colors.white,
                      fontSize: MediaQuery.of(context).size.width > 600 ? 72 : 48,
                      fontWeight: FontWeight.bold,
                      height: 1.1,
                    ),
                  ),
                  Text(
                    'สร้างแอปที่ยอดเยี่ยม',
                    style: TextStyle(
                      color: Colors.grey[400],
                      fontSize: MediaQuery.of(context).size.width > 600 ? 72 : 48,
                      fontWeight: FontWeight.bold,
                      height: 1.1,
                    ),
                  ),
                  SizedBox(height: 24),
                  SizedBox(
                    width: 500,
                    child: Text(
                      'ฉันเป็น Flutter Developer ที่เชี่ยวชาญในการสร้าง '
                      'Mobile, Web, และ Desktop Applications ที่มีคุณภาพสูง',
                      style: TextStyle(
                        color: Colors.grey[400],
                        fontSize: 16,
                        height: 1.6,
                      ),
                    ),
                  ),
                  SizedBox(height: 40),
                  Row(
                    children: [
                      OutlinedButton(
                        onPressed: () {
                          final context = _projectsKey.currentContext;
                          if (context != null) {
                            Scrollable.ensureVisible(context,
                                duration: Duration(milliseconds: 500));
                          }
                        },
                        style: OutlinedButton.styleFrom(
                          foregroundColor: Color(0xFF64FFDA),
                          side: BorderSide(color: Color(0xFF64FFDA)),
                          padding: EdgeInsets.symmetric(
                            horizontal: 32,
                            vertical: 16,
                          ),
                        ),
                        child: Text('ดูผลงาน'),
                      ),
                      SizedBox(width: 16),
                      ElevatedButton(
                        onPressed: () {
                          WebUtils.downloadFile(
                            content: 'Resume content...',
                            fileName: 'resume.pdf',
                          );
                        },
                        style: ElevatedButton.styleFrom(
                          backgroundColor: Color(0xFF64FFDA),
                          foregroundColor: Color(0xFF0A192F),
                          padding: EdgeInsets.symmetric(
                            horizontal: 32,
                            vertical: 16,
                          ),
                        ),
                        child: Text('Download Resume'),
                      ),
                    ],
                  ),
                ],
              ),
            ),
          ),
        ),
      ),
    );
  }
}

// sections/projects_section.dart
class ProjectsSection extends StatelessWidget {
  const ProjectsSection({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    final projects = [
      ProjectData(
        id: '1',
        title: 'E-Commerce App',
        description: 'แอปช้อปปิ้งสมบูรณ์แบบด้วย Flutter',
        tags: ['Flutter', 'Firebase', 'Stripe'],
        imageUrl: 'assets/projects/ecommerce.png',
        githubUrl: 'https://github.com/...',
        demoUrl: 'https://demo.com/...',
      ),
      ProjectData(
        id: '2',
        title: 'Dashboard Web App',
        description: 'Dashboard สำหรับ analytics ด้วย Flutter Web',
        tags: ['Flutter Web', 'GraphQL', 'Charts'],
        imageUrl: 'assets/projects/dashboard.png',
        githubUrl: 'https://github.com/...',
        demoUrl: 'https://demo.com/...',
      ),
    ];

    return Container(
      padding: EdgeInsets.symmetric(horizontal: 64, vertical: 80),
      child: ConstrainedBox(
        constraints: BoxConstraints(maxWidth: 1000),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              children: [
                Text(
                  '03.',
                  style: TextStyle(
                    color: Color(0xFF64FFDA),
                    fontSize: 20,
                    fontFamily: 'Fira Code',
                  ),
                ),
                SizedBox(width: 8),
                Text(
                  'ผลงานที่ผ่านมา',
                  style: TextStyle(
                    color: Colors.white,
                    fontSize: 28,
                    fontWeight: FontWeight.bold,
                  ),
                ),
              ],
            ),
            SizedBox(height: 48),
            ...projects.map((project) => ProjectCard(project: project)),
          ],
        ),
      ),
    );
  }
}

class ProjectCard extends StatefulWidget {
  final ProjectData project;

  const ProjectCard({required this.project});

  @override
  _ProjectCardState createState() => _ProjectCardState();
}

class _ProjectCardState extends State<ProjectCard> {
  bool _isHovered = false;

  @override
  Widget build(BuildContext context) {
    return MouseRegion(
      onEnter: (_) => setState(() => _isHovered = true),
      onExit: (_) => setState(() => _isHovered = false),
      child: AnimatedContainer(
        duration: Duration(milliseconds: 200),
        margin: EdgeInsets.only(bottom: 32),
        padding: EdgeInsets.all(24),
        decoration: BoxDecoration(
          color: _isHovered
              ? Color(0xFF112240)
              : Color(0xFF0D1B2E),
          borderRadius: BorderRadius.circular(8),
          border: Border.all(
            color: _isHovered
                ? Color(0xFF64FFDA).withOpacity(0.5)
                : Colors.transparent,
          ),
          boxShadow: _isHovered
              ? [
                  BoxShadow(
                    color: Color(0xFF64FFDA).withOpacity(0.1),
                    blurRadius: 20,
                    offset: Offset(0, 10),
                  ),
                ]
              : [],
        ),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                Text(
                  widget.project.title,
                  style: TextStyle(
                    color: Colors.white,
                    fontSize: 20,
                    fontWeight: FontWeight.bold,
                  ),
                ),
                Row(
                  children: [
                    if (widget.project.githubUrl != null)
                      IconButton(
                        icon: Icon(Icons.code, color: Colors.grey[400]),
                        onPressed: () =>
                            WebUtils.openInNewTab(widget.project.githubUrl!),
                        tooltip: 'ดู Source Code',
                      ),
                    if (widget.project.demoUrl != null)
                      IconButton(
                        icon: Icon(Icons.open_in_new, color: Colors.grey[400]),
                        onPressed: () =>
                            WebUtils.openInNewTab(widget.project.demoUrl!),
                        tooltip: 'ดู Demo',
                      ),
                  ],
                ),
              ],
            ),
            SizedBox(height: 12),
            Text(
              widget.project.description,
              style: TextStyle(color: Colors.grey[400], height: 1.6),
            ),
            SizedBox(height: 16),
            Wrap(
              spacing: 8,
              runSpacing: 8,
              children: widget.project.tags
                  .map((tag) => Container(
                        padding: EdgeInsets.symmetric(
                          horizontal: 12,
                          vertical: 4,
                        ),
                        decoration: BoxDecoration(
                          color: Color(0xFF1E3A5F),
                          borderRadius: BorderRadius.circular(4),
                        ),
                        child: Text(
                          tag,
                          style: TextStyle(
                            color: Color(0xFF64FFDA),
                            fontSize: 12,
                            fontFamily: 'Fira Code',
                          ),
                        ),
                      ))
                  .toList(),
            ),
          ],
        ),
      ),
    );
  }
}

class ProjectData {
  final String id;
  final String title;
  final String description;
  final List<String> tags;
  final String imageUrl;
  final String? githubUrl;
  final String? demoUrl;

  ProjectData({
    required this.id,
    required this.title,
    required this.description,
    required this.tags,
    required this.imageUrl,
    this.githubUrl,
    this.demoUrl,
  });
}
```

---

## สรุป

Flutter Web Advanced ประกอบด้วย:

1. **Rendering Modes** - HTML vs CanvasKit, ข้อดีข้อเสีย
2. **SEO** - meta tags, Open Graph, canonical URLs
3. **PWA** - manifest.json, Service Worker, install prompt
4. **URL Strategy** - path-based URLs ด้วย usePathUrlStrategy
5. **Web Plugins** - Web Share API, clipboard, file download
6. **Portfolio** - ตัวอย่าง website สมบูรณ์แบบด้วย Flutter Web

---

## จบ Course Flutter ขั้นสูง

ยินดีด้วย! คุณได้เรียนรู้ topics สำคัญทั้งหมดแล้ว:

- **Part 56**: Memory Management
- **Part 57**: GoRouter Navigation
- **Part 58**: Internationalization
- **Part 59**: Accessibility
- **Part 60**: CI/CD Pipeline
- **Part 61**: App Distribution
- **Part 62**: Crashlytics & Analytics
- **Part 63**: Security Best Practices
- **Part 64**: Offline-First Architecture
- **Part 65**: GraphQL Integration
- **Part 66**: WebSocket Real-time
- **Part 67**: Responsive Design
- **Part 68**: Plugin Development
- **Part 69**: Desktop Applications
- **Part 70**: Flutter Web Advanced

ขอให้โชคดีในการพัฒนา Flutter App!
