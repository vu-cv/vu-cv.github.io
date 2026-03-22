---
layout: article
title: AI Streaming Response với SSE trong NestJS & NextJS
tags: [ai, streaming, sse, nestjs, nextjs, openai, realtime]
---
Thay vì đợi LLM trả về toàn bộ câu trả lời (có thể mất 5-30 giây), **Streaming** cho phép hiển thị từng từ ngay khi LLM tạo ra — trải nghiệm giống ChatGPT. Bài này hướng dẫn implement streaming end-to-end từ OpenAI → NestJS → NextJS.

## 1. Streaming hoạt động như thế nào?

```
OpenAI API ──stream──→ NestJS (SSE) ──stream──→ Browser
  "Xin"                 data: {"text":"Xin"}      "Xin"
  " chào"               data: {"text":" chào"}    " chào"
  " bạn"                data: {"text":" bạn"}     " bạn"
  [DONE]                data: [DONE]
```

**SSE (Server-Sent Events)**: Giao thức HTTP đơn giản cho phép server push data xuống client liên tục — phù hợp hơn WebSocket cho streaming one-way.

## 2. NestJS — Streaming Endpoint

### Cách 1: Dùng SSE với Observable

```typescript
// src/chat/chat.controller.ts
import { Controller, Post, Body, Sse, Res } from '@nestjs/common';
import { Observable, from, map } from 'rxjs';
import OpenAI from 'openai';
import { Response } from 'express';

@Controller('chat')
export class ChatController {
  private openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

  @Post('stream')
  @Sse()  // Khai báo SSE endpoint
  async streamChat(@Body() body: { message: string }): Promise<Observable<MessageEvent>> {
    const stream = await this.openai.chat.completions.create({
      model: 'gpt-4o-mini',
      messages: [{ role: 'user', content: body.message }],
      stream: true,
    });

    return new Observable(subscriber => {
      (async () => {
        try {
          for await (const chunk of stream) {
            const text = chunk.choices[0]?.delta?.content ?? '';
            if (text) {
              subscriber.next({ data: JSON.stringify({ text }) } as MessageEvent);
            }
          }
          subscriber.next({ data: '[DONE]' } as MessageEvent);
          subscriber.complete();
        } catch (err) {
          subscriber.error(err);
        }
      })();
    });
  }
}
```

### Cách 2: Dùng Response stream trực tiếp (linh hoạt hơn)

```typescript
@Post('stream/raw')
async streamRaw(
  @Body() body: { message: string; systemPrompt?: string },
  @Res() res: Response,
) {
  // Cấu hình headers cho SSE
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  res.setHeader('Access-Control-Allow-Origin', '*');
  res.flushHeaders();

  const stream = await this.openai.chat.completions.create({
    model: 'gpt-4o-mini',
    messages: [
      ...(body.systemPrompt ? [{ role: 'system' as const, content: body.systemPrompt }] : []),
      { role: 'user', content: body.message },
    ],
    stream: true,
  });

  let fullText = '';

  try {
    for await (const chunk of stream) {
      const text = chunk.choices[0]?.delta?.content ?? '';
      if (text) {
        fullText += text;
        res.write(`data: ${JSON.stringify({ text })}\n\n`);
      }
    }

    // Gửi signal kết thúc kèm full text
    res.write(`data: ${JSON.stringify({ done: true, fullText })}\n\n`);
  } catch (error) {
    res.write(`data: ${JSON.stringify({ error: 'Stream error' })}\n\n`);
  } finally {
    res.end();
  }
}
```

### RAG + Streaming

```typescript
@Post('rag/stream')
async ragStream(@Body() body: { question: string }, @Res() res: Response) {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.flushHeaders();

  // 1. Tìm context từ vector DB (không stream bước này)
  const queryVector = await this.openai.createEmbedding(body.question);
  const chunks = await this.qdrant.search(queryVector, 5);
  const context = chunks.map(c => c.text).join('\n\n');

  // Thông báo cho client biết đã tìm xong context
  res.write(`data: ${JSON.stringify({ type: 'context_ready', sources: chunks.map(c => c.source) })}\n\n`);

  // 2. Stream câu trả lời từ LLM
  const stream = await this.openai.client.chat.completions.create({
    model: 'gpt-4o-mini',
    messages: [
      { role: 'system', content: `Trả lời dựa trên ngữ cảnh:\n${context}` },
      { role: 'user', content: body.question },
    ],
    stream: true,
  });

  for await (const chunk of stream) {
    const text = chunk.choices[0]?.delta?.content ?? '';
    if (text) {
      res.write(`data: ${JSON.stringify({ type: 'text', text })}\n\n`);
    }
  }

  res.write(`data: ${JSON.stringify({ type: 'done' })}\n\n`);
  res.end();
}
```

## 3. NextJS — Nhận và hiển thị stream

### Với App Router + fetch

```tsx
// app/chat/page.tsx
'use client';

import { useState } from 'react';

export default function ChatPage() {
  const [messages, setMessages] = useState<Array<{ role: string; content: string }>>([]);
  const [input, setInput] = useState('');
  const [isStreaming, setIsStreaming] = useState(false);

  async function sendMessage() {
    if (!input.trim() || isStreaming) return;

    const userMessage = input;
    setInput('');
    setMessages(prev => [...prev, { role: 'user', content: userMessage }]);
    setIsStreaming(true);

    // Thêm placeholder cho AI response
    setMessages(prev => [...prev, { role: 'assistant', content: '' }]);

    try {
      const response = await fetch('/api/chat/stream', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ message: userMessage }),
      });

      const reader = response.body!.getReader();
      const decoder = new TextDecoder();

      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        const lines = decoder.decode(value).split('\n');
        for (const line of lines) {
          if (!line.startsWith('data: ')) continue;
          const data = line.slice(6);
          if (data === '[DONE]') break;

          try {
            const { text } = JSON.parse(data);
            if (text) {
              // Cập nhật message cuối cùng (AI response)
              setMessages(prev => {
                const updated = [...prev];
                updated[updated.length - 1] = {
                  role: 'assistant',
                  content: updated[updated.length - 1].content + text,
                };
                return updated;
              });
            }
          } catch {}
        }
      }
    } finally {
      setIsStreaming(false);
    }
  }

  return (
    <div className="flex flex-col h-screen max-w-2xl mx-auto p-4">
      <div className="flex-1 overflow-y-auto space-y-4 mb-4">
        {messages.map((msg, i) => (
          <div key={i} className={`flex ${msg.role === 'user' ? 'justify-end' : 'justify-start'}`}>
            <div className={`rounded-lg px-4 py-2 max-w-[80%] ${
              msg.role === 'user' ? 'bg-blue-500 text-white' : 'bg-gray-100'
            }`}>
              {msg.content || (isStreaming && i === messages.length - 1 ? '▋' : '')}
            </div>
          </div>
        ))}
      </div>

      <div className="flex gap-2">
        <input
          className="flex-1 border rounded-lg px-4 py-2"
          value={input}
          onChange={e => setInput(e.target.value)}
          onKeyDown={e => e.key === 'Enter' && sendMessage()}
          placeholder="Nhập câu hỏi..."
          disabled={isStreaming}
        />
        <button
          onClick={sendMessage}
          disabled={isStreaming}
          className="bg-blue-500 text-white px-4 py-2 rounded-lg disabled:opacity-50"
        >
          {isStreaming ? 'Đang trả lời...' : 'Gửi'}
        </button>
      </div>
    </div>
  );
}
```

### NextJS API Route proxy (nếu cần ẩn backend URL)

```typescript
// app/api/chat/stream/route.ts
import { NextRequest } from 'next/server';

export async function POST(request: NextRequest) {
  const body = await request.json();

  const backendResponse = await fetch(`${process.env.API_URL}/chat/stream`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  });

  // Proxy stream từ backend sang client
  return new Response(backendResponse.body, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
    },
  });
}
```

## 4. Kết luận

Streaming cải thiện UX đáng kể cho AI chatbot:

- **NestJS**: Dùng SSE với `@Sse()` decorator hoặc raw `Response` stream
- **NextJS**: `fetch` → `response.body.getReader()` để đọc stream từng chunk
- **RAG + Stream**: Có thể stream cả metadata (sources) trước khi stream text
- Cursor nhấp nháy (`▋`) khi streaming giúp user biết AI đang "gõ"

Streaming biến trải nghiệm từ "chờ đợi" sang "đối thoại tự nhiên" — luôn implement cho AI chatbot production.
