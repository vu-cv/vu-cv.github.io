---
layout: article
title: NestJS – Real-time với WebSocket & Socket.io
tags: [nestjs, websocket, socket.io, realtime]
---
WebSocket cho phép server và client giao tiếp hai chiều theo thời gian thực — lý tưởng cho chat, notification, live dashboard. NestJS tích hợp Socket.io cực kỳ mượt mà thông qua `@nestjs/websockets`.

## 1. Cài đặt

```bash
npm install --save @nestjs/websockets @nestjs/platform-socket.io socket.io
npm install --save-dev @types/socket.io
```

## 2. Tạo Gateway

Trong NestJS, **Gateway** là tương đương của Controller nhưng cho WebSocket:

```bash
nest g gateway chat
```

`src/chat/chat.gateway.ts`:

```typescript
import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  MessageBody,
  ConnectedSocket,
  OnGatewayConnection,
  OnGatewayDisconnect,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';

@WebSocketGateway({
  cors: { origin: '*' }, // Cấu hình CORS cho client
  namespace: '/chat',    // Namespace (tùy chọn)
})
export class ChatGateway implements OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer()
  server: Server;

  // Khi client kết nối
  handleConnection(client: Socket) {
    console.log(`Client connected: ${client.id}`);
  }

  // Khi client ngắt kết nối
  handleDisconnect(client: Socket) {
    console.log(`Client disconnected: ${client.id}`);
  }

  // Lắng nghe event 'sendMessage' từ client
  @SubscribeMessage('sendMessage')
  handleMessage(
    @MessageBody() data: { room: string; message: string; username: string },
    @ConnectedSocket() client: Socket,
  ) {
    // Broadcast tới tất cả client trong room
    this.server.to(data.room).emit('receiveMessage', {
      username: data.username,
      message: data.message,
      timestamp: new Date().toISOString(),
    });
  }

  // Join vào room
  @SubscribeMessage('joinRoom')
  handleJoinRoom(
    @MessageBody() data: { room: string; username: string },
    @ConnectedSocket() client: Socket,
  ) {
    client.join(data.room);
    // Thông báo cho cả room
    this.server.to(data.room).emit('userJoined', {
      username: data.username,
      message: `${data.username} đã tham gia phòng`,
    });
  }

  // Rời room
  @SubscribeMessage('leaveRoom')
  handleLeaveRoom(
    @MessageBody() data: { room: string; username: string },
    @ConnectedSocket() client: Socket,
  ) {
    client.leave(data.room);
    this.server.to(data.room).emit('userLeft', {
      username: data.username,
      message: `${data.username} đã rời phòng`,
    });
  }
}
```

## 3. Đăng ký Gateway vào Module

```typescript
// src/chat/chat.module.ts
import { Module } from '@nestjs/common';
import { ChatGateway } from './chat.gateway';

@Module({
  providers: [ChatGateway],
})
export class ChatModule {}
```

```typescript
// src/app.module.ts
import { ChatModule } from './chat/chat.module';

@Module({ imports: [ChatModule] })
export class AppModule {}
```

## 4. Emit từ HTTP Controller sang WebSocket

Rất hữu ích khi muốn notify client sau khi có event từ REST API:

```typescript
import { Injectable } from '@nestjs/common';
import { ChatGateway } from './chat.gateway';

@Injectable()
export class NotificationService {
  constructor(private readonly chatGateway: ChatGateway) {}

  notifyNewOrder(orderId: string) {
    // Gửi event tới TẤT CẢ client đang kết nối
    this.chatGateway.server.emit('newOrder', { orderId });
  }

  notifyRoom(room: string, data: object) {
    this.chatGateway.server.to(room).emit('notification', data);
  }
}
```

## 5. Client-side (JavaScript/TypeScript)

```javascript
import { io } from 'socket.io-client';

const socket = io('http://localhost:3000/chat');

// Kết nối
socket.on('connect', () => {
  console.log('Connected:', socket.id);

  // Join room
  socket.emit('joinRoom', { room: 'room-1', username: 'Nguyen Van A' });
});

// Gửi tin nhắn
function sendMessage(message) {
  socket.emit('sendMessage', {
    room: 'room-1',
    message,
    username: 'Nguyen Van A',
  });
}

// Nhận tin nhắn
socket.on('receiveMessage', (data) => {
  console.log(`${data.username}: ${data.message}`);
});

// Nhận thông báo user mới vào room
socket.on('userJoined', (data) => {
  console.log(data.message);
});
```

## 6. Authentication cho WebSocket

Xác thực JWT khi client kết nối:

```typescript
import { Injectable } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { WsException } from '@nestjs/websockets';
import { Socket } from 'socket.io';

@Injectable()
export class WsAuthGuard {
  constructor(private jwtService: JwtService) {}

  canActivate(client: Socket): boolean {
    const token = client.handshake.auth?.token
      ?? client.handshake.headers?.authorization?.split(' ')[1];

    if (!token) throw new WsException('Unauthorized');

    try {
      const payload = this.jwtService.verify(token);
      (client as any).user = payload;
      return true;
    } catch {
      throw new WsException('Invalid token');
    }
  }
}
```

Dùng trong Gateway:
```typescript
handleConnection(client: Socket) {
  try {
    this.wsAuthGuard.canActivate(client);
    console.log(`Authenticated: ${(client as any).user.email}`);
  } catch {
    client.disconnect();
  }
}
```

## 7. Kết luận

NestJS + Socket.io giúp xây dựng real-time features dễ dàng:

- **Gateway** = Controller cho WebSocket
- `@SubscribeMessage` = lắng nghe event từ client
- `server.emit` = broadcast tới tất cả, `server.to(room).emit` = gửi tới room cụ thể
- Tích hợp tốt với HTTP Controller — notify client sau REST API call

Use cases phổ biến: chat app, live notification, real-time dashboard, collaborative editing.
