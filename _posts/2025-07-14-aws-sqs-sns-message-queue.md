---
layout: article
title: AWS SQS & SNS – Message Queue và Fan-out Pattern
tags: [aws, sqs, sns, queue, microservices, event-driven]
---
AWS SQS (Simple Queue Service) và SNS (Simple Notification Service) là hai dịch vụ messaging quan trọng trên AWS. SQS là hàng đợi message, SNS là pub/sub system — kết hợp cả hai tạo ra kiến trúc event-driven mạnh mẽ.

## 1. SQS vs SNS

| | SQS | SNS |
|--|-----|-----|
| **Mô hình** | Queue (1 producer, 1 consumer) | Pub/Sub (1 publisher, N subscribers) |
| **Delivery** | Consumer tự poll message | Push đến subscribers |
| **Persistence** | Có (up to 14 ngày) | Không (fire & forget) |
| **Use case** | Task queue, decouple services | Broadcast event, fan-out |

## 2. Cài đặt SDK

```bash
npm install --save @aws-sdk/client-sqs @aws-sdk/client-sns
```

## 3. SQS — Gửi và nhận message

### Cấu hình client

```typescript
import { SQSClient, SendMessageCommand, ReceiveMessageCommand, DeleteMessageCommand } from '@aws-sdk/client-sqs';

const sqsClient = new SQSClient({
  region: process.env.AWS_REGION ?? 'ap-southeast-1',
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID!,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY!,
  },
});

const QUEUE_URL = process.env.SQS_QUEUE_URL!;
```

### Gửi message

```typescript
async function sendMessage(body: object, groupId?: string) {
  await sqsClient.send(new SendMessageCommand({
    QueueUrl: QUEUE_URL,
    MessageBody: JSON.stringify(body),
    // Delay xử lý (optional)
    DelaySeconds: 0,
    // Message attributes (metadata)
    MessageAttributes: {
      eventType: { DataType: 'String', StringValue: 'order.created' },
      version: { DataType: 'String', StringValue: '1.0' },
    },
    // Chỉ dùng cho FIFO Queue
    // MessageGroupId: groupId,
    // MessageDeduplicationId: crypto.randomUUID(),
  }));
}

// Gửi nhiều message cùng lúc (tối đa 10)
async function sendBatch(messages: object[]) {
  const { SendMessageBatchCommand } = await import('@aws-sdk/client-sqs');
  await sqsClient.send(new SendMessageBatchCommand({
    QueueUrl: QUEUE_URL,
    Entries: messages.map((msg, i) => ({
      Id: String(i),
      MessageBody: JSON.stringify(msg),
    })),
  }));
}
```

### Poll và xử lý message

```typescript
async function pollMessages() {
  while (true) {
    const result = await sqsClient.send(new ReceiveMessageCommand({
      QueueUrl: QUEUE_URL,
      MaxNumberOfMessages: 10,       // Tối đa 10 messages mỗi lần
      WaitTimeSeconds: 20,           // Long polling — chờ đến 20s nếu không có message
      VisibilityTimeout: 60,         // Ẩn message 60s trong lúc xử lý (tránh consumer khác nhận)
      MessageAttributeNames: ['All'],
    }));

    const messages = result.Messages ?? [];
    if (messages.length === 0) continue;

    await Promise.all(messages.map(async (msg) => {
      try {
        const body = JSON.parse(msg.Body!);
        await processMessage(body);

        // Xóa message sau khi xử lý thành công
        await sqsClient.send(new DeleteMessageCommand({
          QueueUrl: QUEUE_URL,
          ReceiptHandle: msg.ReceiptHandle!,
        }));
      } catch (err) {
        console.error('Failed to process message, will retry:', err);
        // KHÔNG xóa → sau khi VisibilityTimeout message sẽ reappear
        // Sau N lần thất bại → chuyển vào Dead Letter Queue (DLQ)
      }
    }));
  }
}
```

### Tích hợp vào NestJS

```typescript
@Injectable()
export class SqsConsumerService implements OnModuleInit {
  private running = false;

  async onModuleInit() {
    this.running = true;
    this.startPolling();
  }

  private async startPolling() {
    while (this.running) {
      await pollMessages();
    }
  }

  async onModuleDestroy() {
    this.running = false;
  }
}
```

## 4. SNS — Publish/Subscribe

### Publish event

```typescript
import { SNSClient, PublishCommand } from '@aws-sdk/client-sns';

const snsClient = new SNSClient({ region: process.env.AWS_REGION });
const TOPIC_ARN = process.env.SNS_TOPIC_ARN!;

async function publishEvent(eventType: string, payload: object) {
  await snsClient.send(new PublishCommand({
    TopicArn: TOPIC_ARN,
    Message: JSON.stringify(payload),
    Subject: eventType,
    MessageAttributes: {
      eventType: { DataType: 'String', StringValue: eventType },
    },
  }));
}

// Publish khi có event
await publishEvent('order.placed', {
  orderId: '123',
  userId: 'abc',
  total: 500000,
});
```

## 5. Fan-out Pattern: SNS → SQS

Đây là pattern phổ biến nhất trên AWS:

```
[Producer]
     ↓ publish
   [SNS Topic "order-events"]
     ↓              ↓              ↓
[SQS: email]  [SQS: inventory]  [SQS: analytics]
     ↓              ↓              ↓
[Email      [Inventory      [Analytics
 Lambda]     Service]        Service]
```

**Lợi ích**:
- Producer chỉ cần publish một lần lên SNS
- Thêm consumer mới không cần sửa producer
- Mỗi queue có thể có retry policy, DLQ riêng

### Cấu hình trong AWS Console / CloudFormation

```yaml
# CloudFormation template
Resources:
  OrderEventsTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: order-events

  EmailQueue:
    Type: AWS::SQS::Queue
    Properties:
      VisibilityTimeout: 60
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt EmailDLQ.Arn
        maxReceiveCount: 3

  EmailDLQ:
    Type: AWS::SQS::Queue

  EmailSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      TopicArn: !Ref OrderEventsTopic
      Protocol: sqs
      Endpoint: !GetAtt EmailQueue.Arn
      # Filter chỉ nhận một số loại event
      FilterPolicy:
        eventType:
          - order.placed
          - order.shipped
```

## 6. Dead Letter Queue (DLQ)

Message xử lý thất bại N lần sẽ chuyển vào DLQ để điều tra:

```typescript
// Monitor DLQ
async function checkDLQ() {
  const result = await sqsClient.send(new ReceiveMessageCommand({
    QueueUrl: process.env.DLQ_URL!,
    MaxNumberOfMessages: 10,
  }));

  for (const msg of result.Messages ?? []) {
    console.error('Dead letter message:', JSON.parse(msg.Body!));
    // Alert, gửi email cảnh báo, lưu vào DB để xử lý thủ công...
  }
}
```

## 7. Kết luận

SQS và SNS là backbone của kiến trúc event-driven trên AWS:

- **SQS**: Hàng đợi message, đảm bảo at-least-once delivery, retry tự động
- **SNS**: Broadcast event đến nhiều subscriber cùng lúc
- **Fan-out**: SNS → nhiều SQS queue — pattern phổ biến nhất
- **DLQ**: Giữ lại message thất bại để điều tra và xử lý lại

Kết hợp SNS + SQS + Lambda tạo ra hệ thống event-driven hoàn toàn serverless, tự động scale.
