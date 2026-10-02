# Part 43: Push Notifications

## บทนำ

Push Notifications เป็นวิธีที่แอปสามารถส่งข้อความหาผู้ใช้ได้แม้ว่าแอปจะไม่ได้เปิดอยู่ ใน Flutter เราใช้ Firebase Cloud Messaging (FCM) สำหรับ remote notifications และ flutter_local_notifications สำหรับ local notifications

---

## 43.1 Firebase Cloud Messaging (FCM)

### ติดตั้ง Dependencies

```yaml
# pubspec.yaml
dependencies:
  flutter:
    sdk: flutter
  firebase_core: ^2.24.2
  firebase_messaging: ^14.7.9
  flutter_local_notifications: ^16.3.0
```

### ตั้งค่า Android

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<manifest>
    <!-- Permissions -->
    <uses-permission android:name="android.permission.INTERNET"/>
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>
    
    <application>
        <!-- Default Notification Channel -->
        <meta-data
            android:name="com.google.firebase.messaging.default_notification_channel_id"
            android:value="high_importance_channel" />
            
        <!-- Default Notification Icon -->
        <meta-data
            android:name="com.google.firebase.messaging.default_notification_icon"
            android:resource="@drawable/ic_notification" />
            
        <!-- Default Notification Color -->
        <meta-data
            android:name="com.google.firebase.messaging.default_notification_color"
            android:resource="@color/notification_color" />
    </application>
</manifest>
```

### ตั้งค่า iOS

```xml
<!-- ios/Runner/Info.plist -->
<key>UIBackgroundModes</key>
<array>
    <string>fetch</string>
    <string>remote-notification</string>
</array>

<key>FirebaseAppDelegateProxyEnabled</key>
<false/>
```

---

## 43.2 เริ่มต้น FCM

### Notification Service

```dart
// lib/services/notification_service.dart
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:flutter/material.dart';

// Background message handler (ต้องเป็น top-level function)
@pragma('vm:entry-point')
Future<void> firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  await Firebase.initializeApp();
  print('Background message: ${message.messageId}');
  
  // แสดง local notification สำหรับ background message
  await NotificationService.showLocalNotification(
    title: message.notification?.title ?? 'แจ้งเตือน',
    body: message.notification?.body ?? '',
    payload: message.data.toString(),
  );
}

class NotificationService {
  static final FirebaseMessaging _fcm = FirebaseMessaging.instance;
  static final FlutterLocalNotificationsPlugin _localNotifications =
      FlutterLocalNotificationsPlugin();
  
  static const String _channelId = 'high_importance_channel';
  static const String _channelName = 'การแจ้งเตือนสำคัญ';
  
  // เริ่มต้น Notification Service
  static Future<void> initialize() async {
    // ขอ permission
    await _requestPermission();
    
    // ตั้งค่า Local Notifications
    await _initializeLocalNotifications();
    
    // ตั้งค่า FCM handlers
    await _setupFCMHandlers();
    
    // ดึง FCM Token
    await _getFCMToken();
  }
  
  static Future<void> _requestPermission() async {
    final settings = await _fcm.requestPermission(
      alert: true,
      badge: true,
      sound: true,
      provisional: false,
      announcement: false,
      carPlay: false,
      criticalAlert: false,
    );
    
    print('Notification permission: ${settings.authorizationStatus}');
  }
  
  static Future<void> _initializeLocalNotifications() async {
    // Android settings
    const androidSettings = AndroidInitializationSettings(
      '@mipmap/ic_launcher',
    );
    
    // iOS settings
    const iosSettings = DarwinInitializationSettings(
      requestAlertPermission: true,
      requestBadgePermission: true,
      requestSoundPermission: true,
    );
    
    const initSettings = InitializationSettings(
      android: androidSettings,
      iOS: iosSettings,
    );
    
    await _localNotifications.initialize(
      initSettings,
      onDidReceiveNotificationResponse: _onNotificationTapped,
    );
    
    // สร้าง Notification Channel สำหรับ Android
    await _createNotificationChannel();
  }
  
  static Future<void> _createNotificationChannel() async {
    const channel = AndroidNotificationChannel(
      _channelId,
      _channelName,
      description: 'ช่องสำหรับการแจ้งเตือนสำคัญ',
      importance: Importance.high,
      playSound: true,
      enableVibration: true,
    );
    
    await _localNotifications
        .resolvePlatformSpecificImplementation<
            AndroidFlutterLocalNotificationsPlugin>()
        ?.createNotificationChannel(channel);
  }
  
  static void _onNotificationTapped(NotificationResponse response) {
    print('Notification tapped: ${response.payload}');
    // Navigate ไปยังหน้าที่เหมาะสม
    // NavigationService.navigateTo(response.payload);
  }
  
  static Future<void> _setupFCMHandlers() async {
    // Background handler
    FirebaseMessaging.onBackgroundMessage(
      firebaseMessagingBackgroundHandler,
    );
    
    // Foreground handler
    FirebaseMessaging.onMessage.listen((RemoteMessage message) {
      print('Foreground message: ${message.messageId}');
      _handleForegroundMessage(message);
    });
    
    // App opened from notification (background -> foreground)
    FirebaseMessaging.onMessageOpenedApp.listen((RemoteMessage message) {
      print('App opened from notification: ${message.messageId}');
      _handleNotificationTap(message);
    });
    
    // App launched from terminated state via notification
    final initialMessage = await _fcm.getInitialMessage();
    if (initialMessage != null) {
      print('App launched from notification: ${initialMessage.messageId}');
      _handleNotificationTap(initialMessage);
    }
  }
  
  static void _handleForegroundMessage(RemoteMessage message) {
    final notification = message.notification;
    if (notification == null) return;
    
    showLocalNotification(
      title: notification.title ?? 'แจ้งเตือน',
      body: notification.body ?? '',
      payload: message.data.toString(),
    );
  }
  
  static void _handleNotificationTap(RemoteMessage message) {
    final data = message.data;
    final type = data['type'];
    
    switch (type) {
      case 'chat':
        print('Navigate to chat: ${data['chatId']}');
        break;
      case 'order':
        print('Navigate to order: ${data['orderId']}');
        break;
      default:
        print('Navigate to home');
    }
  }
  
  static Future<String?> _getFCMToken() async {
    final token = await _fcm.getToken();
    print('FCM Token: $token');
    
    // บันทึก token ใน Firestore
    if (token != null) {
      await _saveTokenToDatabase(token);
    }
    
    // Subscribe to token refresh
    _fcm.onTokenRefresh.listen(_saveTokenToDatabase);
    
    return token;
  }
  
  static Future<void> _saveTokenToDatabase(String token) async {
    // บันทึก token ใน Firestore
    final userId = FirebaseAuth.instance.currentUser?.uid;
    if (userId != null) {
      await FirebaseFirestore.instance
          .collection('users')
          .doc(userId)
          .update({
        'fcmToken': token,
        'tokenUpdatedAt': FieldValue.serverTimestamp(),
      });
    }
  }
  
  // แสดง Local Notification
  static Future<void> showLocalNotification({
    required String title,
    required String body,
    String? payload,
    int id = 0,
  }) async {
    const androidDetails = AndroidNotificationDetails(
      _channelId,
      _channelName,
      channelDescription: 'ช่องสำหรับการแจ้งเตือนสำคัญ',
      importance: Importance.high,
      priority: Priority.high,
      showWhen: true,
      icon: '@mipmap/ic_launcher',
    );
    
    const iosDetails = DarwinNotificationDetails(
      presentAlert: true,
      presentBadge: true,
      presentSound: true,
    );
    
    const details = NotificationDetails(
      android: androidDetails,
      iOS: iosDetails,
    );
    
    await _localNotifications.show(
      id,
      title,
      body,
      details,
      payload: payload,
    );
  }
}
```

---

## 43.3 Local Notifications

### Scheduled Notifications

```dart
class LocalNotificationScheduler {
  static final FlutterLocalNotificationsPlugin _notifications =
      FlutterLocalNotificationsPlugin();
  
  // แสดง notification ทันที
  static Future<void> showImmediate({
    required String title,
    required String body,
  }) async {
    await NotificationService.showLocalNotification(
      title: title,
      body: body,
    );
  }
  
  // นัดหมาย notification ตามเวลา
  static Future<void> scheduleNotification({
    required int id,
    required String title,
    required String body,
    required DateTime scheduledDate,
    String? payload,
  }) async {
    const androidDetails = AndroidNotificationDetails(
      'scheduled_channel',
      'การแจ้งเตือนตามกำหนดการ',
      importance: Importance.high,
      priority: Priority.high,
    );
    
    const details = NotificationDetails(android: androidDetails);
    
    await _notifications.zonedSchedule(
      id,
      title,
      body,
      TZDateTime.from(scheduledDate, local),
      details,
      androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
      uiLocalNotificationDateInterpretation:
          UILocalNotificationDateInterpretation.absoluteTime,
      payload: payload,
    );
  }
  
  // Repeat notification ทุกวัน
  static Future<void> scheduleDailyNotification({
    required int id,
    required String title,
    required String body,
    required Time timeOfDay,
  }) async {
    const androidDetails = AndroidNotificationDetails(
      'daily_channel',
      'การแจ้งเตือนประจำวัน',
      importance: Importance.defaultImportance,
    );
    
    const details = NotificationDetails(android: androidDetails);
    
    await _notifications.periodicallyShow(
      id,
      title,
      body,
      RepeatInterval.daily,
      details,
      androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
    );
  }
  
  // ยกเลิก notification
  static Future<void> cancelNotification(int id) async {
    await _notifications.cancel(id);
  }
  
  // ยกเลิกทั้งหมด
  static Future<void> cancelAllNotifications() async {
    await _notifications.cancelAll();
  }
  
  // แสดง notification แบบ progress
  static Future<void> showProgressNotification({
    required int id,
    required String title,
    required int progress,
    required int maxProgress,
  }) async {
    final androidDetails = AndroidNotificationDetails(
      'progress_channel',
      'ความคืบหน้า',
      importance: Importance.low,
      priority: Priority.low,
      showProgress: true,
      maxProgress: maxProgress,
      progress: progress,
      onlyAlertOnce: true,
    );
    
    final details = NotificationDetails(android: androidDetails);
    
    await _notifications.show(
      id,
      title,
      '$progress/$maxProgress',
      details,
    );
  }
}
```

---

## 43.4 Foreground / Background / Terminated Handling

### สถานะของแอปเมื่อรับ notification

```dart
class FCMStateHandler {
  static Future<void> handleAllStates() async {
    // 1. FOREGROUND - แอปเปิดอยู่
    // FCM ไม่แสดง notification อัตโนมัติ ต้องแสดงเองผ่าน local notifications
    FirebaseMessaging.onMessage.listen((message) {
      print('FOREGROUND: ${message.notification?.title}');
      
      // แสดง local notification
      NotificationService.showLocalNotification(
        title: message.notification?.title ?? '',
        body: message.notification?.body ?? '',
        payload: message.data['route'],
      );
      
      // อัปเดต UI (badge count, in-app notification)
      _updateInAppUI(message);
    });
    
    // 2. BACKGROUND - แอปอยู่ใน background
    // FCM แสดง notification อัตโนมัติ
    // onMessageOpenedApp จะ trigger เมื่อผู้ใช้กด notification
    FirebaseMessaging.onMessageOpenedApp.listen((message) {
      print('BACKGROUND tapped: ${message.notification?.title}');
      _handleNotificationNavigation(message);
    });
    
    // 3. TERMINATED - แอปปิดอยู่
    // FCM แสดง notification อัตโนมัติ
    // getInitialMessage() จะ return message ที่เปิดแอป
    final initialMessage = 
        await FirebaseMessaging.instance.getInitialMessage();
    if (initialMessage != null) {
      print('TERMINATED: App opened from notification');
      _handleNotificationNavigation(initialMessage);
    }
  }
  
  static void _updateInAppUI(RemoteMessage message) {
    // Update badge count, show banner, etc.
    // ใช้ state management เพื่ออัปเดต UI
  }
  
  static void _handleNotificationNavigation(RemoteMessage message) {
    final data = message.data;
    
    // Navigate ตาม notification type
    switch (data['type']) {
      case 'chat':
        // Navigator.pushNamed(context, '/chat', arguments: data['chatId']);
        break;
      case 'post':
        // Navigator.pushNamed(context, '/post', arguments: data['postId']);
        break;
      case 'promo':
        // Navigator.pushNamed(context, '/promo', arguments: data['promoId']);
        break;
    }
  }
}
```

---

## 43.5 ส่ง Notification จาก Server

### ส่งผ่าน Firebase Admin SDK (Node.js)

```javascript
// functions/index.js (Firebase Cloud Functions)
const admin = require('firebase-admin');
admin.initializeApp();

// ส่ง notification ไปยัง token เดียว
async function sendToToken(token, title, body, data = {}) {
  const message = {
    notification: { title, body },
    data,
    token,
    android: {
      notification: {
        channelId: 'high_importance_channel',
        priority: 'high',
      },
    },
    apns: {
      payload: {
        aps: { contentAvailable: true },
      },
    },
  };
  
  return admin.messaging().send(message);
}

// ส่งให้หลาย tokens
async function sendToMultipleTokens(tokens, title, body, data = {}) {
  const message = {
    notification: { title, body },
    data,
    tokens,
  };
  
  const response = await admin.messaging().sendEachForMulticast(message);
  console.log(`ส่งสำเร็จ: ${response.successCount}`);
  console.log(`ส่งไม่สำเร็จ: ${response.failureCount}`);
  
  return response;
}

// ส่งให้ topic
async function sendToTopic(topic, title, body) {
  const message = {
    notification: { title, body },
    topic,
  };
  
  return admin.messaging().send(message);
}

// Cloud Function trigger เมื่อมี message ใหม่
exports.onNewMessage = functions.firestore
  .document('chatRooms/{roomId}/messages/{messageId}')
  .onCreate(async (snapshot, context) => {
    const message = snapshot.data();
    const roomId = context.params.roomId;
    
    // ดึงข้อมูล chat room
    const roomDoc = await admin.firestore()
      .collection('chatRooms')
      .doc(roomId)
      .get();
    
    const room = roomDoc.data();
    const members = room.members;
    
    // ส่ง notification ให้ทุก member ยกเว้น sender
    const tokens = [];
    for (const memberId of members) {
      if (memberId !== message.senderId) {
        const userDoc = await admin.firestore()
          .collection('users')
          .doc(memberId)
          .get();
        
        if (userDoc.data()?.fcmToken) {
          tokens.push(userDoc.data().fcmToken);
        }
      }
    }
    
    if (tokens.length > 0) {
      await sendToMultipleTokens(
        tokens,
        message.senderName,
        message.text,
        { type: 'chat', chatRoomId: roomId }
      );
    }
  });
```

---

## 43.6 Notification Channels (Android)

```dart
class NotificationChannelManager {
  static final FlutterLocalNotificationsPlugin _notifications =
      FlutterLocalNotificationsPlugin();
  
  // สร้าง channels ต่างๆ
  static Future<void> createAllChannels() async {
    final androidPlugin = _notifications
        .resolvePlatformSpecificImplementation<
            AndroidFlutterLocalNotificationsPlugin>();
    
    if (androidPlugin == null) return;
    
    // Channel สำหรับ Chat
    await androidPlugin.createNotificationChannel(
      const AndroidNotificationChannel(
        'chat_channel',
        'ข้อความแชท',
        description: 'การแจ้งเตือนข้อความใหม่จากแชท',
        importance: Importance.high,
        playSound: true,
        enableVibration: true,
        showBadge: true,
      ),
    );
    
    // Channel สำหรับ Order Updates
    await androidPlugin.createNotificationChannel(
      const AndroidNotificationChannel(
        'order_channel',
        'อัปเดตคำสั่งซื้อ',
        description: 'การแจ้งเตือนสถานะคำสั่งซื้อ',
        importance: Importance.defaultImportance,
        playSound: true,
        enableVibration: false,
      ),
    );
    
    // Channel สำหรับ Promotions (Low importance - ไม่มีเสียง)
    await androidPlugin.createNotificationChannel(
      const AndroidNotificationChannel(
        'promo_channel',
        'โปรโมชัน',
        description: 'ข้อเสนอและโปรโมชันพิเศษ',
        importance: Importance.low,
        playSound: false,
        enableVibration: false,
        showBadge: false,
      ),
    );
    
    // Silent Channel (สำหรับ background sync)
    await androidPlugin.createNotificationChannel(
      const AndroidNotificationChannel(
        'silent_channel',
        'การซิงค์ข้อมูล',
        description: 'การซิงค์ข้อมูลเบื้องหลัง',
        importance: Importance.min,
        playSound: false,
        enableVibration: false,
        showBadge: false,
      ),
    );
  }
  
  // ลบ channel
  static Future<void> deleteChannel(String channelId) async {
    final androidPlugin = _notifications
        .resolvePlatformSpecificImplementation<
            AndroidFlutterLocalNotificationsPlugin>();
    
    await androidPlugin?.deleteNotificationChannel(channelId);
  }
}
```

---

## 43.7 Workshop: Notification System

### ระบบ Notification สมบูรณ์

```dart
// notification_manager.dart
class NotificationManager extends ChangeNotifier {
  final List<AppNotification> _notifications = [];
  int _unreadCount = 0;
  
  List<AppNotification> get notifications => 
      List.unmodifiable(_notifications);
  int get unreadCount => _unreadCount;
  
  // เพิ่ม notification
  void addNotification(AppNotification notification) {
    _notifications.insert(0, notification);
    if (!notification.isRead) {
      _unreadCount++;
    }
    notifyListeners();
  }
  
  // Mark as read
  void markAsRead(String notificationId) {
    final index = _notifications.indexWhere(
      (n) => n.id == notificationId,
    );
    
    if (index != -1 && !_notifications[index].isRead) {
      _notifications[index] = _notifications[index].copyWith(isRead: true);
      _unreadCount = (_unreadCount - 1).clamp(0, _notifications.length);
      notifyListeners();
    }
  }
  
  // Mark all as read
  void markAllAsRead() {
    for (int i = 0; i < _notifications.length; i++) {
      if (!_notifications[i].isRead) {
        _notifications[i] = _notifications[i].copyWith(isRead: true);
      }
    }
    _unreadCount = 0;
    notifyListeners();
  }
  
  // ลบ notification
  void removeNotification(String notificationId) {
    final index = _notifications.indexWhere(
      (n) => n.id == notificationId,
    );
    
    if (index != -1) {
      if (!_notifications[index].isRead) {
        _unreadCount = (_unreadCount - 1).clamp(0, _notifications.length);
      }
      _notifications.removeAt(index);
      notifyListeners();
    }
  }
}

// Notification Model
class AppNotification {
  final String id;
  final String title;
  final String body;
  final NotificationType type;
  final DateTime createdAt;
  final bool isRead;
  final Map<String, dynamic>? data;
  
  const AppNotification({
    required this.id,
    required this.title,
    required this.body,
    required this.type,
    required this.createdAt,
    this.isRead = false,
    this.data,
  });
  
  AppNotification copyWith({bool? isRead}) {
    return AppNotification(
      id: id,
      title: title,
      body: body,
      type: type,
      createdAt: createdAt,
      isRead: isRead ?? this.isRead,
      data: data,
    );
  }
}

enum NotificationType {
  chat,
  order,
  promotion,
  system,
}

// Notification Center UI
class NotificationCenterPage extends StatelessWidget {
  const NotificationCenterPage({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('การแจ้งเตือน'),
        actions: [
          Consumer<NotificationManager>(
            builder: (context, manager, _) {
              if (manager.unreadCount == 0) return const SizedBox();
              return TextButton(
                onPressed: manager.markAllAsRead,
                child: const Text(
                  'อ่านทั้งหมด',
                  style: TextStyle(color: Colors.white),
                ),
              );
            },
          ),
        ],
      ),
      body: Consumer<NotificationManager>(
        builder: (context, manager, _) {
          if (manager.notifications.isEmpty) {
            return const Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  Icon(Icons.notifications_off, size: 64, color: Colors.grey),
                  SizedBox(height: 16),
                  Text(
                    'ไม่มีการแจ้งเตือน',
                    style: TextStyle(
                      fontSize: 18,
                      color: Colors.grey,
                    ),
                  ),
                ],
              ),
            );
          }
          
          return ListView.builder(
            itemCount: manager.notifications.length,
            itemBuilder: (context, index) {
              final notification = manager.notifications[index];
              return _NotificationItem(
                notification: notification,
                onTap: () => manager.markAsRead(notification.id),
                onDismiss: () => manager.removeNotification(notification.id),
              );
            },
          );
        },
      ),
    );
  }
}

class _NotificationItem extends StatelessWidget {
  final AppNotification notification;
  final VoidCallback onTap;
  final VoidCallback onDismiss;
  
  const _NotificationItem({
    required this.notification,
    required this.onTap,
    required this.onDismiss,
  });
  
  IconData get _icon {
    switch (notification.type) {
      case NotificationType.chat:
        return Icons.chat;
      case NotificationType.order:
        return Icons.shopping_bag;
      case NotificationType.promotion:
        return Icons.local_offer;
      case NotificationType.system:
        return Icons.info;
    }
  }
  
  Color get _iconColor {
    switch (notification.type) {
      case NotificationType.chat:
        return Colors.blue;
      case NotificationType.order:
        return Colors.orange;
      case NotificationType.promotion:
        return Colors.red;
      case NotificationType.system:
        return Colors.grey;
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return Dismissible(
      key: Key(notification.id),
      direction: DismissDirection.endToStart,
      background: Container(
        alignment: Alignment.centerRight,
        padding: const EdgeInsets.only(right: 16),
        color: Colors.red,
        child: const Icon(Icons.delete, color: Colors.white),
      ),
      onDismissed: (_) => onDismiss(),
      child: InkWell(
        onTap: onTap,
        child: Container(
          color: notification.isRead ? null : Colors.blue.withOpacity(0.05),
          padding: const EdgeInsets.all(16),
          child: Row(
            children: [
              CircleAvatar(
                backgroundColor: _iconColor.withOpacity(0.1),
                child: Icon(_icon, color: _iconColor),
              ),
              const SizedBox(width: 12),
              Expanded(
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      notification.title,
                      style: TextStyle(
                        fontWeight: notification.isRead
                            ? FontWeight.normal
                            : FontWeight.bold,
                      ),
                    ),
                    const SizedBox(height: 4),
                    Text(
                      notification.body,
                      style: const TextStyle(
                        color: Colors.grey,
                        fontSize: 13,
                      ),
                      maxLines: 2,
                      overflow: TextOverflow.ellipsis,
                    ),
                    const SizedBox(height: 4),
                    Text(
                      _formatTime(notification.createdAt),
                      style: const TextStyle(
                        fontSize: 11,
                        color: Colors.grey,
                      ),
                    ),
                  ],
                ),
              ),
              if (!notification.isRead)
                Container(
                  width: 8,
                  height: 8,
                  decoration: const BoxDecoration(
                    shape: BoxShape.circle,
                    color: Colors.blue,
                  ),
                ),
            ],
          ),
        ),
      ),
    );
  }
  
  String _formatTime(DateTime time) {
    final now = DateTime.now();
    final diff = now.difference(time);
    
    if (diff.inMinutes < 1) return 'เมื่อสักครู่';
    if (diff.inHours < 1) return '${diff.inMinutes} นาทีที่แล้ว';
    if (diff.inDays < 1) return '${diff.inHours} ชั่วโมงที่แล้ว';
    if (diff.inDays < 7) return '${diff.inDays} วันที่แล้ว';
    return '${time.day}/${time.month}/${time.year}';
  }
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- การตั้งค่า Firebase Cloud Messaging (FCM)
- การจัดการ Foreground/Background/Terminated notifications
- Local Notifications พร้อม scheduling
- Notification Channels สำหรับ Android
- Workshop: Notification Center สมบูรณ์

**แบบฝึกหัดเพิ่มเติม:**
1. เพิ่ม notification badge บน app icon
2. Implement in-app notification banner
3. สร้าง notification settings ให้ผู้ใช้เลือก type ที่ต้องการรับ
4. เพิ่ม notification history ใน Firestore
