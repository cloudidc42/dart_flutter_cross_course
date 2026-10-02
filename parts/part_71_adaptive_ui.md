# Part 71: Adaptive UI - สร้าง UI ที่ปรับตัวตามแพลตฟอร์ม

## บทนำ

Adaptive UI คือแนวทางการออกแบบ UI ที่ปรับตัวให้เหมาะสมกับแพลตฟอร์ม อุปกรณ์ และขนาดหน้าจอที่แตกต่างกัน Flutter มีความสามารถพิเศษในการสร้าง UI ที่ดูเป็นธรรมชาติบนทั้ง iOS, Android, Web และ Desktop ด้วยโค้ดชุดเดียว

## 1. Platform-specific UI (Material vs Cupertino)

### ความแตกต่างระหว่าง Material และ Cupertino

Material Design เป็น design language ของ Google ที่ใช้บน Android ส่วน Cupertino เป็น design language ของ Apple ที่ใช้บน iOS ทั้งสองมีความแตกต่างทั้งด้านรูปลักษณ์และ UX

```dart
// lib/adaptive/platform_detector.dart
import 'dart:io';
import 'package:flutter/foundation.dart';

enum AppPlatform {
  android,
  ios,
  web,
  macos,
  windows,
  linux,
}

class PlatformDetector {
  static AppPlatform get current {
    if (kIsWeb) return AppPlatform.web;
    if (Platform.isAndroid) return AppPlatform.android;
    if (Platform.isIOS) return AppPlatform.ios;
    if (Platform.isMacOS) return AppPlatform.macos;
    if (Platform.isWindows) return AppPlatform.windows;
    if (Platform.isLinux) return AppPlatform.linux;
    return AppPlatform.android; // default
  }

  static bool get isApple =>
      current == AppPlatform.ios || current == AppPlatform.macos;

  static bool get isMobile =>
      current == AppPlatform.android || current == AppPlatform.ios;

  static bool get isDesktop =>
      current == AppPlatform.macos ||
      current == AppPlatform.windows ||
      current == AppPlatform.linux;
}
```

### สร้าง Adaptive App

```dart
// lib/main.dart
import 'package:flutter/cupertino.dart';
import 'package:flutter/material.dart';
import 'adaptive/platform_detector.dart';

void main() {
  runApp(const AdaptiveApp());
}

class AdaptiveApp extends StatelessWidget {
  const AdaptiveApp({super.key});

  @override
  Widget build(BuildContext context) {
    // ใช้ CupertinoApp สำหรับ iOS/macOS และ MaterialApp สำหรับอื่น ๆ
    if (PlatformDetector.isApple) {
      return CupertinoApp(
        title: 'Adaptive App',
        theme: const CupertinoThemeData(
          primaryColor: CupertinoColors.systemBlue,
          brightness: Brightness.light,
        ),
        home: const AdaptiveHomePage(),
      );
    }

    return MaterialApp(
      title: 'Adaptive App',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue),
        useMaterial3: true,
      ),
      home: const AdaptiveHomePage(),
    );
  }
}
```

### Adaptive Navigation

```dart
// lib/adaptive/adaptive_navigation.dart
import 'package:flutter/cupertino.dart';
import 'package:flutter/material.dart';
import 'platform_detector.dart';

class AdaptiveNavigationBar extends StatelessWidget {
  final int currentIndex;
  final ValueChanged<int> onTap;
  final List<AdaptiveNavigationItem> items;

  const AdaptiveNavigationBar({
    super.key,
    required this.currentIndex,
    required this.onTap,
    required this.items,
  });

  @override
  Widget build(BuildContext context) {
    if (PlatformDetector.isApple) {
      return CupertinoTabBar(
        currentIndex: currentIndex,
        onTap: onTap,
        items: items
            .map((item) => BottomNavigationBarItem(
                  icon: Icon(item.icon),
                  label: item.label,
                ))
            .toList(),
      );
    }

    return NavigationBar(
      selectedIndex: currentIndex,
      onDestinationSelected: onTap,
      destinations: items
          .map((item) => NavigationDestination(
                icon: Icon(item.icon),
                label: item.label,
              ))
          .toList(),
    );
  }
}

class AdaptiveNavigationItem {
  final IconData icon;
  final String label;

  const AdaptiveNavigationItem({
    required this.icon,
    required this.label,
  });
}
```

## 2. Adaptive Widgets

### Adaptive Button

```dart
// lib/adaptive/adaptive_button.dart
import 'package:flutter/cupertino.dart';
import 'package:flutter/material.dart';
import 'platform_detector.dart';

class AdaptiveButton extends StatelessWidget {
  final String text;
  final VoidCallback? onPressed;
  final bool isPrimary;
  final bool isDestructive;

  const AdaptiveButton({
    super.key,
    required this.text,
    this.onPressed,
    this.isPrimary = true,
    this.isDestructive = false,
  });

  @override
  Widget build(BuildContext context) {
    if (PlatformDetector.isApple) {
      if (isDestructive) {
        return CupertinoButton(
          onPressed: onPressed,
          child: Text(
            text,
            style: const TextStyle(color: CupertinoColors.destructiveRed),
          ),
        );
      }

      if (isPrimary) {
        return CupertinoButton.filled(
          onPressed: onPressed,
          child: Text(text),
        );
      }

      return CupertinoButton(
        onPressed: onPressed,
        child: Text(text),
      );
    }

    if (isDestructive) {
      return TextButton(
        onPressed: onPressed,
        style: TextButton.styleFrom(
          foregroundColor: Theme.of(context).colorScheme.error,
        ),
        child: Text(text),
      );
    }

    if (isPrimary) {
      return FilledButton(
        onPressed: onPressed,
        child: Text(text),
      );
    }

    return OutlinedButton(
      onPressed: onPressed,
      child: Text(text),
    );
  }
}
```

### Adaptive TextField

```dart
// lib/adaptive/adaptive_text_field.dart
import 'package:flutter/cupertino.dart';
import 'package:flutter/material.dart';
import 'platform_detector.dart';

class AdaptiveTextField extends StatelessWidget {
  final String? placeholder;
  final TextEditingController? controller;
  final bool obscureText;
  final TextInputType keyboardType;
  final ValueChanged<String>? onChanged;
  final String? errorText;

  const AdaptiveTextField({
    super.key,
    this.placeholder,
    this.controller,
    this.obscureText = false,
    this.keyboardType = TextInputType.text,
    this.onChanged,
    this.errorText,
  });

  @override
  Widget build(BuildContext context) {
    if (PlatformDetector.isApple) {
      return Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          CupertinoTextField(
            controller: controller,
            placeholder: placeholder,
            obscureText: obscureText,
            keyboardType: keyboardType,
            onChanged: onChanged,
            padding: const EdgeInsets.symmetric(
              horizontal: 16,
              vertical: 12,
            ),
            decoration: BoxDecoration(
              border: Border.all(
                color: errorText != null
                    ? CupertinoColors.destructiveRed
                    : CupertinoColors.systemGrey4,
              ),
              borderRadius: BorderRadius.circular(8),
            ),
          ),
          if (errorText != null) ...[
            const SizedBox(height: 4),
            Text(
              errorText!,
              style: const TextStyle(
                color: CupertinoColors.destructiveRed,
                fontSize: 12,
              ),
            ),
          ],
        ],
      );
    }

    return TextField(
      controller: controller,
      obscureText: obscureText,
      keyboardType: keyboardType,
      onChanged: onChanged,
      decoration: InputDecoration(
        hintText: placeholder,
        border: const OutlineInputBorder(),
        errorText: errorText,
      ),
    );
  }
}
```

### Adaptive Switch

```dart
// lib/adaptive/adaptive_switch.dart
import 'package:flutter/cupertino.dart';
import 'package:flutter/material.dart';
import 'platform_detector.dart';

class AdaptiveSwitch extends StatelessWidget {
  final bool value;
  final ValueChanged<bool>? onChanged;

  const AdaptiveSwitch({
    super.key,
    required this.value,
    this.onChanged,
  });

  @override
  Widget build(BuildContext context) {
    if (PlatformDetector.isApple) {
      return CupertinoSwitch(
        value: value,
        onChanged: onChanged,
      );
    }

    return Switch(
      value: value,
      onChanged: onChanged,
    );
  }
}
```

### Adaptive Dialog

```dart
// lib/adaptive/adaptive_dialog.dart
import 'package:flutter/cupertino.dart';
import 'package:flutter/material.dart';
import 'platform_detector.dart';

class AdaptiveDialogAction {
  final String text;
  final VoidCallback onPressed;
  final bool isDestructive;
  final bool isDefaultAction;

  const AdaptiveDialogAction({
    required this.text,
    required this.onPressed,
    this.isDestructive = false,
    this.isDefaultAction = false,
  });
}

Future<void> showAdaptiveDialog({
  required BuildContext context,
  required String title,
  required String message,
  required List<AdaptiveDialogAction> actions,
}) async {
  if (PlatformDetector.isApple) {
    await showCupertinoDialog(
      context: context,
      builder: (context) => CupertinoAlertDialog(
        title: Text(title),
        content: Text(message),
        actions: actions
            .map((action) => CupertinoDialogAction(
                  isDestructiveAction: action.isDestructive,
                  isDefaultAction: action.isDefaultAction,
                  onPressed: action.onPressed,
                  child: Text(action.text),
                ))
            .toList(),
      ),
    );
    return;
  }

  await showDialog(
    context: context,
    builder: (context) => AlertDialog(
      title: Text(title),
      content: Text(message),
      actions: actions
          .map((action) => TextButton(
                onPressed: action.onPressed,
                style: action.isDestructive
                    ? TextButton.styleFrom(
                        foregroundColor: Theme.of(context).colorScheme.error,
                      )
                    : null,
                child: Text(action.text),
              ))
          .toList(),
    ),
  );
}
```

## 3. Form Factor Detection

### Screen Size Detection

```dart
// lib/adaptive/screen_size.dart
import 'package:flutter/material.dart';

enum ScreenSize {
  compact,  // < 600dp (phone portrait)
  medium,   // 600-840dp (phone landscape, tablet portrait)
  expanded, // > 840dp (tablet landscape, desktop)
}

class ScreenSizeHelper {
  static ScreenSize of(BuildContext context) {
    final width = MediaQuery.of(context).size.width;

    if (width < 600) return ScreenSize.compact;
    if (width < 840) return ScreenSize.medium;
    return ScreenSize.expanded;
  }

  static bool isCompact(BuildContext context) =>
      of(context) == ScreenSize.compact;

  static bool isMedium(BuildContext context) =>
      of(context) == ScreenSize.medium;

  static bool isExpanded(BuildContext context) =>
      of(context) == ScreenSize.expanded;
}

// Responsive layout builder
class ResponsiveLayout extends StatelessWidget {
  final Widget compact;
  final Widget? medium;
  final Widget? expanded;

  const ResponsiveLayout({
    super.key,
    required this.compact,
    this.medium,
    this.expanded,
  });

  @override
  Widget build(BuildContext context) {
    final screenSize = ScreenSizeHelper.of(context);

    switch (screenSize) {
      case ScreenSize.expanded:
        return expanded ?? medium ?? compact;
      case ScreenSize.medium:
        return medium ?? compact;
      case ScreenSize.compact:
        return compact;
    }
  }
}
```

### Adaptive Grid

```dart
// lib/adaptive/adaptive_grid.dart
import 'package:flutter/material.dart';
import 'screen_size.dart';

class AdaptiveGrid extends StatelessWidget {
  final List<Widget> children;
  final double spacing;
  final double runSpacing;

  const AdaptiveGrid({
    super.key,
    required this.children,
    this.spacing = 16,
    this.runSpacing = 16,
  });

  int _crossAxisCount(BuildContext context) {
    final screenSize = ScreenSizeHelper.of(context);
    switch (screenSize) {
      case ScreenSize.compact:
        return 2;
      case ScreenSize.medium:
        return 3;
      case ScreenSize.expanded:
        return 4;
    }
  }

  @override
  Widget build(BuildContext context) {
    return GridView.builder(
      gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
        crossAxisCount: _crossAxisCount(context),
        crossAxisSpacing: spacing,
        mainAxisSpacing: runSpacing,
      ),
      itemCount: children.length,
      itemBuilder: (context, index) => children[index],
    );
  }
}
```

### Adaptive Navigation (Rail vs Bar)

```dart
// lib/adaptive/adaptive_scaffold.dart
import 'package:flutter/material.dart';
import 'screen_size.dart';

class AdaptiveScaffold extends StatefulWidget {
  final List<AdaptiveDestination> destinations;
  final List<Widget> pages;
  final String title;

  const AdaptiveScaffold({
    super.key,
    required this.destinations,
    required this.pages,
    required this.title,
  });

  @override
  State<AdaptiveScaffold> createState() => _AdaptiveScaffoldState();
}

class _AdaptiveScaffoldState extends State<AdaptiveScaffold> {
  int _selectedIndex = 0;

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        final screenSize = ScreenSizeHelper.of(context);

        if (screenSize == ScreenSize.expanded) {
          return _buildDesktopLayout();
        }

        if (screenSize == ScreenSize.medium) {
          return _buildTabletLayout();
        }

        return _buildMobileLayout();
      },
    );
  }

  Widget _buildDesktopLayout() {
    return Scaffold(
      body: Row(
        children: [
          NavigationRail(
            selectedIndex: _selectedIndex,
            onDestinationSelected: (index) {
              setState(() => _selectedIndex = index);
            },
            extended: true,
            destinations: widget.destinations
                .map((d) => NavigationRailDestination(
                      icon: Icon(d.icon),
                      label: Text(d.label),
                    ))
                .toList(),
          ),
          const VerticalDivider(width: 1),
          Expanded(
            child: widget.pages[_selectedIndex],
          ),
        ],
      ),
    );
  }

  Widget _buildTabletLayout() {
    return Scaffold(
      body: Row(
        children: [
          NavigationRail(
            selectedIndex: _selectedIndex,
            onDestinationSelected: (index) {
              setState(() => _selectedIndex = index);
            },
            destinations: widget.destinations
                .map((d) => NavigationRailDestination(
                      icon: Icon(d.icon),
                      label: Text(d.label),
                    ))
                .toList(),
          ),
          const VerticalDivider(width: 1),
          Expanded(
            child: widget.pages[_selectedIndex],
          ),
        ],
      ),
    );
  }

  Widget _buildMobileLayout() {
    return Scaffold(
      appBar: AppBar(title: Text(widget.title)),
      body: widget.pages[_selectedIndex],
      bottomNavigationBar: NavigationBar(
        selectedIndex: _selectedIndex,
        onDestinationSelected: (index) {
          setState(() => _selectedIndex = index);
        },
        destinations: widget.destinations
            .map((d) => NavigationDestination(
                  icon: Icon(d.icon),
                  label: d.label,
                ))
            .toList(),
      ),
    );
  }
}

class AdaptiveDestination {
  final IconData icon;
  final String label;

  const AdaptiveDestination({
    required this.icon,
    required this.label,
  });
}
```

## 4. Workshop: Adaptive News Reader

### Project Structure
```
adaptive_news_reader/
├── lib/
│   ├── adaptive/
│   │   ├── adaptive_scaffold.dart
│   │   ├── adaptive_card.dart
│   │   └── screen_size.dart
│   ├── models/
│   │   └── article.dart
│   ├── screens/
│   │   ├── home_screen.dart
│   │   ├── article_list_screen.dart
│   │   └── article_detail_screen.dart
│   └── main.dart
```

### Article Model

```dart
// lib/models/article.dart
class Article {
  final String id;
  final String title;
  final String summary;
  final String content;
  final String imageUrl;
  final String category;
  final DateTime publishedAt;
  final String author;

  const Article({
    required this.id,
    required this.title,
    required this.summary,
    required this.content,
    required this.imageUrl,
    required this.category,
    required this.publishedAt,
    required this.author,
  });

  static List<Article> getMockArticles() {
    return [
      Article(
        id: '1',
        title: 'Flutter 4.0 ประกาศฟีเจอร์ใหม่สุดอลัง',
        summary:
            'Google ประกาศ Flutter 4.0 พร้อมฟีเจอร์ใหม่มากมาย รวมถึง Adaptive UI อัตโนมัติ',
        content: '''
Flutter 4.0 มาพร้อมกับฟีเจอร์ที่นักพัฒนารอคอย...

การปรับปรุงประสิทธิภาพมากกว่า 30%...
        ''',
        imageUrl: 'https://picsum.photos/seed/flutter/800/400',
        category: 'Technology',
        publishedAt: DateTime.now().subtract(const Duration(hours: 2)),
        author: 'Tech Reporter',
      ),
      Article(
        id: '2',
        title: 'Dart 4.0: Macros และ Static Metaprogramming',
        summary: 'Dart ทีมประกาศ Macros ระบบ code generation ที่ทรงพลัง',
        content: '''
Dart Macros คือฟีเจอร์ที่จะเปลี่ยนวิธีเขียน Dart ไปตลอดกาล...
        ''',
        imageUrl: 'https://picsum.photos/seed/dart/800/400',
        category: 'Programming',
        publishedAt: DateTime.now().subtract(const Duration(hours: 5)),
        author: 'Dart Team',
      ),
      Article(
        id: '3',
        title: 'AI ใน Mobile App: ทำได้จริงแค่ไหน?',
        summary: 'สำรวจความเป็นไปได้ของ AI on-device บน Flutter apps',
        content: '''
การรัน AI model บนอุปกรณ์มือถือกลายเป็นความเป็นไปได้...
        ''',
        imageUrl: 'https://picsum.photos/seed/ai/800/400',
        category: 'AI',
        publishedAt: DateTime.now().subtract(const Duration(days: 1)),
        author: 'AI Researcher',
      ),
    ];
  }
}
```

### Adaptive News Card

```dart
// lib/adaptive/adaptive_card.dart
import 'package:flutter/material.dart';
import '../models/article.dart';
import 'screen_size.dart';

class AdaptiveNewsCard extends StatelessWidget {
  final Article article;
  final VoidCallback? onTap;

  const AdaptiveNewsCard({
    super.key,
    required this.article,
    this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    final screenSize = ScreenSizeHelper.of(context);

    if (screenSize == ScreenSize.compact) {
      return _buildCompactCard(context);
    }

    return _buildExpandedCard(context);
  }

  Widget _buildCompactCard(BuildContext context) {
    return Card(
      margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
      child: InkWell(
        onTap: onTap,
        child: Padding(
          padding: const EdgeInsets.all(12),
          child: Row(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              ClipRRect(
                borderRadius: BorderRadius.circular(8),
                child: Image.network(
                  article.imageUrl,
                  width: 80,
                  height: 80,
                  fit: BoxFit.cover,
                  errorBuilder: (context, error, stackTrace) => Container(
                    width: 80,
                    height: 80,
                    color: Colors.grey[300],
                    child: const Icon(Icons.image),
                  ),
                ),
              ),
              const SizedBox(width: 12),
              Expanded(
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Chip(
                      label: Text(
                        article.category,
                        style: const TextStyle(fontSize: 10),
                      ),
                      padding: EdgeInsets.zero,
                    ),
                    Text(
                      article.title,
                      style: Theme.of(context).textTheme.titleSmall?.copyWith(
                            fontWeight: FontWeight.bold,
                          ),
                      maxLines: 2,
                      overflow: TextOverflow.ellipsis,
                    ),
                    const SizedBox(height: 4),
                    Text(
                      article.author,
                      style: Theme.of(context).textTheme.bodySmall?.copyWith(
                            color: Colors.grey[600],
                          ),
                    ),
                  ],
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }

  Widget _buildExpandedCard(BuildContext context) {
    return Card(
      clipBehavior: Clip.antiAlias,
      child: InkWell(
        onTap: onTap,
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            AspectRatio(
              aspectRatio: 16 / 9,
              child: Image.network(
                article.imageUrl,
                fit: BoxFit.cover,
                errorBuilder: (context, error, stackTrace) => Container(
                  color: Colors.grey[300],
                  child: const Icon(Icons.image, size: 48),
                ),
              ),
            ),
            Padding(
              padding: const EdgeInsets.all(16),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Chip(label: Text(article.category)),
                  const SizedBox(height: 8),
                  Text(
                    article.title,
                    style: Theme.of(context).textTheme.titleMedium?.copyWith(
                          fontWeight: FontWeight.bold,
                        ),
                  ),
                  const SizedBox(height: 8),
                  Text(
                    article.summary,
                    style: Theme.of(context).textTheme.bodyMedium,
                    maxLines: 2,
                    overflow: TextOverflow.ellipsis,
                  ),
                  const SizedBox(height: 8),
                  Text(
                    '${article.author} • ${_formatDate(article.publishedAt)}',
                    style: Theme.of(context).textTheme.bodySmall?.copyWith(
                          color: Colors.grey[600],
                        ),
                  ),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }

  String _formatDate(DateTime date) {
    final now = DateTime.now();
    final diff = now.difference(date);

    if (diff.inHours < 1) return '${diff.inMinutes} นาทีที่แล้ว';
    if (diff.inHours < 24) return '${diff.inHours} ชั่วโมงที่แล้ว';
    return '${diff.inDays} วันที่แล้ว';
  }
}
```

### Article List Screen

```dart
// lib/screens/article_list_screen.dart
import 'package:flutter/material.dart';
import '../models/article.dart';
import '../adaptive/adaptive_card.dart';
import '../adaptive/screen_size.dart';
import 'article_detail_screen.dart';

class ArticleListScreen extends StatelessWidget {
  final String category;
  final List<Article> articles;

  const ArticleListScreen({
    super.key,
    required this.category,
    required this.articles,
  });

  @override
  Widget build(BuildContext context) {
    final screenSize = ScreenSizeHelper.of(context);

    if (screenSize == ScreenSize.compact) {
      return _buildListView(context);
    }

    return _buildGridView(context, screenSize == ScreenSize.expanded ? 2 : 2);
  }

  Widget _buildListView(BuildContext context) {
    return ListView.builder(
      itemCount: articles.length,
      itemBuilder: (context, index) => AdaptiveNewsCard(
        article: articles[index],
        onTap: () => _navigateToDetail(context, articles[index]),
      ),
    );
  }

  Widget _buildGridView(BuildContext context, int crossAxisCount) {
    return GridView.builder(
      padding: const EdgeInsets.all(16),
      gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
        crossAxisCount: crossAxisCount,
        crossAxisSpacing: 16,
        mainAxisSpacing: 16,
        childAspectRatio: 0.75,
      ),
      itemCount: articles.length,
      itemBuilder: (context, index) => AdaptiveNewsCard(
        article: articles[index],
        onTap: () => _navigateToDetail(context, articles[index]),
      ),
    );
  }

  void _navigateToDetail(BuildContext context, Article article) {
    Navigator.push(
      context,
      MaterialPageRoute(
        builder: (context) => ArticleDetailScreen(article: article),
      ),
    );
  }
}
```

### Article Detail Screen (Adaptive)

```dart
// lib/screens/article_detail_screen.dart
import 'package:flutter/material.dart';
import '../models/article.dart';
import '../adaptive/screen_size.dart';

class ArticleDetailScreen extends StatelessWidget {
  final Article article;

  const ArticleDetailScreen({
    super.key,
    required this.article,
  });

  @override
  Widget build(BuildContext context) {
    final screenSize = ScreenSizeHelper.of(context);

    if (screenSize == ScreenSize.expanded) {
      return _buildDesktopLayout(context);
    }

    return _buildMobileLayout(context);
  }

  Widget _buildMobileLayout(BuildContext context) {
    return Scaffold(
      body: CustomScrollView(
        slivers: [
          SliverAppBar(
            expandedHeight: 250,
            pinned: true,
            flexibleSpace: FlexibleSpaceBar(
              title: Text(
                article.category,
                style: const TextStyle(fontSize: 14),
              ),
              background: Image.network(
                article.imageUrl,
                fit: BoxFit.cover,
                errorBuilder: (context, error, stackTrace) => Container(
                  color: Colors.grey[300],
                ),
              ),
            ),
          ),
          SliverPadding(
            padding: const EdgeInsets.all(16),
            sliver: SliverList(
              delegate: SliverChildListDelegate([
                Text(
                  article.title,
                  style: Theme.of(context).textTheme.headlineSmall?.copyWith(
                        fontWeight: FontWeight.bold,
                      ),
                ),
                const SizedBox(height: 8),
                _buildAuthorRow(context),
                const SizedBox(height: 16),
                Text(
                  article.content,
                  style: Theme.of(context).textTheme.bodyLarge,
                ),
              ]),
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildDesktopLayout(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(article.category),
      ),
      body: Center(
        child: ConstrainedBox(
          constraints: const BoxConstraints(maxWidth: 800),
          child: SingleChildScrollView(
            padding: const EdgeInsets.all(32),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                ClipRRect(
                  borderRadius: BorderRadius.circular(12),
                  child: AspectRatio(
                    aspectRatio: 16 / 9,
                    child: Image.network(
                      article.imageUrl,
                      fit: BoxFit.cover,
                    ),
                  ),
                ),
                const SizedBox(height: 24),
                Chip(label: Text(article.category)),
                const SizedBox(height: 12),
                Text(
                  article.title,
                  style: Theme.of(context).textTheme.headlineMedium?.copyWith(
                        fontWeight: FontWeight.bold,
                      ),
                ),
                const SizedBox(height: 12),
                _buildAuthorRow(context),
                const Divider(height: 32),
                Text(
                  article.content,
                  style: Theme.of(context).textTheme.bodyLarge?.copyWith(
                        height: 1.8,
                      ),
                ),
              ],
            ),
          ),
        ),
      ),
    );
  }

  Widget _buildAuthorRow(BuildContext context) {
    return Row(
      children: [
        CircleAvatar(
          radius: 16,
          backgroundColor: Colors.blue[100],
          child: Text(
            article.author[0],
            style: TextStyle(color: Colors.blue[800]),
          ),
        ),
        const SizedBox(width: 8),
        Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              article.author,
              style: Theme.of(context).textTheme.titleSmall,
            ),
            Text(
              _formatDate(article.publishedAt),
              style: Theme.of(context).textTheme.bodySmall?.copyWith(
                    color: Colors.grey[600],
                  ),
            ),
          ],
        ),
      ],
    );
  }

  String _formatDate(DateTime date) {
    return '${date.day}/${date.month}/${date.year}';
  }
}
```

### Main App with Adaptive Navigation

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'adaptive/adaptive_scaffold.dart';
import 'models/article.dart';
import 'screens/article_list_screen.dart';

void main() {
  runApp(const AdaptiveNewsApp());
}

class AdaptiveNewsApp extends StatelessWidget {
  const AdaptiveNewsApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Adaptive News Reader',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepOrange),
        useMaterial3: true,
      ),
      home: const NewsHome(),
    );
  }
}

class NewsHome extends StatelessWidget {
  const NewsHome({super.key});

  @override
  Widget build(BuildContext context) {
    final articles = Article.getMockArticles();

    return AdaptiveScaffold(
      title: 'News Reader',
      destinations: const [
        AdaptiveDestination(icon: Icons.home, label: 'หน้าแรก'),
        AdaptiveDestination(icon: Icons.trending_up, label: 'เทรนดิ้ง'),
        AdaptiveDestination(icon: Icons.bookmark, label: 'บันทึก'),
        AdaptiveDestination(icon: Icons.person, label: 'โปรไฟล์'),
      ],
      pages: [
        ArticleListScreen(category: 'ทั้งหมด', articles: articles),
        ArticleListScreen(category: 'เทรนดิ้ง', articles: articles),
        ArticleListScreen(category: 'บันทึก', articles: []),
        const Center(child: Text('โปรไฟล์')),
      ],
    );
  }
}
```

## สรุป

Adaptive UI ช่วยให้เราสร้างแอปที่:
1. **ดูเป็นธรรมชาติ** บนทุกแพลตฟอร์ม ทั้ง iOS, Android, Web, Desktop
2. **ปรับ layout อัตโนมัติ** ตามขนาดหน้าจอ
3. **ใช้ widget ที่เหมาะสม** กับแต่ละแพลตฟอร์ม
4. **ประหยัดเวลา** ด้วยโค้ดชุดเดียว

## แบบทดสอบ

1. อธิบายความแตกต่างระหว่าง `Material` และ `Cupertino` design
2. สร้าง `AdaptiveProgressIndicator` ที่แสดง `CircularProgressIndicator` บน Android และ `CupertinoActivityIndicator` บน iOS
3. ทำไม `LayoutBuilder` จึงเหมาะกว่า `MediaQuery` สำหรับ responsive layout?
4. เพิ่ม `AdaptiveTabBar` ใน Workshop ที่ใช้ `TabBar` บน Android และ `CupertinoSlidingSegmentedControl` บน iOS
