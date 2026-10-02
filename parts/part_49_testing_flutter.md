# Part 49: Testing ใน Flutter

## บทนำ

Testing เป็นส่วนสำคัญของการพัฒนาซอฟต์แวร์คุณภาพ Flutter มีระบบ testing ที่ครบครัน ได้แก่ Unit Tests, Widget Tests, Integration Tests และ Golden Tests

---

## 49.1 ประเภทของ Tests

```
Tests
├── Unit Tests - ทดสอบ logic/functions โดดๆ (เร็ว, ไม่ต้อง UI)
├── Widget Tests - ทดสอบ widgets (ช้าปานกลาง)
├── Integration Tests - ทดสอบทั้งแอป end-to-end (ช้าที่สุด)
└── Golden Tests - เปรียบเทียบ UI screenshot
```

### ติดตั้ง Dependencies

```yaml
# pubspec.yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  integration_test:
    sdk: flutter
  mockito: ^5.4.4
  build_runner: ^2.4.7
  bloc_test: ^9.1.5
```

---

## 49.2 Unit Tests

### ทดสอบ Business Logic

```dart
// test/unit/todo_service_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:myapp/models/todo.dart';
import 'package:myapp/services/todo_service.dart';

void main() {
  group('TodoService', () {
    late TodoService service;
    
    setUp(() {
      // รันก่อนแต่ละ test
      service = TodoService();
    });
    
    tearDown(() {
      // รันหลังแต่ละ test
    });
    
    test('should start with empty todo list', () {
      expect(service.todos, isEmpty);
    });
    
    test('should add todo correctly', () {
      // Arrange
      const todoTitle = 'ทำการบ้าน';
      
      // Act
      service.addTodo(todoTitle);
      
      // Assert
      expect(service.todos.length, equals(1));
      expect(service.todos.first.title, equals(todoTitle));
      expect(service.todos.first.isCompleted, isFalse);
    });
    
    test('should toggle todo completion', () {
      service.addTodo('ออกกำลังกาย');
      final todoId = service.todos.first.id;
      
      service.toggleTodo(todoId);
      
      expect(service.todos.first.isCompleted, isTrue);
      
      service.toggleTodo(todoId);
      expect(service.todos.first.isCompleted, isFalse);
    });
    
    test('should delete todo', () {
      service.addTodo('อ่านหนังสือ');
      service.addTodo('ดูหนัง');
      
      final todoId = service.todos.first.id;
      service.deleteTodo(todoId);
      
      expect(service.todos.length, equals(1));
      expect(service.todos.any((t) => t.id == todoId), isFalse);
    });
    
    test('should filter completed todos', () {
      service.addTodo('งาน A');
      service.addTodo('งาน B');
      service.addTodo('งาน C');
      
      service.toggleTodo(service.todos[0].id);
      service.toggleTodo(service.todos[2].id);
      
      final completed = service.getCompletedTodos();
      expect(completed.length, equals(2));
    });
    
    test('should count incomplete todos', () {
      service.addTodo('งาน 1');
      service.addTodo('งาน 2');
      service.addTodo('งาน 3');
      service.toggleTodo(service.todos[0].id);
      
      expect(service.incompleteCount, equals(2));
    });
    
    group('validation', () {
      test('should throw when adding empty title', () {
        expect(
          () => service.addTodo(''),
          throwsArgumentError,
        );
      });
      
      test('should throw when adding whitespace only title', () {
        expect(
          () => service.addTodo('   '),
          throwsArgumentError,
        );
      });
    });
  });
  
  group('Todo model', () {
    test('should create todo with correct defaults', () {
      final todo = Todo(id: '1', title: 'Test');
      
      expect(todo.id, equals('1'));
      expect(todo.title, equals('Test'));
      expect(todo.isCompleted, isFalse);
      expect(todo.createdAt, isA<DateTime>());
    });
    
    test('copyWith should create new instance with updated values', () {
      final original = Todo(id: '1', title: 'Original');
      final updated = original.copyWith(
        title: 'Updated',
        isCompleted: true,
      );
      
      expect(updated.id, equals(original.id));
      expect(updated.title, equals('Updated'));
      expect(updated.isCompleted, isTrue);
      expect(original.isCompleted, isFalse); // ของเดิมไม่เปลี่ยน
    });
  });
}
```

### ทดสอบพร้อม Mocking

```dart
// test/unit/auth_service_test.dart
import 'package:mockito/annotations.dart';
import 'package:mockito/mockito.dart';

// สร้าง mock class
@GenerateMocks([AuthRepository])
import 'auth_service_test.mocks.dart';

void main() {
  group('AuthService', () {
    late AuthService authService;
    late MockAuthRepository mockRepository;
    
    setUp(() {
      mockRepository = MockAuthRepository();
      authService = AuthService(repository: mockRepository);
    });
    
    test('should login successfully', () async {
      // Arrange - กำหนดให้ mock return ค่าที่ต้องการ
      when(mockRepository.login(
        email: 'test@example.com',
        password: 'password123',
      )).thenAnswer((_) async => User(
        id: '1',
        email: 'test@example.com',
        name: 'Test User',
      ));
      
      // Act
      final user = await authService.login(
        email: 'test@example.com',
        password: 'password123',
      );
      
      // Assert
      expect(user, isNotNull);
      expect(user!.email, equals('test@example.com'));
      
      // ตรวจสอบว่า mock ถูกเรียก
      verify(mockRepository.login(
        email: 'test@example.com',
        password: 'password123',
      )).called(1);
    });
    
    test('should throw AuthException when credentials invalid', () async {
      when(mockRepository.login(
        email: anyNamed('email'),
        password: anyNamed('password'),
      )).thenThrow(const AuthException('Invalid credentials'));
      
      expect(
        () => authService.login(
          email: 'wrong@test.com',
          password: 'wrong',
        ),
        throwsA(isA<AuthException>()),
      );
    });
  });
}
```

---

## 49.3 Widget Tests

### ทดสอบ Widget พื้นฐาน

```dart
// test/widget/todo_item_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:myapp/widgets/todo_item.dart';
import 'package:myapp/models/todo.dart';

void main() {
  group('TodoItem Widget', () {
    final testTodo = Todo(
      id: '1',
      title: 'ทำการบ้านฟิสิกส์',
    );
    
    testWidgets('should display todo title', (tester) async {
      // Build widget
      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: TodoItem(
              todo: testTodo,
              onToggle: (_) {},
              onDelete: (_) {},
            ),
          ),
        ),
      );
      
      // ตรวจสอบว่า title แสดง
      expect(find.text('ทำการบ้านฟิสิกส์'), findsOneWidget);
    });
    
    testWidgets('should show checkbox unchecked when todo not completed',
        (tester) async {
      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: TodoItem(
              todo: testTodo,
              onToggle: (_) {},
              onDelete: (_) {},
            ),
          ),
        ),
      );
      
      // หา Checkbox
      final checkbox = tester.widget<Checkbox>(find.byType(Checkbox));
      expect(checkbox.value, isFalse);
    });
    
    testWidgets('should call onToggle when checkbox tapped', (tester) async {
      String? toggledId;
      
      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: TodoItem(
              todo: testTodo,
              onToggle: (id) => toggledId = id,
              onDelete: (_) {},
            ),
          ),
        ),
      );
      
      // Tap checkbox
      await tester.tap(find.byType(Checkbox));
      await tester.pump();
      
      expect(toggledId, equals(testTodo.id));
    });
    
    testWidgets('should show strikethrough text when completed', (tester) async {
      final completedTodo = testTodo.copyWith(isCompleted: true);
      
      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: TodoItem(
              todo: completedTodo,
              onToggle: (_) {},
              onDelete: (_) {},
            ),
          ),
        ),
      );
      
      final textWidget = tester.widget<Text>(
        find.text('ทำการบ้านฟิสิกส์'),
      );
      
      final textStyle = textWidget.style;
      expect(textStyle?.decoration, equals(TextDecoration.lineThrough));
    });
    
    testWidgets('should call onDelete when delete button pressed',
        (tester) async {
      String? deletedId;
      
      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: TodoItem(
              todo: testTodo,
              onToggle: (_) {},
              onDelete: (id) => deletedId = id,
            ),
          ),
        ),
      );
      
      // Tap delete button
      await tester.tap(find.byIcon(Icons.delete));
      await tester.pump();
      
      expect(deletedId, equals(testTodo.id));
    });
  });
}
```

### ทดสอบ Page ที่ซับซ้อน

```dart
// test/widget/todo_page_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:provider/provider.dart';
import 'package:myapp/pages/todo_page.dart';
import 'package:myapp/providers/todo_provider.dart';

void main() {
  group('TodoPage', () {
    late TodoProvider todoProvider;
    
    setUp(() {
      todoProvider = TodoProvider();
    });
    
    Widget buildApp() {
      return ChangeNotifierProvider.value(
        value: todoProvider,
        child: const MaterialApp(
          home: TodoPage(),
        ),
      );
    }
    
    testWidgets('should show empty state initially', (tester) async {
      await tester.pumpWidget(buildApp());
      
      expect(find.text('ยังไม่มีรายการ'), findsOneWidget);
      expect(find.byType(ListView), findsNothing);
    });
    
    testWidgets('should add todo when form submitted', (tester) async {
      await tester.pumpWidget(buildApp());
      
      // กรอก text field
      await tester.enterText(
        find.byType(TextField),
        'ซื้อของกิน',
      );
      
      // กด submit
      await tester.testTextInput.receiveAction(TextInputAction.done);
      await tester.pump();
      
      // ตรวจสอบ
      expect(find.text('ซื้อของกิน'), findsOneWidget);
      expect(find.text('ยังไม่มีรายการ'), findsNothing);
    });
    
    testWidgets('should show todos from provider', (tester) async {
      todoProvider.addTodo('งาน 1');
      todoProvider.addTodo('งาน 2');
      todoProvider.addTodo('งาน 3');
      
      await tester.pumpWidget(buildApp());
      
      expect(find.text('งาน 1'), findsOneWidget);
      expect(find.text('งาน 2'), findsOneWidget);
      expect(find.text('งาน 3'), findsOneWidget);
    });
    
    testWidgets('should filter by completed status', (tester) async {
      todoProvider.addTodo('งาน A');
      todoProvider.addTodo('งาน B');
      todoProvider.toggleTodo(todoProvider.todos.first.id);
      
      await tester.pumpWidget(buildApp());
      
      // กด filter "เสร็จแล้ว"
      await tester.tap(find.text('เสร็จแล้ว'));
      await tester.pump();
      
      expect(find.text('งาน A'), findsOneWidget);
      expect(find.text('งาน B'), findsNothing);
    });
  });
}
```

---

## 49.4 Integration Tests

```dart
// integration_test/app_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:myapp/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();
  
  group('Todo App E2E Tests', () {
    testWidgets('Complete todo workflow', (tester) async {
      app.main();
      await tester.pumpAndSettle();
      
      // 1. Login
      await tester.enterText(
        find.byKey(const Key('email_field')),
        'test@example.com',
      );
      await tester.enterText(
        find.byKey(const Key('password_field')),
        'password123',
      );
      await tester.tap(find.byKey(const Key('login_button')));
      await tester.pumpAndSettle();
      
      // ตรวจสอบว่า login สำเร็จ
      expect(find.text('หน้าหลัก'), findsOneWidget);
      
      // 2. เพิ่ม todo
      await tester.tap(find.byType(FloatingActionButton));
      await tester.pumpAndSettle();
      
      await tester.enterText(
        find.byKey(const Key('todo_input')),
        'ทดสอบงาน Integration Test',
      );
      
      await tester.tap(find.byKey(const Key('add_button')));
      await tester.pumpAndSettle();
      
      // ตรวจสอบ
      expect(
        find.text('ทดสอบงาน Integration Test'),
        findsOneWidget,
      );
      
      // 3. Toggle todo
      await tester.tap(
        find.byKey(const Key('todo_checkbox_0')),
      );
      await tester.pumpAndSettle();
      
      // ตรวจสอบ strikethrough
      final textWidget = tester.widget<Text>(
        find.text('ทดสอบงาน Integration Test'),
      );
      expect(
        textWidget.style?.decoration,
        equals(TextDecoration.lineThrough),
      );
      
      // 4. ลบ todo
      await tester.longPress(
        find.text('ทดสอบงาน Integration Test'),
      );
      await tester.pumpAndSettle();
      
      await tester.tap(find.text('ลบ'));
      await tester.pumpAndSettle();
      
      expect(
        find.text('ทดสอบงาน Integration Test'),
        findsNothing,
      );
    });
  });
}
```

---

## 49.5 Golden Tests

```dart
// test/golden/todo_item_golden_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:myapp/widgets/todo_item.dart';
import 'package:myapp/models/todo.dart';

void main() {
  group('TodoItem Golden Tests', () {
    testWidgets('normal state matches golden', (tester) async {
      final todo = Todo(id: '1', title: 'ทดสอบ Golden Test');
      
      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: TodoItem(
              todo: todo,
              onToggle: (_) {},
              onDelete: (_) {},
            ),
          ),
        ),
      );
      
      await expectLater(
        find.byType(TodoItem),
        matchesGoldenFile('goldens/todo_item_normal.png'),
      );
    });
    
    testWidgets('completed state matches golden', (tester) async {
      final todo = Todo(
        id: '1',
        title: 'ทดสอบ Golden Test',
        isCompleted: true,
      );
      
      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: TodoItem(
              todo: todo,
              onToggle: (_) {},
              onDelete: (_) {},
            ),
          ),
        ),
      );
      
      await expectLater(
        find.byType(TodoItem),
        matchesGoldenFile('goldens/todo_item_completed.png'),
      );
    });
    
    testWidgets('dark mode matches golden', (tester) async {
      final todo = Todo(id: '1', title: 'Dark Mode Test');
      
      await tester.pumpWidget(
        MaterialApp(
          theme: ThemeData.dark(),
          home: Scaffold(
            body: TodoItem(
              todo: todo,
              onToggle: (_) {},
              onDelete: (_) {},
            ),
          ),
        ),
      );
      
      await expectLater(
        find.byType(TodoItem),
        matchesGoldenFile('goldens/todo_item_dark.png'),
      );
    });
  });
}
```

---

## 49.6 Workshop: Test Todo App

### Test Helpers

```dart
// test/helpers/test_helpers.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:provider/provider.dart';
import 'package:myapp/providers/todo_provider.dart';

// Helper สำหรับสร้าง widget ในการทดสอบ
Widget buildTestApp({
  required Widget child,
  TodoProvider? todoProvider,
}) {
  return MultiProvider(
    providers: [
      ChangeNotifierProvider.value(
        value: todoProvider ?? TodoProvider(),
      ),
    ],
    child: MaterialApp(home: child),
  );
}

// Helper สำหรับ pump หลังจาก async
extension WidgetTesterHelper on WidgetTester {
  Future<void> pumpUntilFound(Finder finder, {Duration? timeout}) async {
    final deadline = DateTime.now().add(timeout ?? const Duration(seconds: 5));
    bool found = false;
    
    while (!found) {
      try {
        found = finder.evaluate().isNotEmpty;
        if (!found) await pump(const Duration(milliseconds: 100));
      } catch (_) {
        await pump(const Duration(milliseconds: 100));
      }
      
      if (DateTime.now().isAfter(deadline)) {
        throw TimeoutException('Widget not found: $finder');
      }
    }
  }
}

// Custom Matcher
Matcher hasTextDecoration(TextDecoration decoration) {
  return _HasTextDecorationMatcher(decoration);
}

class _HasTextDecorationMatcher extends Matcher {
  final TextDecoration _decoration;
  
  _HasTextDecorationMatcher(this._decoration);
  
  @override
  bool matches(item, Map matchState) {
    if (item is Text) {
      return item.style?.decoration == _decoration;
    }
    return false;
  }
  
  @override
  Description describe(Description description) {
    return description.add('Text with decoration $_decoration');
  }
}
```

### ทดสอบ BLoC

```dart
// test/bloc/todo_bloc_test.dart
import 'package:bloc_test/bloc_test.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:myapp/bloc/todo_bloc.dart';

void main() {
  group('TodoBloc', () {
    late TodoBloc bloc;
    
    setUp(() {
      bloc = TodoBloc();
    });
    
    tearDown(() {
      bloc.close();
    });
    
    test('initial state is TodoInitial', () {
      expect(bloc.state, isA<TodoInitial>());
    });
    
    blocTest<TodoBloc, TodoState>(
      'emits [TodoLoaded] when LoadTodos is added',
      build: () => TodoBloc(),
      act: (bloc) => bloc.add(LoadTodos()),
      expect: () => [
        isA<TodoLoading>(),
        isA<TodoLoaded>(),
      ],
    );
    
    blocTest<TodoBloc, TodoState>(
      'emits [TodoLoaded] with new todo when AddTodo is added',
      build: () => TodoBloc(),
      seed: () => const TodoLoaded(todos: []),
      act: (bloc) => bloc.add(const AddTodo(title: 'Test Todo')),
      expect: () => [
        isA<TodoLoaded>().having(
          (s) => s.todos.length,
          'todos count',
          equals(1),
        ),
      ],
    );
    
    blocTest<TodoBloc, TodoState>(
      'emits error state when AddTodo fails',
      build: () => TodoBloc(),
      act: (bloc) => bloc.add(const AddTodo(title: '')),
      expect: () => [isA<TodoError>()],
    );
  });
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- Unit Tests สำหรับทดสอบ business logic
- Widget Tests สำหรับทดสอบ UI components
- Integration Tests สำหรับ end-to-end testing
- Golden Tests สำหรับ UI regression testing
- Mocking ด้วย Mockito
- Workshop: Test Todo App ครบวงจร

**แบบฝึกหัดเพิ่มเติม:**
1. เพิ่ม code coverage report
2. Setup CI/CD ให้รัน tests อัตโนมัติ
3. ทดสอบ network requests พร้อม mock HTTP
4. สร้าง test fixtures สำหรับข้อมูลทดสอบ
