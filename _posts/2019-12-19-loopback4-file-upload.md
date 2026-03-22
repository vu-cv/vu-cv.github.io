---
layout: article
title: Loopback 4 File Upload
tags: [loopback, fileupload, upload]
---
Trong bài viết này, chúng ta sẽ tìm hiểu cách xây dựng chức năng upload file trong Loopback 4 sử dụng `multer` — một middleware phổ biến cho việc xử lý `multipart/form-data`.

## 1. Chuẩn bị

Trước khi bắt đầu, bạn cần có một project Loopback 4 đã được khởi tạo. Nếu chưa có, hãy tham khảo bài viết [Tìm hiểu & Cài đặt Loopback 4](/2019/12/09/tim-hieu-va-cai-dat-loopback-4.html).

Cài đặt thư viện `multer` và type definition của nó:

```bash
npm install --save multer
npm install --save-dev @types/multer
```

## 2. Tạo FileUploadService

Tạo file `src/services/file-upload.service.ts` để cấu hình multer:

```typescript
import {inject} from '@loopback/core';
import {Request} from '@loopback/rest';
import multer from 'multer';
import path from 'path';

const UPLOAD_DIR = path.join(__dirname, '../../uploads');

const storage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, UPLOAD_DIR);
  },
  filename: (req, file, cb) => {
    const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1e9);
    cb(null, uniqueSuffix + path.extname(file.originalname));
  },
});

export const upload = multer({storage});
```

> **Lưu ý:** Hãy tạo thư mục `uploads` ở thư mục gốc của project trước khi chạy.

```bash
mkdir uploads
```

## 3. Tạo FileUploadController

Tạo controller để xử lý request upload file:

```bash
lb4 controller
```

Chọn **Empty Controller** và đặt tên là `FileUpload`.

Sau đó chỉnh sửa file `src/controllers/file-upload.controller.ts`:

```typescript
import {inject} from '@loopback/core';
import {
  post,
  Request,
  requestBody,
  Response,
  RestBindings,
} from '@loopback/rest';
import {upload} from '../services/file-upload.service';

export class FileUploadController {
  constructor(
    @inject(RestBindings.Http.REQUEST) private request: Request,
    @inject(RestBindings.Http.RESPONSE) private response: Response,
  ) {}

  @post('/upload', {
    responses: {
      '200': {
        description: 'Upload file thành công',
        content: {
          'application/json': {
            schema: {
              type: 'object',
              properties: {
                filename: {type: 'string'},
                originalname: {type: 'string'},
                size: {type: 'number'},
              },
            },
          },
        },
      },
    },
  })
  async uploadFile(
    @requestBody({
      description: 'multipart/form-data',
      required: true,
      content: {
        'multipart/form-data': {
          'x-parser': 'stream',
          schema: {
            type: 'object',
            properties: {
              file: {
                type: 'string',
                format: 'binary',
              },
            },
          },
        },
      },
    })
    request: Request,
  ): Promise<object> {
    return new Promise((resolve, reject) => {
      upload.single('file')(this.request, this.response, (err: unknown) => {
        if (err) {
          reject(err);
          return;
        }
        const file = (this.request as any).file;
        if (!file) {
          reject(new Error('Không tìm thấy file trong request'));
          return;
        }
        resolve({
          filename: file.filename,
          originalname: file.originalname,
          size: file.size,
          mimetype: file.mimetype,
        });
      });
    });
  }
}
```

## 4. Đăng ký Controller trong Application

Mở file `src/application.ts` và đảm bảo controller đã được import:

```typescript
import {FileUploadController} from './controllers';
```

Nếu bạn dùng `lb4 controller` để tạo thì Loopback 4 sẽ tự động đăng ký controller vào `src/controllers/index.ts`.

## 5. Giới hạn loại file và kích thước

Để kiểm soát file upload chặt chẽ hơn, bạn có thể thêm `fileFilter` và `limits` vào cấu hình multer:

```typescript
export const upload = multer({
  storage,
  limits: {
    fileSize: 5 * 1024 * 1024, // Tối đa 5MB
  },
  fileFilter: (req, file, cb) => {
    const allowedTypes = ['image/jpeg', 'image/png', 'image/gif', 'application/pdf'];
    if (allowedTypes.includes(file.mimetype)) {
      cb(null, true);
    } else {
      cb(new Error('Loại file không được hỗ trợ'));
    }
  },
});
```

## 6. Upload nhiều file cùng lúc

Thay vì `upload.single('file')`, sử dụng `upload.array('files', 10)` để cho phép upload tối đa 10 file:

```typescript
upload.array('files', 10)(this.request, this.response, (err: unknown) => {
  if (err) {
    reject(err);
    return;
  }
  const files = (this.request as any).files as Express.Multer.File[];
  resolve({
    count: files.length,
    files: files.map(f => ({
      filename: f.filename,
      originalname: f.originalname,
      size: f.size,
    })),
  });
});
```

Cập nhật schema trong `@requestBody` để nhận nhiều file:

```typescript
schema: {
  type: 'object',
  properties: {
    files: {
      type: 'array',
      items: {
        type: 'string',
        format: 'binary',
      },
    },
  },
},
```

## 7. Kiểm tra với Loopback Explorer

Chạy project:

```bash
npm start
```

Truy cập [http://127.0.0.1:3000/explorer](http://127.0.0.1:3000/explorer){:target="_blank"}, tìm endpoint `POST /upload`, chọn **Try it out** và chọn file để test.

Response trả về sẽ có dạng:

```json
{
  "filename": "1734567890123-987654321.png",
  "originalname": "avatar.png",
  "size": 204800,
  "mimetype": "image/png"
}
```

## 8. Kết luận

Chúng ta đã hoàn thành việc xây dựng chức năng upload file trong Loopback 4 với `multer`:

- Upload **một file** bằng `upload.single()`
- Upload **nhiều file** bằng `upload.array()`
- Giới hạn **kích thước** và **loại file** qua `limits` và `fileFilter`
- Expose API qua **Loopback Explorer** để test trực tiếp

Bạn có thể mở rộng thêm bằng cách lưu thông tin file vào database thông qua Repository của Loopback 4 hoặc upload thẳng lên cloud storage như AWS S3, Google Cloud Storage.

Chúc bạn thành công!
