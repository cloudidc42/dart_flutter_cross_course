# Part 42: Firebase Storage

## บทนำ

Firebase Storage เป็นบริการจัดเก็บไฟล์บน cloud ที่รองรับรูปภาพ วิดีโอ เอกสาร และไฟล์ต่างๆ มีความปลอดภัยสูงและรองรับการติดตาม progress ของการอัปโหลด/ดาวน์โหลด

---

## 42.1 การติดตั้งและตั้งค่า

### เพิ่ม dependency

```yaml
# pubspec.yaml
dependencies:
  flutter:
    sdk: flutter
  firebase_core: ^2.24.2
  firebase_storage: ^11.5.6
  image_picker: ^1.0.7
  path: ^1.8.3
  mime: ^1.0.4
```

### Storage Rules เบื้องต้น

```javascript
// storage.rules
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    
    // ให้ผู้ใช้ที่ login แล้วอ่านเขียนได้
    match /users/{userId}/{allPaths=**} {
      allow read: if request.auth != null;
      allow write: if request.auth != null 
                   && request.auth.uid == userId
                   && request.resource.size < 5 * 1024 * 1024; // 5MB
    }
    
    // Public images
    match /public/{allPaths=**} {
      allow read: if true;
      allow write: if request.auth != null;
    }
  }
}
```

---

## 42.2 Upload Files

### Upload พื้นฐาน

```dart
import 'dart:io';
import 'package:firebase_storage/firebase_storage.dart';
import 'package:path/path.dart' as path;

class StorageService {
  final FirebaseStorage _storage = FirebaseStorage.instance;
  
  // Upload ไฟล์พื้นฐาน
  Future<String> uploadFile({
    required File file,
    required String storagePath,
  }) async {
    try {
      final ref = _storage.ref(storagePath);
      final uploadTask = await ref.putFile(file);
      final downloadUrl = await uploadTask.ref.getDownloadURL();
      return downloadUrl;
    } catch (e) {
      throw Exception('Upload ไฟล์ไม่สำเร็จ: $e');
    }
  }
  
  // Upload รูปภาพ profile
  Future<String> uploadProfileImage({
    required String userId,
    required File imageFile,
  }) async {
    final extension = path.extension(imageFile.path);
    final fileName = 'profile$extension';
    final storagePath = 'users/$userId/profile/$fileName';
    
    final ref = _storage.ref(storagePath);
    
    // กำหนด metadata
    final metadata = SettableMetadata(
      contentType: 'image/jpeg',
      customMetadata: {
        'userId': userId,
        'uploadedAt': DateTime.now().toIso8601String(),
      },
    );
    
    final uploadTask = await ref.putFile(file: imageFile, metadata: metadata);
    return await uploadTask.ref.getDownloadURL();
  }
  
  // Upload หลายไฟล์พร้อมกัน
  Future<List<String>> uploadMultipleFiles({
    required List<File> files,
    required String basePath,
  }) async {
    final futures = files.asMap().entries.map((entry) {
      final index = entry.key;
      final file = entry.value;
      final ext = path.extension(file.path);
      return uploadFile(
        file: file,
        storagePath: '$basePath/file_$index$ext',
      );
    });
    
    return await Future.wait(futures);
  }
}
```

### Upload พร้อม Progress Tracking

```dart
class UploadWithProgress {
  final FirebaseStorage _storage = FirebaseStorage.instance;
  
  // Upload พร้อม stream progress
  Stream<double> uploadWithProgressStream({
    required File file,
    required String storagePath,
  }) async* {
    final ref = _storage.ref(storagePath);
    final uploadTask = ref.putFile(file);
    
    yield* uploadTask.snapshotEvents.map((snapshot) {
      switch (snapshot.state) {
        case TaskState.running:
          return snapshot.bytesTransferred / snapshot.totalBytes;
        case TaskState.success:
          return 1.0;
        case TaskState.canceled:
        case TaskState.error:
          return 0.0;
        default:
          return 0.0;
      }
    });
  }
  
  // Upload พร้อม callback progress
  Future<String> uploadWithCallbacks({
    required File file,
    required String storagePath,
    void Function(double progress)? onProgress,
    void Function()? onComplete,
    void Function(String error)? onError,
  }) async {
    final ref = _storage.ref(storagePath);
    final uploadTask = ref.putFile(file);
    
    uploadTask.snapshotEvents.listen(
      (snapshot) {
        final progress = 
            snapshot.bytesTransferred / snapshot.totalBytes;
        onProgress?.call(progress);
      },
      onError: (error) {
        onError?.call(error.toString());
      },
    );
    
    final taskSnapshot = await uploadTask;
    onComplete?.call();
    return await taskSnapshot.ref.getDownloadURL();
  }
}
```

### Upload Widget พร้อม UI

```dart
class FileUploadWidget extends StatefulWidget {
  final String userId;
  final void Function(String url) onUploadComplete;
  
  const FileUploadWidget({
    super.key,
    required this.userId,
    required this.onUploadComplete,
  });
  
  @override
  State<FileUploadWidget> createState() => _FileUploadWidgetState();
}

class _FileUploadWidgetState extends State<FileUploadWidget> {
  final StorageService _storageService = StorageService();
  double? _uploadProgress;
  bool _isUploading = false;
  String? _uploadedUrl;
  
  Future<void> _pickAndUpload() async {
    final picker = ImagePicker();
    final pickedFile = await picker.pickImage(
      source: ImageSource.gallery,
      imageQuality: 80,
    );
    
    if (pickedFile == null) return;
    
    final file = File(pickedFile.path);
    final fileName = path.basename(pickedFile.path);
    final storagePath = 'users/${widget.userId}/uploads/$fileName';
    
    setState(() {
      _isUploading = true;
      _uploadProgress = 0;
    });
    
    await _storageService.uploadWithCallbacks(
      file: file,
      storagePath: storagePath,
      onProgress: (progress) {
        setState(() => _uploadProgress = progress);
      },
      onComplete: () {
        setState(() => _isUploading = false);
      },
      onError: (error) {
        setState(() => _isUploading = false);
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Upload ไม่สำเร็จ: $error')),
        );
      },
    ).then((url) {
      setState(() => _uploadedUrl = url);
      widget.onUploadComplete(url);
    });
  }
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        if (_uploadedUrl != null)
          ClipRRect(
            borderRadius: BorderRadius.circular(8),
            child: Image.network(
              _uploadedUrl!,
              width: 200,
              height: 200,
              fit: BoxFit.cover,
            ),
          ),
        
        if (_isUploading) ...[
          const SizedBox(height: 16),
          LinearProgressIndicator(value: _uploadProgress),
          const SizedBox(height: 8),
          Text(
            'กำลัง upload: ${((_uploadProgress ?? 0) * 100).toStringAsFixed(0)}%',
          ),
        ],
        
        const SizedBox(height: 16),
        ElevatedButton.icon(
          onPressed: _isUploading ? null : _pickAndUpload,
          icon: const Icon(Icons.upload),
          label: Text(_isUploading ? 'กำลัง upload...' : 'เลือกรูปภาพ'),
        ),
      ],
    );
  }
}
```

---

## 42.3 Download URLs และการอ่านไฟล์

```dart
class StorageDownload {
  final FirebaseStorage _storage = FirebaseStorage.instance;
  
  // ได้ Download URL จาก path
  Future<String> getDownloadUrl(String storagePath) async {
    final ref = _storage.ref(storagePath);
    return await ref.getDownloadURL();
  }
  
  // ดาวน์โหลดไฟล์เป็น bytes
  Future<Uint8List?> downloadFile(String storagePath) async {
    try {
      final ref = _storage.ref(storagePath);
      // ดาวน์โหลดสูงสุด 10MB
      final data = await ref.getData(10 * 1024 * 1024);
      return data;
    } catch (e) {
      print('ดาวน์โหลดไม่สำเร็จ: $e');
      return null;
    }
  }
  
  // ดาวน์โหลดไฟล์ไปยัง path ท้องถิ่น
  Future<void> downloadToLocal({
    required String storagePath,
    required String localPath,
    void Function(double)? onProgress,
  }) async {
    final ref = _storage.ref(storagePath);
    final localFile = File(localPath);
    
    final downloadTask = ref.writeToFile(localFile);
    
    downloadTask.snapshotEvents.listen((snapshot) {
      if (snapshot.totalBytes > 0) {
        final progress = 
            snapshot.bytesTransferred / snapshot.totalBytes;
        onProgress?.call(progress);
      }
    });
    
    await downloadTask;
  }
  
  // ดึง metadata ของไฟล์
  Future<FullMetadata> getFileMetadata(String storagePath) async {
    final ref = _storage.ref(storagePath);
    return await ref.getMetadata();
  }
  
  // แสดงรายการไฟล์ใน folder
  Future<List<Reference>> listFiles(String folderPath) async {
    final ref = _storage.ref(folderPath);
    final result = await ref.listAll();
    return result.items;
  }
}
```

---

## 42.4 Delete Files

```dart
class StorageDelete {
  final FirebaseStorage _storage = FirebaseStorage.instance;
  
  // ลบไฟล์เดียว
  Future<void> deleteFile(String storagePath) async {
    try {
      final ref = _storage.ref(storagePath);
      await ref.delete();
      print('ลบไฟล์สำเร็จ: $storagePath');
    } catch (e) {
      if (e is FirebaseException && e.code == 'object-not-found') {
        print('ไม่พบไฟล์: $storagePath');
      } else {
        rethrow;
      }
    }
  }
  
  // ลบไฟล์จาก URL
  Future<void> deleteFileByUrl(String downloadUrl) async {
    final ref = _storage.refFromURL(downloadUrl);
    await ref.delete();
  }
  
  // ลบทุกไฟล์ใน folder
  Future<void> deleteFolder(String folderPath) async {
    final ref = _storage.ref(folderPath);
    final result = await ref.listAll();
    
    // ลบไฟล์ทั้งหมด
    await Future.wait(
      result.items.map((item) => item.delete()),
    );
    
    // ลบ subfolder ด้วย (recursive)
    await Future.wait(
      result.prefixes.map((prefix) => deleteFolder(prefix.fullPath)),
    );
  }
}
```

---

## 42.5 Workshop: Profile Photo Upload

### โครงสร้าง Profile Photo Feature

```dart
// profile_photo_service.dart
import 'dart:io';
import 'package:firebase_storage/firebase_storage.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:image_picker/image_picker.dart';
import 'package:image_cropper/image_cropper.dart';

class ProfilePhotoService {
  final FirebaseStorage _storage = FirebaseStorage.instance;
  final FirebaseFirestore _db = FirebaseFirestore.instance;
  final ImagePicker _picker = ImagePicker();
  
  // เลือกรูปจาก Gallery หรือ Camera
  Future<File?> pickImage(ImageSource source) async {
    final pickedFile = await _picker.pickImage(
      source: source,
      imageQuality: 85,
      maxWidth: 1080,
      maxHeight: 1080,
    );
    
    if (pickedFile == null) return null;
    return File(pickedFile.path);
  }
  
  // Crop รูปภาพ
  Future<File?> cropImage(File imageFile) async {
    final croppedFile = await ImageCropper().cropImage(
      sourcePath: imageFile.path,
      aspectRatio: const CropAspectRatio(ratioX: 1, ratioY: 1),
      compressQuality: 85,
      uiSettings: [
        AndroidUiSettings(
          toolbarTitle: 'ครอบรูปภาพ',
          toolbarColor: Colors.blue,
          toolbarWidgetColor: Colors.white,
          lockAspectRatio: true,
        ),
        IOSUiSettings(
          title: 'ครอบรูปภาพ',
          aspectRatioLockEnabled: true,
        ),
      ],
    );
    
    if (croppedFile == null) return null;
    return File(croppedFile.path);
  }
  
  // Upload Profile Photo พร้อม progress
  Future<String> uploadProfilePhoto({
    required String userId,
    required File imageFile,
    void Function(double)? onProgress,
  }) async {
    final ext = imageFile.path.split('.').last.toLowerCase();
    final storagePath = 'users/$userId/profile/photo.$ext';
    
    // ลบรูปเดิมก่อน (ถ้ามี)
    await _deleteOldProfilePhoto(userId);
    
    final ref = _storage.ref(storagePath);
    final metadata = SettableMetadata(
      contentType: 'image/$ext',
    );
    
    final uploadTask = ref.putFile(imageFile, metadata);
    
    // Track progress
    uploadTask.snapshotEvents.listen((snapshot) {
      if (snapshot.totalBytes > 0) {
        final progress = snapshot.bytesTransferred / snapshot.totalBytes;
        onProgress?.call(progress);
      }
    });
    
    final taskSnapshot = await uploadTask;
    final downloadUrl = await taskSnapshot.ref.getDownloadURL();
    
    // บันทึก URL ใน Firestore
    await _db.collection('users').doc(userId).update({
      'photoUrl': downloadUrl,
      'photoUpdatedAt': FieldValue.serverTimestamp(),
    });
    
    return downloadUrl;
  }
  
  Future<void> _deleteOldProfilePhoto(String userId) async {
    try {
      final userDoc = await _db.collection('users').doc(userId).get();
      final data = userDoc.data();
      
      if (data != null && data['photoUrl'] != null) {
        final oldRef = _storage.refFromURL(data['photoUrl'] as String);
        await oldRef.delete();
      }
    } catch (_) {
      // ไม่มีรูปเดิม ไม่ต้องทำอะไร
    }
  }
}
```

### Profile Edit Page

```dart
// profile_edit_page.dart
import 'package:flutter/material.dart';
import 'dart:io';

class ProfileEditPage extends StatefulWidget {
  final String userId;
  final String? currentPhotoUrl;
  final String userName;
  
  const ProfileEditPage({
    super.key,
    required this.userId,
    this.currentPhotoUrl,
    required this.userName,
  });
  
  @override
  State<ProfileEditPage> createState() => _ProfileEditPageState();
}

class _ProfileEditPageState extends State<ProfileEditPage> {
  final ProfilePhotoService _photoService = ProfilePhotoService();
  
  File? _selectedImage;
  String? _uploadedPhotoUrl;
  double _uploadProgress = 0;
  bool _isUploading = false;
  
  @override
  void initState() {
    super.initState();
    _uploadedPhotoUrl = widget.currentPhotoUrl;
  }
  
  Future<void> _showImageSourceDialog() async {
    final source = await showModalBottomSheet<ImageSource>(
      context: context,
      shape: const RoundedRectangleBorder(
        borderRadius: BorderRadius.vertical(top: Radius.circular(20)),
      ),
      builder: (context) => Padding(
        padding: const EdgeInsets.all(20),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            const Text(
              'เลือกรูปภาพจาก',
              style: TextStyle(
                fontSize: 18,
                fontWeight: FontWeight.bold,
              ),
            ),
            const SizedBox(height: 20),
            ListTile(
              leading: const Icon(Icons.photo_library),
              title: const Text('คลังรูปภาพ'),
              onTap: () => Navigator.pop(context, ImageSource.gallery),
            ),
            ListTile(
              leading: const Icon(Icons.camera_alt),
              title: const Text('กล้องถ่ายภาพ'),
              onTap: () => Navigator.pop(context, ImageSource.camera),
            ),
          ],
        ),
      ),
    );
    
    if (source == null) return;
    
    final image = await _photoService.pickImage(source);
    if (image == null) return;
    
    final croppedImage = await _photoService.cropImage(image);
    if (croppedImage == null) return;
    
    setState(() => _selectedImage = croppedImage);
  }
  
  Future<void> _uploadPhoto() async {
    if (_selectedImage == null) return;
    
    setState(() {
      _isUploading = true;
      _uploadProgress = 0;
    });
    
    try {
      final url = await _photoService.uploadProfilePhoto(
        userId: widget.userId,
        imageFile: _selectedImage!,
        onProgress: (progress) {
          setState(() => _uploadProgress = progress);
        },
      );
      
      setState(() {
        _uploadedPhotoUrl = url;
        _selectedImage = null;
        _isUploading = false;
      });
      
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(
            content: Text('อัปโหลดรูปภาพสำเร็จ'),
            backgroundColor: Colors.green,
          ),
        );
        Navigator.pop(context, url);
      }
    } catch (e) {
      setState(() => _isUploading = false);
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('เกิดข้อผิดพลาด: $e')),
        );
      }
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('แก้ไขรูปโปรไฟล์'),
      ),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(24),
        child: Column(
          children: [
            // Profile Photo Preview
            Center(
              child: Stack(
                children: [
                  CircleAvatar(
                    radius: 80,
                    backgroundColor: Colors.grey[200],
                    backgroundImage: _selectedImage != null
                        ? FileImage(_selectedImage!)
                        : (_uploadedPhotoUrl != null
                            ? NetworkImage(_uploadedPhotoUrl!) as ImageProvider
                            : null),
                    child: (_selectedImage == null && _uploadedPhotoUrl == null)
                        ? Text(
                            widget.userName.isNotEmpty
                                ? widget.userName[0].toUpperCase()
                                : '?',
                            style: const TextStyle(
                              fontSize: 48,
                              fontWeight: FontWeight.bold,
                            ),
                          )
                        : null,
                  ),
                  Positioned(
                    bottom: 0,
                    right: 0,
                    child: CircleAvatar(
                      backgroundColor: Colors.blue,
                      radius: 24,
                      child: IconButton(
                        icon: const Icon(
                          Icons.camera_alt,
                          color: Colors.white,
                          size: 20,
                        ),
                        onPressed: _isUploading 
                            ? null 
                            : _showImageSourceDialog,
                      ),
                    ),
                  ),
                ],
              ),
            ),
            
            const SizedBox(height: 32),
            
            // Upload Progress
            if (_isUploading) ...[
              ClipRRect(
                borderRadius: BorderRadius.circular(4),
                child: LinearProgressIndicator(
                  value: _uploadProgress,
                  minHeight: 8,
                ),
              ),
              const SizedBox(height: 8),
              Text(
                'กำลังอัปโหลด ${(_uploadProgress * 100).toStringAsFixed(0)}%',
                style: const TextStyle(color: Colors.grey),
              ),
              const SizedBox(height: 32),
            ],
            
            // Upload Button
            if (_selectedImage != null && !_isUploading)
              SizedBox(
                width: double.infinity,
                child: ElevatedButton.icon(
                  onPressed: _uploadPhoto,
                  icon: const Icon(Icons.cloud_upload),
                  label: const Text('อัปโหลดรูปภาพ'),
                  style: ElevatedButton.styleFrom(
                    padding: const EdgeInsets.all(16),
                  ),
                ),
              ),
          ],
        ),
      ),
    );
  }
}
```

### แสดง Profile Photo จาก Cache

```dart
// Cached Network Image
// ต้องเพิ่ม package: cached_network_image: ^3.3.1

import 'package:cached_network_image/cached_network_image.dart';

class ProfileAvatar extends StatelessWidget {
  final String? photoUrl;
  final String name;
  final double radius;
  
  const ProfileAvatar({
    super.key,
    this.photoUrl,
    required this.name,
    this.radius = 24,
  });
  
  @override
  Widget build(BuildContext context) {
    return CircleAvatar(
      radius: radius,
      backgroundColor: Colors.blue[100],
      child: photoUrl != null
          ? ClipOval(
              child: CachedNetworkImage(
                imageUrl: photoUrl!,
                width: radius * 2,
                height: radius * 2,
                fit: BoxFit.cover,
                placeholder: (context, url) => CircularProgressIndicator(
                  strokeWidth: 2,
                  valueColor: AlwaysStoppedAnimation<Color>(
                    Colors.blue[400]!,
                  ),
                ),
                errorWidget: (context, url, error) => Text(
                  name.isNotEmpty ? name[0].toUpperCase() : '?',
                  style: TextStyle(
                    fontSize: radius * 0.8,
                    fontWeight: FontWeight.bold,
                    color: Colors.blue[700],
                  ),
                ),
              ),
            )
          : Text(
              name.isNotEmpty ? name[0].toUpperCase() : '?',
              style: TextStyle(
                fontSize: radius * 0.8,
                fontWeight: FontWeight.bold,
                color: Colors.blue[700],
              ),
            ),
    );
  }
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- การ Upload ไฟล์ไปยัง Firebase Storage
- การติดตาม Progress ของการ Upload
- การได้ Download URLs
- การลบไฟล์
- Workshop: Profile Photo Upload พร้อม Image Cropping

**แบบฝึกหัดเพิ่มเติม:**
1. เพิ่มการ compress รูปภาพก่อน upload
2. Implement document upload (PDF)
3. สร้าง photo gallery ที่ดึงรูปจาก Storage
4. เพิ่ม retry mechanism เมื่อ upload ล้มเหลว
