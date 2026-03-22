---
layout: article
title: NestJS – Microservices với TCP & Redis Transport
tags: [nestjs, microservices, tcp, redis, architecture]
---
NestJS có hỗ trợ tích hợp cho microservices với nhiều transport layer khác nhau: TCP, Redis, NATS, RabbitMQ, Kafka, gRPC. Bài này hướng dẫn xây dựng hệ thống microservices đơn giản với TCP và Redis.

## 1. Kiến trúc Microservices trong NestJS

```
[API Gateway / Client]
    ↓ HTTP request
[NestJS App] ─── TCP ──→ [User Service :3001]
             ─── TCP ──→ [Order Service :3002]
             ─── Redis → [Notification Service]
```

NestJS hỗ trợ hai kiểu giao tiếp:
- **Request-Response**: Client gửi, đợi response (như HTTP)
- **Event**: Client publish event, không đợi (fire & forget)

## 2. TCP Transport — Request/Response

### Microservice (Server)

```typescript
// user-service/main.ts
import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
    transport: Transport.TCP,
    options: {
      host: '0.0.0.0',
      port: 3001,
    },
  });
  await app.listen();
  console.log('User Microservice đang chạy trên port 3001');
}
bootstrap();
```

```typescript
// user-service/users.controller.ts
import { Controller } from '@nestjs/common';
import { MessagePattern, Payload } from '@nestjs/microservices';

@Controller()
export class UsersController {
  @MessagePattern({ cmd: 'get_user' })
  async getUser(@Payload() data: { id: string }) {
    return { id: data.id, name: 'Nguyen Van A', email: 'a@example.com' };
  }

  @MessagePattern({ cmd: 'create_user' })
  async createUser(@Payload() dto: { name: string; email: string }) {
    // Lưu vào DB...
    return { id: 'new-id', ...dto, createdAt: new Date() };
  }

  @MessagePattern({ cmd: 'get_users_by_ids' })
  async getUsersByIds(@Payload() data: { ids: string[] }) {
    return data.ids.map(id => ({ id, name: `User ${id}` }));
  }
}
```

### Client — Gọi Microservice

```typescript
// api-gateway/app.module.ts
import { ClientsModule, Transport } from '@nestjs/microservices';

@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'USER_SERVICE',
        transport: Transport.TCP,
        options: { host: 'localhost', port: 3001 },
      },
      {
        name: 'ORDER_SERVICE',
        transport: Transport.TCP,
        options: { host: 'localhost', port: 3002 },
      },
    ]),
  ],
})
export class AppModule {}
```

```typescript
// api-gateway/users.controller.ts
import { Inject, Controller, Get, Post, Body, Param } from '@nestjs/common';
import { ClientProxy } from '@nestjs/microservices';
import { firstValueFrom } from 'rxjs';

@Controller('users')
export class UsersController {
  constructor(
    @Inject('USER_SERVICE') private userService: ClientProxy,
  ) {}

  @Get(':id')
  async getUser(@Param('id') id: string) {
    // Gọi microservice và đợi response
    return firstValueFrom(
      this.userService.send({ cmd: 'get_user' }, { id }),
    );
  }

  @Post()
  async createUser(@Body() dto: { name: string; email: string }) {
    return firstValueFrom(
      this.userService.send({ cmd: 'create_user' }, dto),
    );
  }
}
```

## 3. Redis Transport — Event-based

Redis phù hợp cho event-driven: publish event không cần đợi response.

```typescript
// notification-service/main.ts
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.REDIS,
  options: {
    host: process.env.REDIS_HOST ?? 'localhost',
    port: Number(process.env.REDIS_PORT ?? 6379),
  },
});
```

```typescript
// notification-service/notification.controller.ts
import { EventPattern, Payload } from '@nestjs/microservices';

@Controller()
export class NotificationController {
  @EventPattern('user.registered')
  async handleUserRegistered(@Payload() data: { userId: string; email: string }) {
    console.log(`Gửi welcome email đến ${data.email}`);
    // await emailService.sendWelcome(data.email);
  }

  @EventPattern('order.placed')
  async handleOrderPlaced(@Payload() data: { orderId: string; userId: string }) {
    console.log(`Gửi xác nhận đơn hàng ${data.orderId}`);
  }
}
```

```typescript
// api-gateway — Publish event (không đợi)
@Inject('NOTIFICATION_SERVICE') private notifService: ClientProxy;

// Sau khi user đăng ký
this.notifService.emit('user.registered', { userId: user.id, email: user.email });
```

## 4. Kết hợp HTTP API + Microservices

```typescript
// api-gateway/orders.controller.ts — Orchestrate nhiều microservice
@Post()
async createOrder(@Body() dto: CreateOrderDto, @CurrentUser() user: User) {
  // 1. Validate user
  const userProfile = await firstValueFrom(
    this.userService.send({ cmd: 'get_user' }, { id: user.id }),
  );

  // 2. Tạo order
  const order = await firstValueFrom(
    this.orderService.send({ cmd: 'create_order' }, { ...dto, userId: user.id }),
  );

  // 3. Notify (fire & forget — không cần đợi)
  this.notifService.emit('order.placed', { orderId: order.id, userId: user.id });

  return order;
}
```

## 5. Error Handling trong Microservices

```typescript
// Microservice — throw exception
import { RpcException } from '@nestjs/microservices';

@MessagePattern({ cmd: 'get_user' })
async getUser(@Payload() data: { id: string }) {
  const user = await this.usersService.findById(data.id);
  if (!user) throw new RpcException({ message: 'User not found', status: 404 });
  return user;
}

// Client — catch exception
try {
  const user = await firstValueFrom(
    this.userService.send({ cmd: 'get_user' }, { id }).pipe(
      catchError(err => throwError(() => new NotFoundException(err.message))),
    ),
  );
} catch (e) {
  throw new NotFoundException('Người dùng không tồn tại');
}
```

## 6. Kết luận

NestJS Microservices đơn giản hóa giao tiếp service-to-service:

- **TCP**: Request/Response — phù hợp khi cần đợi kết quả
- **Redis**: Event-based — phù hợp cho notification, background tasks
- `send()` = request/response (có await), `emit()` = fire & forget
- `@MessagePattern` = xử lý request, `@EventPattern` = xử lý event

Bắt đầu với monolith, refactor sang microservices khi thực sự cần — đừng over-engineer từ đầu.
