---
layout: article
title: Tìm hiểu & Cài đặt NestJS
tags: [nestjs, nodejs, typescript]
---
NestJS là một framework Node.js mạnh mẽ, được xây dựng trên TypeScript, lấy cảm hứng từ kiến trúc của Angular. Nó giúp bạn xây dựng các ứng dụng server-side hiệu quả, có khả năng mở rộng và dễ bảo trì.

## 1. Giới thiệu

NestJS kết hợp các yếu tố từ OOP (Object Oriented Programming), FP (Functional Programming) và FRP (Functional Reactive Programming). Bên dưới, NestJS sử dụng Express (mặc định) hoặc Fastify làm HTTP server.

**Ưu điểm nổi bật:**
- Kiến trúc rõ ràng, module hóa chặt chẽ
- Hỗ trợ TypeScript out of the box
- Dependency Injection (DI) tích hợp sẵn
- CLI mạnh mẽ để tạo code nhanh
- Tích hợp tốt với nhiều thư viện như TypeORM, Mongoose, Swagger, GraphQL

## 2. Yêu cầu

- **Node.js** >= 16.x ([tải tại đây](https://nodejs.org/en/download/){:target="_blank"})
- **npm** hoặc **yarn**

Kiểm tra version:
```bash
node -v
npm -v
```

## 3. Cài đặt NestJS CLI

```bash
npm install -g @nestjs/cli
```

Kiểm tra cài đặt thành công:
```bash
nest --version
```

## 4. Tạo project mới

```bash
nest new my-nestjs-app
```

Chọn package manager (npm hoặc yarn), NestJS CLI sẽ tự động tạo toàn bộ cấu trúc project:

```
my-nestjs-app/
├── src/
│   ├── app.controller.ts
│   ├── app.controller.spec.ts
│   ├── app.module.ts
│   ├── app.service.ts
│   └── main.ts
├── test/
├── nest-cli.json
├── package.json
├── tsconfig.json
└── tsconfig.build.json
```

## 5. Cấu trúc project

### main.ts — Entry point

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();
```

### app.module.ts — Root Module

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';

@Module({
  imports: [],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

### app.controller.ts — Controller

```typescript
import { Controller, Get } from '@nestjs/common';
import { AppService } from './app.service';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }
}
```

### app.service.ts — Service

```typescript
import { Injectable } from '@nestjs/common';

@Injectable()
export class AppService {
  getHello(): string {
    return 'Hello World!';
  }
}
```

## 6. Chạy project

```bash
cd my-nestjs-app
npm run start
```

Chế độ watch (tự reload khi có thay đổi):
```bash
npm run start:dev
```

Truy cập [http://localhost:3000](http://localhost:3000){:target="_blank"} — bạn sẽ thấy `Hello World!`

## 7. Tạo Module, Controller, Service mới

NestJS CLI giúp tạo code rất nhanh:

```bash
# Tạo một module
nest g module users

# Tạo một controller
nest g controller users

# Tạo một service
nest g service users

# Hoặc tạo cả resource (CRUD hoàn chỉnh)
nest g resource users
```

Sau khi chạy `nest g resource users`, bạn chọn **REST API** và chọn có tạo CRUD entry points. NestJS sẽ tự sinh ra đầy đủ module, controller, service, DTO, entities.

## 8. Tích hợp Swagger (API Documentation)

```bash
npm install --save @nestjs/swagger swagger-ui-express
```

Cập nhật `main.ts`:

```typescript
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const config = new DocumentBuilder()
    .setTitle('My NestJS App')
    .setDescription('API Documentation')
    .setVersion('1.0')
    .build();

  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, document);

  await app.listen(3000);
}
bootstrap();
```

Truy cập [http://localhost:3000/api](http://localhost:3000/api){:target="_blank"} để xem Swagger UI.

## 9. Kết luận

NestJS là một framework backend hiện đại với kiến trúc rõ ràng và hệ sinh thái phong phú. Những điểm chính bạn cần nhớ:

- Mọi thứ được tổ chức theo **Module**
- **Controller** xử lý HTTP request/response
- **Service** chứa business logic
- **Decorator** (`@Controller`, `@Get`, `@Injectable`...) là cốt lõi của NestJS
- CLI `nest g` giúp tạo code nhanh chóng

Trong các bài tiếp theo, chúng ta sẽ kết nối NestJS với MongoDB, xây dựng Authentication với JWT và nhiều hơn nữa.
