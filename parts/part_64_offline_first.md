# Part 64: Offline-First Architecture

## Offline-First คืออะไร?

Offline-First คือ architecture ที่ออกแบบให้แอปทำงานได้สมบูรณ์แม้ไม่มีอินเทอร์เน็ต โดย:
- อ่านข้อมูลจาก local cache ก่อนเสมอ
- Sync กับ server เมื่อมีการเชื่อมต่อ
- จัดการ conflicts อย่างชาญฉลาด

```yaml
# pubspec.yaml
dependencies:
  connectivity_plus: ^6.0.0
  hive_flutter: ^1.1.0
  drift: ^2.15.0
  sqflite: ^2.3.0
  
dev_dependencies:
  drift_dev: ^2.15.0
  build_runner: ^2.4.0
```

---

## 1. Network Connectivity Detection

### ConnectivityService

```dart
// services/connectivity_service.dart
import 'package:connectivity_plus/connectivity_plus.dart';

class ConnectivityService {
  static final ConnectivityService _instance = ConnectivityService._();
  factory ConnectivityService() => _instance;
  ConnectivityService._();

  final _connectivity = Connectivity();
  
  // Stream ของสถานะการเชื่อมต่อ
  Stream<bool> get onConnectivityChanged =>
      _connectivity.onConnectivityChanged.map(
        (result) => _isConnected(result),
      );

  // ตรวจสอบสถานะปัจจุบัน
  Future<bool> isConnected() async {
    final result = await _connectivity.checkConnectivity();
    return _isConnected(result);
  }

  bool _isConnected(List<ConnectivityResult> results) {
    return results.any((r) =>
        r == ConnectivityResult.mobile ||
        r == ConnectivityResult.wifi ||
        r == ConnectivityResult.ethernet);
  }

  // รอจนกว่าจะมีการเชื่อมต่อ
  Future<void> waitForConnection() async {
    if (await isConnected()) return;

    await onConnectivityChanged
        .firstWhere((isConnected) => isConnected);
  }
}

// Provider สำหรับ connectivity state
class ConnectivityProvider extends ChangeNotifier {
  bool _isOnline = true;
  StreamSubscription? _subscription;

  bool get isOnline => _isOnline;
  bool get isOffline => !_isOnline;

  ConnectivityProvider() {
    _initialize();
  }

  Future<void> _initialize() async {
    _isOnline = await ConnectivityService().isConnected();
    notifyListeners();

    _subscription = ConnectivityService().onConnectivityChanged.listen(
      (isConnected) {
        if (_isOnline != isConnected) {
          _isOnline = isConnected;
          notifyListeners();
        }
      },
    );
  }

  @override
  void dispose() {
    _subscription?.cancel();
    super.dispose();
  }
}

// Offline Banner Widget
class OfflineBanner extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Consumer<ConnectivityProvider>(
      builder: (context, connectivity, _) {
        if (connectivity.isOnline) return SizedBox.shrink();

        return Container(
          width: double.infinity,
          color: Colors.red,
          padding: EdgeInsets.symmetric(vertical: 8),
          child: Row(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              Icon(Icons.wifi_off, color: Colors.white, size: 16),
              SizedBox(width: 8),
              Text(
                'ไม่มีการเชื่อมต่ออินเทอร์เน็ต',
                style: TextStyle(color: Colors.white),
              ),
            ],
          ),
        );
      },
    );
  }
}
```

---

## 2. Local Caching Strategy

### Hive สำหรับ Simple Caching

```dart
// models/note.dart
import 'package:hive/hive.dart';

part 'note.g.dart';

@HiveType(typeId: 0)
class Note extends HiveObject {
  @HiveField(0)
  late String id;

  @HiveField(1)
  late String title;

  @HiveField(2)
  late String content;

  @HiveField(3)
  late DateTime createdAt;

  @HiveField(4)
  late DateTime updatedAt;

  @HiveField(5)
  late bool isSynced; // ตรวจสอบว่า sync กับ server แล้วหรือยัง

  @HiveField(6)
  late bool isDeleted; // soft delete

  Note({
    required this.id,
    required this.title,
    required this.content,
    required this.createdAt,
    required this.updatedAt,
    this.isSynced = false,
    this.isDeleted = false,
  });
}

// main.dart - Initialize Hive
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Hive.initFlutter();
  
  // Register adapters
  Hive.registerAdapter(NoteAdapter());
  
  // Open boxes
  await Hive.openBox<Note>('notes');
  await Hive.openBox('settings');
  
  runApp(MyApp());
}
```

### Local Repository Pattern

```dart
// repositories/notes_repository.dart
class NotesRepository {
  final _localSource = LocalNotesDataSource();
  final _remoteSource = RemoteNotesDataSource();
  final _syncQueue = SyncQueue();
  final _connectivity = ConnectivityService();

  // อ่านข้อมูล: Local first, sync ถ้าออนไลน์
  Future<List<Note>> getNotes() async {
    // 1. คืน local data ก่อนเสมอ (fast)
    final localNotes = await _localSource.getNotes();
    
    // 2. ถ้าออนไลน์ ดึงข้อมูลใหม่ใน background
    if (await _connectivity.isConnected()) {
      _fetchAndUpdateInBackground();
    }
    
    return localNotes;
  }

  // Stream version: อัปเดต UI อัตโนมัติ
  Stream<List<Note>> watchNotes() {
    return _localSource.watchNotes();
  }

  Future<void> _fetchAndUpdateInBackground() async {
    try {
      final remoteNotes = await _remoteSource.getNotes();
      await _localSource.updateFromRemote(remoteNotes);
    } catch (e) {
      // ไม่แสดง error ถ้า background sync ล้มเหลว
      debugPrint('Background sync failed: $e');
    }
  }

  // เพิ่ม Note: บันทึก local ทันที, sync เมื่อออนไลน์
  Future<Note> addNote({
    required String title,
    required String content,
  }) async {
    final note = Note(
      id: _generateId(),
      title: title,
      content: content,
      createdAt: DateTime.now(),
      updatedAt: DateTime.now(),
      isSynced: false,
    );

    // บันทึก local ทันที
    await _localSource.addNote(note);

    // Sync หรือเพิ่มใน queue
    if (await _connectivity.isConnected()) {
      await _syncNote(note);
    } else {
      await _syncQueue.addOperation(
        SyncOperation(
          type: OperationType.create,
          resourceId: note.id,
          data: note.toJson(),
        ),
      );
    }

    return note;
  }

  Future<void> _syncNote(Note note) async {
    try {
      final syncedNote = await _remoteSource.createNote(note);
      await _localSource.updateNote(
        syncedNote.copyWith(isSynced: true),
      );
    } catch (e) {
      // Sync ล้มเหลว - เพิ่มใน queue
      await _syncQueue.addOperation(
        SyncOperation(
          type: OperationType.create,
          resourceId: note.id,
          data: note.toJson(),
        ),
      );
    }
  }

  String _generateId() {
    return DateTime.now().millisecondsSinceEpoch.toString();
  }
}
```

---

## 3. Sync When Online

### SyncManager

```dart
// services/sync_manager.dart
class SyncManager {
  final _queue = SyncQueue();
  final _repository = NotesRepository();
  final _connectivity = ConnectivityService();
  
  bool _isSyncing = false;
  StreamSubscription? _connectivitySubscription;

  void initialize() {
    // Start sync khi có kết nối
    _connectivitySubscription = _connectivity.onConnectivityChanged.listen(
      (isConnected) {
        if (isConnected && !_isSyncing) {
          syncAll();
        }
      },
    );
  }

  Future<void> syncAll() async {
    if (_isSyncing) return;
    if (!await _connectivity.isConnected()) return;

    _isSyncing = true;

    try {
      // 1. Process pending operations
      await _processPendingOperations();

      // 2. Fetch latest data from server
      await _fetchLatestData();

      debugPrint('Sync completed successfully');
    } catch (e) {
      debugPrint('Sync failed: $e');
    } finally {
      _isSyncing = false;
    }
  }

  Future<void> _processPendingOperations() async {
    final operations = await _queue.getPendingOperations();

    for (final operation in operations) {
      try {
        await _processOperation(operation);
        await _queue.markAsCompleted(operation.id);
      } catch (e) {
        // เพิ่ม retry count
        await _queue.incrementRetryCount(operation.id);

        // ถ้า retry เกิน 5 ครั้ง ให้ mark เป็น failed
        if (operation.retryCount >= 5) {
          await _queue.markAsFailed(operation.id, e.toString());
        }
      }
    }
  }

  Future<void> _processOperation(SyncOperation operation) async {
    switch (operation.type) {
      case OperationType.create:
        await RemoteNotesDataSource()
            .createNote(Note.fromJson(operation.data));
        break;
      case OperationType.update:
        await RemoteNotesDataSource()
            .updateNote(Note.fromJson(operation.data));
        break;
      case OperationType.delete:
        await RemoteNotesDataSource().deleteNote(operation.resourceId);
        break;
    }
  }

  Future<void> _fetchLatestData() async {
    final lastSync = await _getLastSyncTime();
    final remoteNotes = await RemoteNotesDataSource()
        .getNotesSince(lastSync);

    for (final note in remoteNotes) {
      await LocalNotesDataSource().upsertNote(note);
    }

    await _updateLastSyncTime();
  }

  Future<DateTime?> _getLastSyncTime() async {
    final box = Hive.box('settings');
    final timestamp = box.get('last_sync_time');
    return timestamp != null ? DateTime.parse(timestamp) : null;
  }

  Future<void> _updateLastSyncTime() async {
    final box = Hive.box('settings');
    await box.put('last_sync_time', DateTime.now().toIso8601String());
  }

  void dispose() {
    _connectivitySubscription?.cancel();
  }
}

// Sync Queue
class SyncQueue {
  Future<List<SyncOperation>> getPendingOperations() async {
    final box = Hive.box<SyncOperation>('sync_queue');
    return box.values
        .where((op) => op.status == OperationStatus.pending)
        .toList()
      ..sort((a, b) => a.createdAt.compareTo(b.createdAt));
  }

  Future<void> addOperation(SyncOperation operation) async {
    final box = Hive.box<SyncOperation>('sync_queue');
    await box.put(operation.id, operation);
  }

  Future<void> markAsCompleted(String id) async {
    final box = Hive.box<SyncOperation>('sync_queue');
    final op = box.get(id);
    if (op != null) {
      op.status = OperationStatus.completed;
      await op.save();
    }
  }

  Future<void> markAsFailed(String id, String error) async {
    final box = Hive.box<SyncOperation>('sync_queue');
    final op = box.get(id);
    if (op != null) {
      op.status = OperationStatus.failed;
      op.errorMessage = error;
      await op.save();
    }
  }

  Future<void> incrementRetryCount(String id) async {
    final box = Hive.box<SyncOperation>('sync_queue');
    final op = box.get(id);
    if (op != null) {
      op.retryCount++;
      await op.save();
    }
  }
}
```

---

## 4. Conflict Resolution

### Strategies สำหรับ Conflict Resolution

```dart
// conflict_resolution.dart

// Strategy 1: Last Write Wins
class LastWriteWinsStrategy {
  Note resolve(Note local, Note remote) {
    // ใช้เวอร์ชันล่าสุดตาม updatedAt
    if (local.updatedAt.isAfter(remote.updatedAt)) {
      return local;
    }
    return remote;
  }
}

// Strategy 2: Server Wins
class ServerWinsStrategy {
  Note resolve(Note local, Note remote) {
    return remote; // server เสมอ
  }
}

// Strategy 3: Client Wins
class ClientWinsStrategy {
  Note resolve(Note local, Note remote) {
    return local; // local เสมอ
  }
}

// Strategy 4: Smart Merge
class SmartMergeStrategy {
  Note resolve(Note local, Note remote, Note? base) {
    if (base == null) {
      // ไม่มี base version - ใช้ last write wins
      return LastWriteWinsStrategy().resolve(local, remote);
    }

    // Three-way merge
    final merged = Note(
      id: local.id,
      createdAt: local.createdAt,
      updatedAt: DateTime.now(),
      isSynced: false,
      title: _mergeField(
        local.title,
        remote.title,
        base.title,
      ),
      content: _mergeField(
        local.content,
        remote.content,
        base.content,
      ),
    );

    return merged;
  }

  String _mergeField(String local, String remote, String base) {
    // ถ้า local เปลี่ยนแต่ remote ไม่เปลี่ยน -> ใช้ local
    if (local != base && remote == base) return local;
    
    // ถ้า remote เปลี่ยนแต่ local ไม่เปลี่ยน -> ใช้ remote
    if (remote != base && local == base) return remote;
    
    // ทั้งคู่เปลี่ยน -> conflict - ใช้ local (หรือแสดง UI ให้ user เลือก)
    if (local != remote) return local; // หรือ throw ConflictException
    
    return local; // ทั้งคู่เหมือนกัน
  }
}

// Conflict Resolution Manager
class ConflictResolutionManager {
  final _strategy = SmartMergeStrategy();

  Future<Note> resolveConflict({
    required Note local,
    required Note remote,
    Note? base,
    ConflictStrategy strategy = ConflictStrategy.smartMerge,
  }) async {
    switch (strategy) {
      case ConflictStrategy.lastWriteWins:
        return LastWriteWinsStrategy().resolve(local, remote);
      case ConflictStrategy.serverWins:
        return ServerWinsStrategy().resolve(local, remote);
      case ConflictStrategy.clientWins:
        return ClientWinsStrategy().resolve(local, remote);
      case ConflictStrategy.smartMerge:
        return _strategy.resolve(local, remote, base);
      case ConflictStrategy.userDecide:
        return await _askUser(local, remote);
    }
  }

  Future<Note> _askUser(Note local, Note remote) async {
    // แสดง UI ให้ user เลือก
    final result = await showDialog<Note>(
      context: navigatorKey.currentContext!,
      barrierDismissible: false,
      builder: (context) => ConflictResolutionDialog(
        local: local,
        remote: remote,
      ),
    );
    return result ?? local;
  }
}

// UI สำหรับ user resolve conflict
class ConflictResolutionDialog extends StatelessWidget {
  final Note local;
  final Note remote;

  const ConflictResolutionDialog({required this.local, required this.remote});

  @override
  Widget build(BuildContext context) {
    return AlertDialog(
      title: Text('ข้อขัดแย้งในข้อมูล'),
      content: Column(
        mainAxisSize: MainAxisSize.min,
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text('พบการแก้ไขจากสองแหล่ง กรุณาเลือกเวอร์ชันที่ต้องการ'),
          SizedBox(height: 16),
          _VersionCard(
            label: 'เวอร์ชันของคุณ',
            note: local,
            onSelect: () => Navigator.pop(context, local),
          ),
          SizedBox(height: 8),
          _VersionCard(
            label: 'เวอร์ชันจาก server',
            note: remote,
            onSelect: () => Navigator.pop(context, remote),
          ),
        ],
      ),
    );
  }
}

class _VersionCard extends StatelessWidget {
  final String label;
  final Note note;
  final VoidCallback onSelect;

  const _VersionCard({
    required this.label,
    required this.note,
    required this.onSelect,
  });

  @override
  Widget build(BuildContext context) {
    return Card(
      child: InkWell(
        onTap: onSelect,
        child: Padding(
          padding: EdgeInsets.all(12),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Text(label, style: TextStyle(fontWeight: FontWeight.bold)),
              SizedBox(height: 4),
              Text(note.title, style: TextStyle(fontSize: 16)),
              SizedBox(height: 4),
              Text(
                note.content,
                maxLines: 3,
                overflow: TextOverflow.ellipsis,
                style: TextStyle(color: Colors.grey[600]),
              ),
              SizedBox(height: 4),
              Text(
                'แก้ไขเมื่อ: ${DateFormat('dd/MM/yyyy HH:mm').format(note.updatedAt)}',
                style: TextStyle(fontSize: 12, color: Colors.grey),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

---

## Workshop: Offline-Capable Notes App

```dart
// Complete Offline-First Notes App

// main.dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Hive.initFlutter();
  Hive.registerAdapter(NoteAdapter());
  Hive.registerAdapter(SyncOperationAdapter());
  await Future.wait([
    Hive.openBox<Note>('notes'),
    Hive.openBox<SyncOperation>('sync_queue'),
    Hive.openBox('settings'),
  ]);

  runApp(
    MultiProvider(
      providers: [
        ChangeNotifierProvider(create: (_) => ConnectivityProvider()),
        ChangeNotifierProvider(create: (_) => NotesProvider()),
      ],
      child: OfflineNotesApp(),
    ),
  );
}

// providers/notes_provider.dart
class NotesProvider extends ChangeNotifier {
  final _repository = NotesRepository();
  final _syncManager = SyncManager();

  List<Note> _notes = [];
  bool _isLoading = false;
  String? _error;
  bool _isSyncing = false;

  List<Note> get notes => _notes.where((n) => !n.isDeleted).toList();
  bool get isLoading => _isLoading;
  String? get error => _error;
  bool get isSyncing => _isSyncing;

  NotesProvider() {
    _initialize();
  }

  void _initialize() {
    _syncManager.initialize();
    _repository.watchNotes().listen((notes) {
      _notes = notes;
      notifyListeners();
    });
    loadNotes();
  }

  Future<void> loadNotes() async {
    _isLoading = true;
    _error = null;
    notifyListeners();

    try {
      _notes = await _repository.getNotes();
    } catch (e) {
      _error = 'ไม่สามารถโหลดข้อมูลได้';
    } finally {
      _isLoading = false;
      notifyListeners();
    }
  }

  Future<void> addNote(String title, String content) async {
    final note = await _repository.addNote(
      title: title,
      content: content,
    );
    _notes.insert(0, note);
    notifyListeners();
  }

  Future<void> updateNote(Note note) async {
    await _repository.updateNote(note);
    final index = _notes.indexWhere((n) => n.id == note.id);
    if (index != -1) {
      _notes[index] = note;
      notifyListeners();
    }
  }

  Future<void> deleteNote(String id) async {
    await _repository.deleteNote(id);
    _notes.removeWhere((n) => n.id == id);
    notifyListeners();
  }

  Future<void> syncNow() async {
    _isSyncing = true;
    notifyListeners();

    try {
      await _syncManager.syncAll();
      await loadNotes();
    } finally {
      _isSyncing = false;
      notifyListeners();
    }
  }

  @override
  void dispose() {
    _syncManager.dispose();
    super.dispose();
  }
}

// screens/notes_screen.dart
class NotesScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('บันทึกของฉัน'),
        actions: [
          Consumer2<ConnectivityProvider, NotesProvider>(
            builder: (context, connectivity, notes, _) {
              return IconButton(
                icon: notes.isSyncing
                    ? SizedBox(
                        width: 20,
                        height: 20,
                        child: CircularProgressIndicator(
                          color: Colors.white,
                          strokeWidth: 2,
                        ),
                      )
                    : Icon(
                        connectivity.isOnline
                            ? Icons.sync
                            : Icons.sync_disabled,
                      ),
                onPressed:
                    connectivity.isOnline && !notes.isSyncing
                        ? () => notes.syncNow()
                        : null,
              );
            },
          ),
        ],
      ),
      body: Column(
        children: [
          OfflineBanner(),
          Expanded(
            child: Consumer<NotesProvider>(
              builder: (context, provider, _) {
                if (provider.isLoading) {
                  return Center(child: CircularProgressIndicator());
                }

                if (provider.notes.isEmpty) {
                  return Center(
                    child: Column(
                      mainAxisAlignment: MainAxisAlignment.center,
                      children: [
                        Icon(Icons.note_add, size: 64, color: Colors.grey),
                        SizedBox(height: 16),
                        Text('ยังไม่มีบันทึก'),
                        Text('กดปุ่ม + เพื่อเพิ่มบันทึกใหม่'),
                      ],
                    ),
                  );
                }

                return RefreshIndicator(
                  onRefresh: () => provider.loadNotes(),
                  child: ListView.builder(
                    itemCount: provider.notes.length,
                    itemBuilder: (context, index) {
                      final note = provider.notes[index];
                      return _NoteCard(
                        note: note,
                        onTap: () => _openNote(context, note),
                        onDelete: () => provider.deleteNote(note.id),
                      );
                    },
                  ),
                );
              },
            ),
          ),
        ],
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _addNote(context),
        child: Icon(Icons.add),
      ),
    );
  }

  void _addNote(BuildContext context) {
    showModalBottomSheet(
      context: context,
      isScrollControlled: true,
      builder: (_) => _AddNoteSheet(),
    );
  }

  void _openNote(BuildContext context, Note note) {
    Navigator.push(
      context,
      MaterialPageRoute(
        builder: (_) => NoteDetailScreen(note: note),
      ),
    );
  }
}

class _NoteCard extends StatelessWidget {
  final Note note;
  final VoidCallback onTap;
  final VoidCallback onDelete;

  const _NoteCard({
    required this.note,
    required this.onTap,
    required this.onDelete,
  });

  @override
  Widget build(BuildContext context) {
    return Card(
      margin: EdgeInsets.symmetric(horizontal: 16, vertical: 4),
      child: ListTile(
        title: Text(note.title),
        subtitle: Text(
          note.content,
          maxLines: 2,
          overflow: TextOverflow.ellipsis,
        ),
        trailing: Row(
          mainAxisSize: MainAxisSize.min,
          children: [
            // Sync status indicator
            Icon(
              note.isSynced ? Icons.cloud_done : Icons.cloud_upload,
              color: note.isSynced ? Colors.green : Colors.orange,
              size: 16,
            ),
            IconButton(
              icon: Icon(Icons.delete, color: Colors.red),
              onPressed: onDelete,
            ),
          ],
        ),
        onTap: onTap,
      ),
    );
  }
}

class _AddNoteSheet extends StatefulWidget {
  @override
  _AddNoteSheetState createState() => _AddNoteSheetState();
}

class _AddNoteSheetState extends State<_AddNoteSheet> {
  final _titleController = TextEditingController();
  final _contentController = TextEditingController();

  @override
  void dispose() {
    _titleController.dispose();
    _contentController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: EdgeInsets.only(
        bottom: MediaQuery.of(context).viewInsets.bottom,
        left: 16,
        right: 16,
        top: 16,
      ),
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          TextField(
            controller: _titleController,
            decoration: InputDecoration(
              labelText: 'หัวข้อ',
              border: OutlineInputBorder(),
            ),
          ),
          SizedBox(height: 12),
          TextField(
            controller: _contentController,
            decoration: InputDecoration(
              labelText: 'เนื้อหา',
              border: OutlineInputBorder(),
            ),
            maxLines: 5,
          ),
          SizedBox(height: 16),
          ElevatedButton(
            onPressed: () {
              if (_titleController.text.isNotEmpty) {
                context.read<NotesProvider>().addNote(
                      _titleController.text,
                      _contentController.text,
                    );
                Navigator.pop(context);
              }
            },
            style: ElevatedButton.styleFrom(
              minimumSize: Size(double.infinity, 48),
            ),
            child: Text('บันทึก'),
          ),
          SizedBox(height: 16),
        ],
      ),
    );
  }
}
```

---

## สรุป

Offline-First Architecture ประกอบด้วย:

1. **Connectivity Detection** - ตรวจสอบ network status
2. **Local Cache** - Hive หรือ Drift สำหรับ local storage
3. **Repository Pattern** - abstraction layer ระหว่าง UI และ data
4. **Sync Queue** - เก็บ operations ที่รอ sync
5. **Conflict Resolution** - จัดการเมื่อข้อมูลขัดแย้ง
6. **Background Sync** - sync อัตโนมัติเมื่อออนไลน์
