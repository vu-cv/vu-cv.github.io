---
layout: article
title: Azure Functions – Serverless với NodeJS
tags: [azure, functions, serverless, nodejs, cloud]
---
Azure Functions là dịch vụ serverless của Microsoft Azure, tương tự AWS Lambda. Bài này hướng dẫn xây dựng và deploy Azure Functions với NodeJS/TypeScript cho các tác vụ phổ biến: HTTP trigger, Timer trigger, và Queue trigger.

## 1. Cài đặt

```bash
npm install -g azure-functions-core-tools@4
npm install -g @azure/functions
```

Tạo project mới:
```bash
func init my-functions --typescript
cd my-functions
```

## 2. HTTP Trigger — REST API

```bash
func new --name HttpUsers --template "HTTP trigger"
```

```typescript
// src/functions/HttpUsers.ts
import { app, HttpRequest, HttpResponseInit, InvocationContext } from '@azure/functions';

export async function httpUsers(
  request: HttpRequest,
  context: InvocationContext,
): Promise<HttpResponseInit> {
  context.log(`HTTP function processed request for url "${request.url}"`);

  const method = request.method;

  if (method === 'GET') {
    const id = request.query.get('id');
    if (id) {
      // Query database...
      return { status: 200, jsonBody: { id, name: 'Nguyen Van A', email: 'a@example.com' } };
    }
    return { status: 200, jsonBody: { users: [] } };
  }

  if (method === 'POST') {
    const body = await request.json() as { name: string; email: string };
    // Create user...
    return { status: 201, jsonBody: { id: 'new-id', ...body } };
  }

  return { status: 405, body: 'Method not allowed' };
}

app.http('HttpUsers', {
  methods: ['GET', 'POST'],
  authLevel: 'anonymous',
  handler: httpUsers,
});
```

## 3. Timer Trigger — Scheduled Job

```typescript
// src/functions/DailyReport.ts
import { app, InvocationContext, Timer } from '@azure/functions';

export async function dailyReport(
  timer: Timer,
  context: InvocationContext,
): Promise<void> {
  const now = new Date().toISOString();
  context.log(`Daily report function ran at: ${now}`);

  if (timer.isPastDue) {
    context.log('Timer is running late!');
  }

  // Tạo báo cáo hàng ngày...
  // await generateAndSendReport();
}

app.timer('DailyReport', {
  // Chạy mỗi ngày lúc 8:00 AM (UTC+7 = 1:00 AM UTC)
  schedule: '0 0 1 * * *',
  handler: dailyReport,
});
```

## 4. Queue Trigger — Xử lý Azure Storage Queue

```typescript
// src/functions/ProcessEmail.ts
import { app, InvocationContext, StorageQueueHandler } from '@azure/functions';

const processEmail: StorageQueueHandler = async (
  queueItem: unknown,
  context: InvocationContext,
): Promise<void> => {
  context.log('Processing queue item:', queueItem);

  const { to, subject, body } = queueItem as { to: string; subject: string; body: string };

  // Gửi email...
  context.log(`Email sent to ${to}`);
};

app.storageQueue('ProcessEmail', {
  queueName: 'email-queue',
  connection: 'AzureStorageConnection',
  handler: processEmail,
});
```

## 5. Blob Trigger — Xử lý khi có file mới upload lên Azure Blob

```typescript
// src/functions/ProcessImage.ts
import { app, InvocationContext, StorageBlobHandler } from '@azure/functions';

const processImage: StorageBlobHandler = async (
  blob: unknown,
  context: InvocationContext,
): Promise<void> => {
  context.log('Blob trigger function processed blob:');
  context.log('Name:', context.triggerMetadata?.name);
  context.log('Size:', (blob as Buffer).length, 'bytes');

  // Resize, watermark, analyze ảnh...
};

app.storageBlob('ProcessImage', {
  path: 'uploads/{name}',    // Trigger khi có file mới trong container "uploads"
  connection: 'AzureStorageConnection',
  handler: processImage,
});
```

## 6. Kết nối Database

```typescript
import { MongoClient } from 'mongodb';

// Kết nối ngoài handler để reuse (tương tự Lambda)
let mongoClient: MongoClient | null = null;

async function getDb() {
  if (!mongoClient) {
    mongoClient = new MongoClient(process.env.MONGODB_URI!);
    await mongoClient.connect();
  }
  return mongoClient.db('mydb');
}

export async function httpUsers(request: HttpRequest, context: InvocationContext) {
  const db = await getDb();
  const users = await db.collection('users').find().toArray();
  return { jsonBody: users };
}
```

## 7. local.settings.json

```json
{
  "IsEncrypted": false,
  "Values": {
    "FUNCTIONS_WORKER_RUNTIME": "node",
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "AzureStorageConnection": "DefaultEndpointsProtocol=http;...",
    "MONGODB_URI": "mongodb://localhost:27017/mydb",
    "NODE_ENV": "development"
  }
}
```

## 8. Chạy local và Deploy

```bash
# Chạy local
func start

# Build TypeScript
npm run build

# Deploy lên Azure (cần Azure CLI đã login)
func azure functionapp publish my-function-app
```

### Deploy qua GitHub Actions

```yaml
- name: Deploy to Azure Functions
  uses: Azure/functions-action@v1
  with:
    app-name: my-function-app
    package: .
    publish-profile: ${{ secrets.AZURE_FUNCTIONAPP_PUBLISH_PROFILE }}
```

## 9. So sánh Azure Functions vs AWS Lambda

| Tiêu chí | Azure Functions | AWS Lambda |
|---------|----------------|------------|
| Free tier | 1M executions/tháng | 1M executions/tháng |
| Max timeout | 10 phút (Consumption) / 230 phút (Premium) | 15 phút |
| Cold start | Tương đương | Tương đương |
| Triggers | Nhiều (HTTP, Timer, Queue, Blob, Service Bus...) | Nhiều (API GW, SQS, S3, EventBridge...) |
| Tích hợp | Azure ecosystem | AWS ecosystem |
| Local dev | Azure Functions Core Tools | SAM CLI |

## 10. Kết luận

Azure Functions phù hợp khi:
- Hệ thống đang dùng Azure (Azure SQL, Cosmos DB, Blob Storage...)
- Cần tích hợp với Azure Service Bus, Event Grid
- Cần timer trigger phức tạp hoặc timeout dài hơn Lambda

Với NodeJS/TypeScript, DX của Azure Functions rất tốt — code giống NestJS controller, dễ học.
