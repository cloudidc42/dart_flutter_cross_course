# Part 44: Camera และ Image Picker

## บทนำ

การทำงานกับกล้องและรูปภาพเป็นฟีเจอร์สำคัญในแอปมือถือ Flutter มี packages หลักๆ ที่ใช้งานได้คือ `image_picker` สำหรับเลือกรูปจาก gallery หรือถ่ายภาพ และ `camera` สำหรับควบคุมกล้องแบบ advanced

---

## 44.1 image_picker Package

### ติดตั้ง

```yaml
# pubspec.yaml
dependencies:
  image_picker: ^1.0.7
  image_cropper: ^5.0.1
  permission_handler: ^11.2.0
  flutter_image_compress: ^2.2.0
```

### ตั้งค่า Android

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<manifest>
    <uses-permission android:name="android.permission.CAMERA"/>
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
    
    <!-- Android 13+ -->
    <uses-permission android:name="android.permission.READ_MEDIA_IMAGES"/>
    <uses-permission android:name="android.permission.READ_MEDIA_VIDEO"/>
    
    <application>
        <!-- FileProvider สำหรับ camera -->
        <provider
            android:name="androidx.core.content.FileProvider"
            android:authorities="${applicationId}.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/file_paths"/>
        </provider>
    </application>
</manifest>
```

### ตั้งค่า iOS

```xml
<!-- ios/Runner/Info.plist -->
<key>NSCameraUsageDescription</key>
<string>แอปต้องการใช้กล้องเพื่อถ่ายรูปโปรไฟล์</string>

<key>NSPhotoLibraryUsageDescription</key>
<string>แอปต้องการเข้าถึงรูปภาพในคลัง</string>

<key>NSMicrophoneUsageDescription</key>
<string>แอปต้องการใช้ไมโครโฟนเพื่อบันทึกวิดีโอ</string>
```

---

## 44.2 การใช้งาน image_picker

### ImagePicker Service

```dart
import 'dart:io';
import 'package:image_picker/image_picker.dart';
import 'package:flutter_image_compress/flutter_image_compress.dart';
import 'package:path_provider/path_provider.dart';
import 'package:path/path.dart' as path;

class ImagePickerService {
  final ImagePicker _picker = ImagePicker();
  
  // เลือกรูปจาก Gallery
  Future<File?> pickFromGallery({
    int imageQuality = 85,
    int? maxWidth,
    int? maxHeight,
  }) async {
    try {
      final pickedFile = await _picker.pickImage(
        source: ImageSource.gallery,
        imageQuality: imageQuality,
        maxWidth: maxWidth?.toDouble(),
        maxHeight: maxHeight?.toDouble(),
      );
      
      if (pickedFile == null) return null;
      return File(pickedFile.path);
    } catch (e) {
      print('ไม่สามารถเลือกรูปได้: $e');
      return null;
    }
  }
  
  // ถ่ายรูปจากกล้อง
  Future<File?> takePhoto({
    CameraDevice preferredCamera = CameraDevice.rear,
    int imageQuality = 85,
    int? maxWidth,
    int? maxHeight,
  }) async {
    try {
      final pickedFile = await _picker.pickImage(
        source: ImageSource.camera,
        preferredCameraDevice: preferredCamera,
        imageQuality: imageQuality,
        maxWidth: maxWidth?.toDouble(),
        maxHeight: maxHeight?.toDouble(),
      );
      
      if (pickedFile == null) return null;
      return File(pickedFile.path);
    } catch (e) {
      print('ไม่สามารถถ่ายรูปได้: $e');
      return null;
    }
  }
  
  // เลือกหลายรูป
  Future<List<File>> pickMultipleImages({
    int? limit,
  }) async {
    try {
      final pickedFiles = await _picker.pickMultiImage(
        imageQuality: 85,
        limit: limit,
      );
      
      return pickedFiles.map((xFile) => File(xFile.path)).toList();
    } catch (e) {
      print('ไม่สามารถเลือกรูปได้: $e');
      return [];
    }
  }
  
  // เลือกวิดีโอ
  Future<File?> pickVideo({
    ImageSource source = ImageSource.gallery,
    Duration? maxDuration,
  }) async {
    try {
      final pickedFile = await _picker.pickVideo(
        source: source,
        maxDuration: maxDuration,
      );
      
      if (pickedFile == null) return null;
      return File(pickedFile.path);
    } catch (e) {
      print('ไม่สามารถเลือกวิดีโอได้: $e');
      return null;
    }
  }
  
  // Compress รูปภาพ
  Future<File?> compressImage(File imageFile) async {
    final dir = await getTemporaryDirectory();
    final ext = path.extension(imageFile.path).toLowerCase();
    final targetPath = '${dir.path}/${DateTime.now().millisecondsSinceEpoch}$ext';
    
    final result = await FlutterImageCompress.compressAndGetFile(
      imageFile.path,
      targetPath,
      quality: 70,
      minWidth: 1024,
      minHeight: 1024,
    );
    
    return result != null ? File(result.path) : null;
  }
}
```

### Image Picker Dialog

```dart
class ImagePickerBottomSheet extends StatelessWidget {
  final void Function(File image) onImageSelected;
  final bool allowMultiple;
  
  const ImagePickerBottomSheet({
    super.key,
    required this.onImageSelected,
    this.allowMultiple = false,
  });
  
  static Future<void> show({
    required BuildContext context,
    required void Function(File image) onImageSelected,
    bool allowMultiple = false,
  }) {
    return showModalBottomSheet(
      context: context,
      shape: const RoundedRectangleBorder(
        borderRadius: BorderRadius.vertical(top: Radius.circular(20)),
      ),
      builder: (context) => ImagePickerBottomSheet(
        onImageSelected: onImageSelected,
        allowMultiple: allowMultiple,
      ),
    );
  }
  
  @override
  Widget build(BuildContext context) {
    final service = ImagePickerService();
    
    return Padding(
      padding: const EdgeInsets.all(20),
      child: Column(
        mainAxisSize: MainAxisSize.min,
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          const Text(
            'เลือกรูปภาพ',
            style: TextStyle(
              fontSize: 18,
              fontWeight: FontWeight.bold,
            ),
          ),
          const SizedBox(height: 20),
          Row(
            children: [
              _OptionCard(
                icon: Icons.camera_alt,
                label: 'ถ่ายรูป',
                color: Colors.blue,
                onTap: () async {
                  Navigator.pop(context);
                  final image = await service.takePhoto();
                  if (image != null) onImageSelected(image);
                },
              ),
              const SizedBox(width: 12),
              _OptionCard(
                icon: Icons.photo_library,
                label: 'คลังรูปภาพ',
                color: Colors.green,
                onTap: () async {
                  Navigator.pop(context);
                  final image = await service.pickFromGallery();
                  if (image != null) onImageSelected(image);
                },
              ),
            ],
          ),
          const SizedBox(height: 8),
        ],
      ),
    );
  }
}

class _OptionCard extends StatelessWidget {
  final IconData icon;
  final String label;
  final Color color;
  final VoidCallback onTap;
  
  const _OptionCard({
    required this.icon,
    required this.label,
    required this.color,
    required this.onTap,
  });
  
  @override
  Widget build(BuildContext context) {
    return Expanded(
      child: InkWell(
        onTap: onTap,
        borderRadius: BorderRadius.circular(12),
        child: Container(
          padding: const EdgeInsets.all(16),
          decoration: BoxDecoration(
            color: color.withOpacity(0.1),
            borderRadius: BorderRadius.circular(12),
            border: Border.all(color: color.withOpacity(0.3)),
          ),
          child: Column(
            children: [
              Icon(icon, size: 40, color: color),
              const SizedBox(height: 8),
              Text(
                label,
                style: TextStyle(
                  color: color,
                  fontWeight: FontWeight.w600,
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

---

## 44.3 Image Cropping

```dart
import 'package:image_cropper/image_cropper.dart';

class ImageCropperService {
  // Crop รูปภาพแบบ square (สำหรับ profile)
  static Future<File?> cropSquare(File imageFile) async {
    final croppedFile = await ImageCropper().cropImage(
      sourcePath: imageFile.path,
      aspectRatio: const CropAspectRatio(ratioX: 1, ratioY: 1),
      compressFormat: ImageCompressFormat.jpg,
      compressQuality: 85,
      uiSettings: [
        AndroidUiSettings(
          toolbarTitle: 'ครอบรูปภาพ',
          toolbarColor: Colors.blue,
          toolbarWidgetColor: Colors.white,
          statusBarColor: Colors.blue[800],
          lockAspectRatio: true,
          cropStyle: CropStyle.circle,
          initAspectRatio: CropAspectRatioPreset.square,
          hideBottomControls: false,
          showCropGrid: true,
        ),
        IOSUiSettings(
          title: 'ครอบรูปภาพ',
          aspectRatioLockEnabled: true,
          resetAspectRatioEnabled: false,
          cropStyle: CropStyle.circle,
        ),
      ],
    );
    
    if (croppedFile == null) return null;
    return File(croppedFile.path);
  }
  
  // Crop รูปภาพ Banner (16:9)
  static Future<File?> cropBanner(File imageFile) async {
    final croppedFile = await ImageCropper().cropImage(
      sourcePath: imageFile.path,
      aspectRatio: const CropAspectRatio(ratioX: 16, ratioY: 9),
      compressQuality: 90,
      uiSettings: [
        AndroidUiSettings(
          toolbarTitle: 'ครอบรูป Banner',
          toolbarColor: Colors.purple,
          toolbarWidgetColor: Colors.white,
          lockAspectRatio: true,
        ),
        IOSUiSettings(
          title: 'ครอบรูป Banner',
          aspectRatioLockEnabled: true,
        ),
      ],
    );
    
    if (croppedFile == null) return null;
    return File(croppedFile.path);
  }
  
  // Crop แบบ free form
  static Future<File?> cropFree(File imageFile) async {
    final croppedFile = await ImageCropper().cropImage(
      sourcePath: imageFile.path,
      uiSettings: [
        AndroidUiSettings(
          toolbarTitle: 'ครอบรูปภาพ',
          toolbarColor: Colors.teal,
          toolbarWidgetColor: Colors.white,
          lockAspectRatio: false,
          initAspectRatio: CropAspectRatioPreset.original,
        ),
        IOSUiSettings(
          title: 'ครอบรูปภาพ',
          aspectRatioLockEnabled: false,
          resetAspectRatioEnabled: true,
        ),
      ],
    );
    
    if (croppedFile == null) return null;
    return File(croppedFile.path);
  }
}
```

---

## 44.4 camera Package - กล้องขั้น Advanced

```dart
import 'package:camera/camera.dart';

class CameraScreen extends StatefulWidget {
  final List<CameraDescription> cameras;
  
  const CameraScreen({super.key, required this.cameras});
  
  @override
  State<CameraScreen> createState() => _CameraScreenState();
}

class _CameraScreenState extends State<CameraScreen> 
    with WidgetsBindingObserver {
  CameraController? _controller;
  int _selectedCameraIndex = 0;
  bool _isRecording = false;
  double _zoomLevel = 1.0;
  double _minZoom = 1.0;
  double _maxZoom = 1.0;
  FlashMode _flashMode = FlashMode.auto;
  
  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
    _initCamera(widget.cameras[0]);
  }
  
  Future<void> _initCamera(CameraDescription camera) async {
    final controller = CameraController(
      camera,
      ResolutionPreset.high,
      enableAudio: true,
      imageFormatGroup: ImageFormatGroup.jpeg,
    );
    
    await controller.initialize();
    
    // ดึงค่า zoom range
    _minZoom = await controller.getMinZoomLevel();
    _maxZoom = await controller.getMaxZoomLevel();
    
    if (mounted) {
      setState(() {
        _controller = controller;
      });
    }
  }
  
  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    final controller = _controller;
    if (controller == null || !controller.value.isInitialized) return;
    
    if (state == AppLifecycleState.inactive) {
      controller.dispose();
    } else if (state == AppLifecycleState.resumed) {
      _initCamera(controller.description);
    }
  }
  
  // สลับกล้องหน้า-หลัง
  Future<void> _switchCamera() async {
    _selectedCameraIndex = 
        (_selectedCameraIndex + 1) % widget.cameras.length;
    
    await _controller?.dispose();
    await _initCamera(widget.cameras[_selectedCameraIndex]);
  }
  
  // ถ่ายรูป
  Future<void> _takePicture() async {
    final controller = _controller;
    if (controller == null || !controller.value.isInitialized) return;
    
    try {
      final image = await controller.takePicture();
      
      if (mounted) {
        // นำรูปไปแสดงหรือบันทึก
        Navigator.pop(context, File(image.path));
      }
    } catch (e) {
      print('ถ่ายรูปไม่สำเร็จ: $e');
    }
  }
  
  // บันทึกวิดีโอ
  Future<void> _toggleRecording() async {
    final controller = _controller;
    if (controller == null || !controller.value.isInitialized) return;
    
    if (_isRecording) {
      // หยุดบันทึก
      final video = await controller.stopVideoRecording();
      setState(() => _isRecording = false);
      
      if (mounted) {
        Navigator.pop(context, File(video.path));
      }
    } else {
      // เริ่มบันทึก
      await controller.startVideoRecording();
      setState(() => _isRecording = true);
    }
  }
  
  // ตั้งค่า Flash
  Future<void> _toggleFlash() async {
    final modes = [FlashMode.auto, FlashMode.always, FlashMode.off];
    final currentIndex = modes.indexOf(_flashMode);
    final nextMode = modes[(currentIndex + 1) % modes.length];
    
    await _controller?.setFlashMode(nextMode);
    setState(() => _flashMode = nextMode);
  }
  
  // Zoom ด้วย gesture
  void _handleScaleUpdate(ScaleUpdateDetails details) {
    final newZoom = (_zoomLevel * details.scale)
        .clamp(_minZoom, _maxZoom);
    
    _controller?.setZoomLevel(newZoom);
    setState(() => _zoomLevel = newZoom);
  }
  
  @override
  Widget build(BuildContext context) {
    final controller = _controller;
    
    if (controller == null || !controller.value.isInitialized) {
      return const Scaffold(
        backgroundColor: Colors.black,
        body: Center(
          child: CircularProgressIndicator(color: Colors.white),
        ),
      );
    }
    
    return Scaffold(
      backgroundColor: Colors.black,
      body: Stack(
        children: [
          // Camera Preview
          Positioned.fill(
            child: GestureDetector(
              onScaleUpdate: _handleScaleUpdate,
              child: CameraPreview(controller),
            ),
          ),
          
          // Top Controls
          Positioned(
            top: 0,
            left: 0,
            right: 0,
            child: SafeArea(
              child: Padding(
                padding: const EdgeInsets.all(16),
                child: Row(
                  mainAxisAlignment: MainAxisAlignment.spaceBetween,
                  children: [
                    IconButton(
                      onPressed: () => Navigator.pop(context),
                      icon: const Icon(
                        Icons.close,
                        color: Colors.white,
                        size: 28,
                      ),
                    ),
                    // Flash button
                    IconButton(
                      onPressed: _toggleFlash,
                      icon: Icon(
                        _flashMode == FlashMode.auto
                            ? Icons.flash_auto
                            : _flashMode == FlashMode.always
                                ? Icons.flash_on
                                : Icons.flash_off,
                        color: Colors.white,
                        size: 28,
                      ),
                    ),
                  ],
                ),
              ),
            ),
          ),
          
          // Zoom indicator
          if (_zoomLevel > 1.0)
            Positioned(
              bottom: 120,
              left: 0,
              right: 0,
              child: Center(
                child: Container(
                  padding: const EdgeInsets.symmetric(
                    horizontal: 12,
                    vertical: 4,
                  ),
                  decoration: BoxDecoration(
                    color: Colors.black54,
                    borderRadius: BorderRadius.circular(20),
                  ),
                  child: Text(
                    '${_zoomLevel.toStringAsFixed(1)}x',
                    style: const TextStyle(
                      color: Colors.white,
                      fontSize: 16,
                    ),
                  ),
                ),
              ),
            ),
          
          // Bottom Controls
          Positioned(
            bottom: 0,
            left: 0,
            right: 0,
            child: SafeArea(
              child: Padding(
                padding: const EdgeInsets.all(24),
                child: Row(
                  mainAxisAlignment: MainAxisAlignment.spaceEvenly,
                  children: [
                    // Gallery button
                    IconButton(
                      onPressed: () async {
                        final service = ImagePickerService();
                        final image = await service.pickFromGallery();
                        if (image != null && mounted) {
                          Navigator.pop(context, image);
                        }
                      },
                      icon: const Icon(
                        Icons.photo_library,
                        color: Colors.white,
                        size: 32,
                      ),
                    ),
                    
                    // Shutter button
                    GestureDetector(
                      onTap: _isRecording ? null : _takePicture,
                      onLongPress: _toggleRecording,
                      child: Container(
                        width: 72,
                        height: 72,
                        decoration: BoxDecoration(
                          shape: BoxShape.circle,
                          border: Border.all(
                            color: Colors.white,
                            width: 3,
                          ),
                          color: _isRecording ? Colors.red : Colors.white,
                        ),
                        child: _isRecording
                            ? const Icon(
                                Icons.stop,
                                color: Colors.white,
                                size: 32,
                              )
                            : null,
                      ),
                    ),
                    
                    // Switch camera button
                    if (widget.cameras.length > 1)
                      IconButton(
                        onPressed: _switchCamera,
                        icon: const Icon(
                          Icons.flip_camera_ios,
                          color: Colors.white,
                          size: 32,
                        ),
                      ),
                  ],
                ),
              ),
            ),
          ),
        ],
      ),
    );
  }
  
  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    _controller?.dispose();
    super.dispose();
  }
}
```

---

## 44.5 Permission Handling

```dart
import 'package:permission_handler/permission_handler.dart';

class PermissionService {
  // ขอ Camera permission
  static Future<bool> requestCameraPermission() async {
    final status = await Permission.camera.request();
    return status.isGranted;
  }
  
  // ขอ Photos permission
  static Future<bool> requestPhotosPermission() async {
    // Android 13+
    if (Platform.isAndroid) {
      final sdkVersion = await _getAndroidSdkVersion();
      if (sdkVersion >= 33) {
        final status = await Permission.photos.request();
        return status.isGranted;
      } else {
        final status = await Permission.storage.request();
        return status.isGranted;
      }
    }
    
    // iOS
    final status = await Permission.photos.request();
    return status.isGranted;
  }
  
  // ตรวจสอบและขอ permissions ที่จำเป็น
  static Future<Map<Permission, PermissionStatus>> 
      requestCameraAndPhotos() async {
    return await [
      Permission.camera,
      Permission.photos,
    ].request();
  }
  
  // แสดง dialog ให้ไปที่ settings เมื่อ permission ถูก deny permanently
  static Future<void> showPermissionDialog(
    BuildContext context,
    String permissionName,
  ) async {
    await showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('ต้องการสิทธิ์การเข้าถึง'),
        content: Text(
          'แอปต้องการสิทธิ์ "$permissionName" เพื่อใช้งานฟีเจอร์นี้ '
          'กรุณาไปที่การตั้งค่าเพื่ออนุญาต',
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('ยกเลิก'),
          ),
          TextButton(
            onPressed: () {
              Navigator.pop(context);
              openAppSettings();
            },
            child: const Text('ไปที่การตั้งค่า'),
          ),
        ],
      ),
    );
  }
}
```

---

## 44.6 Workshop: Profile Photo Selector

```dart
// profile_photo_selector.dart
import 'dart:io';
import 'package:flutter/material.dart';

class ProfilePhotoSelector extends StatefulWidget {
  final String userId;
  final String? currentPhotoUrl;
  
  const ProfilePhotoSelector({
    super.key,
    required this.userId,
    this.currentPhotoUrl,
  });
  
  @override
  State<ProfilePhotoSelector> createState() => _ProfilePhotoSelectorState();
}

class _ProfilePhotoSelectorState extends State<ProfilePhotoSelector> {
  final ImagePickerService _pickerService = ImagePickerService();
  
  File? _selectedFile;
  bool _isLoading = false;
  
  Future<void> _selectPhoto(ImageSource source) async {
    Navigator.pop(context); // ปิด bottom sheet
    
    File? image;
    
    if (source == ImageSource.camera) {
      // ขอ permission กล้อง
      final hasPermission = 
          await PermissionService.requestCameraPermission();
      if (!hasPermission) {
        if (mounted) {
          await PermissionService.showPermissionDialog(
            context, 'กล้องถ่ายภาพ',
          );
        }
        return;
      }
      image = await _pickerService.takePhoto(imageQuality: 90);
    } else {
      // ขอ permission รูปภาพ
      final hasPermission = 
          await PermissionService.requestPhotosPermission();
      if (!hasPermission) {
        if (mounted) {
          await PermissionService.showPermissionDialog(
            context, 'คลังรูปภาพ',
          );
        }
        return;
      }
      image = await _pickerService.pickFromGallery(imageQuality: 90);
    }
    
    if (image == null) return;
    
    // Crop รูปเป็น square
    final croppedImage = await ImageCropperService.cropSquare(image);
    if (croppedImage == null) return;
    
    // Compress
    final compressed = await _pickerService.compressImage(croppedImage);
    
    setState(() {
      _selectedFile = compressed ?? croppedImage;
    });
  }
  
  Future<void> _uploadPhoto() async {
    if (_selectedFile == null) return;
    
    setState(() => _isLoading = true);
    
    try {
      final photoService = ProfilePhotoService();
      final url = await photoService.uploadProfilePhoto(
        userId: widget.userId,
        imageFile: _selectedFile!,
      );
      
      if (mounted) {
        Navigator.pop(context, url);
      }
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('เกิดข้อผิดพลาด: $e')),
        );
      }
    } finally {
      if (mounted) setState(() => _isLoading = false);
    }
  }
  
  void _showSourcePicker() {
    showModalBottomSheet(
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
              'เลือกรูปโปรไฟล์',
              style: TextStyle(
                fontSize: 18,
                fontWeight: FontWeight.bold,
              ),
            ),
            const SizedBox(height: 20),
            ListTile(
              leading: const CircleAvatar(
                backgroundColor: Colors.blue,
                child: Icon(Icons.camera_alt, color: Colors.white),
              ),
              title: const Text('ถ่ายรูปใหม่'),
              subtitle: const Text('ใช้กล้องถ่ายรูปโปรไฟล์'),
              onTap: () => _selectPhoto(ImageSource.camera),
            ),
            ListTile(
              leading: const CircleAvatar(
                backgroundColor: Colors.green,
                child: Icon(Icons.photo_library, color: Colors.white),
              ),
              title: const Text('เลือกจากคลังรูปภาพ'),
              subtitle: const Text('เลือกรูปที่มีอยู่แล้ว'),
              onTap: () => _selectPhoto(ImageSource.gallery),
            ),
            if (widget.currentPhotoUrl != null || _selectedFile != null)
              ListTile(
                leading: const CircleAvatar(
                  backgroundColor: Colors.red,
                  child: Icon(Icons.delete, color: Colors.white),
                ),
                title: const Text('ลบรูปโปรไฟล์'),
                onTap: () {
                  Navigator.pop(context);
                  setState(() => _selectedFile = null);
                },
              ),
          ],
        ),
      ),
    );
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('เลือกรูปโปรไฟล์'),
        actions: [
          if (_selectedFile != null && !_isLoading)
            TextButton(
              onPressed: _uploadPhoto,
              child: const Text(
                'บันทึก',
                style: TextStyle(
                  color: Colors.white,
                  fontWeight: FontWeight.bold,
                  fontSize: 16,
                ),
              ),
            ),
        ],
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            // Photo Preview
            GestureDetector(
              onTap: _showSourcePicker,
              child: Stack(
                alignment: Alignment.center,
                children: [
                  CircleAvatar(
                    radius: 100,
                    backgroundColor: Colors.grey[200],
                    backgroundImage: _selectedFile != null
                        ? FileImage(_selectedFile!) as ImageProvider
                        : (widget.currentPhotoUrl != null
                            ? NetworkImage(widget.currentPhotoUrl!)
                            : null),
                    child: (_selectedFile == null && 
                            widget.currentPhotoUrl == null)
                        ? const Icon(
                            Icons.person,
                            size: 80,
                            color: Colors.grey,
                          )
                        : null,
                  ),
                  if (_isLoading)
                    const CircularProgressIndicator(),
                  if (!_isLoading)
                    Positioned(
                      bottom: 4,
                      right: 4,
                      child: CircleAvatar(
                        radius: 20,
                        backgroundColor: Theme.of(context).primaryColor,
                        child: const Icon(
                          Icons.camera_alt,
                          color: Colors.white,
                          size: 20,
                        ),
                      ),
                    ),
                ],
              ),
            ),
            
            const SizedBox(height: 24),
            
            Text(
              _selectedFile != null
                  ? 'กดปุ่ม "บันทึก" เพื่อใช้รูปนี้'
                  : 'กดที่รูปเพื่อเลือกหรือถ่ายรูปใหม่',
              style: const TextStyle(
                color: Colors.grey,
                fontSize: 14,
              ),
            ),
            
            const SizedBox(height: 32),
            
            ElevatedButton.icon(
              onPressed: _showSourcePicker,
              icon: const Icon(Icons.edit),
              label: const Text('เปลี่ยนรูปโปรไฟล์'),
              style: ElevatedButton.styleFrom(
                padding: const EdgeInsets.symmetric(
                  horizontal: 24,
                  vertical: 12,
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

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- การใช้ `image_picker` เพื่อเลือกรูปจาก gallery และกล้อง
- การ Crop รูปภาพด้วย `image_cropper`
- การใช้ `camera` package สำหรับ custom camera UI
- การจัดการ Permissions บน Android และ iOS
- Workshop: Profile Photo Selector แบบสมบูรณ์

**แบบฝึกหัดเพิ่มเติม:**
1. เพิ่ม filter รูปภาพ (brightness, contrast, saturation)
2. Implement QR Code scanner ด้วย camera
3. สร้าง photo collage จากหลายรูป
4. เพิ่ม face detection และ auto-crop ใบหน้า
