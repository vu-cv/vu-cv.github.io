---
layout: article
title: AWS Lambda – Serverless Functions với NodeJS
tags: [aws, lambda, serverless, nodejs, api-gateway]
---
AWS Lambda cho phép chạy code mà không cần quản lý server — bạn chỉ trả tiền khi code thực sự chạy. Kết hợp với API Gateway, Lambda trở thành nền tảng lý tưởng cho serverless API và background processing.

## 1. Lambda là gì?

- **Event-driven**: Lambda chạy khi có event (HTTP request, S3 upload, SQS message, schedule...)
- **Auto-scaling**: Tự động scale từ 0 đến hàng nghìn concurrent executions
- **Pay per use**: Tính tiền theo số lần gọi và thời gian thực thi (miễn phí 1M requests/tháng)
- **Stateless**: Mỗi lần gọi là một instance độc lập

## 2. Cấu trúc Lambda Function

```javascript
// handler.js — CommonJS (Node.js 18/20)
exports.handler = async (event, context) => {
  console.log('Event:', JSON.stringify(event));

  try {
    // Xử lý logic
    const result = await processEvent(event);

    return {
      statusCode: 200,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*',
      },
      body: JSON.stringify({ success: true, data: result }),
    };
  } catch (error) {
    return {
      statusCode: 500,
      body: JSON.stringify({ message: error.message }),
    };
  }
};
```

```typescript
// handler.ts — TypeScript với ESM
import { APIGatewayProxyEvent, APIGatewayProxyResult, Context } from 'aws-lambda';

export const handler = async (
  event: APIGatewayProxyEvent,
  context: Context,
): Promise<APIGatewayProxyResult> => {
  const body = event.body ? JSON.parse(event.body) : {};
  const pathParams = event.pathParameters;
  const queryParams = event.queryStringParameters;

  return {
    statusCode: 200,
    body: JSON.stringify({ body, pathParams, queryParams }),
  };
};
```

## 3. Deploy với AWS SAM

AWS SAM (Serverless Application Model) là cách đơn giản nhất để định nghĩa và deploy Lambda.

### Cài đặt

```bash
brew install aws-sam-cli
```

### template.yaml

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Runtime: nodejs20.x
    Timeout: 30
    MemorySize: 512
    Environment:
      Variables:
        MONGODB_URI: !Ref MongodbUri
        NODE_ENV: production

Parameters:
  MongodbUri:
    Type: String
    NoEcho: true

Resources:
  # REST API
  ApiFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: dist/handler.handler
      Events:
        GetUsers:
          Type: Api
          Properties:
            Path: /users
            Method: GET
        CreateUser:
          Type: Api
          Properties:
            Path: /users
            Method: POST
        GetUser:
          Type: Api
          Properties:
            Path: /users/{id}
            Method: GET

  # Scheduled job — chạy mỗi ngày lúc 8 giờ sáng
  DailyReportFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: dist/report.handler
      Events:
        Schedule:
          Type: Schedule
          Properties:
            Schedule: cron(0 1 * * ? *)  # 8:00 AM GMT+7

  # Trigger từ S3
  ImageProcessFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: dist/image.handler
      Policies:
        - S3ReadPolicy:
            BucketName: !Ref ImageBucket
      Events:
        S3Event:
          Type: S3
          Properties:
            Bucket: !Ref ImageBucket
            Events: s3:ObjectCreated:*
            Filter:
              S3Key:
                Rules:
                  - Name: suffix
                    Value: '.jpg'

  ImageBucket:
    Type: AWS::S3::Bucket
```

### Deploy

```bash
# Build TypeScript
npm run build

# Test local
sam local invoke ApiFunction --event events/get-users.json
sam local start-api  # Mock API Gateway local

# Deploy lên AWS
sam deploy --guided
```

## 4. Xử lý SQS Message

```typescript
import { SQSEvent, SQSRecord } from 'aws-lambda';

export const handler = async (event: SQSEvent) => {
  const results = await Promise.allSettled(
    event.Records.map(record => processRecord(record))
  );

  // Trả về các message thất bại để SQS retry
  const failures = results
    .map((result, i) => ({ result, record: event.Records[i] }))
    .filter(({ result }) => result.status === 'rejected')
    .map(({ record }) => ({ itemIdentifier: record.messageId }));

  return { batchItemFailures: failures };
};

async function processRecord(record: SQSRecord) {
  const message = JSON.parse(record.body);
  console.log('Processing:', message);
  // Xử lý message...
}
```

## 5. Kết nối MongoDB hiệu quả

Vấn đề lớn nhất khi dùng Lambda với database là **cold start** và **connection pooling**. Giải pháp: reuse connection giữa các invocations trong cùng một container:

```typescript
import mongoose from 'mongoose';

// Connection được giữ ngoài handler function
let cachedConnection: typeof mongoose | null = null;

async function connectDB() {
  if (cachedConnection && mongoose.connection.readyState === 1) {
    return cachedConnection;
  }

  cachedConnection = await mongoose.connect(process.env.MONGODB_URI!, {
    maxPoolSize: 5, // Giới hạn pool nhỏ vì Lambda có nhiều instances
    serverSelectionTimeoutMS: 5000,
  });

  return cachedConnection;
}

export const handler = async (event: any) => {
  await connectDB(); // Sẽ reuse connection nếu container còn sống

  // Xử lý request...
};
```

## 6. Lambda Layers

Chia sẻ dependencies giữa nhiều functions:

```yaml
# template.yaml
Globals:
  Function:
    Layers:
      - !Ref CommonLayer

Resources:
  CommonLayer:
    Type: AWS::Serverless::LayerVersion
    Properties:
      LayerName: common-dependencies
      ContentUri: layers/common/
      CompatibleRuntimes:
        - nodejs20.x
    Metadata:
      BuildMethod: nodejs20.x
```

## 7. Monitoring với CloudWatch

```typescript
import { Logger } from '@aws-lambda-powertools/logger';
import { Metrics, MetricUnits } from '@aws-lambda-powertools/metrics';

const logger = new Logger({ serviceName: 'my-api' });
const metrics = new Metrics({ namespace: 'MyApp' });

export const handler = async (event: any) => {
  logger.info('Processing request', { path: event.path });

  metrics.addMetric('RequestCount', MetricUnits.Count, 1);
  metrics.publishStoredMetrics();
};
```

## 8. Kết luận

AWS Lambda phù hợp khi:

- **Traffic không đều**: Scale về 0 khi không có request, tránh lãng phí
- **Background processing**: Xử lý ảnh, email, webhook sau S3/SQS event
- **Scheduled jobs**: Thay thế cron server
- **Microservices nhỏ**: Mỗi function là một endpoint độc lập

**Không phù hợp** khi: cần kết nối database persistent, WebSocket, xử lý > 15 phút, hay cần file system.
