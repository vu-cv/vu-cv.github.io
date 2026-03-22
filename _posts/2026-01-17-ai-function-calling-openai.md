---
layout: article
title: AI – Function Calling / Tool Use với OpenAI
tags: [ai, openai, function-calling, tool-use, nodejs, nestjs]
---
Function Calling cho phép LLM gọi các hàm/API bên ngoài — thay vì chỉ trả về text, model có thể quyết định khi nào cần tra cứu database, gọi API thời tiết, tính toán, hay gửi email. Đây là nền tảng của AI Agent.

## 1. Cách hoạt động

```
User: "Đơn hàng #123 của tôi đang ở đâu?"

→ LLM xác định: Cần gọi hàm get_order_status({ orderId: "123" })
→ App gọi hàm thật → trả kết quả về cho LLM
→ LLM tổng hợp: "Đơn hàng #123 đang được giao, dự kiến ngày mai"
```

LLM không tự gọi hàm — nó chỉ ra tên hàm và tham số, application gọi hàm thật.

## 2. Định nghĩa Tools

```typescript
import OpenAI from 'openai';

const tools: OpenAI.Chat.ChatCompletionTool[] = [
  {
    type: 'function',
    function: {
      name: 'get_order_status',
      description: 'Lấy trạng thái và vị trí hiện tại của đơn hàng',
      parameters: {
        type: 'object',
        properties: {
          order_id: {
            type: 'string',
            description: 'Mã đơn hàng, ví dụ: ORD-2025-001',
          },
        },
        required: ['order_id'],
      },
    },
  },
  {
    type: 'function',
    function: {
      name: 'search_products',
      description: 'Tìm kiếm sản phẩm theo từ khóa, danh mục, hoặc giá',
      parameters: {
        type: 'object',
        properties: {
          query: { type: 'string', description: 'Từ khóa tìm kiếm' },
          category: { type: 'string', description: 'Danh mục sản phẩm' },
          max_price: { type: 'number', description: 'Giá tối đa (VND)' },
          limit: { type: 'number', description: 'Số lượng kết quả, mặc định 5' },
        },
        required: ['query'],
      },
    },
  },
  {
    type: 'function',
    function: {
      name: 'create_support_ticket',
      description: 'Tạo ticket hỗ trợ khi khách hàng có vấn đề cần giải quyết',
      parameters: {
        type: 'object',
        properties: {
          issue_type: {
            type: 'string',
            enum: ['shipping', 'payment', 'product', 'refund', 'other'],
            description: 'Loại vấn đề',
          },
          description: { type: 'string', description: 'Mô tả chi tiết vấn đề' },
          priority: {
            type: 'string',
            enum: ['low', 'medium', 'high'],
            description: 'Độ ưu tiên',
          },
        },
        required: ['issue_type', 'description'],
      },
    },
  },
];
```

## 3. Tool Handlers

```typescript
// src/ai/tool-handlers.ts
export class ToolHandlers {
  constructor(
    private ordersService: OrdersService,
    private productsService: ProductsService,
    private ticketsService: TicketsService,
  ) {}

  async execute(toolName: string, args: Record<string, any>): Promise<string> {
    switch (toolName) {
      case 'get_order_status': {
        const order = await this.ordersService.findById(args.order_id);
        if (!order) return JSON.stringify({ error: 'Không tìm thấy đơn hàng' });

        return JSON.stringify({
          order_id: order.id,
          status: order.status,
          status_vi: this.translateStatus(order.status),
          created_at: order.createdAt,
          estimated_delivery: order.estimatedDelivery,
          tracking_code: order.trackingCode,
          items_count: order.items.length,
        });
      }

      case 'search_products': {
        const products = await this.productsService.search({
          query: args.query,
          category: args.category,
          maxPrice: args.max_price,
          limit: args.limit ?? 5,
        });

        return JSON.stringify({
          total: products.length,
          products: products.map(p => ({
            id: p.id,
            name: p.name,
            price: p.price,
            price_formatted: `${p.price.toLocaleString('vi-VN')}đ`,
            category: p.category,
            in_stock: p.stock > 0,
            rating: p.rating,
          })),
        });
      }

      case 'create_support_ticket': {
        const ticket = await this.ticketsService.create({
          issueType: args.issue_type,
          description: args.description,
          priority: args.priority ?? 'medium',
        });

        return JSON.stringify({
          ticket_id: ticket.id,
          message: `Ticket #${ticket.id} đã được tạo. Chúng tôi sẽ phản hồi trong vòng 2 giờ.`,
        });
      }

      default:
        return JSON.stringify({ error: `Unknown tool: ${toolName}` });
    }
  }

  private translateStatus(status: string): string {
    const map: Record<string, string> = {
      pending: 'Chờ xác nhận',
      confirmed: 'Đã xác nhận',
      shipping: 'Đang giao hàng',
      delivered: 'Đã giao',
      cancelled: 'Đã hủy',
    };
    return map[status] ?? status;
  }
}
```

## 4. Chat Service với Tool Use

```typescript
// src/ai/chat.service.ts
import OpenAI from 'openai';

@Injectable()
export class ChatService {
  private openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

  constructor(private toolHandlers: ToolHandlers) {}

  async chat(
    userMessage: string,
    conversationHistory: OpenAI.Chat.ChatCompletionMessageParam[] = [],
  ): Promise<string> {
    const messages: OpenAI.Chat.ChatCompletionMessageParam[] = [
      {
        role: 'system',
        content: `Bạn là trợ lý CSKH của ShopXYZ. Dùng các công cụ có sẵn để tra cứu thông tin
        thực tế. Luôn thân thiện và trả lời bằng tiếng Việt.`,
      },
      ...conversationHistory,
      { role: 'user', content: userMessage },
    ];

    // Vòng lặp xử lý tool calls
    while (true) {
      const response = await this.openai.chat.completions.create({
        model: 'gpt-4o-mini',
        messages,
        tools,
        tool_choice: 'auto',
      });

      const message = response.choices[0].message;
      messages.push(message); // Lưu lại response

      // Nếu không có tool call → trả về kết quả cuối
      if (!message.tool_calls || message.tool_calls.length === 0) {
        return message.content ?? '';
      }

      // Xử lý từng tool call song song
      const toolResults = await Promise.all(
        message.tool_calls.map(async (toolCall) => {
          const args = JSON.parse(toolCall.function.arguments);
          const result = await this.toolHandlers.execute(toolCall.function.name, args);

          return {
            role: 'tool' as const,
            tool_call_id: toolCall.id,
            content: result,
          };
        }),
      );

      // Thêm kết quả tool vào conversation
      messages.push(...toolResults);

      // LLM sẽ dùng kết quả tool để tạo response cuối
    }
  }
}
```

## 5. Parallel Tool Calls

LLM có thể gọi nhiều tool cùng lúc:

```
User: "Tìm tai nghe dưới 500k và cho tôi biết đơn hàng #456 đang ở đâu"

→ LLM gọi đồng thời:
   - search_products({ query: "tai nghe", max_price: 500000 })
   - get_order_status({ order_id: "456" })

→ Xử lý song song với Promise.all
→ LLM tổng hợp cả 2 kết quả trong 1 response
```

Code xử lý parallel ở trên (dùng `Promise.all`) đã handle tự động.

## 6. Forced Tool Call

```typescript
// Bắt buộc LLM phải gọi một tool cụ thể
const response = await openai.chat.completions.create({
  model: 'gpt-4o-mini',
  messages,
  tools,
  tool_choice: {
    type: 'function',
    function: { name: 'search_products' },
  },
});
```

## 7. Streaming với Tool Use

```typescript
@Post('chat/stream')
async streamChat(@Body() body: { message: string }, @Res() res: Response) {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.flushHeaders();

  const messages = [/* ... */];
  let toolCalls: any[] = [];

  const stream = await this.openai.chat.completions.create({
    model: 'gpt-4o-mini',
    messages,
    tools,
    stream: true,
  });

  for await (const chunk of stream) {
    const delta = chunk.choices[0]?.delta;

    if (delta?.tool_calls) {
      // Accumulate tool calls từ stream chunks
      for (const tc of delta.tool_calls) {
        if (!toolCalls[tc.index]) toolCalls[tc.index] = { id: '', function: { name: '', arguments: '' } };
        toolCalls[tc.index].id += tc.id ?? '';
        toolCalls[tc.index].function.name += tc.function?.name ?? '';
        toolCalls[tc.index].function.arguments += tc.function?.arguments ?? '';
      }
    } else if (delta?.content) {
      res.write(`data: ${JSON.stringify({ text: delta.content })}\n\n`);
    }

    if (chunk.choices[0]?.finish_reason === 'tool_calls') {
      // Thông báo đang tra cứu
      res.write(`data: ${JSON.stringify({ type: 'tool_start', tools: toolCalls.map(t => t.function.name) })}\n\n`);

      // Xử lý tool calls và tiếp tục
      // ...
    }
  }

  res.end();
}
```

## 8. Kết luận

- **Tools**: Định nghĩa schema rõ ràng với `description` tốt — LLM dựa vào description để quyết định khi nào gọi
- **Vòng lặp**: Tiếp tục gọi LLM sau khi thực thi tool — có thể gọi nhiều tool liên tiếp
- **Parallel**: LLM có thể gọi nhiều tool đồng thời — xử lý với `Promise.all`
- **Forced tool**: Khi muốn đảm bảo LLM phải dùng một tool cụ thể
- **Error handling**: Luôn trả về JSON error từ tool handler thay vì throw exception

Function Calling là cầu nối giữa LLM và hệ thống thực — nền tảng xây dựng AI Agent thực sự hữu ích.
