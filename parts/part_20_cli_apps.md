# Part 20: Dart CLI Applications

## บทนำ

Dart ไม่ได้ใช้แค่สำหรับ Flutter เท่านั้น แต่ยังเหมาะมากสำหรับการสร้าง Command-Line Interface (CLI) applications ที่มีประสิทธิภาพ ในบทนี้เราจะเรียนรู้การสร้าง CLI tools อย่างมืออาชีพ

## 20.1 args Package

```yaml
# pubspec.yaml
dependencies:
  args: ^2.4.2
```

### ArgParser พื้นฐาน

```dart
import 'package:args/args.dart';
import 'dart:io';

void main(List<String> arguments) {
  final parser = ArgParser();

  // Options (มี value: --name=value หรือ --name value)
  parser.addOption(
    'name',
    abbr: 'n',          // short form: -n
    help: 'ชื่อผู้ใช้',
    defaultsTo: 'World',
    valueHelp: 'NAME',
  );

  parser.addOption(
    'output',
    abbr: 'o',
    help: 'ไฟล์ output',
    valueHelp: 'FILE',
  );

  // Flags (boolean: --verbose หรือ --no-verbose)
  parser.addFlag(
    'verbose',
    abbr: 'v',
    help: 'แสดงข้อมูลละเอียด',
    defaultsTo: false,
    negatable: true,  // รองรับ --no-verbose ด้วย
  );

  parser.addFlag(
    'help',
    abbr: 'h',
    help: 'แสดงวิธีใช้',
    negatable: false,
  );

  // Multi-option (รับหลายค่า: --tag=a --tag=b)
  parser.addMultiOption(
    'tag',
    abbr: 't',
    help: 'เพิ่ม tags',
    valueHelp: 'TAG',
  );

  try {
    final results = parser.parse(arguments);

    if (results['help'] as bool) {
      print('Usage: myapp [options]');
      print(parser.usage);
      exit(0);
    }

    final name = results['name'] as String;
    final verbose = results['verbose'] as bool;
    final tags = results['tag'] as List<String>;
    final output = results['output'] as String?;

    if (verbose) {
      print('Mode: verbose');
      print('Name: $name');
      print('Tags: $tags');
      print('Output: ${output ?? "stdout"}');
    }

    print('สวัสดี, $name!');

    // Positional arguments (ที่เหลือหลัง parse)
    final rest = results.rest;
    if (rest.isNotEmpty) {
      print('Arguments เพิ่มเติม: $rest');
    }
  } on FormatException catch (e) {
    print('Error: ${e.message}');
    print(parser.usage);
    exit(1);
  }
}
```

### Subcommands

```dart
import 'package:args/args.dart';
import 'package:args/command_runner.dart';
import 'dart:io';

// Command Runner
class TaskRunner extends CommandRunner<void> {
  TaskRunner()
      : super('task', 'Task Manager - จัดการงานใน CLI') {
    addCommand(AddCommand());
    addCommand(ListCommand());
    addCommand(DoneCommand());
    addCommand(DeleteCommand());
  }
}

// Base command
abstract class BaseCommand extends Command<void> {
  late final TaskStorage storage;

  @override
  Future<void> run() async {
    storage = await TaskStorage.load();
    await execute();
    await storage.save();
  }

  Future<void> execute();
}

// Add command
class AddCommand extends BaseCommand {
  @override
  String get name => 'add';

  @override
  String get description => 'เพิ่มงานใหม่';

  AddCommand() {
    argParser.addOption(
      'priority',
      abbr: 'p',
      help: 'ระดับความสำคัญ (low/medium/high)',
      defaultsTo: 'medium',
      allowed: ['low', 'medium', 'high'],
    );
    argParser.addOption(
      'due',
      abbr: 'd',
      help: 'กำหนดเสร็จ (YYYY-MM-DD)',
    );
  }

  @override
  Future<void> execute() async {
    if (argResults!.rest.isEmpty) {
      usageException('กรุณาระบุชื่องาน');
    }

    final title = argResults!.rest.join(' ');
    final priority = argResults!['priority'] as String;
    final dueStr = argResults!['due'] as String?;

    DateTime? dueDate;
    if (dueStr != null) {
      dueDate = DateTime.tryParse(dueStr);
      if (dueDate == null) {
        usageException('รูปแบบวันที่ไม่ถูกต้อง ใช้ YYYY-MM-DD');
      }
    }

    final task = Task(
      title: title,
      priority: Priority.fromString(priority),
      dueDate: dueDate,
    );

    storage.addTask(task);
    print('✅ เพิ่มงาน: ${task.title} [${task.priority.name}]');
    if (task.dueDate != null) {
      print('   กำหนดเสร็จ: ${task.dueDate!.toLocal().toString().split(' ')[0]}');
    }
  }
}

// List command
class ListCommand extends BaseCommand {
  @override
  String get name => 'list';

  @override
  String get description => 'แสดงรายการงาน';

  ListCommand() {
    argParser.addFlag(
      'all',
      abbr: 'a',
      help: 'แสดงทั้งงานที่เสร็จและยังไม่เสร็จ',
    );
    argParser.addOption(
      'filter',
      abbr: 'f',
      help: 'กรองตาม priority (low/medium/high)',
    );
    argParser.addFlag(
      'sort-by-due',
      help: 'เรียงตามวันกำหนด',
    );
  }

  @override
  Future<void> execute() async {
    var tasks = storage.tasks;

    if (!(argResults!['all'] as bool)) {
      tasks = tasks.where((t) => !t.isDone).toList();
    }

    final filter = argResults!['filter'] as String?;
    if (filter != null) {
      final priority = Priority.fromString(filter);
      tasks = tasks.where((t) => t.priority == priority).toList();
    }

    if (argResults!['sort-by-due'] as bool) {
      tasks.sort((a, b) {
        if (a.dueDate == null && b.dueDate == null) return 0;
        if (a.dueDate == null) return 1;
        if (b.dueDate == null) return -1;
        return a.dueDate!.compareTo(b.dueDate!);
      });
    }

    if (tasks.isEmpty) {
      print('ไม่มีงาน');
      return;
    }

    print('\n📋 รายการงาน (${tasks.length} รายการ)\n');
    print('${'ID'.<7} ${'สถานะ'.<6} ${'ความสำคัญ'.<10} ${'ชื่องาน'.<30} กำหนดเสร็จ');
    print('─' * 70);

    for (final task in tasks) {
      final status = task.isDone ? '✅' : '⏳';
      final priority = switch (task.priority) {
        Priority.high => '🔴 สูง',
        Priority.medium => '🟡 กลาง',
        Priority.low => '🟢 ต่ำ',
      };
      final due = task.dueDate?.toLocal().toString().split(' ')[0] ?? '-';
      final id = task.id.substring(0, 6);

      print('${id.<7} ${status.<6} ${priority.<10} ${task.title.<30} $due');
    }
    print();
  }
}

// Done command
class DoneCommand extends BaseCommand {
  @override
  String get name => 'done';

  @override
  String get description => 'ทำเครื่องหมายงานว่าเสร็จแล้ว';

  @override
  Future<void> execute() async {
    if (argResults!.rest.isEmpty) {
      usageException('กรุณาระบุ ID งาน');
    }

    final id = argResults!.rest.first;
    if (storage.markDone(id)) {
      print('✅ ทำเครื่องหมายงาน $id ว่าเสร็จแล้ว');
    } else {
      print('❌ ไม่พบงาน ID: $id');
    }
  }
}

// Delete command
class DeleteCommand extends BaseCommand {
  @override
  String get name => 'delete';

  @override
  String get description => 'ลบงาน';

  @override
  Future<void> execute() async {
    if (argResults!.rest.isEmpty) {
      usageException('กรุณาระบุ ID งาน');
    }

    final id = argResults!.rest.first;
    if (storage.deleteTask(id)) {
      print('🗑️  ลบงาน $id แล้ว');
    } else {
      print('❌ ไม่พบงาน ID: $id');
    }
  }
}

extension StringPad on String {
  String operator <(int width) => padRight(width);
  String operator >(int width) => padLeft(width);
}
```

## 20.2 stdin/stdout

### อ่าน input จาก stdin

```dart
import 'dart:io';
import 'dart:convert';

// อ่านบรรทัดเดียว
String readLine({String prompt = '> '}) {
  stdout.write(prompt);
  return stdin.readLineSync(encoding: const Utf8Codec()) ?? '';
}

// อ่านตัวเลข
int? readInt({String prompt = '> '}) {
  return int.tryParse(readLine(prompt: prompt));
}

double? readDouble({String prompt = '> '}) {
  return double.tryParse(readLine(prompt: prompt));
}

// Interactive menu
Future<void> showMenu() async {
  while (true) {
    print('\n═══ เมนูหลัก ═══');
    print('1. เพิ่มข้อมูล');
    print('2. แสดงข้อมูล');
    print('3. ค้นหา');
    print('0. ออก');
    print('═════════════════');

    final choice = readInt(prompt: 'เลือก: ');

    switch (choice) {
      case 1:
        print('กรุณาใส่ข้อมูล:');
        final name = readLine(prompt: 'ชื่อ: ');
        final age = readInt(prompt: 'อายุ: ');
        print('บันทึก: $name (อายุ $age ปี)');
      case 2:
        print('แสดงข้อมูลทั้งหมด...');
      case 3:
        final query = readLine(prompt: 'ค้นหา: ');
        print('กำลังค้นหา: "$query"');
      case 0:
        print('ลาก่อน!');
        return;
      default:
        print('❌ ตัวเลือกไม่ถูกต้อง');
    }
  }
}

// อ่าน stdin แบบ stream
Future<void> processStdin() async {
  await for (final line in stdin.transform(utf8.decoder).transform(const LineSplitter())) {
    print('ได้รับ: $line');
    if (line == 'exit') break;
  }
}

// Prompt with validation
String prompt(
  String message, {
  String? Function(String)? validator,
  String? defaultValue,
}) {
  while (true) {
    final display = defaultValue != null ? '$message [$defaultValue]: ' : '$message: ';
    stdout.write(display);

    final input = stdin.readLineSync(encoding: utf8) ?? '';
    final value = input.isEmpty && defaultValue != null ? defaultValue : input;

    if (validator != null) {
      final error = validator(value);
      if (error != null) {
        print('❌ $error');
        continue;
      }
    }

    return value;
  }
}

// Password input (ซ่อนอักขระ)
String promptPassword(String message) {
  stdout.write('$message: ');
  // บน Unix-like systems
  if (Platform.isLinux || Platform.isMacOS) {
    stdin.echoMode = false;
    final pass = stdin.readLineSync() ?? '';
    stdin.echoMode = true;
    print(); // newline หลังกด Enter
    return pass;
  }
  // บน Windows ทำได้ยากกว่า
  return stdin.readLineSync() ?? '';
}
```

## 20.3 Building CLI Tools

### ANSI Colors และ Styling

```dart
class Console {
  // Colors
  static const reset = '\x1B[0m';
  static const red = '\x1B[31m';
  static const green = '\x1B[32m';
  static const yellow = '\x1B[33m';
  static const blue = '\x1B[34m';
  static const magenta = '\x1B[35m';
  static const cyan = '\x1B[36m';
  static const white = '\x1B[37m';

  // Bright colors
  static const brightRed = '\x1B[91m';
  static const brightGreen = '\x1B[92m';
  static const brightYellow = '\x1B[93m';
  static const brightBlue = '\x1B[94m';

  // Styles
  static const bold = '\x1B[1m';
  static const dim = '\x1B[2m';
  static const underline = '\x1B[4m';

  // Background
  static const bgRed = '\x1B[41m';
  static const bgGreen = '\x1B[42m';
  static const bgBlue = '\x1B[44m';

  // Cursor
  static const clearLine = '\x1B[2K\r';

  static bool get supportsAnsi {
    return stdout.supportsAnsiEscapes;
  }

  static String colored(String text, String color) {
    if (!supportsAnsi) return text;
    return '$color$text$reset';
  }

  static void success(String message) =>
      print(colored('✅ $message', green));

  static void error(String message) =>
      print(colored('❌ $message', red));

  static void warning(String message) =>
      print(colored('⚠️  $message', yellow));

  static void info(String message) =>
      print(colored('ℹ️  $message', cyan));

  // Progress bar
  static void progressBar(double progress, {int width = 40}) {
    final filled = (progress * width).round();
    final empty = width - filled;
    final bar = '█' * filled + '░' * empty;
    final percent = (progress * 100).toStringAsFixed(1);
    stdout.write('\r${colored("[$bar]", blue)} $percent%');
  }

  // Spinner
  static Future<void> withSpinner(
    String message,
    Future<void> Function() task,
  ) async {
    const frames = ['⠋', '⠙', '⠹', '⠸', '⠼', '⠴', '⠦', '⠧', '⠇', '⠏'];
    var frame = 0;
    var done = false;

    final timer = Timer.periodic(Duration(milliseconds: 100), (_) {
      if (!done) {
        stdout.write('\r${frames[frame++ % frames.length]} $message...');
      }
    });

    try {
      await task();
      done = true;
      timer.cancel();
      stdout.write('\r${colored('✓', green)} $message\n');
    } catch (e) {
      done = true;
      timer.cancel();
      stdout.write('\r${colored('✗', red)} $message\n');
      rethrow;
    }
  }

  // Table
  static void table(
    List<String> headers,
    List<List<String>> rows, {
    bool color = true,
  }) {
    final colWidths = List.generate(
      headers.length,
      (i) => [
        headers[i].length,
        ...rows.map((r) => i < r.length ? r[i].length : 0),
      ].reduce((a, b) => a > b ? a : b),
    );

    void printRow(List<String> cells, [bool isHeader = false]) {
      final formatted = cells.asMap().entries.map((e) {
        final cell = e.key < cells.length ? cells[e.key] : '';
        return cell.padRight(colWidths[e.key]);
      }).join(' │ ');

      if (isHeader && color && supportsAnsi) {
        print('$bold$formatted$reset');
      } else {
        print(formatted);
      }
    }

    printRow(headers, true);
    print('─' * (colWidths.reduce((a, b) => a + b) + (colWidths.length - 1) * 3));
    for (final row in rows) {
      printRow(row);
    }
  }
}
```

### CLI App Structure

```dart
import 'dart:io';

abstract class CliApp {
  final String name;
  final String version;

  CliApp(this.name, this.version);

  // Entry point
  Future<int> run(List<String> args) async {
    try {
      return await execute(args);
    } on UsageException catch (e) {
      Console.error(e.message);
      if (e.usage != null) print(e.usage);
      return 1;
    } on Exception catch (e) {
      Console.error('เกิดข้อผิดพลาด: $e');
      return 1;
    }
  }

  Future<int> execute(List<String> args);

  void printBanner() {
    print(Console.colored('$name v$version', Console.bold));
    print('─' * 40);
  }
}

class UsageException implements Exception {
  final String message;
  final String? usage;
  UsageException(this.message, [this.usage]);
}
```

## 20.4 Process Management

```dart
import 'dart:io';

class ProcessRunner {
  // รัน command
  static Future<ProcessResult> run(
    String command, {
    List<String> args = const [],
    String? workingDirectory,
    Map<String, String>? environment,
    bool throwOnError = true,
  }) async {
    final result = await Process.run(
      command,
      args,
      workingDirectory: workingDirectory,
      environment: environment,
      runInShell: true,
    );

    if (throwOnError && result.exitCode != 0) {
      throw ProcessException(
        command,
        args,
        result.stderr.toString(),
        result.exitCode,
      );
    }

    return result;
  }

  // รัน และแสดง output แบบ stream
  static Future<int> runWithStream(
    String command, {
    List<String> args = const [],
    String? workingDirectory,
    bool printOutput = true,
  }) async {
    final process = await Process.start(
      command,
      args,
      workingDirectory: workingDirectory,
      runInShell: true,
    );

    if (printOutput) {
      process.stdout
          .transform(const SystemEncoding().decoder)
          .forEach(stdout.write);
      process.stderr
          .transform(const SystemEncoding().decoder)
          .forEach(stderr.write);
    }

    return process.exitCode;
  }

  // ตรวจสอบว่า command มีอยู่
  static Future<bool> commandExists(String command) async {
    try {
      final which = Platform.isWindows ? 'where' : 'which';
      await run(which, args: [command], throwOnError: true);
      return true;
    } catch (_) {
      return false;
    }
  }

  // รันหลาย commands ต่อกัน
  static Future<void> pipeline(
    List<(String, List<String>)> commands, {
    bool stopOnError = true,
  }) async {
    for (final (cmd, args) in commands) {
      print(Console.colored('$ $cmd ${args.join(' ')}', Console.dim));

      try {
        await runWithStream(cmd, args: args);
      } catch (e) {
        Console.error('ล้มเหลวที่: $cmd');
        if (stopOnError) rethrow;
      }
    }
  }
}

// Git helper
class GitHelper {
  final String workingDir;

  GitHelper(this.workingDir);

  Future<String> currentBranch() async {
    final result = await ProcessRunner.run(
      'git',
      args: ['rev-parse', '--abbrev-ref', 'HEAD'],
      workingDirectory: workingDir,
    );
    return result.stdout.toString().trim();
  }

  Future<List<String>> status() async {
    final result = await ProcessRunner.run(
      'git',
      args: ['status', '--short'],
      workingDirectory: workingDir,
    );
    return result.stdout.toString().split('\n').where((l) => l.isNotEmpty).toList();
  }

  Future<void> commit(String message) async {
    await ProcessRunner.run('git', args: ['add', '.'], workingDirectory: workingDir);
    await ProcessRunner.run('git', args: ['commit', '-m', message], workingDirectory: workingDir);
  }
}
```

## 20.5 Workshop: Simple Task Manager CLI

สร้าง task manager CLI แบบสมบูรณ์

```dart
import 'dart:io';
import 'dart:convert';
import 'package:args/command_runner.dart';

// ===== Models =====

enum Priority { low, medium, high }

extension PriorityExtension on Priority {
  String get display {
    return switch (this) {
      Priority.low => '🟢 ต่ำ',
      Priority.medium => '🟡 กลาง',
      Priority.high => '🔴 สูง',
    };
  }

  static Priority fromString(String s) {
    return switch (s.toLowerCase()) {
      'low' || 'l' => Priority.low,
      'high' || 'h' => Priority.high,
      _ => Priority.medium,
    };
  }
}

class Task {
  final String id;
  String title;
  Priority priority;
  bool isDone;
  DateTime? dueDate;
  final DateTime createdAt;
  List<String> tags;

  Task({
    required this.title,
    this.priority = Priority.medium,
    this.dueDate,
    this.tags = const [],
  })  : id = _generateId(),
        isDone = false,
        createdAt = DateTime.now();

  Task._fromMap(Map<String, dynamic> map)
      : id = map['id'] as String,
        title = map['title'] as String,
        priority = Priority.values.firstWhere(
          (p) => p.name == map['priority'],
          orElse: () => Priority.medium,
        ),
        isDone = map['isDone'] as bool,
        dueDate = map['dueDate'] != null
            ? DateTime.parse(map['dueDate'] as String)
            : null,
        createdAt = DateTime.parse(map['createdAt'] as String),
        tags = List<String>.from(map['tags'] as List? ?? []);

  Map<String, dynamic> toMap() => {
    'id': id,
    'title': title,
    'priority': priority.name,
    'isDone': isDone,
    'dueDate': dueDate?.toIso8601String(),
    'createdAt': createdAt.toIso8601String(),
    'tags': tags,
  };

  bool get isOverdue =>
      dueDate != null && !isDone && dueDate!.isBefore(DateTime.now());

  static String _generateId() {
    final timestamp = DateTime.now().millisecondsSinceEpoch;
    return timestamp.toRadixString(16);
  }
}

// ===== Storage =====

class TaskStorage {
  final String filePath;
  List<Task> tasks;

  TaskStorage._({required this.filePath, required this.tasks});

  static Future<TaskStorage> load({String? path}) async {
    final filePath = path ?? _defaultPath();
    final tasks = <Task>[];

    final file = File(filePath);
    if (file.existsSync()) {
      try {
        final content = await file.readAsString();
        final jsonList = jsonDecode(content) as List<dynamic>;
        tasks.addAll(jsonList.map((j) => Task._fromMap(j as Map<String, dynamic>)));
      } catch (e) {
        print('⚠️  ไม่สามารถโหลดข้อมูล: $e');
      }
    }

    return TaskStorage._(filePath: filePath, tasks: tasks);
  }

  Future<void> save() async {
    final file = File(filePath);
    await file.parent.create(recursive: true);
    final json = jsonEncode(tasks.map((t) => t.toMap()).toList());
    await file.writeAsString(json);
  }

  void addTask(Task task) => tasks.add(task);

  bool markDone(String idPrefix) {
    final task = _findTask(idPrefix);
    if (task == null) return false;
    task.isDone = true;
    return true;
  }

  bool deleteTask(String idPrefix) {
    final task = _findTask(idPrefix);
    if (task == null) return false;
    tasks.remove(task);
    return true;
  }

  Task? _findTask(String idPrefix) {
    try {
      return tasks.firstWhere((t) => t.id.startsWith(idPrefix));
    } catch (_) {
      return null;
    }
  }

  static String _defaultPath() {
    final home = Platform.environment['HOME'] ??
        Platform.environment['USERPROFILE'] ?? '.';
    return '$home/.task_manager/tasks.json';
  }
}

// ===== Commands =====

class TaskCommandRunner extends CommandRunner<void> {
  TaskCommandRunner()
      : super('task', 'Task Manager - จัดการงานใน CLI') {
    addCommand(TaskAddCommand());
    addCommand(TaskListCommand());
    addCommand(TaskDoneCommand());
    addCommand(TaskDeleteCommand());
    addCommand(TaskStatsCommand());
  }

  @override
  Future<void> run(Iterable<String> args) async {
    if (args.isEmpty) {
      // แสดง list เมื่อไม่มี arguments
      return super.run(['list']);
    }
    return super.run(args);
  }
}

// Add task command
class TaskAddCommand extends Command<void> {
  @override String get name => 'add';
  @override String get description => 'เพิ่มงานใหม่';

  TaskAddCommand() {
    argParser
      ..addOption('priority', abbr: 'p',
          help: 'ความสำคัญ', defaultsTo: 'medium',
          allowed: ['low', 'medium', 'high'])
      ..addOption('due', abbr: 'd', help: 'กำหนดเสร็จ (YYYY-MM-DD)')
      ..addMultiOption('tag', abbr: 't', help: 'Tag');
  }

  @override
  Future<void> run() async {
    if (argResults!.rest.isEmpty) {
      usageException('กรุณาระบุชื่องาน: task add <ชื่องาน>');
    }

    final storage = await TaskStorage.load();

    final title = argResults!.rest.join(' ');
    final priority = PriorityExtension.fromString(argResults!['priority'] as String);
    final tags = argResults!['tag'] as List<String>;

    DateTime? dueDate;
    final dueStr = argResults!['due'] as String?;
    if (dueStr != null) {
      dueDate = DateTime.tryParse(dueStr);
      if (dueDate == null) {
        usageException('รูปแบบวันที่ไม่ถูกต้อง ใช้ YYYY-MM-DD');
      }
    }

    final task = Task(
      title: title,
      priority: priority,
      dueDate: dueDate,
      tags: tags,
    );

    storage.addTask(task);
    await storage.save();

    print('✅ เพิ่มงาน [${task.id.substring(0, 8)}]: $title');
    print('   ความสำคัญ: ${task.priority.display}');
    if (dueDate != null) {
      print('   กำหนดเสร็จ: ${dueDate.toLocal().toString().split(' ')[0]}');
    }
    if (tags.isNotEmpty) print('   Tags: ${tags.join(', ')}');
  }
}

// List tasks command
class TaskListCommand extends Command<void> {
  @override String get name => 'list';
  @override String get description => 'แสดงรายการงาน';

  TaskListCommand() {
    argParser
      ..addFlag('all', abbr: 'a', help: 'แสดงทั้งหมดรวมที่เสร็จแล้ว')
      ..addFlag('overdue', help: 'แสดงเฉพาะที่เกินกำหนด')
      ..addOption('priority', abbr: 'p', help: 'กรองตาม priority')
      ..addOption('tag', abbr: 't', help: 'กรองตาม tag');
  }

  @override
  Future<void> run() async {
    final storage = await TaskStorage.load();
    var tasks = storage.tasks;

    // Apply filters
    final showAll = argResults!['all'] as bool;
    final overdueOnly = argResults!['overdue'] as bool;
    final priorityFilter = argResults!['priority'] as String?;
    final tagFilter = argResults!['tag'] as String?;

    if (!showAll) tasks = tasks.where((t) => !t.isDone).toList();
    if (overdueOnly) tasks = tasks.where((t) => t.isOverdue).toList();
    if (priorityFilter != null) {
      final p = PriorityExtension.fromString(priorityFilter);
      tasks = tasks.where((t) => t.priority == p).toList();
    }
    if (tagFilter != null) {
      tasks = tasks.where((t) => t.tags.contains(tagFilter)).toList();
    }

    // Sort: overdue first, then by priority, then by created date
    tasks.sort((a, b) {
      if (a.isOverdue && !b.isOverdue) return -1;
      if (!a.isOverdue && b.isOverdue) return 1;
      final pOrder = [Priority.high, Priority.medium, Priority.low];
      final aPriority = pOrder.indexOf(a.priority);
      final bPriority = pOrder.indexOf(b.priority);
      if (aPriority != bPriority) return aPriority.compareTo(bPriority);
      return a.createdAt.compareTo(b.createdAt);
    });

    if (tasks.isEmpty) {
      print('\n📭 ไม่มีงาน');
      return;
    }

    print('\n📋 รายการงาน (${tasks.length} รายการ)\n');

    for (final task in tasks) {
      final status = task.isDone
          ? '✅'
          : task.isOverdue
              ? '🚨'
              : '⏳';

      final id = task.id.substring(0, 8);
      final due = task.dueDate?.toLocal().toString().split(' ')[0];
      final overdueStr = task.isOverdue ? ' (เกินกำหนด!)' : '';

      print('$status [$id] ${task.priority.display.padRight(8)} ${task.title}');
      if (due != null) print('          📅 $due$overdueStr');
      if (task.tags.isNotEmpty) {
        print('          🏷️  ${task.tags.map((t) => '#$t').join(' ')}');
      }
    }
    print();
  }
}

// Done command
class TaskDoneCommand extends Command<void> {
  @override String get name => 'done';
  @override String get description => 'ทำเครื่องหมายงานว่าเสร็จ';

  @override
  Future<void> run() async {
    if (argResults!.rest.isEmpty) {
      usageException('กรุณาระบุ ID งาน');
    }

    final storage = await TaskStorage.load();
    final id = argResults!.rest.first;

    if (storage.markDone(id)) {
      await storage.save();
      print('✅ เสร็จงาน: $id');
    } else {
      print('❌ ไม่พบงาน ID: $id');
    }
  }
}

// Delete command
class TaskDeleteCommand extends Command<void> {
  @override String get name => 'delete';
  @override String get description => 'ลบงาน';

  @override
  Future<void> run() async {
    if (argResults!.rest.isEmpty) {
      usageException('กรุณาระบุ ID งาน');
    }

    final storage = await TaskStorage.load();
    final id = argResults!.rest.first;

    if (storage.deleteTask(id)) {
      await storage.save();
      print('🗑️  ลบงาน: $id');
    } else {
      print('❌ ไม่พบงาน ID: $id');
    }
  }
}

// Stats command
class TaskStatsCommand extends Command<void> {
  @override String get name => 'stats';
  @override String get description => 'สถิติงาน';

  @override
  Future<void> run() async {
    final storage = await TaskStorage.load();
    final tasks = storage.tasks;

    if (tasks.isEmpty) {
      print('ไม่มีข้อมูลสถิติ');
      return;
    }

    final total = tasks.length;
    final done = tasks.where((t) => t.isDone).length;
    final overdue = tasks.where((t) => t.isOverdue).length;
    final high = tasks.where((t) => t.priority == Priority.high && !t.isDone).length;

    print('\n📊 สถิติงาน\n');
    print('ทั้งหมด:      $total งาน');
    print('เสร็จแล้ว:    $done งาน (${(done / total * 100).toStringAsFixed(0)}%)');
    print('ยังค้างอยู่:  ${total - done} งาน');
    if (overdue > 0) print('เกินกำหนด:    $overdue งาน ⚠️');
    if (high > 0) print('ความสำคัญสูง: $high งาน 🔴');

    print('\nตามความสำคัญ:');
    for (final p in Priority.values.reversed) {
      final count = tasks.where((t) => t.priority == p && !t.isDone).length;
      if (count > 0) {
        print('  ${p.display}: $count งาน');
      }
    }

    // Progress bar
    final progress = done / total;
    const width = 30;
    final filled = (progress * width).round();
    final bar = '█' * filled + '░' * (width - filled);
    print('\nความคืบหน้า: [$bar] ${(progress * 100).toStringAsFixed(0)}%');
    print();
  }
}

// ===== Main =====

Future<void> main(List<String> args) async {
  final runner = TaskCommandRunner();

  try {
    await runner.run(args);
  } on UsageException catch (e) {
    print('Error: ${e.message}');
    print(e.usage);
    exit(64);
  }
}
```

### ตัวอย่างการใช้งาน

```bash
# เพิ่มงาน
dart run task.dart add "เขียนรายงาน Q1" --priority=high --due=2024-03-31
dart run task.dart add "ประชุมทีม" -p medium -d 2024-02-15 --tag=meeting
dart run task.dart add "อ่านหนังสือ Dart" -p low -t programming -t learning

# แสดงรายการ
dart run task.dart list
dart run task.dart list --all
dart run task.dart list --priority=high
dart run task.dart list --tag=meeting

# ทำเครื่องหมายเสร็จ
dart run task.dart done abc12345

# ลบงาน
dart run task.dart delete abc12345

# สถิติ
dart run task.dart stats
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **args package** - parse command-line arguments และ subcommands
2. **stdin/stdout** - อ่านและแสดงผลใน terminal
3. **CLI styling** - ANSI colors, progress bars, tables
4. **Process management** - รัน external processes
5. **Workshop** - Task Manager CLI สมบูรณ์

### โครงสร้าง CLI Project ที่ดี

```
my_cli/
├── bin/
│   └── my_cli.dart      # entry point
├── lib/
│   ├── my_cli.dart      # library exports
│   └── src/
│       ├── commands/    # command classes
│       ├── models/      # data models
│       ├── storage/     # persistence
│       └── utils/       # helpers
├── test/
│   └── ...
└── pubspec.yaml
```

### Best Practices

- ใช้ `args` package สำหรับ argument parsing
- ใช้ Command pattern สำหรับ subcommands
- แสดง helpful error messages เมื่อ usage ผิด
- ใช้ exit codes ที่ถูกต้อง (0=success, 1=error, 64=usage error)
- รองรับ `--help` ทุก command
- ใช้ ANSI colors อย่างระมัดระวัง (บาง terminals ไม่รองรับ)
- เก็บ user data ใน `~/.config/` หรือ `~/.local/share/`
