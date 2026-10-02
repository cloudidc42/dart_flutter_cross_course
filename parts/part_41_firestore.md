# Part 41: Firestore Database

## บทนำ

Cloud Firestore เป็นฐานข้อมูล NoSQL แบบ real-time จาก Firebase ที่ช่วยให้เราจัดเก็บและซิงค์ข้อมูลระหว่าง clients ได้แบบ real-time Firestore มีความยืดหยุ่นสูง รองรับการ query ที่ซับซ้อน และ scale ได้ดี

---

## 41.1 Collections และ Documents

### โครงสร้างข้อมูลใน Firestore

Firestore จัดเก็บข้อมูลในรูปแบบ **Collections** และ **Documents** ซึ่งต่างจากฐานข้อมูล SQL แบบดั้งเดิม

```
Firestore Database
├── users (Collection)
│   ├── user_001 (Document)
│   │   ├── name: "สมชาย"
│   │   ├── email: "somchai@example.com"
│   │   └── age: 25
│   └── user_002 (Document)
│       ├── name: "สมหญิง"
│       └── email: "somying@example.com"
├── posts (Collection)
│   ├── post_001 (Document)
│   │   ├── title: "บทความแรก"
│   │   ├── content: "..."
│   │   └── authorId: "user_001"
│   └── post_002 (Document)
└── messages (Collection)
```

### ติดตั้ง Firebase และ Firestore

```yaml
# pubspec.yaml
dependencies:
  flutter:
    sdk: flutter
  firebase_core: ^2.24.2
  cloud_firestore: ^4.14.0
  firebase_auth: ^4.16.0
```

### เริ่มต้นใช้งาน Firebase

```dart
// main.dart
import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'firebase_options.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // เริ่มต้น Firebase
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );
  
  runApp(const MyApp());
}
```

### เรียกใช้งาน Firestore Instance

```dart
import 'package:cloud_firestore/cloud_firestore.dart';

// รับ instance ของ Firestore
final FirebaseFirestore firestore = FirebaseFirestore.instance;

// อ้างอิง Collection
final CollectionReference usersCollection = 
    firestore.collection('users');

// อ้างอิง Document
final DocumentReference userDoc = 
    firestore.collection('users').doc('user_001');
```

---

## 41.2 CRUD Operations

### Create - สร้างข้อมูล

```dart
class FirestoreService {
  final FirebaseFirestore _db = FirebaseFirestore.instance;
  
  // สร้าง Document พร้อม auto-generated ID
  Future<String> createUser({
    required String name,
    required String email,
    required int age,
  }) async {
    try {
      final docRef = await _db.collection('users').add({
        'name': name,
        'email': email,
        'age': age,
        'createdAt': FieldValue.serverTimestamp(),
        'updatedAt': FieldValue.serverTimestamp(),
      });
      
      print('สร้าง user สำเร็จ ID: ${docRef.id}');
      return docRef.id;
    } catch (e) {
      print('เกิดข้อผิดพลาด: $e');
      rethrow;
    }
  }
  
  // สร้าง Document พร้อม custom ID
  Future<void> createUserWithId({
    required String userId,
    required String name,
    required String email,
  }) async {
    await _db.collection('users').doc(userId).set({
      'name': name,
      'email': email,
      'createdAt': FieldValue.serverTimestamp(),
    });
  }
  
  // Merge data - ถ้า Document มีอยู่แล้วจะ merge ไม่ทับ
  Future<void> mergeUserData({
    required String userId,
    required Map<String, dynamic> data,
  }) async {
    await _db.collection('users').doc(userId).set(
      data,
      SetOptions(merge: true),
    );
  }
}
```

### Read - อ่านข้อมูล

```dart
extension FirestoreRead on FirestoreService {
  // อ่าน Document เดียว
  Future<Map<String, dynamic>?> getUserById(String userId) async {
    try {
      final doc = await _db.collection('users').doc(userId).get();
      
      if (doc.exists) {
        return {
          'id': doc.id,
          ...doc.data() as Map<String, dynamic>,
        };
      }
      return null;
    } catch (e) {
      print('ไม่สามารถอ่านข้อมูลได้: $e');
      return null;
    }
  }
  
  // อ่าน Collection ทั้งหมด
  Future<List<Map<String, dynamic>>> getAllUsers() async {
    final snapshot = await _db.collection('users').get();
    
    return snapshot.docs.map((doc) => {
      'id': doc.id,
      ...doc.data(),
    }).toList();
  }
}
```

### Update - อัปเดตข้อมูล

```dart
extension FirestoreUpdate on FirestoreService {
  // อัปเดตบางฟิลด์
  Future<void> updateUserName({
    required String userId,
    required String newName,
  }) async {
    await _db.collection('users').doc(userId).update({
      'name': newName,
      'updatedAt': FieldValue.serverTimestamp(),
    });
  }
  
  // Increment ตัวเลข
  Future<void> incrementUserScore(String userId, int points) async {
    await _db.collection('users').doc(userId).update({
      'score': FieldValue.increment(points),
    });
  }
  
  // เพิ่มข้อมูลใน Array
  Future<void> addTagToUser(String userId, String tag) async {
    await _db.collection('users').doc(userId).update({
      'tags': FieldValue.arrayUnion([tag]),
    });
  }
  
  // ลบข้อมูลออกจาก Array
  Future<void> removeTagFromUser(String userId, String tag) async {
    await _db.collection('users').doc(userId).update({
      'tags': FieldValue.arrayRemove([tag]),
    });
  }
  
  // ลบ Field
  Future<void> deleteUserField(String userId, String fieldName) async {
    await _db.collection('users').doc(userId).update({
      fieldName: FieldValue.delete(),
    });
  }
}
```

### Delete - ลบข้อมูล

```dart
extension FirestoreDelete on FirestoreService {
  // ลบ Document
  Future<void> deleteUser(String userId) async {
    await _db.collection('users').doc(userId).delete();
  }
  
  // ลบ Collection ทั้งหมด (ต้องทำทีละ batch)
  Future<void> deleteCollection(String collectionPath) async {
    const batchSize = 100;
    
    while (true) {
      final snapshot = await _db
          .collection(collectionPath)
          .limit(batchSize)
          .get();
      
      if (snapshot.docs.isEmpty) break;
      
      final batch = _db.batch();
      for (final doc in snapshot.docs) {
        batch.delete(doc.reference);
      }
      await batch.commit();
    }
  }
}
```

---

## 41.3 Real-time Listeners (Snapshots)

### Stream ข้อมูล Document เดียว

```dart
class UserStream extends StatefulWidget {
  final String userId;
  
  const UserStream({super.key, required this.userId});
  
  @override
  State<UserStream> createState() => _UserStreamState();
}

class _UserStreamState extends State<UserStream> {
  final FirebaseFirestore _db = FirebaseFirestore.instance;
  
  @override
  Widget build(BuildContext context) {
    return StreamBuilder<DocumentSnapshot>(
      stream: _db.collection('users').doc(widget.userId).snapshots(),
      builder: (context, snapshot) {
        if (snapshot.connectionState == ConnectionState.waiting) {
          return const CircularProgressIndicator();
        }
        
        if (snapshot.hasError) {
          return Text('เกิดข้อผิดพลาด: ${snapshot.error}');
        }
        
        if (!snapshot.hasData || !snapshot.data!.exists) {
          return const Text('ไม่พบข้อมูล');
        }
        
        final userData = snapshot.data!.data() as Map<String, dynamic>;
        
        return Card(
          child: ListTile(
            title: Text(userData['name'] ?? ''),
            subtitle: Text(userData['email'] ?? ''),
          ),
        );
      },
    );
  }
}
```

### Stream Collection ทั้งหมด

```dart
class UsersListPage extends StatelessWidget {
  const UsersListPage({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('รายชื่อผู้ใช้')),
      body: StreamBuilder<QuerySnapshot>(
        stream: FirebaseFirestore.instance
            .collection('users')
            .orderBy('createdAt', descending: true)
            .snapshots(),
        builder: (context, snapshot) {
          if (snapshot.connectionState == ConnectionState.waiting) {
            return const Center(child: CircularProgressIndicator());
          }
          
          if (!snapshot.hasData || snapshot.data!.docs.isEmpty) {
            return const Center(child: Text('ยังไม่มีผู้ใช้'));
          }
          
          final users = snapshot.data!.docs;
          
          return ListView.builder(
            itemCount: users.length,
            itemBuilder: (context, index) {
              final user = users[index].data() as Map<String, dynamic>;
              final userId = users[index].id;
              
              return ListTile(
                leading: CircleAvatar(
                  child: Text(user['name']?[0] ?? '?'),
                ),
                title: Text(user['name'] ?? ''),
                subtitle: Text(user['email'] ?? ''),
                trailing: IconButton(
                  icon: const Icon(Icons.delete),
                  onPressed: () => _deleteUser(userId),
                ),
              );
            },
          );
        },
      ),
    );
  }
  
  void _deleteUser(String userId) {
    FirebaseFirestore.instance.collection('users').doc(userId).delete();
  }
}
```

### Real-time Updates พร้อม DocumentChanges

```dart
void listenToChanges() {
  FirebaseFirestore.instance
      .collection('messages')
      .orderBy('timestamp')
      .snapshots()
      .listen((snapshot) {
    for (final change in snapshot.docChanges) {
      switch (change.type) {
        case DocumentChangeType.added:
          print('เพิ่มข้อความ: ${change.doc.id}');
          break;
        case DocumentChangeType.modified:
          print('แก้ไขข้อความ: ${change.doc.id}');
          break;
        case DocumentChangeType.removed:
          print('ลบข้อความ: ${change.doc.id}');
          break;
      }
    }
  });
}
```

---

## 41.4 Queries และ Filters

### Query พื้นฐาน

```dart
class FirestoreQueries {
  final FirebaseFirestore _db = FirebaseFirestore.instance;
  
  // Filter ด้วย where
  Future<List<Map<String, dynamic>>> getUsersByAge(int minAge) async {
    final snapshot = await _db
        .collection('users')
        .where('age', isGreaterThanOrEqualTo: minAge)
        .get();
    
    return snapshot.docs
        .map((doc) => {'id': doc.id, ...doc.data()})
        .toList();
  }
  
  // Multiple conditions
  Future<List<Map<String, dynamic>>> getActiveAdultUsers() async {
    final snapshot = await _db
        .collection('users')
        .where('age', isGreaterThanOrEqualTo: 18)
        .where('isActive', isEqualTo: true)
        .get();
    
    return snapshot.docs
        .map((doc) => {'id': doc.id, ...doc.data()})
        .toList();
  }
  
  // whereIn - หลายค่า
  Future<List<Map<String, dynamic>>> getUsersByIds(
    List<String> userIds,
  ) async {
    final snapshot = await _db
        .collection('users')
        .where(FieldPath.documentId, whereIn: userIds)
        .get();
    
    return snapshot.docs
        .map((doc) => {'id': doc.id, ...doc.data()})
        .toList();
  }
  
  // Array contains
  Future<List<Map<String, dynamic>>> getUsersByTag(String tag) async {
    final snapshot = await _db
        .collection('users')
        .where('tags', arrayContains: tag)
        .get();
    
    return snapshot.docs
        .map((doc) => {'id': doc.id, ...doc.data()})
        .toList();
  }
  
  // Ordering และ Limiting
  Future<List<Map<String, dynamic>>> getTopUsers(int limit) async {
    final snapshot = await _db
        .collection('users')
        .orderBy('score', descending: true)
        .limit(limit)
        .get();
    
    return snapshot.docs
        .map((doc) => {'id': doc.id, ...doc.data()})
        .toList();
  }
}
```

### Pagination

```dart
class PaginatedUsersList extends StatefulWidget {
  const PaginatedUsersList({super.key});
  
  @override
  State<PaginatedUsersList> createState() => _PaginatedUsersListState();
}

class _PaginatedUsersListState extends State<PaginatedUsersList> {
  final FirebaseFirestore _db = FirebaseFirestore.instance;
  final List<DocumentSnapshot> _users = [];
  DocumentSnapshot? _lastDocument;
  bool _hasMore = true;
  bool _isLoading = false;
  static const int _pageSize = 10;
  
  @override
  void initState() {
    super.initState();
    _loadMore();
  }
  
  Future<void> _loadMore() async {
    if (_isLoading || !_hasMore) return;
    
    setState(() => _isLoading = true);
    
    Query query = _db
        .collection('users')
        .orderBy('createdAt', descending: true)
        .limit(_pageSize);
    
    if (_lastDocument != null) {
      query = query.startAfterDocument(_lastDocument!);
    }
    
    final snapshot = await query.get();
    
    setState(() {
      _users.addAll(snapshot.docs);
      _isLoading = false;
      
      if (snapshot.docs.length < _pageSize) {
        _hasMore = false;
      } else {
        _lastDocument = snapshot.docs.last;
      }
    });
  }
  
  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: _users.length + (_hasMore ? 1 : 0),
      itemBuilder: (context, index) {
        if (index == _users.length) {
          _loadMore();
          return const Center(child: CircularProgressIndicator());
        }
        
        final user = _users[index].data() as Map<String, dynamic>;
        return ListTile(
          title: Text(user['name'] ?? ''),
          subtitle: Text(user['email'] ?? ''),
        );
      },
    );
  }
}
```

---

## 41.5 Subcollections

### โครงสร้าง Subcollections

```
users (Collection)
└── user_001 (Document)
    ├── name: "สมชาย"
    └── posts (Subcollection)
        ├── post_001 (Document)
        │   ├── title: "..."
        │   └── comments (Subcollection)
        │       └── comment_001 (Document)
        └── post_002 (Document)
```

### ใช้งาน Subcollections

```dart
class SubcollectionService {
  final FirebaseFirestore _db = FirebaseFirestore.instance;
  
  // สร้าง post ใน subcollection ของ user
  Future<String> createPost({
    required String userId,
    required String title,
    required String content,
  }) async {
    final postRef = await _db
        .collection('users')
        .doc(userId)
        .collection('posts')
        .add({
          'title': title,
          'content': content,
          'createdAt': FieldValue.serverTimestamp(),
          'likes': 0,
        });
    
    return postRef.id;
  }
  
  // อ่าน posts ของ user
  Future<List<Map<String, dynamic>>> getUserPosts(String userId) async {
    final snapshot = await _db
        .collection('users')
        .doc(userId)
        .collection('posts')
        .orderBy('createdAt', descending: true)
        .get();
    
    return snapshot.docs
        .map((doc) => {'id': doc.id, ...doc.data()})
        .toList();
  }
  
  // เพิ่ม comment ใน subcollection ของ post
  Future<void> addComment({
    required String userId,
    required String postId,
    required String commentText,
    required String authorId,
  }) async {
    await _db
        .collection('users')
        .doc(userId)
        .collection('posts')
        .doc(postId)
        .collection('comments')
        .add({
          'text': commentText,
          'authorId': authorId,
          'createdAt': FieldValue.serverTimestamp(),
        });
  }
  
  // Stream comments แบบ real-time
  Stream<QuerySnapshot> streamComments({
    required String userId,
    required String postId,
  }) {
    return _db
        .collection('users')
        .doc(userId)
        .collection('posts')
        .doc(postId)
        .collection('comments')
        .orderBy('createdAt')
        .snapshots();
  }
}
```

---

## 41.6 Batch Operations และ Transactions

### Batch Write

```dart
Future<void> batchCreateUsers(
  List<Map<String, dynamic>> usersData,
) async {
  final db = FirebaseFirestore.instance;
  final batch = db.batch();
  
  for (final userData in usersData) {
    final docRef = db.collection('users').doc();
    batch.set(docRef, {
      ...userData,
      'createdAt': FieldValue.serverTimestamp(),
    });
  }
  
  // Execute batch (สูงสุด 500 operations)
  await batch.commit();
}
```

### Transaction

```dart
Future<void> transferPoints({
  required String fromUserId,
  required String toUserId,
  required int points,
}) async {
  final db = FirebaseFirestore.instance;
  
  await db.runTransaction((transaction) async {
    final fromUserRef = db.collection('users').doc(fromUserId);
    final toUserRef = db.collection('users').doc(toUserId);
    
    final fromUserDoc = await transaction.get(fromUserRef);
    final toUserDoc = await transaction.get(toUserRef);
    
    if (!fromUserDoc.exists || !toUserDoc.exists) {
      throw Exception('ไม่พบผู้ใช้');
    }
    
    final fromUserPoints = fromUserDoc.data()!['points'] as int;
    
    if (fromUserPoints < points) {
      throw Exception('คะแนนไม่เพียงพอ');
    }
    
    transaction.update(fromUserRef, {
      'points': FieldValue.increment(-points),
    });
    
    transaction.update(toUserRef, {
      'points': FieldValue.increment(points),
    });
  });
}
```

---

## 41.7 Security Rules

### ตัวอย่าง Security Rules

```javascript
// firestore.rules
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    
    // Helper functions
    function isAuthenticated() {
      return request.auth != null;
    }
    
    function isOwner(userId) {
      return isAuthenticated() && request.auth.uid == userId;
    }
    
    function isValidUser() {
      return request.resource.data.keys().hasAll(['name', 'email'])
          && request.resource.data.name is string
          && request.resource.data.email is string
          && request.resource.data.name.size() > 0;
    }
    
    // Users collection
    match /users/{userId} {
      allow read: if isAuthenticated();
      allow create: if isOwner(userId) && isValidUser();
      allow update: if isOwner(userId);
      allow delete: if isOwner(userId);
      
      // Posts subcollection
      match /posts/{postId} {
        allow read: if isAuthenticated();
        allow create: if isOwner(userId);
        allow update, delete: if isOwner(userId);
      }
    }
    
    // Messages collection - ทุกคนอ่านได้ แต่ต้อง login ถึงจะเขียนได้
    match /messages/{messageId} {
      allow read: if true;
      allow create: if isAuthenticated()
          && request.resource.data.userId == request.auth.uid;
      allow update, delete: if isAuthenticated()
          && resource.data.userId == request.auth.uid;
    }
  }
}
```

---

## 41.8 Workshop: Chat Messages กับ Firestore

### โครงสร้าง Chat App

```
chat_app/
├── lib/
│   ├── main.dart
│   ├── models/
│   │   └── message_model.dart
│   ├── services/
│   │   └── chat_service.dart
│   └── pages/
│       ├── chat_page.dart
│       └── login_page.dart
```

### Message Model

```dart
// lib/models/message_model.dart
import 'package:cloud_firestore/cloud_firestore.dart';

class MessageModel {
  final String id;
  final String text;
  final String senderId;
  final String senderName;
  final DateTime? timestamp;
  final String? imageUrl;
  
  MessageModel({
    required this.id,
    required this.text,
    required this.senderId,
    required this.senderName,
    this.timestamp,
    this.imageUrl,
  });
  
  factory MessageModel.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return MessageModel(
      id: doc.id,
      text: data['text'] ?? '',
      senderId: data['senderId'] ?? '',
      senderName: data['senderName'] ?? '',
      timestamp: (data['timestamp'] as Timestamp?)?.toDate(),
      imageUrl: data['imageUrl'],
    );
  }
  
  Map<String, dynamic> toMap() {
    return {
      'text': text,
      'senderId': senderId,
      'senderName': senderName,
      'timestamp': FieldValue.serverTimestamp(),
      'imageUrl': imageUrl,
    };
  }
}
```

### Chat Service

```dart
// lib/services/chat_service.dart
import 'package:cloud_firestore/cloud_firestore.dart';
import '../models/message_model.dart';

class ChatService {
  final FirebaseFirestore _db = FirebaseFirestore.instance;
  
  // Stream ข้อความแบบ real-time
  Stream<List<MessageModel>> getMessages(String chatRoomId) {
    return _db
        .collection('chatRooms')
        .doc(chatRoomId)
        .collection('messages')
        .orderBy('timestamp', descending: false)
        .snapshots()
        .map((snapshot) => snapshot.docs
            .map((doc) => MessageModel.fromFirestore(doc))
            .toList());
  }
  
  // ส่งข้อความ
  Future<void> sendMessage({
    required String chatRoomId,
    required String text,
    required String senderId,
    required String senderName,
  }) async {
    final message = MessageModel(
      id: '',
      text: text,
      senderId: senderId,
      senderName: senderName,
    );
    
    // เพิ่มข้อความ
    await _db
        .collection('chatRooms')
        .doc(chatRoomId)
        .collection('messages')
        .add(message.toMap());
    
    // อัปเดต lastMessage ใน chatRoom
    await _db.collection('chatRooms').doc(chatRoomId).update({
      'lastMessage': text,
      'lastMessageTime': FieldValue.serverTimestamp(),
      'lastSenderId': senderId,
    });
  }
  
  // ลบข้อความ
  Future<void> deleteMessage({
    required String chatRoomId,
    required String messageId,
  }) async {
    await _db
        .collection('chatRooms')
        .doc(chatRoomId)
        .collection('messages')
        .doc(messageId)
        .delete();
  }
  
  // สร้าง Chat Room ระหว่าง 2 คน
  Future<String> getOrCreateChatRoom(
    String userId1,
    String userId2,
  ) async {
    // สร้าง chatRoomId ที่ unique จาก userIds
    final ids = [userId1, userId2]..sort();
    final chatRoomId = ids.join('_');
    
    final chatRoom = await _db
        .collection('chatRooms')
        .doc(chatRoomId)
        .get();
    
    if (!chatRoom.exists) {
      await _db.collection('chatRooms').doc(chatRoomId).set({
        'members': ids,
        'createdAt': FieldValue.serverTimestamp(),
        'lastMessage': '',
      });
    }
    
    return chatRoomId;
  }
}
```

### Chat Page UI

```dart
// lib/pages/chat_page.dart
import 'package:flutter/material.dart';
import 'package:firebase_auth/firebase_auth.dart';
import '../models/message_model.dart';
import '../services/chat_service.dart';

class ChatPage extends StatefulWidget {
  final String chatRoomId;
  final String otherUserName;
  
  const ChatPage({
    super.key,
    required this.chatRoomId,
    required this.otherUserName,
  });
  
  @override
  State<ChatPage> createState() => _ChatPageState();
}

class _ChatPageState extends State<ChatPage> {
  final ChatService _chatService = ChatService();
  final TextEditingController _messageController = TextEditingController();
  final ScrollController _scrollController = ScrollController();
  
  String get currentUserId => 
      FirebaseAuth.instance.currentUser?.uid ?? '';
  String get currentUserName => 
      FirebaseAuth.instance.currentUser?.displayName ?? 'ผู้ใช้';
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.otherUserName),
        backgroundColor: Colors.blue,
        foregroundColor: Colors.white,
      ),
      body: Column(
        children: [
          // แสดงข้อความ
          Expanded(
            child: StreamBuilder<List<MessageModel>>(
              stream: _chatService.getMessages(widget.chatRoomId),
              builder: (context, snapshot) {
                if (snapshot.connectionState == ConnectionState.waiting) {
                  return const Center(child: CircularProgressIndicator());
                }
                
                if (!snapshot.hasData || snapshot.data!.isEmpty) {
                  return const Center(
                    child: Text(
                      'เริ่มการสนทนา...',
                      style: TextStyle(color: Colors.grey),
                    ),
                  );
                }
                
                final messages = snapshot.data!;
                
                // Scroll ไปที่ข้อความล่าสุดอัตโนมัติ
                WidgetsBinding.instance.addPostFrameCallback((_) {
                  if (_scrollController.hasClients) {
                    _scrollController.animateTo(
                      _scrollController.position.maxScrollExtent,
                      duration: const Duration(milliseconds: 300),
                      curve: Curves.easeOut,
                    );
                  }
                });
                
                return ListView.builder(
                  controller: _scrollController,
                  padding: const EdgeInsets.all(16),
                  itemCount: messages.length,
                  itemBuilder: (context, index) {
                    final message = messages[index];
                    final isMe = message.senderId == currentUserId;
                    
                    return _MessageBubble(
                      message: message,
                      isMe: isMe,
                      onLongPress: isMe
                          ? () => _deleteMessage(message.id)
                          : null,
                    );
                  },
                );
              },
            ),
          ),
          
          // Input Box
          _buildMessageInput(),
        ],
      ),
    );
  }
  
  Widget _buildMessageInput() {
    return Container(
      padding: const EdgeInsets.all(8),
      decoration: BoxDecoration(
        color: Colors.white,
        boxShadow: [
          BoxShadow(
            color: Colors.black.withOpacity(0.1),
            blurRadius: 4,
            offset: const Offset(0, -2),
          ),
        ],
      ),
      child: Row(
        children: [
          Expanded(
            child: TextField(
              controller: _messageController,
              decoration: InputDecoration(
                hintText: 'พิมพ์ข้อความ...',
                border: OutlineInputBorder(
                  borderRadius: BorderRadius.circular(24),
                ),
                contentPadding: const EdgeInsets.symmetric(
                  horizontal: 16,
                  vertical: 8,
                ),
              ),
              maxLines: null,
              textInputAction: TextInputAction.send,
              onSubmitted: (_) => _sendMessage(),
            ),
          ),
          const SizedBox(width: 8),
          CircleAvatar(
            backgroundColor: Colors.blue,
            child: IconButton(
              icon: const Icon(Icons.send, color: Colors.white),
              onPressed: _sendMessage,
            ),
          ),
        ],
      ),
    );
  }
  
  Future<void> _sendMessage() async {
    final text = _messageController.text.trim();
    if (text.isEmpty) return;
    
    _messageController.clear();
    
    await _chatService.sendMessage(
      chatRoomId: widget.chatRoomId,
      text: text,
      senderId: currentUserId,
      senderName: currentUserName,
    );
  }
  
  Future<void> _deleteMessage(String messageId) async {
    final confirm = await showDialog<bool>(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('ลบข้อความ'),
        content: const Text('คุณต้องการลบข้อความนี้ใช่ไหม?'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context, false),
            child: const Text('ยกเลิก'),
          ),
          TextButton(
            onPressed: () => Navigator.pop(context, true),
            child: const Text('ลบ', style: TextStyle(color: Colors.red)),
          ),
        ],
      ),
    );
    
    if (confirm == true) {
      await _chatService.deleteMessage(
        chatRoomId: widget.chatRoomId,
        messageId: messageId,
      );
    }
  }
  
  @override
  void dispose() {
    _messageController.dispose();
    _scrollController.dispose();
    super.dispose();
  }
}

// Message Bubble Widget
class _MessageBubble extends StatelessWidget {
  final MessageModel message;
  final bool isMe;
  final VoidCallback? onLongPress;
  
  const _MessageBubble({
    required this.message,
    required this.isMe,
    this.onLongPress,
  });
  
  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Row(
        mainAxisAlignment:
            isMe ? MainAxisAlignment.end : MainAxisAlignment.start,
        children: [
          if (!isMe) ...[
            CircleAvatar(
              radius: 16,
              child: Text(
                message.senderName.isNotEmpty
                    ? message.senderName[0].toUpperCase()
                    : '?',
              ),
            ),
            const SizedBox(width: 8),
          ],
          GestureDetector(
            onLongPress: onLongPress,
            child: Container(
              constraints: BoxConstraints(
                maxWidth: MediaQuery.of(context).size.width * 0.7,
              ),
              padding: const EdgeInsets.symmetric(
                horizontal: 12,
                vertical: 8,
              ),
              decoration: BoxDecoration(
                color: isMe ? Colors.blue : Colors.grey[200],
                borderRadius: BorderRadius.only(
                  topLeft: const Radius.circular(16),
                  topRight: const Radius.circular(16),
                  bottomLeft: Radius.circular(isMe ? 16 : 4),
                  bottomRight: Radius.circular(isMe ? 4 : 16),
                ),
              ),
              child: Column(
                crossAxisAlignment: isMe
                    ? CrossAxisAlignment.end
                    : CrossAxisAlignment.start,
                children: [
                  if (!isMe)
                    Text(
                      message.senderName,
                      style: const TextStyle(
                        fontSize: 11,
                        fontWeight: FontWeight.bold,
                        color: Colors.blue,
                      ),
                    ),
                  Text(
                    message.text,
                    style: TextStyle(
                      color: isMe ? Colors.white : Colors.black87,
                      fontSize: 15,
                    ),
                  ),
                  const SizedBox(height: 2),
                  Text(
                    _formatTime(message.timestamp),
                    style: TextStyle(
                      fontSize: 10,
                      color: isMe
                          ? Colors.white70
                          : Colors.black38,
                    ),
                  ),
                ],
              ),
            ),
          ),
        ],
      ),
    );
  }
  
  String _formatTime(DateTime? time) {
    if (time == null) return '';
    return '${time.hour.toString().padLeft(2, '0')}:'
        '${time.minute.toString().padLeft(2, '0')}';
  }
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- โครงสร้าง Collections และ Documents ของ Firestore
- การทำ CRUD operations
- การใช้ Real-time listeners ด้วย StreamBuilder
- การ Query และ Filter ข้อมูล
- การใช้ Subcollections
- Security Rules เบื้องต้น
- Workshop: Chat App แบบ real-time

**แบบฝึกหัดเพิ่มเติม:**
1. เพิ่มฟีเจอร์ "กำลังพิมพ์..." ใน chat app
2. เพิ่มการส่งรูปภาพในแชท
3. เพิ่ม read receipt (อ่านแล้ว/ยังไม่ได้อ่าน)
4. Implement Group Chat
