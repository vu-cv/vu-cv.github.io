---
layout: article
title: Kafka – Message Broker cho hệ thống phân tán với NodeJS
tags: [kafka, message-broker, nodejs, nestjs, microservices, event-streaming]
---
Apache Kafka là distributed event streaming platform — xử lý hàng triệu message/giây, đảm bảo thứ tự, replay được message cũ. Phù hợp cho microservices lớn, real-time analytics, event sourcing.

## 1. Kafka vs Redis Pub/Sub vs RabbitMQ

| Tiêu chí | Kafka | Redis Pub/Sub | RabbitMQ |
|---------|-------|--------------|---------|
| Throughput | Rất cao (1M+/s) | Cao (100k+/s) | Trung bình |
| Persistence | Có (log-based) | Không | Có |
| Replay | Có (offset) | Không | Không |
| Consumer Groups | Có | Không | Có |
| Ordering | Trong partition | Không đảm bảo | FIFO |
| Use case | Event streaming, analytics | Realtime notification | Task queue |

## 2. Cài đặt với Docker

```yaml
# docker-compose.yml
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on: [zookeeper]
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'true'
      KAFKA_DEFAULT_REPLICATION_FACTOR: 1
      KAFKA_NUM_PARTITIONS: 3

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
```

## 3. NestJS + KafkaJS

```bash
npm install kafkajs
```

```typescript
// src/kafka/kafka.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy, Logger } from '@nestjs/common';
import { Kafka, Producer, Consumer, Admin } from 'kafkajs';

@Injectable()
export class KafkaService implements OnModuleInit, OnModuleDestroy {
  private readonly logger = new Logger(KafkaService.name);
  private kafka: Kafka;
  private producer: Producer;
  private consumers: Map<string, Consumer> = new Map();

  constructor() {
    this.kafka = new Kafka({
      clientId: 'shopxyz-app',
      brokers: (process.env.KAFKA_BROKERS ?? 'localhost:9092').split(','),
      retry: { initialRetryTime: 100, retries: 8 },
    });
    this.producer = this.kafka.producer({
      allowAutoTopicCreation: true,
    });
  }

  async onModuleInit() {
    await this.producer.connect();
    this.logger.log('Kafka producer connected');
  }

  async onModuleDestroy() {
    await this.producer.disconnect();
    for (const consumer of this.consumers.values()) {
      await consumer.disconnect();
    }
  }

  // Gửi message
  async send(topic: string, messages: Array<{ key?: string; value: any }>): Promise<void> {
    await this.producer.send({
      topic,
      messages: messages.map(m => ({
        key: m.key,
        value: JSON.stringify(m.value),
      })),
    });
  }

  // Subscribe và consume messages
  async subscribe(
    topic: string,
    groupId: string,
    handler: (message: any, key?: string) => Promise<void>,
  ): Promise<void> {
    const consumer = this.kafka.consumer({ groupId });
    await consumer.connect();
    await consumer.subscribe({ topic, fromBeginning: false });

    await consumer.run({
      eachMessage: async ({ message, partition, offset }) => {
        try {
          const value = JSON.parse(message.value!.toString());
          const key = message.key?.toString();
          await handler(value, key);
          this.logger.debug(`Processed: topic=${topic} partition=${partition} offset=${offset}`);
        } catch (error) {
          this.logger.error(`Failed to process message: ${error.message}`);
          // Gửi vào DLQ (Dead Letter Queue)
          await this.send(`${topic}.DLQ`, [{
            value: { error: error.message, originalMessage: message.value!.toString() },
          }]);
        }
      },
    });

    this.consumers.set(`${groupId}-${topic}`, consumer);
  }
}
```

## 4. Topics và Producers

```typescript
// src/events/kafka-events.service.ts
export const TOPICS = {
  USER_CREATED: 'user.created',
  ORDER_PLACED: 'order.placed',
  ORDER_SHIPPED: 'order.shipped',
  PAYMENT_COMPLETED: 'payment.completed',
  INVENTORY_UPDATED: 'inventory.updated',
} as const;

@Injectable()
export class KafkaEventsService {
  constructor(private kafkaService: KafkaService) {}

  async publishUserCreated(user: { id: string; email: string; name: string }) {
    await this.kafkaService.send(TOPICS.USER_CREATED, [{
      key: user.id,          // Key quyết định partition → cùng user luôn cùng partition → đảm bảo thứ tự
      value: {
        userId: user.id,
        email: user.email,
        name: user.name,
        timestamp: new Date().toISOString(),
      },
    }]);
  }

  async publishOrderPlaced(order: {
    orderId: string;
    userId: string;
    items: any[];
    total: number;
  }) {
    await this.kafkaService.send(TOPICS.ORDER_PLACED, [{
      key: order.userId,
      value: { ...order, timestamp: new Date().toISOString() },
    }]);
  }
}
```

## 5. Consumers — Xử lý Events

```typescript
// src/notifications/notification-consumer.service.ts
@Injectable()
export class NotificationConsumerService implements OnModuleInit {
  constructor(
    private kafkaService: KafkaService,
    private mailService: MailService,
  ) {}

  async onModuleInit() {
    // Consumer group 'notifications' nhận order events
    await this.kafkaService.subscribe(
      TOPICS.ORDER_PLACED,
      'notifications-service',
      async (message) => {
        await this.mailService.sendOrderConfirmation(message.userId, message.orderId);
      },
    );

    await this.kafkaService.subscribe(
      TOPICS.USER_CREATED,
      'notifications-service',
      async (message) => {
        await this.mailService.sendWelcomeEmail(message.email, message.name);
      },
    );
  }
}

// src/inventory/inventory-consumer.service.ts
@Injectable()
export class InventoryConsumerService implements OnModuleInit {
  async onModuleInit() {
    // Consumer group khác → nhận cùng message độc lập
    await this.kafkaService.subscribe(
      TOPICS.ORDER_PLACED,
      'inventory-service',  // Khác group → mỗi group đều nhận message
      async (message) => {
        for (const item of message.items) {
          await this.stockService.decrement(item.productId, item.quantity);
        }
      },
    );
  }
}
```

## 6. NestJS Kafka Transport (built-in)

```typescript
// main.ts — Microservice mode
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      brokers: ['localhost:9092'],
    },
    consumer: {
      groupId: 'my-group',
    },
  },
});

// Controller
@Controller()
export class EventsController {
  @MessagePattern('user.created')
  handleUserCreated(@Payload() data: any) {
    console.log('User created:', data);
  }

  @EventPattern('order.placed')
  handleOrderPlaced(@Payload() data: any) {
    console.log('Order placed:', data);
  }
}
```

## 7. Kết luận

- **Consumer Groups**: Nhiều service khác nhau đều nhận cùng 1 message (fan-out)
- **Partitions**: Cùng key → cùng partition → đảm bảo thứ tự per-key
- **Persistence**: Message được lưu trên disk (mặc định 7 ngày) — có thể replay
- **DLQ**: Gửi message lỗi vào topic `.DLQ` để xử lý sau hoặc debug
- **KafkaJS vs NestJS built-in**: KafkaJS linh hoạt hơn, NestJS transport đơn giản hơn

Kafka phù hợp khi: cần throughput cao, cần replay events, nhiều consumer group. Redis đủ tốt cho hệ thống nhỏ hơn.
