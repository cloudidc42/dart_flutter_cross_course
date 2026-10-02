# Part 65: GraphQL Integration ใน Flutter

## GraphQL คืออะไร?

GraphQL เป็น query language สำหรับ API ที่:
- ดึงเฉพาะข้อมูลที่ต้องการ
- Single endpoint
- Type-safe
- Real-time subscriptions
- Strongly typed schema

```yaml
# pubspec.yaml
dependencies:
  graphql_flutter: ^5.1.2
  gql: ^1.0.1
```

---

## 1. graphql_flutter Setup

### การตั้งค่า GraphQL Client

```dart
// graphql/client.dart
import 'package:graphql_flutter/graphql_flutter.dart';

class GraphQLConfig {
  static HttpLink createHttpLink() {
    return HttpLink(
      'https://api.example.com/graphql',
      defaultHeaders: {
        'Content-Type': 'application/json',
      },
    );
  }

  static AuthLink createAuthLink() {
    return AuthLink(
      getToken: () async {
        final token = await SecureStorageService.getAccessToken();
        return 'Bearer $token';
      },
    );
  }

  static WebSocketLink createWebSocketLink() {
    return WebSocketLink(
      'wss://api.example.com/graphql',
      config: SocketClientConfig(
        autoReconnect: true,
        initialPayload: () async {
          final token = await SecureStorageService.getAccessToken();
          return {'Authorization': 'Bearer $token'};
        },
      ),
    );
  }

  static GraphQLClient createClient() {
    final httpLink = createHttpLink();
    final authLink = createAuthLink();
    final wsLink = createWebSocketLink();

    // Split: ใช้ WebSocket สำหรับ subscriptions, HTTP สำหรับ queries/mutations
    final link = Link.split(
      (request) => request.isSubscription,
      wsLink,
      authLink.concat(httpLink),
    );

    return GraphQLClient(
      link: link,
      cache: GraphQLCache(
        store: HiveStore(), // ใช้ Hive สำหรับ persistent cache
      ),
    );
  }
}

// main.dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  await initHiveForFlutter(); // สำหรับ HiveStore
  
  final client = GraphQLConfig.createClient();
  
  runApp(
    GraphQLProvider(
      client: ValueNotifier(client),
      child: MyApp(),
    ),
  );
}
```

---

## 2. Queries

### ตัวอย่าง GraphQL Schema

```graphql
# schema.graphql
type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  createdAt: DateTime!
  tags: [String!]!
  likes: Int!
}

type User {
  id: ID!
  name: String!
  email: String!
  avatar: String
  posts: [Post!]!
}

type Query {
  posts(limit: Int, offset: Int): [Post!]!
  post(id: ID!): Post
  me: User
  searchPosts(query: String!): [Post!]!
}

type Mutation {
  createPost(input: CreatePostInput!): Post!
  updatePost(id: ID!, input: UpdatePostInput!): Post!
  deletePost(id: ID!): Boolean!
  likePost(id: ID!): Post!
}

type Subscription {
  postCreated: Post!
  postLiked(postId: ID!): Post!
}
```

### Query Definitions

```dart
// graphql/queries/post_queries.dart

// Query ดึง posts ทั้งหมด
const String getPosts = r'''
  query GetPosts($limit: Int, $offset: Int) {
    posts(limit: $limit, offset: $offset) {
      id
      title
      content
      createdAt
      likes
      tags
      author {
        id
        name
        avatar
      }
    }
  }
''';

// Query ดึง post เดียว
const String getPost = r'''
  query GetPost($id: ID!) {
    post(id: $id) {
      id
      title
      content
      createdAt
      likes
      tags
      author {
        id
        name
        email
        avatar
      }
    }
  }
''';

// Query ดึงข้อมูล user ปัจจุบัน
const String getMe = r'''
  query GetMe {
    me {
      id
      name
      email
      avatar
      posts {
        id
        title
        createdAt
      }
    }
  }
''';

// Search query
const String searchPosts = r'''
  query SearchPosts($query: String!) {
    searchPosts(query: $query) {
      id
      title
      content
      author {
        name
      }
    }
  }
''';
```

### Query Widget

```dart
// screens/posts_screen.dart
import 'package:graphql_flutter/graphql_flutter.dart';

class PostsScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('บทความ')),
      body: Query(
        options: QueryOptions(
          document: gql(getPosts),
          variables: {
            'limit': 20,
            'offset': 0,
          },
          fetchPolicy: FetchPolicy.cacheAndNetwork,
          // อัปเดตทุก 30 วินาที
          pollInterval: Duration(seconds: 30),
        ),
        builder: (result, {fetchMore, refetch}) {
          if (result.hasException) {
            return ErrorWidget(
              error: result.exception!,
              onRetry: refetch,
            );
          }

          if (result.isLoading && result.data == null) {
            return Center(child: CircularProgressIndicator());
          }

          final posts = (result.data!['posts'] as List)
              .map((p) => Post.fromJson(p))
              .toList();

          return RefreshIndicator(
            onRefresh: () async {
              await refetch!();
            },
            child: ListView.builder(
              itemCount: posts.length,
              itemBuilder: (context, index) {
                // Load more เมื่อถึงท้ายรายการ
                if (index == posts.length - 1) {
                  _loadMore(fetchMore!, posts.length);
                }

                return PostCard(post: posts[index]);
              },
            ),
          );
        },
      ),
    );
  }

  void _loadMore(FetchMore fetchMore, int currentLength) {
    fetchMore(
      FetchMoreOptions(
        variables: {
          'offset': currentLength,
        },
        updateQuery: (previousResult, fetchMoreResult) {
          if (fetchMoreResult == null) return previousResult;

          final previousPosts = previousResult!['posts'] as List;
          final newPosts = fetchMoreResult['posts'] as List;

          return {
            'posts': [...previousPosts, ...newPosts],
          };
        },
      ),
    );
  }
}

// PostCard Widget
class PostCard extends StatelessWidget {
  final Post post;

  const PostCard({required this.post});

  @override
  Widget build(BuildContext context) {
    return Card(
      margin: EdgeInsets.symmetric(horizontal: 16, vertical: 8),
      child: Padding(
        padding: EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              children: [
                CircleAvatar(
                  backgroundImage: post.author.avatar != null
                      ? NetworkImage(post.author.avatar!)
                      : null,
                  child: post.author.avatar == null
                      ? Text(post.author.name[0])
                      : null,
                ),
                SizedBox(width: 8),
                Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(post.author.name,
                        style: TextStyle(fontWeight: FontWeight.bold)),
                    Text(
                      DateFormat('d MMM y').format(post.createdAt),
                      style: TextStyle(color: Colors.grey, fontSize: 12),
                    ),
                  ],
                ),
              ],
            ),
            SizedBox(height: 12),
            Text(post.title,
                style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
            SizedBox(height: 8),
            Text(
              post.content,
              maxLines: 3,
              overflow: TextOverflow.ellipsis,
            ),
            SizedBox(height: 12),
            Row(
              children: [
                Icon(Icons.favorite_border, size: 16),
                SizedBox(width: 4),
                Text('${post.likes}'),
                Spacer(),
                ...post.tags.take(3).map(
                      (tag) => Chip(
                        label: Text(tag, style: TextStyle(fontSize: 12)),
                        materialTapTargetSize: MaterialTapTargetSize.shrinkWrap,
                      ),
                    ),
              ],
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## 3. Mutations

### Mutation Definitions

```dart
// graphql/mutations/post_mutations.dart
const String createPostMutation = r'''
  mutation CreatePost($input: CreatePostInput!) {
    createPost(input: $input) {
      id
      title
      content
      createdAt
      author {
        id
        name
      }
    }
  }
''';

const String updatePostMutation = r'''
  mutation UpdatePost($id: ID!, $input: UpdatePostInput!) {
    updatePost(id: $id, input: $input) {
      id
      title
      content
    }
  }
''';

const String deletePostMutation = r'''
  mutation DeletePost($id: ID!) {
    deletePost(id: $id)
  }
''';

const String likePostMutation = r'''
  mutation LikePost($id: ID!) {
    likePost(id: $id) {
      id
      likes
    }
  }
''';
```

### Mutation Widget

```dart
// screens/create_post_screen.dart
class CreatePostScreen extends StatefulWidget {
  @override
  _CreatePostScreenState createState() => _CreatePostScreenState();
}

class _CreatePostScreenState extends State<CreatePostScreen> {
  final _formKey = GlobalKey<FormState>();
  final _titleController = TextEditingController();
  final _contentController = TextEditingController();
  final List<String> _tags = [];

  @override
  void dispose() {
    _titleController.dispose();
    _contentController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('เขียนบทความใหม่'),
        actions: [
          Mutation(
            options: MutationOptions(
              document: gql(createPostMutation),
              onCompleted: (data) {
                ScaffoldMessenger.of(context).showSnackBar(
                  SnackBar(content: Text('บันทึกบทความสำเร็จ!')),
                );
                Navigator.pop(context);
              },
              onError: (error) {
                ScaffoldMessenger.of(context).showSnackBar(
                  SnackBar(
                    content: Text('เกิดข้อผิดพลาด: ${error?.graphqlErrors.first.message}'),
                    backgroundColor: Colors.red,
                  ),
                );
              },
              // อัปเดต cache หลัง mutation
              update: (cache, result) {
                if (result!.data != null) {
                  final newPost = result.data!['createPost'];
                  
                  // อ่าน query ปัจจุบันจาก cache
                  final request = QueryOptions(
                    document: gql(getPosts),
                    variables: {'limit': 20, 'offset': 0},
                  ).asRequest;
                  
                  final existing = cache.readQuery(request);
                  if (existing != null) {
                    // เพิ่ม post ใหม่ขึ้นต้น
                    cache.writeQuery(
                      request,
                      data: {
                        'posts': [newPost, ...existing['posts']],
                      },
                    );
                  }
                }
              },
            ),
            builder: (runMutation, result) {
              return TextButton(
                onPressed: result!.isLoading
                    ? null
                    : () => _handleSubmit(runMutation),
                child: result.isLoading
                    ? SizedBox(
                        width: 20,
                        height: 20,
                        child: CircularProgressIndicator(strokeWidth: 2),
                      )
                    : Text('เผยแพร่',
                        style: TextStyle(color: Colors.white)),
              );
            },
          ),
        ],
      ),
      body: Form(
        key: _formKey,
        child: ListView(
          padding: EdgeInsets.all(16),
          children: [
            TextFormField(
              controller: _titleController,
              decoration: InputDecoration(
                labelText: 'หัวข้อ',
                border: OutlineInputBorder(),
              ),
              validator: (v) =>
                  v?.isEmpty == true ? 'กรุณากรอกหัวข้อ' : null,
            ),
            SizedBox(height: 16),
            TextFormField(
              controller: _contentController,
              decoration: InputDecoration(
                labelText: 'เนื้อหา',
                border: OutlineInputBorder(),
              ),
              maxLines: 10,
              validator: (v) =>
                  v?.isEmpty == true ? 'กรุณากรอกเนื้อหา' : null,
            ),
            SizedBox(height: 16),
            _TagInput(
              tags: _tags,
              onTagsChanged: (tags) => setState(() => _tags.clear()..addAll(tags)),
            ),
          ],
        ),
      ),
    );
  }

  void _handleSubmit(RunMutation runMutation) {
    if (_formKey.currentState!.validate()) {
      runMutation({
        'input': {
          'title': _titleController.text,
          'content': _contentController.text,
          'tags': _tags,
        },
      });
    }
  }
}
```

---

## 4. Subscriptions

### Subscription Definitions

```dart
// graphql/subscriptions/post_subscriptions.dart
const String onPostCreated = r'''
  subscription OnPostCreated {
    postCreated {
      id
      title
      author {
        name
        avatar
      }
      createdAt
    }
  }
''';

const String onPostLiked = r'''
  subscription OnPostLiked($postId: ID!) {
    postLiked(postId: $postId) {
      id
      likes
    }
  }
''';
```

### Subscription Widget

```dart
// widgets/live_posts_widget.dart
class LivePostsWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Subscription(
      options: SubscriptionOptions(
        document: gql(onPostCreated),
      ),
      builder: (result) {
        if (result.hasException) {
          return Text('Error: ${result.exception}');
        }

        if (result.isLoading) {
          return SizedBox.shrink();
        }

        final post = Post.fromJson(result.data!['postCreated']);
        
        return AnimatedContainer(
          duration: Duration(milliseconds: 300),
          margin: EdgeInsets.all(8),
          padding: EdgeInsets.all(12),
          decoration: BoxDecoration(
            color: Colors.blue.shade50,
            borderRadius: BorderRadius.circular(8),
            border: Border.all(color: Colors.blue.shade200),
          ),
          child: Row(
            children: [
              Icon(Icons.fiber_new, color: Colors.blue),
              SizedBox(width: 8),
              Expanded(
                child: Text(
                  '${post.author.name} เพิ่งเผยแพร่: ${post.title}',
                ),
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

## 5. Caching

### Cache Policies

```dart
// การใช้ FetchPolicy ต่างๆ
Query(
  options: QueryOptions(
    document: gql(getPosts),
    // cacheFirst: อ่านจาก cache ถ้ามี ถ้าไม่มีค่อย network
    fetchPolicy: FetchPolicy.cacheFirst,
  ),
  builder: (result, {fetchMore, refetch}) => ...,
);

// networkOnly: ดึงจาก network เสมอ (ไม่ใช้ cache)
Query(
  options: QueryOptions(
    document: gql(getPosts),
    fetchPolicy: FetchPolicy.networkOnly,
  ),
  builder: ...,
);

// cacheAndNetwork: แสดง cache ก่อน แล้วอัปเดตเมื่อ network กลับมา
Query(
  options: QueryOptions(
    document: gql(getPosts),
    fetchPolicy: FetchPolicy.cacheAndNetwork,
  ),
  builder: ...,
);

// noCache: ไม่ใช้ cache เลย
Query(
  options: QueryOptions(
    document: gql(getPosts),
    fetchPolicy: FetchPolicy.noCache,
  ),
  builder: ...,
);
```

### Optimistic UI

```dart
// Optimistic response สำหรับ like action
class LikeButton extends StatelessWidget {
  final Post post;

  const LikeButton({required this.post});

  @override
  Widget build(BuildContext context) {
    return Mutation(
      options: MutationOptions(
        document: gql(likePostMutation),
        // Optimistic response - อัปเดต UI ทันทีก่อนที่ server จะตอบ
        optimisticResult: {
          'likePost': {
            '__typename': 'Post',
            'id': post.id,
            'likes': post.likes + 1,
          },
        },
        update: (cache, result) {
          // อัปเดต cache ด้วยข้อมูลจริงจาก server
          if (result!.data != null) {
            cache.writeFragment(
              Request(
                operation: Operation(
                  document: gql(r'''
                    fragment UpdateLikes on Post {
                      id
                      likes
                    }
                  '''),
                ),
                variables: {'id': post.id},
              ),
              data: result.data!['likePost'],
            );
          }
        },
      ),
      builder: (runMutation, result) {
        return IconButton(
          icon: Icon(
            Icons.favorite,
            color: Colors.red,
          ),
          onPressed: () => runMutation({'id': post.id}),
        );
      },
    );
  }
}
```

---

## Workshop: GraphQL Blog App

```dart
// Complete Blog App with GraphQL

// repositories/post_repository.dart
class PostRepository {
  final GraphQLClient _client;

  PostRepository(this._client);

  Future<List<Post>> getPosts({int limit = 20, int offset = 0}) async {
    final result = await _client.query(
      QueryOptions(
        document: gql(getPosts),
        variables: {'limit': limit, 'offset': offset},
        fetchPolicy: FetchPolicy.cacheAndNetwork,
      ),
    );

    if (result.hasException) {
      throw result.exception!;
    }

    return (result.data!['posts'] as List)
        .map((p) => Post.fromJson(p))
        .toList();
  }

  Future<Post> getPost(String id) async {
    final result = await _client.query(
      QueryOptions(
        document: gql(getPost),
        variables: {'id': id},
      ),
    );

    if (result.hasException) throw result.exception!;
    return Post.fromJson(result.data!['post']);
  }

  Future<Post> createPost({
    required String title,
    required String content,
    List<String> tags = const [],
  }) async {
    final result = await _client.mutate(
      MutationOptions(
        document: gql(createPostMutation),
        variables: {
          'input': {
            'title': title,
            'content': content,
            'tags': tags,
          },
        },
      ),
    );

    if (result.hasException) throw result.exception!;
    return Post.fromJson(result.data!['createPost']);
  }

  Stream<Post> watchNewPosts() {
    return _client.subscribe(
      SubscriptionOptions(
        document: gql(onPostCreated),
      ),
    ).where((result) => !result.hasException && result.data != null)
        .map((result) => Post.fromJson(result.data!['postCreated']));
  }
}

// providers/blog_provider.dart
class BlogProvider extends ChangeNotifier {
  final PostRepository _repository;

  List<Post> _posts = [];
  bool _isLoading = false;
  String? _error;
  int _currentPage = 0;
  bool _hasMore = true;
  StreamSubscription? _newPostsSubscription;

  List<Post> get posts => _posts;
  bool get isLoading => _isLoading;
  String? get error => _error;

  BlogProvider(GraphQLClient client)
      : _repository = PostRepository(client) {
    loadPosts();
    _watchNewPosts();
  }

  Future<void> loadPosts({bool refresh = false}) async {
    if (refresh) {
      _currentPage = 0;
      _hasMore = true;
    }

    if (!_hasMore || _isLoading) return;

    _isLoading = true;
    _error = null;
    notifyListeners();

    try {
      final newPosts = await _repository.getPosts(
        limit: 20,
        offset: _currentPage * 20,
      );

      if (refresh) {
        _posts = newPosts;
      } else {
        _posts = [..._posts, ...newPosts];
      }

      _hasMore = newPosts.length == 20;
      _currentPage++;
    } catch (e) {
      _error = 'ไม่สามารถโหลดบทความได้';
    } finally {
      _isLoading = false;
      notifyListeners();
    }
  }

  void _watchNewPosts() {
    _newPostsSubscription = _repository.watchNewPosts().listen(
      (newPost) {
        // เพิ่ม post ใหม่ขึ้นต้น
        _posts = [newPost, ..._posts];
        notifyListeners();
      },
    );
  }

  Future<void> createPost({
    required String title,
    required String content,
    List<String> tags = const [],
  }) async {
    final post = await _repository.createPost(
      title: title,
      content: content,
      tags: tags,
    );

    _posts = [post, ..._posts];
    notifyListeners();
  }

  @override
  void dispose() {
    _newPostsSubscription?.cancel();
    super.dispose();
  }
}

// screens/blog_home_screen.dart
class BlogHomeScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Blog'),
        actions: [
          IconButton(
            icon: Icon(Icons.create),
            onPressed: () => Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => CreatePostScreen()),
            ),
          ),
        ],
      ),
      body: Consumer<BlogProvider>(
        builder: (context, provider, _) {
          if (provider.isLoading && provider.posts.isEmpty) {
            return Center(child: CircularProgressIndicator());
          }

          if (provider.error != null && provider.posts.isEmpty) {
            return Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  Text(provider.error!),
                  ElevatedButton(
                    onPressed: () => provider.loadPosts(refresh: true),
                    child: Text('ลองใหม่'),
                  ),
                ],
              ),
            );
          }

          return RefreshIndicator(
            onRefresh: () => provider.loadPosts(refresh: true),
            child: ListView.builder(
              itemCount: provider.posts.length + 1,
              itemBuilder: (context, index) {
                if (index == provider.posts.length) {
                  if (provider.isLoading) {
                    return Center(
                      child: Padding(
                        padding: EdgeInsets.all(16),
                        child: CircularProgressIndicator(),
                      ),
                    );
                  }
                  return SizedBox.shrink();
                }

                if (index == provider.posts.length - 3) {
                  provider.loadPosts(); // Load more
                }

                return PostCard(
                  post: provider.posts[index],
                  onTap: () => Navigator.push(
                    context,
                    MaterialPageRoute(
                      builder: (_) => PostDetailScreen(
                        postId: provider.posts[index].id,
                      ),
                    ),
                  ),
                );
              },
            ),
          );
        },
      ),
    );
  }
}
```

---

## สรุป

GraphQL Integration ใน Flutter ประกอบด้วย:

1. **Setup** - GraphQL client พร้อม auth และ WebSocket
2. **Queries** - ดึงข้อมูลด้วย Query widget
3. **Mutations** - แก้ไขข้อมูลและอัปเดต cache
4. **Subscriptions** - Real-time data ด้วย WebSocket
5. **Caching** - FetchPolicy ต่างๆ และ Optimistic UI
6. **Repository Pattern** - แยก data layer ออกจาก UI
