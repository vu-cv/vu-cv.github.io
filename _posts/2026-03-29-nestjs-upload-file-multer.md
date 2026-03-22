---
layout: article
title: NestJS – Upload File với Multer
tags: [nestjs, upload, multer, file]
---
NestJS tích hợp sẵn Multer thông qua `@nestjs/platform-express`, giúp xử lý upload file `multipart/form-data` rất đơn giản mà không cần cài thêm package nào.

## 1. Cài đặt type definition

```bash
npm install --save-dev @types/multer
```

## 2. Upload một file

### Controller

```typescript
import {
  Controller,
  Post,
  UseInterceptors,
  UploadedFile,
  BadRequestException,
} from '@nestjs/common';
import { FileInterceptor } from '@nestjs/platform-express';
import { diskStorage } from 'multer';
import { extname } from 'path';

@Controller('upload')
export class UploadController {
  @Post('single')
  @UseInterceptors(
    FileInterceptor('file', {
      storage: diskStorage({
        destination: './uploads',
        filename: (req, file, cb) => {
          const unique = Date.now() + '-' + Math.round(Math.random() * 1e9);
          cb(null, unique + extname(file.originalname));
        },
      }),
      limits: { fileSize: 5 * 1024 * 1024 }, // 5MB
      fileFilter: (req, file, cb) => {
        const allowed = /jpeg|jpg|png|gif|pdf/;
        const isAllowed = allowed.test(extname(file.originalname).toLowerCase());
        if (isAllowed) {
          cb(null, true);
        } else {
          cb(new BadRequestException('Chỉ cho phép ảnh và PDF'), false);
        }
      },
    }),
  )
  uploadSingle(@UploadedFile() file: Express.Multer.File) {
    if (!file) throw new BadRequestException('Không tìm thấy file');
    return {
      filename: file.filename,
      originalname: file.originalname,
      size: file.size,
      mimetype: file.mimetype,
      path: file.path,
    };
  }
}
```

Tạo thư mục uploads trước khi chạy:
```bash
mkdir uploads
```

## 3. Upload nhiều file

```typescript
import { FilesInterceptor } from '@nestjs/platform-express';

@Post('multiple')
@UseInterceptors(
  FilesInterceptor('files', 10, {
    storage: diskStorage({
      destination: './uploads',
      filename: (req, file, cb) => {
        const unique = Date.now() + '-' + Math.round(Math.random() * 1e9);
        cb(null, unique + extname(file.originalname));
      },
    }),
  }),
)
uploadMultiple(@UploadedFiles() files: Express.Multer.File[]) {
  return {
    count: files.length,
    files: files.map(f => ({
      filename: f.filename,
      originalname: f.originalname,
      size: f.size,
    })),
  };
}
```

Thêm import `UploadedFiles`:
```typescript
import { UploadedFiles } from '@nestjs/common';
```

## 4. Tách cấu hình Multer ra riêng

Để tái sử dụng, tạo file `src/upload/multer.config.ts`:

```typescript
import { diskStorage } from 'multer';
import { extname } from 'path';
import { BadRequestException } from '@nestjs/common';

export const multerConfig = {
  storage: diskStorage({
    destination: './uploads',
    filename: (req, file, cb) => {
      const unique = `${Date.now()}-${Math.round(Math.random() * 1e9)}`;
      cb(null, unique + extname(file.originalname));
    },
  }),
  limits: { fileSize: 5 * 1024 * 1024 },
  fileFilter: (req: any, file: Express.Multer.File, cb: any) => {
    const allowedMimes = ['image/jpeg', 'image/png', 'image/gif', 'application/pdf'];
    if (allowedMimes.includes(file.mimetype)) {
      cb(null, true);
    } else {
      cb(new BadRequestException('Loại file không được hỗ trợ'), false);
    }
  },
};
```

Dùng lại trong controller:
```typescript
@UseInterceptors(FileInterceptor('file', multerConfig))
```

## 5. Serve file tĩnh

Để truy cập file đã upload qua URL, cấu hình static assets trong `main.ts`:

```typescript
import { NestFactory } from '@nestjs/core';
import { NestExpressApplication } from '@nestjs/platform-express';
import { join } from 'path';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create<NestExpressApplication>(AppModule);
  app.useStaticAssets(join(__dirname, '..', 'uploads'), { prefix: '/uploads' });
  await app.listen(3000);
}
bootstrap();
```

Sau đó file sẽ truy cập được tại: `http://localhost:3000/uploads/<filename>`

## 6. Validation với Pipe

NestJS cho phép validate file bằng `ParseFilePipe`:

```typescript
import { ParseFilePipe, MaxFileSizeValidator, FileTypeValidator } from '@nestjs/common';

@Post('validated')
@UseInterceptors(FileInterceptor('file'))
uploadValidated(
  @UploadedFile(
    new ParseFilePipe({
      validators: [
        new MaxFileSizeValidator({ maxSize: 2 * 1024 * 1024 }), // 2MB
        new FileTypeValidator({ fileType: /image\/(jpeg|png)/ }),
      ],
    }),
  )
  file: Express.Multer.File,
) {
  return { filename: file.filename, size: file.size };
}
```

## 7. Test với curl

```bash
# Upload một file
curl -X POST http://localhost:3000/upload/single \
  -F "file=@/path/to/image.jpg"

# Upload nhiều file
curl -X POST http://localhost:3000/upload/multiple \
  -F "files=@/path/to/image1.jpg" \
  -F "files=@/path/to/image2.png"
```

## 8. Kết luận

NestJS + Multer cho phép xử lý upload file linh hoạt:

- `FileInterceptor` — một file, `FilesInterceptor` — nhiều file
- `diskStorage` — lưu vào đĩa, `memoryStorage` — lưu vào RAM (buffer)
- `ParseFilePipe` — validate kích thước và loại file
- `useStaticAssets` — serve file upload qua URL

Bài tiếp theo chúng ta sẽ tìm hiểu **NextJS 14** — framework React cho frontend.
