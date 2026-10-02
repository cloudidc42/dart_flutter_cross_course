# Part 66: WebSocket และ Real-time Apps

## WebSocket คืออะไร?

WebSocket เป็น protocol ที่ให้การสื่อสารแบบ bidirectional (สองทาง) real-time ระหว่าง client และ server ผ่าน connection เดียว ต่างจาก HTTP ที่:
- HTTP: Client ส่ง request, Server ตอบ response (จบ)
- WebSocket: เปิด connection ค้างไว้, ทั้งสองฝ่ายส่งข้อมูลได้ตลอดเวลา

```yaml
# pubspec.yaml
dependencies:
  web_socket_channel: ^3.0.0
```

---

## 1. web_socket_channel

### การเชื่อมต่อ WebSocket พื้นฐาน

```dart
// websocket/websocket_client.dart
import 'package:web_socket_channel/web_socket_channel.dart';
import 'dart:convert';

class WebSocketClient {
  WebSocketChannel? _channel;
  final String _url;
  bool _isConnected = false;
  
  WebSocketClient(this._url);

  void connect() {
    _channel = WebSocketChannel.connect(Uri.parse(_url));
    _isConnected = true;
    
    _channel!.stream.listen(
      (message) {
        print('Received: $message');
      },
      onDone: () {
        _isConnected = false;
        print('Connection closed');
      },
      onError: (error) {
        _isConnected = false;
        print('Error: $error');
      },
    );
  }

  void send(Map<String, dynamic> data) {
    if (_isConnected && _channel != null) {
      _channel!.sink.add(json.encode(data));
    }
  }

  void disconnect() {
    _channel?.sink.close();
    _isConnected = false;
  }

  bool get isConnected => _isConnected;
}
```

### WebSocket Manager ที่สมบูรณ์

```dart
// websocket/websocket_manager.dart
import 'dart:async';
import 'dart:convert';
import 'package:web_socket_channel/web_socket_channel.dart';

enum ConnectionState { disconnected, connecting, connected, reconnecting }

class WebSocketManager {
  static final WebSocketManager _instance = WebSocketManager._();
  factory WebSocketManager() => _instance;
  WebSocketManager._();

  WebSocketChannel? _channel;
  StreamController<Map<String, dynamic>>? _messageController;
  StreamController<ConnectionState>? _stateController;

  ConnectionState _state = ConnectionState.disconnected;
  String? _url;
  Timer? _reconnectTimer;
  Timer? _pingTimer;
  int _reconnectAttempts = 0;
  static const int _maxReconnectAttempts = 5;
  static const Duration _reconnectDelay = Duration(seconds: 3);
  static const Duration _pingInterval = Duration(seconds: 30);

  Stream<Map<String, dynamic>> get messages =>
      _messageController!.stream;
  Stream<ConnectionState> get connectionState =>
      _stateController!.stream;
  ConnectionState get currentState => _state;
  bool get isConnected => _state == ConnectionState.connected;

  void initialize(String url) {
    _url = url;
    _messageController =
        StreamController<Map<String, dynamic>>.broadcast();
    _stateController = StreamController<ConnectionState>.broadcast();
  }

  Future<void> connect() async {
    if (_state == ConnectionState.connected ||
        _state == ConnectionState.connecting) {
      return;
    }

    _updateState(ConnectionState.connecting);

    try {
      final token = await SecureStorageService.getAccessToken();
      
      _channel = WebSocketChannel.connect(
        Uri.parse(_url!),
        protocols: ['v1'],
      );

      // ส่ง auth token แรก
      _channel!.sink.add(json.encode({
        'type': 'auth',
        'token': token,
      }));

      _channel!.stream.listen(
        _handleMessage,
        onDone: _handleDisconnect,
        onError: _handleError,
      );

      _updateState(ConnectionState.connected);
      _reconnectAttempts = 0;
      _startPingTimer();
    } catch (e) {
      _updateState(ConnectionState.disconnected);
      _scheduleReconnect();
    }
  }

  void _handleMessage(dynamic data) {
    try {
      final message = json.decode(data as String) as Map<String, dynamic>;
      
      // Handle pong
      if (message['type'] == 'pong') return;

      _messageController?.add(message);
    } catch (e) {
      debugPrint('Failed to parse message: $e');
    }
  }

  void _handleDisconnect() {
    _updateState(ConnectionState.disconnected);
    _stopPingTimer();
    
    if (_reconnectAttempts < _maxReconnectAttempts) {
      _scheduleReconnect();
    }
  }

  void _handleError(Object error) {
    debugPrint('WebSocket error: $error');
    _updateState(ConnectionState.disconnected);
    _stopPingTimer();
    _scheduleReconnect();
  }

  void _scheduleReconnect() {
    if (_reconnectAttempts >= _maxReconnectAttempts) {
      debugPrint('Max reconnect attempts reached');
      return;
    }

    _updateState(ConnectionState.reconnecting);
    _reconnectAttempts++;

    final delay = _reconnectDelay * _reconnectAttempts;
    _reconnectTimer = Timer(delay, () {
      debugPrint('Reconnecting... attempt $_reconnectAttempts');
      connect();
    });
  }

  void _startPingTimer() {
    _pingTimer = Timer.periodic(_pingInterval, (_) {
      send({'type': 'ping'});
    });
  }

  void _stopPingTimer() {
    _pingTimer?.cancel();
    _pingTimer = null;
  }

  void _updateState(ConnectionState newState) {
    _state = newState;
    _stateController?.add(newState);
  }

  void send(Map<String, dynamic> data) {
    if (!isConnected) {
      debugPrint('Cannot send: not connected');
      return;
    }

    _channel?.sink.add(json.encode(data));
  }

  void disconnect() {
    _reconnectTimer?.cancel();
    _stopPingTimer();
    _channel?.sink.close();
    _updateState(ConnectionState.disconnected);
  }

  void dispose() {
    disconnect();
    _messageController?.close();
    _stateController?.close();
  }
}
```

---

## 2. Real-time Chat

### Chat Models

```dart
// models/chat_message.dart
class ChatMessage {
  final String id;
  final String senderId;
  final String senderName;
  final String? senderAvatar;
  final String content;
  final DateTime timestamp;
  final MessageType type;
  final MessageStatus status;

  ChatMessage({
    required this.id,
    required this.senderId,
    required this.senderName,
    this.senderAvatar,
    required this.content,
    required this.timestamp,
    this.type = MessageType.text,
    this.status = MessageStatus.sent,
  });

  factory ChatMessage.fromJson(Map<String, dynamic> json) {
    return ChatMessage(
      id: json['id'],
      senderId: json['sender_id'],
      senderName: json['sender_name'],
      senderAvatar: json['sender_avatar'],
      content: json['content'],
      timestamp: DateTime.parse(json['timestamp']),
      type: MessageType.values.firstWhere(
        (t) => t.name == json['type'],
        orElse: () => MessageType.text,
      ),
      status: MessageStatus.values.firstWhere(
        (s) => s.name == (json['status'] ?? 'sent'),
        orElse: () => MessageStatus.sent,
      ),
    );
  }

  Map<String, dynamic> toJson() => {
    'id': id,
    'sender_id': senderId,
    'sender_name': senderName,
    'sender_avatar': senderAvatar,
    'content': content,
    'timestamp': timestamp.toIso8601String(),
    'type': type.name,
    'status': status.name,
  };

  ChatMessage copyWith({MessageStatus? status}) {
    return ChatMessage(
      id: id,
      senderId: senderId,
      senderName: senderName,
      senderAvatar: senderAvatar,
      content: content,
      timestamp: timestamp,
      type: type,
      status: status ?? this.status,
    );
  }
}

enum MessageType { text, image, file, system }
enum MessageStatus { sending, sent, delivered, read, failed }
```

### Chat Service

```dart
// services/chat_service.dart
class ChatService {
  final _wsManager = WebSocketManager();
  final StreamController<ChatMessage> _messageController =
      StreamController.broadcast();
  final StreamController<TypingEvent> _typingController =
      StreamController.broadcast();

  Stream<ChatMessage> get messages => _messageController.stream;
  Stream<TypingEvent> get typingEvents => _typingController.stream;

  void initialize() {
    _wsManager.initialize('wss://chat.example.com/ws');
    _wsManager.messages.listen(_handleWebSocketMessage);
  }

  Future<void> connect() => _wsManager.connect();

  void _handleWebSocketMessage(Map<String, dynamic> message) {
    switch (message['type']) {
      case 'message':
        final chatMessage = ChatMessage.fromJson(message['data']);
        _messageController.add(chatMessage);
        break;
      case 'typing':
        final typingEvent = TypingEvent.fromJson(message);
        _typingController.add(typingEvent);
        break;
      case 'message_status':
        // Handle read receipts
        break;
    }
  }

  Future<String> sendMessage({
    required String roomId,
    required String content,
    MessageType type = MessageType.text,
  }) async {
    final tempId = DateTime.now().millisecondsSinceEpoch.toString();

    _wsManager.send({
      'type': 'send_message',
      'data': {
        'temp_id': tempId,
        'room_id': roomId,
        'content': content,
        'message_type': type.name,
      },
    });

    return tempId;
  }

  void sendTypingIndicator(String roomId, bool isTyping) {
    _wsManager.send({
      'type': 'typing',
      'room_id': roomId,
      'is_typing': isTyping,
    });
  }

  void markAsRead(String roomId, String messageId) {
    _wsManager.send({
      'type': 'mark_read',
      'room_id': roomId,
      'message_id': messageId,
    });
  }

  void joinRoom(String roomId) {
    _wsManager.send({
      'type': 'join_room',
      'room_id': roomId,
    });
  }

  void leaveRoom(String roomId) {
    _wsManager.send({
      'type': 'leave_room',
      'room_id': roomId,
    });
  }

  void dispose() {
    _messageController.close();
    _typingController.close();
    _wsManager.disconnect();
  }
}

class TypingEvent {
  final String userId;
  final String userName;
  final String roomId;
  final bool isTyping;

  TypingEvent({
    required this.userId,
    required this.userName,
    required this.roomId,
    required this.isTyping,
  });

  factory TypingEvent.fromJson(Map<String, dynamic> json) {
    return TypingEvent(
      userId: json['user_id'],
      userName: json['user_name'],
      roomId: json['room_id'],
      isTyping: json['is_typing'],
    );
  }
}
```

---

## 3. Connection Management

### Connection Status Widget

```dart
// widgets/connection_indicator.dart
class ConnectionIndicator extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return StreamBuilder<ConnectionState>(
      stream: WebSocketManager().connectionState,
      initialData: ConnectionState.disconnected,
      builder: (context, snapshot) {
        final state = snapshot.data!;
        return _buildIndicator(state);
      },
    );
  }

  Widget _buildIndicator(ConnectionState state) {
    switch (state) {
      case ConnectionState.connected:
        return Icon(Icons.wifi, color: Colors.green, size: 16);
      case ConnectionState.connecting:
        return SizedBox(
          width: 16,
          height: 16,
          child: CircularProgressIndicator(strokeWidth: 2),
        );
      case ConnectionState.reconnecting:
        return Row(
          mainAxisSize: MainAxisSize.min,
          children: [
            SizedBox(
              width: 12,
              height: 12,
              child: CircularProgressIndicator(
                strokeWidth: 2,
                color: Colors.orange,
              ),
            ),
            SizedBox(width: 4),
            Text('กำลังเชื่อมต่อ...', style: TextStyle(fontSize: 12)),
          ],
        );
      case ConnectionState.disconnected:
        return Row(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(Icons.wifi_off, color: Colors.red, size: 16),
            SizedBox(width: 4),
            Text('ไม่มีการเชื่อมต่อ', style: TextStyle(fontSize: 12)),
          ],
        );
    }
  }
}
```

---

## Workshop: Real-time Chat App

```dart
// Complete Chat App

// providers/chat_provider.dart
class ChatProvider extends ChangeNotifier {
  final ChatService _chatService;
  final String currentUserId;
  final String roomId;

  List<ChatMessage> _messages = [];
  Set<String> _typingUsers = {};
  bool _isLoading = false;
  final _localTempMessages = <String, ChatMessage>{};

  List<ChatMessage> get messages => _messages;
  Set<String> get typingUsers => _typingUsers;
  bool get isLoading => _isLoading;
  bool get anyoneTyping =>
      _typingUsers.isNotEmpty && !_typingUsers.contains(currentUserId);

  StreamSubscription? _messageSubscription;
  StreamSubscription? _typingSubscription;

  ChatProvider({
    required this.currentUserId,
    required this.roomId,
    ChatService? chatService,
  }) : _chatService = chatService ?? ChatService() {
    _initialize();
  }

  void _initialize() {
    _chatService.initialize();

    _messageSubscription = _chatService.messages.listen(_handleNewMessage);
    _typingSubscription = _chatService.typingEvents.listen(_handleTypingEvent);

    _chatService.connect().then((_) {
      _chatService.joinRoom(roomId);
      _loadHistory();
    });
  }

  Future<void> _loadHistory() async {
    _isLoading = true;
    notifyListeners();

    try {
      final history = await ApiService().getChatHistory(roomId);
      _messages = history;
    } catch (e) {
      debugPrint('Failed to load chat history: $e');
    } finally {
      _isLoading = false;
      notifyListeners();
    }
  }

  void _handleNewMessage(ChatMessage message) {
    // ลบ temp message ถ้ามี
    _localTempMessages.remove(message.id);

    // เพิ่มหรืออัปเดต message
    final existingIndex = _messages.indexWhere(
      (m) => m.id == message.id,
    );

    if (existingIndex != -1) {
      _messages[existingIndex] = message;
    } else {
      _messages.add(message);
      _messages.sort((a, b) => a.timestamp.compareTo(b.timestamp));
    }

    // Mark as read ถ้าไม่ใช่ message ของเรา
    if (message.senderId != currentUserId) {
      _chatService.markAsRead(roomId, message.id);
    }

    notifyListeners();
  }

  void _handleTypingEvent(TypingEvent event) {
    if (event.roomId != roomId) return;

    if (event.isTyping) {
      _typingUsers.add(event.userId);
    } else {
      _typingUsers.remove(event.userId);
    }
    notifyListeners();
  }

  Future<void> sendMessage(String content) async {
    if (content.trim().isEmpty) return;

    final tempId = 'temp_${DateTime.now().millisecondsSinceEpoch}';
    final tempMessage = ChatMessage(
      id: tempId,
      senderId: currentUserId,
      senderName: 'ฉัน',
      content: content,
      timestamp: DateTime.now(),
      status: MessageStatus.sending,
    );

    // เพิ่ม optimistic message
    _messages.add(tempMessage);
    _localTempMessages[tempId] = tempMessage;
    notifyListeners();

    try {
      await _chatService.sendMessage(
        roomId: roomId,
        content: content,
      );
    } catch (e) {
      // Mark as failed
      final index = _messages.indexWhere((m) => m.id == tempId);
      if (index != -1) {
        _messages[index] = tempMessage.copyWith(status: MessageStatus.failed);
        notifyListeners();
      }
    }
  }

  Timer? _typingTimer;

  void onTypingChanged(String text) {
    if (text.isNotEmpty) {
      _chatService.sendTypingIndicator(roomId, true);

      // หยุดส่ง typing indicator หลัง 3 วินาที
      _typingTimer?.cancel();
      _typingTimer = Timer(Duration(seconds: 3), () {
        _chatService.sendTypingIndicator(roomId, false);
      });
    } else {
      _typingTimer?.cancel();
      _chatService.sendTypingIndicator(roomId, false);
    }
  }

  @override
  void dispose() {
    _typingTimer?.cancel();
    _messageSubscription?.cancel();
    _typingSubscription?.cancel();
    _chatService.leaveRoom(roomId);
    _chatService.dispose();
    super.dispose();
  }
}

// screens/chat_screen.dart
class ChatScreen extends StatefulWidget {
  final String roomId;
  final String roomName;

  const ChatScreen({required this.roomId, required this.roomName});

  @override
  _ChatScreenState createState() => _ChatScreenState();
}

class _ChatScreenState extends State<ChatScreen> {
  final _textController = TextEditingController();
  final _scrollController = ScrollController();
  late final ChatProvider _chatProvider;
  String? _currentUserId;

  @override
  void initState() {
    super.initState();
    _initializeChat();
  }

  Future<void> _initializeChat() async {
    _currentUserId = await SecureStorageService.getUserId();
    setState(() {
      _chatProvider = ChatProvider(
        currentUserId: _currentUserId!,
        roomId: widget.roomId,
      );
    });
  }

  @override
  void dispose() {
    _textController.dispose();
    _scrollController.dispose();
    _chatProvider.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    if (_currentUserId == null) {
      return Scaffold(body: Center(child: CircularProgressIndicator()));
    }

    return ChangeNotifierProvider.value(
      value: _chatProvider,
      child: Scaffold(
        appBar: AppBar(
          title: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Text(widget.roomName),
              ConnectionIndicator(),
            ],
          ),
        ),
        body: Column(
          children: [
            Expanded(
              child: Consumer<ChatProvider>(
                builder: (context, provider, _) {
                  if (provider.isLoading) {
                    return Center(child: CircularProgressIndicator());
                  }

                  WidgetsBinding.instance.addPostFrameCallback((_) {
                    _scrollToBottom();
                  });

                  return ListView.builder(
                    controller: _scrollController,
                    padding: EdgeInsets.symmetric(vertical: 8),
                    itemCount: provider.messages.length +
                        (provider.anyoneTyping ? 1 : 0),
                    itemBuilder: (context, index) {
                      if (index == provider.messages.length) {
                        return _TypingIndicator(
                          typingUsers: provider.typingUsers,
                        );
                      }

                      final message = provider.messages[index];
                      final isMe = message.senderId == _currentUserId;
                      final showAvatar = !isMe &&
                          (index == 0 ||
                              provider.messages[index - 1].senderId !=
                                  message.senderId);

                      return MessageBubble(
                        message: message,
                        isMe: isMe,
                        showAvatar: showAvatar,
                      );
                    },
                  );
                },
              ),
            ),

            // Typing indicator
            Consumer<ChatProvider>(
              builder: (context, provider, _) {
                if (!provider.anyoneTyping) return SizedBox.shrink();
                return Padding(
                  padding: EdgeInsets.symmetric(horizontal: 16, vertical: 4),
                  child: Align(
                    alignment: Alignment.centerLeft,
                    child: Text(
                      '${provider.typingUsers.length > 1 ? 'หลายคน' : 'กำลัง'}พิมพ์...',
                      style: TextStyle(
                        color: Colors.grey,
                        fontSize: 12,
                        fontStyle: FontStyle.italic,
                      ),
                    ),
                  ),
                );
              },
            ),

            // Input bar
            _MessageInputBar(
              controller: _textController,
              onTextChanged: _chatProvider.onTypingChanged,
              onSend: () {
                final text = _textController.text;
                if (text.isNotEmpty) {
                  _chatProvider.sendMessage(text);
                  _textController.clear();
                }
              },
            ),
          ],
        ),
      ),
    );
  }

  void _scrollToBottom() {
    if (_scrollController.hasClients) {
      _scrollController.animateTo(
        _scrollController.position.maxScrollExtent,
        duration: Duration(milliseconds: 200),
        curve: Curves.easeOut,
      );
    }
  }
}

// MessageBubble Widget
class MessageBubble extends StatelessWidget {
  final ChatMessage message;
  final bool isMe;
  final bool showAvatar;

  const MessageBubble({
    required this.message,
    required this.isMe,
    required this.showAvatar,
  });

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: EdgeInsets.symmetric(horizontal: 12, vertical: 2),
      child: Row(
        mainAxisAlignment:
            isMe ? MainAxisAlignment.end : MainAxisAlignment.start,
        crossAxisAlignment: CrossAxisAlignment.end,
        children: [
          if (!isMe) ...[
            SizedBox(
              width: 32,
              child: showAvatar
                  ? CircleAvatar(
                      radius: 16,
                      backgroundImage: message.senderAvatar != null
                          ? NetworkImage(message.senderAvatar!)
                          : null,
                      child: message.senderAvatar == null
                          ? Text(message.senderName[0])
                          : null,
                    )
                  : null,
            ),
            SizedBox(width: 8),
          ],
          Flexible(
            child: Column(
              crossAxisAlignment:
                  isMe ? CrossAxisAlignment.end : CrossAxisAlignment.start,
              children: [
                if (!isMe && showAvatar)
                  Text(
                    message.senderName,
                    style: TextStyle(
                      fontSize: 12,
                      color: Colors.grey[600],
                    ),
                  ),
                Container(
                  constraints: BoxConstraints(
                    maxWidth: MediaQuery.of(context).size.width * 0.7,
                  ),
                  padding: EdgeInsets.symmetric(horizontal: 12, vertical: 8),
                  decoration: BoxDecoration(
                    color: isMe ? Colors.blue : Colors.grey[200],
                    borderRadius: BorderRadius.only(
                      topLeft: Radius.circular(16),
                      topRight: Radius.circular(16),
                      bottomLeft: isMe ? Radius.circular(16) : Radius.circular(4),
                      bottomRight: isMe ? Radius.circular(4) : Radius.circular(16),
                    ),
                  ),
                  child: Text(
                    message.content,
                    style: TextStyle(
                      color: isMe ? Colors.white : Colors.black87,
                    ),
                  ),
                ),
                Row(
                  mainAxisSize: MainAxisSize.min,
                  children: [
                    Text(
                      DateFormat('HH:mm').format(message.timestamp),
                      style: TextStyle(
                        fontSize: 11,
                        color: Colors.grey,
                      ),
                    ),
                    if (isMe) ...[
                      SizedBox(width: 4),
                      _StatusIcon(status: message.status),
                    ],
                  ],
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}

class _StatusIcon extends StatelessWidget {
  final MessageStatus status;

  const _StatusIcon({required this.status});

  @override
  Widget build(BuildContext context) {
    switch (status) {
      case MessageStatus.sending:
        return SizedBox(
          width: 12,
          height: 12,
          child: CircularProgressIndicator(strokeWidth: 1.5),
        );
      case MessageStatus.sent:
        return Icon(Icons.check, size: 14, color: Colors.grey);
      case MessageStatus.delivered:
        return Icon(Icons.done_all, size: 14, color: Colors.grey);
      case MessageStatus.read:
        return Icon(Icons.done_all, size: 14, color: Colors.blue);
      case MessageStatus.failed:
        return Icon(Icons.error_outline, size: 14, color: Colors.red);
    }
  }
}

class _TypingIndicator extends StatefulWidget {
  final Set<String> typingUsers;

  const _TypingIndicator({required this.typingUsers});

  @override
  _TypingIndicatorState createState() => _TypingIndicatorState();
}

class _TypingIndicatorState extends State<_TypingIndicator>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: Duration(milliseconds: 1000),
    )..repeat();
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: EdgeInsets.symmetric(horizontal: 20, vertical: 4),
      child: Row(
        mainAxisSize: MainAxisSize.min,
        children: List.generate(3, (index) {
          return AnimatedBuilder(
            animation: _controller,
            builder: (context, _) {
              final delay = index * 0.3;
              final value =
                  ((_controller.value + delay) % 1.0 < 0.5) ? 1.0 : 0.3;
              return Opacity(
                opacity: value,
                child: Container(
                  width: 8,
                  height: 8,
                  margin: EdgeInsets.symmetric(horizontal: 2),
                  decoration: BoxDecoration(
                    color: Colors.grey,
                    shape: BoxShape.circle,
                  ),
                ),
              );
            },
          );
        }),
      ),
    );
  }
}

class _MessageInputBar extends StatelessWidget {
  final TextEditingController controller;
  final Function(String) onTextChanged;
  final VoidCallback onSend;

  const _MessageInputBar({
    required this.controller,
    required this.onTextChanged,
    required this.onSend,
  });

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: EdgeInsets.all(8),
      decoration: BoxDecoration(
        color: Colors.white,
        boxShadow: [
          BoxShadow(
            color: Colors.black12,
            blurRadius: 4,
            offset: Offset(0, -2),
          ),
        ],
      ),
      child: SafeArea(
        child: Row(
          children: [
            Expanded(
              child: TextField(
                controller: controller,
                onChanged: onTextChanged,
                decoration: InputDecoration(
                  hintText: 'พิมพ์ข้อความ...',
                  border: OutlineInputBorder(
                    borderRadius: BorderRadius.circular(24),
                    borderSide: BorderSide.none,
                  ),
                  filled: true,
                  fillColor: Colors.grey[100],
                  contentPadding: EdgeInsets.symmetric(
                    horizontal: 16,
                    vertical: 8,
                  ),
                ),
                textInputAction: TextInputAction.send,
                onSubmitted: (_) => onSend(),
                maxLines: null,
              ),
            ),
            SizedBox(width: 8),
            CircleAvatar(
              backgroundColor: Colors.blue,
              child: IconButton(
                icon: Icon(Icons.send, color: Colors.white),
                onPressed: onSend,
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## สรุป

WebSocket และ Real-time Apps ใน Flutter ประกอบด้วย:

1. **web_socket_channel** - library สำหรับ WebSocket
2. **Connection Management** - auto-reconnect, ping/pong
3. **Chat Service** - abstraction layer สำหรับ real-time messaging
4. **Optimistic UI** - แสดงข้อความทันทีก่อน server ตอบ
5. **Typing Indicators** - แสดงเมื่อมีคนกำลังพิมพ์
6. **Read Receipts** - track สถานะการอ่านข้อความ
