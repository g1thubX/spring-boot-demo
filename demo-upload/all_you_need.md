# 文件上传下载 - 完全学习指南

## 📚 简介

文件上传和下载是 Web 应用的基本功能。Spring Boot 提供了完整的文件处理支持，支持本地磁盘存储、云存储服务（如七牛云、阿里 OSS）等。

### 为什么学习这个？
- 🎯 **掌握文件上传的完整流程**
- 🎯 **理解文件安全性问题和防护措施**
- 🎯 **集成云存储服务**
- 🎯 **处理大文件上传和断点续传**

---

## 🎯 核心概念

### 1. 文件上传的完整流程

```
客户端选择文件
   ↓
表单提交（multipart/form-data）
   ↓
Spring 接收 MultipartFile
   ↓
验证文件（类型、大小）
   ↓
保存文件（本地/云）
   ↓
返回文件 URL 或 ID
```

### 2. 文件上传的三种方式

#### a) 本地磁盘存储（简单）

```java
@RestController
@RequestMapping("/files")
public class FileUploadController {
    
    @Value("${file.upload.path:uploads}")
    private String uploadPath;
    
    @PostMapping("/upload")
    public ApiResponse<FileInfo> uploadFile(
        @RequestParam("file") MultipartFile file) {
        
        // 验证文件
        if (file.isEmpty()) {
            return ApiResponse.error(400, "文件不能为空");
        }
        
        if (file.getSize() > 10 * 1024 * 1024) {  // 10MB
            return ApiResponse.error(400, "文件大小不能超过 10MB");
        }
        
        String originalFilename = file.getOriginalFilename();
        String suffix = originalFilename.substring(originalFilename.lastIndexOf("."));
        String filename = UUID.randomUUID().toString() + suffix;
        
        try {
            // 创建目录
            File uploadDir = new File(uploadPath);
            if (!uploadDir.exists()) {
                uploadDir.mkdirs();
            }
            
            // 保存文件
            File destFile = new File(uploadDir, filename);
            file.transferTo(destFile);
            
            FileInfo fileInfo = new FileInfo(
                filename,
                originalFilename,
                file.getSize(),
                "/files/download/" + filename
            );
            
            return ApiResponse.success(fileInfo, "文件上传成功");
        } catch (IOException e) {
            return ApiResponse.error(500, "文件保存失败");
        }
    }
    
    @GetMapping("/download/{filename}")
    public ResponseEntity<Resource> download(@PathVariable String filename) 
            throws IOException {
        File file = new File(uploadPath, filename);
        
        if (!file.exists()) {
            return ResponseEntity.notFound().build();
        }
        
        Resource resource = new FileSystemResource(file);
        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, 
                "attachment; filename=\"" + file.getName() + "\"")
            .body(resource);
    }
}
```

#### b) 云存储（七牛云）

```java
@Service
public class QiniuUploadService {
    
    @Value("${qiniu.access-key}")
    private String accessKey;
    
    @Value("${qiniu.secret-key}")
    private String secretKey;
    
    @Value("${qiniu.bucket}")
    private String bucket;
    
    public String uploadFile(MultipartFile file) {
        Auth auth = Auth.create(accessKey, secretKey);
        UploadManager uploadManager = new UploadManager(new Configuration());
        
        try {
            String key = UUID.randomUUID().toString();
            Response response = uploadManager.put(
                file.getBytes(),
                key,
                auth.uploadToken(bucket)
            );
            
            if (response.isOK()) {
                return "https://cdn.example.com/" + key;  // 返回 URL
            } else {
                throw new RuntimeException("七牛上传失败");
            }
        } catch (Exception e) {
            throw new RuntimeException("文件上传失败", e);
        }
    }
}
```

#### c) 阿里 OSS 存储

```java
@Service
public class AliyunOssService {
    
    @Value("${aliyun.oss.endpoint}")
    private String endpoint;
    
    @Value("${aliyun.oss.access-key-id}")
    private String accessKeyId;
    
    @Value("${aliyun.oss.access-key-secret}")
    private String accessKeySecret;
    
    @Value("${aliyun.oss.bucket}")
    private String bucketName;
    
    public String uploadFile(MultipartFile file) {
        OSS ossClient = new OSSClientBuilder()
            .build(endpoint, accessKeyId, accessKeySecret);
        
        try {
            String objectName = UUID.randomUUID().toString();
            
            ossClient.putObject(bucketName, objectName, file.getInputStream());
            
            return "https://" + bucketName + "." + endpoint + "/" + objectName;
        } catch (IOException e) {
            throw new RuntimeException("文件上传失败", e);
        } finally {
            ossClient.shutdown();
        }
    }
}
```

### 3. 文件验证和安全性

```java
@Service
public class FileValidationService {
    
    private static final List<String> ALLOWED_TYPES = Arrays.asList(
        "image/jpeg", "image/png", "image/gif",
        "application/pdf", "application/msword"
    );
    
    private static final long MAX_FILE_SIZE = 10 * 1024 * 1024;  // 10MB
    
    public void validateFile(MultipartFile file) {
        // 1. 检查文件是否为空
        if (file.isEmpty()) {
            throw new RuntimeException("文件不能为空");
        }
        
        // 2. 检查文件大小
        if (file.getSize() > MAX_FILE_SIZE) {
            throw new RuntimeException("文件大小不能超过 10MB");
        }
        
        // 3. 检查文件类型（MIME 类型）
        String contentType = file.getContentType();
        if (!ALLOWED_TYPES.contains(contentType)) {
            throw new RuntimeException("不支持的文件类型");
        }
        
        // 4. 检查文件名（防止目录遍历）
        String filename = file.getOriginalFilename();
        if (filename.contains("../") || filename.contains("..\\")) {
            throw new RuntimeException("文件名非法");
        }
    }
}
```

---

## 💡 实现细节

### 1. 大文件上传和断点续传

```java
@RestController
@RequestMapping("/files/chunk")
public class ChunkedUploadController {
    
    private static final int CHUNK_SIZE = 1024 * 1024;  // 1MB
    
    @PostMapping("/upload")
    public ApiResponse uploadChunk(
        @RequestParam("file") MultipartFile chunk,
        @RequestParam("chunkNumber") Integer chunkNumber,
        @RequestParam("totalChunks") Integer totalChunks,
        @RequestParam("uploadId") String uploadId) {
        
        try {
            // 保存分块
            File chunkFile = new File("temp/" + uploadId + "/" + chunkNumber);
            chunk.transferTo(chunkFile);
            
            // 如果所有分块都上传完成，合并文件
            if (chunkNumber.equals(totalChunks)) {
                mergeChunks(uploadId, totalChunks);
            }
            
            return ApiResponse.success(null, "分块上传成功");
        } catch (IOException e) {
            return ApiResponse.error(500, "分块上传失败");
        }
    }
    
    private void mergeChunks(String uploadId, Integer totalChunks) {
        // 合并所有分块
        File targetFile = new File("uploads/" + uploadId + ".zip");
        
        try (FileOutputStream fos = new FileOutputStream(targetFile)) {
            for (int i = 1; i <= totalChunks; i++) {
                File chunkFile = new File("temp/" + uploadId + "/" + i);
                try (FileInputStream fis = new FileInputStream(chunkFile)) {
                    byte[] buffer = new byte[1024];
                    int length;
                    while ((length = fis.read(buffer)) > 0) {
                        fos.write(buffer, 0, length);
                    }
                }
                chunkFile.delete();  // 删除分块文件
            }
        } catch (IOException e) {
            throw new RuntimeException("文件合并失败", e);
        }
    }
}
```

### 2. 文件压缩

```java
@Service
public class FileCompressionService {
    
    public File compressFile(MultipartFile file) throws IOException {
        String outputPath = "temp/" + UUID.randomUUID().toString() + ".zip";
        
        try (ZipOutputStream zos = new ZipOutputStream(new FileOutputStream(outputPath))) {
            ZipEntry entry = new ZipEntry(file.getOriginalFilename());
            zos.putNextEntry(entry);
            
            zos.write(file.getBytes());
            zos.closeEntry();
        }
        
        return new File(outputPath);
    }
}
```

---

## 🏢 行业应用

### 1. 用户头像上传

```java
@Service
public class AvatarUploadService {
    
    public String uploadAvatar(Long userId, MultipartFile file) {
        // 生成文件名: user_{userId}.jpg
        String filename = "user_" + userId + ".jpg";
        
        // 缩小图片大小（100x100）
        BufferedImage image = ImageIO.read(file.getInputStream());
        BufferedImage resized = resizeImage(image, 100, 100);
        
        // 保存
        File outputFile = new File("avatars/" + filename);
        ImageIO.write(resized, "jpg", outputFile);
        
        return "/avatars/" + filename;
    }
}
```

### 2. 文档上传和预览

```java
@Service
public class DocumentUploadService {
    
    public Document uploadDocument(Long userId, MultipartFile file) {
        // 验证文档格式
        String filename = file.getOriginalFilename();
        String ext = filename.substring(filename.lastIndexOf("."));
        
        if (!isSupportedDocumentType(ext)) {
            throw new RuntimeException("不支持的文档格式");
        }
        
        // 保存文档
        String filePath = "documents/" + UUID.randomUUID().toString() + ext;
        file.transferTo(new File(filePath));
        
        // 提取文本用于搜索
        String text = extractText(filePath);
        
        // 保存到数据库
        Document doc = new Document();
        doc.setUserId(userId);
        doc.setFilePath(filePath);
        doc.setFileName(filename);
        doc.setContent(text);
        
        return documentRepository.save(doc);
    }
}
```

---

## 🚀 快速开始

### 1. 依赖

```xml
<!-- 本地存储 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- 七牛云 -->
<dependency>
    <groupId>com.qiniu</groupId>
    <artifactId>qiniu-java-sdk</artifactId>
    <version>7.7.0</version>
</dependency>
```

### 2. 配置

```yaml
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 10MB

file:
  upload:
    path: ./uploads
```

### 3. 上传文件

```java
@RestController
public class UploadController {
    
    @PostMapping("/upload")
    public ApiResponse upload(@RequestParam MultipartFile file) {
        // 保存文件
        String filename = UUID.randomUUID().toString();
        file.transferTo(new File("uploads/" + filename));
        
        return ApiResponse.success(filename);
    }
}
```

---

## 🧪 测试

使用 curl 上传文件：

```bash
curl -F "file=@test.txt" http://localhost:8080/upload
```

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| MultipartFile | 文件上传接口 |
| 文件验证 | 检查类型、大小、名称 |
| 本地存储 | File 存储 |
| 云存储 | 七牛、阿里 OSS 等 |
| 断点续传 | 分块上传和合并 |
| 文件压缩 | ZIP 压缩 |

---

**恭喜！** 🎉 你已经掌握了文件上传和下载的完整知识！
