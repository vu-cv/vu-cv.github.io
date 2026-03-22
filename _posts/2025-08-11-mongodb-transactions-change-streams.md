---
layout: article
title: MongoDB – Transactions & Change Streams
tags: [mongodb, transactions, change-streams, nodejs, realtime]
---
Hai tính năng nâng cao của MongoDB thường ít được biết đến nhưng cực kỳ hữu ích: **Transactions** (đảm bảo ACID khi thao tác nhiều document) và **Change Streams** (lắng nghe thay đổi dữ liệu theo thời gian thực).

## 1. Transactions

MongoDB hỗ trợ multi-document ACID transactions từ phiên bản 4.0 (Replica Set) và 4.2 (Sharded Cluster).

### Khi nào cần Transaction?

```
❌ Không dùng Transaction:
- Chỉ thao tác một document duy nhất (MongoDB đảm bảo atomic mặc định)

✅ Cần Transaction:
- Chuyển tiền: trừ tài khoản A + cộng tài khoản B (2 documents)
- Tạo đơn hàng: tạo order + trừ stock sản phẩm
- Đặt chỗ: tạo booking + cập nhật slot available
```

### Transaction cơ bản với Mongoose

```typescript
import mongoose from 'mongoose';

async function transferMoney(fromId: string, toId: string, amount: number) {
  const session = await mongoose.startSession();

  try {
    session.startTransaction();

    const sender = await Wallet.findById(fromId).session(session);
    const receiver = await Wallet.findById(toId).session(session);

    if (!sender || !receiver) throw new Error('Tài khoản không tồn tại');
    if (sender.balance < amount) throw new Error('Số dư không đủ');

    await Wallet.findByIdAndUpdate(fromId, { $inc: { balance: -amount } }, { session });
    await Wallet.findByIdAndUpdate(toId,   { $inc: { balance: +amount } }, { session });

    // Ghi lịch sử giao dịch
    await Transaction.create([{
      from: fromId, to: toId, amount,
      status: 'completed',
      createdAt: new Date(),
    }], { session });

    await session.commitTransaction();
    return { success: true };
  } catch (error) {
    await session.abortTransaction(); // Rollback tất cả thay đổi
    throw error;
  } finally {
    session.endSession();
  }
}
```

### Transaction với withTransaction helper

```typescript
// Cú pháp gọn hơn — tự động retry khi có transient error
async function createOrder(userId: string, items: OrderItem[]) {
  const session = await mongoose.startSession();

  return session.withTransaction(async () => {
    // Kiểm tra và trừ stock
    for (const item of items) {
      const result = await Product.findOneAndUpdate(
        { _id: item.productId, stock: { $gte: item.quantity } },
        { $inc: { stock: -item.quantity } },
        { session, new: true },
      );

      if (!result) {
        throw new Error(`Sản phẩm ${item.productId} không đủ hàng`);
      }
    }

    // Tạo đơn hàng
    const [order] = await Order.create([{ userId, items, status: 'pending' }], { session });
    return order;
  });
}
```

### Transaction trong NestJS Service

```typescript
@Injectable()
export class OrdersService {
  constructor(
    @InjectConnection() private readonly connection: mongoose.Connection,
  ) {}

  async createOrder(dto: CreateOrderDto) {
    const session = await this.connection.startSession();

    return session.withTransaction(async () => {
      // Tất cả operations đều truyền { session }
      const order = await this.orderModel.create([dto], { session });
      await this.stockModel.bulkWrite(
        dto.items.map(i => ({
          updateOne: {
            filter: { _id: i.productId, stock: { $gte: i.quantity } },
            update: { $inc: { stock: -i.quantity } },
          },
        })),
        { session },
      );
      return order[0];
    });
  }
}
```

## 2. Change Streams

Change Streams cho phép lắng nghe mọi thay đổi trong collection theo thời gian thực — như trigger database nhưng ở application layer.

> **Yêu cầu**: MongoDB phải chạy dưới dạng Replica Set (ít nhất 1 node). Dùng MongoDB Atlas hoặc cấu hình local replica set.

### Cơ bản

```typescript
import { Collection, ChangeStream } from 'mongodb';

async function watchCollection() {
  const db = mongoose.connection.db;
  const collection = db.collection('orders');

  const changeStream: ChangeStream = collection.watch([], {
    fullDocument: 'updateLookup', // Trả về full document sau khi update
  });

  changeStream.on('change', (event) => {
    switch (event.operationType) {
      case 'insert':
        console.log('Đơn hàng mới:', event.fullDocument);
        break;
      case 'update':
        console.log('Đơn hàng cập nhật:', event.fullDocument);
        break;
      case 'delete':
        console.log('Đơn hàng xóa:', event.documentKey._id);
        break;
    }
  });

  changeStream.on('error', (err) => console.error('Change stream error:', err));

  // Đóng khi không dùng nữa
  // await changeStream.close();
}
```

### Filter chỉ lắng nghe event cụ thể

```typescript
// Chỉ lắng nghe khi status thay đổi thành 'shipped'
const pipeline = [
  {
    $match: {
      operationType: 'update',
      'updateDescription.updatedFields.status': 'shipped',
    },
  },
];

const changeStream = collection.watch(pipeline, { fullDocument: 'updateLookup' });

changeStream.on('change', async (event) => {
  const order = event.fullDocument;
  // Gửi email thông báo đơn hàng đã giao
  await emailService.sendShippingNotification(order.userId, order._id);
  // Push notification qua WebSocket
  gateway.server.to(`user-${order.userId}`).emit('orderShipped', { orderId: order._id });
});
```

### Tích hợp vào NestJS — Real-time sync

```typescript
@Injectable()
export class OrderChangeStreamService implements OnModuleInit, OnModuleDestroy {
  private changeStream: ChangeStream;

  constructor(
    @InjectConnection() private connection: mongoose.Connection,
    private readonly notificationGateway: NotificationGateway,
    private readonly emailService: EmailService,
  ) {}

  async onModuleInit() {
    const collection = this.connection.collection('orders');

    this.changeStream = collection.watch(
      [{ $match: { operationType: { $in: ['insert', 'update'] } } }],
      { fullDocument: 'updateLookup' },
    );

    this.changeStream.on('change', async (event) => {
      if (event.operationType === 'insert') {
        await this.handleNewOrder(event.fullDocument);
      } else if (event.operationType === 'update') {
        await this.handleOrderUpdate(event.fullDocument);
      }
    });
  }

  private async handleNewOrder(order: any) {
    // Notify admin dashboard
    this.notificationGateway.server.emit('newOrder', { orderId: order._id, total: order.total });
  }

  private async handleOrderUpdate(order: any) {
    if (order.status === 'shipped') {
      await this.emailService.sendShippingEmail(order.userId);
      this.notificationGateway.server.to(`user-${order.userId}`).emit('orderShipped', order);
    }
  }

  async onModuleDestroy() {
    await this.changeStream?.close();
  }
}
```

## 3. Resume Token — Tiếp tục sau khi reconnect

```typescript
let resumeToken: any;

changeStream.on('change', (event) => {
  resumeToken = event._id; // Lưu lại token
  processEvent(event);
});

// Khi reconnect, tiếp tục từ vị trí đã dừng
const changeStream = collection.watch(pipeline, {
  resumeAfter: resumeToken, // Không bỏ sót event nào
});
```

## 4. Kết luận

- **Transactions**: Đảm bảo ACID khi thao tác nhiều document — dùng `withTransaction` để auto-retry
- **Change Streams**: Lắng nghe thay đổi dữ liệu real-time — kết hợp tốt với WebSocket và event-driven architecture
- Cả hai đều yêu cầu MongoDB Replica Set (Atlas đã cấu hình sẵn)
- **Resume Token** giúp Change Stream tiếp tục đúng chỗ sau khi reconnect — không mất event
