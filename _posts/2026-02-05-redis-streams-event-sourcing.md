---
layout: article
title: Redis Streams – Event Sourcing nhẹ với NodeJS
tags: [redis, streams, event-sourcing, nodejs, nestjs, real-time]
---
Redis Streams là cấu trúc dữ liệu log append-only trong Redis — nhẹ hơn Kafka, tích hợp sẵn nếu đã dùng Redis. Phù hợp cho event sourcing quy mô vừa, activity logs, real-time feeds.

## 1. Redis Streams vs Pub/Sub vs Kafka

```
Redis Pub/Sub:   Fire & forget — không lưu message
Redis Streams:   Persistent log — consumer group, pending messages
Kafka:           Distributed log — multi-broker, replay lâu dài
```

Redis Streams nằm giữa — có persistence và consumer groups như Kafka, nhưng single-node.

## 2. Các khái niệm

```
Stream (topic):   orders:events
Entry:            { id: "1735689600000-0", orderId: "123", action: "created" }
Consumer Group:   notifications, inventory, analytics
Consumer:         notifications-1, notifications-2 (trong cùng group)
PEL:              Pending Entry List — messages đã đọc nhưng chưa ACK
```

## 3. Thêm message vào Stream

```typescript
// src/streams/redis-streams.service.ts
import { Injectable, Logger } from '@nestjs/common';
import Redis from 'ioredis';

@Injectable()
export class RedisStreamsService {
  private readonly logger = new Logger(RedisStreamsService.name);
  private redis: Redis;

  constructor() {
    this.redis = new Redis(process.env.REDIS_URL!);
  }

  // Thêm message vào stream
  async addToStream(
    stream: string,
    fields: Record<string, string>,
  ): Promise<string> {
    // '*' = auto-generate ID dựa trên timestamp
    const messageId = await this.redis.xadd(stream, '*', ...Object.entries(fields).flat());
    return messageId!;
  }

  // Đọc messages mới nhất (không dùng consumer group)
  async readStream(stream: string, lastId = '$', count = 10): Promise<any[]> {
    const results = await this.redis.xread('COUNT', count, 'BLOCK', 0, 'STREAMS', stream, lastId);
    if (!results) return [];

    return results[0][1].map(([id, fields]) => ({
      id,
      ...this.parseFields(fields as string[]),
    }));
  }

  // Tạo consumer group
  async createGroup(stream: string, group: string, startId = '$'): Promise<void> {
    try {
      await this.redis.xgroup('CREATE', stream, group, startId, 'MKSTREAM');
    } catch (e) {
      if (!e.message.includes('BUSYGROUP')) throw e; // Ignore nếu group đã tồn tại
    }
  }

  // Consumer group — đọc và xử lý messages
  async readGroupMessages(
    stream: string,
    group: string,
    consumer: string,
    count = 10,
    blockMs = 2000,
  ): Promise<Array<{ id: string; fields: Record<string, string> }>> {
    const results = await this.redis.xreadgroup(
      'GROUP', group, consumer,
      'COUNT', count,
      'BLOCK', blockMs,
      'STREAMS', stream, '>',  // '>' = chỉ lấy messages chưa được deliver
    );

    if (!results) return [];

    return results[0][1].map(([id, fields]) => ({
      id: id as string,
      fields: this.parseFields(fields as string[]),
    }));
  }

  // ACK message đã xử lý xong
  async ack(stream: string, group: string, messageId: string): Promise<void> {
    await this.redis.xack(stream, group, messageId);
  }

  // Đọc messages pending (đã deliver nhưng chưa ACK — dùng khi restart)
  async readPending(stream: string, group: string, consumer: string): Promise<any[]> {
    const results = await this.redis.xreadgroup(
      'GROUP', group, consumer,
      'COUNT', 100,
      'STREAMS', stream, '0',  // '0' = lấy pending messages
    );

    if (!results) return [];
    return results[0][1].map(([id, fields]) => ({
      id: id as string,
      fields: this.parseFields(fields as string[]),
    }));
  }

  // Trim stream giữ tối đa N entries
  async trimStream(stream: string, maxLen: number): Promise<void> {
    await this.redis.xtrim(stream, 'MAXLEN', '~', maxLen);
  }

  private parseFields(fields: string[]): Record<string, string> {
    const obj: Record<string, string> = {};
    for (let i = 0; i < fields.length; i += 2) {
      obj[fields[i]] = fields[i + 1];
    }
    return obj;
  }
}
```

## 4. Event Producer

```typescript
// src/orders/orders.service.ts
export const STREAMS = {
  ORDERS: 'orders:events',
  USERS: 'users:events',
  PAYMENTS: 'payments:events',
} as const;

@Injectable()
export class OrdersService {
  constructor(
    private streamsService: RedisStreamsService,
    private orderModel: Model<Order>,
  ) {}

  async createOrder(dto: CreateOrderDto, userId: string): Promise<Order> {
    const order = await this.orderModel.create({ ...dto, userId });

    // Ghi vào stream
    await this.streamsService.addToStream(STREAMS.ORDERS, {
      action: 'created',
      orderId: order._id.toString(),
      userId,
      total: order.total.toString(),
      items: JSON.stringify(order.items),
      timestamp: Date.now().toString(),
    });

    return order;
  }

  async updateStatus(orderId: string, status: string): Promise<void> {
    await this.orderModel.findByIdAndUpdate(orderId, { status });

    await this.streamsService.addToStream(STREAMS.ORDERS, {
      action: 'status_updated',
      orderId,
      status,
      timestamp: Date.now().toString(),
    });
  }
}
```

## 5. Consumer với Worker loop

```typescript
// src/notifications/stream-consumer.service.ts
@Injectable()
export class NotificationStreamConsumer implements OnModuleInit, OnModuleDestroy {
  private readonly logger = new Logger(NotificationStreamConsumer.name);
  private running = true;
  private readonly STREAM = STREAMS.ORDERS;
  private readonly GROUP = 'notifications';
  private readonly CONSUMER = `notifications-${process.pid}`;

  constructor(
    private streamsService: RedisStreamsService,
    private mailService: MailService,
  ) {}

  async onModuleInit() {
    await this.streamsService.createGroup(this.STREAM, this.GROUP, '0');

    // Xử lý pending messages trước (từ lần restart trước)
    await this.processPending();

    // Bắt đầu consume loop
    this.startConsumeLoop();
  }

  async onModuleDestroy() {
    this.running = false;
  }

  private async processPending() {
    const pending = await this.streamsService.readPending(
      this.STREAM, this.GROUP, this.CONSUMER,
    );
    for (const msg of pending) {
      await this.processMessage(msg.id, msg.fields);
    }
  }

  private async startConsumeLoop() {
    this.logger.log(`Starting consumer: ${this.CONSUMER}`);

    while (this.running) {
      try {
        const messages = await this.streamsService.readGroupMessages(
          this.STREAM, this.GROUP, this.CONSUMER,
          count: 10,
          blockMs: 2000,
        );

        for (const { id, fields } of messages) {
          await this.processMessage(id, fields);
        }
      } catch (error) {
        if (this.running) {
          this.logger.error('Consumer error:', error.message);
          await new Promise(r => setTimeout(r, 1000)); // Wait before retry
        }
      }
    }
  }

  private async processMessage(id: string, fields: Record<string, string>) {
    try {
      switch (fields.action) {
        case 'created':
          await this.mailService.sendOrderConfirmation(fields.userId, fields.orderId);
          break;
        case 'status_updated':
          if (fields.status === 'shipped') {
            await this.mailService.sendShippingNotification(fields.userId, fields.orderId);
          }
          break;
      }

      // ACK sau khi xử lý thành công
      await this.streamsService.ack(this.STREAM, this.GROUP, id);
    } catch (error) {
      this.logger.error(`Failed to process ${id}: ${error.message}`);
      // Không ACK → message sẽ ở PEL, xử lý lại khi restart
    }
  }
}
```

## 6. Activity Feed — Real-time

```typescript
// User activity feed
async recordActivity(userId: string, action: string, metadata: Record<string, string>) {
  const stream = `user:${userId}:activity`;
  await this.streamsService.addToStream(stream, {
    action,
    ...metadata,
    timestamp: Date.now().toString(),
  });

  // Chỉ giữ 1000 entries gần nhất
  await this.streamsService.trimStream(stream, 1000);
}

// Đọc activity feed (paginate với ID)
async getActivity(userId: string, before?: string, limit = 20) {
  const stream = `user:${userId}:activity`;
  const end = before ?? '+';    // '+' = từ mới nhất
  const start = '-';             // '-' = đến cũ nhất

  const entries = await this.redis.xrevrange(stream, end, start, 'COUNT', limit);
  return entries.map(([id, fields]) => ({ id, ...this.parseFields(fields) }));
}
```

## 7. Kết luận

- **`XADD`**: Thêm message, ID tự động có timestamp → dễ query theo thời gian
- **Consumer Groups**: Nhiều service đều nhận message — mỗi group nhận một lần
- **ACK**: Chỉ ACK sau khi xử lý thành công — không mất message khi crash
- **Pending**: Xử lý lại khi restart — đọc PEL trước khi consume mới
- **`MAXLEN`**: Tự động trim stream — tránh bộ nhớ phình to

Redis Streams là middle ground tốt — dễ setup hơn Kafka, reliable hơn Pub/Sub. Phù hợp khi đã có Redis và cần event streaming nhẹ.
