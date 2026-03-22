---
layout: article
title: AI – Multi-modal: Phân tích hình ảnh với OpenAI Vision & Gemini
tags: [ai, multimodal, vision, openai, gemini, nodejs, nestjs]
---
Multi-modal AI cho phép model xử lý cả văn bản lẫn hình ảnh — nhận diện sản phẩm từ ảnh, phân tích hóa đơn, đọc tài liệu scan, mô tả ảnh. GPT-4o và Gemini đều hỗ trợ mạnh.

## 1. OpenAI Vision — Phân tích ảnh

```typescript
// src/ai/vision.service.ts
import { Injectable } from '@nestjs/common';
import OpenAI from 'openai';
import * as fs from 'fs';

@Injectable()
export class VisionService {
  private openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

  // Phân tích ảnh từ URL
  async analyzeImageUrl(imageUrl: string, prompt: string): Promise<string> {
    const response = await this.openai.chat.completions.create({
      model: 'gpt-4o',
      messages: [
        {
          role: 'user',
          content: [
            {
              type: 'image_url',
              image_url: { url: imageUrl, detail: 'high' }, // 'low' | 'high' | 'auto'
            },
            { type: 'text', text: prompt },
          ],
        },
      ],
      max_tokens: 1000,
    });

    return response.choices[0].message.content ?? '';
  }

  // Phân tích ảnh từ file (base64)
  async analyzeImageFile(filePath: string, prompt: string): Promise<string> {
    const imageBuffer = fs.readFileSync(filePath);
    const base64Image = imageBuffer.toString('base64');
    const mimeType = this.getMimeType(filePath);

    const response = await this.openai.chat.completions.create({
      model: 'gpt-4o',
      messages: [
        {
          role: 'user',
          content: [
            {
              type: 'image_url',
              image_url: {
                url: `data:${mimeType};base64,${base64Image}`,
              },
            },
            { type: 'text', text: prompt },
          ],
        },
      ],
    });

    return response.choices[0].message.content ?? '';
  }

  // Phân tích buffer (từ upload)
  async analyzeImageBuffer(buffer: Buffer, mimeType: string, prompt: string): Promise<string> {
    const base64 = buffer.toString('base64');

    const response = await this.openai.chat.completions.create({
      model: 'gpt-4o',
      messages: [{
        role: 'user',
        content: [
          {
            type: 'image_url',
            image_url: { url: `data:${mimeType};base64,${base64}` },
          },
          { type: 'text', text: prompt },
        ],
      }],
    });

    return response.choices[0].message.content ?? '';
  }

  // So sánh nhiều ảnh
  async compareImages(imageUrls: string[], prompt: string): Promise<string> {
    const imageContent = imageUrls.map(url => ({
      type: 'image_url' as const,
      image_url: { url },
    }));

    const response = await this.openai.chat.completions.create({
      model: 'gpt-4o',
      messages: [{
        role: 'user',
        content: [
          ...imageContent,
          { type: 'text', text: prompt },
        ],
      }],
    });

    return response.choices[0].message.content ?? '';
  }

  private getMimeType(filePath: string): string {
    const ext = filePath.split('.').pop()?.toLowerCase();
    const map: Record<string, string> = {
      jpg: 'image/jpeg', jpeg: 'image/jpeg',
      png: 'image/png', gif: 'image/gif',
      webp: 'image/webp',
    };
    return map[ext ?? ''] ?? 'image/jpeg';
  }
}
```

## 2. Use Cases thực tế

```typescript
// src/ai/product-vision.service.ts
@Injectable()
export class ProductVisionService {
  constructor(private visionService: VisionService) {}

  // Nhận diện sản phẩm từ ảnh
  async identifyProduct(imageUrl: string): Promise<{
    name: string;
    category: string;
    attributes: Record<string, string>;
    suggestedPrice?: string;
  }> {
    const result = await this.visionService.analyzeImageUrl(imageUrl, `
      Phân tích hình ảnh sản phẩm này. Trả về JSON:
      {
        "name": "Tên sản phẩm",
        "category": "Danh mục",
        "brand": "Thương hiệu (nếu nhận ra)",
        "color": "Màu sắc",
        "condition": "Mới/Cũ/Like new",
        "attributes": { "key": "value" },
        "suggestedTags": ["tag1", "tag2"]
      }
      Chỉ trả về JSON, không giải thích thêm.
    `);

    return JSON.parse(result);
  }

  // Đọc hóa đơn, receipt
  async extractReceipt(imageBuffer: Buffer): Promise<{
    merchant: string;
    date: string;
    items: Array<{ name: string; quantity: number; price: number }>;
    total: number;
    tax?: number;
  }> {
    const result = await this.visionService.analyzeImageBuffer(
      imageBuffer,
      'image/jpeg',
      `Đọc hóa đơn trong ảnh và trả về JSON:
      {
        "merchant": "Tên cửa hàng",
        "date": "YYYY-MM-DD",
        "items": [{"name": "...", "quantity": 1, "price": 0}],
        "total": 0,
        "tax": 0
      }
      Chỉ trả về JSON.`
    );

    return JSON.parse(result);
  }

  // Kiểm tra chất lượng ảnh sản phẩm (cho seller upload)
  async checkProductImageQuality(imageUrl: string): Promise<{
    score: number;       // 1-10
    issues: string[];
    suggestions: string[];
    approved: boolean;
  }> {
    const result = await this.visionService.analyzeImageUrl(imageUrl, `
      Đánh giá chất lượng ảnh sản phẩm này cho e-commerce. Trả về JSON:
      {
        "score": 8,
        "issues": ["Nền không trắng", "Hơi tối"],
        "suggestions": ["Dùng nền trắng", "Tăng độ sáng"],
        "approved": true
      }
      Tiêu chí: nền sạch, đủ sáng, sản phẩm rõ ràng, không blur, không watermark.
      Approved khi score >= 6.
    `);

    return JSON.parse(result);
  }

  // Mô tả ảnh tự động (accessibility)
  async generateAltText(imageUrl: string): Promise<string> {
    return this.visionService.analyzeImageUrl(imageUrl,
      'Viết mô tả ngắn gọn (1-2 câu) cho ảnh này để làm alt text, bằng tiếng Việt.'
    );
  }
}
```

## 3. Gemini Vision

```bash
npm install @google/generative-ai
```

```typescript
import { GoogleGenerativeAI } from '@google/generative-ai';
import * as fs from 'fs';

@Injectable()
export class GeminiVisionService {
  private genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);
  private model = this.genAI.getGenerativeModel({ model: 'gemini-1.5-flash' });

  // Phân tích ảnh từ URL (Gemini cần download trước)
  async analyzeFromBuffer(buffer: Buffer, mimeType: string, prompt: string): Promise<string> {
    const result = await this.model.generateContent([
      { inlineData: { data: buffer.toString('base64'), mimeType } },
      prompt,
    ]);

    return result.response.text();
  }

  // Phân tích file local
  async analyzeLocalFile(filePath: string, prompt: string): Promise<string> {
    const buffer = fs.readFileSync(filePath);
    const mimeType = 'image/jpeg';
    return this.analyzeFromBuffer(buffer, mimeType, prompt);
  }

  // Video analysis (Gemini hỗ trợ video)
  async analyzeVideo(videoBuffer: Buffer, prompt: string): Promise<string> {
    const result = await this.model.generateContent([
      { inlineData: { data: videoBuffer.toString('base64'), mimeType: 'video/mp4' } },
      prompt,
    ]);
    return result.response.text();
  }
}
```

## 4. Controller — Upload và phân tích ảnh

```typescript
// src/products/products.controller.ts
@Controller('products')
export class ProductsController {
  constructor(private productVisionService: ProductVisionService) {}

  @Post('analyze-image')
  @UseInterceptors(FileInterceptor('image'))
  async analyzeProductImage(
    @UploadedFile() file: Express.Multer.File,
  ) {
    // 1. Upload lên S3
    const imageUrl = await this.s3Service.upload(file);

    // 2. Phân tích với AI
    const [productInfo, qualityCheck] = await Promise.all([
      this.productVisionService.identifyProduct(imageUrl),
      this.productVisionService.checkProductImageQuality(imageUrl),
    ]);

    return {
      imageUrl,
      productInfo,
      qualityCheck,
    };
  }

  @Post('extract-receipt')
  @UseInterceptors(FileInterceptor('receipt'))
  async extractReceipt(@UploadedFile() file: Express.Multer.File) {
    return this.productVisionService.extractReceipt(file.buffer);
  }
}
```

## 5. NextJS — Camera/Upload UI

```tsx
'use client';
import { useState, useRef } from 'react';

export function ImageAnalyzer() {
  const [result, setResult] = useState<any>(null);
  const [loading, setLoading] = useState(false);
  const fileRef = useRef<HTMLInputElement>(null);

  async function analyzeImage(file: File) {
    setLoading(true);
    const formData = new FormData();
    formData.append('image', file);

    const res = await fetch('/api/products/analyze-image', {
      method: 'POST',
      body: formData,
    });
    const data = await res.json();
    setResult(data);
    setLoading(false);
  }

  return (
    <div>
      <input
        ref={fileRef}
        type="file"
        accept="image/*"
        capture="environment"  // Mobile camera
        onChange={e => e.target.files?.[0] && analyzeImage(e.target.files[0])}
        className="hidden"
      />
      <button onClick={() => fileRef.current?.click()}>
        📷 Chụp ảnh sản phẩm
      </button>

      {loading && <p>Đang phân tích ảnh...</p>}

      {result && (
        <div>
          <h3>{result.productInfo.name}</h3>
          <p>Danh mục: {result.productInfo.category}</p>
          <p>Chất lượng ảnh: {result.qualityCheck.score}/10</p>
          {!result.qualityCheck.approved && (
            <ul className="text-red-500">
              {result.qualityCheck.issues.map((i: string) => <li key={i}>{i}</li>)}
            </ul>
          )}
        </div>
      )}
    </div>
  );
}
```

## 6. Kết luận

- **GPT-4o**: Tốt nhất cho hiểu nội dung phức tạp, OCR, phân tích chi tiết
- **Gemini Flash**: Nhanh và rẻ hơn, hỗ trợ video
- **Base64 vs URL**: URL tiện lợi hơn nhưng phải publicly accessible; base64 cho private files
- **`detail: 'high'`**: Xử lý ảnh độ phân giải cao hơn (tốn token hơn)
- **JSON output**: Luôn yêu cầu model trả về JSON để parse dễ — kết hợp với `response_format: { type: 'json_object' }`

Multi-modal mở ra nhiều use case mới: tự động tag sản phẩm, kiểm duyệt ảnh, đọc tài liệu scan — giảm manual work đáng kể.
