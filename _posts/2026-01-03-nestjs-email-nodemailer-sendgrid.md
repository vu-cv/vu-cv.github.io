---
layout: article
title: NestJS – Gửi Email với Nodemailer & SendGrid
tags: [nestjs, email, nodemailer, sendgrid, nodejs]
---
Gửi email là tính năng không thể thiếu trong ứng dụng web: xác nhận đăng ký, reset mật khẩu, thông báo đơn hàng. Bài này hướng dẫn hai cách phổ biến: **Nodemailer** (SMTP linh hoạt) và **SendGrid** (dịch vụ email chuyên nghiệp).

## 1. Nodemailer — Gửi qua SMTP

### Cài đặt

```bash
npm install nodemailer
npm install -D @types/nodemailer
```

### Tạo MailService

```typescript
// src/mail/mail.service.ts
import { Injectable, Logger } from '@nestjs/common';
import * as nodemailer from 'nodemailer';
import { Transporter } from 'nodemailer';

export interface MailOptions {
  to: string | string[];
  subject: string;
  html: string;
  text?: string;
  attachments?: Array<{ filename: string; path: string }>;
}

@Injectable()
export class MailService {
  private transporter: Transporter;
  private readonly logger = new Logger(MailService.name);

  constructor() {
    this.transporter = nodemailer.createTransport({
      host: process.env.SMTP_HOST,         // smtp.gmail.com
      port: Number(process.env.SMTP_PORT), // 587
      secure: false,                        // true cho port 465
      auth: {
        user: process.env.SMTP_USER,        // your@gmail.com
        pass: process.env.SMTP_PASS,        // App Password (không phải mật khẩu Gmail)
      },
    });
  }

  async sendMail(options: MailOptions): Promise<void> {
    try {
      await this.transporter.sendMail({
        from: `"${process.env.MAIL_FROM_NAME}" <${process.env.SMTP_USER}>`,
        ...options,
      });
      this.logger.log(`Email sent to ${options.to}`);
    } catch (error) {
      this.logger.error(`Failed to send email: ${error.message}`);
      throw error;
    }
  }
}
```

### MailModule

```typescript
// src/mail/mail.module.ts
import { Module, Global } from '@nestjs/common';
import { MailService } from './mail.service';

@Global()  // Global để dùng ở bất kỳ module nào mà không cần import
@Module({
  providers: [MailService],
  exports: [MailService],
})
export class MailModule {}
```

## 2. Template Email với Handlebars

```bash
npm install handlebars
```

```typescript
// src/mail/mail.service.ts
import * as Handlebars from 'handlebars';
import * as fs from 'fs';
import * as path from 'path';

@Injectable()
export class MailService {
  // ...

  private compileTemplate(templateName: string, context: Record<string, any>): string {
    const templatePath = path.join(__dirname, 'templates', `${templateName}.hbs`);
    const templateSource = fs.readFileSync(templatePath, 'utf-8');
    const template = Handlebars.compile(templateSource);
    return template(context);
  }

  async sendWelcomeEmail(to: string, name: string): Promise<void> {
    const html = this.compileTemplate('welcome', { name, year: new Date().getFullYear() });
    await this.sendMail({
      to,
      subject: `Chào mừng ${name} đến với ShopXYZ!`,
      html,
    });
  }

  async sendPasswordReset(to: string, resetToken: string): Promise<void> {
    const resetUrl = `${process.env.FRONTEND_URL}/reset-password?token=${resetToken}`;
    const html = this.compileTemplate('password-reset', { resetUrl });
    await this.sendMail({
      to,
      subject: 'Đặt lại mật khẩu của bạn',
      html,
    });
  }

  async sendOrderConfirmation(to: string, order: any): Promise<void> {
    const html = this.compileTemplate('order-confirmation', { order });
    await this.sendMail({
      to,
      subject: `Xác nhận đơn hàng #${order.id}`,
      html,
    });
  }
}
```

```html
<!-- src/mail/templates/welcome.hbs -->
<!DOCTYPE html>
<html>
<body style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto;">
  <h2>Chào mừng, {{name}}! 🎉</h2>
  <p>Cảm ơn bạn đã đăng ký tài khoản tại ShopXYZ.</p>
  <a href="{{shopUrl}}" style="
    background: #007bff;
    color: white;
    padding: 12px 24px;
    border-radius: 4px;
    text-decoration: none;
    display: inline-block;
    margin-top: 16px;
  ">Bắt đầu mua sắm</a>
  <p style="color: #666; font-size: 12px; margin-top: 32px;">
    © {{year}} ShopXYZ. All rights reserved.
  </p>
</body>
</html>
```

## 3. SendGrid — Dịch vụ email chuyên nghiệp

SendGrid cung cấp deliverability cao, analytics, và không cần cấu hình SMTP.

```bash
npm install @sendgrid/mail
```

```typescript
// src/mail/sendgrid.service.ts
import { Injectable, Logger } from '@nestjs/common';
import * as sgMail from '@sendgrid/mail';

@Injectable()
export class SendGridService {
  private readonly logger = new Logger(SendGridService.name);

  constructor() {
    sgMail.setApiKey(process.env.SENDGRID_API_KEY!);
  }

  async sendMail(options: {
    to: string | string[];
    subject: string;
    html: string;
    text?: string;
  }): Promise<void> {
    try {
      await sgMail.send({
        from: { email: process.env.SENDGRID_FROM_EMAIL!, name: 'ShopXYZ' },
        ...options,
      });
    } catch (error) {
      this.logger.error('SendGrid error:', error.response?.body ?? error.message);
      throw error;
    }
  }

  // Dùng Dynamic Template (thiết kế sẵn trên SendGrid dashboard)
  async sendWithTemplate(
    to: string,
    templateId: string,
    dynamicData: Record<string, any>,
  ): Promise<void> {
    await sgMail.send({
      to,
      from: process.env.SENDGRID_FROM_EMAIL!,
      templateId,
      dynamicTemplateData: dynamicData,
    });
  }
}
```

### Dynamic Template với SendGrid

```typescript
// Gửi email dùng template thiết kế sẵn trên SendGrid
await this.sendGridService.sendWithTemplate(
  user.email,
  'd-abc123xyz',  // Template ID từ SendGrid dashboard
  {
    name: user.name,
    order_id: order.id,
    items: order.items,
    total: order.total,
    tracking_url: `https://shop.example.com/track/${order.id}`,
  },
);
```

## 4. Queue Email — Không block request

Gửi email nên bất đồng bộ để không làm chậm API response:

```typescript
// Dùng BullMQ (xem bài NestJS Queue BullMQ)
@Injectable()
export class UsersService {
  constructor(
    @InjectQueue('mail') private mailQueue: Queue,
  ) {}

  async register(dto: RegisterDto) {
    const user = await this.usersModel.create(dto);

    // Đẩy vào queue — không đợi gửi xong
    await this.mailQueue.add('welcome', {
      to: user.email,
      name: user.name,
    });

    return user;
  }
}

// Worker xử lý queue
@Processor('mail')
export class MailProcessor extends WorkerHost {
  constructor(private mailService: MailService) { super(); }

  async process(job: Job) {
    if (job.name === 'welcome') {
      await this.mailService.sendWelcomeEmail(job.data.to, job.data.name);
    }
  }
}
```

## 5. Retry khi gửi thất bại

```typescript
async sendMailWithRetry(options: MailOptions, maxRetries = 3): Promise<void> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      await this.sendMail(options);
      return;
    } catch (error) {
      if (attempt === maxRetries) throw error;
      const delay = attempt * 2000; // Exponential backoff: 2s, 4s, 6s
      this.logger.warn(`Retry ${attempt}/${maxRetries} after ${delay}ms`);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

## 6. Biến môi trường

```env
# Nodemailer (Gmail)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your@gmail.com
SMTP_PASS=abcd-efgh-ijkl-mnop   # Google App Password

# SendGrid
SENDGRID_API_KEY=SG.xxxxxxxxxxxx
SENDGRID_FROM_EMAIL=noreply@shopxyz.com

MAIL_FROM_NAME=ShopXYZ
FRONTEND_URL=https://shopxyz.com
```

> **Lưu ý**: Gmail cần bật 2FA và tạo **App Password** tại myaccount.google.com → Security → App passwords.

## 7. Kết luận

- **Nodemailer**: Linh hoạt, dùng bất kỳ SMTP nào (Gmail, Mailgun, AWS SES)
- **SendGrid**: Deliverability cao, dashboard analytics, Dynamic Template mạnh
- **Handlebars template**: Tách biệt logic và HTML email
- **Queue**: Bắt buộc khi gửi email hàng loạt — không block API response
- **Retry**: Xử lý trường hợp SMTP tạm thời lỗi

Trong production, ưu tiên dùng SendGrid hoặc AWS SES thay vì Gmail SMTP.
