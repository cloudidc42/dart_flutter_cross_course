# Part 69: Desktop Applications ด้วย Flutter

## Flutter Desktop Support

Flutter รองรับ Desktop platforms:
- **Windows** - Windows 10+
- **macOS** - macOS 10.14+
- **Linux** - Ubuntu 18.04+

```bash
# Enable desktop support
flutter config --enable-windows-desktop
flutter config --enable-macos-desktop
flutter config --enable-linux-desktop

# Create desktop app
flutter create --platforms=windows,macos,linux my_desktop_app

# Run on desktop
flutter run -d windows
flutter run -d macos
flutter run -d linux
```

---

## 1. Window Management

### window_manager Package

```yaml
# pubspec.yaml
dependencies:
  window_manager: ^0.3.8
  flutter_acrylic: ^1.1.3  # Windows acrylic effects
```

```dart
// main.dart
import 'package:window_manager/window_manager.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  if (isDesktop) {
    await windowManager.ensureInitialized();
    
    WindowOptions windowOptions = WindowOptions(
      size: Size(1200, 800),
      minimumSize: Size(800, 600),
      center: true,
      backgroundColor: Colors.transparent,
      skipTaskbar: false,
      titleBarStyle: TitleBarStyle.default,
      title: 'My Desktop App',
    );
    
    windowManager.waitUntilReadyToShow(windowOptions, () async {
      await windowManager.show();
      await windowManager.focus();
    });
  }
  
  runApp(MyDesktopApp());
}

bool get isDesktop =>
    Platform.isWindows || Platform.isMacOS || Platform.isLinux;
```

### Window Controls

```dart
// widgets/custom_title_bar.dart
class CustomTitleBar extends StatelessWidget with WindowListener {
  const CustomTitleBar();

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onPanStart: (details) {
        // อนุญาตให้ drag window
        windowManager.startDragging();
      },
      child: Container(
        height: 40,
        color: Theme.of(context).colorScheme.surface,
        child: Row(
          children: [
            // App icon
            Padding(
              padding: EdgeInsets.symmetric(horizontal: 12),
              child: Image.asset('assets/icon.png', width: 20, height: 20),
            ),
            
            // Title
            Text(
              'My App',
              style: TextStyle(fontSize: 14),
            ),
            
            Spacer(),
            
            // Window controls
            if (Platform.isWindows)
              _WindowsControls()
            else if (Platform.isMacOS)
              _MacOSControls(),
          ],
        ),
      ),
    );
  }
}

class _WindowsControls extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Row(
      children: [
        // Minimize
        _TitleBarButton(
          icon: Icons.minimize,
          onPressed: () => windowManager.minimize(),
        ),
        
        // Maximize/Restore
        FutureBuilder<bool>(
          future: windowManager.isMaximized(),
          builder: (context, snapshot) {
            return _TitleBarButton(
              icon: snapshot.data == true
                  ? Icons.filter_none
                  : Icons.crop_square,
              onPressed: () async {
                if (await windowManager.isMaximized()) {
                  windowManager.unmaximize();
                } else {
                  windowManager.maximize();
                }
              },
            );
          },
        ),
        
        // Close
        _TitleBarButton(
          icon: Icons.close,
          onPressed: () => windowManager.close(),
          hoverColor: Colors.red,
        ),
      ],
    );
  }
}

class _TitleBarButton extends StatefulWidget {
  final IconData icon;
  final VoidCallback onPressed;
  final Color? hoverColor;

  const _TitleBarButton({
    required this.icon,
    required this.onPressed,
    this.hoverColor,
  });

  @override
  _TitleBarButtonState createState() => _TitleBarButtonState();
}

class _TitleBarButtonState extends State<_TitleBarButton> {
  bool _isHovered = false;

  @override
  Widget build(BuildContext context) {
    return MouseRegion(
      onEnter: (_) => setState(() => _isHovered = true),
      onExit: (_) => setState(() => _isHovered = false),
      cursor: SystemMouseCursors.click,
      child: GestureDetector(
        onTap: widget.onPressed,
        child: AnimatedContainer(
          duration: Duration(milliseconds: 100),
          width: 46,
          height: 40,
          color: _isHovered
              ? (widget.hoverColor ?? Colors.grey.withOpacity(0.3))
              : Colors.transparent,
          child: Icon(widget.icon, size: 16),
        ),
      ),
    );
  }
}
```

---

## 2. Desktop-specific UI Patterns

### Context Menu (Right-click)

```dart
// widgets/context_menu.dart
import 'package:flutter/gestures.dart';

class ContextMenuArea extends StatelessWidget {
  final Widget child;
  final List<ContextMenuItem> menuItems;

  const ContextMenuArea({
    required this.child,
    required this.menuItems,
  });

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onSecondaryTapUp: (details) {
        _showContextMenu(context, details.globalPosition);
      },
      child: Listener(
        onPointerDown: (event) {
          if (event.kind == PointerDeviceKind.mouse &&
              event.buttons == kSecondaryMouseButton) {
            _showContextMenu(context, event.position);
          }
        },
        child: child,
      ),
    );
  }

  void _showContextMenu(BuildContext context, Offset position) {
    showMenu(
      context: context,
      position: RelativeRect.fromLTRB(
        position.dx,
        position.dy,
        position.dx + 1,
        position.dy + 1,
      ),
      items: menuItems
          .map((item) => PopupMenuItem(
                value: item.value,
                enabled: item.enabled,
                child: Row(
                  children: [
                    if (item.icon != null) ...[
                      Icon(item.icon, size: 16),
                      SizedBox(width: 8),
                    ],
                    Text(item.label),
                    if (item.shortcut != null) ...[
                      Spacer(),
                      Text(
                        item.shortcut!,
                        style: TextStyle(
                          color: Colors.grey,
                          fontSize: 12,
                        ),
                      ),
                    ],
                  ],
                ),
              ))
          .toList(),
    ).then((value) {
      if (value != null) {
        final item = menuItems.firstWhere((i) => i.value == value);
        item.onPressed?.call();
      }
    });
  }
}

class ContextMenuItem {
  final String label;
  final dynamic value;
  final IconData? icon;
  final String? shortcut;
  final bool enabled;
  final VoidCallback? onPressed;

  ContextMenuItem({
    required this.label,
    required this.value,
    this.icon,
    this.shortcut,
    this.enabled = true,
    this.onPressed,
  });
}
```

### Keyboard Shortcuts

```dart
// shortcuts/app_shortcuts.dart
class AppShortcuts extends StatelessWidget {
  final Widget child;

  const AppShortcuts({required this.child});

  @override
  Widget build(BuildContext context) {
    return Shortcuts(
      shortcuts: {
        // File operations
        SingleActivator(LogicalKeyboardKey.keyN, control: true):
            NewFileIntent(),
        SingleActivator(LogicalKeyboardKey.keyO, control: true):
            OpenFileIntent(),
        SingleActivator(LogicalKeyboardKey.keyS, control: true):
            SaveFileIntent(),
        SingleActivator(LogicalKeyboardKey.keyS,
                control: true, shift: true):
            SaveAsFileIntent(),

        // Edit operations
        SingleActivator(LogicalKeyboardKey.keyZ, control: true):
            UndoIntent(),
        SingleActivator(LogicalKeyboardKey.keyY, control: true):
            RedoIntent(),
        SingleActivator(LogicalKeyboardKey.keyA, control: true):
            SelectAllIntent(),
        SingleActivator(LogicalKeyboardKey.keyF, control: true):
            FindIntent(),

        // View
        SingleActivator(LogicalKeyboardKey.equal, control: true):
            ZoomInIntent(),
        SingleActivator(LogicalKeyboardKey.minus, control: true):
            ZoomOutIntent(),

        // App
        SingleActivator(LogicalKeyboardKey.keyQ, control: true):
            QuitIntent(),

        // macOS
        if (Platform.isMacOS) ..._macOSShortcuts,
      },
      child: Actions(
        actions: {
          NewFileIntent: CallbackAction<NewFileIntent>(
            onInvoke: (_) => _handleNewFile(context),
          ),
          SaveFileIntent: CallbackAction<SaveFileIntent>(
            onInvoke: (_) => _handleSave(context),
          ),
          QuitIntent: CallbackAction<QuitIntent>(
            onInvoke: (_) => _handleQuit(context),
          ),
        },
        child: child,
      ),
    );
  }

  Map<ShortcutActivator, Intent> get _macOSShortcuts => {
        SingleActivator(LogicalKeyboardKey.keyN, meta: true):
            NewFileIntent(),
        SingleActivator(LogicalKeyboardKey.keyS, meta: true):
            SaveFileIntent(),
        SingleActivator(LogicalKeyboardKey.keyQ, meta: true):
            QuitIntent(),
      };

  void _handleNewFile(BuildContext context) {
    // Handle new file
  }

  void _handleSave(BuildContext context) {
    // Handle save
  }

  void _handleQuit(BuildContext context) {
    windowManager.close();
  }
}

// Intent classes
class NewFileIntent extends Intent {}
class OpenFileIntent extends Intent {}
class SaveFileIntent extends Intent {}
class SaveAsFileIntent extends Intent {}
class UndoIntent extends Intent {}
class RedoIntent extends Intent {}
class SelectAllIntent extends Intent {}
class FindIntent extends Intent {}
class ZoomInIntent extends Intent {}
class ZoomOutIntent extends Intent {}
class QuitIntent extends Intent {}
```

---

## 3. File System Access

```dart
// services/file_service.dart
import 'dart:io';
import 'package:file_picker/file_picker.dart';
import 'package:path_provider/path_provider.dart';

class FileService {
  // เปิด file picker สำหรับเลือกไฟล์
  Future<String?> openFile({List<String>? allowedExtensions}) async {
    final result = await FilePicker.platform.pickFiles(
      type: FileType.custom,
      allowedExtensions: allowedExtensions ?? ['txt', 'md', 'dart'],
      allowMultiple: false,
    );

    return result?.files.single.path;
  }

  // เปิด file picker สำหรับเลือกหลายไฟล์
  Future<List<String>> openFiles({List<String>? allowedExtensions}) async {
    final result = await FilePicker.platform.pickFiles(
      type: FileType.custom,
      allowedExtensions: allowedExtensions,
      allowMultiple: true,
    );

    return result?.files.map((f) => f.path!).toList() ?? [];
  }

  // Save file dialog
  Future<String?> saveFile({
    String? suggestedName,
    List<String>? allowedExtensions,
  }) async {
    return await FilePicker.platform.saveFile(
      fileName: suggestedName,
      type: FileType.custom,
      allowedExtensions: allowedExtensions,
    );
  }

  // เลือก directory
  Future<String?> selectDirectory() async {
    return await FilePicker.platform.getDirectoryPath();
  }

  // อ่านไฟล์
  Future<String> readFile(String path) async {
    return await File(path).readAsString();
  }

  // เขียนไฟล์
  Future<void> writeFile(String path, String content) async {
    await File(path).writeAsString(content);
  }

  // ดึง documents directory
  Future<String> getDocumentsDirectory() async {
    final dir = await getApplicationDocumentsDirectory();
    return dir.path;
  }

  // ดึง downloads directory
  Future<String?> getDownloadsDirectory() async {
    if (Platform.isWindows) {
      return '${Platform.environment['USERPROFILE']}\\Downloads';
    } else if (Platform.isMacOS || Platform.isLinux) {
      return '${Platform.environment['HOME']}/Downloads';
    }
    return null;
  }

  // ตรวจสอบว่าไฟล์มีการเปลี่ยนแปลง
  Stream<FileSystemEvent> watchFile(String path) {
    return File(path).watch();
  }

  // Watch directory for changes
  Stream<FileSystemEvent> watchDirectory(String path) {
    return Directory(path).watch(recursive: true);
  }
}
```

---

## Workshop: Desktop Note-taking App

```dart
// Complete Desktop Note-taking App

// main.dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  if (Platform.isWindows || Platform.isMacOS || Platform.isLinux) {
    await windowManager.ensureInitialized();

    WindowOptions options = WindowOptions(
      size: Size(1200, 800),
      minimumSize: Size(800, 500),
      center: true,
      title: 'NotePad Pro',
    );

    windowManager.waitUntilReadyToShow(options, () async {
      await windowManager.show();
    });
  }

  runApp(
    MultiProvider(
      providers: [
        ChangeNotifierProvider(create: (_) => NotesProvider()),
        ChangeNotifierProvider(create: (_) => EditorProvider()),
        ChangeNotifierProvider(create: (_) => ThemeProvider()),
      ],
      child: DesktopNoteApp(),
    ),
  );
}

// app.dart
class DesktopNoteApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Consumer<ThemeProvider>(
      builder: (context, themeProvider, _) {
        return MaterialApp(
          title: 'NotePad Pro',
          theme: themeProvider.lightTheme,
          darkTheme: themeProvider.darkTheme,
          themeMode: themeProvider.themeMode,
          home: AppShortcuts(child: MainWindow()),
        );
      },
    );
  }
}

// screens/main_window.dart
class MainWindow extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(
        children: [
          // Custom title bar (macOS/Windows)
          if (Platform.isMacOS || Platform.isWindows)
            CustomTitleBar(),

          // Menu bar
          _MenuBar(),

          // Main content
          Expanded(
            child: Row(
              children: [
                // Sidebar
                SizedBox(
                  width: 250,
                  child: NotesSidebar(),
                ),
                VerticalDivider(width: 1),

                // Editor
                Expanded(
                  child: NoteEditor(),
                ),
              ],
            ),
          ),

          // Status bar
          _StatusBar(),
        ],
      ),
    );
  }
}

// providers/notes_provider.dart
class NotesProvider extends ChangeNotifier {
  final _fileService = FileService();

  List<NoteFile> _notes = [];
  NoteFile? _selectedNote;
  String? _currentDirectory;

  List<NoteFile> get notes => _notes;
  NoteFile? get selectedNote => _selectedNote;

  Future<void> openDirectory() async {
    final path = await _fileService.selectDirectory();
    if (path != null) {
      _currentDirectory = path;
      await _loadNotesFromDirectory(path);
    }
  }

  Future<void> _loadNotesFromDirectory(String path) async {
    final dir = Directory(path);
    final files = await dir
        .list()
        .where((entity) =>
            entity is File &&
            (entity.path.endsWith('.md') || entity.path.endsWith('.txt')))
        .cast<File>()
        .toList();

    _notes = files
        .map((f) => NoteFile(
              path: f.path,
              name: f.path.split(Platform.pathSeparator).last,
            ))
        .toList();

    notifyListeners();
  }

  Future<void> selectNote(NoteFile note) async {
    _selectedNote = note;
    final content = await _fileService.readFile(note.path);
    note.content = content;
    notifyListeners();
  }

  Future<void> createNote() async {
    if (_currentDirectory == null) {
      final path = await _fileService.saveFile(
        suggestedName: 'New Note.md',
        allowedExtensions: ['md', 'txt'],
      );
      if (path != null) {
        await File(path).writeAsString('');
        final note = NoteFile(
          path: path,
          name: path.split(Platform.pathSeparator).last,
          content: '',
        );
        _notes.insert(0, note);
        _selectedNote = note;
        notifyListeners();
      }
    }
  }

  Future<void> saveCurrentNote() async {
    if (_selectedNote != null && _selectedNote!.content != null) {
      await _fileService.writeFile(
        _selectedNote!.path,
        _selectedNote!.content!,
      );
      _selectedNote!.isSaved = true;
      notifyListeners();
    }
  }

  void updateNoteContent(String content) {
    if (_selectedNote != null) {
      _selectedNote!.content = content;
      _selectedNote!.isSaved = false;
      notifyListeners();
    }
  }
}

// screens/notes_sidebar.dart
class NotesSidebar extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Consumer<NotesProvider>(
      builder: (context, provider, _) {
        return Column(
          children: [
            // Toolbar
            Container(
              padding: EdgeInsets.symmetric(horizontal: 8, vertical: 4),
              child: Row(
                children: [
                  IconButton(
                    icon: Icon(Icons.folder_open, size: 20),
                    onPressed: () => provider.openDirectory(),
                    tooltip: 'เปิดโฟลเดอร์',
                  ),
                  IconButton(
                    icon: Icon(Icons.add, size: 20),
                    onPressed: () => provider.createNote(),
                    tooltip: 'บันทึกใหม่',
                  ),
                  Spacer(),
                  Text(
                    '${provider.notes.length} ไฟล์',
                    style: TextStyle(fontSize: 12, color: Colors.grey),
                  ),
                ],
              ),
            ),
            Divider(height: 1),
            Expanded(
              child: provider.notes.isEmpty
                  ? Center(
                      child: Column(
                        mainAxisAlignment: MainAxisAlignment.center,
                        children: [
                          Icon(Icons.folder, size: 48, color: Colors.grey),
                          SizedBox(height: 8),
                          Text('เปิดโฟลเดอร์เพื่อเริ่มต้น',
                              style: TextStyle(color: Colors.grey)),
                        ],
                      ),
                    )
                  : ListView.builder(
                      itemCount: provider.notes.length,
                      itemBuilder: (context, index) {
                        final note = provider.notes[index];
                        final isSelected = provider.selectedNote == note;
                        return ContextMenuArea(
                          menuItems: [
                            ContextMenuItem(
                              label: 'เปิด',
                              value: 'open',
                              icon: Icons.open_in_new,
                              onPressed: () => provider.selectNote(note),
                            ),
                            ContextMenuItem(
                              label: 'ลบ',
                              value: 'delete',
                              icon: Icons.delete,
                              onPressed: () {},
                            ),
                          ],
                          child: ListTile(
                            dense: true,
                            selected: isSelected,
                            leading: Icon(
                              note.path.endsWith('.md')
                                  ? Icons.description
                                  : Icons.text_snippet,
                              size: 16,
                            ),
                            title: Text(
                              note.name,
                              style: TextStyle(fontSize: 13),
                              overflow: TextOverflow.ellipsis,
                            ),
                            trailing: !note.isSaved
                                ? Container(
                                    width: 8,
                                    height: 8,
                                    decoration: BoxDecoration(
                                      color: Colors.blue,
                                      shape: BoxShape.circle,
                                    ),
                                  )
                                : null,
                            onTap: () => provider.selectNote(note),
                          ),
                        );
                      },
                    ),
            ),
          ],
        );
      },
    );
  }
}

// screens/note_editor.dart
class NoteEditor extends StatefulWidget {
  @override
  _NoteEditorState createState() => _NoteEditorState();
}

class _NoteEditorState extends State<NoteEditor> {
  final _controller = TextEditingController();
  Timer? _autoSaveTimer;

  @override
  void initState() {
    super.initState();
    _controller.addListener(_onTextChanged);
  }

  void _onTextChanged() {
    context.read<NotesProvider>().updateNoteContent(_controller.text);

    // Auto-save after 2 seconds of inactivity
    _autoSaveTimer?.cancel();
    _autoSaveTimer = Timer(Duration(seconds: 2), () {
      context.read<NotesProvider>().saveCurrentNote();
    });
  }

  @override
  void dispose() {
    _controller.removeListener(_onTextChanged);
    _controller.dispose();
    _autoSaveTimer?.cancel();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Consumer<NotesProvider>(
      builder: (context, provider, _) {
        final note = provider.selectedNote;

        if (note == null) {
          return Center(
            child: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                Icon(Icons.note, size: 64, color: Colors.grey),
                SizedBox(height: 16),
                Text('เลือกบันทึกเพื่อเริ่มแก้ไข',
                    style: TextStyle(color: Colors.grey)),
              ],
            ),
          );
        }

        // อัปเดต controller เมื่อเปลี่ยน note
        if (_controller.text != note.content) {
          _controller.text = note.content ?? '';
        }

        return Column(
          children: [
            // Editor toolbar
            Container(
              padding: EdgeInsets.symmetric(horizontal: 8, vertical: 4),
              child: Row(
                children: [
                  Text(
                    note.name,
                    style: TextStyle(fontWeight: FontWeight.bold),
                  ),
                  SizedBox(width: 8),
                  if (!note.isSaved)
                    Text(
                      '(แก้ไขแล้ว)',
                      style: TextStyle(color: Colors.grey, fontSize: 12),
                    ),
                  Spacer(),
                  TextButton.icon(
                    onPressed: () => provider.saveCurrentNote(),
                    icon: Icon(Icons.save, size: 16),
                    label: Text('บันทึก'),
                  ),
                ],
              ),
            ),
            Divider(height: 1),
            Expanded(
              child: TextField(
                controller: _controller,
                maxLines: null,
                expands: true,
                decoration: InputDecoration(
                  border: InputBorder.none,
                  contentPadding: EdgeInsets.all(24),
                ),
                style: TextStyle(
                  fontFamily: 'Cascadia Code, Fira Code, monospace',
                  fontSize: 14,
                  height: 1.6,
                ),
              ),
            ),
          ],
        );
      },
    );
  }
}

class NoteFile {
  final String path;
  final String name;
  String? content;
  bool isSaved;

  NoteFile({
    required this.path,
    required this.name,
    this.content,
    this.isSaved = true,
  });
}
```

---

## สรุป

Desktop Applications ด้วย Flutter ประกอบด้วย:

1. **Window Management** - window_manager สำหรับควบคุม window
2. **Custom Title Bar** - สร้าง title bar เอง
3. **Keyboard Shortcuts** - Shortcuts widget
4. **Context Menu** - right-click menu
5. **File System Access** - file_picker สำหรับ open/save
6. **Desktop UI Patterns** - Sidebar, Toolbar, Status bar
