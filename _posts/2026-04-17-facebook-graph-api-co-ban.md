---
layout: article
title: Facebook Graph API – Gửi tin nhắn & Quản lý Page
tags: [facebook, graph-api, webhook, messenger, chatbot]
---
Facebook Graph API cho phép lập trình viên tương tác với Facebook Platform: gửi tin nhắn Messenger, quản lý Page, đọc bình luận, v.v. Bài này hướng dẫn các tác vụ thực tế nhất.

## 1. Chuẩn bị

### Tạo Facebook App

1. Truy cập [developers.facebook.com](https://developers.facebook.com){:target="_blank"} → **My Apps → Create App**
2. Chọn loại app: **Business** (cho Messenger Bot) hoặc **Consumer**
3. Vào **Add Products → Messenger → Set Up**

### Lấy Page Access Token

1. Vào **Messenger → Settings → Access Tokens**
2. Chọn Page của bạn → Generate Token
3. Copy **Page Access Token** (dạng `EAABx...`)

Thêm vào `.env`:
```
FB_PAGE_ACCESS_TOKEN=EAABx...
FB_VERIFY_TOKEN=your-webhook-verify-token
FB_APP_SECRET=your-app-secret
```

## 2. Cài đặt

```bash
npm install --save axios
```

## 3. Gửi tin nhắn qua Messenger

### Gửi text message

```typescript
import axios from 'axios';

const BASE_URL = 'https://graph.facebook.com/v19.0';
const PAGE_ACCESS_TOKEN = process.env.FB_PAGE_ACCESS_TOKEN;

async function sendTextMessage(recipientId: string, text: string) {
  await axios.post(
    `${BASE_URL}/me/messages`,
    {
      recipient: { id: recipientId },
      message: { text },
    },
    { params: { access_token: PAGE_ACCESS_TOKEN } },
  );
}

// Sử dụng
await sendTextMessage('12345678', 'Xin chào! Tôi có thể giúp gì cho bạn?');
```

### Gửi tin nhắn với Quick Replies

```typescript
async function sendQuickReplies(recipientId: string) {
  await axios.post(
    `${BASE_URL}/me/messages`,
    {
      recipient: { id: recipientId },
      message: {
        text: 'Bạn muốn làm gì?',
        quick_replies: [
          { content_type: 'text', title: 'Xem sản phẩm', payload: 'VIEW_PRODUCTS' },
          { content_type: 'text', title: 'Liên hệ hỗ trợ', payload: 'CONTACT_SUPPORT' },
          { content_type: 'text', title: 'Theo dõi đơn hàng', payload: 'TRACK_ORDER' },
        ],
      },
    },
    { params: { access_token: PAGE_ACCESS_TOKEN } },
  );
}
```

### Gửi Generic Template (Card)

```typescript
async function sendCard(recipientId: string) {
  await axios.post(
    `${BASE_URL}/me/messages`,
    {
      recipient: { id: recipientId },
      message: {
        attachment: {
          type: 'template',
          payload: {
            template_type: 'generic',
            elements: [
              {
                title: 'iPhone 15 Pro',
                subtitle: 'Chip A17 Pro, Camera 48MP',
                image_url: 'https://example.com/iphone.jpg',
                buttons: [
                  { type: 'web_url', url: 'https://example.com/iphone', title: 'Xem chi tiết' },
                  { type: 'postback', title: 'Mua ngay', payload: 'BUY_IPHONE15' },
                ],
              },
            ],
          },
        },
      },
    },
    { params: { access_token: PAGE_ACCESS_TOKEN } },
  );
}
```

## 4. Webhook — Nhận tin nhắn từ người dùng

Webhook là endpoint server của bạn mà Facebook sẽ gọi khi có event mới.

### Xác thực Webhook (GET)

```typescript
// NestJS Controller
import { Controller, Get, Post, Body, Query, Res, HttpStatus } from '@nestjs/common';
import { Response } from 'express';

@Controller('webhook')
export class WebhookController {
  @Get()
  verifyWebhook(@Query() query: any, @Res() res: Response) {
    const mode = query['hub.mode'];
    const token = query['hub.verify_token'];
    const challenge = query['hub.challenge'];

    if (mode === 'subscribe' && token === process.env.FB_VERIFY_TOKEN) {
      return res.status(HttpStatus.OK).send(challenge);
    }
    return res.status(HttpStatus.FORBIDDEN).send('Verification failed');
  }
```

### Nhận và xử lý Event (POST)

```typescript
  @Post()
  async handleWebhook(@Body() body: any, @Res() res: Response) {
    // Luôn trả 200 ngay lập tức để Facebook không retry
    res.status(HttpStatus.OK).send('EVENT_RECEIVED');

    if (body.object !== 'page') return;

    for (const entry of body.entry) {
      for (const event of entry.messaging) {
        if (event.message) {
          await this.handleMessage(event);
        } else if (event.postback) {
          await this.handlePostback(event);
        }
      }
    }
  }

  private async handleMessage(event: any) {
    const senderId = event.sender.id;
    const text = event.message.text;

    console.log(`Tin nhắn từ ${senderId}: ${text}`);

    // Echo lại tin nhắn
    await sendTextMessage(senderId, `Bạn vừa nhắn: ${text}`);
  }

  private async handlePostback(event: any) {
    const senderId = event.sender.id;
    const payload = event.postback.payload;

    switch (payload) {
      case 'VIEW_PRODUCTS':
        await sendCard(senderId);
        break;
      case 'CONTACT_SUPPORT':
        await sendTextMessage(senderId, 'Vui lòng gọi 1800-xxxx để được hỗ trợ.');
        break;
    }
  }
}
```

## 5. Đọc bình luận trên Page Post

```typescript
async function getPostComments(postId: string) {
  const response = await axios.get(`${BASE_URL}/${postId}/comments`, {
    params: {
      fields: 'id,message,from,created_time',
      access_token: PAGE_ACCESS_TOKEN,
    },
  });
  return response.data.data;
}
```

## 6. Đăng bài lên Page

```typescript
async function publishPost(message: string, imageUrl?: string) {
  const PAGE_ID = process.env.FB_PAGE_ID;

  if (imageUrl) {
    // Đăng kèm ảnh
    await axios.post(`${BASE_URL}/${PAGE_ID}/photos`, {
      url: imageUrl,
      caption: message,
      access_token: PAGE_ACCESS_TOKEN,
    });
  } else {
    await axios.post(`${BASE_URL}/${PAGE_ID}/feed`, {
      message,
      access_token: PAGE_ACCESS_TOKEN,
    });
  }
}
```

## 7. Đăng ký Webhook

Sau khi server đã chạy với HTTPS:

1. Vào **Messenger → Settings → Webhooks**
2. Nhấn **Add Callback URL**
3. Nhập URL webhook: `https://yourdomain.com/webhook`
4. Nhập **Verify Token** (khớp với `FB_VERIFY_TOKEN` trong .env)
5. Chọn events cần nhận: `messages`, `messaging_postbacks`, `messaging_referrals`

> Webhook phải có HTTPS. Dùng [ngrok](https://ngrok.com){:target="_blank"} để test local:
> ```bash
> ngrok http 3000
> # → https://xxxx.ngrok.io/webhook
> ```

## 8. Kết luận

Facebook Graph API cho phép xây dựng nhiều tính năng mạnh mẽ:

- **Messenger Bot**: nhận và gửi tin nhắn tự động với webhook
- **Quick Replies / Templates**: tương tác phong phú
- **Page Management**: đăng bài, đọc bình luận
- **Postback**: xử lý click button trong Messenger

Tận dụng tốt Graph API và bạn có thể xây dựng chatbot bán hàng, CSKH tự động hoàn toàn trên Messenger.
