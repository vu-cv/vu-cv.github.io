---
layout: article
title: NestJS – Caching với Redis
tags: [nestjs, redis, caching, performance]
---
Caching là kỹ thuật tăng hiệu suất hiệu quả nhất — thay vì tính toán hoặc query database mỗi lần, kết quả được lưu tạm và tái sử dụng. Redis là in-memory store lý tưởng cho caching trong NestJS.

## 1. Cài đặt

```bash
npm install --save @nestjs/cache-manager cache-manager cache-manager-redis-yet redis
```

## 2. Cấu hình Cache Module

```typescript
// app.module.ts
import { CacheModule } from '@nestjs/cache-manager';
import { redisStore } from 'cache-manager-redis-yet';

@Module({
  imports: [
    CacheModule.registerAsync({
      isGlobal: true,
      useFactory: async () => ({
        store: await redisStore({
          socket: {
            host: process.env.REDIS_HOST ?? 'localhost',
            port: Number(process.env.REDIS_PORT ?? 6379),
          },
        }),
        ttl: 60 * 1000, // TTL mặc định: 60 giây (milliseconds)
      }),
    }),
  ],
})
export class AppModule {}
```

## 3. Cache tự động với Interceptor

Cách đơn giản nhất — cache toàn bộ response của một endpoint:

```typescript
import { Controller, Get, UseInterceptors, Param } from '@nestjs/common';
import { CacheInterceptor, CacheTTL, CacheKey } from '@nestjs/cache-manager';

@Controller('products')
@UseInterceptors(CacheInterceptor) // Áp dụng cho toàn bộ controller
export class ProductsController {
  constructor(private productsService: ProductsService) {}

  @Get()
  @CacheTTL(30 * 1000) // Override TTL: 30 giây
  findAll() {
    return this.productsService.findAll();
  }

  @Get(':id')
  @CacheKey('product-detail') // Custom cache key
  @CacheTTL(60 * 1000)
  findOne(@Param('id') id: string) {
    return this.productsService.findOne(id);
  }
}
```

## 4. Cache thủ công với CacheManager

Kiểm soát chi tiết hơn khi cần logic phức tạp:

```typescript
import { Injectable, Inject } from '@nestjs/common';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';

@Injectable()
export class ProductsService {
  constructor(
    @Inject(CACHE_MANAGER) private cacheManager: Cache,
    private readonly productRepo: ProductRepository,
  ) {}

  async findAll(): Promise<Product[]> {
    const cacheKey = 'products:all';

    // 1. Kiểm tra cache
    const cached = await this.cacheManager.get<Product[]>(cacheKey);
    if (cached) return cached;

    // 2. Query DB
    const products = await this.productRepo.findAll();

    // 3. Lưu vào cache (TTL 5 phút)
    await this.cacheManager.set(cacheKey, products, 5 * 60 * 1000);

    return products;
  }

  async findOne(id: string): Promise<Product> {
    const cacheKey = `product:${id}`;
    const cached = await this.cacheManager.get<Product>(cacheKey);
    if (cached) return cached;

    const product = await this.productRepo.findById(id);
    await this.cacheManager.set(cacheKey, product, 10 * 60 * 1000);
    return product;
  }

  async update(id: string, data: UpdateProductDto): Promise<Product> {
    const product = await this.productRepo.update(id, data);

    // Xóa cache sau khi update
    await this.cacheManager.del(`product:${id}`);
    await this.cacheManager.del('products:all');

    return product;
  }

  async delete(id: string): Promise<void> {
    await this.productRepo.delete(id);
    await this.cacheManager.del(`product:${id}`);
    await this.cacheManager.del('products:all');
  }
}
```

## 5. Cache Pattern — Cache-Aside

Pattern phổ biến nhất: kiểm tra cache trước, nếu miss thì query DB và điền vào cache:

```typescript
// Utility function tái sử dụng
async function withCache<T>(
  cacheManager: Cache,
  key: string,
  ttl: number,
  fetcher: () => Promise<T>,
): Promise<T> {
  const cached = await cacheManager.get<T>(key);
  if (cached !== null && cached !== undefined) return cached;

  const data = await fetcher();
  await cacheManager.set(key, data, ttl);
  return data;
}

// Sử dụng
async findUserOrders(userId: string) {
  return withCache(
    this.cacheManager,
    `user:${userId}:orders`,
    5 * 60 * 1000,
    () => this.orderRepo.findByUserId(userId),
  );
}
```

## 6. Cache Invalidation theo Pattern (Redis)

Xóa nhiều key theo pattern — dùng trực tiếp Redis client:

```typescript
import { Injectable } from '@nestjs/common';
import { createClient } from 'redis';

@Injectable()
export class CacheService {
  private redisClient = createClient({
    socket: { host: process.env.REDIS_HOST, port: Number(process.env.REDIS_PORT) },
  });

  async deleteByPattern(pattern: string) {
    const keys = await this.redisClient.keys(pattern);
    if (keys.length > 0) {
      await this.redisClient.del(keys);
    }
  }
}

// Ví dụ: Xóa tất cả cache liên quan đến user khi user logout
await this.cacheService.deleteByPattern(`user:${userId}:*`);
```

## 7. Cache phân tán — nhiều instance

Khi chạy nhiều instance NestJS (horizontal scaling), Redis cache tự động được chia sẻ giữa tất cả instances — không cần thêm cấu hình gì.

```
[Instance 1] ──┐
[Instance 2] ──┼──→ [Redis] ← Shared cache
[Instance 3] ──┘
```

## 8. Chiến lược TTL hợp lý

| Loại dữ liệu | TTL đề xuất |
|-------------|-------------|
| Dữ liệu tĩnh (config, danh mục) | 1–24 giờ |
| Danh sách sản phẩm | 5–30 phút |
| Chi tiết sản phẩm | 10–60 phút |
| Kết quả tìm kiếm | 1–5 phút |
| Session user | 15–30 phút |
| Rate limit counter | 1 phút |

## 9. Kết luận

Redis + NestJS Cache Module giúp giảm load database đáng kể:

- `CacheInterceptor` — cache endpoint nhanh không cần code thêm
- `CACHE_MANAGER` — cache thủ công với kiểm soát chi tiết
- Luôn **xóa cache** sau khi update/delete để tránh stale data
- Redis chia sẻ cache giữa nhiều instance — phù hợp production

Kết hợp caching đúng cách có thể giảm response time từ 200ms xuống còn 2ms.
