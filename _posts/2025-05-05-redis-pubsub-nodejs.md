---
layout: article
title: Redis Pub/Sub – Giao tiếp real-time giữa các service
tags: [redis, pubsub, nodejs, microservices, realtime]
---
Redis Pub/Sub (Publish/Subscribe) là cơ chế messaging nhẹ và nhanh, cho phép các service giao tiếp với nhau theo mô hình publisher-subscriber. Rất hữu ích trong microservices và hệ thống event-driven.

## 1. Pub/Sub là gì?

```
[Publisher] → publish("user-events", {type: "registered", userId: "123"})
                        ↓
                   [Redis Channel]
                        ↓
[Subscriber 1] ← email-service nhận và gửi welcome email
[Subscriber 2] ← analytics-service nhận và cập nhật metrics
[Subscriber 3] ← notification-service nhận và push notification
```

- **Publisher**: Gửi message vào channel, không biết ai nhận
- **Subscriber**: Lắng nghe channel, nhận message theo thời gian thực
- **Channel**: Kênh giao tiếp theo tên

## 2. Cài đặt

```bash
npm install --save redis
```

## 3. Publisher & Subscriber cơ bản

```typescript
import { createClient, RedisClientType } from 'redis';

// Tạo kết nối Redis
function createRedisClient(): RedisClientType {
  const client = createClient({
    socket: {
      host: process.env.REDIS_HOST ?? 'localhost',
      port: Number(process.env.REDIS_PORT ?? 6379),
    },
  });

  client.on('error', err => console.error('Redis error:', err));
  return client as RedisClientType;
}

// Publisher
async function publishEvent(channel: string, data: object) {
  const publisher = createRedisClient();
  await publisher.connect();

  await publisher.publish(channel, JSON.stringify(data));

  await publisher.disconnect();
}

// Subscriber
async function subscribeToChannel(channel: string, handler: (data: any) => void) {
  const subscriber = createRedisClient();
  await subscriber.connect();

  await subscriber.subscribe(channel, (message) => {
    try {
      const data = JSON.parse(message);
      handler(data);
    } catch (e) {
      console.error('Invalid message:', message);
    }
  });

  console.log(`Subscribed to "${channel}"`);
  // subscriber.disconnect() chỉ gọi khi muốn dừng
}
```

> **Quan trọng**: Client đang subscribe **không thể** dùng cho lệnh Redis khác. Phải tạo 2 client riêng biệt — một cho pub, một cho sub.

## 4. Tích hợp NestJS — Event Bus nội bộ

```typescript
// src/redis/redis-pubsub.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { createClient, RedisClientType } from 'redis';

type MessageHandler = (data: any) => void | Promise<void>;

@Injectable()
export class RedisPubSubService implements OnModuleInit, OnModuleDestroy {
  private publisher: RedisClientType;
  private subscriber: RedisClientType;
  private handlers = new Map<string, MessageHandler[]>();

  async onModuleInit() {
    const options = {
      socket: {
        host: process.env.REDIS_HOST,
        port: Number(process.env.REDIS_PORT),
      },
    };

    this.publisher = createClient(options) as RedisClientType;
    this.subscriber = createClient(options) as RedisClientType;

    await this.publisher.connect();
    await this.subscriber.connect();
  }

  async publish(channel: string, data: object): Promise<void> {
    await this.publisher.publish(channel, JSON.stringify(data));
  }

  async subscribe(channel: string, handler: MessageHandler): Promise<void> {
    if (!this.handlers.has(channel)) {
      this.handlers.set(channel, []);
      await this.subscriber.subscribe(channel, async (message) => {
        const data = JSON.parse(message);
        const channelHandlers = this.handlers.get(channel) ?? [];
        await Promise.all(channelHandlers.map(h => h(data)));
      });
    }
    this.handlers.get(channel)!.push(handler);
  }

  async onModuleDestroy() {
    await this.publisher.disconnect();
    await this.subscriber.disconnect();
  }
}
```

## 5. Sử dụng trong Service

```typescript
// Trong UserService — Publish event khi user đăng ký
@Injectable()
export class UsersService {
  constructor(private readonly pubSub: RedisPubSubService) {}

  async register(dto: RegisterDto) {
    const user = await this.createUser(dto);

    // Publish event — các service khác tự xử lý
    await this.pubSub.publish('user-events', {
      type: 'user.registered',
      payload: { userId: user.id, email: user.email, name: user.name },
      timestamp: new Date().toISOString(),
    });

    return user;
  }
}

// Trong EmailService — Subscribe và gửi email
@Injectable()
export class EmailListenerService implements OnModuleInit {
  constructor(private readonly pubSub: RedisPubSubService) {}

  async onModuleInit() {
    await this.pubSub.subscribe('user-events', async (event) => {
      if (event.type === 'user.registered') {
        await this.sendWelcomeEmail(event.payload.email, event.payload.name);
      }
    });
  }

  private async sendWelcomeEmail(email: string, name: string) {
    console.log(`Gửi welcome email đến ${email}`);
    // await mailer.send(...)
  }
}
```

## 6. Pattern Subscribe — Wildcard

Lắng nghe nhiều channel cùng lúc với wildcard:

```typescript
// Lắng nghe tất cả events bắt đầu bằng "order-"
await subscriber.pSubscribe('order-*', (message, channel) => {
  console.log(`Channel: ${channel}, Message: ${message}`);
});

// Publish
await publisher.publish('order-created', JSON.stringify({ orderId: '123' }));
await publisher.publish('order-shipped', JSON.stringify({ orderId: '123' }));
// Cả hai đều được nhận
```

## 7. So sánh Pub/Sub vs Queue (BullMQ)

| Tiêu chí | Redis Pub/Sub | BullMQ (Queue) |
|---------|--------------|----------------|
| **Delivery** | At-most-once (fire & forget) | At-least-once (có retry) |
| **Message persistence** | Không (mất khi subscriber offline) | Có (lưu trong Redis) |
| **Multiple consumers** | Tất cả nhận | Chỉ một consumer xử lý |
| **Retry** | Không | Có |
| **Use case** | Broadcast event, real-time | Background job, task queue |

## 8. Kết luận

Redis Pub/Sub phù hợp cho:

- **Broadcast event** giữa nhiều service (user registered, order placed)
- **Cache invalidation** — notify tất cả instances xóa cache
- **Real-time notification** kết hợp với WebSocket
- **Decoupling** services — service không cần biết nhau, chỉ cần biết channel

Với tác vụ cần **đảm bảo xử lý** và **retry**, dùng BullMQ thay vì Pub/Sub.
