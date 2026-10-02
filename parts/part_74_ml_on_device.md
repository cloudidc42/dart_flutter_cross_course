# Part 74: Machine Learning on Device - ML บนอุปกรณ์

## บทนำ

On-device ML ช่วยให้แอปทำงาน AI ได้โดยไม่ต้องพึ่งอินเทอร์เน็ต ข้อมูลของผู้ใช้ยังคงอยู่ในอุปกรณ์ และ latency ต่ำกว่าการเรียก API มาก

## 1. tflite_flutter Package

### การติดตั้ง

```yaml
# pubspec.yaml
dependencies:
  tflite_flutter: ^0.10.4
  image: ^4.1.3
  camera: ^0.10.5+5

flutter:
  assets:
    - assets/models/
    - assets/labels/
```

### โหลดและรัน Model

```dart
// lib/ml/model_helper.dart
import 'package:tflite_flutter/tflite_flutter.dart';
import 'package:flutter/services.dart';

class TFLiteModelHelper {
  Interpreter? _interpreter;
  List<String>? _labels;

  // โหลด model จาก assets
  Future<void> loadModel({
    required String modelPath,
    String? labelsPath,
  }) async {
    try {
      // โหลด model
      _interpreter = await Interpreter.fromAsset(modelPath);
      
      // โหลด labels ถ้ามี
      if (labelsPath != null) {
        await _loadLabels(labelsPath);
      }
      
      // แสดง input/output details
      final inputDetails = _interpreter!.getInputTensors();
      final outputDetails = _interpreter!.getOutputTensors();
      
      print('Input: ${inputDetails.map((t) => t.shape)}');
      print('Output: ${outputDetails.map((t) => t.shape)}');
    } catch (e) {
      print('Error loading model: $e');
      rethrow;
    }
  }

  Future<void> _loadLabels(String path) async {
    final rawLabels = await rootBundle.loadString(path);
    _labels = rawLabels.split('\n')
        .map((l) => l.trim())
        .where((l) => l.isNotEmpty)
        .toList();
  }

  List<String>? get labels => _labels;

  void dispose() {
    _interpreter?.close();
  }
}
```

## 2. Image Classification

### Image Preprocessing

```dart
// lib/ml/image_classifier.dart
import 'dart:io';
import 'dart:typed_data';
import 'package:image/image.dart' as img;
import 'package:tflite_flutter/tflite_flutter.dart';
import 'package:flutter/services.dart';

class ImageClassifier {
  static const int _inputSize = 224;
  static const int _numChannels = 3;
  static const int _numResults = 5;

  Interpreter? _interpreter;
  List<String>? _labels;
  bool _isLoaded = false;

  Future<void> initialize() async {
    if (_isLoaded) return;

    // โหลด MobileNet model
    _interpreter = await Interpreter.fromAsset(
      'assets/models/mobilenet_v2.tflite',
    );

    // โหลด labels
    final labelsData = await rootBundle.loadString(
      'assets/labels/imagenet_labels.txt',
    );
    _labels = labelsData.split('\n')
        .where((l) => l.trim().isNotEmpty)
        .toList();

    _isLoaded = true;
  }

  // จำแนก image จาก File
  Future<List<ClassificationResult>> classifyImage(File imageFile) async {
    if (!_isLoaded) await initialize();

    // อ่าน image
    final imageBytes = await imageFile.readAsBytes();
    final image = img.decodeImage(imageBytes);
    if (image == null) throw Exception('Cannot decode image');

    // Preprocess image
    final input = _preprocessImage(image);

    // รัน inference
    final output = List.filled(
      1 * 1001, // MobileNet มี 1001 classes
      0.0,
    ).reshape([1, 1001]);

    _interpreter!.run(input, output);

    // Post-process results
    return _postProcess(output[0] as List<double>);
  }

  // Preprocess image เป็น tensor
  List<List<List<List<double>>>> _preprocessImage(img.Image image) {
    // Resize เป็น 224x224
    final resized = img.copyResize(
      image,
      width: _inputSize,
      height: _inputSize,
    );

    // แปลงเป็น [1, 224, 224, 3] float array
    final input = List.generate(
      1,
      (_) => List.generate(
        _inputSize,
        (y) => List.generate(
          _inputSize,
          (x) {
            final pixel = resized.getPixel(x, y);
            // Normalize ค่า pixel เป็น [-1, 1]
            return [
              (pixel.r.toDouble() - 127.5) / 127.5,
              (pixel.g.toDouble() - 127.5) / 127.5,
              (pixel.b.toDouble() - 127.5) / 127.5,
            ];
          },
        ),
      ),
    );

    return input;
  }

  // Post-process output เป็น classification results
  List<ClassificationResult> _postProcess(List<double> output) {
    // สร้าง pairs ของ index และ confidence
    final results = output.asMap().entries.map(
      (e) => ClassificationResult(
        label: _labels != null && e.key < _labels!.length
            ? _labels![e.key]
            : 'Unknown (${e.key})',
        confidence: e.value,
        index: e.key,
      ),
    ).toList();

    // Sort โดย confidence สูงสุด
    results.sort((a, b) => b.confidence.compareTo(a.confidence));

    // Return top results
    return results.take(_numResults).toList();
  }

  void dispose() {
    _interpreter?.close();
    _isLoaded = false;
  }
}

class ClassificationResult {
  final String label;
  final double confidence;
  final int index;

  const ClassificationResult({
    required this.label,
    required this.confidence,
    required this.index,
  });

  String get formattedConfidence =>
      '${(confidence * 100).toStringAsFixed(1)}%';
}
```

## 3. Text Analysis

### Text Classifier (Sentiment Analysis)

```dart
// lib/ml/text_classifier.dart
import 'package:tflite_flutter/tflite_flutter.dart';
import 'package:flutter/services.dart';
import 'dart:typed_data';

class TextClassifier {
  static const int _maxSequenceLength = 256;
  static const int _vocabSize = 10000;

  Interpreter? _interpreter;
  Map<String, int>? _vocab;

  Future<void> initialize() async {
    // โหลด model
    _interpreter = await Interpreter.fromAsset(
      'assets/models/text_classification.tflite',
    );

    // โหลด vocabulary
    final vocabData = await rootBundle.loadString(
      'assets/models/vocab.json',
    );
    // Parse vocab...
    // _vocab = json.decode(vocabData) as Map<String, int>;
  }

  Future<SentimentResult> analyzeSentiment(String text) async {
    if (_interpreter == null) await initialize();

    // Tokenize text
    final tokens = _tokenize(text);

    // Pad หรือ truncate ให้ได้ length ที่กำหนด
    final paddedTokens = _padTokens(tokens);

    // สร้าง input tensor
    final input = [paddedTokens];
    final output = List.filled(2, 0.0).reshape([1, 2]);

    _interpreter!.run(input, output);

    final scores = output[0] as List<double>;
    final positiveScore = scores[1];
    final negativeScore = scores[0];

    return SentimentResult(
      text: text,
      positiveScore: positiveScore,
      negativeScore: negativeScore,
      sentiment: positiveScore > negativeScore
          ? Sentiment.positive
          : Sentiment.negative,
    );
  }

  List<int> _tokenize(String text) {
    if (_vocab == null) return [];

    return text
        .toLowerCase()
        .split(RegExp(r'\s+'))
        .map((word) => _vocab![word] ?? 0)
        .toList();
  }

  List<int> _padTokens(List<int> tokens) {
    final padded = List<int>.filled(_maxSequenceLength, 0);
    final length = tokens.length.clamp(0, _maxSequenceLength);
    for (int i = 0; i < length; i++) {
      padded[i] = tokens[i];
    }
    return padded;
  }

  void dispose() {
    _interpreter?.close();
  }
}

enum Sentiment { positive, negative, neutral }

class SentimentResult {
  final String text;
  final double positiveScore;
  final double negativeScore;
  final Sentiment sentiment;

  const SentimentResult({
    required this.text,
    required this.positiveScore,
    required this.negativeScore,
    required this.sentiment,
  });

  String get sentimentLabel {
    switch (sentiment) {
      case Sentiment.positive:
        return 'เชิงบวก 😊';
      case Sentiment.negative:
        return 'เชิงลบ 😔';
      case Sentiment.neutral:
        return 'กลาง 😐';
    }
  }
}
```

## 4. Workshop: Image Classifier App

### Camera Screen

```dart
// lib/screens/camera_screen.dart
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:camera/camera.dart';
import 'package:image_picker/image_picker.dart';
import '../ml/image_classifier.dart';
import 'results_screen.dart';

class CameraScreen extends StatefulWidget {
  const CameraScreen({super.key});

  @override
  State<CameraScreen> createState() => _CameraScreenState();
}

class _CameraScreenState extends State<CameraScreen> {
  CameraController? _cameraController;
  List<CameraDescription>? _cameras;
  bool _isLoading = false;
  final _classifier = ImageClassifier();
  final _picker = ImagePicker();

  @override
  void initState() {
    super.initState();
    _initCamera();
    _classifier.initialize();
  }

  Future<void> _initCamera() async {
    _cameras = await availableCameras();
    if (_cameras!.isEmpty) return;

    _cameraController = CameraController(
      _cameras![0],
      ResolutionPreset.medium,
    );

    await _cameraController!.initialize();
    if (mounted) setState(() {});
  }

  Future<void> _takePicture() async {
    if (_cameraController == null || !_cameraController!.value.isInitialized) {
      return;
    }

    setState(() => _isLoading = true);

    try {
      final image = await _cameraController!.takePicture();
      await _classifyImage(File(image.path));
    } finally {
      if (mounted) setState(() => _isLoading = false);
    }
  }

  Future<void> _pickFromGallery() async {
    final pickedFile = await _picker.pickImage(
      source: ImageSource.gallery,
    );
    if (pickedFile == null) return;

    setState(() => _isLoading = true);
    try {
      await _classifyImage(File(pickedFile.path));
    } finally {
      if (mounted) setState(() => _isLoading = false);
    }
  }

  Future<void> _classifyImage(File imageFile) async {
    final results = await _classifier.classifyImage(imageFile);

    if (!mounted) return;

    Navigator.push(
      context,
      MaterialPageRoute(
        builder: (context) => ResultsScreen(
          imageFile: imageFile,
          results: results,
        ),
      ),
    );
  }

  @override
  void dispose() {
    _cameraController?.dispose();
    _classifier.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Image Classifier'),
        backgroundColor: Colors.black,
        foregroundColor: Colors.white,
      ),
      backgroundColor: Colors.black,
      body: Stack(
        children: [
          if (_cameraController != null &&
              _cameraController!.value.isInitialized)
            CameraPreview(_cameraController!)
          else
            const Center(
              child: CircularProgressIndicator(color: Colors.white),
            ),

          if (_isLoading)
            Container(
              color: Colors.black54,
              child: const Center(
                child: Column(
                  mainAxisSize: MainAxisSize.min,
                  children: [
                    CircularProgressIndicator(color: Colors.white),
                    SizedBox(height: 16),
                    Text(
                      'กำลังวิเคราะห์...',
                      style: TextStyle(color: Colors.white),
                    ),
                  ],
                ),
              ),
            ),

          Positioned(
            bottom: 40,
            left: 0,
            right: 0,
            child: Row(
              mainAxisAlignment: MainAxisAlignment.spaceEvenly,
              children: [
                FloatingActionButton(
                  onPressed: _pickFromGallery,
                  heroTag: 'gallery',
                  child: const Icon(Icons.photo_library),
                ),
                FloatingActionButton(
                  onPressed: _takePicture,
                  heroTag: 'capture',
                  backgroundColor: Colors.white,
                  child: const Icon(Icons.camera_alt, color: Colors.black),
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}
```

### Results Screen

```dart
// lib/screens/results_screen.dart
import 'dart:io';
import 'package:flutter/material.dart';
import '../ml/image_classifier.dart';

class ResultsScreen extends StatelessWidget {
  final File imageFile;
  final List<ClassificationResult> results;

  const ResultsScreen({
    super.key,
    required this.imageFile,
    required this.results,
  });

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('ผลการวิเคราะห์'),
      ),
      body: Column(
        children: [
          // แสดงรูปภาพ
          Expanded(
            flex: 2,
            child: Container(
              width: double.infinity,
              margin: const EdgeInsets.all(16),
              decoration: BoxDecoration(
                borderRadius: BorderRadius.circular(16),
                boxShadow: [
                  BoxShadow(
                    color: Colors.black.withOpacity(0.2),
                    blurRadius: 8,
                    offset: const Offset(0, 4),
                  ),
                ],
              ),
              child: ClipRRect(
                borderRadius: BorderRadius.circular(16),
                child: Image.file(
                  imageFile,
                  fit: BoxFit.cover,
                ),
              ),
            ),
          ),

          // แสดงผลลัพธ์
          Expanded(
            flex: 3,
            child: Padding(
              padding: const EdgeInsets.all(16),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(
                    'ผลการจำแนก:',
                    style: Theme.of(context).textTheme.titleLarge?.copyWith(
                          fontWeight: FontWeight.bold,
                        ),
                  ),
                  const SizedBox(height: 12),
                  ...results.map(
                    (result) => _buildResultItem(context, result),
                  ),
                ],
              ),
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildResultItem(BuildContext context, ClassificationResult result) {
    return Padding(
      padding: const EdgeInsets.only(bottom: 12),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Row(
            mainAxisAlignment: MainAxisAlignment.spaceBetween,
            children: [
              Expanded(
                child: Text(
                  result.label,
                  style: Theme.of(context).textTheme.bodyMedium?.copyWith(
                        fontWeight: result == results.first
                            ? FontWeight.bold
                            : FontWeight.normal,
                      ),
                  overflow: TextOverflow.ellipsis,
                ),
              ),
              Text(
                result.formattedConfidence,
                style: TextStyle(
                  color: result == results.first
                      ? Theme.of(context).colorScheme.primary
                      : Colors.grey,
                  fontWeight: result == results.first
                      ? FontWeight.bold
                      : FontWeight.normal,
                ),
              ),
            ],
          ),
          const SizedBox(height: 4),
          LinearProgressIndicator(
            value: result.confidence,
            backgroundColor: Colors.grey[200],
            color: result == results.first
                ? Theme.of(context).colorScheme.primary
                : Colors.grey[400],
          ),
        ],
      ),
    );
  }
}
```

### Real-time Object Detection

```dart
// lib/ml/object_detector.dart
import 'dart:typed_data';
import 'package:tflite_flutter/tflite_flutter.dart';
import 'package:image/image.dart' as img;

class DetectionResult {
  final String label;
  final double confidence;
  final Rect boundingBox; // normalized (0-1)

  const DetectionResult({
    required this.label,
    required this.confidence,
    required this.boundingBox,
  });
}

class Rect {
  final double left;
  final double top;
  final double right;
  final double bottom;

  const Rect({
    required this.left,
    required this.top,
    required this.right,
    required this.bottom,
  });

  double get width => right - left;
  double get height => bottom - top;
}

class ObjectDetector {
  static const int _inputSize = 300;
  
  Interpreter? _interpreter;
  List<String>? _labels;

  Future<void> initialize() async {
    _interpreter = await Interpreter.fromAsset(
      'assets/models/ssd_mobilenet.tflite',
    );
  }

  Future<List<DetectionResult>> detect(img.Image image) async {
    if (_interpreter == null) await initialize();

    final input = _preprocessImage(image);
    
    // SSD output tensors
    final outputLocations = List.filled(1 * 10 * 4, 0.0).reshape([1, 10, 4]);
    final outputClasses = List.filled(1 * 10, 0.0).reshape([1, 10]);
    final outputScores = List.filled(1 * 10, 0.0).reshape([1, 10]);
    final numDetections = [0.0];

    final outputs = {
      0: outputLocations,
      1: outputClasses,
      2: outputScores,
      3: numDetections,
    };

    _interpreter!.runForMultipleInputs([input], outputs);

    final count = numDetections[0].toInt();
    final results = <DetectionResult>[];

    for (int i = 0; i < count; i++) {
      final score = (outputScores[0] as List<double>)[i];
      if (score < 0.5) continue;

      final classIndex = (outputClasses[0] as List<double>)[i].toInt();
      final bbox = (outputLocations[0] as List<List<double>>)[i];

      results.add(DetectionResult(
        label: _labels != null && classIndex < _labels!.length
            ? _labels![classIndex]
            : 'Object $classIndex',
        confidence: score,
        boundingBox: Rect(
          top: bbox[0],
          left: bbox[1],
          bottom: bbox[2],
          right: bbox[3],
        ),
      ));
    }

    return results;
  }

  List<List<List<List<int>>>> _preprocessImage(img.Image image) {
    final resized = img.copyResize(
      image,
      width: _inputSize,
      height: _inputSize,
    );

    return List.generate(
      1,
      (_) => List.generate(
        _inputSize,
        (y) => List.generate(
          _inputSize,
          (x) {
            final pixel = resized.getPixel(x, y);
            return [pixel.r.toInt(), pixel.g.toInt(), pixel.b.toInt()];
          },
        ),
      ),
    );
  }

  void dispose() {
    _interpreter?.close();
  }
}
```

## สรุป

On-device ML ใน Flutter:
1. **Privacy** - ข้อมูลไม่ออกจากอุปกรณ์
2. **Offline** - ทำงานได้ไม่ต้องใช้อินเทอร์เน็ต
3. **Low latency** - ไม่ต้องรอ network
4. **tflite_flutter** เป็น package หลักสำหรับ TensorFlow Lite

## แบบทดสอบ

1. อธิบาย preprocessing ที่จำเป็นก่อนส่ง image เข้า TFLite model
2. Quantization คืออะไรและทำไมจึงสำคัญสำหรับ mobile ML?
3. เพิ่ม face detection ใน Camera screen โดยใช้ Google ML Kit
4. สร้าง Isolate สำหรับ ML inference เพื่อไม่ให้ UI freeze
