# Part 67: Responsive Design ใน Flutter

## Responsive Design คืออะไร?

Responsive Design คือการออกแบบ UI ที่ปรับตัวตามขนาดหน้าจอต่างๆ:
- Phone (360-480dp)
- Tablet (600-900dp)
- Desktop (1200dp+)

---

## 1. LayoutBuilder

### LayoutBuilder พื้นฐาน

```dart
// LayoutBuilder ให้ข้อมูล constraints ของ parent
class ResponsiveContainer extends StatelessWidget {
  final Widget child;
  
  const ResponsiveContainer({required this.child});

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        final maxWidth = constraints.maxWidth;
        
        if (maxWidth < 600) {
          // Phone layout
          return _PhoneLayout(child: child);
        } else if (maxWidth < 900) {
          // Tablet layout
          return _TabletLayout(child: child);
        } else {
          // Desktop layout
          return _DesktopLayout(child: child);
        }
      },
    );
  }
}

// ตัวอย่างการใช้ LayoutBuilder สำหรับ Grid
class ResponsiveGrid extends StatelessWidget {
  final List<Widget> children;

  const ResponsiveGrid({required this.children});

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        int crossAxisCount;
        double childAspectRatio;

        if (constraints.maxWidth < 600) {
          crossAxisCount = 2;
          childAspectRatio = 0.75;
        } else if (constraints.maxWidth < 900) {
          crossAxisCount = 3;
          childAspectRatio = 0.8;
        } else {
          crossAxisCount = 4;
          childAspectRatio = 0.85;
        }

        return GridView.count(
          crossAxisCount: crossAxisCount,
          childAspectRatio: childAspectRatio,
          crossAxisSpacing: 16,
          mainAxisSpacing: 16,
          children: children,
        );
      },
    );
  }
}
```

### Adaptive Layout

```dart
// Widget ที่แสดงต่างกันตามขนาดหน้าจอ
class AdaptiveScaffold extends StatelessWidget {
  final String title;
  final List<NavigationItem> navigationItems;
  final Widget body;

  const AdaptiveScaffold({
    required this.title,
    required this.navigationItems,
    required this.body,
  });

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        if (constraints.maxWidth >= 900) {
          // Desktop: Side navigation
          return _DesktopScaffold(
            title: title,
            navigationItems: navigationItems,
            body: body,
          );
        } else if (constraints.maxWidth >= 600) {
          // Tablet: Rail navigation
          return _TabletScaffold(
            title: title,
            navigationItems: navigationItems,
            body: body,
          );
        } else {
          // Phone: Bottom navigation
          return _PhoneScaffold(
            title: title,
            navigationItems: navigationItems,
            body: body,
          );
        }
      },
    );
  }
}

class _PhoneScaffold extends StatefulWidget {
  final String title;
  final List<NavigationItem> navigationItems;
  final Widget body;

  const _PhoneScaffold({
    required this.title,
    required this.navigationItems,
    required this.body,
  });

  @override
  _PhoneScaffoldState createState() => _PhoneScaffoldState();
}

class _PhoneScaffoldState extends State<_PhoneScaffold> {
  int _currentIndex = 0;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text(widget.title)),
      body: widget.navigationItems[_currentIndex].page,
      bottomNavigationBar: NavigationBar(
        selectedIndex: _currentIndex,
        onDestinationSelected: (index) =>
            setState(() => _currentIndex = index),
        destinations: widget.navigationItems
            .map((item) => NavigationDestination(
                  icon: item.icon,
                  label: item.label,
                ))
            .toList(),
      ),
    );
  }
}

class _TabletScaffold extends StatefulWidget {
  final String title;
  final List<NavigationItem> navigationItems;
  final Widget body;

  const _TabletScaffold({
    required this.title,
    required this.navigationItems,
    required this.body,
  });

  @override
  _TabletScaffoldState createState() => _TabletScaffoldState();
}

class _TabletScaffoldState extends State<_TabletScaffold> {
  int _currentIndex = 0;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text(widget.title)),
      body: Row(
        children: [
          NavigationRail(
            selectedIndex: _currentIndex,
            onDestinationSelected: (index) =>
                setState(() => _currentIndex = index),
            labelType: NavigationRailLabelType.selected,
            destinations: widget.navigationItems
                .map((item) => NavigationRailDestination(
                      icon: item.icon,
                      label: Text(item.label),
                    ))
                .toList(),
          ),
          VerticalDivider(width: 1),
          Expanded(
            child: widget.navigationItems[_currentIndex].page,
          ),
        ],
      ),
    );
  }
}

class _DesktopScaffold extends StatefulWidget {
  final String title;
  final List<NavigationItem> navigationItems;
  final Widget body;

  const _DesktopScaffold({
    required this.title,
    required this.navigationItems,
    required this.body,
  });

  @override
  _DesktopScaffoldState createState() => _DesktopScaffoldState();
}

class _DesktopScaffoldState extends State<_DesktopScaffold> {
  int _currentIndex = 0;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Row(
        children: [
          NavigationDrawer(
            selectedIndex: _currentIndex,
            onDestinationSelected: (index) =>
                setState(() => _currentIndex = index),
            children: [
              Padding(
                padding: EdgeInsets.all(16),
                child: Text(
                  widget.title,
                  style: TextStyle(
                    fontSize: 20,
                    fontWeight: FontWeight.bold,
                  ),
                ),
              ),
              Divider(),
              ...widget.navigationItems.map(
                (item) => NavigationDrawerDestination(
                  icon: item.icon,
                  label: Text(item.label),
                ),
              ),
            ],
          ),
          VerticalDivider(width: 1),
          Expanded(
            child: widget.navigationItems[_currentIndex].page,
          ),
        ],
      ),
    );
  }
}
```

---

## 2. MediaQuery

### การใช้ MediaQuery

```dart
// MediaQuery ให้ข้อมูลเกี่ยวกับหน้าจอ
class MediaQueryExample extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final mediaQuery = MediaQuery.of(context);
    final screenWidth = mediaQuery.size.width;
    final screenHeight = mediaQuery.size.height;
    final devicePixelRatio = mediaQuery.devicePixelRatio;
    final orientation = mediaQuery.orientation;
    final textScaleFactor = mediaQuery.textScaler;
    final padding = mediaQuery.padding; // safe area
    final viewInsets = mediaQuery.viewInsets; // keyboard

    return Scaffold(
      body: Padding(
        padding: EdgeInsets.only(
          // หลีกเลี่ยง notch และ status bar
          top: mediaQuery.padding.top,
          bottom: mediaQuery.padding.bottom,
        ),
        child: Column(
          children: [
            Text('Screen size: ${screenWidth.toInt()}x${screenHeight.toInt()}'),
            Text('Pixel ratio: $devicePixelRatio'),
            Text('Orientation: $orientation'),
          ],
        ),
      ),
    );
  }
}

// Responsive padding
class ResponsivePadding extends StatelessWidget {
  final Widget child;

  const ResponsivePadding({required this.child});

  @override
  Widget build(BuildContext context) {
    final width = MediaQuery.of(context).size.width;
    
    double horizontal;
    if (width < 600) {
      horizontal = 16;
    } else if (width < 900) {
      horizontal = 32;
    } else {
      horizontal = (width - 900) / 2 + 32; // max width 900 + centered
    }

    return Padding(
      padding: EdgeInsets.symmetric(horizontal: horizontal),
      child: child,
    );
  }
}
```

---

## 3. Breakpoints

### Breakpoint System

```dart
// breakpoints.dart
class Breakpoints {
  static const double phone = 600;
  static const double tablet = 900;
  static const double desktop = 1200;
  static const double largeDesktop = 1600;

  static ScreenSize of(BuildContext context) {
    final width = MediaQuery.of(context).size.width;
    return fromWidth(width);
  }

  static ScreenSize fromWidth(double width) {
    if (width < phone) return ScreenSize.phone;
    if (width < tablet) return ScreenSize.tablet;
    if (width < desktop) return ScreenSize.desktop;
    return ScreenSize.largeDesktop;
  }

  static bool isPhone(BuildContext context) => of(context) == ScreenSize.phone;
  static bool isTablet(BuildContext context) => of(context) == ScreenSize.tablet;
  static bool isDesktop(BuildContext context) =>
      of(context) == ScreenSize.desktop ||
      of(context) == ScreenSize.largeDesktop;
  static bool isWideScreen(BuildContext context) =>
      of(context) != ScreenSize.phone;
}

enum ScreenSize { phone, tablet, desktop, largeDesktop }

// Responsive Widget Helper
class Responsive extends StatelessWidget {
  final Widget? phone;
  final Widget? tablet;
  final Widget? desktop;

  const Responsive({
    this.phone,
    this.tablet,
    this.desktop,
  }) : assert(
          phone != null || tablet != null || desktop != null,
          'At least one layout must be provided',
        );

  @override
  Widget build(BuildContext context) {
    final size = Breakpoints.of(context);

    switch (size) {
      case ScreenSize.phone:
        return phone ?? tablet ?? desktop!;
      case ScreenSize.tablet:
        return tablet ?? desktop ?? phone!;
      case ScreenSize.desktop:
      case ScreenSize.largeDesktop:
        return desktop ?? tablet ?? phone!;
    }
  }
}

// ตัวอย่างการใช้งาน
class ProductsPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Responsive(
      phone: _PhoneProductList(),
      tablet: _TabletProductGrid(),
      desktop: _DesktopProductLayout(),
    );
  }
}
```

---

## 4. Adaptive Widgets

### Text ที่ปรับขนาดตามหน้าจอ

```dart
// adaptive_text.dart
class AdaptiveText extends StatelessWidget {
  final String text;
  final TextStyle? baseStyle;
  final Map<ScreenSize, double>? fontSizes;

  const AdaptiveText(
    this.text, {
    this.baseStyle,
    this.fontSizes,
  });

  @override
  Widget build(BuildContext context) {
    final size = Breakpoints.of(context);
    final defaultSizes = {
      ScreenSize.phone: 14.0,
      ScreenSize.tablet: 16.0,
      ScreenSize.desktop: 18.0,
      ScreenSize.largeDesktop: 20.0,
    };

    final fontSize = fontSizes?[size] ?? defaultSizes[size]!;

    return Text(
      text,
      style: baseStyle?.copyWith(fontSize: fontSize) ??
          TextStyle(fontSize: fontSize),
    );
  }
}

// Adaptive Icon
class AdaptiveIcon extends StatelessWidget {
  final IconData icon;
  final Map<ScreenSize, double>? sizes;
  final Color? color;

  const AdaptiveIcon(
    this.icon, {
    this.sizes,
    this.color,
  });

  @override
  Widget build(BuildContext context) {
    final size = Breakpoints.of(context);
    final defaultSizes = {
      ScreenSize.phone: 24.0,
      ScreenSize.tablet: 28.0,
      ScreenSize.desktop: 32.0,
      ScreenSize.largeDesktop: 36.0,
    };

    return Icon(
      icon,
      size: sizes?[size] ?? defaultSizes[size]!,
      color: color,
    );
  }
}
```

### Adaptive Spacing

```dart
// adaptive_spacing.dart
class AdaptiveSpacing {
  static double xs(BuildContext context) {
    return Breakpoints.isPhone(context) ? 4 : 8;
  }

  static double sm(BuildContext context) {
    return Breakpoints.isPhone(context) ? 8 : 12;
  }

  static double md(BuildContext context) {
    return Breakpoints.isPhone(context) ? 16 : 24;
  }

  static double lg(BuildContext context) {
    return Breakpoints.isPhone(context) ? 24 : 32;
  }

  static double xl(BuildContext context) {
    return Breakpoints.isPhone(context) ? 32 : 48;
  }
}

// ใช้งาน
class SpacingExample extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        SizedBox(height: AdaptiveSpacing.md(context)),
        Padding(
          padding: EdgeInsets.symmetric(
            horizontal: AdaptiveSpacing.lg(context),
          ),
          child: Text('Content'),
        ),
      ],
    );
  }
}
```

---

## Workshop: Responsive Dashboard

```dart
// Dashboard ที่ปรับตัวตามขนาดหน้าจอ

// dashboard_screen.dart
class DashboardScreen extends StatefulWidget {
  @override
  _DashboardScreenState createState() => _DashboardScreenState();
}

class _DashboardScreenState extends State<DashboardScreen> {
  int _selectedNavIndex = 0;

  final List<DashboardSection> _sections = [
    DashboardSection(title: 'ภาพรวม', icon: Icons.dashboard),
    DashboardSection(title: 'รายงาน', icon: Icons.bar_chart),
    DashboardSection(title: 'ผู้ใช้', icon: Icons.people),
    DashboardSection(title: 'การตั้งค่า', icon: Icons.settings),
  ];

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        if (constraints.maxWidth >= 900) {
          return _buildDesktopLayout();
        } else {
          return _buildMobileLayout();
        }
      },
    );
  }

  Widget _buildDesktopLayout() {
    return Scaffold(
      body: Row(
        children: [
          // Sidebar
          Container(
            width: 250,
            color: Colors.grey[900],
            child: Column(
              children: [
                Container(
                  padding: EdgeInsets.all(24),
                  child: Row(
                    children: [
                      FlutterLogo(size: 32),
                      SizedBox(width: 12),
                      Text(
                        'Dashboard',
                        style: TextStyle(
                          color: Colors.white,
                          fontSize: 20,
                          fontWeight: FontWeight.bold,
                        ),
                      ),
                    ],
                  ),
                ),
                Divider(color: Colors.grey[700]),
                ...List.generate(_sections.length, (index) {
                  final section = _sections[index];
                  final isSelected = _selectedNavIndex == index;
                  return ListTile(
                    leading: Icon(
                      section.icon,
                      color: isSelected ? Colors.blue : Colors.grey[400],
                    ),
                    title: Text(
                      section.title,
                      style: TextStyle(
                        color: isSelected ? Colors.white : Colors.grey[400],
                      ),
                    ),
                    selected: isSelected,
                    selectedTileColor: Colors.blue.withOpacity(0.2),
                    onTap: () =>
                        setState(() => _selectedNavIndex = index),
                  );
                }),
              ],
            ),
          ),

          // Main content
          Expanded(
            child: _buildMainContent(),
          ),
        ],
      ),
    );
  }

  Widget _buildMobileLayout() {
    return Scaffold(
      appBar: AppBar(
        title: Text(_sections[_selectedNavIndex].title),
        backgroundColor: Colors.grey[900],
        foregroundColor: Colors.white,
      ),
      drawer: Drawer(
        backgroundColor: Colors.grey[900],
        child: Column(
          children: [
            DrawerHeader(
              decoration: BoxDecoration(color: Colors.grey[800]),
              child: Row(
                children: [
                  FlutterLogo(size: 40),
                  SizedBox(width: 12),
                  Text(
                    'Dashboard',
                    style: TextStyle(
                      color: Colors.white,
                      fontSize: 20,
                    ),
                  ),
                ],
              ),
            ),
            ...List.generate(_sections.length, (index) {
              final section = _sections[index];
              final isSelected = _selectedNavIndex == index;
              return ListTile(
                leading: Icon(
                  section.icon,
                  color: isSelected ? Colors.blue : Colors.grey[400],
                ),
                title: Text(
                  section.title,
                  style: TextStyle(
                    color: isSelected ? Colors.white : Colors.grey[400],
                  ),
                ),
                selected: isSelected,
                onTap: () {
                  setState(() => _selectedNavIndex = index);
                  Navigator.pop(context);
                },
              );
            }),
          ],
        ),
      ),
      body: _buildMainContent(),
    );
  }

  Widget _buildMainContent() {
    return SingleChildScrollView(
      padding: EdgeInsets.all(24),
      child: LayoutBuilder(
        builder: (context, constraints) {
          final isWide = constraints.maxWidth >= 600;

          return Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Text(
                _sections[_selectedNavIndex].title,
                style: TextStyle(
                  fontSize: 28,
                  fontWeight: FontWeight.bold,
                ),
              ),
              SizedBox(height: 24),

              // Stats cards
              _buildStatsSection(isWide),

              SizedBox(height: 24),

              // Charts section
              _buildChartsSection(isWide),

              SizedBox(height: 24),

              // Table section
              _buildTableSection(),
            ],
          );
        },
      ),
    );
  }

  Widget _buildStatsSection(bool isWide) {
    final stats = [
      StatCard(title: 'ผู้ใช้ทั้งหมด', value: '12,345', icon: Icons.people, color: Colors.blue),
      StatCard(title: 'รายได้วันนี้', value: '฿45,678', icon: Icons.payments, color: Colors.green),
      StatCard(title: 'คำสั่งซื้อ', value: '234', icon: Icons.shopping_cart, color: Colors.orange),
      StatCard(title: 'อัตราแปลง', value: '3.2%', icon: Icons.trending_up, color: Colors.purple),
    ];

    if (isWide) {
      return GridView.count(
        crossAxisCount: 4,
        shrinkWrap: true,
        physics: NeverScrollableScrollPhysics(),
        crossAxisSpacing: 16,
        mainAxisSpacing: 16,
        childAspectRatio: 1.5,
        children: stats.map((s) => _StatCardWidget(stat: s)).toList(),
      );
    } else {
      return GridView.count(
        crossAxisCount: 2,
        shrinkWrap: true,
        physics: NeverScrollableScrollPhysics(),
        crossAxisSpacing: 12,
        mainAxisSpacing: 12,
        childAspectRatio: 1.5,
        children: stats.map((s) => _StatCardWidget(stat: s)).toList(),
      );
    }
  }

  Widget _buildChartsSection(bool isWide) {
    if (isWide) {
      return Row(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Expanded(
            flex: 2,
            child: _ChartCard(title: 'รายได้รายเดือน'),
          ),
          SizedBox(width: 16),
          Expanded(
            flex: 1,
            child: _ChartCard(title: 'สัดส่วนหมวดหมู่'),
          ),
        ],
      );
    } else {
      return Column(
        children: [
          _ChartCard(title: 'รายได้รายเดือน'),
          SizedBox(height: 16),
          _ChartCard(title: 'สัดส่วนหมวดหมู่'),
        ],
      );
    }
  }

  Widget _buildTableSection() {
    return Card(
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Padding(
            padding: EdgeInsets.all(16),
            child: Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                Text(
                  'คำสั่งซื้อล่าสุด',
                  style: TextStyle(
                    fontSize: 18,
                    fontWeight: FontWeight.bold,
                  ),
                ),
                TextButton(
                  onPressed: () {},
                  child: Text('ดูทั้งหมด'),
                ),
              ],
            ),
          ),
          SingleChildScrollView(
            scrollDirection: Axis.horizontal,
            child: DataTable(
              columns: [
                DataColumn(label: Text('ID')),
                DataColumn(label: Text('ลูกค้า')),
                DataColumn(label: Text('สินค้า')),
                DataColumn(label: Text('ยอดรวม')),
                DataColumn(label: Text('สถานะ')),
              ],
              rows: List.generate(
                5,
                (index) => DataRow(
                  cells: [
                    DataCell(Text('#${1000 + index}')),
                    DataCell(Text('ลูกค้า ${index + 1}')),
                    DataCell(Text('สินค้า ${index + 1}')),
                    DataCell(Text('฿${(index + 1) * 1000}')),
                    DataCell(
                      Chip(
                        label: Text('สำเร็จ'),
                        backgroundColor: Colors.green.shade100,
                      ),
                    ),
                  ],
                ),
              ),
            ),
          ),
        ],
      ),
    );
  }
}

class _StatCardWidget extends StatelessWidget {
  final StatCard stat;

  const _StatCardWidget({required this.stat});

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Padding(
        padding: EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          mainAxisAlignment: MainAxisAlignment.spaceBetween,
          children: [
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                Text(stat.title, style: TextStyle(color: Colors.grey[600])),
                Icon(stat.icon, color: stat.color),
              ],
            ),
            Text(
              stat.value,
              style: TextStyle(
                fontSize: 24,
                fontWeight: FontWeight.bold,
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class _ChartCard extends StatelessWidget {
  final String title;

  const _ChartCard({required this.title});

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Padding(
        padding: EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(title,
                style: TextStyle(
                    fontSize: 16, fontWeight: FontWeight.bold)),
            SizedBox(height: 16),
            Container(
              height: 200,
              color: Colors.grey[100],
              child: Center(
                child: Text('Chart Placeholder',
                    style: TextStyle(color: Colors.grey)),
              ),
            ),
          ],
        ),
      ),
    );
  }
}

// Models
class StatCard {
  final String title;
  final String value;
  final IconData icon;
  final Color color;

  StatCard({
    required this.title,
    required this.value,
    required this.icon,
    required this.color,
  });
}

class DashboardSection {
  final String title;
  final IconData icon;

  DashboardSection({required this.title, required this.icon});
}

class NavigationItem {
  final String label;
  final Icon icon;
  final Widget page;

  NavigationItem({
    required this.label,
    required this.icon,
    required this.page,
  });
}
```

---

## สรุป

Responsive Design ใน Flutter ประกอบด้วย:

1. **LayoutBuilder** - ปรับ layout ตาม constraints
2. **MediaQuery** - ข้อมูลหน้าจอและ safe area
3. **Breakpoints** - จุดหักเหสำหรับ layouts ต่างๆ
4. **Adaptive Widgets** - widgets ที่ปรับตาม screen size
5. **Navigation Patterns** - Bottom Nav / Rail / Drawer
6. **Responsive Grid** - ปรับจำนวน columns อัตโนมัติ
