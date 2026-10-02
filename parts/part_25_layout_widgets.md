# Part 25: Layout Widgets

## บทนำ

Layout Widgets ใช้จัดเรียง widget ต่างๆ บนหน้าจอ Flutter มี layout widgets ที่หลากหลาย ตั้งแต่ Row/Column ง่ายๆ ไปจนถึง Stack และ CustomScrollView

---

## Row และ Column

Row จัดเรียงแนวนอน, Column จัดเรียงแนวตั้ง

### MainAxisAlignment

```dart
// MainAxis คือแกนหลักของ widget
// Row: แกนหลัก = แนวนอน
// Column: แกนหลัก = แนวตั้ง

Row(
  mainAxisAlignment: MainAxisAlignment.start,       // เริ่มจากซ้าย (default)
  children: [...],
)

Row(
  mainAxisAlignment: MainAxisAlignment.end,         // ชิดขวา
  children: [...],
)

Row(
  mainAxisAlignment: MainAxisAlignment.center,      // ตรงกลาง
  children: [...],
)

Row(
  mainAxisAlignment: MainAxisAlignment.spaceBetween, // ช่องว่างระหว่าง
  children: [...],
)

Row(
  mainAxisAlignment: MainAxisAlignment.spaceAround,  // ช่องว่างรอบ
  children: [...],
)

Row(
  mainAxisAlignment: MainAxisAlignment.spaceEvenly,  // ช่องว่างเท่ากัน
  children: [...],
)
```

### CrossAxisAlignment

```dart
// CrossAxis คือแกนตั้งฉากกับแกนหลัก
// Row: แกนตั้งฉาก = แนวตั้ง
// Column: แกนตั้งฉาก = แนวนอน

Row(
  crossAxisAlignment: CrossAxisAlignment.start,    // บนสุด
  crossAxisAlignment: CrossAxisAlignment.end,      // ล่างสุด
  crossAxisAlignment: CrossAxisAlignment.center,   // กลาง (default)
  crossAxisAlignment: CrossAxisAlignment.stretch,  // ยืดเต็มความสูง
  crossAxisAlignment: CrossAxisAlignment.baseline, // ตาม text baseline
  children: [...]
)

Column(
  crossAxisAlignment: CrossAxisAlignment.start,    // ชิดซ้าย
  crossAxisAlignment: CrossAxisAlignment.end,      // ชิดขวา
  crossAxisAlignment: CrossAxisAlignment.center,   // กลาง (default)
  crossAxisAlignment: CrossAxisAlignment.stretch,  // ยืดเต็มความกว้าง
  children: [...]
)
```

### MainAxisSize

```dart
// ขนาดของ Row/Column ตามแกนหลัก
Row(
  mainAxisSize: MainAxisSize.max,  // เต็ม parent width (default)
  children: [...],
)

Row(
  mainAxisSize: MainAxisSize.min,  // แค่พอดีกับ children
  children: [...],
)
```

### ตัวอย่างการใช้งาน

```dart
// Navigation bar ที่ bottom
class BottomNavBar extends StatelessWidget {
  const BottomNavBar({super.key});

  @override
  Widget build(BuildContext context) {
    return Container(
      height: 60,
      color: Colors.white,
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceEvenly,
        children: [
          _NavItem(icon: Icons.home, label: 'หน้าหลัก', isActive: true),
          _NavItem(icon: Icons.search, label: 'ค้นหา'),
          _NavItem(icon: Icons.shopping_bag, label: 'คำสั่งซื้อ'),
          _NavItem(icon: Icons.person, label: 'โปรไฟล์'),
        ],
      ),
    );
  }
}

class _NavItem extends StatelessWidget {
  final IconData icon;
  final String label;
  final bool isActive;

  const _NavItem({
    required this.icon,
    required this.label,
    this.isActive = false,
  });

  @override
  Widget build(BuildContext context) {
    final color = isActive ? Theme.of(context).colorScheme.primary : Colors.grey;
    return Column(
      mainAxisAlignment: MainAxisAlignment.center,
      children: [
        Icon(icon, color: color, size: 24),
        const SizedBox(height: 2),
        Text(label, style: TextStyle(color: color, fontSize: 11)),
      ],
    );
  }
}
```

---

## Expanded และ Flexible

ใช้กับ Row/Column เพื่อให้ widget ขยายตาม flex ratio

### Expanded

```dart
// Expanded: ขยายเต็ม space ที่เหลือ
Row(
  children: [
    // Icon ขนาดคงที่
    const Icon(Icons.person),
    const SizedBox(width: 8),
    
    // Text ขยายเต็มพื้นที่ที่เหลือ
    Expanded(
      child: Text(
        'ชื่อผู้ใช้ที่ยาวมากๆ',
        overflow: TextOverflow.ellipsis,
      ),
    ),
    
    // Button ขนาดคงที่
    const Text('100 pts'),
  ],
)

// Expanded กับ flex ratio
Row(
  children: [
    Expanded(
      flex: 2,    // ได้ 2/3 ของ space
      child: Container(color: Colors.red, height: 50),
    ),
    Expanded(
      flex: 1,    // ได้ 1/3 ของ space
      child: Container(color: Colors.blue, height: 50),
    ),
  ],
)
```

### Flexible

```dart
// Flexible: ขยายได้ แต่ไม่บังคับ
Row(
  children: [
    Flexible(
      child: Text('ข้อความยาว'),  // ยืดได้แต่ไม่ต้องเต็ม
    ),
    const Icon(Icons.arrow_forward),
  ],
)

// Flexible.fit
Flexible(
  fit: FlexFit.tight,   // เหมือน Expanded (ขยายเต็ม)
  child: ...,
)

Flexible(
  fit: FlexFit.loose,   // ขยายได้แต่ไม่เกินขนาด child (default)
  child: ...,
)
```

---

## Stack และ Positioned

`Stack` วาง widget ซ้อนกัน ใช้สำหรับ overlay และ absolute positioning

```dart
// Stack พื้นฐาน
Stack(
  children: [
    // Layer 1 (ล่างสุด)
    Container(
      width: 200,
      height: 200,
      color: Colors.blue,
    ),
    
    // Layer 2 (กลาง)
    Container(
      width: 150,
      height: 150,
      color: Colors.green.withOpacity(0.7),
    ),
    
    // Layer 3 (บนสุด)
    const Text('On top'),
  ],
)
```

### Stack alignment

```dart
Stack(
  alignment: Alignment.center,      // จัดกลาง (default: topLeft)
  alignment: Alignment.topRight,    // มุมขวาบน
  alignment: Alignment.bottomCenter, // กลางล่าง
  children: [...],
)
```

### Positioned

```dart
// กำหนดตำแหน่งแบบ absolute
Stack(
  children: [
    // Background
    Image.network('https://picsum.photos/400/300', fit: BoxFit.cover),
    
    // Positioned widgets
    const Positioned(
      top: 16,
      left: 16,
      child: Text('Top Left', style: TextStyle(color: Colors.white)),
    ),
    
    const Positioned(
      bottom: 0,
      left: 0,
      right: 0,
      child: _GradientOverlay(),
    ),
    
    Positioned(
      bottom: 16,
      left: 16,
      right: 80,
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: const [
          Text('Product Name',
            style: TextStyle(color: Colors.white, fontSize: 18, fontWeight: FontWeight.bold)),
          Text('฿999',
            style: TextStyle(color: Colors.white70, fontSize: 14)),
        ],
      ),
    ),
    
    const Positioned(
      bottom: 16,
      right: 16,
      child: CircleAvatar(
        backgroundColor: Colors.white,
        child: Icon(Icons.favorite_border, color: Colors.red),
      ),
    ),
  ],
)

// Positioned.fill - เต็ม Stack
Positioned.fill(
  child: Container(
    color: Colors.black.withOpacity(0.5),
  ),
)
```

### ตัวอย่าง Stack ใน Profile Card

```dart
class ProfileCard extends StatelessWidget {
  const ProfileCard({super.key});

  @override
  Widget build(BuildContext context) {
    return SizedBox(
      height: 220,
      child: Stack(
        clipBehavior: Clip.none,
        children: [
          // Background card
          Container(
            margin: const EdgeInsets.only(top: 50),
            padding: const EdgeInsets.fromLTRB(16, 60, 16, 16),
            decoration: BoxDecoration(
              color: Colors.white,
              borderRadius: BorderRadius.circular(16),
              boxShadow: [
                BoxShadow(color: Colors.black.withOpacity(0.1),
                  blurRadius: 10, offset: const Offset(0, 4)),
              ],
            ),
            child: Column(
              children: [
                const Text('John Doe',
                  style: TextStyle(fontSize: 20, fontWeight: FontWeight.bold)),
                const SizedBox(height: 4),
                Text('Flutter Developer',
                  style: TextStyle(color: Colors.grey[600])),
                const SizedBox(height: 16),
                Row(
                  mainAxisAlignment: MainAxisAlignment.spaceEvenly,
                  children: [
                    _StatColumn(label: 'โพสต์', value: '234'),
                    _StatColumn(label: 'ผู้ติดตาม', value: '1.2k'),
                    _StatColumn(label: 'ติดตาม', value: '456'),
                  ],
                ),
              ],
            ),
          ),
          
          // Avatar (positioned ออกนอก card)
          Positioned(
            top: 0,
            left: 0,
            right: 0,
            child: Center(
              child: CircleAvatar(
                radius: 50,
                backgroundImage: const NetworkImage(
                  'https://picsum.photos/seed/avatar/100/100'),
                child: Positioned.fill(
                  child: Align(
                    alignment: Alignment.bottomRight,
                    child: Container(
                      width: 24, height: 24,
                      decoration: const BoxDecoration(
                        color: Colors.green,
                        shape: BoxShape.circle,
                        border: Border.fromBorderSide(
                          BorderSide(color: Colors.white, width: 2)),
                      ),
                    ),
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

class _StatColumn extends StatelessWidget {
  final String label;
  final String value;
  
  const _StatColumn({required this.label, required this.value});

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text(value, style: const TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
        Text(label, style: TextStyle(color: Colors.grey[600], fontSize: 12)),
      ],
    );
  }
}
```

---

## Wrap

`Wrap` คล้าย Row/Column แต่ขึ้นบรรทัดใหม่เมื่อ overflow

```dart
// Wrap ที่ขึ้นบรรทัดใหม่
Wrap(
  spacing: 8,           // ระยะห่างแนวนอน
  runSpacing: 8,        // ระยะห่างแนวตั้ง (ระหว่างบรรทัด)
  alignment: WrapAlignment.start,
  children: [
    Chip(label: Text('Flutter')),
    Chip(label: Text('Dart')),
    Chip(label: Text('Firebase')),
    Chip(label: Text('Riverpod')),
    Chip(label: Text('Clean Architecture')),
    Chip(label: Text('Testing')),
  ],
)

// Wrap แนวตั้ง
Wrap(
  direction: Axis.vertical,
  spacing: 8,
  runSpacing: 8,
  children: [...],
)
```

### ตัวอย่าง Tag/Chip selector

```dart
class TagSelector extends StatefulWidget {
  final List<String> tags;
  
  const TagSelector({super.key, required this.tags});

  @override
  State<TagSelector> createState() => _TagSelectorState();
}

class _TagSelectorState extends State<TagSelector> {
  final Set<String> _selectedTags = {};

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        const Text('เลือกหมวดหมู่ที่สนใจ:',
          style: TextStyle(fontWeight: FontWeight.bold)),
        const SizedBox(height: 8),
        Wrap(
          spacing: 8,
          runSpacing: 8,
          children: widget.tags.map((tag) {
            final isSelected = _selectedTags.contains(tag);
            return FilterChip(
              label: Text(tag),
              selected: isSelected,
              onSelected: (selected) {
                setState(() {
                  if (selected) {
                    _selectedTags.add(tag);
                  } else {
                    _selectedTags.remove(tag);
                  }
                });
              },
            );
          }).toList(),
        ),
        if (_selectedTags.isNotEmpty) ...[
          const SizedBox(height: 8),
          Text('เลือก: ${_selectedTags.join(', ')}'),
        ],
      ],
    );
  }
}
```

---

## Center และ Align

```dart
// Center: จัดกลางทั้งแนวตั้งและแนวนอน
const Center(
  child: Text('Centered'),
)

// Align: จัดตำแหน่งแบบกำหนดเอง
Align(
  alignment: Alignment.topLeft,
  child: const Text('Top Left'),
)

Align(
  alignment: Alignment.bottomRight,
  child: const Text('Bottom Right'),
)

// FractionalOffset - ใช้ 0.0-1.0
Align(
  alignment: const Alignment(0.5, -0.8),  // x=0.5, y=-0.8
  child: const Text('Custom position'),
)
```

---

## AspectRatio

`AspectRatio` กำหนดสัดส่วนความกว้าง:ความสูงของ widget

```dart
// 16:9 video container
AspectRatio(
  aspectRatio: 16 / 9,
  child: Container(
    color: Colors.black,
    child: const Icon(Icons.play_circle, size: 64, color: Colors.white),
  ),
)

// Square image
AspectRatio(
  aspectRatio: 1,  // 1:1
  child: Image.network('url', fit: BoxFit.cover),
)

// Portrait
AspectRatio(
  aspectRatio: 3 / 4,
  child: Image.network('url', fit: BoxFit.cover),
)
```

---

## ConstrainedBox และ UnconstrainedBox

```dart
// กำหนด constraints
ConstrainedBox(
  constraints: const BoxConstraints(
    minWidth: 100,
    maxWidth: 300,
    minHeight: 50,
    maxHeight: 150,
  ),
  child: Container(color: Colors.blue),
)

// BoxConstraints shortcuts
BoxConstraints.tight(const Size(200, 100))   // ขนาดตาย
BoxConstraints.expand()                       // เต็ม parent
BoxConstraints.loose(const Size(300, 200))    // max size

// UnconstrainedBox - ไม่มี constraint จาก parent
UnconstrainedBox(
  child: Container(
    width: 500,  // อาจ overflow ออกนอกหน้าจอ
    height: 100,
    color: Colors.red,
  ),
)
```

---

## IntrinsicHeight และ IntrinsicWidth

```dart
// ทำให้ children ใน Row มีความสูงเท่ากัน
IntrinsicHeight(
  child: Row(
    crossAxisAlignment: CrossAxisAlignment.stretch,
    children: [
      Container(
        color: Colors.blue,
        padding: const EdgeInsets.all(16),
        child: const Text('Short'),
      ),
      const VerticalDivider(),
      Container(
        color: Colors.green,
        padding: const EdgeInsets.all(16),
        child: const Text('Much longer\ncontent here\nand more'),
      ),
    ],
  ),
)
```

---

## Workshop: Responsive Profile Page

สร้างหน้า Profile ที่ responsive ทั้ง portrait และ landscape

### lib/screens/profile_screen.dart

```dart
import 'package:flutter/material.dart';

class ProfileData {
  final String name;
  final String username;
  final String bio;
  final String avatarUrl;
  final int posts;
  final int followers;
  final int following;
  final List<String> skills;
  final List<Post> recentPosts;

  const ProfileData({
    required this.name,
    required this.username,
    required this.bio,
    required this.avatarUrl,
    required this.posts,
    required this.followers,
    required this.following,
    required this.skills,
    required this.recentPosts,
  });
}

class Post {
  final String id;
  final String imageUrl;
  final String title;
  final int likes;

  const Post({
    required this.id,
    required this.imageUrl,
    required this.title,
    required this.likes,
  });
}

class ProfileScreen extends StatefulWidget {
  const ProfileScreen({super.key});

  @override
  State<ProfileScreen> createState() => _ProfileScreenState();
}

class _ProfileScreenState extends State<ProfileScreen>
    with SingleTickerProviderStateMixin {
  late TabController _tabController;
  bool _isFollowing = false;

  final ProfileData _profile = const ProfileData(
    name: 'อารยา นักพัฒนา',
    username: '@araya_dev',
    bio: 'Flutter Developer | Mobile & Web | ชอบสร้างสิ่งที่สวยงามและใช้งานได้จริง 🚀',
    avatarUrl: 'https://picsum.photos/seed/profile/200/200',
    posts: 128,
    followers: 2847,
    following: 341,
    skills: ['Flutter', 'Dart', 'Firebase', 'Riverpod', 'Clean Architecture', 'UI/UX'],
    recentPosts: [
      Post(id: '1', imageUrl: 'https://picsum.photos/seed/p1/400/400', title: 'Flutter Animation', likes: 234),
      Post(id: '2', imageUrl: 'https://picsum.photos/seed/p2/400/400', title: 'State Management', likes: 187),
      Post(id: '3', imageUrl: 'https://picsum.photos/seed/p3/400/400', title: 'Clean Code', likes: 312),
      Post(id: '4', imageUrl: 'https://picsum.photos/seed/p4/400/400', title: 'Widget Tree', likes: 145),
      Post(id: '5', imageUrl: 'https://picsum.photos/seed/p5/400/400', title: 'Performance Tips', likes: 278),
      Post(id: '6', imageUrl: 'https://picsum.photos/seed/p6/400/400', title: 'Testing', likes: 193),
    ],
  );

  @override
  void initState() {
    super.initState();
    _tabController = TabController(length: 3, vsync: this);
  }

  @override
  void dispose() {
    _tabController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final isLandscape =
        MediaQuery.of(context).orientation == Orientation.landscape;

    return Scaffold(
      body: isLandscape
          ? _buildLandscapeLayout()
          : _buildPortraitLayout(),
    );
  }

  Widget _buildPortraitLayout() {
    return NestedScrollView(
      headerSliverBuilder: (context, innerBoxIsScrolled) {
        return [
          SliverAppBar(
            expandedHeight: 300,
            floating: false,
            pinned: true,
            flexibleSpace: FlexibleSpaceBar(
              background: _buildProfileHeader(),
            ),
            bottom: TabBar(
              controller: _tabController,
              tabs: const [
                Tab(icon: Icon(Icons.grid_on)),
                Tab(icon: Icon(Icons.video_library)),
                Tab(icon: Icon(Icons.bookmark_border)),
              ],
            ),
          ),
        ];
      },
      body: TabBarView(
        controller: _tabController,
        children: [
          _buildPostsGrid(),
          const Center(child: Text('Videos')),
          const Center(child: Text('Saved')),
        ],
      ),
    );
  }

  Widget _buildLandscapeLayout() {
    return Row(
      children: [
        // Left: Profile info
        SizedBox(
          width: 300,
          child: SingleChildScrollView(
            child: _buildProfileHeader(showStats: true),
          ),
        ),

        // Divider
        const VerticalDivider(width: 1),

        // Right: Posts
        Expanded(child: _buildPostsGrid()),
      ],
    );
  }

  Widget _buildProfileHeader({bool showStats = false}) {
    final theme = Theme.of(context);

    return Container(
      padding: const EdgeInsets.fromLTRB(16, 60, 16, 16),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.center,
        children: [
          // Avatar with online indicator
          Stack(
            children: [
              CircleAvatar(
                radius: 50,
                backgroundImage: NetworkImage(_profile.avatarUrl),
              ),
              Positioned(
                bottom: 4,
                right: 4,
                child: Container(
                  width: 18,
                  height: 18,
                  decoration: BoxDecoration(
                    color: Colors.green,
                    shape: BoxShape.circle,
                    border: Border.all(color: Colors.white, width: 2),
                  ),
                ),
              ),
            ],
          ),

          const SizedBox(height: 12),

          // Name
          Text(
            _profile.name,
            style: theme.textTheme.headlineSmall?.copyWith(
              fontWeight: FontWeight.bold,
            ),
          ),

          // Username
          Text(
            _profile.username,
            style: theme.textTheme.bodyMedium?.copyWith(
              color: theme.colorScheme.primary,
            ),
          ),

          const SizedBox(height: 8),

          // Bio
          Text(
            _profile.bio,
            style: theme.textTheme.bodySmall?.copyWith(
              color: theme.colorScheme.onSurfaceVariant,
              height: 1.5,
            ),
            textAlign: TextAlign.center,
          ),

          const SizedBox(height: 16),

          // Stats Row
          _buildStatsRow(),

          const SizedBox(height: 16),

          // Action buttons
          _buildActionButtons(),

          const SizedBox(height: 16),

          // Skills
          _buildSkillsWrap(),
        ],
      ),
    );
  }

  Widget _buildStatsRow() {
    return Row(
      mainAxisAlignment: MainAxisAlignment.spaceEvenly,
      children: [
        _StatItem(value: '${_profile.posts}', label: 'โพสต์'),
        _StatDivider(),
        _StatItem(value: _formatNumber(_profile.followers), label: 'ผู้ติดตาม'),
        _StatDivider(),
        _StatItem(value: '${_profile.following}', label: 'ติดตาม'),
      ],
    );
  }

  Widget _buildActionButtons() {
    final theme = Theme.of(context);

    return Row(
      children: [
        Expanded(
          child: ElevatedButton(
            onPressed: () => setState(() => _isFollowing = !_isFollowing),
            style: ElevatedButton.styleFrom(
              backgroundColor: _isFollowing ? null : theme.colorScheme.primary,
              foregroundColor:
                  _isFollowing ? null : theme.colorScheme.onPrimary,
            ),
            child: Text(_isFollowing ? 'ติดตามอยู่' : 'ติดตาม'),
          ),
        ),
        const SizedBox(width: 8),
        Expanded(
          child: OutlinedButton(
            onPressed: () {},
            child: const Text('ส่งข้อความ'),
          ),
        ),
        const SizedBox(width: 8),
        IconButton.outlined(
          onPressed: () {},
          icon: const Icon(Icons.person_add_alt),
        ),
      ],
    );
  }

  Widget _buildSkillsWrap() {
    return Align(
      alignment: Alignment.centerLeft,
      child: Wrap(
        spacing: 6,
        runSpacing: 6,
        children: _profile.skills.map((skill) {
          return Chip(
            label: Text(skill),
            materialTapTargetSize: MaterialTapTargetSize.shrinkWrap,
            padding: EdgeInsets.zero,
          );
        }).toList(),
      ),
    );
  }

  Widget _buildPostsGrid() {
    return GridView.builder(
      padding: EdgeInsets.zero,
      gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
        crossAxisCount: 3,
        crossAxisSpacing: 2,
        mainAxisSpacing: 2,
      ),
      itemCount: _profile.recentPosts.length,
      itemBuilder: (context, index) {
        final post = _profile.recentPosts[index];
        return GestureDetector(
          onTap: () {},
          child: Stack(
            fit: StackFit.expand,
            children: [
              Image.network(post.imageUrl, fit: BoxFit.cover),
              Positioned(
                bottom: 4,
                left: 4,
                child: Row(
                  children: [
                    const Icon(Icons.favorite, size: 12, color: Colors.white),
                    const SizedBox(width: 2),
                    Text(
                      '${post.likes}',
                      style: const TextStyle(
                        color: Colors.white,
                        fontSize: 10,
                        fontWeight: FontWeight.bold,
                      ),
                    ),
                  ],
                ),
              ),
            ],
          ),
        );
      },
    );
  }

  String _formatNumber(int number) {
    if (number >= 1000) {
      return '${(number / 1000).toStringAsFixed(1)}k';
    }
    return number.toString();
  }
}

class _StatItem extends StatelessWidget {
  final String value;
  final String label;

  const _StatItem({required this.value, required this.label});

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text(value,
          style: const TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
        Text(label,
          style: TextStyle(color: Colors.grey[600], fontSize: 12)),
      ],
    );
  }
}

class _StatDivider extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Container(
      height: 30,
      width: 1,
      color: Colors.grey[300],
    );
  }
}
```

### lib/main.dart

```dart
import 'package:flutter/material.dart';
import 'screens/profile_screen.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Responsive Profile',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.purple),
        useMaterial3: true,
      ),
      home: const ProfileScreen(),
    );
  }
}
```

---

## เปรียบเทียบ Layout Widgets

| Widget | ทิศทาง | ใช้สำหรับ |
|--------|--------|-----------|
| Row | แนวนอน | จัดเรียง widget ซ้าย-ขวา |
| Column | แนวตั้ง | จัดเรียง widget บน-ล่าง |
| Stack | ซ้อน | Overlay, absolute positioning |
| Wrap | อัตโนมัติ | Tag, chip ที่ขึ้นบรรทัดใหม่ได้ |
| Expanded | ขยาย | เต็ม space ที่เหลือ |
| Flexible | ยืดหยุ่น | ขยายได้แต่ไม่บังคับ |
| Center | กลาง | จัดกลาง |
| Align | กำหนด | จัดตำแหน่งเอง |
| AspectRatio | สัดส่วน | รักษา ratio |

## สรุปบทที่ 25

ในบทนี้เราได้เรียนรู้:

1. **Row/Column**: mainAxis, crossAxis, flex
2. **Expanded/Flexible**: การแบ่งพื้นที่
3. **Stack/Positioned**: absolute positioning, overlay
4. **Wrap**: auto-wrapping layout
5. **Center/Align**: การจัดตำแหน่ง
6. **AspectRatio**: รักษาสัดส่วน
7. **Workshop**: Responsive profile page

บทต่อไปเราจะเรียน Navigation และ Routing
