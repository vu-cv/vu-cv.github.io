---
layout: article
title: LangChain – Xây dựng AI Agent đơn giản với NodeJS
tags: [ai, langchain, agent, llm, nodejs, tools]
---
LangChain là framework phổ biến nhất để xây dựng ứng dụng AI — từ chatbot đơn giản đến AI Agent phức tạp có khả năng sử dụng tools, tìm kiếm web, chạy code. Bài này giới thiệu LangChain.js và xây dựng agent thực tế.

## 1. LangChain là gì?

LangChain cung cấp các abstraction layers:

- **Models**: Wrapper cho OpenAI, Gemini, Anthropic, Ollama...
- **Prompts**: Quản lý và format prompt linh hoạt
- **Chains**: Chuỗi các bước xử lý (prompt → model → parser → ...)
- **Memory**: Lưu trữ lịch sử cuộc trò chuyện
- **Tools**: Cho phép AI gọi function thực tế (search, calculator, database...)
- **Agents**: AI tự quyết định dùng tool nào, khi nào

## 2. Cài đặt

```bash
npm install --save langchain @langchain/openai @langchain/community
```

## 3. Model & Chain cơ bản

```typescript
import { ChatOpenAI } from '@langchain/openai';
import { ChatPromptTemplate } from '@langchain/core/prompts';
import { StringOutputParser } from '@langchain/core/output_parsers';

// Model
const model = new ChatOpenAI({
  model: 'gpt-4o-mini',
  temperature: 0.7,
  apiKey: process.env.OPENAI_API_KEY,
});

// Prompt template
const prompt = ChatPromptTemplate.fromMessages([
  ['system', 'Bạn là trợ lý AI chuyên về {domain}. Trả lời bằng tiếng Việt.'],
  ['human', '{question}'],
]);

// Chain: prompt → model → parser
const chain = prompt.pipe(model).pipe(new StringOutputParser());

// Chạy
const answer = await chain.invoke({
  domain: 'lập trình',
  question: 'NestJS và Express khác nhau thế nào?',
});
console.log(answer);
```

## 4. Memory — Lưu lịch sử trò chuyện

```typescript
import { ChatOpenAI } from '@langchain/openai';
import { ConversationChain } from 'langchain/chains';
import { BufferMemory } from 'langchain/memory';
import { ChatPromptTemplate, MessagesPlaceholder } from '@langchain/core/prompts';

const memory = new BufferMemory({
  returnMessages: true,
  memoryKey: 'history',
});

const prompt = ChatPromptTemplate.fromMessages([
  ['system', 'Bạn là trợ lý AI thân thiện. Trả lời bằng tiếng Việt.'],
  new MessagesPlaceholder('history'),
  ['human', '{input}'],
]);

const chain = new ConversationChain({
  llm: new ChatOpenAI({ model: 'gpt-4o-mini' }),
  memory,
  prompt,
});

// Lần 1
const res1 = await chain.invoke({ input: 'Tên tôi là An' });
// Lần 2 — model nhớ tên An
const res2 = await chain.invoke({ input: 'Tên tôi là gì?' });
console.log(res2.response); // "Tên bạn là An"
```

## 5. Tools — Cho AI dùng công cụ thực tế

```typescript
import { tool } from '@langchain/core/tools';
import { z } from 'zod';

// Tool tìm kiếm đơn hàng
const lookupOrderTool = tool(
  async ({ orderId }: { orderId: string }) => {
    // Query database thực tế
    const order = await db.orders.findById(orderId);
    if (!order) return `Không tìm thấy đơn hàng ${orderId}`;
    return JSON.stringify({
      id: order.id,
      status: order.status,
      total: order.total,
      estimatedDelivery: order.estimatedDelivery,
    });
  },
  {
    name: 'lookup_order',
    description: 'Tra cứu thông tin đơn hàng theo mã đơn hàng',
    schema: z.object({
      orderId: z.string().describe('Mã đơn hàng cần tra cứu'),
    }),
  },
);

// Tool tính toán
const calculatorTool = tool(
  async ({ expression }: { expression: string }) => {
    // Dùng thư viện an toàn thay vì eval
    const result = evaluate(expression);
    return String(result);
  },
  {
    name: 'calculator',
    description: 'Tính toán biểu thức toán học',
    schema: z.object({
      expression: z.string().describe('Biểu thức toán học, ví dụ: 150000 * 0.1'),
    }),
  },
);

// Tool lấy thời gian hiện tại
const getCurrentTimeTool = tool(
  async () => new Date().toLocaleString('vi-VN'),
  {
    name: 'get_current_time',
    description: 'Lấy thời gian hiện tại',
    schema: z.object({}),
  },
);
```

## 6. Agent — AI tự quyết định dùng tool nào

```typescript
import { createToolCallingAgent, AgentExecutor } from 'langchain/agents';
import { ChatOpenAI } from '@langchain/openai';
import { ChatPromptTemplate, MessagesPlaceholder } from '@langchain/core/prompts';

const tools = [lookupOrderTool, calculatorTool, getCurrentTimeTool];

const model = new ChatOpenAI({
  model: 'gpt-4o',
  temperature: 0,
}).bindTools(tools);

const prompt = ChatPromptTemplate.fromMessages([
  ['system', `Bạn là trợ lý CSKH của shop online.
Hãy dùng các công cụ có sẵn để trả lời câu hỏi của khách hàng.
Luôn trả lời bằng tiếng Việt, thân thiện và chính xác.`],
  new MessagesPlaceholder('chat_history'),
  ['human', '{input}'],
  new MessagesPlaceholder('agent_scratchpad'),
]);

const agent = createToolCallingAgent({ llm: model, tools, prompt });

const executor = new AgentExecutor({
  agent,
  tools,
  verbose: true, // Log từng bước để debug
  maxIterations: 5,
});

// Chạy agent
const result = await executor.invoke({
  input: 'Đơn hàng ORD-12345 của tôi đang ở đâu rồi?',
  chat_history: [],
});

console.log(result.output);
// "Đơn hàng ORD-12345 hiện đang ở trạng thái 'Đang giao hàng',
//  dự kiến giao vào ngày 25/12/2025. Tổng giá trị đơn hàng là 500.000đ."
```

## 7. RAG Chain với LangChain

```typescript
import { ChatOpenAI } from '@langchain/openai';
import { OpenAIEmbeddings } from '@langchain/openai';
import { QdrantVectorStore } from '@langchain/community/vectorstores/qdrant';
import { createRetrievalChain } from 'langchain/chains/retrieval';
import { createStuffDocumentsChain } from 'langchain/chains/combine_documents';

const embeddings = new OpenAIEmbeddings({ model: 'text-embedding-3-small' });

const vectorStore = await QdrantVectorStore.fromExistingCollection(embeddings, {
  url: process.env.QDRANT_URL,
  collectionName: 'knowledge_base',
});

const retriever = vectorStore.asRetriever({ k: 5 });

const model = new ChatOpenAI({ model: 'gpt-4o-mini' });

const questionAnswerChain = await createStuffDocumentsChain({
  llm: model,
  prompt: ChatPromptTemplate.fromMessages([
    ['system', `Trả lời dựa trên ngữ cảnh sau:
{context}
Nếu không có thông tin, hãy nói không biết.`],
    ['human', '{input}'],
  ]),
});

const ragChain = await createRetrievalChain({
  retriever,
  combineDocsChain: questionAnswerChain,
});

const response = await ragChain.invoke({
  input: 'Chính sách hoàn tiền như thế nào?',
});

console.log(response.answer);
```

## 8. Streaming Response

```typescript
const stream = await chain.stream({
  input: 'Giải thích Docker là gì?',
  chat_history: [],
});

process.stdout.write('AI: ');
for await (const chunk of stream) {
  if (chunk.output) process.stdout.write(chunk.output);
}
```

## 9. Kết luận

LangChain.js cung cấp building blocks để xây dựng AI apps phức tạp:

- **Chain**: Kết hợp prompt + model + parser thành pipeline
- **Memory**: Lưu lịch sử để AI nhớ context
- **Tools**: Cho AI khả năng gọi function thực tế (DB, API, search)
- **Agent**: AI tự quyết định orchestrate tools theo mục tiêu
- **RAG Chain**: Tích hợp tìm kiếm vector store sẵn có

LangChain phù hợp cho prototyping nhanh; khi cần production-grade hãy cân nhắc viết pipeline tùy chỉnh để kiểm soát tốt hơn.
