---
layout: article
title: Tổng quan về RAG (Retrieval Augmented Generation)
tags: [ai, rag, llm, chatbot, vector-db]
---
RAG (Retrieval Augmented Generation) là kỹ thuật kết hợp **tìm kiếm thông tin** với **sinh văn bản từ LLM** để tạo ra câu trả lời chính xác, cập nhật và có nguồn gốc rõ ràng. Đây là nền tảng của hầu hết các AI chatbot doanh nghiệp hiện đại.

## 1. Tại sao cần RAG?

LLM (Large Language Model) như GPT-4, Gemini có kiến thức rộng nhưng có những giới hạn:

- **Knowledge cutoff**: Không biết sự kiện sau thời điểm training
- **Hallucination**: Có thể tự "bịa" thông tin nghe có vẻ hợp lý nhưng sai
- **Không có dữ liệu nội bộ**: Không biết gì về tài liệu, sản phẩm của công ty bạn

**RAG giải quyết tất cả vấn đề trên** bằng cách cung cấp ngữ cảnh từ dữ liệu thực tế vào prompt trước khi LLM trả lời.

## 2. Kiến trúc RAG

```
[Câu hỏi của người dùng]
         ↓
  [Embedding Model]  ← Chuyển câu hỏi thành vector
         ↓
  [Vector Database]  ← Tìm các đoạn văn bản liên quan nhất
         ↓
  [Relevant Chunks]  ← Các đoạn tài liệu phù hợp
         ↓
[Prompt = Câu hỏi + Ngữ cảnh]
         ↓
      [LLM]          ← Sinh câu trả lời dựa trên ngữ cảnh
         ↓
  [Câu trả lời]
```

## 3. Hai giai đoạn chính

### Giai đoạn 1: Indexing (Xử lý tài liệu)

Đây là bước chuẩn bị, thường chạy một lần (hoặc khi có tài liệu mới):

```
Tài liệu (PDF, Word, Web...)
    ↓ Load
Raw Text
    ↓ Chunk (chia nhỏ)
Các đoạn văn bản nhỏ (~500 tokens/đoạn)
    ↓ Embed
Vectors (mảng số thực, ví dụ: 1536 chiều)
    ↓ Store
Vector Database (Qdrant, Pinecone, Weaviate...)
```

**Chunking** là bước quan trọng — chia tài liệu quá lớn thành các đoạn nhỏ có độ chồng lấp (overlap) để không mất context:

```
┌─────────────────────────────────────────────┐
│ Chunk 1 [0–500 tokens]                      │
│              Chunk 2 [400–900 tokens]        │  ← overlap 100 tokens
│                           Chunk 3 [800–...]  │
└─────────────────────────────────────────────┘
```

### Giai đoạn 2: Retrieval + Generation (Thời gian thực)

```typescript
// Pseudocode minh họa
async function rag(userQuestion: string): Promise<string> {
  // 1. Embed câu hỏi
  const questionVector = await embed(userQuestion);

  // 2. Tìm top-k chunks liên quan nhất
  const relevantChunks = await vectorDb.search(questionVector, topK: 5);

  // 3. Tạo prompt với ngữ cảnh
  const prompt = `
    Dựa vào các thông tin sau:
    ${relevantChunks.map(c => c.text).join('\n\n')}

    Hãy trả lời câu hỏi: ${userQuestion}
    Nếu không tìm thấy thông tin liên quan, hãy nói "Tôi không có thông tin về vấn đề này."
  `;

  // 4. Gọi LLM
  return await llm.complete(prompt);
}
```

## 4. Các thành phần cần thiết

| Thành phần | Ví dụ phổ biến | Chức năng |
|-----------|----------------|-----------|
| Embedding Model | text-embedding-3-small (OpenAI), Gemini Embedding | Chuyển text → vector |
| Vector Database | Qdrant, Pinecone, Weaviate, pgvector | Lưu và tìm kiếm vector |
| LLM | GPT-4o, Claude 3.5, Gemini 1.5 Pro | Sinh câu trả lời |
| Document Loader | LangChain, LlamaIndex | Đọc PDF, Word, Web... |

## 5. Các chiến lược RAG nâng cao

### Hybrid Search

Kết hợp tìm kiếm vector (semantic) với full-text search (keyword) để tăng độ chính xác:

```
Semantic Search (vector similarity)  +  BM25 (keyword)
                    ↓
              Re-ranking
                    ↓
          Top-k relevant chunks
```

### Reranker

Dùng một model nhỏ (như Cohere Rerank, BGE Reranker) để sắp xếp lại kết quả tìm kiếm cho chính xác hơn trước khi đưa vào LLM.

### HyDE (Hypothetical Document Embeddings)

Thay vì embed câu hỏi trực tiếp, dùng LLM tạo ra một câu trả lời giả định, rồi embed câu trả lời đó để tìm kiếm — giúp kết quả tìm kiếm tốt hơn.

### Self-RAG

Model tự quyết định khi nào cần retrieve thêm thông tin thay vì luôn luôn retrieve.

## 6. Đánh giá chất lượng RAG

| Metric | Đo lường gì |
|--------|------------|
| Faithfulness | Câu trả lời có trung thực với ngữ cảnh không? |
| Answer Relevancy | Câu trả lời có liên quan đến câu hỏi không? |
| Context Precision | Ngữ cảnh được retrieve có chính xác không? |
| Context Recall | Có bỏ sót thông tin quan trọng không? |

Framework đánh giá phổ biến: **RAGAS** (RAG Assessment)

## 7. Khi nào dùng RAG?

**Phù hợp với RAG:**
- Chatbot hỗ trợ khách hàng với knowledge base nội bộ
- Tìm kiếm trong tài liệu nội bộ (policy, manual, báo cáo)
- Hệ thống Q&A trên dữ liệu cập nhật thường xuyên
- Ứng dụng cần cite nguồn tài liệu

**Không cần RAG:**
- Câu hỏi general knowledge mà LLM đã biết
- Tác vụ sáng tạo (viết thơ, code, dịch thuật)
- Phân tích dữ liệu có cấu trúc (dùng SQL hoặc function calling tốt hơn)

## 8. Kết luận

RAG = **Tìm kiếm thông minh** + **LLM mạnh mẽ**:

1. **Indexing**: Chunk → Embed → Store vào Vector DB
2. **Retrieval**: Embed câu hỏi → Tìm chunks liên quan
3. **Generation**: LLM trả lời dựa trên chunks

Các bài tiếp theo sẽ đi sâu vào từng thành phần: **Embedding**, **Qdrant Vector Database**, và **Xây dựng RAG Chatbot hoàn chỉnh với NestJS**.
