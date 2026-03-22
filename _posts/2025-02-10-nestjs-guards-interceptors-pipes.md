---
layout: article
title: NestJS – Guards, Interceptors & Pipes
tags: [nestjs, guards, interceptors, pipes, middleware]
---
NestJS có hệ thống request lifecycle rõ ràng với nhiều lớp xử lý: **Guard** (phân quyền), **Interceptor** (biến đổi request/response), **Pipe** (validate & transform). Hiểu rõ ba khái niệm này giúp bạn xây dựng API sạch và nhất quán.

## 1. Request Lifecycle trong NestJS

```
Request
  → Middleware
  → Guards        ← Có được phép không?
  → Interceptors  ← Before (transform request)
  → Pipes         ← Validate & transform input
  → Controller
  → Service
  → Interceptors  ← After (transform response)
  → Exception Filters
Response
```

## 2. Guards — Phân quyền

Guard quyết định request có được xử lý không. Trả về `true` = cho qua, `false` = 403.

### Role-based Guard

```typescript
import { Injectable, CanActivate, ExecutionContext, ForbiddenException } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

// Decorator để đánh dấu role cần thiết
export const Roles = (...roles: string[]) => SetMetadata('roles', roles);

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>('roles', [
      context.getHandler(),
      context.getClass(),
    ]);

    if (!requiredRoles) return true; // Không yêu cầu role cụ thể

    const { user } = context.switchToHttp().getRequest();
    const hasRole = requiredRoles.some(role => user?.roles?.includes(role));

    if (!hasRole) throw new ForbiddenException('Bạn không có quyền thực hiện thao tác này');
    return true;
  }
}
```

```typescript
// Dùng trong Controller
@Controller('admin')
@UseGuards(JwtAuthGuard, RolesGuard)
export class AdminController {
  @Get('users')
  @Roles('admin')
  getAllUsers() { ... }

  @Delete('users/:id')
  @Roles('admin', 'super-admin')
  deleteUser(@Param('id') id: string) { ... }
}
```

### Ownership Guard — Chỉ owner mới được sửa

```typescript
@Injectable()
export class OwnerGuard implements CanActivate {
  constructor(private postService: PostsService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const req = context.switchToHttp().getRequest();
    const post = await this.postService.findOne(req.params.id);

    if (post.userId !== req.user.id) {
      throw new ForbiddenException('Bạn không phải chủ sở hữu bài viết này');
    }
    return true;
  }
}
```

## 3. Interceptors — Biến đổi Request & Response

Interceptor dùng pattern RxJS Observable — chạy cả trước và sau controller.

### Response Transform Interceptor — Chuẩn hóa format response

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, { success: boolean; data: T }> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      map(data => ({
        success: true,
        data,
        timestamp: new Date().toISOString(),
      })),
    );
  }
}
```

Kết quả mọi API sẽ trả về cùng format:
```json
{
  "success": true,
  "data": { ... },
  "timestamp": "2025-06-01T10:00:00.000Z"
}
```

### Logging Interceptor — Ghi log thời gian xử lý

```typescript
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  private readonly logger = new Logger('HTTP');

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const req = context.switchToHttp().getRequest();
    const { method, url } = req;
    const start = Date.now();

    return next.handle().pipe(
      tap(() => {
        const ms = Date.now() - start;
        this.logger.log(`${method} ${url} — ${ms}ms`);
      }),
    );
  }
}
```

### Timeout Interceptor — Tự động timeout request

```typescript
import { timeout, catchError } from 'rxjs/operators';
import { TimeoutError, throwError } from 'rxjs';
import { RequestTimeoutException } from '@nestjs/common';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(5000), // 5 giây
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException('Request timeout'));
        }
        return throwError(() => err);
      }),
    );
  }
}
```

### Cache Interceptor (tự viết)

```typescript
@Injectable()
export class HttpCacheInterceptor extends CacheInterceptor {
  trackBy(context: ExecutionContext): string | undefined {
    const req = context.switchToHttp().getRequest();
    // Không cache nếu có auth header (data user cụ thể)
    if (req.headers.authorization) return undefined;
    return super.trackBy(context);
  }
}
```

## 4. Pipes — Validate & Transform

### ParseIntPipe, ParseUUIDPipe built-in

```typescript
@Get(':id')
findOne(
  @Param('id', ParseUUIDPipe) id: string,       // Tự validate UUID
  @Query('page', new ParseIntPipe({ optional: true })) page = 1,
) { ... }
```

### Custom Transform Pipe

```typescript
@Injectable()
export class TrimStringsPipe implements PipeTransform {
  transform(value: any) {
    if (typeof value === 'string') return value.trim();
    if (typeof value === 'object' && value !== null) {
      return Object.fromEntries(
        Object.entries(value).map(([k, v]) => [
          k,
          typeof v === 'string' ? v.trim() : v,
        ]),
      );
    }
    return value;
  }
}
```

### ParseFilePipe với custom validator

```typescript
import { ParseFilePipe, FileValidator } from '@nestjs/common';

export class ImageOnlyValidator extends FileValidator {
  isValid(file?: Express.Multer.File): boolean {
    return ['image/jpeg', 'image/png', 'image/webp'].includes(file?.mimetype ?? '');
  }
  buildErrorMessage(): string {
    return 'Chỉ chấp nhận file ảnh (JPEG, PNG, WebP)';
  }
}

// Dùng trong controller
@Post('avatar')
@UseInterceptors(FileInterceptor('file'))
uploadAvatar(
  @UploadedFile(new ParseFilePipe({
    validators: [new ImageOnlyValidator({})],
  }))
  file: Express.Multer.File,
) { ... }
```

## 5. Áp dụng Global

```typescript
// main.ts
app.useGlobalGuards(new JwtAuthGuard());
app.useGlobalInterceptors(new TransformInterceptor(), new LoggingInterceptor());
app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
```

Hoặc register qua DI (có inject dependency):
```typescript
// app.module.ts
providers: [
  { provide: APP_GUARD, useClass: JwtAuthGuard },
  { provide: APP_INTERCEPTOR, useClass: TransformInterceptor },
  { provide: APP_INTERCEPTOR, useClass: LoggingInterceptor },
  { provide: APP_PIPE, useClass: ValidationPipe },
]
```

## 6. Kết luận

| Khái niệm | Vị trí | Chức năng chính |
|-----------|--------|----------------|
| **Guard** | Trước controller | Phân quyền (có được phép không?) |
| **Interceptor** | Bao quanh controller | Transform request/response, logging, cache, timeout |
| **Pipe** | Trước controller method | Validate và transform input data |

Ba lớp này giúp tách biệt cross-cutting concerns ra khỏi business logic — controller chỉ cần lo xử lý nghiệp vụ.
