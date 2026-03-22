---
layout: article
title: NestJS – Rate Limiting & Throttling bảo vệ API
tags: [nestjs, rate-limiting, throttling, security, nodejs]
---
Rate Limiting là kỹ thuật giới hạn số request từ một client trong một khoảng thời gian nhất định — bảo vệ API khỏi abuse, DDoS nhẹ, và giảm tải server. NestJS cung cấp `@nestjs/throttler` tích hợp sẵn, dễ dùng.

## 1. Cài đặt

```bash
npm install @nestjs/throttler
```

## 2. Cấu hình cơ bản

```typescript
// app.module.ts
import { ThrottlerModule, ThrottlerGuard } from '@nestjs/throttler';
import { APP_GUARD } from '@nestjs/core';

@Module({
  imports: [
    ThrottlerModule.forRoot([
      {
        name: 'short',
        ttl: 1000,    // 1 giây
        limit: 3,     // Tối đa 3 request/giây
      },
      {
        name: 'medium',
        ttl: 10000,   // 10 giây
        limit: 20,    // Tối đa 20 request/10 giây
      },
      {
        name: 'long',
        ttl: 60000,   // 1 phút
        limit: 100,   // Tối đa 100 request/phút
      },
    ]),
  ],
  providers: [
    {
      provide: APP_GUARD,
      useClass: ThrottlerGuard, // Áp dụng globally
    },
  ],
})
export class AppModule {}
```

Khi vượt giới hạn, client nhận HTTP 429 Too Many Requests.

## 3. Skip throttling cho một số route

```typescript
import { SkipThrottle, Throttle } from '@nestjs/throttler';

@Controller('users')
export class UsersController {
  // Route này bị skip hoàn toàn
  @SkipThrottle()
  @Get('health')
  health() {
    return { status: 'ok' };
  }

  // Override giới hạn cụ thể cho route này
  @Throttle({ short: { ttl: 1000, limit: 1 } }) // Chỉ 1 request/giây cho login
  @Post('login')
  login(@Body() dto: LoginDto) {
    return this.authService.login(dto);
  }

  // Route bình thường — theo config global
  @Get()
  findAll() {
    return this.usersService.findAll();
  }
}
```

## 4. Custom Key — Throttle theo user ID

Mặc định throttle theo IP. Khi user đã đăng nhập, nên throttle theo user ID:

```typescript
// throttler.guard.ts
import { ThrottlerGuard } from '@nestjs/throttler';
import { ExecutionContext, Injectable } from '@nestjs/common';

@Injectable()
export class CustomThrottlerGuard extends ThrottlerGuard {
  protected async getTracker(req: Record<string, any>): Promise<string> {
    // Nếu đã auth → dùng userId, chưa auth → dùng IP
    return req.user?.id ?? req.ip;
  }

  // Custom error message
  protected throwThrottlingException(): void {
    throw new ThrottlerException('Quá nhiều request. Vui lòng thử lại sau.');
  }
}
```

```typescript
// Dùng custom guard thay vì mặc định
providers: [
  {
    provide: APP_GUARD,
    useClass: CustomThrottlerGuard,
  },
],
```

## 5. Throttling với Redis (Distributed)

Khi chạy nhiều instance, cần store throttle state trên Redis:

```bash
npm install @nestjs/throttler ioredis throttler-storage-redis
```

```typescript
import { ThrottlerStorageRedisService } from 'nestjs-throttler-storage-redis';
import Redis from 'ioredis';

ThrottlerModule.forRootAsync({
  useFactory: () => ({
    throttlers: [
      { ttl: 60000, limit: 100 },
    ],
    storage: new ThrottlerStorageRedisService(
      new Redis({ host: 'localhost', port: 6379 }),
    ),
  }),
}),
```

## 6. Rate Limiting per Route type (API vs Auth)

```typescript
// Tạo nhiều profile throttling
ThrottlerModule.forRoot([
  {
    name: 'api',
    ttl: 60000,
    limit: 60,   // 60 req/phút — cho API thông thường
  },
  {
    name: 'auth',
    ttl: 60000,
    limit: 5,    // 5 req/phút — cho login/register
  },
]),
```

```typescript
@Controller('auth')
export class AuthController {
  @Throttle({ auth: { ttl: 60000, limit: 5 } })
  @Post('login')
  login(@Body() dto: LoginDto) {}

  @Throttle({ auth: { ttl: 60000, limit: 3 } })
  @Post('forgot-password')
  forgotPassword(@Body() dto: ForgotPasswordDto) {}
}
```

## 7. Response Headers

Khi throttling active, NestJS tự thêm headers:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1735689600
Retry-After: 30    ← Khi bị 429
```

Client có thể đọc headers để hiển thị "Thử lại sau X giây".

## 8. Kết hợp với Helmet & CORS

```typescript
// main.ts
import helmet from 'helmet';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Security headers
  app.use(helmet());

  // CORS
  app.enableCors({
    origin: process.env.ALLOWED_ORIGINS?.split(',') ?? '*',
    methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  });

  await app.listen(3000);
}
```

## 9. Kết luận

- **`ThrottlerModule.forRoot()`**: Cấu hình multiple throttle windows (short/medium/long)
- **`@SkipThrottle()`**: Bỏ qua cho health check, public endpoints
- **`@Throttle()`**: Override giới hạn cho route cụ thể (auth endpoints nên strict hơn)
- **Custom key**: Throttle theo user ID thay vì IP khi đã authenticate
- **Redis storage**: Bắt buộc khi scale horizontal (nhiều instance)

Rate Limiting là lớp bảo vệ đầu tiên — luôn enable trước khi deploy production.
