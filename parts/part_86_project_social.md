# Part 86: Real-World Project: Social Media App - โปรเจกต์จริง: แอป Social Media

## บทนำ

สร้าง Social Media App ที่มีฟีเจอร์ครบครัน ตั้งแต่ user profiles, feed, comments, likes ไปจนถึง real-time notifications

## สถาปัตยกรรม

```
social_app/
├── lib/
│   ├── features/
│   │   ├── auth/
│   │   ├── profile/
│   │   ├── feed/
│   │   ├── post/
│   │   └── notifications/
│   └── shared/
```

## 1. User Profiles

### User Entity

```dart
// lib/features/profile/domain/entities/user_profile.dart
import 'package:equatable/equatable.dart';

class UserProfile extends Equatable {
  final String id;
  final String username;
  final String displayName;
  final String bio;
  final String? avatarUrl;
  final String? coverUrl;
  final int followerCount;
  final int followingCount;
  final int postCount;
  final bool isVerified;
  final bool isPrivate;
  final bool isFollowing; // current user following this user?
  final bool isFollowedBy; // this user following current user?
  final DateTime joinedAt;
  final List<String> interests;

  const UserProfile({
    required this.id,
    required this.username,
    required this.displayName,
    required this.bio,
    this.avatarUrl,
    this.coverUrl,
    this.followerCount = 0,
    this.followingCount = 0,
    this.postCount = 0,
    this.isVerified = false,
    this.isPrivate = false,
    this.isFollowing = false,
    this.isFollowedBy = false,
    required this.joinedAt,
    this.interests = const [],
  });

  @override
  List<Object?> get props => [id, username, isFollowing];
}
```

### Profile Screen

```dart
// lib/features/profile/presentation/screens/profile_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:cached_network_image/cached_network_image.dart';

class ProfileScreen extends StatelessWidget {
  final String userId;

  const ProfileScreen({super.key, required this.userId});

  @override
  Widget build(BuildContext context) {
    return BlocBuilder<ProfileBloc, ProfileState>(
      builder: (context, state) {
        if (state is ProfileLoaded) {
          return _buildProfile(context, state.profile);
        }
        return const Scaffold(
          body: Center(child: CircularProgressIndicator()),
        );
      },
    );
  }

  Widget _buildProfile(BuildContext context, UserProfile profile) {
    return Scaffold(
      body: CustomScrollView(
        slivers: [
          // Cover image + Avatar
          SliverAppBar(
            expandedHeight: 250,
            pinned: true,
            flexibleSpace: FlexibleSpaceBar(
              background: Stack(
                clipBehavior: Clip.none,
                children: [
                  // Cover photo
                  SizedBox(
                    width: double.infinity,
                    height: 200,
                    child: profile.coverUrl != null
                        ? CachedNetworkImage(
                            imageUrl: profile.coverUrl!,
                            fit: BoxFit.cover,
                          )
                        : Container(
                            decoration: BoxDecoration(
                              gradient: LinearGradient(
                                colors: [
                                  Colors.blue.shade400,
                                  Colors.purple.shade400,
                                ],
                              ),
                            ),
                          ),
                  ),

                  // Avatar
                  Positioned(
                    bottom: -40,
                    left: 16,
                    child: Container(
                      decoration: BoxDecoration(
                        shape: BoxShape.circle,
                        border: Border.all(
                          color: Theme.of(context).colorScheme.background,
                          width: 4,
                        ),
                      ),
                      child: CircleAvatar(
                        radius: 48,
                        backgroundImage: profile.avatarUrl != null
                            ? CachedNetworkImageProvider(profile.avatarUrl!)
                            : null,
                        child: profile.avatarUrl == null
                            ? Text(
                                profile.displayName[0].toUpperCase(),
                                style: const TextStyle(fontSize: 32),
                              )
                            : null,
                      ),
                    ),
                  ),
                ],
              ),
            ),
          ),

          // Profile info
          SliverToBoxAdapter(
            child: Padding(
              padding: const EdgeInsets.fromLTRB(16, 56, 16, 0),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Row(
                    children: [
                      Expanded(
                        child: Column(
                          crossAxisAlignment: CrossAxisAlignment.start,
                          children: [
                            Row(
                              children: [
                                Text(
                                  profile.displayName,
                                  style: Theme.of(context)
                                      .textTheme
                                      .headlineSmall
                                      ?.copyWith(
                                        fontWeight: FontWeight.bold,
                                      ),
                                ),
                                if (profile.isVerified) ...[
                                  const SizedBox(width: 4),
                                  const Icon(
                                    Icons.verified,
                                    color: Colors.blue,
                                    size: 20,
                                  ),
                                ],
                              ],
                            ),
                            Text(
                              '@${profile.username}',
                              style: const TextStyle(color: Colors.grey),
                            ),
                          ],
                        ),
                      ),
                      _buildFollowButton(context, profile),
                    ],
                  ),
                  const SizedBox(height: 12),
                  if (profile.bio.isNotEmpty)
                    Text(profile.bio),
                  const SizedBox(height: 16),
                  _buildStats(context, profile),
                ],
              ),
            ),
          ),

          // Posts grid
          SliverPadding(
            padding: const EdgeInsets.only(top: 16),
            sliver: SliverGrid(
              delegate: SliverChildBuilderDelegate(
                (context, index) => _buildPostThumbnail(index),
                childCount: profile.postCount,
              ),
              gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
                crossAxisCount: 3,
                crossAxisSpacing: 2,
                mainAxisSpacing: 2,
              ),
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildFollowButton(BuildContext context, UserProfile profile) {
    if (profile.id == 'currentUserId') {
      return OutlinedButton(
        onPressed: () => context.push('/edit-profile'),
        child: const Text('แก้ไขโปรไฟล์'),
      );
    }

    return FilledButton(
      onPressed: () => context.read<ProfileBloc>().add(
        profile.isFollowing
            ? UnfollowUserEvent(userId: profile.id)
            : FollowUserEvent(userId: profile.id),
      ),
      style: FilledButton.styleFrom(
        backgroundColor:
            profile.isFollowing ? Colors.grey[200] : null,
        foregroundColor:
            profile.isFollowing ? Colors.black : null,
      ),
      child: Text(profile.isFollowing ? 'กำลังติดตาม' : 'ติดตาม'),
    );
  }

  Widget _buildStats(BuildContext context, UserProfile profile) {
    return Row(
      children: [
        _buildStatItem(context, '${profile.postCount}', 'โพสต์'),
        const SizedBox(width: 24),
        InkWell(
          onTap: () => context.push('/followers/${profile.id}'),
          child: _buildStatItem(
              context, '${_formatCount(profile.followerCount)}', 'ผู้ติดตาม'),
        ),
        const SizedBox(width: 24),
        InkWell(
          onTap: () => context.push('/following/${profile.id}'),
          child: _buildStatItem(
              context, '${_formatCount(profile.followingCount)}', 'กำลังติดตาม'),
        ),
      ],
    );
  }

  Widget _buildStatItem(BuildContext context, String value, String label) {
    return Column(
      children: [
        Text(
          value,
          style: Theme.of(context).textTheme.titleMedium?.copyWith(
                fontWeight: FontWeight.bold,
              ),
        ),
        Text(label, style: const TextStyle(color: Colors.grey)),
      ],
    );
  }

  Widget _buildPostThumbnail(int index) {
    return Container(
      color: Colors.grey[200],
      child: const Center(child: Icon(Icons.image)),
    );
  }

  String _formatCount(int count) {
    if (count >= 1000000) return '${(count / 1000000).toStringAsFixed(1)}M';
    if (count >= 1000) return '${(count / 1000).toStringAsFixed(1)}K';
    return '$count';
  }
}
```

## 2. Feed with Posts

### Post Entity

```dart
// lib/features/feed/domain/entities/post.dart
import 'package:equatable/equatable.dart';

class Post extends Equatable {
  final String id;
  final String authorId;
  final String authorName;
  final String authorUsername;
  final String? authorAvatarUrl;
  final bool authorIsVerified;
  final String content;
  final List<PostMedia> media;
  final int likeCount;
  final int commentCount;
  final int shareCount;
  final bool isLiked;
  final bool isBookmarked;
  final DateTime createdAt;
  final PostVisibility visibility;
  final List<String> tags;
  final String? locationName;

  const Post({
    required this.id,
    required this.authorId,
    required this.authorName,
    required this.authorUsername,
    this.authorAvatarUrl,
    this.authorIsVerified = false,
    required this.content,
    this.media = const [],
    this.likeCount = 0,
    this.commentCount = 0,
    this.shareCount = 0,
    this.isLiked = false,
    this.isBookmarked = false,
    required this.createdAt,
    this.visibility = PostVisibility.public,
    this.tags = const [],
    this.locationName,
  });

  @override
  List<Object?> get props => [id, isLiked, likeCount, isBookmarked];
}

enum PostVisibility { public, followers, private }

class PostMedia {
  final String url;
  final PostMediaType type;
  final double? aspectRatio;
  final String? altText;

  const PostMedia({
    required this.url,
    required this.type,
    this.aspectRatio,
    this.altText,
  });
}

enum PostMediaType { image, video, gif }
```

### Feed Screen

```dart
// lib/features/feed/presentation/screens/feed_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

class FeedScreen extends StatefulWidget {
  const FeedScreen({super.key});

  @override
  State<FeedScreen> createState() => _FeedScreenState();
}

class _FeedScreenState extends State<FeedScreen> {
  final _scrollController = ScrollController();

  @override
  void initState() {
    super.initState();
    _scrollController.addListener(_onScroll);
    context.read<FeedBloc>().add(const LoadFeedEvent());
  }

  void _onScroll() {
    if (_scrollController.position.pixels >=
        _scrollController.position.maxScrollExtent - 200) {
      context.read<FeedBloc>().add(const LoadMoreFeedEvent());
    }
  }

  @override
  void dispose() {
    _scrollController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Feed'),
        actions: [
          IconButton(
            icon: const Icon(Icons.add_box_outlined),
            onPressed: () => context.push('/create-post'),
          ),
          IconButton(
            icon: const Icon(Icons.notifications_outlined),
            onPressed: () => context.push('/notifications'),
          ),
        ],
      ),
      body: RefreshIndicator(
        onRefresh: () async {
          context.read<FeedBloc>().add(const RefreshFeedEvent());
        },
        child: BlocBuilder<FeedBloc, FeedState>(
          builder: (context, state) {
            if (state is FeedLoading) {
              return const Center(child: CircularProgressIndicator());
            }

            if (state is FeedLoaded) {
              return ListView.separated(
                controller: _scrollController,
                itemCount: state.posts.length + (state.hasMore ? 1 : 0),
                separatorBuilder: (context, index) =>
                    const Divider(height: 1),
                itemBuilder: (context, index) {
                  if (index == state.posts.length) {
                    return const Center(
                      child: Padding(
                        padding: EdgeInsets.all(16),
                        child: CircularProgressIndicator(),
                      ),
                    );
                  }
                  return PostCard(post: state.posts[index]);
                },
              );
            }

            return const SizedBox();
          },
        ),
      ),
    );
  }
}
```

## 3. Comments and Likes

### Post Card with Actions

```dart
// lib/features/feed/presentation/widgets/post_card.dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

class PostCard extends StatelessWidget {
  final Post post;

  const PostCard({super.key, required this.post});

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        // Header
        _buildHeader(context),

        // Content
        Padding(
          padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 8),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Text(post.content),
              if (post.tags.isNotEmpty) ...[
                const SizedBox(height: 4),
                Wrap(
                  spacing: 4,
                  children: post.tags
                      .map((tag) => InkWell(
                            onTap: () => context.push('/hashtag/$tag'),
                            child: Text(
                              '#$tag',
                              style: TextStyle(
                                color:
                                    Theme.of(context).colorScheme.primary,
                              ),
                            ),
                          ))
                      .toList(),
                ),
              ],
            ],
          ),
        ),

        // Media
        if (post.media.isNotEmpty)
          PostMediaView(mediaList: post.media),

        // Actions
        _buildActions(context),
      ],
    );
  }

  Widget _buildHeader(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.all(12),
      child: Row(
        children: [
          InkWell(
            onTap: () => context.push('/profile/${post.authorId}'),
            child: CircleAvatar(
              radius: 20,
              backgroundImage: post.authorAvatarUrl != null
                  ? CachedNetworkImageProvider(post.authorAvatarUrl!)
                  : null,
              child: post.authorAvatarUrl == null
                  ? Text(post.authorName[0].toUpperCase())
                  : null,
            ),
          ),
          const SizedBox(width: 8),
          Expanded(
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Row(
                  children: [
                    InkWell(
                      onTap: () =>
                          context.push('/profile/${post.authorId}'),
                      child: Text(
                        post.authorName,
                        style: const TextStyle(fontWeight: FontWeight.bold),
                      ),
                    ),
                    if (post.authorIsVerified)
                      const Icon(
                        Icons.verified,
                        size: 14,
                        color: Colors.blue,
                      ),
                  ],
                ),
                Text(
                  _formatTime(post.createdAt),
                  style: const TextStyle(
                    color: Colors.grey,
                    fontSize: 12,
                  ),
                ),
              ],
            ),
          ),
          IconButton(
            icon: const Icon(Icons.more_horiz),
            onPressed: () => _showPostMenu(context),
          ),
        ],
      ),
    );
  }

  Widget _buildActions(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 4),
      child: Row(
        children: [
          _buildLikeButton(context),
          _buildCommentButton(context),
          _buildShareButton(context),
          const Spacer(),
          _buildBookmarkButton(context),
        ],
      ),
    );
  }

  Widget _buildLikeButton(BuildContext context) {
    return InkWell(
      onTap: () => context.read<FeedBloc>().add(
        post.isLiked
            ? UnlikePostEvent(postId: post.id)
            : LikePostEvent(postId: post.id),
      ),
      child: Padding(
        padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 12),
        child: Row(
          children: [
            AnimatedSwitcher(
              duration: const Duration(milliseconds: 200),
              transitionBuilder: (child, animation) => ScaleTransition(
                scale: animation,
                child: child,
              ),
              child: Icon(
                post.isLiked ? Icons.favorite : Icons.favorite_border,
                key: ValueKey(post.isLiked),
                color: post.isLiked ? Colors.red : null,
              ),
            ),
            const SizedBox(width: 4),
            Text('${_formatCount(post.likeCount)}'),
          ],
        ),
      ),
    );
  }

  Widget _buildCommentButton(BuildContext context) {
    return InkWell(
      onTap: () => context.push('/post/${post.id}/comments'),
      child: Padding(
        padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 12),
        child: Row(
          children: [
            const Icon(Icons.chat_bubble_outline),
            const SizedBox(width: 4),
            Text('${_formatCount(post.commentCount)}'),
          ],
        ),
      ),
    );
  }

  Widget _buildShareButton(BuildContext context) {
    return IconButton(
      icon: const Icon(Icons.share_outlined),
      onPressed: () => _sharePost(context),
    );
  }

  Widget _buildBookmarkButton(BuildContext context) {
    return IconButton(
      icon: Icon(
        post.isBookmarked ? Icons.bookmark : Icons.bookmark_border,
        color: post.isBookmarked ? Colors.amber : null,
      ),
      onPressed: () => context.read<FeedBloc>().add(
        post.isBookmarked
            ? UnbookmarkPostEvent(postId: post.id)
            : BookmarkPostEvent(postId: post.id),
      ),
    );
  }

  void _showPostMenu(BuildContext context) {
    showModalBottomSheet(
      context: context,
      builder: (context) => Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          ListTile(
            leading: const Icon(Icons.report_outlined),
            title: const Text('รายงาน'),
            onTap: () {
              Navigator.pop(context);
              _reportPost(context);
            },
          ),
          ListTile(
            leading: const Icon(Icons.not_interested),
            title: const Text('ไม่สนใจโพสต์นี้'),
            onTap: () {
              Navigator.pop(context);
              context.read<FeedBloc>().add(HidePostEvent(postId: post.id));
            },
          ),
        ],
      ),
    );
  }

  void _sharePost(BuildContext context) {
    // Share implementation
  }

  void _reportPost(BuildContext context) {
    // Report implementation
  }

  String _formatTime(DateTime dateTime) {
    final diff = DateTime.now().difference(dateTime);
    if (diff.inSeconds < 60) return 'เพิ่งเมื่อกี้';
    if (diff.inMinutes < 60) return '${diff.inMinutes} นาทีที่แล้ว';
    if (diff.inHours < 24) return '${diff.inHours} ชั่วโมงที่แล้ว';
    if (diff.inDays < 7) return '${diff.inDays} วันที่แล้ว';
    return '${dateTime.day}/${dateTime.month}/${dateTime.year}';
  }

  String _formatCount(int count) {
    if (count >= 1000000) return '${(count / 1000000).toStringAsFixed(1)}M';
    if (count >= 1000) return '${(count / 1000).toStringAsFixed(1)}K';
    return '$count';
  }
}
```

## 4. Follow/Unfollow

### Follow Service

```dart
// lib/features/profile/domain/usecases/follow_user_usecase.dart
import 'package:dartz/dartz.dart';
import '../../../../core/errors/failures.dart';
import '../repositories/profile_repository.dart';

class FollowUserUseCase {
  final ProfileRepository _repository;

  const FollowUserUseCase({required ProfileRepository repository})
      : _repository = repository;

  Future<Either<Failure, void>> call(String userId) {
    return _repository.followUser(userId);
  }
}

class UnfollowUserUseCase {
  final ProfileRepository _repository;

  const UnfollowUserUseCase({required ProfileRepository repository})
      : _repository = repository;

  Future<Either<Failure, void>> call(String userId) {
    return _repository.unfollowUser(userId);
  }
}
```

## 5. Real-time Notifications

### Notification Service

```dart
// lib/features/notifications/data/services/notification_service.dart
import 'package:firebase_messaging/firebase_messaging.dart';

class NotificationService {
  final FirebaseMessaging _messaging = FirebaseMessaging.instance;

  Future<void> initialize() async {
    // ขอ permission
    await _messaging.requestPermission(
      alert: true,
      badge: true,
      sound: true,
    );

    // รับ FCM token
    final token = await _messaging.getToken();
    print('FCM Token: $token');

    // ส่ง token ไป server
    await _saveTokenToServer(token!);

    // ฟัง token refresh
    _messaging.onTokenRefresh.listen(_saveTokenToServer);

    // ตั้งค่า foreground message handler
    FirebaseMessaging.onMessage.listen(_handleForegroundMessage);

    // ตั้งค่า background message handler
    FirebaseMessaging.onBackgroundMessage(_handleBackgroundMessage);
  }

  void _handleForegroundMessage(RemoteMessage message) {
    print('Got foreground message: ${message.notification?.title}');
    // แสดง in-app notification
  }

  Future<void> _saveTokenToServer(String token) async {
    // TODO: ส่ง token ไป API server
  }
}

@pragma('vm:entry-point')
Future<void> _handleBackgroundMessage(RemoteMessage message) async {
  print('Got background message: ${message.notification?.title}');
}
```

### Notification List Screen

```dart
// lib/features/notifications/presentation/screens/notifications_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

class NotificationsScreen extends StatelessWidget {
  const NotificationsScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('การแจ้งเตือน'),
        actions: [
          TextButton(
            onPressed: () =>
                context.read<NotificationBloc>().add(
                  const MarkAllAsReadEvent(),
                ),
            child: const Text('อ่านทั้งหมด'),
          ),
        ],
      ),
      body: BlocBuilder<NotificationBloc, NotificationState>(
        builder: (context, state) {
          if (state is NotificationsLoaded) {
            if (state.notifications.isEmpty) {
              return const Center(
                child: Column(
                  mainAxisAlignment: MainAxisAlignment.center,
                  children: [
                    Icon(Icons.notifications_none, size: 64, color: Colors.grey),
                    SizedBox(height: 16),
                    Text('ยังไม่มีการแจ้งเตือน'),
                  ],
                ),
              );
            }

            return ListView.builder(
              itemCount: state.notifications.length,
              itemBuilder: (context, index) {
                final notification = state.notifications[index];
                return NotificationTile(notification: notification);
              },
            );
          }
          return const Center(child: CircularProgressIndicator());
        },
      ),
    );
  }
}

class NotificationTile extends StatelessWidget {
  final AppNotification notification;

  const NotificationTile({super.key, required this.notification});

  @override
  Widget build(BuildContext context) {
    return ListTile(
      tileColor: notification.isRead ? null : Colors.blue.withOpacity(0.05),
      leading: Stack(
        children: [
          CircleAvatar(
            backgroundImage: notification.actorAvatarUrl != null
                ? CachedNetworkImageProvider(notification.actorAvatarUrl!)
                : null,
            child: notification.actorAvatarUrl == null
                ? Text(notification.actorName[0].toUpperCase())
                : null,
          ),
          Positioned(
            right: 0,
            bottom: 0,
            child: Container(
              padding: const EdgeInsets.all(2),
              decoration: const BoxDecoration(
                color: Colors.white,
                shape: BoxShape.circle,
              ),
              child: Icon(
                _notificationIcon(notification.type),
                size: 12,
                color: _notificationColor(notification.type),
              ),
            ),
          ),
        ],
      ),
      title: RichText(
        text: TextSpan(
          style: DefaultTextStyle.of(context).style,
          children: [
            TextSpan(
              text: notification.actorName,
              style: const TextStyle(fontWeight: FontWeight.bold),
            ),
            TextSpan(text: ' ${_notificationText(notification.type)}'),
          ],
        ),
      ),
      subtitle: Text(
        _formatTime(notification.createdAt),
        style: const TextStyle(color: Colors.grey, fontSize: 12),
      ),
      trailing: notification.previewImageUrl != null
          ? ClipRRect(
              borderRadius: BorderRadius.circular(4),
              child: Image.network(
                notification.previewImageUrl!,
                width: 48,
                height: 48,
                fit: BoxFit.cover,
              ),
            )
          : null,
      onTap: () => _handleNotificationTap(context, notification),
    );
  }

  IconData _notificationIcon(NotificationType type) {
    switch (type) {
      case NotificationType.like:
        return Icons.favorite;
      case NotificationType.comment:
        return Icons.chat_bubble;
      case NotificationType.follow:
        return Icons.person_add;
      case NotificationType.mention:
        return Icons.alternate_email;
      case NotificationType.share:
        return Icons.share;
    }
  }

  Color _notificationColor(NotificationType type) {
    switch (type) {
      case NotificationType.like:
        return Colors.red;
      case NotificationType.comment:
        return Colors.blue;
      case NotificationType.follow:
        return Colors.green;
      case NotificationType.mention:
        return Colors.orange;
      case NotificationType.share:
        return Colors.purple;
    }
  }

  String _notificationText(NotificationType type) {
    switch (type) {
      case NotificationType.like:
        return 'ถูกใจโพสต์ของคุณ';
      case NotificationType.comment:
        return 'แสดงความคิดเห็นในโพสต์ของคุณ';
      case NotificationType.follow:
        return 'เริ่มติดตามคุณ';
      case NotificationType.mention:
        return 'กล่าวถึงคุณในโพสต์';
      case NotificationType.share:
        return 'แชร์โพสต์ของคุณ';
    }
  }

  String _formatTime(DateTime dateTime) {
    final diff = DateTime.now().difference(dateTime);
    if (diff.inMinutes < 60) return '${diff.inMinutes}m';
    if (diff.inHours < 24) return '${diff.inHours}h';
    return '${diff.inDays}d';
  }

  void _handleNotificationTap(
    BuildContext context,
    AppNotification notification,
  ) {
    context.read<NotificationBloc>().add(
      MarkAsReadEvent(notificationId: notification.id),
    );

    switch (notification.type) {
      case NotificationType.like:
      case NotificationType.comment:
      case NotificationType.share:
        if (notification.postId != null) {
          context.push('/post/${notification.postId}');
        }
        break;
      case NotificationType.follow:
        context.push('/profile/${notification.actorId}');
        break;
      case NotificationType.mention:
        if (notification.postId != null) {
          context.push('/post/${notification.postId}');
        }
        break;
    }
  }
}
```

## สรุป

Social Media App ครอบคลุม:
1. **User Profiles** - Avatar, stats, follow button
2. **Feed** - Infinite scroll, pull-to-refresh
3. **Posts** - Text, media, tags, actions
4. **Real-time** - Push notifications via FCM

## แบบทดสอบ

1. เพิ่ม Stories feature (Instagram-style) ใน app
2. Implement DM (Direct Messages) ด้วย WebSocket
3. สร้าง post creation screen ที่รองรับ text, images, location
4. เพิ่ม search ที่ค้นหา users, posts, hashtags พร้อมกัน
