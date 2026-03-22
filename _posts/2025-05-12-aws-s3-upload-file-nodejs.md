---
layout: article
title: AWS S3 – Upload & Quản lý file từ NodeJS
tags: [aws, s3, nodejs, cloud, upload]
---
Amazon S3 (Simple Storage Service) là dịch vụ lưu trữ object phổ biến nhất trên cloud. Bài này hướng dẫn tích hợp S3 vào ứng dụng Node.js để upload, download và quản lý file.

## 1. Chuẩn bị AWS

### Tạo S3 Bucket

1. Vào [AWS Console → S3](https://s3.console.aws.amazon.com){:target="_blank"}
2. Nhấn **Create bucket**
3. Đặt tên bucket (phải unique toàn cầu), chọn region
4. **Block all public access**: bật nếu bucket private, tắt nếu cần public file

### Tạo IAM User cho S3

1. Vào **IAM → Users → Add users**
2. Chọn **Programmatic access**
3. Gán policy **AmazonS3FullAccess** (hoặc policy tùy chỉnh chặt hơn)
4. Lưu lại **Access Key ID** và **Secret Access Key**

### Policy tùy chỉnh (khuyến nghị)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::your-bucket-name",
        "arn:aws:s3:::your-bucket-name/*"
      ]
    }
  ]
}
```

## 2. Cài đặt SDK

```bash
npm install --save @aws-sdk/client-s3 @aws-sdk/s3-request-presigner
```

## 3. Cấu hình

Thêm vào `.env`:
```
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=your-secret-key
AWS_REGION=ap-southeast-1
S3_BUCKET_NAME=your-bucket-name
```

Tạo file `src/config/s3.config.ts`:

```typescript
import { S3Client } from '@aws-sdk/client-s3';

export const s3Client = new S3Client({
  region: process.env.AWS_REGION!,
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID!,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY!,
  },
});

export const S3_BUCKET = process.env.S3_BUCKET_NAME!;
```

## 4. Upload file lên S3

```typescript
import { PutObjectCommand } from '@aws-sdk/client-s3';
import { s3Client, S3_BUCKET } from './config/s3.config';
import { randomUUID } from 'crypto';
import { extname } from 'path';

export async function uploadToS3(
  file: Express.Multer.File,
  folder = 'uploads',
): Promise<string> {
  const key = `${folder}/${randomUUID()}${extname(file.originalname)}`;

  const command = new PutObjectCommand({
    Bucket: S3_BUCKET,
    Key: key,
    Body: file.buffer,
    ContentType: file.mimetype,
    // ACL: 'public-read',  // Bỏ comment nếu muốn file public
  });

  await s3Client.send(command);

  // Trả về URL file
  return `https://${S3_BUCKET}.s3.${process.env.AWS_REGION}.amazonaws.com/${key}`;
}
```

> Dùng `memoryStorage` của Multer để có `file.buffer`:
> ```typescript
> import { memoryStorage } from 'multer';
> FileInterceptor('file', { storage: memoryStorage() })
> ```

## 5. Tạo Presigned URL

Presigned URL cho phép client upload thẳng lên S3 mà không cần qua server — giảm tải server và tăng tốc upload:

```typescript
import { PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { s3Client, S3_BUCKET } from './config/s3.config';

export async function getPresignedUploadUrl(
  filename: string,
  contentType: string,
  expiresIn = 300, // 5 phút
): Promise<{ uploadUrl: string; key: string }> {
  const key = `uploads/${Date.now()}-${filename}`;

  const command = new PutObjectCommand({
    Bucket: S3_BUCKET,
    Key: key,
    ContentType: contentType,
  });

  const uploadUrl = await getSignedUrl(s3Client, command, { expiresIn });

  return { uploadUrl, key };
}
```

Client dùng presigned URL để upload:
```javascript
// Frontend
const { uploadUrl, key } = await fetch('/api/presign?filename=photo.jpg').then(r => r.json());
await fetch(uploadUrl, { method: 'PUT', body: file, headers: { 'Content-Type': file.type } });
```

## 6. Download / Lấy URL có thời hạn

```typescript
import { GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

export async function getPresignedDownloadUrl(key: string, expiresIn = 3600) {
  const command = new GetObjectCommand({ Bucket: S3_BUCKET, Key: key });
  return getSignedUrl(s3Client, command, { expiresIn });
}
```

## 7. Xóa file

```typescript
import { DeleteObjectCommand } from '@aws-sdk/client-s3';

export async function deleteFromS3(key: string): Promise<void> {
  const command = new DeleteObjectCommand({ Bucket: S3_BUCKET, Key: key });
  await s3Client.send(command);
}
```

## 8. Liệt kê file trong folder

```typescript
import { ListObjectsV2Command } from '@aws-sdk/client-s3';

export async function listFiles(prefix = 'uploads/') {
  const command = new ListObjectsV2Command({
    Bucket: S3_BUCKET,
    Prefix: prefix,
  });

  const response = await s3Client.send(command);
  return response.Contents?.map(obj => ({
    key: obj.Key,
    size: obj.Size,
    lastModified: obj.LastModified,
  })) ?? [];
}
```

## 9. Tích hợp vào NestJS Controller

```typescript
@Post('upload')
@UseInterceptors(FileInterceptor('file', { storage: memoryStorage() }))
async uploadFile(@UploadedFile() file: Express.Multer.File) {
  const url = await this.s3Service.uploadToS3(file);
  return { url };
}

@Get('presign')
async getPresignedUrl(@Query('filename') filename: string, @Query('type') type: string) {
  return this.s3Service.getPresignedUploadUrl(filename, type);
}
```

## 10. Kết luận

AWS S3 là giải pháp lưu trữ file đáng tin cậy và tiết kiệm chi phí:

- **Direct upload**: Server nhận file rồi forward lên S3 — đơn giản nhưng tốn băng thông server
- **Presigned URL**: Client upload thẳng lên S3 — nhanh hơn, giảm tải server (khuyến nghị)
- Kết hợp S3 với **CloudFront CDN** để phân phối file toàn cầu với latency thấp

Bài tiếp theo: **Azure Blob Storage với NestJS**.
