---
layout: article
title: Stripe – Tích hợp thanh toán vào NestJS & NextJS
tags: [stripe, payment, nestjs, nextjs, nodejs, ecommerce]
---
Stripe là cổng thanh toán phổ biến nhất cho developer — API rõ ràng, sandbox mạnh, hỗ trợ thẻ quốc tế, và webhook đáng tin cậy. Bài này hướng dẫn implement thanh toán end-to-end.

## 1. Cài đặt

```bash
# Backend (NestJS)
npm install stripe

# Frontend (NextJS)
npm install @stripe/stripe-js @stripe/react-stripe-js
```

## 2. NestJS — Payment Service

```typescript
// src/payments/stripe.service.ts
import { Injectable, Logger } from '@nestjs/common';
import Stripe from 'stripe';

@Injectable()
export class StripeService {
  private stripe: Stripe;
  private readonly logger = new Logger(StripeService.name);

  constructor() {
    this.stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
      apiVersion: '2024-12-18.acacia',
    });
  }

  // Tạo Payment Intent — phương thức chính
  async createPaymentIntent(
    amount: number,        // VND không hỗ trợ — dùng USD (cents)
    currency: string = 'usd',
    metadata?: Record<string, string>,
  ): Promise<Stripe.PaymentIntent> {
    return this.stripe.paymentIntents.create({
      amount,    // 1000 = $10.00
      currency,
      automatic_payment_methods: { enabled: true },
      metadata,
    });
  }

  // Tạo Checkout Session (hosted page của Stripe)
  async createCheckoutSession(order: {
    orderId: string;
    items: Array<{ name: string; price: number; quantity: number; image?: string }>;
    successUrl: string;
    cancelUrl: string;
    customerEmail?: string;
  }): Promise<Stripe.Checkout.Session> {
    return this.stripe.checkout.sessions.create({
      mode: 'payment',
      line_items: order.items.map(item => ({
        price_data: {
          currency: 'usd',
          product_data: {
            name: item.name,
            images: item.image ? [item.image] : [],
          },
          unit_amount: item.price, // cents
        },
        quantity: item.quantity,
      })),
      customer_email: order.customerEmail,
      success_url: order.successUrl,
      cancel_url: order.cancelUrl,
      metadata: { orderId: order.orderId },
      expires_at: Math.floor(Date.now() / 1000) + 1800, // Hết hạn sau 30 phút
    });
  }

  // Xác nhận webhook
  constructWebhookEvent(payload: Buffer, signature: string): Stripe.Event {
    return this.stripe.webhooks.constructEvent(
      payload,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET!,
    );
  }

  async retrievePaymentIntent(id: string): Promise<Stripe.PaymentIntent> {
    return this.stripe.paymentIntents.retrieve(id);
  }

  // Hoàn tiền
  async refund(paymentIntentId: string, amount?: number): Promise<Stripe.Refund> {
    return this.stripe.refunds.create({
      payment_intent: paymentIntentId,
      amount, // Undefined = hoàn toàn bộ
    });
  }
}
```

## 3. Payments Controller

```typescript
// src/payments/payments.controller.ts
import { Controller, Post, Body, Headers, Req, RawBodyRequest } from '@nestjs/common';

@Controller('payments')
export class PaymentsController {
  constructor(
    private stripeService: StripeService,
    private ordersService: OrdersService,
  ) {}

  // Tạo checkout session
  @Post('checkout')
  @UseGuards(JwtAuthGuard)
  async createCheckout(
    @Body() body: { orderId: string },
    @CurrentUser() user: User,
  ) {
    const order = await this.ordersService.findById(body.orderId);

    const session = await this.stripeService.createCheckoutSession({
      orderId: order.id,
      items: order.items.map(item => ({
        name: item.product.name,
        price: Math.round(item.product.price * 100), // USD cents
        quantity: item.quantity,
        image: item.product.imageUrl,
      })),
      customerEmail: user.email,
      successUrl: `${process.env.FRONTEND_URL}/orders/${order.id}?payment=success`,
      cancelUrl: `${process.env.FRONTEND_URL}/checkout?payment=cancelled`,
    });

    return { url: session.url, sessionId: session.id };
  }

  // Webhook từ Stripe — KHÔNG dùng JSON middleware, cần raw body
  @Post('webhook')
  async handleWebhook(
    @Headers('stripe-signature') signature: string,
    @Req() req: RawBodyRequest<Request>,
  ) {
    let event: Stripe.Event;

    try {
      event = this.stripeService.constructWebhookEvent(req.rawBody!, signature);
    } catch (err) {
      throw new BadRequestException(`Webhook verification failed: ${err.message}`);
    }

    switch (event.type) {
      case 'checkout.session.completed': {
        const session = event.data.object as Stripe.Checkout.Session;
        await this.ordersService.markAsPaid(
          session.metadata!.orderId,
          session.payment_intent as string,
        );
        break;
      }

      case 'payment_intent.payment_failed': {
        const intent = event.data.object as Stripe.PaymentIntent;
        await this.ordersService.markAsFailed(
          intent.metadata.orderId,
          intent.last_payment_error?.message,
        );
        break;
      }

      case 'charge.refunded': {
        const charge = event.data.object as Stripe.Charge;
        await this.ordersService.markAsRefunded(charge.payment_intent as string);
        break;
      }
    }

    return { received: true };
  }
}
```

### Raw Body cho Webhook

```typescript
// main.ts — Bắt buộc để Stripe webhook hoạt động
async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    rawBody: true,  // Enable raw body
  });
  // ...
}
```

## 4. NextJS — Checkout Flow

```tsx
// app/checkout/page.tsx
'use client';
import { useState } from 'react';
import { loadStripe } from '@stripe/stripe-js';

const stripePromise = loadStripe(process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY!);

export default function CheckoutPage() {
  const [loading, setLoading] = useState(false);

  async function handleCheckout(orderId: string) {
    setLoading(true);
    try {
      const res = await fetch('/api/payments/checkout', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ orderId }),
      });

      const { url } = await res.json();

      // Redirect đến Stripe Checkout page
      window.location.href = url;
    } finally {
      setLoading(false);
    }
  }

  return (
    <button
      onClick={() => handleCheckout('order-123')}
      disabled={loading}
      className="bg-purple-600 text-white px-6 py-3 rounded-lg"
    >
      {loading ? 'Đang xử lý...' : 'Thanh toán ngay'}
    </button>
  );
}
```

### Payment Elements (Custom UI)

```tsx
// app/checkout/payment-form.tsx
'use client';
import { Elements, PaymentElement, useStripe, useElements } from '@stripe/react-stripe-js';

function CheckoutForm({ clientSecret }: { clientSecret: string }) {
  const stripe = useStripe();
  const elements = useElements();
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    if (!stripe || !elements) return;

    setLoading(true);
    const { error } = await stripe.confirmPayment({
      elements,
      confirmParams: {
        return_url: `${window.location.origin}/orders/success`,
      },
    });

    if (error) setError(error.message ?? 'Có lỗi xảy ra');
    setLoading(false);
  }

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <PaymentElement />
      {error && <p className="text-red-500 text-sm">{error}</p>}
      <button type="submit" disabled={loading || !stripe}>
        {loading ? 'Đang xử lý...' : 'Thanh toán'}
      </button>
    </form>
  );
}

export function StripeCheckout({ clientSecret }: { clientSecret: string }) {
  return (
    <Elements stripe={stripePromise} options={{ clientSecret }}>
      <CheckoutForm clientSecret={clientSecret} />
    </Elements>
  );
}
```

## 5. Test với Stripe Test Cards

```
Thẻ thành công:    4242 4242 4242 4242
Thẻ bị từ chối:   4000 0000 0000 0002
Yêu cầu 3DS:       4000 0025 0000 3155
Ngày hết hạn:      Bất kỳ tháng/năm tương lai
CVC:               Bất kỳ 3 số
```

```bash
# Test webhook local
npm install -g stripe
stripe listen --forward-to localhost:3000/api/payments/webhook
```

## 6. Kết luận

- **Checkout Session**: Redirect đến Stripe — đơn giản, không cần xử lý card data
- **Payment Intent + Elements**: Custom UI — phức tạp hơn nhưng UX tốt hơn
- **Webhook**: Bắt buộc để xác nhận thanh toán — không tin vào success URL
- **Raw body**: Cần enable trong NestJS để Stripe signature verification hoạt động
- **Test cards**: Test mọi scenario trước khi go live

Luôn xác nhận thanh toán qua Webhook, không bao giờ tin vào frontend callback.
