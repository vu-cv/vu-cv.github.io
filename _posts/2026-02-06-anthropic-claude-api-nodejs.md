---
layout: article
title: Anthropic Claude API – Xây dựng AI App với NodeJS
tags: [anthropic, claude, ai, api, nodejs, nestjs, llm]
---
Claude là LLM của Anthropic — nổi tiếng về khả năng phân tích, viết code, và an toàn. Bài này hướng dẫn tích hợp Claude API vào NestJS, từ text generation đến tool use và streaming.

## 1. Cài đặt

```bash
npm install @anthropic-ai/sdk
```

## 2. Basic Chat

```typescript
// src/ai/claude.service.ts
import { Injectable, Logger } from '@nestjs/common';
import Anthropic from '@anthropic-ai/sdk';

@Injectable()
export class ClaudeService {
  private readonly logger = new Logger(ClaudeService.name);
  private client: Anthropic;

  constructor() {
    this.client = new Anthropic({
      apiKey: process.env.ANTHROPIC_API_KEY!,
    });
  }

  // Basic message
  async chat(prompt: string, systemPrompt?: string): Promise<string> {
    const message = await this.client.messages.create({
      model: 'claude-opus-4-6',     // Mạnh nhất — 'claude-sonnet-4-6' cho balanced
      max_tokens: 1024,
      system: systemPrompt,
      messages: [
        { role: 'user', content: prompt },
      ],
    });

    const content = message.content[0];
    if (content.type !== 'text') throw new Error('Unexpected response type');
    return content.text;
  }

  // Multi-turn conversation
  async continueConversation(
    history: Array<{ role: 'user' | 'assistant'; content: string }>,
    newMessage: string,
    systemPrompt?: string,
  ): Promise<string> {
    const messages = [
      ...history.map(m => ({ role: m.role, content: m.content })),
      { role: 'user' as const, content: newMessage },
    ];

    const response = await this.client.messages.create({
      model: 'claude-sonnet-4-6',
      max_tokens: 2048,
      system: systemPrompt,
      messages,
    });

    return (response.content[0] as Anthropic.TextBlock).text;
  }
}
```

## 3. Models và khi nào dùng

```typescript
const CLAUDE_MODELS = {
  // Mạnh nhất — phân tích phức tạp, code khó
  OPUS: 'claude-opus-4-6',

  // Balanced — production chatbot, coding assistant
  SONNET: 'claude-sonnet-4-6',

  // Nhanh và rẻ — tasks đơn giản, classification, summarization
  HAIKU: 'claude-haiku-4-5-20251001',
};

// Chi phí tham khảo (input/output per 1M tokens)
// Opus:   $15 / $75
// Sonnet: $3  / $15
// Haiku:  $0.25 / $1.25
```

## 4. Streaming

```typescript
@Post('chat/stream')
async streamChat(@Body() body: { message: string }, @Res() res: Response) {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.flushHeaders();

  let fullText = '';

  const stream = this.client.messages.stream({
    model: 'claude-sonnet-4-6',
    max_tokens: 1024,
    messages: [{ role: 'user', content: body.message }],
  });

  stream.on('text', (text) => {
    fullText += text;
    res.write(`data: ${JSON.stringify({ text })}\n\n`);
  });

  stream.on('message', (message) => {
    // Message complete — gửi usage info
    res.write(`data: ${JSON.stringify({
      done: true,
      fullText,
      usage: {
        inputTokens: message.usage.input_tokens,
        outputTokens: message.usage.output_tokens,
      },
    })}\n\n`);
    res.end();
  });

  stream.on('error', (error) => {
    res.write(`data: ${JSON.stringify({ error: error.message })}\n\n`);
    res.end();
  });
}
```

## 5. Tool Use (Function Calling)

```typescript
const tools: Anthropic.Tool[] = [
  {
    name: 'search_products',
    description: 'Tìm kiếm sản phẩm trong database theo từ khóa và bộ lọc',
    input_schema: {
      type: 'object',
      properties: {
        query: { type: 'string', description: 'Từ khóa tìm kiếm' },
        max_price: { type: 'number', description: 'Giá tối đa (VND)' },
        category: { type: 'string', description: 'Danh mục sản phẩm' },
      },
      required: ['query'],
    },
  },
  {
    name: 'get_order_status',
    description: 'Lấy trạng thái đơn hàng theo mã đơn',
    input_schema: {
      type: 'object',
      properties: {
        order_id: { type: 'string' },
      },
      required: ['order_id'],
    },
  },
];

async chatWithTools(userMessage: string): Promise<string> {
  const messages: Anthropic.MessageParam[] = [
    { role: 'user', content: userMessage },
  ];

  while (true) {
    const response = await this.client.messages.create({
      model: 'claude-sonnet-4-6',
      max_tokens: 1024,
      tools,
      messages,
    });

    // Nếu Claude quyết định dùng tool
    if (response.stop_reason === 'tool_use') {
      messages.push({ role: 'assistant', content: response.content });

      // Xử lý từng tool call
      const toolResults: Anthropic.ToolResultBlockParam[] = [];

      for (const block of response.content) {
        if (block.type !== 'tool_use') continue;

        let result: string;
        try {
          result = await this.executeTool(block.name, block.input as Record<string, any>);
        } catch (e) {
          result = JSON.stringify({ error: e.message });
        }

        toolResults.push({
          type: 'tool_result',
          tool_use_id: block.id,
          content: result,
        });
      }

      messages.push({ role: 'user', content: toolResults });
    } else {
      // Stop reason là 'end_turn' → trả về text
      const textBlock = response.content.find(b => b.type === 'text');
      return (textBlock as Anthropic.TextBlock)?.text ?? '';
    }
  }
}

private async executeTool(name: string, input: Record<string, any>): Promise<string> {
  switch (name) {
    case 'search_products': {
      const products = await this.productsService.search(input);
      return JSON.stringify(products.slice(0, 5));
    }
    case 'get_order_status': {
      const order = await this.ordersService.findById(input.order_id);
      return JSON.stringify(order ?? { error: 'Not found' });
    }
    default:
      return JSON.stringify({ error: `Unknown tool: ${name}` });
  }
}
```

## 6. Vision — Phân tích ảnh

```typescript
async analyzeImage(imageUrl: string, question: string): Promise<string> {
  const response = await this.client.messages.create({
    model: 'claude-opus-4-6',
    max_tokens: 1024,
    messages: [{
      role: 'user',
      content: [
        {
          type: 'image',
          source: { type: 'url', url: imageUrl },
        },
        { type: 'text', text: question },
      ],
    }],
  });

  return (response.content[0] as Anthropic.TextBlock).text;
}

// Từ base64 (file upload)
async analyzeImageBuffer(buffer: Buffer, mimeType: string, question: string): Promise<string> {
  const response = await this.client.messages.create({
    model: 'claude-opus-4-6',
    max_tokens: 1024,
    messages: [{
      role: 'user',
      content: [
        {
          type: 'image',
          source: {
            type: 'base64',
            media_type: mimeType as any,
            data: buffer.toString('base64'),
          },
        },
        { type: 'text', text: question },
      ],
    }],
  });

  return (response.content[0] as Anthropic.TextBlock).text;
}
```

## 7. Structured Output với JSON

```typescript
async extractStructuredData<T>(text: string, schema: string): Promise<T> {
  const prompt = `Phân tích văn bản sau và trả về JSON theo schema:

Schema:
${schema}

Văn bản:
${text}

Trả về CHÍNH XÁC JSON, không giải thích thêm.`;

  const response = await this.client.messages.create({
    model: 'claude-haiku-4-5-20251001', // Haiku đủ cho structured extraction
    max_tokens: 1024,
    messages: [{ role: 'user', content: prompt }],
  });

  const text_content = (response.content[0] as Anthropic.TextBlock).text;

  // Parse JSON từ response
  const jsonMatch = text_content.match(/```json\n?([\s\S]*?)\n?```/) ?? [null, text_content];
  return JSON.parse(jsonMatch[1]);
}

// Dùng
const product = await this.claudeService.extractStructuredData<{
  name: string; price: number; category: string;
}>(
  rawText,
  `{ "name": string, "price": number, "category": string }`,
);
```

## 8. System Prompt thực tế

```typescript
const CUSTOMER_SUPPORT_SYSTEM = `Bạn là trợ lý CSKH của ShopXYZ. Hãy tuân thủ:

TÍNH CÁCH:
- Thân thiện, lịch sự, kiên nhẫn
- Sử dụng tiếng Việt tự nhiên, không quá trang trọng
- Gọi khách là "bạn"

PHẠM VI:
- Tra cứu đơn hàng, vận chuyển
- Chính sách đổi trả (7 ngày kể từ ngày nhận)
- Hướng dẫn thanh toán
- Tư vấn sản phẩm

GIỚI HẠN:
- Không bịa thông tin, không chắc thì nói "Tôi cần kiểm tra"
- Khiếu nại nghiêm trọng → escalate lên agent người thật
- Không thảo luận về đối thủ

ĐỊNH DẠNG:
- Câu ngắn gọn, rõ ràng
- Dùng bullet points khi liệt kê nhiều items
- Không quá 3 đoạn văn`;
```

## 9. Kết luận

- **Models**: Haiku (nhanh/rẻ) → Sonnet (balanced) → Opus (mạnh nhất)
- **Tool Use**: Tương tự OpenAI Function Calling — Claude quyết định khi nào gọi tool
- **Streaming**: `client.messages.stream()` với event handlers — tốt hơn dùng `.on('text')`
- **Vision**: Hỗ trợ image_url và base64 — Claude Opus vision rất tốt
- **Pricing**: Token-based — dùng Haiku cho classification/extraction, Sonnet cho chat

Claude nổi trội ở phân tích văn bản dài, code review, và reasoning — thử so sánh với GPT-4o cho use case cụ thể trước khi chọn.
