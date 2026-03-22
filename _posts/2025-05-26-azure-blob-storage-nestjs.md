---
layout: article
title: Azure Blob Storage – Lưu trữ file với NestJS
tags: [azure, blob, nestjs, cloud, upload]
---
Azure Blob Storage là dịch vụ lưu trữ object của Microsoft Azure, tương tự AWS S3. Bài này hướng dẫn tích hợp Azure Blob Storage vào ứng dụng NestJS để upload và quản lý file.

## 1. Chuẩn bị Azure

### Tạo Storage Account

1. Vào [Azure Portal](https://portal.azure.com){:target="_blank"} → **Storage accounts → Create**
2. Chọn **Resource group**, đặt tên **Storage account name** (chữ thường, không dấu)
3. Chọn **Region** gần nhất (Southeast Asia nếu dùng tại Việt Nam)
4. **Redundancy**: LRS (Locally Redundant) là đủ cho dev, GRS cho production

### Lấy Connection String

Vào Storage Account → **Access keys → Show keys → Copy Connection string**

```
DefaultEndpointsProtocol=https;AccountName=mystorageaccount;AccountKey=xxx;EndpointSuffix=core.windows.net
```

### Tạo Container

1. Vào **Containers → + Container**
2. Đặt tên container (ví dụ: `uploads`)
3. **Public access level**:
   - `Private`: Chỉ truy cập qua SAS token hoặc connection string
   - `Blob`: Public đọc file, không list được
   - `Container`: Public hoàn toàn

## 2. Cài đặt SDK

```bash
npm install --save @azure/storage-blob
```

## 3. Cấu hình

Thêm vào `.env`:
```
AZURE_STORAGE_CONNECTION_STRING=DefaultEndpointsProtocol=https;AccountName=...
AZURE_STORAGE_CONTAINER=uploads
```

Tạo `src/config/azure-storage.config.ts`:

```typescript
import { BlobServiceClient } from '@azure/storage-blob';

export const blobServiceClient = BlobServiceClient.fromConnectionString(
  process.env.AZURE_STORAGE_CONNECTION_STRING!,
);

export const CONTAINER_NAME = process.env.AZURE_STORAGE_CONTAINER!;
```

## 4. Tạo Azure Blob Service

`src/azure-blob/azure-blob.service.ts`:

```typescript
import { Injectable, InternalServerErrorException } from '@nestjs/common';
import { BlobServiceClient, BlockBlobClient } from '@azure/storage-blob';
import { randomUUID } from 'crypto';
import { extname } from 'path';

@Injectable()
export class AzureBlobService {
  private readonly containerClient;

  constructor() {
    const blobServiceClient = BlobServiceClient.fromConnectionString(
      process.env.AZURE_STORAGE_CONNECTION_STRING!,
    );
    this.containerClient = blobServiceClient.getContainerClient(
      process.env.AZURE_STORAGE_CONTAINER!,
    );
  }

  private getBlockBlobClient(blobName: string): BlockBlobClient {
    return this.containerClient.getBlockBlobClient(blobName);
  }

  async upload(file: Express.Multer.File, folder = 'uploads'): Promise<string> {
    const blobName = `${folder}/${randomUUID()}${extname(file.originalname)}`;
    const blockBlobClient = this.getBlockBlobClient(blobName);

    await blockBlobClient.uploadData(file.buffer, {
      blobHTTPHeaders: { blobContentType: file.mimetype },
    });

    return blockBlobClient.url;
  }

  async delete(blobName: string): Promise<void> {
    const blockBlobClient = this.getBlockBlobClient(blobName);
    await blockBlobClient.deleteIfExists();
  }

  async generateSasUrl(blobName: string, expiresInMinutes = 60): Promise<string> {
    const blockBlobClient = this.getBlockBlobClient(blobName);

    const expiresOn = new Date();
    expiresOn.setMinutes(expiresOn.getMinutes() + expiresInMinutes);

    const sasUrl = await blockBlobClient.generateSasUrl({
      permissions: { read: true } as any,
      expiresOn,
    });

    return sasUrl;
  }

  async listBlobs(prefix = 'uploads/'): Promise<string[]> {
    const blobs: string[] = [];
    for await (const blob of this.containerClient.listBlobsFlat({ prefix })) {
      blobs.push(blob.name);
    }
    return blobs;
  }
}
```

## 5. Tạo Module

`src/azure-blob/azure-blob.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { AzureBlobService } from './azure-blob.service';
import { AzureBlobController } from './azure-blob.controller';

@Module({
  providers: [AzureBlobService],
  controllers: [AzureBlobController],
  exports: [AzureBlobService],
})
export class AzureBlobModule {}
```

## 6. Tạo Controller

`src/azure-blob/azure-blob.controller.ts`:

```typescript
import {
  Controller,
  Post,
  Delete,
  Get,
  Param,
  UseInterceptors,
  UploadedFile,
  Query,
} from '@nestjs/common';
import { FileInterceptor } from '@nestjs/platform-express';
import { memoryStorage } from 'multer';
import { AzureBlobService } from './azure-blob.service';

@Controller('blob')
export class AzureBlobController {
  constructor(private readonly blobService: AzureBlobService) {}

  @Post('upload')
  @UseInterceptors(FileInterceptor('file', { storage: memoryStorage() }))
  async upload(@UploadedFile() file: Express.Multer.File) {
    const url = await this.blobService.upload(file);
    return { url };
  }

  @Delete(':blobName')
  async delete(@Param('blobName') blobName: string) {
    await this.blobService.delete(blobName);
    return { success: true };
  }

  @Get('sas/:blobName')
  async getSasUrl(
    @Param('blobName') blobName: string,
    @Query('minutes') minutes: string,
  ) {
    const url = await this.blobService.generateSasUrl(blobName, Number(minutes) || 60);
    return { url };
  }

  @Get('list')
  async list() {
    const blobs = await this.blobService.listBlobs();
    return { blobs };
  }
}
```

## 7. SAS Token — Truy cập có thời hạn

SAS (Shared Access Signature) cho phép truy cập file với thời hạn định sẵn:

```typescript
// Tạo SAS cho phép đọc trong 1 giờ
const sasUrl = await blobService.generateSasUrl('uploads/image.jpg', 60);
// https://mystorageaccount.blob.core.windows.net/uploads/uploads/image.jpg?sv=2024-11-04&se=...
```

## 8. So sánh AWS S3 vs Azure Blob Storage

| Tiêu chí | AWS S3 | Azure Blob Storage |
|---------|--------|--------------------|
| Giá (per GB/tháng) | $0.023 | $0.018 (LRS) |
| Free tier | 5GB (12 tháng) | 5GB (12 tháng) |
| CDN tích hợp | CloudFront | Azure CDN |
| SDK | @aws-sdk/client-s3 | @azure/storage-blob |
| Presigned URL | Có (Presigned URL) | Có (SAS Token) |
| Tích hợp với | Lambda, EC2, ECS | Azure Functions, App Service |

## 9. Kết luận

Azure Blob Storage hoạt động tương tự AWS S3 nhưng với SDK và cách quản lý khác:

- **Connection String** thay vì Access Key + Secret Key
- **Container** thay vì Bucket
- **SAS Token** thay vì Presigned URL
- **BlockBlobClient** là interface chính để upload/download

Nếu hệ thống của bạn đang dùng Azure (Azure AD, Azure Functions, Azure SQL), dùng Blob Storage là lựa chọn tự nhiên để tối ưu chi phí và tích hợp.
