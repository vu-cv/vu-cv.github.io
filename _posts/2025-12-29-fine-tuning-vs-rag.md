---
layout: article
title: Fine-tuning vs RAG – Khi nào dùng gì?
tags: [ai, fine-tuning, rag, llm, machine-learning]
---
Khi muốn LLM "biết" về dữ liệu của bạn, có hai hướng chính: **Fine-tuning** (train lại model) và **RAG** (cung cấp ngữ cảnh runtime). Đây là câu hỏi thực tế mà bất kỳ ai xây dựng AI application cũng phải đối mặt.

## 1. Fine-tuning là gì?

Fine-tuning là quá trình **tiếp tục train** một model đã pre-trained trên tập dữ liệu của bạn. Model "học thuộc" kiến thức mới vào weights của nó.

```
[Pre-trained LLM]
      ↓
[Training trên dữ liệu của bạn]
      ↓
[Fine-tuned Model]  ← Kiến thức nằm bên TRONG model
```

**Ví dụ**: Fine-tune GPT-3.5 với 1000 cặp Q&A về chính sách công ty → Model trả lời chính xác mà không cần cung cấp context.

## 2. RAG là gì?

RAG (Retrieval Augmented Generation) **không thay đổi model** — thay vào đó, mỗi lần user hỏi, hệ thống tìm kiếm tài liệu liên quan và đưa vào prompt:

```
[LLM không đổi]
      ↑
[Prompt = Câu hỏi + Tài liệu liên quan]  ← Kiến thức ở NGOÀI model
      ↑
[Vector DB tìm kiếm]
```

## 3. So sánh trực tiếp

| Tiêu chí | Fine-tuning | RAG |
|---------|-------------|-----|
| **Chi phí ban đầu** | Cao (training cost) | Thấp |
| **Chi phí vận hành** | Thấp (inference nhanh) | Trung bình (thêm embedding + search) |
| **Tốc độ cập nhật dữ liệu** | Chậm (phải train lại) | Nhanh (chỉ cần update vector DB) |
| **Độ chính xác nguồn** | Khó trace | Dễ cite nguồn |
| **Hallucination** | Vẫn có thể xảy ra | Giảm đáng kể (có ngữ cảnh thực) |
| **Kiến thức tối đa** | Bị giới hạn bởi training data | Không giới hạn (thêm tài liệu bất cứ lúc nào) |
| **Style/Format** | Rất tốt (học được tone) | Phụ thuộc vào base model |
| **Độ phức tạp setup** | Cao | Trung bình |

## 4. Khi nào dùng Fine-tuning?

### Phù hợp

**Style và format đặc biệt**: Khi cần model viết theo tone riêng (thương hiệu, chuyên ngành y tế, pháp lý) mà prompt engineering không đủ.

```
❌ RAG: "Hãy viết như nhân viên ngân hàng chuyên nghiệp..."
          (phải nhắc mỗi lần)

✅ Fine-tuning: Model đã học luôn tone ngân hàng chuyên nghiệp
```

**Tác vụ có cấu trúc cố định**: Phân loại văn bản, extract thông tin, nhận dạng ý định (intent).

```python
# Ví dụ dữ liệu fine-tuning để phân loại intent
{
  "messages": [
    {"role": "user", "content": "Tôi muốn hủy đơn hàng"},
    {"role": "assistant", "content": "INTENT: cancel_order"}
  ]
}
```

**Kiến thức ngầm định**: Quy tắc nghiệp vụ phức tạp khó diễn giải bằng text.

**Giảm chi phí inference**: Fine-tune model nhỏ hơn (GPT-3.5) để thay thế model lớn (GPT-4) cho tác vụ cụ thể → rẻ hơn 10-50x.

### Không phù hợp

- Dữ liệu thay đổi thường xuyên (sản phẩm, giá, chính sách)
- Cần cite nguồn tài liệu cụ thể
- Dữ liệu nhạy cảm không muốn đưa vào training
- Ít dữ liệu (< 50-100 examples)

## 5. Khi nào dùng RAG?

### Phù hợp

**Knowledge base doanh nghiệp**: FAQ, policy, tài liệu kỹ thuật — thường xuyên thay đổi.

**Cần cite nguồn**: Legal, y tế, tài chính — user cần biết thông tin lấy từ đâu.

**Dữ liệu lớn và đa dạng**: Không thể đưa vào context window một lần, không thể train hết.

**Triển khai nhanh**: Không cần training, chỉ cần index tài liệu là dùng được.

### Không phù hợp

- Cần học style/format đặc biệt
- Dữ liệu mang tính procedural (cách làm một việc gì đó) hơn là factual
- Tốc độ response cực kỳ quan trọng (thêm latency do search)

## 6. Kết hợp cả hai (Hybrid)

Trong thực tế, thường kết hợp cả hai:

```
Fine-tuned Model (học style + domain knowledge)
          ↑
RAG (cung cấp thông tin cụ thể, cập nhật)
          ↑
User question
```

**Ví dụ thực tế**: Chatbot y tế
- **Fine-tune**: Model học terminology y khoa, tone chuyên nghiệp, biết từ chối ngoài phạm vi
- **RAG**: Tìm kiếm phác đồ điều trị, thông tin thuốc cập nhật nhất

## 7. Chi phí Fine-tuning thực tế

| Model | Giá train (per 1M tokens) | Ghi chú |
|-------|--------------------------|---------|
| GPT-4o mini | $3 | Rẻ nhất OpenAI |
| GPT-3.5 Turbo | $8 | |
| GPT-4o | $25 | |
| Gemini 1.5 Flash | Liên hệ | |
| Llama 3 (tự host) | Chi phí GPU | Open source |

**Ví dụ**: Fine-tune với 1000 examples × 500 tokens = 500K tokens → ~$1.5 với GPT-4o mini.

## 8. Quyết định nhanh

```
Dữ liệu thay đổi thường? → RAG
Cần cite nguồn?          → RAG
Ít dữ liệu (<100 mẫu)?  → RAG (hoặc prompt engineering)

Cần style đặc biệt?      → Fine-tuning
Giảm chi phí inference?  → Fine-tuning
Tác vụ phân loại/extract → Fine-tuning

Muốn tốt nhất?           → Kết hợp cả hai
```

## 9. Kết luận

- **RAG**: Lựa chọn mặc định cho hầu hết use case — linh hoạt, rẻ, dễ cập nhật
- **Fine-tuning**: Khi cần style riêng, tác vụ cấu trúc, hoặc giảm chi phí ở scale lớn
- **Hybrid**: Production-grade AI systems thường dùng cả hai

Đừng fine-tune khi RAG là đủ — tiết kiệm thời gian, tiền bạc và complexity.
