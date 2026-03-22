---
layout: article
title: NestJS – Xử lý tác vụ nền với BullMQ & Redis
tags: [nestjs, bullmq, redis, queue, background-job]
---
Nhiều tác vụ không nên xử lý trực tiếp trong HTTP request như gửi email, resize ảnh, export báo cáo — vì chúng tốn thời gian và có thể làm timeout. **BullMQ** kết hợp **Redis** giải quyết bài toán này bằng cách đưa tác vụ vào hàng đợi và xử lý nền.

## 1. Cài đặt

```bash
npm install --save @nestjs/bullmq bullmq
```

Cần có Redis đang chạy. Dùng Docker:
```bash
docker run -d -p 6379:6379 redis:alpine
```

## 2. Cấu hình BullMQ Module

```typescript
// app.module.ts
import { BullModule } from '@nestjs/bullmq';

@Module({
  imports: [
    BullModule.forRoot({
      connection: {
        host: process.env.REDIS_HOST ?? 'localhost',
        port: Number(process.env.REDIS_PORT ?? 6379),
      },
    }),
    BullModule.registerQueue({ name: 'email' }),
    BullModule.registerQueue({ name: 'report' }),
  ],
})
export class AppModule {}
```

## 3. Tạo Producer — Đẩy job vào Queue

```typescript
// src/email/email.service.ts
import { Injectable } from '@nestjs/common';
import { InjectQueue } from '@nestjs/bullmq';
import { Queue } from 'bullmq';

@Injectable()
export class EmailService {
  constructor(@InjectQueue('email') private emailQueue: Queue) {}

  async sendWelcomeEmail(userId: string, email: string) {
    await this.emailQueue.add(
      'welcome',          // Tên job
      { userId, email },  // Dữ liệu job
      {
        attempts: 3,      // Retry tối đa 3 lần nếu thất bại
        backoff: {
          type: 'exponential',
          delay: 2000,    // Retry sau 2s, 4s, 8s...
        },
        removeOnComplete: 100, // Giữ lại 100 job hoàn thành gần nhất
        removeOnFail: 50,
      },
    );
  }

  async sendBulkEmail(recipients: string[]) {
    const jobs = recipients.map(email => ({
      name: 'bulk',
      data: { email },
    }));
    // Đẩy nhiều job cùng lúc
    await this.emailQueue.addBulk(jobs);
  }

  async scheduleReminder(userId: string, delayMs: number) {
    await this.emailQueue.add(
      'reminder',
      { userId },
      { delay: delayMs }, // Chạy sau N milliseconds
    );
  }
}
```

## 4. Tạo Consumer — Xử lý job

```typescript
// src/email/email.processor.ts
import { Processor, WorkerHost, OnWorkerEvent } from '@nestjs/bullmq';
import { Job } from 'bullmq';
import { Logger } from '@nestjs/common';

@Processor('email')
export class EmailProcessor extends WorkerHost {
  private readonly logger = new Logger(EmailProcessor.name);

  async process(job: Job): Promise<any> {
    this.logger.log(`Processing job ${job.id} [${job.name}]`);

    switch (job.name) {
      case 'welcome':
        return this.handleWelcome(job);
      case 'bulk':
        return this.handleBulk(job);
      case 'reminder':
        return this.handleReminder(job);
      default:
        throw new Error(`Unknown job: ${job.name}`);
    }
  }

  private async handleWelcome(job: Job<{ userId: string; email: string }>) {
    const { email } = job.data;
    // Gửi email thực tế (nodemailer, SendGrid, AWS SES...)
    this.logger.log(`Sending welcome email to ${email}`);
    // await mailer.send(...)
    return { sent: true, email };
  }

  private async handleBulk(job: Job<{ email: string }>) {
    this.logger.log(`Sending bulk email to ${job.data.email}`);
    // await mailer.send(...)
  }

  private async handleReminder(job: Job<{ userId: string }>) {
    this.logger.log(`Sending reminder to user ${job.data.userId}`);
  }

  @OnWorkerEvent('completed')
  onCompleted(job: Job) {
    this.logger.log(`Job ${job.id} completed`);
  }

  @OnWorkerEvent('failed')
  onFailed(job: Job, error: Error) {
    this.logger.error(`Job ${job.id} failed: ${error.message}`);
  }
}
```

## 5. Đăng ký vào Module

```typescript
// src/email/email.module.ts
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bullmq';
import { EmailService } from './email.service';
import { EmailProcessor } from './email.processor';

@Module({
  imports: [BullModule.registerQueue({ name: 'email' })],
  providers: [EmailService, EmailProcessor],
  exports: [EmailService],
})
export class EmailModule {}
```

## 6. Sử dụng trong Controller

```typescript
@Controller('users')
export class UsersController {
  constructor(
    private readonly usersService: UsersService,
    private readonly emailService: EmailService,
  ) {}

  @Post('register')
  async register(@Body() dto: RegisterDto) {
    const user = await this.usersService.create(dto);

    // Đẩy vào queue — không block HTTP response
    await this.emailService.sendWelcomeEmail(user.id, user.email);

    return { message: 'Đăng ký thành công' };
  }
}
```

## 7. Job với Progress

Theo dõi tiến trình của job dài (ví dụ export file):

```typescript
// Trong processor
private async handleExport(job: Job<{ reportId: string }>) {
  const total = 1000;
  for (let i = 0; i < total; i++) {
    // Xử lý từng record...
    await job.updateProgress(Math.round((i / total) * 100));
  }
  return { file: 'report.xlsx' };
}
```

```typescript
// Trong service — poll progress từ client
async getJobProgress(jobId: string) {
  const job = await this.emailQueue.getJob(jobId);
  return {
    state: await job.getState(), // waiting | active | completed | failed
    progress: job.progress,
    result: job.returnvalue,
  };
}
```

## 8. Cron Job với BullMQ

```typescript
// Chạy lại mỗi ngày lúc 8 giờ sáng
await this.reportQueue.add(
  'daily-report',
  {},
  {
    repeat: { cron: '0 8 * * *' },
    jobId: 'daily-report', // ID cố định tránh tạo trùng
  },
);
```

## 9. Bull Board — Dashboard quản lý Queue

```bash
npm install --save @bull-board/nestjs @bull-board/express
```

```typescript
// app.module.ts
import { BullBoardModule } from '@bull-board/nestjs';
import { BullMQAdapter } from '@bull-board/api/bullMQAdapter';
import { ExpressAdapter } from '@bull-board/express';

BullBoardModule.forRoot({ route: '/queues', adapter: ExpressAdapter }),
BullBoardModule.forFeature({ name: 'email', adapter: BullMQAdapter }),
```

Truy cập `http://localhost:3000/queues` để xem dashboard monitor queue.

## 10. Kết luận

BullMQ + Redis là combo lý tưởng cho background jobs trong NestJS:

- **Producer**: Đẩy job vào queue, không block request
- **Consumer (Processor)**: Xử lý job nền, hỗ trợ retry tự động
- **Delay**: Lên lịch job chạy sau N giây
- **Cron**: Lặp lại theo lịch cố định
- **Bull Board**: Giao diện monitor queue trực quan

Use cases: gửi email/SMS, resize ảnh, export báo cáo, sync dữ liệu, push notification.
