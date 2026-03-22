---
layout: article
title: WhatsApp Business API – Gửi tin nhắn tự động với NodeJS
tags: [whatsapp, api, chatbot, nodejs, messaging]
---
WhatsApp Business API (Cloud API) cho phép doanh nghiệp gửi tin nhắn tự động, thông báo đơn hàng, OTP, và xây dựng chatbot trực tiếp trên WhatsApp — nền tảng nhắn tin phổ biến nhất thế giới.

## 1. Chuẩn bị

### Tạo Meta Business Account

1. Truy cập [developers.facebook.com](https://developers.facebook.com){:target="_blank"} → **My Apps → Create App → Business**
2. Vào **WhatsApp → Set Up**
3. Tạo **WhatsApp Business Account** và **Phone Number**
4. Lấy **Phone Number ID**, **Access Token**, **Business Account ID**

Thêm vào `.env`:
```
WHATSAPP_PHONE_ID=1234567890
WHATSAPP_TOKEN=EAABx...
WHATSAPP_VERIFY_TOKEN=your-webhook-verify-token
```

## 2. Cài đặt

```bash
npm install --save axios
```

## 3. Gửi tin nhắn văn bản

```typescript
import axios from 'axios';

const WA_BASE = 'https://graph.facebook.com/v19.0';
const PHONE_ID = process.env.WHATSAPP_PHONE_ID;
const TOKEN = process.env.WHATSAPP_TOKEN;

async function sendTextMessage(to: string, message: string) {
  await axios.post(
    `${WA_BASE}/${PHONE_ID}/messages`,
    {
      messaging_product: 'whatsapp',
      to,              // Số điện thoại: '84912345678' (không có dấu +)
      type: 'text',
      text: { body: message },
    },
    { headers: { Authorization: `Bearer ${TOKEN}` } },
  );
}

await sendTextMessage('84912345678', 'Xin chào từ hệ thống!');
```

## 4. Gửi Template Message (Thông báo chủ động)

WhatsApp chỉ cho phép gửi tin nhắn tự động đến user khi dùng **template đã được duyệt**:

```typescript
async function sendOrderNotification(to: string, orderInfo: {
  customerName: string;
  orderId: string;
  trackingUrl: string;
}) {
  await axios.post(
    `${WA_BASE}/${PHONE_ID}/messages`,
    {
      messaging_product: 'whatsapp',
      to,
      type: 'template',
      template: {
        name: 'order_shipped',     // Tên template đã đăng ký trên Meta
        language: { code: 'vi' },
        components: [
          {
            type: 'body',
            parameters: [
              { type: 'text', text: orderInfo.customerName },
              { type: 'text', text: orderInfo.orderId },
            ],
          },
          {
            type: 'button',
            sub_type: 'url',
            index: '0',
            parameters: [
              { type: 'text', text: orderInfo.trackingUrl },
            ],
          },
        ],
      },
    },
    { headers: { Authorization: `Bearer ${TOKEN}` } },
  );
}
```

## 5. Gửi Interactive Message (Button, List)

Trong cửa sổ 24h sau khi user nhắn tin, có thể gửi message phong phú:

### Button Message

```typescript
async function sendButtonMessage(to: string) {
  await axios.post(
    `${WA_BASE}/${PHONE_ID}/messages`,
    {
      messaging_product: 'whatsapp',
      to,
      type: 'interactive',
      interactive: {
        type: 'button',
        body: { text: 'Bạn muốn làm gì tiếp theo?' },
        action: {
          buttons: [
            { type: 'reply', reply: { id: 'track_order', title: 'Theo dõi đơn hàng' } },
            { type: 'reply', reply: { id: 'contact_support', title: 'Liên hệ hỗ trợ' } },
            { type: 'reply', reply: { id: 'view_products', title: 'Xem sản phẩm' } },
          ],
        },
      },
    },
    { headers: { Authorization: `Bearer ${TOKEN}` } },
  );
}
```

### List Message

```typescript
async function sendListMessage(to: string) {
  await axios.post(
    `${WA_BASE}/${PHONE_ID}/messages`,
    {
      messaging_product: 'whatsapp',
      to,
      type: 'interactive',
      interactive: {
        type: 'list',
        body: { text: 'Chọn danh mục sản phẩm bạn cần:' },
        action: {
          button: 'Chọn danh mục',
          sections: [
            {
              title: 'Điện tử',
              rows: [
                { id: 'phones', title: 'Điện thoại', description: 'iPhone, Samsung, Xiaomi...' },
                { id: 'laptops', title: 'Laptop', description: 'MacBook, Dell, Lenovo...' },
              ],
            },
            {
              title: 'Thời trang',
              rows: [
                { id: 'shirts', title: 'Áo', description: 'Áo thun, sơ mi...' },
                { id: 'shoes', title: 'Giày', description: 'Nike, Adidas...' },
              ],
            },
          ],
        },
      },
    },
    { headers: { Authorization: `Bearer ${TOKEN}` } },
  );
}
```

## 6. Gửi Media (Ảnh, PDF, Video)

```typescript
async function sendImage(to: string, imageUrl: string, caption?: string) {
  await axios.post(
    `${WA_BASE}/${PHONE_ID}/messages`,
    {
      messaging_product: 'whatsapp',
      to,
      type: 'image',
      image: { link: imageUrl, caption },
    },
    { headers: { Authorization: `Bearer ${TOKEN}` } },
  );
}

async function sendDocument(to: string, pdfUrl: string, filename: string) {
  await axios.post(
    `${WA_BASE}/${PHONE_ID}/messages`,
    {
      messaging_product: 'whatsapp',
      to,
      type: 'document',
      document: { link: pdfUrl, filename, caption: 'Hóa đơn đơn hàng của bạn' },
    },
    { headers: { Authorization: `Bearer ${TOKEN}` } },
  );
}
```

## 7. Webhook — Nhận tin nhắn từ user

```typescript
// NestJS Controller
@Controller('webhook/whatsapp')
export class WhatsappWebhookController {
  // Xác thực webhook (GET)
  @Get()
  verify(@Query() q: any, @Res() res: Response) {
    if (q['hub.verify_token'] === process.env.WHATSAPP_VERIFY_TOKEN) {
      return res.send(q['hub.challenge']);
    }
    return res.status(403).send('Forbidden');
  }

  // Nhận event (POST)
  @Post()
  async receive(@Body() body: any, @Res() res: Response) {
    res.sendStatus(200); // Luôn trả 200 ngay lập tức

    const entry = body.entry?.[0];
    const changes = entry?.changes?.[0];
    const value = changes?.value;

    if (value?.messages?.[0]) {
      await this.handleMessage(value.messages[0], value.metadata.phone_number_id);
    }
  }

  private async handleMessage(message: any, phoneNumberId: string) {
    const from = message.from;

    if (message.type === 'text') {
      const text = message.text.body.toLowerCase();
      if (text.includes('đơn hàng')) {
        await sendButtonMessage(from);
      } else {
        await sendTextMessage(from, `Bạn nhắn: "${message.text.body}"`);
      }
    } else if (message.type === 'interactive') {
      const buttonId = message.interactive?.button_reply?.id
        ?? message.interactive?.list_reply?.id;

      switch (buttonId) {
        case 'track_order':
          await sendTextMessage(from, 'Vui lòng nhập mã đơn hàng của bạn:');
          break;
        case 'contact_support':
          await sendTextMessage(from, 'Nhân viên sẽ liên hệ bạn trong vòng 30 phút.');
          break;
      }
    }
  }
}
```

## 8. Gửi OTP qua WhatsApp

```typescript
async function sendOtp(to: string, otp: string) {
  // Dùng template 'authentication' được Meta cung cấp sẵn
  await axios.post(
    `${WA_BASE}/${PHONE_ID}/messages`,
    {
      messaging_product: 'whatsapp',
      to,
      type: 'template',
      template: {
        name: 'authentication',
        language: { code: 'vi' },
        components: [
          {
            type: 'body',
            parameters: [{ type: 'text', text: otp }],
          },
          {
            type: 'button',
            sub_type: 'otp',
            index: '0',
            parameters: [{ type: 'payload', payload: otp }],
          },
        ],
      },
    },
    { headers: { Authorization: `Bearer ${TOKEN}` } },
  );
}
```

## 9. Kết luận

WhatsApp Business API (Cloud API) mạnh mẽ hơn Messenger API nhiều vì:

- **Template messages**: Gửi thông báo chủ động (OTP, đơn hàng, nhắc nhở)
- **Interactive messages**: Button, List phong phú trong 24h window
- **Media**: Ảnh, PDF, video, audio, location
- **Webhook**: Nhận và xử lý tin nhắn real-time

Đây là kênh liên lạc với tỉ lệ mở cao nhất hiện nay (>95%), lý tưởng cho CSKH, OTP, và chatbot bán hàng.
