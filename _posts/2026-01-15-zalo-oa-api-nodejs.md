---
layout: article
title: Zalo OA API – Gửi tin nhắn tự động với NodeJS
tags: [zalo, oa, api, nodejs, chatbot, messaging]
---
Zalo Official Account (OA) API cho phép doanh nghiệp gửi tin nhắn tự động đến khách hàng qua Zalo — kênh messaging phổ biến nhất Việt Nam với 70+ triệu người dùng. Bài này hướng dẫn integrate Zalo OA API vào NestJS.

## 1. Thiết lập Zalo OA

1. Đăng ký Official Account tại [oa.zalo.me](https://oa.zalo.me)
2. Vào **Zalo for Developers** → Tạo ứng dụng
3. Lấy: `App ID`, `Secret Key`, `OA Access Token`
4. Cấu hình Webhook URL (nhận tin nhắn từ user)

## 2. Authentication — Lấy Access Token

```typescript
// src/zalo/zalo-auth.service.ts
import { Injectable, Logger } from '@nestjs/common';
import axios from 'axios';

@Injectable()
export class ZaloAuthService {
  private readonly logger = new Logger(ZaloAuthService.name);
  private accessToken: string = '';
  private tokenExpiry: number = 0;

  // OA Access Token (lấy từ Zalo for Developers console)
  // Token có thời hạn ~90 ngày, cần refresh định kỳ
  getOAToken(): string {
    return process.env.ZALO_OA_ACCESS_TOKEN!;
  }

  // User Access Token — lấy sau khi user cho phép
  async getUserAccessToken(code: string): Promise<string> {
    const res = await axios.post('https://oauth.zaloapp.com/v4/access_token', null, {
      params: {
        app_id: process.env.ZALO_APP_ID,
        app_secret: process.env.ZALO_SECRET_KEY,
        code,
        grant_type: 'authorization_code',
      },
    });
    return res.data.access_token;
  }

  // Lấy profile user từ access token
  async getUserProfile(accessToken: string): Promise<{
    id: string;
    name: string;
    picture: { data: { url: string } };
  }> {
    const res = await axios.get('https://graph.zalo.me/v2.0/me', {
      params: {
        access_token: accessToken,
        fields: 'id,name,picture',
      },
    });
    return res.data;
  }
}
```

## 3. Gửi tin nhắn

```typescript
// src/zalo/zalo-messaging.service.ts
import { Injectable, Logger } from '@nestjs/common';
import axios, { AxiosInstance } from 'axios';

@Injectable()
export class ZaloMessagingService {
  private readonly logger = new Logger(ZaloMessagingService.name);
  private readonly api: AxiosInstance;
  private readonly BASE_URL = 'https://openapi.zalo.me/v3.0/oa';

  constructor(private authService: ZaloAuthService) {
    this.api = axios.create({ baseURL: this.BASE_URL });
  }

  private getHeaders() {
    return {
      access_token: this.authService.getOAToken(),
      'Content-Type': 'application/json',
    };
  }

  // Gửi tin nhắn text đến follower
  async sendTextMessage(userId: string, text: string): Promise<void> {
    await this.api.post('/message/cs', {
      recipient: { user_id: userId },
      message: { text },
    }, { headers: this.getHeaders() });

    this.logger.log(`Sent text to ${userId}`);
  }

  // Gửi tin nhắn với nút bấm
  async sendButtonMessage(userId: string, options: {
    text: string;
    buttons: Array<{ title: string; payload?: string; url?: string }>;
  }): Promise<void> {
    await this.api.post('/message/cs', {
      recipient: { user_id: userId },
      message: {
        attachment: {
          type: 'template',
          payload: {
            template_type: 'request_user_info',
            elements: [{
              title: options.text,
              buttons: options.buttons.map(btn => ({
                title: btn.title,
                type: btn.url ? 'oa.open.url' : 'oa.query.show',
                payload: btn.payload ?? btn.url,
              })),
            }],
          },
        },
      },
    }, { headers: this.getHeaders() });
  }

  // Gửi hình ảnh
  async sendImageMessage(userId: string, imageUrl: string): Promise<void> {
    await this.api.post('/message/cs', {
      recipient: { user_id: userId },
      message: {
        attachment: {
          type: 'template',
          payload: {
            template_type: 'media',
            elements: [{ media_type: 'image', url: imageUrl }],
          },
        },
      },
    }, { headers: this.getHeaders() });
  }

  // Transaction message (ZNS) — gửi cho người chưa follow OA
  async sendTransactionMessage(phone: string, templateId: string, templateData: Record<string, string>): Promise<void> {
    await axios.post('https://business.openapi.zalo.me/message/template', {
      phone,
      template_id: templateId,
      template_data: templateData,
      tracking_id: Date.now().toString(),
    }, {
      headers: {
        access_token: this.authService.getOAToken(),
        'Content-Type': 'application/json',
      },
    });
  }
}
```

## 4. ZNS — Zalo Notification Service

ZNS cho phép gửi tin nhắn đến số điện thoại người dùng (không cần follow OA):

```typescript
// Gửi OTP
async sendOTP(phone: string, otp: string): Promise<void> {
  await this.sendTransactionMessage(phone, process.env.ZALO_OTP_TEMPLATE_ID!, {
    otp,
    expiry: '5 phút',
    app_name: 'ShopXYZ',
  });
}

// Xác nhận đơn hàng
async sendOrderConfirmation(phone: string, order: any): Promise<void> {
  await this.sendTransactionMessage(phone, process.env.ZALO_ORDER_TEMPLATE_ID!, {
    order_id: order.id,
    customer_name: order.customerName,
    total: order.total.toLocaleString('vi-VN') + 'đ',
    delivery_date: order.estimatedDelivery,
  });
}

// Thông báo giao hàng
async sendDeliveryNotification(phone: string, trackingCode: string): Promise<void> {
  await this.sendTransactionMessage(phone, process.env.ZALO_DELIVERY_TEMPLATE_ID!, {
    tracking_code: trackingCode,
    tracking_url: `https://shopxyz.com/track/${trackingCode}`,
  });
}
```

## 5. Webhook — Nhận tin nhắn từ user

```typescript
// src/zalo/zalo-webhook.controller.ts
import { Controller, Post, Get, Body, Query, Res } from '@nestjs/common';
import { Response } from 'express';

@Controller('zalo/webhook')
export class ZaloWebhookController {
  constructor(private messagingService: ZaloMessagingService) {}

  // Verify webhook
  @Get()
  verify(
    @Query('challenge') challenge: string,
    @Res() res: Response,
  ) {
    // Zalo gửi GET request để verify — trả về challenge
    res.send(challenge);
  }

  @Post()
  async handleEvent(@Body() body: any) {
    const { event_name, sender, message } = body;

    switch (event_name) {
      case 'user_send_text':
        await this.handleTextMessage(sender.id, message.text);
        break;

      case 'user_send_image':
        await this.messagingService.sendTextMessage(
          sender.id,
          'Cảm ơn bạn đã gửi ảnh! Chúng tôi sẽ xem xét và phản hồi sớm.',
        );
        break;

      case 'follow':
        await this.handleNewFollower(sender.id, sender.display_name);
        break;

      case 'unfollow':
        // Log hoặc cập nhật DB
        break;
    }

    return { status: 'ok' };
  }

  private async handleTextMessage(userId: string, text: string) {
    const textLower = text.toLowerCase();

    if (textLower.includes('đơn hàng') || textLower.includes('order')) {
      await this.messagingService.sendButtonMessage(userId, {
        text: 'Bạn muốn kiểm tra đơn hàng? Nhấn nút bên dưới:',
        buttons: [
          { title: '🔍 Tra cứu đơn hàng', url: 'https://shopxyz.com/orders' },
          { title: '📞 Liên hệ CSKH', url: 'https://shopxyz.com/contact' },
        ],
      });
    } else if (textLower.includes('xin chào') || textLower === 'hi') {
      await this.messagingService.sendTextMessage(userId,
        'Xin chào! Tôi là trợ lý ShopXYZ. Tôi có thể giúp bạn:\n' +
        '• Tra cứu đơn hàng\n' +
        '• Chính sách đổi trả\n' +
        '• Khuyến mãi hiện tại\n\n' +
        'Bạn cần hỗ trợ gì?'
      );
    } else {
      await this.messagingService.sendTextMessage(userId,
        'Cảm ơn bạn đã liên hệ ShopXYZ! CSKH của chúng tôi sẽ phản hồi trong vòng 15 phút.'
      );
    }
  }

  private async handleNewFollower(userId: string, name: string) {
    await this.messagingService.sendTextMessage(
      userId,
      `Xin chào ${name}! Cảm ơn bạn đã quan tâm ShopXYZ OA.\n\n` +
      'Tại đây bạn sẽ nhận được:\n' +
      '✅ Thông báo đơn hàng\n' +
      '✅ Khuyến mãi độc quyền\n' +
      '✅ Hỗ trợ CSKH nhanh chóng'
    );
  }
}
```

## 6. Tích hợp vào Order Flow

```typescript
// src/orders/orders.service.ts
@Injectable()
export class OrdersService {
  constructor(
    private zaloMessaging: ZaloMessagingService,
  ) {}

  async createOrder(dto: CreateOrderDto, user: User) {
    const order = await this.orderModel.create({ ...dto, userId: user.id });

    // Gửi ZNS nếu có số điện thoại
    if (user.phone) {
      await this.zaloMessaging.sendOrderConfirmation(user.phone, order)
        .catch(err => this.logger.warn('ZNS failed:', err.message));
    }

    return order;
  }

  async updateStatus(orderId: string, status: string) {
    const order = await this.orderModel.findByIdAndUpdate(orderId, { status }, { new: true });
    const user = await this.usersModel.findById(order.userId);

    if (status === 'shipped' && user?.phone) {
      await this.zaloMessaging.sendDeliveryNotification(user.phone, order.trackingCode);
    }

    return order;
  }
}
```

## 7. Biến môi trường

```env
ZALO_APP_ID=123456789
ZALO_SECRET_KEY=your_secret_key
ZALO_OA_ACCESS_TOKEN=your_oa_token
ZALO_OTP_TEMPLATE_ID=template_id_1
ZALO_ORDER_TEMPLATE_ID=template_id_2
ZALO_DELIVERY_TEMPLATE_ID=template_id_3
```

## 8. Kết luận

- **OA Message**: Gửi cho follower — text, button, image
- **ZNS**: Gửi cho bất kỳ số điện thoại Việt Nam — phù hợp OTP, đơn hàng
- **Webhook**: Nhận tin nhắn từ user, xử lý chatbot logic
- **Follow event**: Gửi welcome message khi có follower mới
- ZNS có phí theo gói — cần đăng ký template được Zalo duyệt trước

Zalo OA + ZNS là bộ đôi hiệu quả cho ecommerce Việt Nam: ZNS cho transactional, OA message cho customer service.
