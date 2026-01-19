# HƯỚNG DẪN KẾT NỐI FLUTTER VỚI LABAPI SỬ DỤNG DIO

Tài liệu này hướng dẫn chi tiết cách kết nối ứng dụng Flutter với **LabAPI (ASP.NET Core 8)** sử dụng thư viện **Dio** để thực hiện các thao tác CRUD cho Post, Comment, Reply và Like.

---

## 1. Cài đặt thư viện (Dependencies)

Mở file **`pubspec.yaml`** và thêm các dependencies sau:

```yaml
dependencies:
  flutter:
    sdk: flutter
  dio: ^5.4.0
  http_parser: ^4.0.2
  image_picker: ^1.0.7  # Để chọn ảnh từ thiết bị
```

Sau đó chạy lệnh:

```bash
flutter pub get
```

---

## 2. Cấu hình Base URL

Tạo file **`lib/config/api_config.dart`** để cấu hình thông tin API:

```dart
class ApiConfig {
  // Thay đổi base URL phù hợp với môi trường của bạn
  static const String baseUrl = "https://lab-api.aiotlab.edu.vn";
  
  // Cho development (Android Emulator)
  // static const String baseUrl = "http://10.0.2.2:5000";
  
  // Cho development (iOS Simulator hoặc thiết bị thật)
  // static const String baseUrl = "http://localhost:5000";
  
  static const String apiVersion = "v1";
  static const String postsEndpoint = "/api/$apiVersion/posts";
  static const String commentsEndpoint = "/api/$apiVersion/comments";
}
```

**Lưu ý quan trọng về Base URL:**
- **Android Emulator**: Sử dụng `10.0.2.2` thay vì `localhost`
- **iOS Simulator**: Có thể dùng `localhost`
- **Thiết bị thật**: Phải dùng địa chỉ IP của máy chạy API (VD: `http://192.168.1.100:5000`)
- **Production**: Sử dụng domain thực tế (VD: `https://api.example.com`)

---

## 3. Tạo Models (Data Transfer Objects)

### 3.1. Post Model

Tạo file **`lib/models/post_model.dart`**:

```dart
import 'comment_model.dart';

class PostModel {
  final int id;
  final String title;
  final String content;
  final String? imageUrl;
  final DateTime createdAt;
  final DateTime? updatedAt;
  final int likeCount;
  final List<CommentModel> comments;

  PostModel({
    required this.id,
    required this.title,
    required this.content,
    this.imageUrl,
    required this.createdAt,
    this.updatedAt,
    required this.likeCount,
    required this.comments,
  });

  factory PostModel.fromJson(Map<String, dynamic> json) {
    return PostModel(
      id: json['id'],
      title: json['title'],
      content: json['content'],
      imageUrl: json['imageUrl'],
      createdAt: DateTime.parse(json['createdAt']),
      updatedAt: json['updatedAt'] != null 
          ? DateTime.parse(json['updatedAt']) 
          : null,
      likeCount: json['likeCount'],
      comments: (json['comments'] as List)
          .map((c) => CommentModel.fromJson(c))
          .toList(),
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'title': title,
      'content': content,
      'imageUrl': imageUrl,
      'createdAt': createdAt.toIso8601String(),
      'updatedAt': updatedAt?.toIso8601String(),
      'likeCount': likeCount,
      'comments': comments.map((c) => c.toJson()).toList(),
    };
  }
}
```

### 3.2. Comment Model

Tạo file **`lib/models/comment_model.dart`**:

```dart
class CommentModel {
  final int id;
  final String content;
  final DateTime createdAt;
  final int likeCount;
  final List<CommentModel> replies;

  CommentModel({
    required this.id,
    required this.content,
    required this.createdAt,
    required this.likeCount,
    required this.replies,
  });

  factory CommentModel.fromJson(Map<String, dynamic> json) {
    return CommentModel(
      id: json['id'],
      content: json['content'],
      createdAt: DateTime.parse(json['createdAt']),
      likeCount: json['likeCount'],
      replies: (json['replies'] as List)
          .map((r) => CommentModel.fromJson(r))
          .toList(),
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'content': content,
      'createdAt': createdAt.toIso8601String(),
      'likeCount': likeCount,
      'replies': replies.map((r) => r.toJson()).toList(),
    };
  }
}
```

### 3.3. Like Response Model

Tạo file **`lib/models/like_response_model.dart`**:

```dart
class LikeResponseModel {
  final int targetId;
  final int likeCount;

  LikeResponseModel({
    required this.targetId,
    required this.likeCount,
  });

  factory LikeResponseModel.fromJson(Map<String, dynamic> json) {
    return LikeResponseModel(
      targetId: json['targetId'],
      likeCount: json['likeCount'],
    );
  }
}
```

---

## 4. Tạo API Service

Tạo file **`lib/services/api_service.dart`** để xử lý tất cả các request API:

```dart
import 'dart:io';
import 'package:dio/dio.dart';
import 'package:http_parser/http_parser.dart';
import '../config/api_config.dart';
import '../models/post_model.dart';
import '../models/comment_model.dart';
import '../models/like_response_model.dart';

class ApiService {
  late Dio _dio;

  ApiService() {
    _dio = Dio(BaseOptions(
      baseUrl: ApiConfig.baseUrl,
      connectTimeout: const Duration(seconds: 30),
      receiveTimeout: const Duration(seconds: 30),
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
      },
    ));

    // Thêm interceptor để log request/response (Optional)
    _dio.interceptors.add(LogInterceptor(
      requestBody: true,
      responseBody: true,
      error: true,
    ));
  }

  // ========== POST ENDPOINTS ==========

  /// Lấy tất cả posts
  Future<List<PostModel>> getPosts() async {
    try {
      final response = await _dio.get(ApiConfig.postsEndpoint);
      
      if (response.statusCode == 200) {
        List<dynamic> data = response.data;
        return data.map((json) => PostModel.fromJson(json)).toList();
      } else {
        throw Exception('Failed to load posts');
      }
    } on DioException catch (e) {
      throw _handleError(e);
    }
  }

  /// Lấy post theo ID
  Future<PostModel> getPostById(int id) async {
    try {
      final response = await _dio.get('${ApiConfig.postsEndpoint}/$id');
      
      if (response.statusCode == 200) {
        return PostModel.fromJson(response.data);
      } else {
        throw Exception('Post not found');
      }
    } on DioException catch (e) {
      throw _handleError(e);
    }
  }

  /// Tạo post mới (với hoặc không có ảnh)
  Future<PostModel> createPost({
    required String title,
    required String content,
    File? imageFile,
  }) async {
    try {
      FormData formData = FormData.fromMap({
        'title': title,
        'content': content,
      });

      // Thêm ảnh nếu có
      if (imageFile != null) {
        String fileName = imageFile.path.split('/').last;
        String extension = fileName.split('.').last.toLowerCase();
        
        formData.files.add(MapEntry(
          'image',
          await MultipartFile.fromFile(
            imageFile.path,
            filename: fileName,
            contentType: MediaType('image', extension),
          ),
        ));
      }

      final response = await _dio.post(
        ApiConfig.postsEndpoint,
        data: formData,
        options: Options(
          contentType: 'multipart/form-data',
        ),
      );

      if (response.statusCode == 201) {
        return PostModel.fromJson(response.data);
      } else {
        throw Exception('Failed to create post');
      }
    } on DioException catch (e) {
      throw _handleError(e);
    }
  }

  /// Cập nhật post (với hoặc không có ảnh mới)
  Future<PostModel> updatePost({
    required int id,
    required String title,
    required String content,
    File? imageFile,
  }) async {
    try {
      FormData formData = FormData.fromMap({
        'title': title,
        'content': content,
      });

      // Thêm ảnh mới nếu có
      if (imageFile != null) {
        String fileName = imageFile.path.split('/').last;
        String extension = fileName.split('.').last.toLowerCase();
        
        formData.files.add(MapEntry(
          'image',
          await MultipartFile.fromFile(
            imageFile.path,
            filename: fileName,
            contentType: MediaType('image', extension),
          ),
        ));
      }

      final response = await _dio.put(
        '${ApiConfig.postsEndpoint}/$id',
        data: formData,
        options: Options(
          contentType: 'multipart/form-data',
        ),
      );

      if (response.statusCode == 200) {
        return PostModel.fromJson(response.data);
      } else {
        throw Exception('Failed to update post');
      }
    } on DioException catch (e) {
      throw _handleError(e);
    }
  }

  /// Xóa post
  Future<void> deletePost(int id) async {
    try {
      final response = await _dio.delete('${ApiConfig.postsEndpoint}/$id');
      
      if (response.statusCode != 200) {
        throw Exception('Failed to delete post');
      }
    } on DioException catch (e) {
      throw _handleError(e);
    }
  }

  // ========== COMMENT ENDPOINTS ==========

  /// Thêm comment hoặc reply cho post
  Future<CommentModel> addComment({
    required int postId,
    required String content,
    int? parentCommentId, // Null = comment, có giá trị = reply
  }) async {
    try {
      final response = await _dio.post(
        '${ApiConfig.postsEndpoint}/$postId/comments',
        data: {
          'content': content,
          if (parentCommentId != null) 'parentCommentId': parentCommentId,
        },
      );

      if (response.statusCode == 201) {
        return CommentModel.fromJson(response.data);
      } else {
        throw Exception('Failed to add comment');
      }
    } on DioException catch (e) {
      throw _handleError(e);
    }
  }

  /// Xóa comment
  Future<void> deleteComment(int commentId) async {
    try {
      final response = await _dio.delete(
        '${ApiConfig.commentsEndpoint}/$commentId',
      );
      
      if (response.statusCode != 200) {
        throw Exception('Failed to delete comment');
      }
    } on DioException catch (e) {
      throw _handleError(e);
    }
  }

  // ========== LIKE POST ENDPOINTS ==========

  /// Thêm like cho post
  Future<LikeResponseModel> addLikeToPost(int postId) async {
    try {
      final response = await _dio.post(
        '${ApiConfig.postsEndpoint}/$postId/like',
      );

      if (response.statusCode == 201) {
        return LikeResponseModel.fromJson(response.data);
      } else {
        throw Exception('Failed to add like');
      }
    } on DioException catch (e) {
      throw _handleError(e);
    }
  }

  /// Bỏ like cho post
  Future<LikeResponseModel> removeLikeFromPost(int postId) async {
    try {
      final response = await _dio.delete(
        '${ApiConfig.postsEndpoint}/$postId/like',
      );

      if (response.statusCode == 200) {
        return LikeResponseModel.fromJson(response.data);
      } else {
        throw Exception('Failed to remove like');
      }
    } on DioException catch (e) {
      throw _handleError(e);
    }
  }

  // ========== LIKE COMMENT ENDPOINTS ==========

  /// Thêm like cho comment
  Future<LikeResponseModel> addLikeToComment(int commentId) async {
    try {
      final response = await _dio.post(
        '${ApiConfig.commentsEndpoint}/$commentId/like',
      );

      if (response.statusCode == 201) {
        return LikeResponseModel.fromJson(response.data);
      } else {
        throw Exception('Failed to add like to comment');
      }
    } on DioException catch (e) {
      throw _handleError(e);
    }
  }

  /// Bỏ like cho comment
  Future<LikeResponseModel> removeLikeFromComment(int commentId) async {
    try {
      final response = await _dio.delete(
        '${ApiConfig.commentsEndpoint}/$commentId/like',
      );

      if (response.statusCode == 200) {
        return LikeResponseModel.fromJson(response.data);
      } else {
        throw Exception('Failed to remove like from comment');
      }
    } on DioException catch (e) {
      throw _handleError(e);
    }
  }

  // ========== ERROR HANDLER ==========

  String _handleError(DioException error) {
    switch (error.type) {
      case DioExceptionType.connectionTimeout:
        return 'Connection timeout. Vui lòng kiểm tra kết nối mạng.';
      case DioExceptionType.sendTimeout:
        return 'Send timeout. Vui lòng thử lại.';
      case DioExceptionType.receiveTimeout:
        return 'Receive timeout. Server không phản hồi.';
      case DioExceptionType.badResponse:
        final statusCode = error.response?.statusCode;
        if (statusCode == 400) {
          return 'Dữ liệu không hợp lệ: ${error.response?.data}';
        } else if (statusCode == 404) {
          return 'Không tìm thấy tài nguyên.';
        } else if (statusCode == 500) {
          return 'Lỗi server. Vui lòng thử lại sau.';
        }
        return 'Lỗi server: ${error.response?.statusCode}';
      case DioExceptionType.cancel:
        return 'Request đã bị hủy.';
      case DioExceptionType.unknown:
        if (error.error is SocketException) {
          return 'Không có kết nối Internet.';
        }
        return 'Lỗi không xác định: ${error.message}';
      default:
        return 'Có lỗi xảy ra: ${error.message}';
    }
  }
}
```

---

## 5. Sử dụng API Service trong Widget

### 5.1. Ví dụ: Hiển thị danh sách Posts

Tạo file **`lib/screens/posts_screen.dart`**:

```dart
import 'package:flutter/material.dart';
import '../services/api_service.dart';
import '../models/post_model.dart';

class PostsScreen extends StatefulWidget {
  const PostsScreen({Key? key}) : super(key: key);

  @override
  State<PostsScreen> createState() => _PostsScreenState();
}

class _PostsScreenState extends State<PostsScreen> {
  final ApiService _apiService = ApiService();
  List<PostModel> _posts = [];
  bool _isLoading = false;
  String? _error;

  @override
  void initState() {
    super.initState();
    _loadPosts();
  }

  Future<void> _loadPosts() async {
    setState(() {
      _isLoading = true;
      _error = null;
    });

    try {
      final posts = await _apiService.getPosts();
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
    return Scaffold(
      appBar: AppBar(
        title: const Text('Posts'),
        actions: [
          IconButton(
            icon: const Icon(Icons.refresh),
            onPressed: _loadPosts,
          ),
        ],
      ),
      body: _buildBody(),
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          // Navigate to create post screen
        },
        child: const Icon(Icons.add),
      ),
    );
  }

  Widget _buildBody() {
    if (_isLoading) {
      return const Center(child: CircularProgressIndicator());
    }

    if (_error != null) {
      return Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text('Error: $_error'),
            const SizedBox(height: 16),
            ElevatedButton(
              onPressed: _loadPosts,
              child: const Text('Retry'),
            ),
          ],
        ),
      );
    }

    if (_posts.isEmpty) {
      return const Center(child: Text('No posts available'));
    }

    return RefreshIndicator(
      onRefresh: _loadPosts,
      child: ListView.builder(
        itemCount: _posts.length,
        itemBuilder: (context, index) {
          final post = _posts[index];
          return _buildPostCard(post);
        },
      ),
    );
  }

  Widget _buildPostCard(PostModel post) {
    return Card(
      margin: const EdgeInsets.all(8),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          if (post.imageUrl != null)
            Image.network(
              '${ApiConfig.baseUrl}${post.imageUrl}',
              width: double.infinity,
              height: 200,
              fit: BoxFit.cover,
              errorBuilder: (context, error, stackTrace) {
                return Container(
                  height: 200,
                  color: Colors.grey[300],
                  child: const Icon(Icons.error),
                );
              },
            ),
          Padding(
            padding: const EdgeInsets.all(16),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(
                  post.title,
                  style: const TextStyle(
                    fontSize: 20,
                    fontWeight: FontWeight.bold,
                  ),
                ),
                const SizedBox(height: 8),
                Text(post.content),
                const SizedBox(height: 16),
                Row(
                  children: [
                    IconButton(
                      icon: const Icon(Icons.thumb_up_outlined),
                      onPressed: () => _toggleLike(post.id),
                    ),
                    Text('${post.likeCount} likes'),
                    const SizedBox(width: 16),
                    IconButton(
                      icon: const Icon(Icons.comment_outlined),
                      onPressed: () {
                        // Navigate to comments
                      },
                    ),
                    Text('${post.comments.length} comments'),
                  ],
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }

  Future<void> _toggleLike(int postId) async {
    try {
      // Simplified - just add like
      await _apiService.addLikeToPost(postId);
      _loadPosts(); // Reload to get updated like count
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Error: ${e.toString()}')),
      );
    }
  }
}
```

### 5.2. Ví dụ: Tạo Post mới với ảnh

Tạo file **`lib/screens/create_post_screen.dart`**:

```dart
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:image_picker/image_picker.dart';
import '../services/api_service.dart';

class CreatePostScreen extends StatefulWidget {
  const CreatePostScreen({Key? key}) : super(key: key);

  @override
  State<CreatePostScreen> createState() => _CreatePostScreenState();
}

class _CreatePostScreenState extends State<CreatePostScreen> {
  final _formKey = GlobalKey<FormState>();
  final _titleController = TextEditingController();
  final _contentController = TextEditingController();
  final _apiService = ApiService();
  final _imagePicker = ImagePicker();
  
  File? _selectedImage;
  bool _isLoading = false;

  Future<void> _pickImage() async {
    try {
      final XFile? image = await _imagePicker.pickImage(
        source: ImageSource.gallery,
        maxWidth: 1920,
        maxHeight: 1080,
        imageQuality: 85,
      );

      if (image != null) {
        setState(() {
          _selectedImage = File(image.path);
        });
      }
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Error picking image: $e')),
      );
    }
  }

  Future<void> _createPost() async {
    if (!_formKey.currentState!.validate()) {
      return;
    }

    setState(() => _isLoading = true);

    try {
      await _apiService.createPost(
        title: _titleController.text,
        content: _contentController.text,
        imageFile: _selectedImage,
      );

      if (mounted) {
        Navigator.pop(context, true); // Return true to indicate success
      }
    } catch (e) {
      setState(() => _isLoading = false);
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Error: $e')),
      );
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Create Post'),
      ),
      body: _isLoading
          ? const Center(child: CircularProgressIndicator())
          : SingleChildScrollView(
              padding: const EdgeInsets.all(16),
              child: Form(
                key: _formKey,
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.stretch,
                  children: [
                    TextFormField(
                      controller: _titleController,
                      decoration: const InputDecoration(
                        labelText: 'Title',
                        border: OutlineInputBorder(),
                      ),
                      validator: (value) {
                        if (value == null || value.isEmpty) {
                          return 'Please enter a title';
                        }
                        return null;
                      },
                    ),
                    const SizedBox(height: 16),
                    TextFormField(
                      controller: _contentController,
                      decoration: const InputDecoration(
                        labelText: 'Content',
                        border: OutlineInputBorder(),
                      ),
                      maxLines: 5,
                      validator: (value) {
                        if (value == null || value.isEmpty) {
                          return 'Please enter content';
                        }
                        return null;
                      },
                    ),
                    const SizedBox(height: 16),
                    if (_selectedImage != null) ...[
                      Stack(
                        children: [
                          Image.file(
                            _selectedImage!,
                            height: 200,
                            width: double.infinity,
                            fit: BoxFit.cover,
                          ),
                          Positioned(
                            top: 8,
                            right: 8,
                            child: IconButton(
                              icon: const Icon(Icons.close, color: Colors.white),
                              onPressed: () {
                                setState(() => _selectedImage = null);
                              },
                              style: IconButton.styleFrom(
                                backgroundColor: Colors.black54,
                              ),
                            ),
                          ),
                        ],
                      ),
                      const SizedBox(height: 16),
                    ],
                    OutlinedButton.icon(
                      onPressed: _pickImage,
                      icon: const Icon(Icons.image),
                      label: Text(_selectedImage == null 
                          ? 'Select Image' 
                          : 'Change Image'),
                    ),
                    const SizedBox(height: 24),
                    ElevatedButton(
                      onPressed: _createPost,
                      style: ElevatedButton.styleFrom(
                        padding: const EdgeInsets.all(16),
                      ),
                      child: const Text('Create Post'),
                    ),
                  ],
                ),
              ),
            ),
    );
  }

  @override
  void dispose() {
    _titleController.dispose();
    _contentController.dispose();
    super.dispose();
  }
}
```

### 5.3. Ví dụ: Thêm Comment và Reply

```dart
Future<void> _addComment(int postId, String content) async {
  try {
    await _apiService.addComment(
      postId: postId,
      content: content,
    );
    // Reload post to show new comment
  } catch (e) {
    print('Error adding comment: $e');
  }
}

Future<void> _addReply(int postId, int parentCommentId, String content) async {
  try {
    await _apiService.addComment(
      postId: postId,
      content: content,
      parentCommentId: parentCommentId, // Thêm parent để tạo reply
    );
    // Reload post to show new reply
  } catch (e) {
    print('Error adding reply: $e');
  }
}
```

---

## 6. Tổng hợp các HTTP Status Codes

| Code | Ý nghĩa | Xử lý trong Flutter |
|------|---------|---------------------|
| 200 OK | Thành công | Hiển thị dữ liệu |
| 201 Created | Tạo mới thành công | Navigate về màn hình trước |
| 400 Bad Request | Dữ liệu không hợp lệ | Hiển thị lỗi validation |
| 404 Not Found | Không tìm thấy | Hiển thị "Not found" |
| 500 Internal Server Error | Lỗi server | Hiển thị "Server error" |

---

## 7. Các lỗi thường gặp và Cách khắc phục

### 7.1. Lỗi Connection Timeout
**Thông báo:** *"DioException [connection timeout]: The connection has timed out"*

**Nguyên nhân:**
- API server không chạy
- Base URL sai
- Firewall chặn kết nối

**Cách sửa:**
- Kiểm tra API server đang chạy (chạy `dotnet run` trong project API)
- Với Android Emulator: Dùng `10.0.2.2` thay vì `localhost`
- Với iOS Simulator: Có thể dùng `localhost`
- Với thiết bị thật: Dùng IP của máy tính (VD: `192.168.1.100`)

### 7.2. Lỗi 400 Bad Request khi Upload ảnh
**Thông báo:** *"Only .jpg, .jpeg, .png files are allowed"*

**Nguyên nhân:**
- File không đúng định dạng
- File quá 5MB

**Cách sửa:**
```dart
// Kiểm tra kích thước file trước khi upload
final fileSize = await _selectedImage!.length();
if (fileSize > 5 * 1024 * 1024) {
  throw Exception('File size must be less than 5MB');
}

// Kiểm tra extension
final extension = _selectedImage!.path.split('.').last.toLowerCase();
if (!['jpg', 'jpeg', 'png'].contains(extension)) {
  throw Exception('Only .jpg, .jpeg, .png files are allowed');
}
```

### 7.3. Lỗi hiển thị ảnh từ server
**Thông báo:** Ảnh không hiển thị hoặc lỗi 404

**Nguyên nhân:**
- URL ảnh không đầy đủ
- Thư mục `wwwroot/uploads` chưa được tạo

**Cách sửa:**
```dart
// Ghép base URL với image URL
Image.network(
  '${ApiConfig.baseUrl}${post.imageUrl}',
  errorBuilder: (context, error, stackTrace) {
    return const Icon(Icons.broken_image);
  },
)
```

### 7.4. Lỗi CORS (Web)
**Thông báo:** *"Access to XMLHttpRequest has been blocked by CORS policy"*

**Nguyên nhân:**
- API chưa cấu hình CORS cho Flutter Web

**Cách sửa (trong Program.cs của API):**
```csharp
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(p =>
        p.AllowAnyOrigin()
         .AllowAnyMethod()
         .AllowAnyHeader());
});

// Sau đó thêm app.UseCors(); trong pipeline
```

### 7.5. Lỗi Certificate SSL (Development)
**Thông báo:** *"HandshakeException: Handshake error in client"*

**Cách sửa:**
```dart
// Trong development, có thể bypass SSL certificate (KHÔNG dùng cho production)
class MyHttpOverrides extends HttpOverrides {
  @override
  HttpClient createHttpClient(SecurityContext? context) {
    return super.createHttpClient(context)
      ..badCertificateCallback = (X509Certificate cert, String host, int port) => true;
  }
}

// Trong main.dart
void main() {
  HttpOverrides.global = MyHttpOverrides();
  runApp(const MyApp());
}
```

---

## 8. Checklist trước khi chạy ứng dụng

- [ ] API server đang chạy (`dotnet run`)
- [ ] Đã cấu hình đúng Base URL trong `api_config.dart`
- [ ] Đã cài đặt dependencies (`flutter pub get`)
- [ ] Đã thêm permissions trong `AndroidManifest.xml`:
  ```xml
  <uses-permission android:name="android.permission.INTERNET" />
  <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
  ```
- [ ] Đã thêm permissions trong `Info.plist` (iOS):
  ```xml
  <key>NSPhotoLibraryUsageDescription</key>
  <string>We need access to your photos to upload images</string>
  ```

---

## 9. Best Practices

### 9.1. Quản lý State
Sử dụng state management như **Provider**, **Riverpod**, hoặc **Bloc** để quản lý dữ liệu từ API thay vì `setState` đơn giản.

### 9.2. Caching
Sử dụng **Hive** hoặc **SharedPreferences** để cache dữ liệu offline.

### 9.3. Error Handling
Luôn bọc API calls trong `try-catch` và hiển thị thông báo lỗi thân thiện.

### 9.4. Loading States
Hiển thị loading indicator khi đang gọi API để cải thiện UX.

### 9.5. Pagination
Với danh sách lớn, implement pagination để giảm tải server và cải thiện performance.

---

## 10. Tài liệu tham khảo

- **Dio Documentation**: https://pub.dev/packages/dio
- **Image Picker**: https://pub.dev/packages/image_picker
- **LabAPI README**: Xem file `README.md` trong project API

---

> **Lưu ý:** Tài liệu này dựa trên **LabAPI v1.0** với ASP.NET Core 8. Nếu API được nâng cấp lên phiên bản mới, hãy cập nhật lại `apiVersion` trong `ApiConfig`.
