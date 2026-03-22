---
layout: article
title: NestJS – gRPC với Protocol Buffers
tags: [nestjs, grpc, protobuf, microservices, nodejs, typescript]
---
gRPC là framework RPC hiệu năng cao của Google — dùng Protocol Buffers (binary format) thay vì JSON, tốt hơn REST cho internal microservices communication: nhanh hơn (~7x), type-safe hơn, hỗ trợ streaming hai chiều.

## 1. gRPC vs REST vs GraphQL

| Tiêu chí | gRPC | REST | GraphQL |
|---------|------|------|---------|
| Protocol | HTTP/2 | HTTP/1.1 | HTTP |
| Format | Binary (Protobuf) | JSON | JSON |
| Speed | Rất nhanh | Nhanh | Nhanh |
| Streaming | Bidirectional | SSE/WS | Subscriptions |
| Type safety | Schema-first | OpenAPI | Schema-first |
| Browser support | Cần proxy | Native | Native |
| Use case | Internal services | Public API | Flexible queries |

## 2. Cài đặt

```bash
npm install @grpc/grpc-js @grpc/proto-loader
npm install @nestjs/microservices
npm install -D grpc-tools ts-protoc-gen
```

## 3. Định nghĩa Proto file

```protobuf
// proto/users.proto
syntax = "proto3";

package users;

option java_package = "com.shopxyz.users";

service UsersService {
  // Unary RPC
  rpc GetUser(GetUserRequest) returns (User);
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc UpdateUser(UpdateUserRequest) returns (User);
  rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);

  // Server streaming — stream danh sách users
  rpc ListUsers(ListUsersRequest) returns (stream User);

  // Client streaming — upload nhiều users
  rpc BulkCreateUsers(stream CreateUserRequest) returns (BulkCreateResponse);

  // Bidirectional streaming — chat-like
  rpc ChatWithSupport(stream ChatMessage) returns (stream ChatMessage);
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
  string role = 4;
  int64 created_at = 5;
}

message GetUserRequest {
  string id = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
  string password = 3;
}

message UpdateUserRequest {
  string id = 1;
  optional string name = 2;
  optional string avatar = 3;
}

message DeleteUserRequest {
  string id = 1;
}

message DeleteUserResponse {
  bool success = 1;
}

message ListUsersRequest {
  int32 page = 1;
  int32 limit = 2;
  string search = 3;
}

message BulkCreateResponse {
  int32 created = 1;
  int32 failed = 2;
  repeated string errors = 3;
}

message ChatMessage {
  string user_id = 1;
  string content = 2;
  int64 timestamp = 3;
}
```

```protobuf
// proto/orders.proto
syntax = "proto3";
package orders;

service OrdersService {
  rpc CreateOrder(CreateOrderRequest) returns (Order);
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc UpdateOrderStatus(UpdateStatusRequest) returns (Order);
}

message Order {
  string id = 1;
  string user_id = 2;
  string status = 3;
  double total = 4;
  repeated OrderItem items = 5;
}

message OrderItem {
  string product_id = 1;
  int32 quantity = 2;
  double price = 3;
}

message CreateOrderRequest {
  string user_id = 1;
  repeated OrderItem items = 2;
}

message GetOrderRequest {
  string id = 1;
}

message UpdateStatusRequest {
  string id = 1;
  string status = 2;
}
```

## 4. gRPC Server (NestJS Microservice)

```typescript
// user-service/main.ts
import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { join } from 'path';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
    transport: Transport.GRPC,
    options: {
      package: 'users',
      protoPath: join(__dirname, '../proto/users.proto'),
      url: '0.0.0.0:50051',
    },
  });
  await app.listen();
  console.log('Users gRPC service running on :50051');
}
bootstrap();
```

```typescript
// user-service/users.controller.ts
import { Controller } from '@nestjs/common';
import { GrpcMethod, GrpcStreamMethod } from '@nestjs/microservices';
import { Observable, Subject } from 'rxjs';

@Controller()
export class UsersController {
  constructor(private usersService: UsersService) {}

  // Unary RPC
  @GrpcMethod('UsersService', 'GetUser')
  async getUser(data: { id: string }) {
    const user = await this.usersService.findById(data.id);
    if (!user) throw new RpcException({ code: 5, message: 'User not found' }); // 5 = NOT_FOUND
    return user;
  }

  @GrpcMethod('UsersService', 'CreateUser')
  async createUser(data: { name: string; email: string; password: string }) {
    return this.usersService.create(data);
  }

  // Server streaming
  @GrpcMethod('UsersService', 'ListUsers')
  listUsers(data: { page: number; limit: number }): Observable<any> {
    const subject = new Subject();

    (async () => {
      const users = await this.usersService.findAll(data);
      for (const user of users) {
        subject.next(user);
        await new Promise(r => setTimeout(r, 10)); // Simulate delay
      }
      subject.complete();
    })();

    return subject.asObservable();
  }

  // Client streaming
  @GrpcStreamMethod('UsersService', 'BulkCreateUsers')
  async bulkCreateUsers(data$: Observable<any>): Promise<any> {
    let created = 0;
    let failed = 0;
    const errors: string[] = [];

    await new Promise<void>((resolve) => {
      data$.subscribe({
        next: async (userDto) => {
          try {
            await this.usersService.create(userDto);
            created++;
          } catch (e) {
            failed++;
            errors.push(e.message);
          }
        },
        complete: () => resolve(),
      });
    });

    return { created, failed, errors };
  }
}
```

## 5. gRPC Client

```typescript
// api-gateway/app.module.ts
import { ClientsModule, Transport } from '@nestjs/microservices';

@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'USERS_SERVICE',
        transport: Transport.GRPC,
        options: {
          package: 'users',
          protoPath: join(__dirname, '../proto/users.proto'),
          url: 'localhost:50051',
        },
      },
      {
        name: 'ORDERS_SERVICE',
        transport: Transport.GRPC,
        options: {
          package: 'orders',
          protoPath: join(__dirname, '../proto/orders.proto'),
          url: 'localhost:50052',
        },
      },
    ]),
  ],
})
export class AppModule {}
```

```typescript
// api-gateway/users.controller.ts
import { Controller, Get, Post, Param, Body, OnModuleInit } from '@nestjs/common';
import { Client, ClientGrpc } from '@nestjs/microservices';
import { firstValueFrom } from 'rxjs';

interface UsersServiceGrpc {
  getUser(data: { id: string }): Observable<any>;
  createUser(data: any): Observable<any>;
  listUsers(data: any): Observable<any>;
}

@Controller('users')
export class UsersController implements OnModuleInit {
  private usersGrpc: UsersServiceGrpc;

  constructor(
    @Inject('USERS_SERVICE') private client: ClientGrpc,
  ) {}

  onModuleInit() {
    this.usersGrpc = this.client.getService<UsersServiceGrpc>('UsersService');
  }

  @Get(':id')
  async getUser(@Param('id') id: string) {
    return firstValueFrom(this.usersGrpc.getUser({ id }));
  }

  @Post()
  async createUser(@Body() dto: any) {
    return firstValueFrom(this.usersGrpc.createUser(dto));
  }

  @Get()
  listUsers(@Query('page') page = '1') {
    // Server streaming → Observable → client có thể stream response
    return this.usersGrpc.listUsers({ page: parseInt(page), limit: 20 });
  }
}
```

## 6. TypeScript types từ Proto (code generation)

```bash
# Generate TypeScript interfaces từ .proto files
npx grpc_tools_node_protoc \
  --plugin=protoc-gen-ts=./node_modules/.bin/protoc-gen-ts \
  --ts_out=./src/generated \
  --js_out=import_style=commonjs:./src/generated \
  --grpc_out=grpc_js:./src/generated \
  -I ./proto \
  proto/*.proto
```

## 7. Kết luận

- **Protobuf**: Binary format — nhỏ hơn JSON ~5x, parse nhanh hơn
- **HTTP/2**: Multiplexing, header compression, server push
- **Streaming**: Unary/Server/Client/Bidirectional — linh hoạt hơn REST
- **`@GrpcMethod`**: Xử lý unary call; `@GrpcStreamMethod`: xử lý client streaming
- **Giới hạn**: Browser không gọi trực tiếp gRPC — cần gateway hoặc gRPC-Web

gRPC là lựa chọn tốt nhất cho internal microservices communication — REST vẫn phù hợp cho public API.
