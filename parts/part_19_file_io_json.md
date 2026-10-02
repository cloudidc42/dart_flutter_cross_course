# Part 19: File I/O และ JSON ใน Dart

## บทนำ

การอ่านเขียนไฟล์และจัดการ JSON เป็นทักษะพื้นฐานที่จำเป็นสำหรับแอปพลิเคชันทุกประเภท ทั้ง CLI apps, server-side, และ Flutter apps

## 19.1 File Reading/Writing

### dart:io

```dart
import 'dart:io';

// === อ่านไฟล์ ===

// อ่านทั้งไฟล์ (sync - สำหรับ CLI/server)
String readFileSync(String path) {
  try {
    return File(path).readAsStringSync();
  } on FileSystemException catch (e) {
    throw Exception('ไม่สามารถอ่านไฟล์ได้: ${e.message}');
  }
}

// อ่านทั้งไฟล์ (async)
Future<String> readFileAsync(String path) async {
  try {
    return await File(path).readAsString();
  } on FileSystemException catch (e) {
    throw Exception('ไม่สามารถอ่านไฟล์ได้: ${e.message}');
  }
}

// อ่านแบบ bytes
Future<List<int>> readBytesAsync(String path) async {
  return File(path).readAsBytes();
}

// อ่านทีละบรรทัด
Future<List<String>> readLines(String path) async {
  return File(path).readAsLines();
}

// อ่านแบบ stream (สำหรับไฟล์ใหญ่)
Stream<String> readLargeFile(String path) {
  return File(path)
      .openRead()
      .transform(const SystemEncoding().decoder)
      .transform(const LineSplitter());
}

// === เขียนไฟล์ ===

// เขียนทั้งไฟล์ (sync)
void writeFileSync(String path, String content) {
  File(path).writeAsStringSync(content);
}

// เขียนทั้งไฟล์ (async)
Future<void> writeFileAsync(String path, String content) async {
  await File(path).writeAsString(content);
}

// ต่อท้ายไฟล์
Future<void> appendToFile(String path, String content) async {
  await File(path).writeAsString(
    content,
    mode: FileMode.append,
  );
}

// เขียน bytes
Future<void> writeBytesAsync(String path, List<int> bytes) async {
  await File(path).writeAsBytes(bytes);
}

// เขียนแบบ stream
Future<void> writeLargeFile(String path, Stream<List<int>> data) async {
  final sink = File(path).openWrite();
  try {
    await sink.addStream(data);
  } finally {
    await sink.close();
  }
}
```

### ตรวจสอบและจัดการไฟล์

```dart
import 'dart:io';
import 'package:path/path.dart' as path;

class FileUtils {
  // ตรวจสอบว่าไฟล์มีอยู่
  static bool exists(String filePath) {
    return File(filePath).existsSync();
  }

  // ตรวจสอบว่า directory มีอยู่
  static bool directoryExists(String dirPath) {
    return Directory(dirPath).existsSync();
  }

  // สร้าง directory (ถ้าไม่มี)
  static void ensureDirectory(String dirPath) {
    final dir = Directory(dirPath);
    if (!dir.existsSync()) {
      dir.createSync(recursive: true);
      print('สร้าง directory: $dirPath');
    }
  }

  // ลบไฟล์
  static void deleteFile(String filePath) {
    final file = File(filePath);
    if (file.existsSync()) {
      file.deleteSync();
    }
  }

  // ย้ายไฟล์
  static Future<void> moveFile(String from, String to) async {
    await File(from).rename(to);
  }

  // คัดลอกไฟล์
  static Future<void> copyFile(String from, String to) async {
    await File(from).copy(to);
  }

  // ขนาดไฟล์
  static int fileSize(String filePath) {
    return File(filePath).lengthSync();
  }

  // รายชื่อไฟล์ใน directory
  static List<String> listFiles(String dirPath, {String? extension}) {
    final dir = Directory(dirPath);
    if (!dir.existsSync()) return [];

    return dir
        .listSync()
        .whereType<File>()
        .where((f) => extension == null || f.path.endsWith(extension))
        .map((f) => f.path)
        .toList();
  }

  // ขนาดไฟล์ที่อ่านง่าย
  static String formatSize(int bytes) {
    if (bytes < 1024) return '$bytes B';
    if (bytes < 1024 * 1024) return '${(bytes / 1024).toStringAsFixed(1)} KB';
    if (bytes < 1024 * 1024 * 1024) {
      return '${(bytes / (1024 * 1024)).toStringAsFixed(1)} MB';
    }
    return '${(bytes / (1024 * 1024 * 1024)).toStringAsFixed(2)} GB';
  }
}

// ตัวอย่างการใช้งาน
Future<void> fileExample() async {
  const dir = '/tmp/dart_example';
  const file = '$dir/test.txt';

  // สร้าง directory
  FileUtils.ensureDirectory(dir);

  // เขียนไฟล์
  await writeFileAsync(file, 'บรรทัดที่ 1\nบรรทัดที่ 2\nบรรทัดที่ 3');
  print('ขนาดไฟล์: ${FileUtils.formatSize(FileUtils.fileSize(file))}');

  // อ่านทีละบรรทัด
  final lines = await readLines(file);
  lines.asMap().forEach((i, line) =>
    print('บรรทัด ${i + 1}: $line'));

  // ต่อท้าย
  await appendToFile(file, '\nบรรทัดที่ 4 (เพิ่มทีหลัง)');

  // อ่านทั้งหมด
  final content = await readFileAsync(file);
  print('เนื้อหาทั้งหมด:\n$content');

  // ล้าง
  FileUtils.deleteFile(file);
}
```

## 19.2 JSON Encode/Decode

### dart:convert

```dart
import 'dart:convert';

void basicJsonDemo() {
  print('=== JSON Encode/Decode ===');

  // Encode: Dart object → JSON string
  var data = {
    'name': 'สมชาย',
    'age': 30,
    'hobbies': ['อ่านหนังสือ', 'ดูหนัง'],
    'address': {
      'city': 'กรุงเทพฯ',
      'zip': '10110',
    },
    'active': true,
    'score': 9.5,
  };

  var jsonString = jsonEncode(data);
  print('JSON: $jsonString');

  // Decode: JSON string → Dart object
  var decoded = jsonDecode(jsonString) as Map<String, dynamic>;
  print('Name: ${decoded['name']}');
  print('Hobbies: ${decoded['hobbies']}');

  // Pretty print
  var encoder = JsonEncoder.withIndent('  ');
  var prettyJson = encoder.convert(data);
  print('\nFormatted JSON:\n$prettyJson');

  // Decode list
  var jsonList = '[1, 2, 3, 4, 5]';
  var list = jsonDecode(jsonList) as List<dynamic>;
  print('\nList: $list');
  var intList = list.cast<int>();
  print('Int list: $intList');
}

// JSON กับ DateTime (ต้องแปลงด้วยตัวเอง)
String encodeWithDateTime(Map<String, dynamic> data) {
  return jsonEncode(data, toEncodable: (obj) {
    if (obj is DateTime) return obj.toIso8601String();
    return obj.toString();
  });
}

Map<String, dynamic> decodeWithDateTime(String json) {
  var decoded = jsonDecode(json) as Map<String, dynamic>;
  // แปลง string กลับเป็น DateTime
  return decoded.map((key, value) {
    if (value is String) {
      final dt = DateTime.tryParse(value);
      if (dt != null) return MapEntry(key, dt);
    }
    return MapEntry(key, value);
  });
}
```

### Model Classes กับ JSON

```dart
// Manual JSON serialization
class Product {
  final String id;
  final String name;
  final double price;
  final int stock;
  final List<String> tags;
  final DateTime createdAt;

  Product({
    required this.id,
    required this.name,
    required this.price,
    required this.stock,
    required this.tags,
    required this.createdAt,
  });

  // fromJson constructor
  factory Product.fromJson(Map<String, dynamic> json) {
    return Product(
      id: json['id'] as String,
      name: json['name'] as String,
      price: (json['price'] as num).toDouble(),
      stock: json['stock'] as int,
      tags: List<String>.from(json['tags'] as List? ?? []),
      createdAt: DateTime.parse(json['created_at'] as String),
    );
  }

  // toJson method
  Map<String, dynamic> toJson() => {
    'id': id,
    'name': name,
    'price': price,
    'stock': stock,
    'tags': tags,
    'created_at': createdAt.toIso8601String(),
  };

  // fromJsonList
  static List<Product> fromJsonList(List<dynamic> jsonList) {
    return jsonList
        .map((json) => Product.fromJson(json as Map<String, dynamic>))
        .toList();
  }

  @override
  String toString() => 'Product($name, ฿$price)';
}

// ใช้งาน
void testProductJson() {
  var product = Product(
    id: 'p001',
    name: 'Flutter Book',
    price: 350.0,
    stock: 100,
    tags: ['programming', 'flutter', 'dart'],
    createdAt: DateTime(2024, 1, 15),
  );

  // Serialize
  var json = product.toJson();
  var jsonString = jsonEncode(json);
  print('JSON: $jsonString');

  // Deserialize
  var decoded = Product.fromJson(jsonDecode(jsonString));
  print('Decoded: $decoded');
  print('Created at: ${decoded.createdAt}');

  // List
  var jsonList = jsonEncode([product.toJson()]);
  var products = Product.fromJsonList(jsonDecode(jsonList) as List);
  print('List: $products');
}
```

## 19.3 jsonSerializable Package

### Setup

```yaml
# pubspec.yaml
dependencies:
  json_annotation: ^4.8.0

dev_dependencies:
  build_runner: ^2.4.0
  json_serializable: ^6.7.0
```

```dart
// lib/models/order.dart
import 'package:json_annotation/json_annotation.dart';

part 'order.g.dart';

@JsonSerializable(explicitToJson: true)
class Order {
  final String id;
  final String customerId;
  final List<OrderItem> items;

  @JsonKey(name: 'created_at')
  final DateTime createdAt;

  @JsonKey(name: 'total_amount')
  final double totalAmount;

  @JsonKey(defaultValue: 'pending')
  final String status;

  @JsonKey(includeIfNull: false)
  final String? notes;

  Order({
    required this.id,
    required this.customerId,
    required this.items,
    required this.createdAt,
    required this.totalAmount,
    required this.status,
    this.notes,
  });

  factory Order.fromJson(Map<String, dynamic> json) =>
      _$OrderFromJson(json);

  Map<String, dynamic> toJson() => _$OrderToJson(this);
}

@JsonSerializable()
class OrderItem {
  final String productId;
  final String productName;
  final int quantity;
  final double unitPrice;

  OrderItem({
    required this.productId,
    required this.productName,
    required this.quantity,
    required this.unitPrice,
  });

  double get total => quantity * unitPrice;

  factory OrderItem.fromJson(Map<String, dynamic> json) =>
      _$OrderItemFromJson(json);

  Map<String, dynamic> toJson() => _$OrderItemToJson(this);
}

// สร้าง generated file:
// dart run build_runner build
// หรือ watch:
// dart run build_runner watch
```

### Custom JSON Converters

```dart
// json_annotation Custom Converter
import 'package:json_annotation/json_annotation.dart';

class DateTimeConverter implements JsonConverter<DateTime, String> {
  const DateTimeConverter();

  @override
  DateTime fromJson(String json) => DateTime.parse(json);

  @override
  String toJson(DateTime object) => object.toIso8601String();
}

class DurationConverter implements JsonConverter<Duration, int> {
  const DurationConverter();

  @override
  Duration fromJson(int json) => Duration(milliseconds: json);

  @override
  int toJson(Duration object) => object.inMilliseconds;
}

// ใช้งาน
@JsonSerializable()
class TimedEvent {
  final String name;

  @DateTimeConverter()
  final DateTime startTime;

  @DurationConverter()
  final Duration duration;

  TimedEvent({
    required this.name,
    required this.startTime,
    required this.duration,
  });
}
```

## 19.4 CSV Parsing

```dart
import 'dart:io';

class CsvParser {
  final String delimiter;
  final bool hasHeader;

  CsvParser({
    this.delimiter = ',',
    this.hasHeader = true,
  });

  // Parse CSV string
  List<Map<String, String>> parseString(String csv) {
    final lines = csv
        .split('\n')
        .map((line) => line.trim())
        .where((line) => line.isNotEmpty)
        .toList();

    if (lines.isEmpty) return [];

    List<String> headers;
    int dataStart;

    if (hasHeader) {
      headers = _parseLine(lines[0]);
      dataStart = 1;
    } else {
      headers = List.generate(
        _parseLine(lines[0]).length,
        (i) => 'column$i',
      );
      dataStart = 0;
    }

    return lines.skip(dataStart).map((line) {
      final values = _parseLine(line);
      return Map.fromIterables(
        headers,
        values.length >= headers.length
            ? values.take(headers.length)
            : [...values, ...List.filled(headers.length - values.length, '')],
      );
    }).toList();
  }

  // Parse CSV file
  Future<List<Map<String, String>>> parseFile(String filePath) async {
    final content = await File(filePath).readAsString();
    return parseString(content);
  }

  // Parse ทีละบรรทัดสำหรับไฟล์ใหญ่
  Stream<Map<String, String>> parseFileStream(String filePath) async* {
    final file = File(filePath);
    final lines = file
        .openRead()
        .transform(const SystemEncoding().decoder)
        .transform(const LineSplitter());

    List<String>? headers;

    await for (final line in lines) {
      if (line.trim().isEmpty) continue;

      if (headers == null) {
        if (hasHeader) {
          headers = _parseLine(line);
          continue;
        } else {
          headers = List.generate(
            _parseLine(line).length,
            (i) => 'column$i',
          );
        }
      }

      final values = _parseLine(line);
      yield Map.fromIterables(
        headers,
        values.length >= headers.length
            ? values.take(headers.length)
            : [...values, ...List.filled(headers.length - values.length, '')],
      );
    }
  }

  List<String> _parseLine(String line) {
    final result = <String>[];
    var current = StringBuffer();
    var inQuotes = false;

    for (var i = 0; i < line.length; i++) {
      final char = line[i];

      if (char == '"') {
        if (inQuotes && i + 1 < line.length && line[i + 1] == '"') {
          current.write('"');
          i++; // skip next quote
        } else {
          inQuotes = !inQuotes;
        }
      } else if (char == delimiter && !inQuotes) {
        result.add(current.toString().trim());
        current.clear();
      } else {
        current.write(char);
      }
    }

    result.add(current.toString().trim());
    return result;
  }

  // สร้าง CSV string จาก data
  String stringify(
    List<Map<String, dynamic>> data, {
    List<String>? columns,
  }) {
    if (data.isEmpty) return '';

    final cols = columns ?? data.first.keys.toList();
    final lines = <String>[];

    if (hasHeader) {
      lines.add(cols.map(_escapeField).join(delimiter));
    }

    for (final row in data) {
      lines.add(cols.map((col) => _escapeField(row[col]?.toString() ?? '')).join(delimiter));
    }

    return lines.join('\n');
  }

  String _escapeField(String field) {
    if (field.contains(delimiter) || field.contains('"') || field.contains('\n')) {
      return '"${field.replaceAll('"', '""')}"';
    }
    return field;
  }
}

void testCsvParser() {
  final parser = CsvParser();

  const csv = '''
ชื่อ,อีเมล,อายุ,เมือง
สมชาย,somchai@test.com,30,กรุงเทพฯ
สมหญิง,somying@test.com,25,"เชียงใหม่, เหนือ"
สมศักดิ์,somsak@test.com,35,ภูเก็ต
''';

  final records = parser.parseString(csv);
  print('CSV Records:');
  for (final record in records) {
    print('  ${record['ชื่อ']} | ${record['อีเมล']} | ${record['เมือง']}');
  }

  // สร้าง CSV ใหม่
  final newCsv = parser.stringify(records);
  print('\nGenerated CSV:\n$newCsv');
}
```

## 19.5 Workshop: Config File Reader

สร้างระบบ config file ที่รองรับ JSON และ environment variables

```dart
import 'dart:io';
import 'dart:convert';

// ===== Config Models =====

class DatabaseConfig {
  final String host;
  final int port;
  final String name;
  final String username;
  final String password;
  final int maxConnections;

  DatabaseConfig({
    required this.host,
    required this.port,
    required this.name,
    required this.username,
    required this.password,
    this.maxConnections = 10,
  });

  factory DatabaseConfig.fromJson(Map<String, dynamic> json) {
    return DatabaseConfig(
      host: json['host'] as String? ?? 'localhost',
      port: json['port'] as int? ?? 5432,
      name: json['name'] as String,
      username: json['username'] as String,
      password: json['password'] as String,
      maxConnections: json['max_connections'] as int? ?? 10,
    );
  }

  Map<String, dynamic> toJson() => {
    'host': host,
    'port': port,
    'name': name,
    'username': username,
    'password': '***',
    'max_connections': maxConnections,
  };

  @override
  String toString() => jsonEncode(toJson());
}

class ServerConfig {
  final String host;
  final int port;
  final bool isHttps;
  final Duration timeout;

  ServerConfig({
    this.host = '0.0.0.0',
    this.port = 8080,
    this.isHttps = false,
    Duration? timeout,
  }) : timeout = timeout ?? const Duration(seconds: 30);

  factory ServerConfig.fromJson(Map<String, dynamic> json) {
    return ServerConfig(
      host: json['host'] as String? ?? '0.0.0.0',
      port: json['port'] as int? ?? 8080,
      isHttps: json['https'] as bool? ?? false,
      timeout: Duration(seconds: json['timeout_seconds'] as int? ?? 30),
    );
  }
}

class AppConfig {
  final String environment;
  final String appName;
  final String version;
  final ServerConfig server;
  final DatabaseConfig database;
  final Map<String, dynamic> extras;

  AppConfig({
    required this.environment,
    required this.appName,
    required this.version,
    required this.server,
    required this.database,
    this.extras = const {},
  });

  bool get isDevelopment => environment == 'development';
  bool get isProduction => environment == 'production';

  factory AppConfig.fromJson(Map<String, dynamic> json) {
    return AppConfig(
      environment: json['environment'] as String? ?? 'development',
      appName: json['app_name'] as String? ?? 'MyApp',
      version: json['version'] as String? ?? '1.0.0',
      server: ServerConfig.fromJson(
        json['server'] as Map<String, dynamic>? ?? {},
      ),
      database: DatabaseConfig.fromJson(
        json['database'] as Map<String, dynamic>,
      ),
      extras: json['extras'] as Map<String, dynamic>? ?? {},
    );
  }
}

// ===== Config Loader =====

class ConfigLoader {
  static AppConfig? _cached;

  // โหลด config จากไฟล์
  static AppConfig loadFromFile(String path) {
    final file = File(path);
    if (!file.existsSync()) {
      throw FileSystemException('ไม่พบไฟล์ config', path);
    }

    final content = file.readAsStringSync();
    return _parseAndOverride(content);
  }

  // โหลด config จาก string (สำหรับทดสอบ)
  static AppConfig loadFromString(String jsonContent) {
    return _parseAndOverride(jsonContent);
  }

  // Override ด้วย environment variables
  static AppConfig _parseAndOverride(String jsonContent) {
    final json = jsonDecode(jsonContent) as Map<String, dynamic>;

    // Override ด้วย ENV variables
    final env = Platform.environment;

    if (env.containsKey('APP_ENV')) {
      json['environment'] = env['APP_ENV'];
    }
    if (env.containsKey('DB_HOST')) {
      (json['database'] as Map<String, dynamic>)['host'] = env['DB_HOST'];
    }
    if (env.containsKey('DB_PORT')) {
      (json['database'] as Map<String, dynamic>)['port'] =
          int.tryParse(env['DB_PORT']!) ?? 5432;
    }
    if (env.containsKey('DB_PASSWORD')) {
      (json['database'] as Map<String, dynamic>)['password'] = env['DB_PASSWORD'];
    }
    if (env.containsKey('SERVER_PORT')) {
      (json['server'] as Map<String, dynamic>)['port'] =
          int.tryParse(env['SERVER_PORT']!) ?? 8080;
    }

    return AppConfig.fromJson(json);
  }

  // Singleton pattern
  static AppConfig get instance {
    if (_cached == null) {
      throw StateError('Config ยังไม่ได้โหลด กรุณาเรียก loadFromFile() ก่อน');
    }
    return _cached!;
  }

  static void setInstance(AppConfig config) {
    _cached = config;
  }

  // Validate config
  static List<String> validate(AppConfig config) {
    final errors = <String>[];

    if (config.appName.isEmpty) errors.add('app_name ต้องไม่ว่างเปล่า');
    if (config.database.name.isEmpty) errors.add('database.name ต้องไม่ว่างเปล่า');
    if (config.database.username.isEmpty) errors.add('database.username ต้องไม่ว่างเปล่า');
    if (config.server.port <= 0 || config.server.port > 65535) {
      errors.add('server.port ต้องอยู่ระหว่าง 1-65535');
    }

    return errors;
  }
}

// ===== Config File Manager =====

class ConfigFileManager {
  final String configDir;

  ConfigFileManager(this.configDir);

  // สร้าง config ตัวอย่าง
  Future<void> createSampleConfig(String environment) async {
    final config = {
      'environment': environment,
      'app_name': 'MyDartApp',
      'version': '1.0.0',
      'server': {
        'host': '0.0.0.0',
        'port': environment == 'production' ? 80 : 8080,
        'https': environment == 'production',
        'timeout_seconds': 30,
      },
      'database': {
        'host': 'localhost',
        'port': 5432,
        'name': 'myapp_$environment',
        'username': 'dbuser',
        'password': 'CHANGE_THIS_PASSWORD',
        'max_connections': environment == 'production' ? 50 : 5,
      },
      'extras': {
        'debug': environment != 'production',
        'log_level': environment == 'production' ? 'info' : 'debug',
      },
    };

    final encoder = JsonEncoder.withIndent('  ');
    final jsonContent = encoder.convert(config);
    final filePath = '$configDir/config.$environment.json';

    await File(filePath).writeAsString(jsonContent);
    print('สร้างไฟล์ config: $filePath');
  }

  // โหลด config ตาม environment
  AppConfig loadForEnvironment(String environment) {
    final path = '$configDir/config.$environment.json';

    // ลองโหลดไฟล์เฉพาะ environment ก่อน
    if (File(path).existsSync()) {
      return ConfigLoader.loadFromFile(path);
    }

    // ถ้าไม่มี ลองโหลด default config
    final defaultPath = '$configDir/config.json';
    if (File(defaultPath).existsSync()) {
      return ConfigLoader.loadFromFile(defaultPath);
    }

    throw FileSystemException(
      'ไม่พบไฟล์ config สำหรับ environment: $environment',
      path,
    );
  }
}

// ===== Workshop Main =====

Future<void> main() async {
  print('=== Workshop: Config File Reader ===\n');

  const configDir = '/tmp/dart_config_example';

  // สร้าง directory
  Directory(configDir).createSync(recursive: true);

  final manager = ConfigFileManager(configDir);

  // Test 1: สร้าง config files
  print('1. สร้าง config files:');
  await manager.createSampleConfig('development');
  await manager.createSampleConfig('production');

  // Test 2: โหลด config
  print('\n2. โหลด config:');

  // จาก string (สำหรับทดสอบ)
  final testJson = '''
  {
    "environment": "test",
    "app_name": "TestApp",
    "version": "1.0.0-test",
    "server": {"port": 9090},
    "database": {
      "host": "testdb",
      "port": 5432,
      "name": "testdb",
      "username": "testuser",
      "password": "testpass"
    }
  }
  ''';

  final testConfig = ConfigLoader.loadFromString(testJson);
  print('  Environment: ${testConfig.environment}');
  print('  App: ${testConfig.appName} v${testConfig.version}');
  print('  Server port: ${testConfig.server.port}');
  print('  DB host: ${testConfig.database.host}');

  // โหลดจากไฟล์
  final devConfig = manager.loadForEnvironment('development');
  print('\n  Dev config:');
  print('    Environment: ${devConfig.environment}');
  print('    Server: ${devConfig.server.host}:${devConfig.server.port}');
  print('    DB: ${devConfig.database.host}:${devConfig.database.port}/${devConfig.database.name}');
  print('    Debug: ${devConfig.extras['debug']}');

  // Test 3: Validation
  print('\n3. Validation:');
  var errors = ConfigLoader.validate(devConfig);
  if (errors.isEmpty) {
    print('  Config ถูกต้อง ✅');
  } else {
    print('  พบข้อผิดพลาด:');
    errors.forEach((e) => print('  - $e'));
  }

  // Test invalid config
  final invalidJson = '''
  {
    "environment": "test",
    "app_name": "",
    "version": "1.0.0",
    "server": {"port": 99999},
    "database": {"name": "", "username": "", "password": ""}
  }
  ''';

  final invalidConfig = ConfigLoader.loadFromString(invalidJson);
  var invalidErrors = ConfigLoader.validate(invalidConfig);
  print('\n  Invalid config errors:');
  invalidErrors.forEach((e) => print('  - $e'));

  // Test 4: CSV
  print('\n4. CSV Parsing:');
  final csvData = '''
ProductID,Name,Price,Stock,Category
P001,Flutter Book,350,100,Books
P002,Dart Guide,280,50,Books
P003,Keyboard,1200,20,Electronics
P004,Mouse,500,35,Electronics
''';

  final csvParser = CsvParser();
  final products = csvParser.parseString(csvData);
  print('  โหลด ${products.length} สินค้า');

  final electronics = products.where((p) => p['Category'] == 'Electronics');
  print('  Electronics:');
  electronics.forEach((p) =>
    print('    ${p['Name']}: ฿${p['Price']}'));

  // สรุปราคาเฉลี่ย
  final prices = products
      .map((p) => double.tryParse(p['Price'] ?? '0') ?? 0)
      .toList();
  final avgPrice = prices.fold(0.0, (a, b) => a + b) / prices.length;
  print('  ราคาเฉลี่ย: ฿${avgPrice.toStringAsFixed(2)}');

  // Test 5: JSON round-trip
  print('\n5. JSON Round-trip:');
  final product = {
    'id': 'P001',
    'name': 'Flutter Book',
    'price': 350.0,
    'tags': ['programming', 'flutter'],
    'created_at': DateTime.now().toIso8601String(),
  };

  final encoded = jsonEncode(product);
  final decoded = jsonDecode(encoded) as Map<String, dynamic>;

  print('  Original: ${product['name']} ฿${product['price']}');
  print('  Decoded:  ${decoded['name']} ฿${decoded['price']}');
  print('  Tags: ${decoded['tags']}');

  // ล้าง temp files
  Directory(configDir).deleteSync(recursive: true);
  print('\n  ล้างไฟล์ชั่วคราวแล้ว');

  print('\n=== เสร็จสิ้น Workshop ===');
}
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **File I/O** - อ่านเขียนไฟล์ทั้ง sync และ async
2. **JSON** - encode/decode ด้วย dart:convert
3. **Model serialization** - fromJson/toJson patterns
4. **jsonSerializable** - code generation สำหรับ JSON
5. **CSV parsing** - อ่านและสร้างไฟล์ CSV
6. **Workshop** - Config file reader ที่รองรับ environment variables

### Best Practices

- ใช้ async file I/O ในแอปจริง (ไม่บล็อก event loop)
- Validate JSON ก่อน parse (try/catch)
- ใช้ jsonSerializable สำหรับ model ที่ซับซ้อน
- เก็บ sensitive data ใน environment variables ไม่ใช่ config files
- ใช้ Stream สำหรับไฟล์ขนาดใหญ่
