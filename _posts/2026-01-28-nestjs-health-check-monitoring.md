---
layout: article
title: NestJS – Health Check & Monitoring với Terminus
tags: [nestjs, health-check, monitoring, terminus, kubernetes, devops]
---
Health Check là endpoint `/health` mà load balancer, Kubernetes, và monitoring tools dùng để kiểm tra xem app có đang hoạt động không. NestJS cung cấp `@nestjs/terminus` để implement dễ dàng.

## 1. Cài đặt

```bash
npm install @nestjs/terminus @nestjs/axios
```

## 2. Cấu hình cơ bản

```typescript
// src/health/health.module.ts
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { HttpModule } from '@nestjs/axios';
import { HealthController } from './health.controller';
import { DatabaseHealthIndicator } from './indicators/database.health';

@Module({
  imports: [TerminusModule, HttpModule],
  controllers: [HealthController],
  providers: [DatabaseHealthIndicator],
})
export class HealthModule {}
```

```typescript
// src/health/health.controller.ts
import { Controller, Get } from '@nestjs/common';
import {
  HealthCheckService,
  HealthCheck,
  MongooseHealthIndicator,
  MemoryHealthIndicator,
  DiskHealthIndicator,
  HttpHealthIndicator,
} from '@nestjs/terminus';
import { InjectConnection } from '@nestjs/mongoose';
import { Connection } from 'mongoose';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private mongoose: MongooseHealthIndicator,
    private memory: MemoryHealthIndicator,
    private disk: DiskHealthIndicator,
    private http: HttpHealthIndicator,
  ) {}

  // Liveness probe — app có đang chạy không?
  @Get('live')
  @HealthCheck()
  liveness() {
    return this.health.check([
      // Chỉ check bộ nhớ — không check external deps
      () => this.memory.checkHeap('memory_heap', 512 * 1024 * 1024), // 512MB
    ]);
  }

  // Readiness probe — app có sẵn sàng nhận traffic không?
  @Get('ready')
  @HealthCheck()
  readiness() {
    return this.health.check([
      // Check database
      () => this.mongoose.pingCheck('mongodb'),

      // Check disk
      () => this.disk.checkStorage('storage', {
        thresholdPercent: 0.9, // Alert khi dùng > 90% disk
        path: '/',
      }),

      // Check external API
      () => this.http.pingCheck('payment-api', 'https://api.stripe.com/v1'),
    ]);
  }

  // Startup probe — lần đầu khởi động
  @Get('startup')
  @HealthCheck()
  startup() {
    return this.health.check([
      () => this.mongoose.pingCheck('mongodb'),
    ]);
  }
}
```

Response khi tất cả OK:

```json
{
  "status": "ok",
  "info": {
    "mongodb": { "status": "up" },
    "memory_heap": { "status": "up" },
    "storage": { "status": "up" }
  },
  "error": {},
  "details": { ... }
}
```

Response khi có lỗi (HTTP 503):

```json
{
  "status": "error",
  "info": { "memory_heap": { "status": "up" } },
  "error": {
    "mongodb": { "status": "down", "message": "Connection timeout" }
  }
}
```

## 3. Custom Health Indicator

```typescript
// src/health/indicators/redis.health.ts
import { Injectable } from '@nestjs/common';
import { HealthIndicator, HealthIndicatorResult, HealthCheckError } from '@nestjs/terminus';
import Redis from 'ioredis';

@Injectable()
export class RedisHealthIndicator extends HealthIndicator {
  private redis: Redis;

  constructor() {
    super();
    this.redis = new Redis(process.env.REDIS_URL!);
  }

  async isHealthy(key: string): Promise<HealthIndicatorResult> {
    try {
      const result = await this.redis.ping();
      const isHealthy = result === 'PONG';

      if (!isHealthy) {
        throw new HealthCheckError(
          'Redis check failed',
          this.getStatus(key, false),
        );
      }

      return this.getStatus(key, true, {
        latency: await this.measureLatency(),
      });
    } catch (error) {
      throw new HealthCheckError(
        'Redis check failed',
        this.getStatus(key, false, { error: error.message }),
      );
    }
  }

  private async measureLatency(): Promise<number> {
    const start = Date.now();
    await this.redis.ping();
    return Date.now() - start;
  }
}

// src/health/indicators/queue.health.ts
@Injectable()
export class QueueHealthIndicator extends HealthIndicator {
  constructor(@InjectQueue('default') private queue: Queue) { super(); }

  async isHealthy(key: string): Promise<HealthIndicatorResult> {
    try {
      const [waiting, active, failed] = await Promise.all([
        this.queue.getWaitingCount(),
        this.queue.getActiveCount(),
        this.queue.getFailedCount(),
      ]);

      const isHealthy = failed < 100; // Alert nếu > 100 failed jobs

      return this.getStatus(key, isHealthy, { waiting, active, failed });
    } catch (error) {
      throw new HealthCheckError('Queue check', this.getStatus(key, false));
    }
  }
}
```

```typescript
// Thêm vào controller
@Get('ready')
@HealthCheck()
readiness() {
  return this.health.check([
    () => this.mongoose.pingCheck('mongodb'),
    () => this.redisHealth.isHealthy('redis'),
    () => this.queueHealth.isHealthy('queue'),
  ]);
}
```

## 4. Kubernetes Probes

```yaml
# k8s/deployment.yaml
spec:
  containers:
    - name: api
      image: shopxyz/api:latest
      ports:
        - containerPort: 3000

      # Kiểm tra app còn sống không
      livenessProbe:
        httpGet:
          path: /health/live
          port: 3000
        initialDelaySeconds: 30
        periodSeconds: 10
        failureThreshold: 3

      # Kiểm tra app sẵn sàng nhận traffic chưa
      readinessProbe:
        httpGet:
          path: /health/ready
          port: 3000
        initialDelaySeconds: 10
        periodSeconds: 5
        failureThreshold: 3

      # Chờ app khởi động xong (slow start)
      startupProbe:
        httpGet:
          path: /health/startup
          port: 3000
        failureThreshold: 30   # Chờ tối đa 30 * 10s = 5 phút
        periodSeconds: 10
```

## 5. Metrics với Prometheus

```bash
npm install @willsoto/nestjs-prometheus prom-client
```

```typescript
// app.module.ts
import { PrometheusModule } from '@willsoto/nestjs-prometheus';

PrometheusModule.register({
  path: '/metrics',  // Prometheus scrape tại đây
  defaultMetrics: { enabled: true },
}),
```

```typescript
// Custom metrics
import { makeCounterProvider, makeHistogramProvider } from '@willsoto/nestjs-prometheus';
import { Counter, Histogram, InjectMetric } from '@willsoto/nestjs-prometheus';

// Đăng ký metrics
providers: [
  makeCounterProvider({ name: 'http_requests_total', help: 'Total HTTP requests', labelNames: ['method', 'route', 'status'] }),
  makeHistogramProvider({ name: 'http_request_duration_ms', help: 'Request duration', labelNames: ['method', 'route'] }),
],

// Dùng trong interceptor
@Injectable()
export class MetricsInterceptor implements NestInterceptor {
  constructor(
    @InjectMetric('http_requests_total') private counter: Counter<string>,
    @InjectMetric('http_request_duration_ms') private histogram: Histogram<string>,
  ) {}

  intercept(context: ExecutionContext, next: CallHandler) {
    const req = context.switchToHttp().getRequest();
    const start = Date.now();

    return next.handle().pipe(
      tap(() => {
        const duration = Date.now() - start;
        const status = context.switchToHttp().getResponse().statusCode;

        this.counter.inc({ method: req.method, route: req.route?.path, status });
        this.histogram.observe({ method: req.method, route: req.route?.path }, duration);
      }),
    );
  }
}
```

## 6. Kết luận

- **`/health/live`**: Liveness probe — chỉ check app còn sống, không check external deps
- **`/health/ready`**: Readiness probe — check database, Redis, external APIs
- **`/health/startup`**: Startup probe — cho slow-starting apps (DB migration)
- **Custom Indicator**: Extend `HealthIndicator` để check bất kỳ dependency nào
- **Prometheus**: Export metrics cho Grafana dashboard — monitoring production

Ba probe (liveness/readiness/startup) phải được cấu hình đúng trong Kubernetes — sai có thể gây restart loop hoặc traffic routing sai.
