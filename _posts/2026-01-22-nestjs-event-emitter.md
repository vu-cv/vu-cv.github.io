---
layout: article
title: NestJS – Event Emitter & Event-Driven Architecture nội bộ
tags: [nestjs, event-emitter, event-driven, nodejs, architecture]
---
Event Emitter cho phép các module giao tiếp theo kiểu publish/subscribe mà không cần import trực tiếp nhau — giảm coupling, dễ test, dễ mở rộng. Khác với Redis Pub/Sub hay Kafka, Event Emitter chạy **in-process** — phù hợp cho monolith.

## 1. Cài đặt

```bash
npm install @nestjs/event-emitter
```

```typescript
// app.module.ts
import { EventEmitterModule } from '@nestjs/event-emitter';

@Module({
  imports: [
    EventEmitterModule.forRoot({
      wildcard: true,        // Hỗ trợ wildcard: 'user.*'
      delimiter: '.',        // Separator cho namespace
      maxListeners: 20,      // Số listener tối đa
      verboseMemoryLeak: true,
    }),
  ],
})
export class AppModule {}
```

## 2. Định nghĩa Events

```typescript
// src/events/user.events.ts
export class UserCreatedEvent {
  constructor(
    public readonly userId: string,
    public readonly email: string,
    public readonly name: string,
  ) {}
}

export class UserUpdatedEvent {
  constructor(
    public readonly userId: string,
    public readonly changes: Partial<{ name: string; avatar: string }>,
  ) {}
}

export class OrderPlacedEvent {
  constructor(
    public readonly orderId: string,
    public readonly userId: string,
    public readonly total: number,
    public readonly items: Array<{ productId: string; quantity: number }>,
  ) {}
}

export class PaymentCompletedEvent {
  constructor(
    public readonly orderId: string,
    public readonly amount: number,
    public readonly method: string,
  ) {}
}
```

## 3. Emit Events

```typescript
// src/users/users.service.ts
import { EventEmitter2 } from '@nestjs/event-emitter';
import { UserCreatedEvent } from '../events/user.events';

@Injectable()
export class UsersService {
  constructor(
    @InjectModel(User.name) private userModel: Model<User>,
    private eventEmitter: EventEmitter2,
  ) {}

  async create(dto: CreateUserDto): Promise<User> {
    const user = await this.userModel.create(dto);

    // Emit event — không cần biết ai lắng nghe
    this.eventEmitter.emit(
      'user.created',
      new UserCreatedEvent(user._id.toString(), user.email, user.name),
    );

    return user;
  }

  async update(id: string, dto: UpdateUserDto): Promise<User> {
    const user = await this.userModel.findByIdAndUpdate(id, dto, { new: true });

    this.eventEmitter.emit('user.updated', new UserUpdatedEvent(id, dto));

    return user!;
  }
}

// src/orders/orders.service.ts
@Injectable()
export class OrdersService {
  constructor(private eventEmitter: EventEmitter2) {}

  async placeOrder(dto: CreateOrderDto, userId: string): Promise<Order> {
    const order = await this.orderModel.create({ ...dto, userId });

    this.eventEmitter.emit(
      'order.placed',
      new OrderPlacedEvent(order._id.toString(), userId, order.total, order.items),
    );

    return order;
  }
}
```

## 4. Listen Events

```typescript
// src/notifications/notification.listener.ts
import { OnEvent } from '@nestjs/event-emitter';

@Injectable()
export class NotificationListener {
  constructor(
    private mailService: MailService,
    private smsService: SmsService,
  ) {}

  @OnEvent('user.created')
  async handleUserCreated(event: UserCreatedEvent) {
    await this.mailService.sendWelcomeEmail(event.email, event.name);
  }

  @OnEvent('order.placed')
  async handleOrderPlaced(event: OrderPlacedEvent) {
    // Gửi email xác nhận đơn hàng
    await this.mailService.sendOrderConfirmation(event.orderId, event.userId);

    // Gửi thông báo admin
    await this.smsService.notifyAdmin(`Đơn hàng mới: ${event.orderId}`);
  }

  @OnEvent('payment.completed')
  async handlePaymentCompleted(event: PaymentCompletedEvent) {
    await this.mailService.sendPaymentReceipt(event.orderId, event.amount);
  }
}
```

```typescript
// src/inventory/inventory.listener.ts
@Injectable()
export class InventoryListener {
  constructor(private stockService: StockService) {}

  @OnEvent('order.placed')
  async handleOrderPlaced(event: OrderPlacedEvent) {
    // Trừ kho sau khi có đơn hàng
    for (const item of event.items) {
      await this.stockService.decrement(item.productId, item.quantity);
    }
  }
}
```

```typescript
// src/analytics/analytics.listener.ts
@Injectable()
export class AnalyticsListener {
  // Wildcard — lắng nghe tất cả user events
  @OnEvent('user.*')
  trackUserEvent(event: any) {
    this.analyticsService.track('user_event', {
      eventName: event.constructor.name,
      userId: event.userId,
      timestamp: new Date(),
    });
  }

  // Lắng nghe mọi events
  @OnEvent('**')
  logAllEvents(event: any) {
    this.logger.debug(`Event: ${event.constructor.name}`);
  }
}
```

## 5. Async Events

```typescript
// Async listener — EventEmitter chờ listener hoàn thành
@OnEvent('order.placed', { async: true })
async handleOrderPlaced(event: OrderPlacedEvent): Promise<void> {
  await this.longRunningTask(event.orderId);
}

// Promise.all — chạy song song nhiều async listeners
EventEmitterModule.forRoot({
  global: true,
  // Mặc định async listeners chạy song song
})
```

## 6. Đăng ký listener module

```typescript
// src/notifications/notifications.module.ts
@Module({
  providers: [NotificationListener, MailService, SmsService],
})
export class NotificationsModule {}

// src/inventory/inventory.module.ts
@Module({
  providers: [InventoryListener, StockService],
})
export class InventoryModule {}

// app.module.ts — Import các listener module
@Module({
  imports: [
    EventEmitterModule.forRoot({ wildcard: true }),
    UsersModule,
    OrdersModule,
    NotificationsModule,  // Tự động register listeners
    InventoryModule,
    AnalyticsModule,
  ],
})
export class AppModule {}
```

## 7. Testing với Event Emitter

```typescript
describe('UsersService', () => {
  let service: UsersService;
  let eventEmitter: EventEmitter2;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        UsersService,
        { provide: getModelToken(User.name), useValue: mockUserModel },
        { provide: EventEmitter2, useValue: { emit: jest.fn() } },
      ],
    }).compile();

    service = module.get(UsersService);
    eventEmitter = module.get(EventEmitter2);
  });

  it('nên emit user.created event khi tạo user', async () => {
    mockUserModel.create.mockResolvedValue(mockUser);

    await service.create(createUserDto);

    expect(eventEmitter.emit).toHaveBeenCalledWith(
      'user.created',
      expect.objectContaining({ email: mockUser.email }),
    );
  });
});
```

## 8. So sánh Event Emitter vs Message Queue

| Tiêu chí | Event Emitter | Redis/Kafka |
|---------|--------------|-------------|
| Scope | In-process (cùng instance) | Cross-service |
| Durability | Mất khi restart | Persistent |
| Retry | Không tự động | Có |
| Scale | Single instance | Multi-instance |
| Latency | Cực thấp (< 1ms) | Thấp (< 10ms) |
| Use case | Monolith internal | Microservices |

## 9. Kết luận

- **`@OnEvent('event.name')`**: Đăng ký listener, hỗ trợ wildcard `user.*` và `**`
- **`eventEmitter.emit()`**: Publish event từ bất kỳ service nào
- **Async**: Listener async chạy song song, không block caller
- **Decoupling**: UsersService không cần biết MailService hay InventoryService tồn tại
- **Testing**: Mock `EventEmitter2` để verify event được emit đúng

Event Emitter là bước đầu tiên của event-driven — khi cần scale microservices, migrate sang Redis/Kafka mà không cần sửa business logic.
