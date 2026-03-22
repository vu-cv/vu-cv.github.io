---
layout: article
title: Qdrant Vector Database – Cài đặt & Tìm kiếm ngữ nghĩa
tags: [qdrant, vector-db, ai, embedding, semantic-search]
---
Qdrant là vector database mã nguồn mở hiệu suất cao, được viết bằng Rust, hỗ trợ tìm kiếm ngữ nghĩa cực nhanh. Đây là một trong những lựa chọn tốt nhất để build RAG chatbot và search engine AI.

## 1. Tại sao Qdrant?

- **Hiệu suất cao**: Viết bằng Rust, rất nhanh và ít tốn RAM
- **Filtering phong phú**: Kết hợp vector search với filter theo metadata
- **Dễ dùng**: REST API và client SDK cho nhiều ngôn ngữ
- **Tự host**: Chạy được trên server của riêng bạn hoặc dùng Qdrant Cloud
- **Hỗ trợ nhiều distance metric**: Cosine, Dot Product, Euclidean

## 2. Cài đặt

### Dùng Docker (khuyến nghị)

```bash
docker pull qdrant/qdrant
docker run -p 6333:6333 -p 6334:6334 \
  -v $(pwd)/qdrant_storage:/qdrant/storage \
  qdrant/qdrant
```

- REST API: `http://localhost:6333`
- Dashboard: `http://localhost:6333/dashboard`
- gRPC: `localhost:6334`

### Dùng Docker Compose

```yaml
# docker-compose.yml
services:
  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"
      - "6334:6334"
    volumes:
      - ./qdrant_storage:/qdrant/storage
    environment:
      QDRANT__SERVICE__API_KEY: "your-api-key"  # Optional
```

## 3. Cài đặt Client SDK (Node.js)

```bash
npm install --save @qdrant/js-client-rest
```

## 4. Kết nối và tạo Collection

```typescript
import { QdrantClient } from '@qdrant/js-client-rest';

const client = new QdrantClient({ url: 'http://localhost:6333' });

// Tạo collection với vector 1536 chiều (OpenAI text-embedding-3-small)
await client.createCollection('documents', {
  vectors: {
    size: 1536,
    distance: 'Cosine',
  },
});

console.log('Collection created!');
```

## 5. Insert Points (vectors)

Mỗi "Point" trong Qdrant gồm: `id`, `vector`, và `payload` (metadata tùy ý):

```typescript
import { v4 as uuidv4 } from 'uuid';
import OpenAI from 'openai';

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

async function embedText(text: string): Promise<number[]> {
  const res = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: text,
  });
  return res.data[0].embedding;
}

async function insertDocuments(docs: Array<{ text: string; source: string }>) {
  const points = await Promise.all(
    docs.map(async (doc) => ({
      id: uuidv4(),
      vector: await embedText(doc.text),
      payload: {
        text: doc.text,
        source: doc.source,
        createdAt: new Date().toISOString(),
      },
    })),
  );

  await client.upsert('documents', {
    wait: true, // Đợi cho đến khi index xong
    points,
  });

  console.log(`Inserted ${points.length} points`);
}

// Sử dụng
await insertDocuments([
  { text: 'Chính sách hoàn tiền trong vòng 30 ngày kể từ ngày mua hàng', source: 'policy.pdf' },
  { text: 'Hướng dẫn đổi trả sản phẩm bị lỗi từ nhà sản xuất', source: 'policy.pdf' },
  { text: 'Thời gian giao hàng từ 2-5 ngày làm việc', source: 'faq.pdf' },
]);
```

## 6. Tìm kiếm ngữ nghĩa

```typescript
async function search(query: string, topK = 5) {
  const queryVector = await embedText(query);

  const results = await client.search('documents', {
    vector: queryVector,
    limit: topK,
    with_payload: true, // Trả về payload (metadata)
  });

  return results.map(r => ({
    score: r.score,
    text: (r.payload as any).text,
    source: (r.payload as any).source,
  }));
}

// Sử dụng
const results = await search('Tôi muốn trả lại hàng');
results.forEach(r => {
  console.log(`Score: ${r.score.toFixed(3)} | ${r.text}`);
});
// Score: 0.891 | Chính sách hoàn tiền trong vòng 30 ngày...
// Score: 0.847 | Hướng dẫn đổi trả sản phẩm bị lỗi...
```

## 7. Tìm kiếm với Filter (Hybrid)

Kết hợp vector search với filter theo metadata — tính năng cực mạnh của Qdrant:

```typescript
// Chỉ tìm trong tài liệu có source = 'policy.pdf'
const results = await client.search('documents', {
  vector: queryVector,
  limit: 5,
  with_payload: true,
  filter: {
    must: [
      { key: 'source', match: { value: 'policy.pdf' } },
    ],
  },
});

// Filter phức tạp hơn
const results2 = await client.search('documents', {
  vector: queryVector,
  limit: 5,
  filter: {
    must: [
      { key: 'category', match: { any: ['support', 'policy'] } },
      { key: 'language', match: { value: 'vi' } },
    ],
    must_not: [
      { key: 'isArchived', match: { value: true } },
    ],
  },
});
```

## 8. Quản lý Collection

```typescript
// Liệt kê tất cả collections
const collections = await client.getCollections();
console.log(collections.collections.map(c => c.name));

// Xem thông tin collection
const info = await client.getCollection('documents');
console.log(info.points_count); // Số lượng points

// Xóa points theo filter
await client.delete('documents', {
  filter: {
    must: [{ key: 'source', match: { value: 'old-doc.pdf' } }],
  },
});

// Xóa collection
await client.deleteCollection('documents');
```

## 9. Tích hợp vào NestJS Service

```typescript
import { Injectable, OnModuleInit } from '@nestjs/common';
import { QdrantClient } from '@qdrant/js-client-rest';

@Injectable()
export class QdrantService implements OnModuleInit {
  private client: QdrantClient;
  private readonly COLLECTION = 'documents';

  async onModuleInit() {
    this.client = new QdrantClient({ url: process.env.QDRANT_URL! });
    await this.ensureCollection();
  }

  private async ensureCollection() {
    const collections = await this.client.getCollections();
    const exists = collections.collections.some(c => c.name === this.COLLECTION);
    if (!exists) {
      await this.client.createCollection(this.COLLECTION, {
        vectors: { size: 1536, distance: 'Cosine' },
      });
    }
  }

  async upsert(points: Array<{ id: string; vector: number[]; payload: object }>) {
    await this.client.upsert(this.COLLECTION, { wait: true, points });
  }

  async search(vector: number[], topK = 5, filter?: object) {
    return this.client.search(this.COLLECTION, {
      vector,
      limit: topK,
      with_payload: true,
      ...(filter && { filter }),
    });
  }
}
```

## 10. Qdrant Cloud

Nếu không muốn tự host, dùng Qdrant Cloud (có free tier):

```typescript
const client = new QdrantClient({
  url: 'https://xxxx.us-east4-0.gcp.cloud.qdrant.io',
  apiKey: 'your-api-key',
});
```

## 11. Kết luận

Qdrant là vector database lý tưởng cho RAG và semantic search:

- **Collection** = database table, **Point** = row (id + vector + payload)
- `upsert` để insert/update, `search` để tìm kiếm theo vector
- **Filter** = kết hợp vector similarity với điều kiện metadata — rất mạnh
- Chạy tốt với Docker, dễ scale lên Qdrant Cloud

Bài tiếp theo: **Xây dựng AI Chatbot hoàn chỉnh với RAG + Qdrant + NestJS**.
