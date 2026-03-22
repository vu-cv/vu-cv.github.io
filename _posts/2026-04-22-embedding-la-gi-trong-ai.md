---
layout: article
title: Embedding là gì? Tại sao quan trọng trong AI?
tags: [ai, embedding, vector, nlp, semantic-search]
---
Embedding là nền tảng của rất nhiều ứng dụng AI hiện đại: từ tìm kiếm ngữ nghĩa, RAG chatbot, đến gợi ý sản phẩm. Bài này giải thích embedding là gì và cách sử dụng trong thực tế.

## 1. Embedding là gì?

**Embedding** là quá trình chuyển đổi dữ liệu (text, ảnh, âm thanh...) thành một **vector số thực nhiều chiều** mà máy tính có thể xử lý và so sánh.

```
"Tôi yêu lập trình" → [0.12, -0.45, 0.78, 0.03, ..., 0.91]  (1536 chiều)
"I love coding"      → [0.11, -0.44, 0.79, 0.02, ..., 0.90]  (rất gần nhau!)
"Hà Nội hôm nay mưa" → [0.55, 0.23, -0.67, 0.88, ..., -0.12] (rất khác)
```

**Điểm mấu chốt**: Các câu có nghĩa tương đồng sẽ có vector gần nhau trong không gian vector, dù ngôn ngữ khác nhau.

## 2. Tại sao không dùng tìm kiếm từ khóa thông thường?

```
Câu hỏi: "Làm thế nào để trả lại hàng?"

Keyword search sẽ tìm:
✅ "Chính sách trả hàng"
❌ "Quy trình hoàn trả đơn hàng"   ← từ khóa khác nhau
❌ "Hướng dẫn đổi/trả sản phẩm"   ← từ khóa khác nhau

Semantic search (embedding) sẽ tìm:
✅ "Chính sách trả hàng"
✅ "Quy trình hoàn trả đơn hàng"   ← hiểu nghĩa tương đồng
✅ "Hướng dẫn đổi/trả sản phẩm"   ← hiểu nghĩa tương đồng
```

## 3. Đo độ tương đồng — Cosine Similarity

Hai vector được so sánh bằng **Cosine Similarity**:

```
similarity = cos(θ) = (A · B) / (|A| × |B|)
```

- Giá trị từ `-1` đến `1`
- `1.0` = giống hệt nhau
- `0.0` = không liên quan
- `-1.0` = đối lập hoàn toàn

```typescript
function cosineSimilarity(a: number[], b: number[]): number {
  const dot = a.reduce((sum, ai, i) => sum + ai * b[i], 0);
  const magA = Math.sqrt(a.reduce((sum, ai) => sum + ai * ai, 0));
  const magB = Math.sqrt(b.reduce((sum, bi) => sum + bi * bi, 0));
  return dot / (magA * magB);
}
```

## 4. Các Embedding Model phổ biến

| Model | Provider | Chiều | Ghi chú |
|-------|----------|-------|---------|
| text-embedding-3-small | OpenAI | 1536 | Rẻ, nhanh, tốt cho tiếng Anh |
| text-embedding-3-large | OpenAI | 3072 | Chính xác hơn, đắt hơn |
| text-embedding-004 | Google | 768 | Hỗ trợ đa ngôn ngữ tốt |
| multilingual-e5-large | Microsoft | 1024 | Open-source, đa ngôn ngữ |
| bge-m3 | BAAI | 1024 | Open-source, mạnh cho tiếng Việt |

## 5. Thực hành với OpenAI Embedding

### Cài đặt

```bash
npm install --save openai
```

### Tạo embedding

```typescript
import OpenAI from 'openai';

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

async function createEmbedding(text: string): Promise<number[]> {
  const response = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: text,
  });
  return response.data[0].embedding;
}

// Sử dụng
const vector = await createEmbedding('Tôi muốn trả lại sản phẩm');
console.log(vector.length); // 1536
```

### Batch embedding (nhiều text cùng lúc)

```typescript
async function createEmbeddings(texts: string[]): Promise<number[][]> {
  const response = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: texts, // Gửi nhiều text một lần — tiết kiệm API call
  });
  return response.data.map(item => item.embedding);
}
```

## 6. Thực hành với Google Gemini Embedding

```bash
npm install --save @google/generative-ai
```

```typescript
import { GoogleGenerativeAI } from '@google/generative-ai';

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);
const model = genAI.getGenerativeModel({ model: 'text-embedding-004' });

async function embedWithGemini(text: string): Promise<number[]> {
  const result = await model.embedContent(text);
  return result.embedding.values;
}
```

## 7. Ứng dụng thực tế

### Semantic Search đơn giản (không cần Vector DB)

```typescript
interface Document {
  id: string;
  text: string;
  embedding?: number[];
}

class SimpleSemanticSearch {
  private documents: Document[] = [];

  async addDocument(id: string, text: string) {
    const embedding = await createEmbedding(text);
    this.documents.push({ id, text, embedding });
  }

  async search(query: string, topK = 3): Promise<Document[]> {
    const queryEmbedding = await createEmbedding(query);

    const scored = this.documents.map(doc => ({
      ...doc,
      score: cosineSimilarity(queryEmbedding, doc.embedding!),
    }));

    return scored
      .sort((a, b) => b.score - a.score)
      .slice(0, topK);
  }
}

// Sử dụng
const search = new SimpleSemanticSearch();
await search.addDocument('1', 'Chính sách hoàn tiền trong vòng 30 ngày');
await search.addDocument('2', 'Hướng dẫn đổi trả sản phẩm lỗi');
await search.addDocument('3', 'Cách đặt hàng online');

const results = await search.search('Tôi muốn trả lại hàng');
// → ['1', '2'] với score cao nhất
```

### Phát hiện văn bản trùng lặp

```typescript
async function findDuplicates(texts: string[], threshold = 0.95) {
  const embeddings = await createEmbeddings(texts);
  const duplicates: Array<[number, number, number]> = [];

  for (let i = 0; i < embeddings.length; i++) {
    for (let j = i + 1; j < embeddings.length; j++) {
      const sim = cosineSimilarity(embeddings[i], embeddings[j]);
      if (sim >= threshold) {
        duplicates.push([i, j, sim]);
      }
    }
  }
  return duplicates;
}
```

## 8. Chi phí và tối ưu

**Chi phí OpenAI text-embedding-3-small**: `$0.02 / 1M tokens`

- 1 trang A4 ≈ 500 tokens → embed 1M trang ≈ $20
- Rất rẻ — thường không phải lo về chi phí

**Tối ưu:**
- Cache embedding của tài liệu không đổi — không cần re-embed
- Batch nhiều text vào một API call
- Với dữ liệu nhạy cảm, dùng model local (bge-m3, multilingual-e5)

## 9. Kết luận

Embedding là "ngôn ngữ" mà máy tính dùng để hiểu nghĩa văn bản:

- Chuyển text → vector để so sánh và tìm kiếm theo nghĩa
- Cosine Similarity đo độ tương đồng giữa hai vector
- OpenAI `text-embedding-3-small` là lựa chọn tốt nhất cho đa số trường hợp
- **Embedding là bước đầu tiên** trong pipeline RAG — tạo ra vector rồi lưu vào Vector Database

Bài tiếp theo: **Qdrant Vector Database – Cài đặt & Tìm kiếm ngữ nghĩa**.
