---
layout: article
title: Tích hợp OpenAI & Gemini API vào NestJS
tags: [ai, openai, gemini, nestjs, llm]
---
OpenAI và Google Gemini là hai LLM API mạnh nhất hiện nay. Bài này hướng dẫn tích hợp cả hai vào NestJS, xây dựng một service linh hoạt có thể switch giữa các provider.

## 1. Cài đặt

```bash
# OpenAI SDK
npm install --save openai

# Google Gemini SDK
npm install --save @google/generative-ai
```

Thêm vào `.env`:
```
OPENAI_API_KEY=sk-...
GEMINI_API_KEY=AIza...
AI_PROVIDER=openai  # hoặc gemini
```

## 2. OpenAI Integration

### Text Generation

```typescript
import OpenAI from 'openai';

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

// Chat Completion
async function chatWithOpenAI(prompt: string): Promise<string> {
  const res = await openai.chat.completions.create({
    model: 'gpt-4o-mini',
    messages: [{ role: 'user', content: prompt }],
    temperature: 0.7,
    max_tokens: 1000,
  });
  return res.choices[0].message.content ?? '';
}

// Với system prompt
async function chatWithSystem(system: string, user: string): Promise<string> {
  const res = await openai.chat.completions.create({
    model: 'gpt-4o',
    messages: [
      { role: 'system', content: system },
      { role: 'user', content: user },
    ],
  });
  return res.choices[0].message.content ?? '';
}
```

### Streaming Response

```typescript
import { Response } from 'express';

async function streamOpenAI(prompt: string, res: Response) {
  const stream = await openai.chat.completions.create({
    model: 'gpt-4o-mini',
    messages: [{ role: 'user', content: prompt }],
    stream: true,
  });

  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');

  for await (const chunk of stream) {
    const text = chunk.choices[0]?.delta?.content ?? '';
    if (text) {
      res.write(`data: ${JSON.stringify({ text })}\n\n`);
    }
  }
  res.write('data: [DONE]\n\n');
  res.end();
}
```

### Function Calling (Tool Use)

```typescript
const tools: OpenAI.ChatCompletionTool[] = [
  {
    type: 'function',
    function: {
      name: 'get_weather',
      description: 'Lấy thông tin thời tiết hiện tại của một thành phố',
      parameters: {
        type: 'object',
        properties: {
          city: { type: 'string', description: 'Tên thành phố' },
          unit: { type: 'string', enum: ['celsius', 'fahrenheit'] },
        },
        required: ['city'],
      },
    },
  },
];

async function chatWithTools(question: string) {
  const res = await openai.chat.completions.create({
    model: 'gpt-4o',
    messages: [{ role: 'user', content: question }],
    tools,
  });

  const toolCall = res.choices[0].message.tool_calls?.[0];
  if (toolCall) {
    const args = JSON.parse(toolCall.function.arguments);
    console.log('Model muốn gọi:', toolCall.function.name, args);
    // → Gọi function thực tế và trả kết quả lại cho model
  }
}
```

### Vision — Phân tích ảnh

```typescript
async function analyzeImage(imageUrl: string, question: string): Promise<string> {
  const res = await openai.chat.completions.create({
    model: 'gpt-4o',
    messages: [
      {
        role: 'user',
        content: [
          { type: 'image_url', image_url: { url: imageUrl } },
          { type: 'text', text: question },
        ],
      },
    ],
  });
  return res.choices[0].message.content ?? '';
}

// Sử dụng
const description = await analyzeImage(
  'https://example.com/product.jpg',
  'Mô tả sản phẩm trong ảnh này bằng tiếng Việt',
);
```

## 3. Google Gemini Integration

```typescript
import { GoogleGenerativeAI, HarmCategory, HarmBlockThreshold } from '@google/generative-ai';

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);

// Text generation
async function chatWithGemini(prompt: string): Promise<string> {
  const model = genAI.getGenerativeModel({ model: 'gemini-1.5-flash' });
  const result = await model.generateContent(prompt);
  return result.response.text();
}

// Multi-turn conversation
async function geminiMultiTurn() {
  const model = genAI.getGenerativeModel({ model: 'gemini-1.5-pro' });
  const chat = model.startChat({
    history: [
      { role: 'user', parts: [{ text: 'Xin chào!' }] },
      { role: 'model', parts: [{ text: 'Xin chào! Tôi có thể giúp gì cho bạn?' }] },
    ],
  });

  const res = await chat.sendMessage('Giải thích RAG là gì?');
  return res.response.text();
}

// Vision với Gemini
async function analyzeImageGemini(imageBase64: string, mimeType: string, prompt: string) {
  const model = genAI.getGenerativeModel({ model: 'gemini-1.5-pro' });
  const result = await model.generateContent([
    prompt,
    { inlineData: { data: imageBase64, mimeType } },
  ]);
  return result.response.text();
}

// Streaming
async function streamGemini(prompt: string) {
  const model = genAI.getGenerativeModel({ model: 'gemini-1.5-flash' });
  const stream = await model.generateContentStream(prompt);

  for await (const chunk of stream.stream) {
    process.stdout.write(chunk.text());
  }
}
```

## 4. Unified AI Service trong NestJS

Xây dựng service trừu tượng hóa cả hai provider:

```typescript
// src/ai/ai.service.ts
import { Injectable } from '@nestjs/common';
import OpenAI from 'openai';
import { GoogleGenerativeAI } from '@google/generative-ai';

export interface ChatMessage {
  role: 'user' | 'assistant';
  content: string;
}

@Injectable()
export class AiService {
  private openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
  private gemini = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);
  private provider = process.env.AI_PROVIDER ?? 'openai';

  async chat(messages: ChatMessage[], systemPrompt?: string): Promise<string> {
    if (this.provider === 'gemini') {
      return this.chatWithGemini(messages, systemPrompt);
    }
    return this.chatWithOpenAI(messages, systemPrompt);
  }

  private async chatWithOpenAI(messages: ChatMessage[], system?: string): Promise<string> {
    const openaiMessages: OpenAI.ChatCompletionMessageParam[] = [
      ...(system ? [{ role: 'system' as const, content: system }] : []),
      ...messages.map(m => ({ role: m.role, content: m.content })),
    ];

    const res = await this.openai.chat.completions.create({
      model: 'gpt-4o-mini',
      messages: openaiMessages,
    });
    return res.choices[0].message.content ?? '';
  }

  private async chatWithGemini(messages: ChatMessage[], system?: string): Promise<string> {
    const model = this.gemini.getGenerativeModel({
      model: 'gemini-1.5-flash',
      systemInstruction: system,
    });

    const history = messages.slice(0, -1).map(m => ({
      role: m.role === 'assistant' ? 'model' : 'user',
      parts: [{ text: m.content }],
    }));

    const chat = model.startChat({ history });
    const lastMessage = messages[messages.length - 1].content;
    const result = await chat.sendMessage(lastMessage);
    return result.response.text();
  }

  async embed(text: string): Promise<number[]> {
    // Luôn dùng OpenAI cho embedding (chất lượng tốt hơn)
    const res = await this.openai.embeddings.create({
      model: 'text-embedding-3-small',
      input: text,
    });
    return res.data[0].embedding;
  }
}
```

## 5. Controller với Streaming

```typescript
import { Controller, Post, Body, Res } from '@nestjs/common';
import { Response } from 'express';
import OpenAI from 'openai';

@Controller('ai')
export class AiController {
  private openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

  @Post('chat')
  async chat(@Body() body: { message: string }) {
    return { reply: await chatWithOpenAI(body.message) };
  }

  @Post('stream')
  async stream(@Body() body: { message: string }, @Res() res: Response) {
    await streamOpenAI(body.message, res);
  }
}
```

## 6. So sánh OpenAI vs Gemini

| Tiêu chí | GPT-4o | Gemini 1.5 Pro |
|---------|--------|----------------|
| Chất lượng code | Rất tốt | Tốt |
| Đa ngôn ngữ | Tốt | Rất tốt |
| Context window | 128K tokens | 1M tokens |
| Vision | Có | Có |
| Giá (per 1M tokens) | $5 input / $15 output | $3.5 input / $10.5 output |
| Free tier | Không | Có (Gemini Flash) |
| Tốc độ | Nhanh | Rất nhanh (Flash) |

## 7. Kết luận

- **OpenAI GPT-4o**: Tốt nhất cho coding, reasoning, tool use
- **Gemini 1.5 Pro**: Context window cực lớn (1M tokens), giá cạnh tranh
- **Gemini 1.5 Flash**: Miễn phí, nhanh, đủ tốt cho hầu hết tác vụ
- **Unified Service**: Tách provider ra khỏi business logic để dễ switch

Đây là bài cuối trong chuỗi về AI. Kết hợp tất cả kiến thức từ RAG, Qdrant, Embedding và các API này, bạn đã có đủ nền tảng để xây dựng một AI application production-ready.
