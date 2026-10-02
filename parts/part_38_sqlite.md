# Part 38: SQLite Database

## SQLite ใน Flutter

SQLite เป็น relational database ที่ทำงานบนเครื่อง เหมาะสำหรับ structured data ที่ต้องการ queries ซับซ้อน

```yaml
# pubspec.yaml
dependencies:
  sqflite: ^2.3.0
  path: ^1.8.3
```

---

## sqflite Package

### Database Creation

```dart
import 'dart:async';
import 'package:sqflite/sqflite.dart';
import 'package:path/path.dart';

class DatabaseHelper {
  static const String _dbName = 'notes.db';
  static const int _dbVersion = 1;
  
  Database? _database;
  
  Future<Database> get database async {
    if (_database != null) return _database!;
    _database = await _initDatabase();
    return _database!;
  }
  
  Future<Database> _initDatabase() async {
    // ได้ path ของ database
    final dbPath = await getDatabasesPath();
    final path = join(dbPath, _dbName);
    
    return await openDatabase(
      path,
      version: _dbVersion,
      onCreate: _createTables,
      onUpgrade: _upgradeTables,
    );
  }
  
  Future<void> _createTables(Database db, int version) async {
    // สร้าง tables
    await db.execute('''
      CREATE TABLE notes (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        title TEXT NOT NULL,
        content TEXT,
        color INTEGER DEFAULT 0xFF2196F3,
        is_pinned INTEGER DEFAULT 0,
        created_at TEXT NOT NULL,
        updated_at TEXT NOT NULL
      )
    ''');
    
    await db.execute('''
      CREATE TABLE tags (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL UNIQUE,
        color INTEGER DEFAULT 0xFF4CAF50
      )
    ''');
    
    await db.execute('''
      CREATE TABLE note_tags (
        note_id INTEGER NOT NULL,
        tag_id INTEGER NOT NULL,
        PRIMARY KEY (note_id, tag_id),
        FOREIGN KEY (note_id) REFERENCES notes(id) ON DELETE CASCADE,
        FOREIGN KEY (tag_id) REFERENCES tags(id) ON DELETE CASCADE
      )
    ''');
    
    // Index สำหรับ performance
    await db.execute('''
      CREATE INDEX idx_notes_created_at ON notes(created_at DESC)
    ''');
    
    await db.execute('''
      CREATE INDEX idx_notes_is_pinned ON notes(is_pinned DESC)
    ''');
  }
  
  Future<void> _upgradeTables(Database db, int oldVersion, int newVersion) async {
    if (oldVersion < 2) {
      // Migration สำหรับ version 2
      await db.execute('ALTER TABLE notes ADD COLUMN category TEXT');
    }
    if (oldVersion < 3) {
      // Migration สำหรับ version 3
      await db.execute('''
        CREATE TABLE IF NOT EXISTS reminders (
          id INTEGER PRIMARY KEY AUTOINCREMENT,
          note_id INTEGER NOT NULL,
          remind_at TEXT NOT NULL,
          FOREIGN KEY (note_id) REFERENCES notes(id) ON DELETE CASCADE
        )
      ''');
    }
  }
  
  Future<void> close() async {
    final db = _database;
    if (db != null) {
      await db.close();
      _database = null;
    }
  }
}
```

---

## Database Migration

### Migration Strategy

```dart
class DatabaseMigration {
  static Future<void> migrate(
    Database db,
    int oldVersion,
    int newVersion,
  ) async {
    for (int version = oldVersion + 1; version <= newVersion; version++) {
      await _runMigration(db, version);
    }
  }
  
  static Future<void> _runMigration(Database db, int version) async {
    switch (version) {
      case 2:
        await _migration_v2(db);
        break;
      case 3:
        await _migration_v3(db);
        break;
      case 4:
        await _migration_v4(db);
        break;
    }
  }
  
  static Future<void> _migration_v2(Database db) async {
    print('Running migration to v2...');
    
    // เพิ่ม column ใหม่
    await db.execute('ALTER TABLE notes ADD COLUMN category TEXT DEFAULT "General"');
    
    // สร้าง table ใหม่
    await db.execute('''
      CREATE TABLE IF NOT EXISTS categories (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL UNIQUE,
        icon TEXT,
        created_at TEXT NOT NULL DEFAULT (datetime('now'))
      )
    ''');
    
    // Populate default data
    await db.insert('categories', {'name': 'General', 'icon': 'note'});
    await db.insert('categories', {'name': 'Work', 'icon': 'work'});
    await db.insert('categories', {'name': 'Personal', 'icon': 'person'});
    
    print('Migration to v2 completed');
  }
  
  static Future<void> _migration_v3(Database db) async {
    print('Running migration to v3...');
    
    // เพิ่ม full-text search
    await db.execute('''
      CREATE VIRTUAL TABLE IF NOT EXISTS notes_fts USING fts5(
        title, content, content='notes', content_rowid='id'
      )
    ''');
    
    // Populate FTS table จาก existing notes
    await db.execute('''
      INSERT INTO notes_fts(rowid, title, content)
      SELECT id, title, content FROM notes
    ''');
    
    print('Migration to v3 completed');
  }
  
  static Future<void> _migration_v4(Database db) async {
    print('Running migration to v4...');
    
    // Rename column (SQLite ไม่ support RENAME COLUMN ตรงๆ ต้องทำผ่าน temp table)
    await db.execute('''
      CREATE TABLE notes_new (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        title TEXT NOT NULL,
        content TEXT,
        note_color INTEGER DEFAULT 0xFF2196F3,  -- renamed from color
        is_pinned INTEGER DEFAULT 0,
        category TEXT DEFAULT "General",
        created_at TEXT NOT NULL,
        updated_at TEXT NOT NULL
      )
    ''');
    
    await db.execute('''
      INSERT INTO notes_new 
      SELECT id, title, content, color, is_pinned, category, created_at, updated_at
      FROM notes
    ''');
    
    await db.execute('DROP TABLE notes');
    await db.execute('ALTER TABLE notes_new RENAME TO notes');
    
    print('Migration to v4 completed');
  }
}
```

---

## CRUD Operations

### Note Model

```dart
// models/note.dart
class Note {
  final int? id;
  final String title;
  final String content;
  final int color;
  final bool isPinned;
  final String category;
  final DateTime createdAt;
  final DateTime updatedAt;
  final List<Tag> tags;
  
  const Note({
    this.id,
    required this.title,
    this.content = '',
    this.color = 0xFF2196F3,
    this.isPinned = false,
    this.category = 'General',
    DateTime? createdAt,
    DateTime? updatedAt,
    this.tags = const [],
  })  : createdAt = createdAt ?? const _Now(),
        updatedAt = updatedAt ?? const _Now();
  
  Note copyWith({
    int? id,
    String? title,
    String? content,
    int? color,
    bool? isPinned,
    String? category,
    DateTime? createdAt,
    DateTime? updatedAt,
    List<Tag>? tags,
  }) {
    return Note(
      id: id ?? this.id,
      title: title ?? this.title,
      content: content ?? this.content,
      color: color ?? this.color,
      isPinned: isPinned ?? this.isPinned,
      category: category ?? this.category,
      createdAt: createdAt ?? this.createdAt,
      updatedAt: updatedAt ?? this.updatedAt,
      tags: tags ?? this.tags,
    );
  }
  
  // Convert to Map for database
  Map<String, dynamic> toMap() {
    return {
      if (id != null) 'id': id,
      'title': title,
      'content': content,
      'color': color,
      'is_pinned': isPinned ? 1 : 0,
      'category': category,
      'created_at': createdAt.toIso8601String(),
      'updated_at': updatedAt.toIso8601String(),
    };
  }
  
  // Create from database Map
  factory Note.fromMap(Map<String, dynamic> map) {
    return Note(
      id: map['id'] as int?,
      title: map['title'] as String,
      content: map['content'] as String? ?? '',
      color: map['color'] as int? ?? 0xFF2196F3,
      isPinned: (map['is_pinned'] as int?) == 1,
      category: map['category'] as String? ?? 'General',
      createdAt: DateTime.parse(map['created_at'] as String),
      updatedAt: DateTime.parse(map['updated_at'] as String),
    );
  }
  
  @override
  String toString() => 'Note($id: $title)';
}

class Tag {
  final int? id;
  final String name;
  final int color;
  
  const Tag({this.id, required this.name, this.color = 0xFF4CAF50});
  
  Map<String, dynamic> toMap() => {
    if (id != null) 'id': id,
    'name': name,
    'color': color,
  };
  
  factory Tag.fromMap(Map<String, dynamic> map) => Tag(
    id: map['id'] as int?,
    name: map['name'] as String,
    color: map['color'] as int? ?? 0xFF4CAF50,
  );
}
```

### Notes Repository

```dart
// repositories/notes_repository.dart
class NotesRepository {
  final DatabaseHelper _dbHelper;
  
  NotesRepository(this._dbHelper);
  
  // ===== NOTES =====
  
  // Create
  Future<Note> insertNote(Note note) async {
    final db = await _dbHelper.database;
    
    final noteMap = note.toMap();
    noteMap.remove('id');  // Auto-increment
    
    final id = await db.insert(
      'notes',
      noteMap,
      conflictAlgorithm: ConflictAlgorithm.replace,
    );
    
    return note.copyWith(id: id);
  }
  
  // Read single
  Future<Note?> getNoteById(int id) async {
    final db = await _dbHelper.database;
    
    final maps = await db.query(
      'notes',
      where: 'id = ?',
      whereArgs: [id],
    );
    
    if (maps.isEmpty) return null;
    
    final note = Note.fromMap(maps.first);
    final tags = await getNoteTags(id);
    return note.copyWith(tags: tags);
  }
  
  // Read all
  Future<List<Note>> getAllNotes({
    String? category,
    bool? isPinned,
    String? orderBy,
  }) async {
    final db = await _dbHelper.database;
    
    String? where;
    List<dynamic>? whereArgs;
    
    final conditions = <String>[];
    final args = <dynamic>[];
    
    if (category != null) {
      conditions.add('category = ?');
      args.add(category);
    }
    
    if (isPinned != null) {
      conditions.add('is_pinned = ?');
      args.add(isPinned ? 1 : 0);
    }
    
    if (conditions.isNotEmpty) {
      where = conditions.join(' AND ');
      whereArgs = args;
    }
    
    final maps = await db.query(
      'notes',
      where: where,
      whereArgs: whereArgs,
      orderBy: orderBy ?? 'is_pinned DESC, updated_at DESC',
    );
    
    return Future.wait(maps.map((map) async {
      final note = Note.fromMap(map);
      final tags = await getNoteTags(note.id!);
      return note.copyWith(tags: tags);
    }));
  }
  
  // Update
  Future<int> updateNote(Note note) async {
    if (note.id == null) throw Exception('Cannot update note without id');
    
    final db = await _dbHelper.database;
    
    final updateMap = note.toMap();
    updateMap['updated_at'] = DateTime.now().toIso8601String();
    updateMap.remove('id');
    
    return await db.update(
      'notes',
      updateMap,
      where: 'id = ?',
      whereArgs: [note.id],
    );
  }
  
  // Delete
  Future<int> deleteNote(int id) async {
    final db = await _dbHelper.database;
    
    return await db.delete(
      'notes',
      where: 'id = ?',
      whereArgs: [id],
    );
  }
  
  // Delete multiple
  Future<int> deleteNotes(List<int> ids) async {
    final db = await _dbHelper.database;
    
    final placeholders = List.filled(ids.length, '?').join(', ');
    return await db.delete(
      'notes',
      where: 'id IN ($placeholders)',
      whereArgs: ids,
    );
  }
  
  // Toggle pin
  Future<void> togglePin(int id) async {
    final db = await _dbHelper.database;
    
    await db.rawUpdate('''
      UPDATE notes 
      SET is_pinned = CASE WHEN is_pinned = 1 THEN 0 ELSE 1 END,
          updated_at = ?
      WHERE id = ?
    ''', [DateTime.now().toIso8601String(), id]);
  }
  
  // ===== TAGS =====
  
  Future<int> insertTag(Tag tag) async {
    final db = await _dbHelper.database;
    return await db.insert(
      'tags',
      tag.toMap(),
      conflictAlgorithm: ConflictAlgorithm.ignore,
    );
  }
  
  Future<List<Tag>> getAllTags() async {
    final db = await _dbHelper.database;
    final maps = await db.query('tags', orderBy: 'name ASC');
    return maps.map(Tag.fromMap).toList();
  }
  
  Future<List<Tag>> getNoteTags(int noteId) async {
    final db = await _dbHelper.database;
    
    final maps = await db.rawQuery('''
      SELECT tags.* FROM tags
      INNER JOIN note_tags ON tags.id = note_tags.tag_id
      WHERE note_tags.note_id = ?
    ''', [noteId]);
    
    return maps.map(Tag.fromMap).toList();
  }
  
  Future<void> setNoteTags(int noteId, List<Tag> tags) async {
    final db = await _dbHelper.database;
    
    // Delete existing tags for this note
    await db.delete('note_tags', where: 'note_id = ?', whereArgs: [noteId]);
    
    // Insert new tags
    for (final tag in tags) {
      int tagId = tag.id ?? 0;
      if (tagId == 0) {
        tagId = await insertTag(tag);
      }
      await db.insert('note_tags', {
        'note_id': noteId,
        'tag_id': tagId,
      });
    }
  }
}
```

---

## Transactions

### Transaction Examples

```dart
class NotesTransactionRepository extends NotesRepository {
  NotesTransactionRepository(super.dbHelper);
  
  // Transaction: Insert note with tags atomically
  Future<Note> insertNoteWithTags(Note note, List<Tag> tags) async {
    final db = await _dbHelper.database;
    
    return await db.transaction((txn) async {
      // Insert note
      final noteMap = note.toMap()..remove('id');
      final noteId = await txn.insert('notes', noteMap);
      
      // Insert/find tags and create relationships
      for (final tag in tags) {
        // Insert tag if not exists
        int tagId;
        if (tag.id != null) {
          tagId = tag.id!;
        } else {
          // Try to find existing tag
          final existing = await txn.query(
            'tags',
            where: 'name = ?',
            whereArgs: [tag.name],
          );
          
          if (existing.isNotEmpty) {
            tagId = existing.first['id'] as int;
          } else {
            tagId = await txn.insert('tags', tag.toMap()..remove('id'));
          }
        }
        
        // Create note-tag relationship
        await txn.insert('note_tags', {
          'note_id': noteId,
          'tag_id': tagId,
        });
      }
      
      return note.copyWith(id: noteId, tags: tags);
    });
  }
  
  // Transaction: Move notes to another category
  Future<void> moveNotesToCategory(
    List<int> noteIds,
    String newCategory,
  ) async {
    final db = await _dbHelper.database;
    
    await db.transaction((txn) async {
      final now = DateTime.now().toIso8601String();
      
      for (final noteId in noteIds) {
        await txn.update(
          'notes',
          {'category': newCategory, 'updated_at': now},
          where: 'id = ?',
          whereArgs: [noteId],
        );
      }
    });
  }
  
  // Transaction: Duplicate note
  Future<Note> duplicateNote(int noteId) async {
    final db = await _dbHelper.database;
    
    return await db.transaction((txn) async {
      // Get original note
      final original = await txn.query(
        'notes',
        where: 'id = ?',
        whereArgs: [noteId],
      );
      
      if (original.isEmpty) throw Exception('Note not found');
      
      // Create copy
      final copyMap = Map<String, dynamic>.from(original.first);
      copyMap.remove('id');
      copyMap['title'] = 'Copy of ${copyMap['title']}';
      copyMap['created_at'] = DateTime.now().toIso8601String();
      copyMap['updated_at'] = DateTime.now().toIso8601String();
      
      final newId = await txn.insert('notes', copyMap);
      
      // Copy tags
      final tags = await txn.query(
        'note_tags',
        where: 'note_id = ?',
        whereArgs: [noteId],
      );
      
      for (final tag in tags) {
        await txn.insert('note_tags', {
          'note_id': newId,
          'tag_id': tag['tag_id'],
        });
      }
      
      return Note.fromMap({...copyMap, 'id': newId});
    });
  }
}
```

---

## Complex Queries

### Search and Filter

```dart
class NotesSearchRepository {
  final DatabaseHelper _dbHelper;
  
  NotesSearchRepository(this._dbHelper);
  
  // Full-text search
  Future<List<Note>> searchNotes(String query) async {
    if (query.trim().isEmpty) return [];
    
    final db = await _dbHelper.database;
    
    final maps = await db.rawQuery('''
      SELECT DISTINCT notes.*
      FROM notes
      LEFT JOIN note_tags ON notes.id = note_tags.note_id
      LEFT JOIN tags ON note_tags.tag_id = tags.id
      WHERE 
        notes.title LIKE ? OR
        notes.content LIKE ? OR
        tags.name LIKE ?
      ORDER BY notes.is_pinned DESC, notes.updated_at DESC
    ''', ['%$query%', '%$query%', '%$query%']);
    
    return maps.map(Note.fromMap).toList();
  }
  
  // Filter with multiple criteria
  Future<List<Note>> filterNotes({
    String? category,
    List<int>? tagIds,
    bool? isPinned,
    DateTime? startDate,
    DateTime? endDate,
    String? orderBy,
    int? limit,
    int? offset,
  }) async {
    final db = await _dbHelper.database;
    
    final conditions = <String>[];
    final args = <dynamic>[];
    
    if (category != null) {
      conditions.add('notes.category = ?');
      args.add(category);
    }
    
    if (isPinned != null) {
      conditions.add('notes.is_pinned = ?');
      args.add(isPinned ? 1 : 0);
    }
    
    if (startDate != null) {
      conditions.add('notes.created_at >= ?');
      args.add(startDate.toIso8601String());
    }
    
    if (endDate != null) {
      conditions.add('notes.created_at <= ?');
      args.add(endDate.toIso8601String());
    }
    
    String sql = 'SELECT DISTINCT notes.* FROM notes';
    
    if (tagIds != null && tagIds.isNotEmpty) {
      sql += ' INNER JOIN note_tags ON notes.id = note_tags.note_id';
      final placeholders = List.filled(tagIds.length, '?').join(', ');
      conditions.add('note_tags.tag_id IN ($placeholders)');
      args.addAll(tagIds);
    }
    
    if (conditions.isNotEmpty) {
      sql += ' WHERE ${conditions.join(' AND ')}';
    }
    
    sql += ' ORDER BY ${orderBy ?? "is_pinned DESC, updated_at DESC"}';
    
    if (limit != null) {
      sql += ' LIMIT ?';
      args.add(limit);
    }
    
    if (offset != null) {
      sql += ' OFFSET ?';
      args.add(offset);
    }
    
    final maps = await db.rawQuery(sql, args);
    return maps.map(Note.fromMap).toList();
  }
  
  // Aggregate queries
  Future<Map<String, dynamic>> getStatistics() async {
    final db = await _dbHelper.database;
    
    final total = Sqflite.firstIntValue(
      await db.rawQuery('SELECT COUNT(*) FROM notes'),
    );
    
    final pinned = Sqflite.firstIntValue(
      await db.rawQuery('SELECT COUNT(*) FROM notes WHERE is_pinned = 1'),
    );
    
    final byCategory = await db.rawQuery('''
      SELECT category, COUNT(*) as count
      FROM notes
      GROUP BY category
      ORDER BY count DESC
    ''');
    
    final tagCloud = await db.rawQuery('''
      SELECT tags.name, COUNT(note_tags.note_id) as usage_count
      FROM tags
      LEFT JOIN note_tags ON tags.id = note_tags.tag_id
      GROUP BY tags.id
      ORDER BY usage_count DESC
      LIMIT 10
    ''');
    
    return {
      'total': total ?? 0,
      'pinned': pinned ?? 0,
      'by_category': byCategory,
      'tag_cloud': tagCloud,
    };
  }
}
```

---

## Workshop: Notes App with SQLite

### Main App Structure

```dart
// main.dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  final dbHelper = DatabaseHelper();
  final repository = NotesRepository(dbHelper);
  
  runApp(
    ChangeNotifierProvider(
      create: (_) => NotesViewModel(repository),
      child: const NotesApp(),
    ),
  );
}

class NotesApp extends StatelessWidget {
  const NotesApp({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Notes',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.amber),
        useMaterial3: true,
      ),
      home: const NotesListPage(),
    );
  }
}

// ViewModel
class NotesViewModel extends ChangeNotifier {
  final NotesRepository _repository;
  
  List<Note> _notes = [];
  bool _isLoading = false;
  String _error = '';
  String _searchQuery = '';
  String? _selectedCategory;
  
  List<Note> get notes => _notes;
  bool get isLoading => _isLoading;
  String get error => _error;
  String get searchQuery => _searchQuery;
  
  NotesViewModel(this._repository) {
    loadNotes();
  }
  
  Future<void> loadNotes() async {
    _isLoading = true;
    _error = '';
    notifyListeners();
    
    try {
      if (_searchQuery.isNotEmpty) {
        // จำลองการค้นหา
        final allNotes = await _repository.getAllNotes(
          category: _selectedCategory,
        );
        _notes = allNotes
            .where((n) =>
                n.title.toLowerCase().contains(_searchQuery.toLowerCase()) ||
                n.content.toLowerCase().contains(_searchQuery.toLowerCase()))
            .toList();
      } else {
        _notes = await _repository.getAllNotes(
          category: _selectedCategory,
        );
      }
    } catch (e) {
      _error = e.toString();
    }
    
    _isLoading = false;
    notifyListeners();
  }
  
  void setSearchQuery(String query) {
    _searchQuery = query;
    loadNotes();
  }
  
  void setCategory(String? category) {
    _selectedCategory = category;
    loadNotes();
  }
  
  Future<void> addNote(Note note) async {
    await _repository.insertNote(note);
    await loadNotes();
  }
  
  Future<void> updateNote(Note note) async {
    await _repository.updateNote(note);
    await loadNotes();
  }
  
  Future<void> deleteNote(int id) async {
    await _repository.deleteNote(id);
    await loadNotes();
  }
  
  Future<void> togglePin(int id) async {
    await _repository.togglePin(id);
    await loadNotes();
  }
}
```

### Notes List Page

```dart
// pages/notes_list_page.dart
class NotesListPage extends StatefulWidget {
  const NotesListPage({Key? key}) : super(key: key);
  
  @override
  State<NotesListPage> createState() => _NotesListPageState();
}

class _NotesListPageState extends State<NotesListPage> {
  final _searchController = TextEditingController();
  bool _isSearching = false;
  
  @override
  void dispose() {
    _searchController.dispose();
    super.dispose();
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: _isSearching
            ? TextField(
                controller: _searchController,
                autofocus: true,
                decoration: const InputDecoration(
                  hintText: 'Search notes...',
                  border: InputBorder.none,
                ),
                onChanged: (query) => context.read<NotesViewModel>().setSearchQuery(query),
              )
            : const Text('Notes'),
        actions: [
          IconButton(
            onPressed: () {
              setState(() {
                _isSearching = !_isSearching;
                if (!_isSearching) {
                  _searchController.clear();
                  context.read<NotesViewModel>().setSearchQuery('');
                }
              });
            },
            icon: Icon(_isSearching ? Icons.close : Icons.search),
          ),
        ],
      ),
      body: Consumer<NotesViewModel>(
        builder: (context, viewModel, _) {
          if (viewModel.isLoading) {
            return const Center(child: CircularProgressIndicator());
          }
          
          if (viewModel.error.isNotEmpty) {
            return Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  Text('Error: ${viewModel.error}'),
                  ElevatedButton(
                    onPressed: viewModel.loadNotes,
                    child: const Text('Retry'),
                  ),
                ],
              ),
            );
          }
          
          if (viewModel.notes.isEmpty) {
            return Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  const Icon(Icons.note_add, size: 80, color: Colors.grey),
                  const SizedBox(height: 16),
                  Text(
                    viewModel.searchQuery.isNotEmpty
                        ? 'No notes found for "${viewModel.searchQuery}"'
                        : 'No notes yet. Tap + to add one!',
                    style: const TextStyle(color: Colors.grey),
                  ),
                ],
              ),
            );
          }
          
          return RefreshIndicator(
            onRefresh: viewModel.loadNotes,
            child: MasonryGridView.count(
              crossAxisCount: 2,
              mainAxisSpacing: 8,
              crossAxisSpacing: 8,
              padding: const EdgeInsets.all(16),
              itemCount: viewModel.notes.length,
              itemBuilder: (context, index) {
                return NoteCard(
                  note: viewModel.notes[index],
                  onTap: () => _navigateToNote(viewModel.notes[index]),
                  onDelete: () => viewModel.deleteNote(viewModel.notes[index].id!),
                  onTogglePin: () => viewModel.togglePin(viewModel.notes[index].id!),
                );
              },
            ),
          );
        },
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _navigateToNote(null),
        child: const Icon(Icons.add),
      ),
    );
  }
  
  void _navigateToNote(Note? note) async {
    final result = await Navigator.push<Note>(
      context,
      MaterialPageRoute(
        builder: (_) => NoteEditPage(note: note),
      ),
    );
    
    if (result != null && mounted) {
      final viewModel = context.read<NotesViewModel>();
      if (result.id == null) {
        await viewModel.addNote(result);
      } else {
        await viewModel.updateNote(result);
      }
    }
  }
}

class NoteCard extends StatelessWidget {
  final Note note;
  final VoidCallback onTap;
  final VoidCallback onDelete;
  final VoidCallback onTogglePin;
  
  const NoteCard({
    required this.note,
    required this.onTap,
    required this.onDelete,
    required this.onTogglePin,
  });
  
  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: onTap,
      child: Card(
        color: Color(note.color).withOpacity(0.2),
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(12),
          side: BorderSide(color: Color(note.color).withOpacity(0.3)),
        ),
        child: Padding(
          padding: const EdgeInsets.all(12),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Row(
                children: [
                  Expanded(
                    child: Text(
                      note.title,
                      style: const TextStyle(
                        fontWeight: FontWeight.bold,
                        fontSize: 15,
                      ),
                      maxLines: 2,
                      overflow: TextOverflow.ellipsis,
                    ),
                  ),
                  if (note.isPinned)
                    const Icon(Icons.push_pin, size: 14, color: Colors.amber),
                ],
              ),
              if (note.content.isNotEmpty) ...[
                const SizedBox(height: 4),
                Text(
                  note.content,
                  maxLines: 4,
                  overflow: TextOverflow.ellipsis,
                  style: TextStyle(
                    color: Colors.grey[700],
                    fontSize: 13,
                  ),
                ),
              ],
              if (note.tags.isNotEmpty) ...[
                const SizedBox(height: 8),
                Wrap(
                  spacing: 4,
                  children: note.tags
                      .take(3)
                      .map((tag) => Container(
                            padding: const EdgeInsets.symmetric(
                              horizontal: 6,
                              vertical: 2,
                            ),
                            decoration: BoxDecoration(
                              color: Color(tag.color).withOpacity(0.3),
                              borderRadius: BorderRadius.circular(8),
                            ),
                            child: Text(
                              tag.name,
                              style: const TextStyle(fontSize: 10),
                            ),
                          ))
                      .toList(),
                ),
              ],
              const SizedBox(height: 8),
              Row(
                children: [
                  Text(
                    _formatDate(note.updatedAt),
                    style: const TextStyle(fontSize: 11, color: Colors.grey),
                  ),
                  const Spacer(),
                  InkWell(
                    onTap: onTogglePin,
                    child: Icon(
                      note.isPinned ? Icons.push_pin : Icons.push_pin_outlined,
                      size: 16,
                      color: note.isPinned ? Colors.amber : Colors.grey,
                    ),
                  ),
                  const SizedBox(width: 8),
                  InkWell(
                    onTap: () => _confirmDelete(context),
                    child: const Icon(Icons.delete_outline, size: 16, color: Colors.grey),
                  ),
                ],
              ),
            ],
          ),
        ),
      ),
    );
  }
  
  String _formatDate(DateTime date) {
    final now = DateTime.now();
    final diff = now.difference(date);
    
    if (diff.inMinutes < 1) return 'Just now';
    if (diff.inHours < 1) return '${diff.inMinutes}m ago';
    if (diff.inDays < 1) return '${diff.inHours}h ago';
    if (diff.inDays < 7) return '${diff.inDays}d ago';
    
    return '${date.day}/${date.month}/${date.year}';
  }
  
  void _confirmDelete(BuildContext context) {
    showDialog(
      context: context,
      builder: (ctx) => AlertDialog(
        title: const Text('Delete Note'),
        content: Text('Delete "${note.title}"?'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(ctx),
            child: const Text('Cancel'),
          ),
          ElevatedButton(
            onPressed: () {
              onDelete();
              Navigator.pop(ctx);
            },
            style: ElevatedButton.styleFrom(backgroundColor: Colors.red),
            child: const Text('Delete', style: TextStyle(color: Colors.white)),
          ),
        ],
      ),
    );
  }
}
```

### Note Edit Page

```dart
// pages/note_edit_page.dart
class NoteEditPage extends StatefulWidget {
  final Note? note;
  
  const NoteEditPage({this.note, Key? key}) : super(key: key);
  
  @override
  State<NoteEditPage> createState() => _NoteEditPageState();
}

class _NoteEditPageState extends State<NoteEditPage> {
  late final TextEditingController _titleController;
  late final TextEditingController _contentController;
  int _selectedColor = 0xFF2196F3;
  bool _isPinned = false;
  
  final List<int> _colorOptions = [
    0xFF2196F3, // Blue
    0xFFFF5722, // Deep Orange
    0xFF4CAF50, // Green
    0xFF9C27B0, // Purple
    0xFFFF9800, // Orange
    0xFF009688, // Teal
    0xFFE91E63, // Pink
    0xFF607D8B, // Blue Grey
  ];
  
  @override
  void initState() {
    super.initState();
    _titleController = TextEditingController(text: widget.note?.title ?? '');
    _contentController = TextEditingController(text: widget.note?.content ?? '');
    _selectedColor = widget.note?.color ?? 0xFF2196F3;
    _isPinned = widget.note?.isPinned ?? false;
  }
  
  @override
  void dispose() {
    _titleController.dispose();
    _contentController.dispose();
    super.dispose();
  }
  
  void _saveNote() {
    if (_titleController.text.trim().isEmpty) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Title cannot be empty')),
      );
      return;
    }
    
    final note = Note(
      id: widget.note?.id,
      title: _titleController.text.trim(),
      content: _contentController.text.trim(),
      color: _selectedColor,
      isPinned: _isPinned,
      category: widget.note?.category ?? 'General',
      createdAt: widget.note?.createdAt ?? DateTime.now(),
      updatedAt: DateTime.now(),
    );
    
    Navigator.pop(context, note);
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Color(_selectedColor).withOpacity(0.1),
      appBar: AppBar(
        backgroundColor: Colors.transparent,
        leading: IconButton(
          onPressed: () => Navigator.pop(context),
          icon: const Icon(Icons.arrow_back),
        ),
        actions: [
          IconButton(
            onPressed: () => setState(() => _isPinned = !_isPinned),
            icon: Icon(
              _isPinned ? Icons.push_pin : Icons.push_pin_outlined,
              color: _isPinned ? Colors.amber : null,
            ),
          ),
          TextButton(
            onPressed: _saveNote,
            child: const Text('Save'),
          ),
        ],
      ),
      body: Column(
        children: [
          // Color picker
          SingleChildScrollView(
            scrollDirection: Axis.horizontal,
            padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
            child: Row(
              children: _colorOptions.map((color) {
                return GestureDetector(
                  onTap: () => setState(() => _selectedColor = color),
                  child: Container(
                    width: 32,
                    height: 32,
                    margin: const EdgeInsets.only(right: 8),
                    decoration: BoxDecoration(
                      color: Color(color),
                      shape: BoxShape.circle,
                      border: _selectedColor == color
                          ? Border.all(color: Colors.white, width: 3)
                          : null,
                      boxShadow: _selectedColor == color
                          ? [BoxShadow(
                              color: Color(color).withOpacity(0.5),
                              blurRadius: 8,
                            )]
                          : null,
                    ),
                  ),
                );
              }).toList(),
            ),
          ),
          
          // Title
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 16),
            child: TextField(
              controller: _titleController,
              style: const TextStyle(
                fontSize: 22,
                fontWeight: FontWeight.bold,
              ),
              decoration: const InputDecoration(
                hintText: 'Title',
                border: InputBorder.none,
                hintStyle: TextStyle(
                  fontWeight: FontWeight.bold,
                  color: Colors.grey,
                ),
              ),
              textCapitalization: TextCapitalization.sentences,
            ),
          ),
          
          const Divider(indent: 16, endIndent: 16),
          
          // Content
          Expanded(
            child: Padding(
              padding: const EdgeInsets.symmetric(horizontal: 16),
              child: TextField(
                controller: _contentController,
                maxLines: null,
                expands: true,
                textAlignVertical: TextAlignVertical.top,
                decoration: const InputDecoration(
                  hintText: 'Write your note here...',
                  border: InputBorder.none,
                ),
              ),
            ),
          ),
          
          // Bottom info
          Padding(
            padding: const EdgeInsets.all(16),
            child: Text(
              widget.note != null
                  ? 'Edited ${_formatDate(DateTime.now())}'
                  : 'New note',
              style: const TextStyle(color: Colors.grey, fontSize: 12),
            ),
          ),
        ],
      ),
    );
  }
  
  String _formatDate(DateTime date) {
    return '${date.day}/${date.month}/${date.year} ${date.hour}:${date.minute.toString().padLeft(2, '0')}';
  }
}
```

---

## สรุป SQLite

### เมื่อไหร่ใช้ SQLite
- ข้อมูล relational (users, posts, tags)
- ต้องการ queries ซับซ้อน (JOIN, GROUP BY)
- ข้อมูลจำนวนมาก
- ต้องการ ACID transactions

### Best Practices
```dart
// 1. ใช้ parameterized queries ป้องกัน SQL injection
await db.query('users', where: 'id = ?', whereArgs: [userId]);

// 2. ใช้ transaction สำหรับ multiple operations
await db.transaction((txn) async {
  await txn.insert('table1', data1);
  await txn.insert('table2', data2);
});

// 3. สร้าง indexes สำหรับ columns ที่ query บ่อย
CREATE INDEX idx_notes_created ON notes(created_at DESC);

// 4. Close database เมื่อเสร็จ
await db.close();

// 5. Migration ด้วย version system
openDatabase(path, version: 2, onUpgrade: migrate);
```
