---
layout: article
title: AWS CloudFront – CDN cho Static Assets & API Acceleration
tags: [aws, cloudfront, cdn, s3, nodejs, cloud]
---
CloudFront là CDN (Content Delivery Network) của AWS — phân phối nội dung từ các edge location gần người dùng nhất, giảm latency và tăng tốc website. Bài này hướng dẫn setup CloudFront cho S3 static hosting và API acceleration.

## 1. CloudFront hoạt động như thế nào?

```
User (Hà Nội)
    ↓ request image.jpg
CloudFront Edge (Singapore) ← Gần nhất
    ↓ Cache hit? → Trả về ngay (< 20ms)
    ↓ Cache miss? → Fetch từ S3 (Sydney) → Cache → Trả về
```

**Origin**: Nguồn thật của nội dung (S3, ALB, API Gateway, EC2)
**Edge Location**: Server CloudFront gần người dùng (300+ vị trí toàn cầu)
**Cache**: Lưu response tại edge — request sau trả về ngay không cần fetch origin

## 2. Setup CloudFront + S3 (Static Website)

### Bước 1: Tạo S3 bucket

```bash
# Tạo bucket
aws s3 mb s3://shopxyz-static-assets --region ap-southeast-1

# Upload files
aws s3 sync ./dist s3://shopxyz-static-assets --delete
```

### Bước 2: Tạo CloudFront Distribution qua AWS CLI

```bash
aws cloudfront create-distribution --distribution-config '{
  "CallerReference": "shopxyz-2026",
  "Comment": "ShopXYZ Static Assets",
  "DefaultCacheBehavior": {
    "TargetOriginId": "S3-shopxyz-static-assets",
    "ViewerProtocolPolicy": "redirect-to-https",
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "Compress": true
  },
  "Origins": {
    "Quantity": 1,
    "Items": [{
      "Id": "S3-shopxyz-static-assets",
      "DomainName": "shopxyz-static-assets.s3.amazonaws.com",
      "S3OriginConfig": { "OriginAccessIdentity": "" }
    }]
  },
  "Enabled": true,
  "HttpVersion": "http2and3",
  "PriceClass": "PriceClass_200"
}'
```

### Bước 3: Custom Domain + SSL

```bash
# Tạo certificate (phải ở us-east-1 cho CloudFront)
aws acm request-certificate \
  --domain-name cdn.shopxyz.com \
  --validation-method DNS \
  --region us-east-1
```

## 3. CloudFront với NodeJS SDK

```bash
npm install @aws-sdk/client-cloudfront
```

```typescript
// src/cdn/cloudfront.service.ts
import { Injectable } from '@nestjs/common';
import {
  CloudFrontClient,
  CreateInvalidationCommand,
  GetDistributionCommand,
} from '@aws-sdk/client-cloudfront';

@Injectable()
export class CloudFrontService {
  private client: CloudFrontClient;
  private distributionId = process.env.CLOUDFRONT_DISTRIBUTION_ID!;

  constructor() {
    this.client = new CloudFrontClient({
      region: 'us-east-1',  // CloudFront luôn dùng us-east-1
      credentials: {
        accessKeyId: process.env.AWS_ACCESS_KEY_ID!,
        secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY!,
      },
    });
  }

  // Invalidate cache khi cập nhật file
  async invalidate(paths: string[]): Promise<string> {
    const command = new CreateInvalidationCommand({
      DistributionId: this.distributionId,
      InvalidationBatch: {
        CallerReference: Date.now().toString(),
        Paths: {
          Quantity: paths.length,
          Items: paths, // ['/images/banner.jpg', '/css/main.css']
        },
      },
    });

    const result = await this.client.send(command);
    return result.Invalidation!.Id!;
  }

  // Invalidate tất cả (khi deploy mới)
  async invalidateAll(): Promise<string> {
    return this.invalidate(['/*']);
  }

  getCdnUrl(s3Key: string): string {
    return `https://${process.env.CLOUDFRONT_DOMAIN}/${s3Key}`;
  }
}
```

## 4. Kết hợp S3 Upload + CloudFront URL

```typescript
// src/upload/upload.service.ts
@Injectable()
export class UploadService {
  constructor(
    private s3Service: S3Service,
    private cloudFrontService: CloudFrontService,
  ) {}

  async uploadProductImage(file: Express.Multer.File, productId: string) {
    const key = `products/${productId}/${Date.now()}-${file.originalname}`;

    // Upload lên S3
    await this.s3Service.upload(key, file.buffer, file.mimetype);

    // Trả về CDN URL (không phải S3 URL)
    const cdnUrl = this.cloudFrontService.getCdnUrl(key);

    return { key, url: cdnUrl };
  }

  async updateProductImage(oldKey: string, newFile: Express.Multer.File, productId: string) {
    const result = await this.uploadProductImage(newFile, productId);

    // Invalidate cache URL cũ
    await this.cloudFrontService.invalidate([`/${oldKey}`]);

    // Xóa file cũ trên S3
    await this.s3Service.delete(oldKey);

    return result;
  }
}
```

## 5. Cache Policy & Cache Control Headers

```typescript
// Cấu hình Cache-Control khi upload S3
await s3Client.send(new PutObjectCommand({
  Bucket: 'shopxyz-static-assets',
  Key: key,
  Body: buffer,
  ContentType: mimeType,
  CacheControl: getCacheControl(mimeType),
}));

function getCacheControl(mimeType: string): string {
  if (mimeType.startsWith('image/')) {
    return 'public, max-age=31536000, immutable'; // 1 năm — images ít thay đổi
  }
  if (mimeType === 'text/html') {
    return 'public, max-age=0, must-revalidate'; // Luôn revalidate HTML
  }
  if (mimeType.includes('javascript') || mimeType.includes('css')) {
    return 'public, max-age=31536000, immutable'; // JS/CSS với hash trong filename
  }
  return 'public, max-age=3600'; // Default 1 giờ
}
```

## 6. CloudFront Functions — Edge Logic

Chạy JavaScript nhẹ tại edge (không dùng Lambda@Edge):

```javascript
// cloudfront-function.js — URL rewrite
function handler(event) {
  var request = event.request;
  var uri = request.uri;

  // Redirect www → non-www
  if (request.headers.host.value.startsWith('www.')) {
    return {
      statusCode: 301,
      headers: {
        location: { value: 'https://shopxyz.com' + uri }
      }
    };
  }

  // Thêm .html nếu không có extension
  if (!uri.includes('.') && !uri.endsWith('/')) {
    request.uri = uri + '.html';
  }

  return request;
}
```

## 7. GitHub Actions — Deploy + Invalidate

```yaml
- name: Deploy to S3 & Invalidate CloudFront
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  run: |
    # Upload
    aws s3 sync ./dist s3://${{ secrets.S3_BUCKET }} \
      --delete \
      --cache-control "public, max-age=31536000, immutable" \
      --exclude "*.html" \
      --region ap-southeast-1

    # HTML không cache
    aws s3 sync ./dist s3://${{ secrets.S3_BUCKET }} \
      --include "*.html" \
      --cache-control "public, max-age=0, must-revalidate" \
      --region ap-southeast-1

    # Invalidate HTML files
    aws cloudfront create-invalidation \
      --distribution-id ${{ secrets.CF_DISTRIBUTION_ID }} \
      --paths "/*.html" "/index.html"
```

## 8. Kết luận

- **CloudFront + S3**: CDN tiêu chuẩn cho static assets — giảm latency từ ~200ms → ~20ms
- **Invalidation**: Khi cập nhật file, invalidate cache tại edge (`/*` hoặc path cụ thể)
- **Cache-Control**: Images/JS/CSS có hash → `immutable`, HTML → `no-cache`
- **CDN URL**: Luôn trả CDN URL trong API response, không bao giờ trả S3 URL trực tiếp
- **PriceClass_200**: Dùng Asia-Pacific edge locations — đủ cho user Việt Nam, giá hợp lý

CloudFront là lớp bắt buộc khi dùng S3 cho production — vừa nhanh hơn vừa bảo mật hơn (ẩn S3 URL).
