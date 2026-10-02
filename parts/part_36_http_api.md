# Part 36: HTTP Requests and REST APIs

## การทำงานกับ HTTP ใน Flutter

Flutter ใช้ `http` package สำหรับการเชื่อมต่อกับ REST APIs

```yaml
# pubspec.yaml
dependencies:
  http: ^1.1.0
  # หรือใช้ dio ที่มี features มากกว่า
  dio: ^5.3.0
```

---

## http Package

### GET Request พื้นฐาน

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;

// GET request ง่ายๆ
Future<void> fetchData() async {
  try {
    final response = await http.get(
      Uri.parse('https://jsonplaceholder.typicode.com/posts/1'),
    );
    
    if (response.statusCode == 200) {
      final data = jsonDecode(response.body);
      print('Title: ${data['title']}');
    } else {
      print('Error: ${response.statusCode}');
    }
  } catch (e) {
    print('Network error: $e');
  }
}

// GET request กับ query parameters
Future<List<Map<String, dynamic>>> searchPosts(String query) async {
  final uri = Uri.parse('https://jsonplaceholder.typicode.com/posts').replace(
    queryParameters: {
      'userId': '1',
      '_limit': '5',
      'q': query,
    },
  );
  
  final response = await http.get(uri);
  
  if (response.statusCode == 200) {
    final List<dynamic> data = jsonDecode(response.body);
    return data.cast<Map<String, dynamic>>();
  }
  
  throw Exception('Failed to search posts: ${response.statusCode}');
}
```

---

## GET, POST, PUT, DELETE

### GET - ดึงข้อมูล

```dart
class ApiService {
  static const String _baseUrl = 'https://jsonplaceholder.typicode.com';
  
  // GET all posts
  Future<List<Post>> getPosts() async {
    final response = await http.get(
      Uri.parse('$_baseUrl/posts'),
    );
    
    if (response.statusCode == 200) {
      final List<dynamic> jsonData = jsonDecode(response.body);
      return jsonData.map((json) => Post.fromJson(json)).toList();
    }
    
    throw HttpException('Failed to load posts: ${response.statusCode}');
  }
  
  // GET single post
  Future<Post> getPost(int id) async {
    final response = await http.get(
      Uri.parse('$_baseUrl/posts/$id'),
    );
    
    if (response.statusCode == 200) {
      return Post.fromJson(jsonDecode(response.body));
    } else if (response.statusCode == 404) {
      throw HttpException('Post not found');
    }
    
    throw HttpException('Failed to load post: ${response.statusCode}');
  }
  
  // GET with pagination
  Future<List<Post>> getPostsPaginated({int page = 1, int limit = 10}) async {
    final uri = Uri.parse('$_baseUrl/posts').replace(
      queryParameters: {
        '_page': '$page',
        '_limit': '$limit',
      },
    );
    
    final response = await http.get(uri);
    
    if (response.statusCode == 200) {
      final List<dynamic> jsonData = jsonDecode(response.body);
      return jsonData.map((json) => Post.fromJson(json)).toList();
    }
    
    throw HttpException('Failed to load posts');
  }
}
```

### POST - สร้างข้อมูล

```dart
// POST request
Future<Post> createPost(String title, String body, int userId) async {
  final response = await http.post(
    Uri.parse('$_baseUrl/posts'),
    headers: {
      'Content-Type': 'application/json; charset=UTF-8',
    },
    body: jsonEncode({
      'title': title,
      'body': body,
      'userId': userId,
    }),
  );
  
  if (response.statusCode == 201) {
    return Post.fromJson(jsonDecode(response.body));
  }
  
  throw HttpException('Failed to create post: ${response.statusCode}');
}
```

### PUT/PATCH - อัปเดตข้อมูล

```dart
// PUT - replace entire resource
Future<Post> updatePost(int id, String title, String body) async {
  final response = await http.put(
    Uri.parse('$_baseUrl/posts/$id'),
    headers: {
      'Content-Type': 'application/json; charset=UTF-8',
    },
    body: jsonEncode({
      'id': id,
      'title': title,
      'body': body,
      'userId': 1,
    }),
  );
  
  if (response.statusCode == 200) {
    return Post.fromJson(jsonDecode(response.body));
  }
  
  throw HttpException('Failed to update post');
}

// PATCH - partial update
Future<Post> patchPost(int id, {String? title, String? body}) async {
  final Map<String, dynamic> updates = {};
  if (title != null) updates['title'] = title;
  if (body != null) updates['body'] = body;
  
  final response = await http.patch(
    Uri.parse('$_baseUrl/posts/$id'),
    headers: {
      'Content-Type': 'application/json; charset=UTF-8',
    },
    body: jsonEncode(updates),
  );
  
  if (response.statusCode == 200) {
    return Post.fromJson(jsonDecode(response.body));
  }
  
  throw HttpException('Failed to patch post');
}
```

### DELETE - ลบข้อมูล

```dart
// DELETE request
Future<void> deletePost(int id) async {
  final response = await http.delete(
    Uri.parse('$_baseUrl/posts/$id'),
  );
  
  if (response.statusCode != 200) {
    throw HttpException('Failed to delete post: ${response.statusCode}');
  }
}
```

---

## Headers และ Authentication

### Basic Headers

```dart
class AuthenticatedApiService {
  final String _baseUrl = 'https://api.example.com';
  String? _authToken;
  
  // Set auth token
  void setToken(String token) {
    _authToken = token;
  }
  
  // Common headers
  Map<String, String> get _headers => {
    'Content-Type': 'application/json',
    'Accept': 'application/json',
    if (_authToken != null) 'Authorization': 'Bearer $_authToken',
    'X-App-Version': '1.0.0',
    'X-Platform': 'flutter',
  };
  
  Future<dynamic> get(String path) async {
    final response = await http.get(
      Uri.parse('$_baseUrl$path'),
      headers: _headers,
    );
    
    return _handleResponse(response);
  }
  
  Future<dynamic> post(String path, Map<String, dynamic> data) async {
    final response = await http.post(
      Uri.parse('$_baseUrl$path'),
      headers: _headers,
      body: jsonEncode(data),
    );
    
    return _handleResponse(response);
  }
  
  dynamic _handleResponse(http.Response response) {
    switch (response.statusCode) {
      case 200:
      case 201:
        return jsonDecode(response.body);
      case 401:
        throw UnauthorizedException('Not authenticated');
      case 403:
        throw ForbiddenException('Not authorized');
      case 404:
        throw NotFoundException('Resource not found');
      case 422:
        final error = jsonDecode(response.body);
        throw ValidationException(error['message'] ?? 'Validation failed');
      case 500:
        throw ServerException('Internal server error');
      default:
        throw HttpException('Error: ${response.statusCode}');
    }
  }
}

// Custom exceptions
class UnauthorizedException implements Exception {
  final String message;
  UnauthorizedException(this.message);
}

class ForbiddenException implements Exception {
  final String message;
  ForbiddenException(this.message);
}

class NotFoundException implements Exception {
  final String message;
  NotFoundException(this.message);
}

class ValidationException implements Exception {
  final String message;
  ValidationException(this.message);
}

class ServerException implements Exception {
  final String message;
  ServerException(this.message);
}
```

### API Key Authentication

```dart
class WeatherApiService {
  static const String _apiKey = 'YOUR_API_KEY';
  static const String _baseUrl = 'https://api.openweathermap.org/data/2.5';
  
  Future<Map<String, dynamic>> getWeather(String city) async {
    final uri = Uri.parse('$_baseUrl/weather').replace(
      queryParameters: {
        'q': city,
        'appid': _apiKey,
        'units': 'metric',
        'lang': 'th',
      },
    );
    
    final response = await http.get(uri);
    
    if (response.statusCode == 200) {
      return jsonDecode(response.body);
    } else if (response.statusCode == 401) {
      throw Exception('Invalid API key');
    } else if (response.statusCode == 404) {
      throw Exception('City not found: $city');
    }
    
    throw Exception('Weather API error: ${response.statusCode}');
  }
}
```

---

## JSON Parsing

### Manual JSON Parsing

```dart
// Model class
class Post {
  final int id;
  final int userId;
  final String title;
  final String body;
  
  const Post({
    required this.id,
    required this.userId,
    required this.title,
    required this.body,
  });
  
  // factory constructor จาก JSON
  factory Post.fromJson(Map<String, dynamic> json) {
    return Post(
      id: json['id'] as int,
      userId: json['userId'] as int,
      title: json['title'] as String,
      body: json['body'] as String,
    );
  }
  
  // แปลงกลับเป็น JSON
  Map<String, dynamic> toJson() => {
    'id': id,
    'userId': userId,
    'title': title,
    'body': body,
  };
  
  @override
  String toString() => 'Post($id: $title)';
}

// Nested JSON
class User {
  final int id;
  final String name;
  final String email;
  final Address address;
  final Company company;
  
  const User({
    required this.id,
    required this.name,
    required this.email,
    required this.address,
    required this.company,
  });
  
  factory User.fromJson(Map<String, dynamic> json) {
    return User(
      id: json['id'] as int,
      name: json['name'] as String,
      email: json['email'] as String,
      address: Address.fromJson(json['address'] as Map<String, dynamic>),
      company: Company.fromJson(json['company'] as Map<String, dynamic>),
    );
  }
}

class Address {
  final String street;
  final String city;
  final String zipcode;
  
  const Address({
    required this.street,
    required this.city,
    required this.zipcode,
  });
  
  factory Address.fromJson(Map<String, dynamic> json) {
    return Address(
      street: json['street'] as String,
      city: json['city'] as String,
      zipcode: json['zipcode'] as String,
    );
  }
}

class Company {
  final String name;
  final String catchPhrase;
  
  const Company({required this.name, required this.catchPhrase});
  
  factory Company.fromJson(Map<String, dynamic> json) {
    return Company(
      name: json['name'] as String,
      catchPhrase: json['catchPhrase'] as String,
    );
  }
}
```

### JSON กับ Nullable Fields

```dart
class Article {
  final int id;
  final String title;
  final String? subtitle;  // nullable
  final String? imageUrl;
  final DateTime publishedAt;
  final List<String> tags;
  
  const Article({
    required this.id,
    required this.title,
    this.subtitle,
    this.imageUrl,
    required this.publishedAt,
    required this.tags,
  });
  
  factory Article.fromJson(Map<String, dynamic> json) {
    return Article(
      id: json['id'] as int,
      title: json['title'] as String,
      subtitle: json['subtitle'] as String?,  // safe cast
      imageUrl: json['image_url'] as String?,
      publishedAt: DateTime.parse(json['published_at'] as String),
      tags: (json['tags'] as List<dynamic>?)
              ?.map((tag) => tag as String)
              .toList() ?? [],
    );
  }
  
  Map<String, dynamic> toJson() => {
    'id': id,
    'title': title,
    if (subtitle != null) 'subtitle': subtitle,
    if (imageUrl != null) 'image_url': imageUrl,
    'published_at': publishedAt.toIso8601String(),
    'tags': tags,
  };
}
```

---

## Error Handling

### Comprehensive Error Handling

```dart
enum ApiErrorType {
  network,
  timeout,
  unauthorized,
  forbidden,
  notFound,
  validation,
  server,
  unknown,
}

class ApiError {
  final ApiErrorType type;
  final String message;
  final int? statusCode;
  final Map<String, dynamic>? details;
  
  const ApiError({
    required this.type,
    required this.message,
    this.statusCode,
    this.details,
  });
  
  @override
  String toString() => 'ApiError($type: $message)';
}

class RobustApiService {
  static const String _baseUrl = 'https://jsonplaceholder.typicode.com';
  static const Duration _timeout = Duration(seconds: 30);
  
  Future<T> _request<T>(
    Future<http.Response> Function() requestFn,
    T Function(Map<String, dynamic>) parser,
  ) async {
    try {
      final response = await requestFn().timeout(_timeout);
      return _handleResponse(response, parser);
    } on TimeoutException {
      throw ApiError(
        type: ApiErrorType.timeout,
        message: 'Request timed out. Please check your connection.',
      );
    } on SocketException {
      throw ApiError(
        type: ApiErrorType.network,
        message: 'No internet connection.',
      );
    } on FormatException catch (e) {
      throw ApiError(
        type: ApiErrorType.unknown,
        message: 'Failed to parse response: ${e.message}',
      );
    }
  }
  
  T _handleResponse<T>(
    http.Response response,
    T Function(Map<String, dynamic>) parser,
  ) {
    final body = response.body.isNotEmpty ? jsonDecode(response.body) : null;
    
    switch (response.statusCode) {
      case 200:
      case 201:
        if (body is Map<String, dynamic>) {
          return parser(body);
        }
        throw ApiError(
          type: ApiErrorType.unknown,
          message: 'Unexpected response format',
        );
      
      case 400:
        throw ApiError(
          type: ApiErrorType.validation,
          message: body?['message'] ?? 'Bad request',
          statusCode: 400,
          details: body as Map<String, dynamic>?,
        );
      
      case 401:
        throw ApiError(
          type: ApiErrorType.unauthorized,
          message: 'Authentication required',
          statusCode: 401,
        );
      
      case 403:
        throw ApiError(
          type: ApiErrorType.forbidden,
          message: 'You don\'t have permission',
          statusCode: 403,
        );
      
      case 404:
        throw ApiError(
          type: ApiErrorType.notFound,
          message: 'Resource not found',
          statusCode: 404,
        );
      
      case 422:
        throw ApiError(
          type: ApiErrorType.validation,
          message: body?['message'] ?? 'Validation failed',
          statusCode: 422,
          details: body as Map<String, dynamic>?,
        );
      
      case 500:
      case 502:
      case 503:
        throw ApiError(
          type: ApiErrorType.server,
          message: 'Server error. Please try again later.',
          statusCode: response.statusCode,
        );
      
      default:
        throw ApiError(
          type: ApiErrorType.unknown,
          message: 'Unexpected error: ${response.statusCode}',
          statusCode: response.statusCode,
        );
    }
  }
  
  Future<Post> getPost(int id) {
    return _request(
      () => http.get(Uri.parse('$_baseUrl/posts/$id')),
      Post.fromJson,
    );
  }
}
```

---

## Loading States

### ViewModel Pattern

```dart
// State สำหรับ loading
sealed class ApiState<T> {
  const ApiState();
}

class ApiInitial<T> extends ApiState<T> {
  const ApiInitial();
}

class ApiLoading<T> extends ApiState<T> {
  const ApiLoading();
}

class ApiSuccess<T> extends ApiState<T> {
  final T data;
  const ApiSuccess(this.data);
}

class ApiError<T> extends ApiState<T> {
  final String message;
  const ApiError(this.message);
}

// ViewModel
class PostsViewModel extends ChangeNotifier {
  ApiState<List<Post>> _state = const ApiInitial();
  
  ApiState<List<Post>> get state => _state;
  
  Future<void> fetchPosts() async {
    _state = const ApiLoading();
    notifyListeners();
    
    try {
      await Future.delayed(const Duration(seconds: 1));
      final posts = [
        Post(id: 1, userId: 1, title: 'Post 1', body: 'Body 1'),
        Post(id: 2, userId: 1, title: 'Post 2', body: 'Body 2'),
      ];
      _state = ApiSuccess(posts);
    } catch (e) {
      _state = ApiError(e.toString());
    }
    
    notifyListeners();
  }
}
```

---

## Workshop: User List from JSONPlaceholder API

### Models

```dart
// models/user.dart
class UserModel {
  final int id;
  final String name;
  final String username;
  final String email;
  final String phone;
  final String website;
  final AddressModel address;
  final CompanyModel company;
  
  const UserModel({
    required this.id,
    required this.name,
    required this.username,
    required this.email,
    required this.phone,
    required this.website,
    required this.address,
    required this.company,
  });
  
  factory UserModel.fromJson(Map<String, dynamic> json) {
    return UserModel(
      id: json['id'] as int,
      name: json['name'] as String,
      username: json['username'] as String,
      email: json['email'] as String,
      phone: json['phone'] as String,
      website: json['website'] as String,
      address: AddressModel.fromJson(json['address'] as Map<String, dynamic>),
      company: CompanyModel.fromJson(json['company'] as Map<String, dynamic>),
    );
  }
}

class AddressModel {
  final String street;
  final String suite;
  final String city;
  final String zipcode;
  
  const AddressModel({
    required this.street,
    required this.suite,
    required this.city,
    required this.zipcode,
  });
  
  factory AddressModel.fromJson(Map<String, dynamic> json) {
    return AddressModel(
      street: json['street'] as String,
      suite: json['suite'] as String,
      city: json['city'] as String,
      zipcode: json['zipcode'] as String,
    );
  }
  
  String get fullAddress => '$suite $street, $city $zipcode';
}

class CompanyModel {
  final String name;
  final String catchPhrase;
  
  const CompanyModel({required this.name, required this.catchPhrase});
  
  factory CompanyModel.fromJson(Map<String, dynamic> json) {
    return CompanyModel(
      name: json['name'] as String,
      catchPhrase: json['catchPhrase'] as String,
    );
  }
}

// models/post.dart
class PostModel {
  final int id;
  final int userId;
  final String title;
  final String body;
  
  const PostModel({
    required this.id,
    required this.userId,
    required this.title,
    required this.body,
  });
  
  factory PostModel.fromJson(Map<String, dynamic> json) {
    return PostModel(
      id: json['id'] as int,
      userId: json['userId'] as int,
      title: json['title'] as String,
      body: json['body'] as String,
    );
  }
}
```

### API Service

```dart
// services/jsonplaceholder_service.dart
import 'dart:convert';
import 'dart:io';
import 'package:http/http.dart' as http;

class JsonPlaceholderService {
  static const String _baseUrl = 'https://jsonplaceholder.typicode.com';
  
  final http.Client _client;
  
  JsonPlaceholderService({http.Client? client})
      : _client = client ?? http.Client();
  
  void dispose() => _client.close();
  
  Future<List<UserModel>> getUsers() async {
    final response = await _client.get(
      Uri.parse('$_baseUrl/users'),
    );
    
    if (response.statusCode == 200) {
      final List<dynamic> data = jsonDecode(response.body);
      return data.map((json) => UserModel.fromJson(json)).toList();
    }
    
    throw Exception('Failed to fetch users: ${response.statusCode}');
  }
  
  Future<UserModel> getUser(int id) async {
    final response = await _client.get(
      Uri.parse('$_baseUrl/users/$id'),
    );
    
    if (response.statusCode == 200) {
      return UserModel.fromJson(jsonDecode(response.body));
    }
    
    throw Exception('Failed to fetch user $id');
  }
  
  Future<List<PostModel>> getUserPosts(int userId) async {
    final response = await _client.get(
      Uri.parse('$_baseUrl/posts').replace(
        queryParameters: {'userId': '$userId'},
      ),
    );
    
    if (response.statusCode == 200) {
      final List<dynamic> data = jsonDecode(response.body);
      return data.map((json) => PostModel.fromJson(json)).toList();
    }
    
    throw Exception('Failed to fetch posts for user $userId');
  }
  
  Future<PostModel> createPost(PostModel post) async {
    final response = await _client.post(
      Uri.parse('$_baseUrl/posts'),
      headers: {'Content-Type': 'application/json; charset=UTF-8'},
      body: jsonEncode({
        'userId': post.userId,
        'title': post.title,
        'body': post.body,
      }),
    );
    
    if (response.statusCode == 201) {
      return PostModel.fromJson(jsonDecode(response.body));
    }
    
    throw Exception('Failed to create post');
  }
}
```

### Main App and Screens

```dart
// main.dart
import 'package:flutter/material.dart';

void main() {
  runApp(const UserListApp());
}

class UserListApp extends StatelessWidget {
  const UserListApp({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'User List',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue),
        useMaterial3: true,
      ),
      home: const UsersScreen(),
    );
  }
}

// users_screen.dart
class UsersScreen extends StatefulWidget {
  const UsersScreen({Key? key}) : super(key: key);
  
  @override
  State<UsersScreen> createState() => _UsersScreenState();
}

class _UsersScreenState extends State<UsersScreen> {
  final _service = JsonPlaceholderService();
  
  List<UserModel> _users = [];
  List<UserModel> _filteredUsers = [];
  bool _isLoading = false;
  String _error = '';
  final _searchController = TextEditingController();
  
  @override
  void initState() {
    super.initState();
    _fetchUsers();
    _searchController.addListener(_filterUsers);
  }
  
  @override
  void dispose() {
    _service.dispose();
    _searchController.dispose();
    super.dispose();
  }
  
  void _filterUsers() {
    final query = _searchController.text.toLowerCase();
    setState(() {
      _filteredUsers = _users.where((user) {
        return user.name.toLowerCase().contains(query) ||
            user.email.toLowerCase().contains(query) ||
            user.company.name.toLowerCase().contains(query);
      }).toList();
    });
  }
  
  Future<void> _fetchUsers() async {
    setState(() {
      _isLoading = true;
      _error = '';
    });
    
    try {
      final users = await _service.getUsers();
      setState(() {
        _users = users;
        _filteredUsers = users;
        _isLoading = false;
      });
    } catch (e) {
      setState(() {
        _error = e.toString();
        _isLoading = false;
      });
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Users'),
        actions: [
          IconButton(
            onPressed: _fetchUsers,
            icon: const Icon(Icons.refresh),
          ),
        ],
        bottom: PreferredSize(
          preferredSize: const Size.fromHeight(60),
          child: Padding(
            padding: const EdgeInsets.fromLTRB(16, 0, 16, 8),
            child: TextField(
              controller: _searchController,
              decoration: InputDecoration(
                hintText: 'Search users...',
                prefixIcon: const Icon(Icons.search),
                filled: true,
                fillColor: Colors.white,
                border: OutlineInputBorder(
                  borderRadius: BorderRadius.circular(12),
                  borderSide: BorderSide.none,
                ),
                suffixIcon: _searchController.text.isNotEmpty
                    ? IconButton(
                        onPressed: () {
                          _searchController.clear();
                          _filterUsers();
                        },
                        icon: const Icon(Icons.clear),
                      )
                    : null,
              ),
            ),
          ),
        ),
      ),
      body: _buildBody(),
    );
  }
  
  Widget _buildBody() {
    if (_isLoading) {
      return const Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            CircularProgressIndicator(),
            SizedBox(height: 16),
            Text('Loading users...'),
          ],
        ),
      );
    }
    
    if (_error.isNotEmpty) {
      return Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            const Icon(Icons.error_outline, size: 64, color: Colors.red),
            const SizedBox(height: 16),
            Text(
              'Failed to load users',
              style: Theme.of(context).textTheme.titleLarge,
            ),
            const SizedBox(height: 8),
            Text(_error, textAlign: TextAlign.center),
            const SizedBox(height: 16),
            ElevatedButton.icon(
              onPressed: _fetchUsers,
              icon: const Icon(Icons.refresh),
              label: const Text('Try Again'),
            ),
          ],
        ),
      );
    }
    
    if (_filteredUsers.isEmpty) {
      return Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            const Icon(Icons.search_off, size: 64, color: Colors.grey),
            const SizedBox(height: 16),
            Text(
              'No users found for "${_searchController.text}"',
              style: const TextStyle(color: Colors.grey),
            ),
          ],
        ),
      );
    }
    
    return RefreshIndicator(
      onRefresh: _fetchUsers,
      child: ListView.builder(
        itemCount: _filteredUsers.length,
        itemBuilder: (context, index) {
          return UserListTile(
            user: _filteredUsers[index],
            onTap: () => Navigator.push(
              context,
              MaterialPageRoute(
                builder: (_) => UserDetailScreen(user: _filteredUsers[index]),
              ),
            ),
          );
        },
      ),
    );
  }
}

// User List Tile
class UserListTile extends StatelessWidget {
  final UserModel user;
  final VoidCallback onTap;
  
  const UserListTile({required this.user, required this.onTap});
  
  @override
  Widget build(BuildContext context) {
    return ListTile(
      leading: CircleAvatar(
        backgroundColor: Colors.primaries[user.id % Colors.primaries.length],
        child: Text(
          user.name[0].toUpperCase(),
          style: const TextStyle(color: Colors.white, fontWeight: FontWeight.bold),
        ),
      ),
      title: Text(user.name, style: const TextStyle(fontWeight: FontWeight.w600)),
      subtitle: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text(user.email),
          Text(
            user.company.name,
            style: const TextStyle(fontSize: 12, color: Colors.grey),
          ),
        ],
      ),
      isThreeLine: true,
      trailing: const Icon(Icons.arrow_forward_ios, size: 16),
      onTap: onTap,
    );
  }
}

// User Detail Screen
class UserDetailScreen extends StatefulWidget {
  final UserModel user;
  
  const UserDetailScreen({required this.user, Key? key}) : super(key: key);
  
  @override
  State<UserDetailScreen> createState() => _UserDetailScreenState();
}

class _UserDetailScreenState extends State<UserDetailScreen> {
  final _service = JsonPlaceholderService();
  List<PostModel> _posts = [];
  bool _isLoading = true;
  String _error = '';
  
  @override
  void initState() {
    super.initState();
    _fetchPosts();
  }
  
  @override
  void dispose() {
    _service.dispose();
    super.dispose();
  }
  
  Future<void> _fetchPosts() async {
    try {
      final posts = await _service.getUserPosts(widget.user.id);
      setState(() {
        _posts = posts;
        _isLoading = false;
      });
    } catch (e) {
      setState(() {
        _error = e.toString();
        _isLoading = false;
      });
    }
  }
  
  @override
  Widget build(BuildContext context) {
    final user = widget.user;
    
    return Scaffold(
      appBar: AppBar(
        title: Text(user.name),
      ),
      body: SingleChildScrollView(
        child: Column(
          children: [
            // User Profile Card
            Card(
              margin: const EdgeInsets.all(16),
              child: Padding(
                padding: const EdgeInsets.all(16),
                child: Column(
                  children: [
                    CircleAvatar(
                      radius: 40,
                      backgroundColor: Colors.primaries[user.id % Colors.primaries.length],
                      child: Text(
                        user.name[0],
                        style: const TextStyle(fontSize: 32, color: Colors.white),
                      ),
                    ),
                    const SizedBox(height: 12),
                    Text(user.name, style: Theme.of(context).textTheme.headlineSmall),
                    Text('@${user.username}', style: const TextStyle(color: Colors.grey)),
                    const SizedBox(height: 16),
                    _InfoRow(icon: Icons.email, text: user.email),
                    _InfoRow(icon: Icons.phone, text: user.phone),
                    _InfoRow(icon: Icons.web, text: user.website),
                    _InfoRow(icon: Icons.location_on, text: user.address.fullAddress),
                    _InfoRow(icon: Icons.business, text: user.company.name),
                    _InfoRow(icon: Icons.format_quote, text: user.company.catchPhrase, isItalic: true),
                  ],
                ),
              ),
            ),
            
            // Posts Section
            Padding(
              padding: const EdgeInsets.symmetric(horizontal: 16),
              child: Row(
                children: [
                  Text(
                    'Posts',
                    style: Theme.of(context).textTheme.titleLarge,
                  ),
                  const SizedBox(width: 8),
                  if (!_isLoading)
                    Badge(label: Text('${_posts.length}')),
                ],
              ),
            ),
            
            if (_isLoading)
              const Padding(
                padding: EdgeInsets.all(24),
                child: CircularProgressIndicator(),
              )
            else if (_error.isNotEmpty)
              Padding(
                padding: const EdgeInsets.all(16),
                child: Text('Error: $_error'),
              )
            else
              ListView.builder(
                shrinkWrap: true,
                physics: const NeverScrollableScrollPhysics(),
                itemCount: _posts.length,
                itemBuilder: (context, index) {
                  final post = _posts[index];
                  return Card(
                    margin: const EdgeInsets.symmetric(
                      horizontal: 16,
                      vertical: 4,
                    ),
                    child: ListTile(
                      title: Text(
                        post.title,
                        style: const TextStyle(fontWeight: FontWeight.w600),
                      ),
                      subtitle: Text(
                        post.body,
                        maxLines: 2,
                        overflow: TextOverflow.ellipsis,
                      ),
                    ),
                  );
                },
              ),
          ],
        ),
      ),
    );
  }
}

class _InfoRow extends StatelessWidget {
  final IconData icon;
  final String text;
  final bool isItalic;
  
  const _InfoRow({
    required this.icon,
    required this.text,
    this.isItalic = false,
  });
  
  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Row(
        children: [
          Icon(icon, size: 16, color: Colors.grey),
          const SizedBox(width: 8),
          Expanded(
            child: Text(
              text,
              style: TextStyle(
                fontStyle: isItalic ? FontStyle.italic : FontStyle.normal,
              ),
            ),
          ),
        ],
      ),
    );
  }
}
```

---

## สรุป HTTP APIs

### Checklist สำหรับ HTTP Requests
```
1. จัดการ loading state
2. จัดการ error state  
3. จัดการ no connection
4. timeout handling
5. retry mechanism
6. JSON parsing ที่ safe
7. dispose client เมื่อเสร็จ
```

### Best Practices
- ใช้ Repository Pattern แยก API calls ออกจาก UI
- Centralize error handling
- ใช้ http.Client แทน http.get/post ตรงๆ (testable)
- อย่าลืม dispose Client
- Log requests ใน development
