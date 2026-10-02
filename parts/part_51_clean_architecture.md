# Part 51: Clean Architecture

## บทนำ

Clean Architecture เป็นรูปแบบการออกแบบซอฟต์แวร์ที่แยก concerns ออกเป็น layers ชัดเจน ทำให้ code maintainable, testable และ scalable มากขึ้น Uncle Bob (Robert C. Martin) เป็นผู้พัฒนาแนวคิดนี้

---

## 51.1 โครงสร้าง Layers

```
┌─────────────────────────────────────────┐
│           Presentation Layer            │
│   (UI, Widgets, Pages, ViewModels)      │
├─────────────────────────────────────────┤
│             Domain Layer                │
│   (Use Cases, Entities, Repositories)  │
│         (Business Logic HERE)           │
├─────────────────────────────────────────┤
│              Data Layer                 │
│  (Repository Impl, Data Sources, DTOs)  │
└─────────────────────────────────────────┘

กฎสำคัญ: Dependencies ชี้เข้าใน (inward)
Data → Domain ← Presentation
```

### โครงสร้าง Folder

```
lib/
├── features/
│   └── todo/
│       ├── data/
│       │   ├── datasources/
│       │   │   ├── todo_local_datasource.dart
│       │   │   └── todo_remote_datasource.dart
│       │   ├── models/
│       │   │   └── todo_model.dart
│       │   └── repositories/
│       │       └── todo_repository_impl.dart
│       ├── domain/
│       │   ├── entities/
│       │   │   └── todo.dart
│       │   ├── repositories/
│       │   │   └── todo_repository.dart  (abstract)
│       │   └── usecases/
│       │       ├── get_todos.dart
│       │       ├── add_todo.dart
│       │       ├── update_todo.dart
│       │       └── delete_todo.dart
│       └── presentation/
│           ├── bloc/
│           │   ├── todo_bloc.dart
│           │   ├── todo_event.dart
│           │   └── todo_state.dart
│           ├── pages/
│           │   └── todo_page.dart
│           └── widgets/
│               └── todo_item.dart
└── core/
    ├── error/
    │   ├── exceptions.dart
    │   └── failures.dart
    ├── usecases/
    │   └── usecase.dart
    └── utils/
        └── either.dart
```

---

## 51.2 Domain Layer

### Entities

```dart
// lib/features/todo/domain/entities/todo.dart
import 'package:equatable/equatable.dart';

class Todo extends Equatable {
  final String id;
  final String title;
  final String? description;
  final bool isCompleted;
  final DateTime createdAt;
  final DateTime? completedAt;
  final int priority;
  
  const Todo({
    required this.id,
    required this.title,
    this.description,
    this.isCompleted = false,
    required this.createdAt,
    this.completedAt,
    this.priority = 0,
  });
  
  Todo copyWith({
    String? title,
    String? description,
    bool? isCompleted,
    DateTime? completedAt,
    int? priority,
  }) {
    return Todo(
      id: id,
      title: title ?? this.title,
      description: description ?? this.description,
      isCompleted: isCompleted ?? this.isCompleted,
      createdAt: createdAt,
      completedAt: completedAt ?? this.completedAt,
      priority: priority ?? this.priority,
    );
  }
  
  @override
  List<Object?> get props => [
    id, title, description, isCompleted,
    createdAt, completedAt, priority,
  ];
}
```

### Repository Interface (Abstract)

```dart
// lib/features/todo/domain/repositories/todo_repository.dart
import 'package:dartz/dartz.dart';
import '../../../../core/error/failures.dart';
import '../entities/todo.dart';

abstract class TodoRepository {
  Future<Either<Failure, List<Todo>>> getTodos();
  Future<Either<Failure, Todo>> getTodoById(String id);
  Future<Either<Failure, Todo>> addTodo(Todo todo);
  Future<Either<Failure, Todo>> updateTodo(Todo todo);
  Future<Either<Failure, Unit>> deleteTodo(String id);
  Stream<List<Todo>> watchTodos();
}
```

### Use Cases

```dart
// lib/core/usecases/usecase.dart
import 'package:dartz/dartz.dart';
import '../error/failures.dart';

abstract class UseCase<Type, Params> {
  Future<Either<Failure, Type>> call(Params params);
}

// สำหรับ use case ที่ไม่มี params
class NoParams {
  const NoParams();
}
```

```dart
// lib/features/todo/domain/usecases/get_todos.dart
import 'package:dartz/dartz.dart';
import '../../../../core/error/failures.dart';
import '../../../../core/usecases/usecase.dart';
import '../entities/todo.dart';
import '../repositories/todo_repository.dart';

class GetTodos implements UseCase<List<Todo>, NoParams> {
  final TodoRepository repository;
  
  const GetTodos(this.repository);
  
  @override
  Future<Either<Failure, List<Todo>>> call(NoParams params) async {
    return await repository.getTodos();
  }
}
```

```dart
// lib/features/todo/domain/usecases/add_todo.dart
import 'package:dartz/dartz.dart';
import '../../../../core/error/failures.dart';
import '../../../../core/usecases/usecase.dart';
import '../entities/todo.dart';
import '../repositories/todo_repository.dart';

class AddTodo implements UseCase<Todo, AddTodoParams> {
  final TodoRepository repository;
  
  const AddTodo(this.repository);
  
  @override
  Future<Either<Failure, Todo>> call(AddTodoParams params) async {
    // Validate
    if (params.title.trim().isEmpty) {
      return Left(const ValidationFailure('Title cannot be empty'));
    }
    
    final todo = Todo(
      id: DateTime.now().millisecondsSinceEpoch.toString(),
      title: params.title.trim(),
      description: params.description,
      createdAt: DateTime.now(),
      priority: params.priority,
    );
    
    return await repository.addTodo(todo);
  }
}

class AddTodoParams {
  final String title;
  final String? description;
  final int priority;
  
  const AddTodoParams({
    required this.title,
    this.description,
    this.priority = 0,
  });
}
```

---

## 51.3 Data Layer

### Models (DTOs)

```dart
// lib/features/todo/data/models/todo_model.dart
import '../../domain/entities/todo.dart';

class TodoModel extends Todo {
  const TodoModel({
    required super.id,
    required super.title,
    super.description,
    super.isCompleted,
    required super.createdAt,
    super.completedAt,
    super.priority,
  });
  
  // สร้างจาก JSON
  factory TodoModel.fromJson(Map<String, dynamic> json) {
    return TodoModel(
      id: json['id'] as String,
      title: json['title'] as String,
      description: json['description'] as String?,
      isCompleted: json['isCompleted'] as bool? ?? false,
      createdAt: DateTime.fromMillisecondsSinceEpoch(
        json['createdAt'] as int,
      ),
      completedAt: json['completedAt'] != null
          ? DateTime.fromMillisecondsSinceEpoch(json['completedAt'] as int)
          : null,
      priority: json['priority'] as int? ?? 0,
    );
  }
  
  // แปลงเป็น JSON
  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'title': title,
      'description': description,
      'isCompleted': isCompleted,
      'createdAt': createdAt.millisecondsSinceEpoch,
      'completedAt': completedAt?.millisecondsSinceEpoch,
      'priority': priority,
    };
  }
  
  // สร้างจาก Entity
  factory TodoModel.fromEntity(Todo todo) {
    return TodoModel(
      id: todo.id,
      title: todo.title,
      description: todo.description,
      isCompleted: todo.isCompleted,
      createdAt: todo.createdAt,
      completedAt: todo.completedAt,
      priority: todo.priority,
    );
  }
  
  // สร้างจาก Firestore
  factory TodoModel.fromFirestore(Map<String, dynamic> data, String id) {
    return TodoModel(
      id: id,
      title: data['title'] as String,
      description: data['description'] as String?,
      isCompleted: data['isCompleted'] as bool? ?? false,
      createdAt: (data['createdAt'] as Timestamp).toDate(),
      completedAt: (data['completedAt'] as Timestamp?)?.toDate(),
      priority: data['priority'] as int? ?? 0,
    );
  }
}
```

### Data Sources

```dart
// lib/features/todo/data/datasources/todo_local_datasource.dart
import 'dart:convert';
import 'package:shared_preferences/shared_preferences.dart';
import '../../../../core/error/exceptions.dart';
import '../models/todo_model.dart';

abstract class TodoLocalDataSource {
  Future<List<TodoModel>> getCachedTodos();
  Future<void> cacheTodos(List<TodoModel> todos);
  Future<void> clearCache();
}

class TodoLocalDataSourceImpl implements TodoLocalDataSource {
  static const String _todosKey = 'CACHED_TODOS';
  
  final SharedPreferences sharedPreferences;
  
  TodoLocalDataSourceImpl({required this.sharedPreferences});
  
  @override
  Future<List<TodoModel>> getCachedTodos() async {
    final jsonString = sharedPreferences.getString(_todosKey);
    
    if (jsonString == null) {
      throw const CacheException('No cached todos found');
    }
    
    final List<dynamic> jsonList = json.decode(jsonString);
    return jsonList
        .map((json) => TodoModel.fromJson(json as Map<String, dynamic>))
        .toList();
  }
  
  @override
  Future<void> cacheTodos(List<TodoModel> todos) async {
    final jsonString = json.encode(
      todos.map((todo) => todo.toJson()).toList(),
    );
    await sharedPreferences.setString(_todosKey, jsonString);
  }
  
  @override
  Future<void> clearCache() async {
    await sharedPreferences.remove(_todosKey);
  }
}
```

```dart
// lib/features/todo/data/datasources/todo_remote_datasource.dart
import 'package:cloud_firestore/cloud_firestore.dart';
import '../../../../core/error/exceptions.dart';
import '../models/todo_model.dart';

abstract class TodoRemoteDataSource {
  Future<List<TodoModel>> getTodos();
  Future<TodoModel> addTodo(TodoModel todo);
  Future<TodoModel> updateTodo(TodoModel todo);
  Future<void> deleteTodo(String id);
  Stream<List<TodoModel>> watchTodos();
}

class TodoFirestoreDataSource implements TodoRemoteDataSource {
  final FirebaseFirestore _db;
  final String _userId;
  
  TodoFirestoreDataSource({
    required FirebaseFirestore db,
    required String userId,
  }) : _db = db, _userId = userId;
  
  CollectionReference get _todosRef =>
      _db.collection('users').doc(_userId).collection('todos');
  
  @override
  Future<List<TodoModel>> getTodos() async {
    try {
      final snapshot = await _todosRef
          .orderBy('createdAt', descending: true)
          .get();
      
      return snapshot.docs
          .map((doc) => TodoModel.fromFirestore(
                doc.data() as Map<String, dynamic>,
                doc.id,
              ))
          .toList();
    } catch (e) {
      throw ServerException(e.toString());
    }
  }
  
  @override
  Future<TodoModel> addTodo(TodoModel todo) async {
    try {
      final docRef = await _todosRef.add({
        'title': todo.title,
        'description': todo.description,
        'isCompleted': todo.isCompleted,
        'createdAt': FieldValue.serverTimestamp(),
        'priority': todo.priority,
      });
      
      final doc = await docRef.get();
      return TodoModel.fromFirestore(
        doc.data() as Map<String, dynamic>,
        doc.id,
      );
    } catch (e) {
      throw ServerException(e.toString());
    }
  }
  
  @override
  Future<TodoModel> updateTodo(TodoModel todo) async {
    try {
      await _todosRef.doc(todo.id).update({
        'title': todo.title,
        'description': todo.description,
        'isCompleted': todo.isCompleted,
        'completedAt': todo.isCompleted
            ? FieldValue.serverTimestamp()
            : null,
        'priority': todo.priority,
      });
      
      final doc = await _todosRef.doc(todo.id).get();
      return TodoModel.fromFirestore(
        doc.data() as Map<String, dynamic>,
        doc.id,
      );
    } catch (e) {
      throw ServerException(e.toString());
    }
  }
  
  @override
  Future<void> deleteTodo(String id) async {
    try {
      await _todosRef.doc(id).delete();
    } catch (e) {
      throw ServerException(e.toString());
    }
  }
  
  @override
  Stream<List<TodoModel>> watchTodos() {
    return _todosRef
        .orderBy('createdAt', descending: true)
        .snapshots()
        .map((snapshot) => snapshot.docs
            .map((doc) => TodoModel.fromFirestore(
                  doc.data() as Map<String, dynamic>,
                  doc.id,
                ))
            .toList());
  }
}
```

### Repository Implementation

```dart
// lib/features/todo/data/repositories/todo_repository_impl.dart
import 'package:dartz/dartz.dart';
import '../../../../core/error/exceptions.dart';
import '../../../../core/error/failures.dart';
import '../../domain/entities/todo.dart';
import '../../domain/repositories/todo_repository.dart';
import '../datasources/todo_local_datasource.dart';
import '../datasources/todo_remote_datasource.dart';
import '../models/todo_model.dart';

class TodoRepositoryImpl implements TodoRepository {
  final TodoRemoteDataSource remoteDataSource;
  final TodoLocalDataSource localDataSource;
  final NetworkInfo networkInfo;
  
  TodoRepositoryImpl({
    required this.remoteDataSource,
    required this.localDataSource,
    required this.networkInfo,
  });
  
  @override
  Future<Either<Failure, List<Todo>>> getTodos() async {
    if (await networkInfo.isConnected) {
      try {
        final todos = await remoteDataSource.getTodos();
        await localDataSource.cacheTodos(todos);
        return Right(todos);
      } on ServerException catch (e) {
        return Left(ServerFailure(e.message));
      }
    } else {
      try {
        final todos = await localDataSource.getCachedTodos();
        return Right(todos);
      } on CacheException catch (e) {
        return Left(CacheFailure(e.message));
      }
    }
  }
  
  @override
  Future<Either<Failure, Todo>> addTodo(Todo todo) async {
    try {
      final model = TodoModel.fromEntity(todo);
      final result = await remoteDataSource.addTodo(model);
      return Right(result);
    } on ServerException catch (e) {
      return Left(ServerFailure(e.message));
    }
  }
  
  @override
  Future<Either<Failure, Todo>> updateTodo(Todo todo) async {
    try {
      final model = TodoModel.fromEntity(todo);
      final result = await remoteDataSource.updateTodo(model);
      return Right(result);
    } on ServerException catch (e) {
      return Left(ServerFailure(e.message));
    }
  }
  
  @override
  Future<Either<Failure, Unit>> deleteTodo(String id) async {
    try {
      await remoteDataSource.deleteTodo(id);
      return const Right(unit);
    } on ServerException catch (e) {
      return Left(ServerFailure(e.message));
    }
  }
  
  @override
  Stream<List<Todo>> watchTodos() {
    return remoteDataSource.watchTodos();
  }
  
  @override
  Future<Either<Failure, Todo>> getTodoById(String id) {
    throw UnimplementedError();
  }
}
```

---

## 51.4 Dependency Injection ด้วย get_it

```yaml
# pubspec.yaml
dependencies:
  get_it: ^7.6.7
  injectable: ^2.3.2

dev_dependencies:
  injectable_generator: ^2.4.1
```

```dart
// lib/core/di/injection_container.dart
import 'package:get_it/get_it.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:shared_preferences/shared_preferences.dart';

final GetIt sl = GetIt.instance;

Future<void> setupDependencies() async {
  // External
  final sharedPrefs = await SharedPreferences.getInstance();
  sl.registerLazySingleton<SharedPreferences>(() => sharedPrefs);
  sl.registerLazySingleton<FirebaseFirestore>(
    () => FirebaseFirestore.instance,
  );
  sl.registerLazySingleton<FirebaseAuth>(() => FirebaseAuth.instance);
  
  // Core
  sl.registerLazySingleton<NetworkInfo>(
    () => NetworkInfoImpl(sl()),
  );
  
  // Data Sources
  sl.registerLazySingleton<TodoLocalDataSource>(
    () => TodoLocalDataSourceImpl(sharedPreferences: sl()),
  );
  sl.registerLazySingleton<TodoRemoteDataSource>(
    () => TodoFirestoreDataSource(
      db: sl(),
      userId: FirebaseAuth.instance.currentUser!.uid,
    ),
  );
  
  // Repositories
  sl.registerLazySingleton<TodoRepository>(
    () => TodoRepositoryImpl(
      remoteDataSource: sl(),
      localDataSource: sl(),
      networkInfo: sl(),
    ),
  );
  
  // Use Cases
  sl.registerLazySingleton(() => GetTodos(sl()));
  sl.registerLazySingleton(() => AddTodo(sl()));
  sl.registerLazySingleton(() => UpdateTodo(sl()));
  sl.registerLazySingleton(() => DeleteTodo(sl()));
  
  // BLoC
  sl.registerFactory(() => TodoBloc(
    getTodos: sl(),
    addTodo: sl(),
    updateTodo: sl(),
    deleteTodo: sl(),
  ));
}
```

---

## 51.5 Presentation Layer (BLoC)

```dart
// lib/features/todo/presentation/bloc/todo_bloc.dart
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:dartz/dartz.dart';
import '../../domain/usecases/get_todos.dart';
import '../../domain/usecases/add_todo.dart';
import 'todo_event.dart';
import 'todo_state.dart';

class TodoBloc extends Bloc<TodoEvent, TodoState> {
  final GetTodos getTodos;
  final AddTodo addTodo;
  final UpdateTodo updateTodo;
  final DeleteTodo deleteTodo;
  
  TodoBloc({
    required this.getTodos,
    required this.addTodo,
    required this.updateTodo,
    required this.deleteTodo,
  }) : super(const TodoInitial()) {
    on<LoadTodosEvent>(_onLoadTodos);
    on<AddTodoEvent>(_onAddTodo);
    on<UpdateTodoEvent>(_onUpdateTodo);
    on<DeleteTodoEvent>(_onDeleteTodo);
    on<ToggleTodoEvent>(_onToggleTodo);
  }
  
  Future<void> _onLoadTodos(
    LoadTodosEvent event,
    Emitter<TodoState> emit,
  ) async {
    emit(const TodoLoading());
    
    final result = await getTodos(const NoParams());
    
    result.fold(
      (failure) => emit(TodoError(message: failure.message)),
      (todos) => emit(TodoLoaded(todos: todos)),
    );
  }
  
  Future<void> _onAddTodo(
    AddTodoEvent event,
    Emitter<TodoState> emit,
  ) async {
    final result = await addTodo(AddTodoParams(
      title: event.title,
      description: event.description,
      priority: event.priority,
    ));
    
    result.fold(
      (failure) => emit(TodoError(message: failure.message)),
      (todo) {
        if (state is TodoLoaded) {
          final currentTodos = (state as TodoLoaded).todos;
          emit(TodoLoaded(todos: [todo, ...currentTodos]));
        }
      },
    );
  }
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- โครงสร้าง Clean Architecture 3 layers
- Domain Layer: Entities, Repository Interfaces, Use Cases
- Data Layer: Models, Data Sources, Repository Implementation
- Dependency Injection ด้วย get_it
- Workshop: Refactor Todo App เป็น Clean Architecture

**แบบฝึกหัดเพิ่มเติม:**
1. เพิ่ม feature categories/tags สำหรับ todo
2. Implement search และ filter ด้วย use cases
3. เพิ่ม offline sync mechanism
4. สร้าง test สำหรับทุก layer
