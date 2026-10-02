# Part 99: Career Development และ Portfolio

## 🎯 เป้าหมายของ Part นี้
- สร้าง Portfolio ที่แข็งแกร่ง
- GitHub profile optimization
- Open source contributions
- Interview preparation
- Flutter developer roadmap

---

## 1. สร้าง Portfolio ที่น่าประทับใจ

### โปรเจคที่ควรมีใน Portfolio:

```
1. แอป E-Commerce (แสดง: State management, Firebase, UI)
2. แอป Social Media (แสดง: Real-time, Firestore, Complex UI)
3. แอป Personal Finance (แสดง: Security, Charts, Local DB)
4. Package ที่ publish บน pub.dev (แสดง: เขียน library ได้)
5. Contribution ไปยัง Flutter/Dart open source
```

### Portfolio GitHub README:
```markdown
# [ชื่อของคุณ] - Flutter Developer 🚀

> Building cross-platform applications with Flutter

## 📱 Featured Projects

### [ShopApp](link) - E-Commerce Flutter App
[![Flutter](https://img.shields.io/badge/Flutter-3.x-02569B?logo=flutter)](https://flutter.dev)
[![Firebase](https://img.shields.io/badge/Firebase-FFCA28?logo=firebase)](https://firebase.google.com)
- Cross-platform (iOS, Android, Web)
- Clean Architecture + Riverpod
- Firebase Authentication & Firestore
- 50+ unit & widget tests

### [SocialApp](link) - Social Media App
- Real-time feed with Firestore
- Story feature with animations
- Image upload & CDN

### [FinApp](link) - Personal Finance Tracker
- Biometric authentication
- Interactive charts (fl_chart)
- Offline-first with SQLite

## 🛠 Tech Stack
| Area | Technologies |
|------|-------------|
| Mobile | Flutter, Dart |
| State | Riverpod, BLoC, Provider |
| Backend | Firebase, REST APIs, GraphQL |
| Database | Firestore, SQLite, Hive |
| Testing | flutter_test, mockito, Patrol |
| CI/CD | GitHub Actions, Fastlane |

## 📊 GitHub Stats
![stats](https://github-readme-stats.vercel.app/api?username=yourusername)

## 📫 Contact
- Email: your@email.com
- LinkedIn: linkedin.com/in/yourname
- Twitter: @yourhandle
```

---

## 2. Interview Preparation

### คำถาม Technical ที่ต้องรู้:

```dart
// Q1: อธิบาย Widget Tree, Element Tree, Render Tree
/*
- Widget Tree: immutable description ของ UI (blueprint)
- Element Tree: mutable instance ที่ manage lifecycle
- Render Tree: ทำ layout และ paint

เมื่อ setState() ถูกเรียก:
1. Widget rebuild (สร้าง Widget Tree ใหม่)
2. Element reconcile (compare old vs new widgets)
3. Update Render Tree เฉพาะส่วนที่เปลี่ยน
*/

// Q2: const Widget มีประโยชน์อย่างไร?
/*
const Widget:
- สร้างครั้งเดียว, ไม่ rebuild แม้ parent rebuild
- ประหยัด memory
- ปรับปรุง performance
*/
const Text('Hello'); // ✅ ไม่ rebuild
Text('Hello');       // ❌ rebuild ทุกครั้ง

// Q3: อธิบาย BuildContext
/*
BuildContext:
- reference ไปยัง location ของ widget ใน tree
- ใช้หา InheritedWidgets (Theme, MediaQuery, etc.)
- ใช้กับ Navigator.of(context), Theme.of(context)
*/

// Q4: StatelessWidget vs StatefulWidget
/*
StatelessWidget:
- ไม่มี mutable state
- build() เรียกครั้งเดียว (หรือเมื่อ parent rebuild)
- เร็วกว่า

StatefulWidget:
- มี State object แยก
- rebuild เมื่อ setState()
- มี lifecycle: initState, didUpdateWidget, dispose
*/

// Q5: ทำไม dispose() ถึงสำคัญ?
class MyWidget extends StatefulWidget {
  @override
  State<MyWidget> createState() => _MyWidgetState();
}

class _MyWidgetState extends State<MyWidget> {
  late AnimationController _controller;
  late StreamSubscription _sub;
  
  @override
  void initState() {
    super.initState();
    _controller = AnimationController(vsync: this);
    _sub = someStream.listen((_) {});
  }
  
  @override
  void dispose() {
    _controller.dispose(); // ถ้าไม่ dispose จะ memory leak!
    _sub.cancel();         // ถ้าไม่ cancel จะ memory leak!
    super.dispose();
  }
}

// Q6: Keys ใน Flutter คืออะไร?
/*
Keys ช่วย Flutter identify widgets ระหว่าง rebuilds:
- ValueKey, ObjectKey - ใช้ค่าเป็น identifier
- UniqueKey - สร้าง unique id ทุกครั้ง
- GlobalKey - เข้าถึง State จากที่อื่น

ใช้เมื่อ:
- Reorder items ใน list
- มีการเพิ่ม/ลบ items ที่ stateful
*/

// Q7: Provider vs InheritedWidget
/*
InheritedWidget คือ Flutter built-in mechanism สำหรับ share data
Provider เป็น wrapper ที่สะดวกกว่า ใช้ InheritedWidget ข้างใต้

Provider ดีกว่าเพราะ:
- API ที่ใช้งานง่ายกว่า
- Type-safe
- dispose() จัดการให้
*/
```

### Behavioral Questions:
```
Q: อธิบาย project ที่ท้าทายที่สุดที่คุณทำ
A: อธิบาย STAR method:
   - Situation: project context
   - Task: สิ่งที่คุณต้องทำ
   - Action: วิธีที่คุณแก้ปัญหา
   - Result: ผลลัพธ์ที่ได้

Q: คุณ handle performance issues ใน Flutter อย่างไร?
A: 
1. ใช้ DevTools profiler หา bottleneck
2. ตรวจสอบ const widgets
3. ใช้ ListView.builder แทน ListView
4. RepaintBoundary สำหรับ complex animations
5. compute() สำหรับ heavy processing
6. Image caching
7. Lazy loading

Q: Agile/Scrum experience?
A: Sprint planning, daily standups, retrospectives
```

---

## 3. Flutter Developer Roadmap

```
Level 1 - Beginner (1-3 เดือน):
✅ Dart basics
✅ Flutter widgets
✅ State management (setState)
✅ Navigation
✅ Forms
✅ HTTP requests
✅ สร้างแอปง่ายๆ ได้

Level 2 - Junior Developer (3-6 เดือน):
✅ Provider/Riverpod
✅ Firebase (Auth, Firestore, Storage)
✅ Local storage (SQLite, Hive)
✅ Custom widgets
✅ Animations
✅ Testing basics
✅ สร้าง full-stack app ได้

Level 3 - Mid Developer (6-12 เดือน):
✅ Clean Architecture
✅ BLoC pattern
✅ Design patterns
✅ Performance optimization
✅ CI/CD
✅ App Store deployment
✅ Advanced Flutter (Custom painters, Render Objects)

Level 4 - Senior Developer (1-2+ ปี):
✅ Large-scale architecture
✅ Team leadership
✅ Code review
✅ Mentoring
✅ Performance profiling
✅ Security implementation
✅ Open source contributions

Level 5 - Principal/Lead Developer:
✅ Technical strategy
✅ Cross-functional collaboration
✅ Platform decisions
✅ Framework contributions
✅ Community leadership
```

---

## 4. Salary Ranges (Thailand 2024-2025)

```
Junior Flutter Developer (0-2 ปี):
- Bangkok: 25,000 - 50,000 THB/เดือน
- Remote: สูงขึ้น 20-30%

Mid Flutter Developer (2-4 ปี):
- Bangkok: 50,000 - 90,000 THB/เดือน

Senior Flutter Developer (4+ ปี):
- Bangkok: 90,000 - 150,000+ THB/เดือน

Lead/Principal (6+ ปี):
- Bangkok: 150,000 - 250,000+ THB/เดือน

Freelance:
- Thai clients: 500 - 2,000 THB/ชั่วโมง
- International: $30 - $100+/ชั่วโมง
```

---

## 5. Resources สำหรับเรียนต่อ

```
📚 Official Docs:
- flutter.dev
- dart.dev
- pub.dev

🎥 YouTube Channels:
- Flutter Official
- Reso Coder
- The Flutter Way
- Robert Brunhage
- Mitch Koko

📖 Books:
- "Flutter & Dart: The Complete Developer's Guide" - Maximilian Schwarzmüller
- "Programming Flutter" - Carmine Zaccagnino
- "Flutter in Action" - Eric Windmill

🎓 Courses:
- Udemy: Flutter & Dart - The Complete Guide (Maximilian)
- Flutter Apprentice (raywenderlich.com)
- freeCodeCamp Flutter course

🌐 Communities:
- Flutter Discord (discord.gg/flutter)
- Flutter Reddit (r/FlutterDev)
- Flutter Dev Thailand (Facebook Group)
- Stack Overflow [flutter] tag

📱 Practice:
- Flutter challenges: flutterchallenge.com
- UI/UX inspiration: dribbble.com, behance.net
- Build mobile UI challenges
```

---

## 6. สรุป Part 99

สิ่งที่เรียนรู้:
- ✅ Portfolio building strategies
- ✅ Technical interview preparation
- ✅ Flutter developer roadmap
- ✅ Salary expectations (Thailand market)
- ✅ Learning resources

---

## ➡️ Part สุดท้าย
**Part 100: Best Practices สรุปรวม - Master Checklist**
