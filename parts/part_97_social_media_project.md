# Part 97: Real-World Project - Social Media App

## 🎯 เป้าหมายของ Part นี้
- สร้าง Social Media app
- Real-time updates ด้วย Firestore
- Feed algorithm
- Story feature
- Like/Comment/Share
- Follow system

---

## 1. App Architecture

```
SocialApp Features:
├── Authentication
├── Feed (infinite scroll, algorithm)
├── Stories (ephemeral content)
├── Posts (create, view, like, comment)
├── Profile
├── Follow/Followers
├── Search
├── Notifications
└── Direct Messages
```

---

## 2. Data Models

```dart
// domain/entities/post.dart
class Post {
  final String id;
  final String userId;
  final String userDisplayName;
  final String userPhotoUrl;
  final String content;
  final List<String> images;
  final int likeCount;
  final int commentCount;
  final int shareCount;
  final bool isLiked;          // current user's like status
  final DateTime createdAt;
  final List<String> hashtags;
  final String? location;
  
  const Post({
    required this.id,
    required this.userId,
    required this.userDisplayName,
    required this.userPhotoUrl,
    required this.content,
    this.images = const [],
    this.likeCount = 0,
    this.commentCount = 0,
    this.shareCount = 0,
    this.isLiked = false,
    required this.createdAt,
    this.hashtags = const [],
    this.location,
  });
  
  Post copyWith({bool? isLiked, int? likeCount}) {
    return Post(
      id: id,
      userId: userId,
      userDisplayName: userDisplayName,
      userPhotoUrl: userPhotoUrl,
      content: content,
      images: images,
      likeCount: likeCount ?? this.likeCount,
      commentCount: commentCount,
      shareCount: shareCount,
      isLiked: isLiked ?? this.isLiked,
      createdAt: createdAt,
      hashtags: hashtags,
      location: location,
    );
  }
}

class Story {
  final String id;
  final String userId;
  final String userDisplayName;
  final String userPhotoUrl;
  final List<StoryItem> items;
  final bool hasViewed;
  
  const Story({
    required this.id,
    required this.userId,
    required this.userDisplayName,
    required this.userPhotoUrl,
    required this.items,
    this.hasViewed = false,
  });
}

class StoryItem {
  final String id;
  final String type; // 'image' or 'video'
  final String url;
  final DateTime createdAt;
  final DateTime expiresAt;
  final int viewCount;
  
  const StoryItem({
    required this.id,
    required this.type,
    required this.url,
    required this.createdAt,
    required this.expiresAt,
    this.viewCount = 0,
  });
  
  bool get isExpired => DateTime.now().isAfter(expiresAt);
}

class UserProfile {
  final String id;
  final String displayName;
  final String username;
  final String photoUrl;
  final String bio;
  final int postCount;
  final int followerCount;
  final int followingCount;
  final bool isFollowing;
  final bool isCurrentUser;
  
  const UserProfile({
    required this.id,
    required this.displayName,
    required this.username,
    required this.photoUrl,
    required this.bio,
    required this.postCount,
    required this.followerCount,
    required this.followingCount,
    required this.isFollowing,
    required this.isCurrentUser,
  });
}
```

---

## 3. Feed Implementation

```dart
// features/feed/presentation/pages/feed_page.dart
class FeedPage extends ConsumerStatefulWidget {
  const FeedPage({super.key});

  @override
  ConsumerState<FeedPage> createState() => _FeedPageState();
}

class _FeedPageState extends ConsumerState<FeedPage> {
  final _scrollController = ScrollController();

  @override
  void initState() {
    super.initState();
    _scrollController.addListener(_onScroll);
  }

  @override
  void dispose() {
    _scrollController.dispose();
    super.dispose();
  }

  void _onScroll() {
    if (_scrollController.position.pixels >= 
        _scrollController.position.maxScrollExtent - 200) {
      ref.read(feedProvider.notifier).loadMore();
    }
  }

  @override
  Widget build(BuildContext context) {
    final feedState = ref.watch(feedProvider);
    final stories = ref.watch(storiesProvider);

    return Scaffold(
      appBar: AppBar(
        title: const Text('SocialApp'),
        actions: [
          IconButton(
            icon: const Icon(Icons.add_box_outlined),
            onPressed: () => context.push('/create-post'),
          ),
          IconButton(
            icon: const Icon(Icons.send_outlined),
            onPressed: () => context.push('/messages'),
          ),
        ],
      ),
      body: RefreshIndicator(
        onRefresh: () => ref.read(feedProvider.notifier).refresh(),
        child: CustomScrollView(
          controller: _scrollController,
          slivers: [
            // Stories bar:
            SliverToBoxAdapter(
              child: stories.when(
                loading: () => const StoriesLoadingBar(),
                error: (_, __) => const SizedBox.shrink(),
                data: (stories) => StoriesBar(stories: stories),
              ),
            ),
            
            // Feed posts:
            if (feedState.error != null)
              SliverToBoxAdapter(
                child: ErrorBanner(message: feedState.error!),
              ),
            
            SliverList(
              delegate: SliverChildBuilderDelegate(
                (context, index) {
                  if (index >= feedState.posts.length) {
                    return feedState.isLoading
                        ? const LoadingIndicator()
                        : const EndOfFeedMessage();
                  }
                  return PostCard(post: feedState.posts[index]);
                },
                childCount: feedState.posts.length + 1,
              ),
            ),
          ],
        ),
      ),
    );
  }
}

// Stories Bar:
class StoriesBar extends StatelessWidget {
  final List<Story> stories;

  const StoriesBar({super.key, required this.stories});

  @override
  Widget build(BuildContext context) {
    return SizedBox(
      height: 100,
      child: ListView.builder(
        scrollDirection: Axis.horizontal,
        padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 8),
        itemCount: stories.length + 1, // +1 for "Add Story"
        itemBuilder: (context, index) {
          if (index == 0) {
            return const AddStoryButton();
          }
          return StoryAvatar(story: stories[index - 1]);
        },
      ),
    );
  }
}

class StoryAvatar extends StatelessWidget {
  final Story story;

  const StoryAvatar({super.key, required this.story});

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: () => context.push('/stories/${story.userId}'),
      child: Container(
        margin: const EdgeInsets.only(right: 12),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Container(
              width: 60,
              height: 60,
              decoration: BoxDecoration(
                shape: BoxShape.circle,
                gradient: story.hasViewed
                    ? null
                    : const LinearGradient(
                        colors: [Color(0xFFF77737), Color(0xFFE1306C), Color(0xFF833AB4)],
                        begin: Alignment.topRight,
                        end: Alignment.bottomLeft,
                      ),
                color: story.hasViewed ? Colors.grey.shade300 : null,
              ),
              padding: const EdgeInsets.all(2),
              child: CircleAvatar(
                backgroundImage: NetworkImage(story.userPhotoUrl),
                radius: 28,
              ),
            ),
            const SizedBox(height: 4),
            SizedBox(
              width: 60,
              child: Text(
                story.userDisplayName.split(' ').first,
                maxLines: 1,
                overflow: TextOverflow.ellipsis,
                textAlign: TextAlign.center,
                style: const TextStyle(fontSize: 12),
              ),
            ),
          ],
        ),
      ),
    );
  }
}

// Post Card:
class PostCard extends ConsumerWidget {
  final Post post;

  const PostCard({super.key, required this.post});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Card(
      margin: const EdgeInsets.only(bottom: 8),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          // Header
          ListTile(
            leading: GestureDetector(
              onTap: () => context.push('/profile/${post.userId}'),
              child: CircleAvatar(
                backgroundImage: NetworkImage(post.userPhotoUrl),
              ),
            ),
            title: Text(post.userDisplayName),
            subtitle: Text(_formatTime(post.createdAt)),
            trailing: IconButton(
              icon: const Icon(Icons.more_horiz),
              onPressed: () => _showOptions(context),
            ),
          ),
          
          // Content
          if (post.content.isNotEmpty)
            Padding(
              padding: const EdgeInsets.symmetric(horizontal: 16),
              child: ExpandableText(
                text: post.content,
                maxLines: 4,
              ),
            ),
          
          // Images
          if (post.images.isNotEmpty) ...[
            const SizedBox(height: 8),
            if (post.images.length == 1)
              AspectRatio(
                aspectRatio: 1,
                child: CachedNetworkImage(
                  imageUrl: post.images.first,
                  fit: BoxFit.cover,
                ),
              )
            else
              PostImageGrid(images: post.images),
          ],
          
          // Stats
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
            child: Row(
              children: [
                Text('${post.likeCount} likes'),
                const Spacer(),
                Text('${post.commentCount} comments'),
                const SizedBox(width: 8),
                Text('${post.shareCount} shares'),
              ],
            ),
          ),
          
          const Divider(height: 1),
          
          // Action buttons
          Row(
            children: [
              _ActionButton(
                icon: post.isLiked ? Icons.favorite : Icons.favorite_border,
                color: post.isLiked ? Colors.red : null,
                label: 'ถูกใจ',
                onTap: () => ref.read(feedProvider.notifier).toggleLike(post.id),
              ),
              _ActionButton(
                icon: Icons.comment_outlined,
                label: 'ความคิดเห็น',
                onTap: () => context.push('/post/${post.id}/comments'),
              ),
              _ActionButton(
                icon: Icons.share_outlined,
                label: 'แชร์',
                onTap: () => _sharePost(context),
              ),
              _ActionButton(
                icon: Icons.bookmark_border,
                label: 'บันทึก',
                onTap: () => ref.read(savedPostsProvider.notifier).save(post.id),
              ),
            ],
          ),
        ],
      ),
    );
  }
  
  String _formatTime(DateTime time) {
    final now = DateTime.now();
    final diff = now.difference(time);
    
    if (diff.inMinutes < 1) return 'เมื่อกี้';
    if (diff.inHours < 1) return '${diff.inMinutes} นาทีที่แล้ว';
    if (diff.inDays < 1) return '${diff.inHours} ชั่วโมงที่แล้ว';
    if (diff.inDays < 7) return '${diff.inDays} วันที่แล้ว';
    
    return '${time.day}/${time.month}/${time.year}';
  }
  
  void _showOptions(BuildContext context) {
    showModalBottomSheet(
      context: context,
      builder: (context) => Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          ListTile(
            leading: const Icon(Icons.report),
            title: const Text('รายงาน'),
            onTap: () {},
          ),
          ListTile(
            leading: const Icon(Icons.block),
            title: const Text('บล็อก'),
            onTap: () {},
          ),
        ],
      ),
    );
  }
  
  void _sharePost(BuildContext context) {
    Share.share('ดูโพสต์นี้: ${post.content}');
  }
}

class _ActionButton extends StatelessWidget {
  final IconData icon;
  final Color? color;
  final String label;
  final VoidCallback onTap;

  const _ActionButton({
    required this.icon,
    this.color,
    required this.label,
    required this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    return Expanded(
      child: TextButton.icon(
        onPressed: onTap,
        icon: Icon(icon, color: color),
        label: Text(label),
        style: TextButton.styleFrom(
          foregroundColor: color ?? Theme.of(context).colorScheme.onSurface,
        ),
      ),
    );
  }
}
```

---

## 4. Story Viewer

```dart
class StoryViewer extends StatefulWidget {
  final List<Story> stories;
  final int initialStoryIndex;
  
  const StoryViewer({
    super.key,
    required this.stories,
    this.initialStoryIndex = 0,
  });

  @override
  State<StoryViewer> createState() => _StoryViewerState();
}

class _StoryViewerState extends State<StoryViewer>
    with SingleTickerProviderStateMixin {
  late AnimationController _progressController;
  int _storyIndex = 0;
  int _itemIndex = 0;
  
  Story get currentStory => widget.stories[_storyIndex];
  StoryItem get currentItem => currentStory.items[_itemIndex];
  
  @override
  void initState() {
    super.initState();
    _storyIndex = widget.initialStoryIndex;
    _progressController = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 5),
    );
    _startProgress();
  }
  
  void _startProgress() {
    _progressController.forward(from: 0).then((_) {
      _nextItem();
    });
  }
  
  void _nextItem() {
    if (_itemIndex < currentStory.items.length - 1) {
      setState(() => _itemIndex++);
      _startProgress();
    } else if (_storyIndex < widget.stories.length - 1) {
      setState(() {
        _storyIndex++;
        _itemIndex = 0;
      });
      _startProgress();
    } else {
      Navigator.of(context).pop();
    }
  }
  
  void _prevItem() {
    if (_itemIndex > 0) {
      setState(() => _itemIndex--);
      _startProgress();
    } else if (_storyIndex > 0) {
      setState(() {
        _storyIndex--;
        _itemIndex = currentStory.items.length - 1;
      });
      _startProgress();
    }
  }

  @override
  void dispose() {
    _progressController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.black,
      body: GestureDetector(
        onTapDown: (details) {
          final screenWidth = MediaQuery.of(context).size.width;
          if (details.localPosition.dx < screenWidth / 2) {
            _prevItem();
          } else {
            _nextItem();
          }
        },
        child: Stack(
          fit: StackFit.expand,
          children: [
            // Story content:
            CachedNetworkImage(
              imageUrl: currentItem.url,
              fit: BoxFit.contain,
            ),
            
            // Progress bars:
            Positioned(
              top: MediaQuery.of(context).padding.top + 8,
              left: 8,
              right: 8,
              child: Row(
                children: List.generate(
                  currentStory.items.length,
                  (index) => Expanded(
                    child: Padding(
                      padding: const EdgeInsets.symmetric(horizontal: 2),
                      child: AnimatedBuilder(
                        animation: _progressController,
                        builder: (context, child) {
                          double progress;
                          if (index < _itemIndex) {
                            progress = 1.0; // completed
                          } else if (index == _itemIndex) {
                            progress = _progressController.value;
                          } else {
                            progress = 0.0; // not started
                          }
                          
                          return LinearProgressIndicator(
                            value: progress,
                            backgroundColor: Colors.white.withOpacity(0.3),
                            valueColor: const AlwaysStoppedAnimation(Colors.white),
                            minHeight: 2,
                          );
                        },
                      ),
                    ),
                  ),
                ),
              ),
            ),
            
            // User info:
            Positioned(
              top: MediaQuery.of(context).padding.top + 24,
              left: 16,
              right: 16,
              child: Row(
                children: [
                  CircleAvatar(
                    backgroundImage: NetworkImage(currentStory.userPhotoUrl),
                    radius: 20,
                  ),
                  const SizedBox(width: 8),
                  Text(
                    currentStory.userDisplayName,
                    style: const TextStyle(
                      color: Colors.white,
                      fontWeight: FontWeight.bold,
                    ),
                  ),
                  const Spacer(),
                  IconButton(
                    icon: const Icon(Icons.close, color: Colors.white),
                    onPressed: () => Navigator.of(context).pop(),
                  ),
                ],
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

## 5. Firestore Data Structure

```javascript
// Firestore collections:

users/{userId}
  - displayName: string
  - username: string
  - photoUrl: string
  - bio: string
  - followerCount: number
  - followingCount: number
  - postCount: number
  - createdAt: timestamp

posts/{postId}
  - userId: string (ref to users)
  - content: string
  - images: string[]
  - likeCount: number
  - commentCount: number
  - hashtags: string[]
  - location: string?
  - createdAt: timestamp

posts/{postId}/likes/{userId}  ← subcollection
  - createdAt: timestamp

posts/{postId}/comments/{commentId}
  - userId: string
  - content: string
  - likeCount: number
  - createdAt: timestamp

follows/{followerId}_{followingId}
  - followerId: string
  - followingId: string
  - createdAt: timestamp

stories/{storyId}
  - userId: string
  - items: StoryItem[]
  - expiresAt: timestamp (24 hours from creation)

notifications/{userId}/items/{notifId}
  - type: 'like' | 'comment' | 'follow' | 'mention'
  - fromUserId: string
  - postId: string?
  - isRead: boolean
  - createdAt: timestamp
```

---

## 6. สรุป Part 97

สิ่งที่เรียนรู้:
- ✅ Social media app architecture
- ✅ Real-time feed ด้วย Firestore
- ✅ Stories feature with progress animation
- ✅ Like/Comment system
- ✅ Follow/Unfollow
- ✅ Firestore data structure design

---

## ➡️ Part ถัดไป
**Part 98: Real-World Project - Fintech App**
