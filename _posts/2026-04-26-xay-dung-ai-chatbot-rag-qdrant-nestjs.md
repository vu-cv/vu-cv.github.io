---
layout: article
title: Xây dựng AI Chatbot với RAG + Qdrant + NestJS
tags: [ai, chatbot, rag, qdrant, nestjs, openai]
---
Bài này là phần tổng hợp — chúng ta sẽ xây dựng một AI chatbot hoàn chỉnh có khả năng trả lời câu hỏi dựa trên tài liệu nội bộ (knowledge base). Stack sử dụng: **NestJS + Qdrant + OpenAI**.

## 1. Kiến trúc tổng quan

```
[User hỏi]
    ↓
[NestJS API]
    ├── [Embedding Service] → embed câu hỏi (OpenAI)
    ├── [Qdrant Service]    → tìm top-5 chunks liên quan
    └── [Chat Service]      → gọi GPT-4o với context → trả lời
```

Hai luồng chính:
- **Indexing**: Upload tài liệu → chunk → embed → lưu Qdrant
- **Chat**: User hỏi → embed câu hỏi → tìm Qdrant → LLM trả lời

## 2. Cấu trúc project

```
src/
├── chat/
│   ├── chat.module.ts
│   ├── chat.controller.ts
│   └── chat.service.ts
├── documents/
│   ├── documents.module.ts
│   ├── documents.controller.ts
│   └── documents.service.ts
├── qdrant/
│   ├── qdrant.module.ts
│   └── qdrant.service.ts
├── openai/
│   ├── openai.module.ts
│   └── openai.service.ts
└── app.module.ts
```

## 3. OpenAI Service

`src/openai/openai.service.ts`:

```typescript
import { Injectable } from '@nestjs/common';
import OpenAI from 'openai';

@Injectable()
export class OpenAIService {
  private readonly client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

  async createEmbedding(text: string): Promise<number[]> {
    const res = await this.client.embeddings.create({
      model: 'text-embedding-3-small',
      input: text,
    });
    return res.data[0].embedding;
  }

  async createEmbeddings(texts: string[]): Promise<number[][]> {
    const res = await this.client.embeddings.create({
      model: 'text-embedding-3-small',
      input: texts,
    });
    return res.data.map(d => d.embedding);
  }

  async chat(
    systemPrompt: string,
    userMessage: string,
    history: Array<{ role: 'user' | 'assistant'; content: string }> = [],
  ): Promise<string> {
    const messages: OpenAI.ChatCompletionMessageParam[] = [
      { role: 'system', content: systemPrompt },
      ...history,
      { role: 'user', content: userMessage },
    ];

    const res = await this.client.chat.completions.create({
      model: 'gpt-4o-mini',
      messages,
      temperature: 0.3, // Thấp hơn = nhất quán hơn, ít sáng tạo hơn
    });

    return res.choices[0].message.content ?? '';
  }
}
```

## 4. Qdrant Service

`src/qdrant/qdrant.service.ts`:

```typescript
import { Injectable, OnModuleInit } from '@nestjs/common';
import { QdrantClient } from '@qdrant/js-client-rest';
import { v4 as uuidv4 } from 'uuid';

export interface SearchResult {
  text: string;
  source: string;
  score: number;
}

@Injectable()
export class QdrantService implements OnModuleInit {
  private client: QdrantClient;
  private readonly COLLECTION = 'knowledge_base';
  private readonly VECTOR_SIZE = 1536;

  async onModuleInit() {
    this.client = new QdrantClient({ url: process.env.QDRANT_URL ?? 'http://localhost:6333' });
    await this.ensureCollection();
  }

  private async ensureCollection() {
    const { collections } = await this.client.getCollections();
    if (!collections.some(c => c.name === this.COLLECTION)) {
      await this.client.createCollection(this.COLLECTION, {
        vectors: { size: this.VECTOR_SIZE, distance: 'Cosine' },
      });
    }
  }

  async upsertChunks(
    chunks: Array<{ text: string; source: string; vector: number[] }>,
  ) {
    const points = chunks.map(chunk => ({
      id: uuidv4(),
      vector: chunk.vector,
      payload: { text: chunk.text, source: chunk.source },
    }));
    await this.client.upsert(this.COLLECTION, { wait: true, points });
  }

  async search(vector: number[], topK = 5): Promise<SearchResult[]> {
    const results = await this.client.search(this.COLLECTION, {
      vector,
      limit: topK,
      with_payload: true,
    });

    return results.map(r => ({
      text: (r.payload as any).text,
      source: (r.payload as any).source,
      score: r.score,
    }));
  }

  async deleteBySource(source: string) {
    await this.client.delete(this.COLLECTION, {
      filter: { must: [{ key: 'source', match: { value: source } }] },
    });
  }
}
```

## 5. Documents Service — Indexing Pipeline

`src/documents/documents.service.ts`:

```typescript
import { Injectable } from '@nestjs/common';
import { OpenAIService } from '../openai/openai.service';
import { QdrantService } from '../qdrant/qdrant.service';

@Injectable()
export class DocumentsService {
  constructor(
    private readonly openai: OpenAIService,
    private readonly qdrant: QdrantService,
  ) {}

  // Chia text thành chunks có overlap
  private chunkText(text: string, chunkSize = 500, overlap = 100): string[] {
    const words = text.split(/\s+/);
    const chunks: string[] = [];

    for (let i = 0; i < words.length; i += chunkSize - overlap) {
      const chunk = words.slice(i, i + chunkSize).join(' ');
      if (chunk.trim()) chunks.push(chunk);
    }
    return chunks;
  }

  async indexDocument(text: string, source: string): Promise<number> {
    // 1. Chunk
    const chunks = this.chunkText(text);

    // 2. Embed tất cả chunks (batch)
    const vectors = await this.openai.createEmbeddings(chunks);

    // 3. Lưu vào Qdrant
    await this.qdrant.upsertChunks(
      chunks.map((text, i) => ({ text, source, vector: vectors[i] })),
    );

    return chunks.length;
  }

  async removeDocument(source: string) {
    await this.qdrant.deleteBySource(source);
  }
}
```

`src/documents/documents.controller.ts`:

```typescript
import { Controller, Post, Delete, Param, Body } from '@nestjs/common';
import { DocumentsService } from './documents.service';

@Controller('documents')
export class DocumentsController {
  constructor(private readonly documentsService: DocumentsService) {}

  @Post('index')
  async index(@Body() body: { text: string; source: string }) {
    const count = await this.documentsService.indexDocument(body.text, body.source);
    return { chunksIndexed: count };
  }

  @Delete(':source')
  async remove(@Param('source') source: string) {
    await this.documentsService.removeDocument(source);
    return { success: true };
  }
}
```

## 6. Chat Service — RAG Pipeline

`src/chat/chat.service.ts`:

```typescript
import { Injectable } from '@nestjs/common';
import { OpenAIService } from '../openai/openai.service';
import { QdrantService } from '../qdrant/qdrant.service';

const SYSTEM_PROMPT = `Bạn là trợ lý AI hỗ trợ khách hàng.
Hãy trả lời dựa trên thông tin được cung cấp trong phần NGỮCẢNH.
Nếu không tìm thấy thông tin liên quan, hãy nói: "Tôi không có thông tin về vấn đề này, vui lòng liên hệ bộ phận hỗ trợ."
Trả lời bằng tiếng Việt, ngắn gọn và thân thiện.`;

@Injectable()
export class ChatService {
  constructor(
    private readonly openai: OpenAIService,
    private readonly qdrant: QdrantService,
  ) {}

  async chat(
    question: string,
    history: Array<{ role: 'user' | 'assistant'; content: string }> = [],
  ): Promise<{ answer: string; sources: string[] }> {
    // 1. Embed câu hỏi
    const questionVector = await this.openai.createEmbedding(question);

    // 2. Tìm top-5 chunks liên quan
    const relevantChunks = await this.qdrant.search(questionVector, 5);

    // 3. Lọc chỉ lấy chunks có score đủ cao
    const goodChunks = relevantChunks.filter(c => c.score > 0.7);

    // 4. Tạo prompt với ngữ cảnh
    const context = goodChunks.length > 0
      ? goodChunks.map((c, i) => `[${i + 1}] ${c.text}`).join('\n\n')
      : 'Không tìm thấy thông tin liên quan.';

    const systemWithContext = `${SYSTEM_PROMPT}\n\nNGỮCẢNH:\n${context}`;

    // 5. Gọi LLM
    const answer = await this.openai.chat(systemWithContext, question, history);

    const sources = [...new Set(goodChunks.map(c => c.source))];

    return { answer, sources };
  }
}
```

`src/chat/chat.controller.ts`:

```typescript
import { Controller, Post, Body } from '@nestjs/common';
import { ChatService } from './chat.service';

@Controller('chat')
export class ChatController {
  constructor(private readonly chatService: ChatService) {}

  @Post()
  async chat(
    @Body() body: {
      question: string;
      history?: Array<{ role: 'user' | 'assistant'; content: string }>;
    },
  ) {
    return this.chatService.chat(body.question, body.history);
  }
}
```

## 7. Test End-to-End

```bash
# 1. Index tài liệu
curl -X POST http://localhost:3000/documents/index \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Chính sách hoàn tiền: Khách hàng có thể yêu cầu hoàn tiền trong vòng 30 ngày kể từ ngày mua hàng nếu sản phẩm bị lỗi từ nhà sản xuất. Thời gian hoàn tiền từ 3-5 ngày làm việc.",
    "source": "policy.txt"
  }'

# 2. Chat
curl -X POST http://localhost:3000/chat \
  -H "Content-Type: application/json" \
  -d '{"question": "Tôi mua hàng 2 tuần trước, sản phẩm bị lỗi, có được hoàn tiền không?"}'
```

Response:
```json
{
  "answer": "Có, bạn vẫn đủ điều kiện hoàn tiền! Theo chính sách của chúng tôi, bạn có thể yêu cầu hoàn tiền trong vòng 30 ngày kể từ ngày mua nếu sản phẩm bị lỗi từ nhà sản xuất. Thời gian hoàn tiền sẽ mất 3-5 ngày làm việc.",
  "sources": ["policy.txt"]
}
```

## 8. Kết luận

Chúng ta đã xây dựng xong một RAG chatbot hoàn chỉnh:

- **Indexing**: Upload tài liệu → chunk → embed → lưu Qdrant
- **Chat**: Embed câu hỏi → tìm Qdrant → LLM sinh câu trả lời
- **Nguồn tài liệu** được trả về trong response để có thể cite

Đây là kiến trúc nền tảng có thể mở rộng thêm: streaming response, multi-turn conversation với memory, reranker, hybrid search...
