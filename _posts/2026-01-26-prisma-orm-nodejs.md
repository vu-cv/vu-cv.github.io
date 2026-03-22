---
layout: article
title: Prisma ORM – Database toolkit hiện đại cho NodeJS
tags: [prisma, orm, typescript, postgresql, mysql, nodejs]
---
Prisma là ORM thế hệ mới với type-safety tuyệt vời — schema định nghĩa một lần, Prisma Client tự generate với đầy đủ TypeScript types. So với TypeORM, Prisma DX (developer experience) tốt hơn đáng kể.

## 1. Cài đặt

```bash
npm install prisma @prisma/client
npx prisma init --datasource-provider postgresql
```

Tạo ra:
- `prisma/schema.prisma` — schema file
- `.env` với `DATABASE_URL`

## 2. Định nghĩa Schema

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(uuid())
  email     String   @unique
  name      String
  password  String
  avatar    String?
  role      Role     @default(USER)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  posts     Post[]
  orders    Order[]
  profile   Profile?

  @@index([email])
  @@map("users") // Tên bảng trong DB
}

enum Role {
  USER
  ADMIN
  MODERATOR
}

model Profile {
  id     String  @id @default(uuid())
  bio    String?
  phone  String?
  userId String  @unique
  user   User    @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("profiles")
}

model Post {
  id         String   @id @default(uuid())
  title      String
  content    String
  published  Boolean  @default(false)
  viewCount  Int      @default(0)
  authorId   String
  author     User     @relation(fields: [authorId], references: [id])
  tags       Tag[]
  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt

  @@index([authorId])
  @@map("posts")
}

model Tag {
  id    String @id @default(uuid())
  name  String @unique
  posts Post[]

  @@map("tags")
}

model Order {
  id         String      @id @default(uuid())
  userId     String
  user       User        @relation(fields: [userId], references: [id])
  status     OrderStatus @default(PENDING)
  total      Decimal     @db.Decimal(10, 2)
  items      OrderItem[]
  createdAt  DateTime    @default(now())

  @@map("orders")
}

enum OrderStatus {
  PENDING
  CONFIRMED
  SHIPPING
  DELIVERED
  CANCELLED
}

model OrderItem {
  id        String  @id @default(uuid())
  orderId   String
  order     Order   @relation(fields: [orderId], references: [id])
  productId String
  quantity  Int
  price     Decimal @db.Decimal(10, 2)

  @@map("order_items")
}
```

## 3. Migrations

```bash
# Tạo migration từ schema changes
npx prisma migrate dev --name add_user_role

# Apply migration (production)
npx prisma migrate deploy

# Reset DB (dev only)
npx prisma migrate reset

# Xem DB trong browser
npx prisma studio
```

## 4. Prisma Client — CRUD

```typescript
// src/prisma/prisma.service.ts
import { Injectable, OnModuleInit } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit {
  async onModuleInit() {
    await this.$connect();
  }
}
```

```typescript
// src/users/users.service.ts
import { Injectable, NotFoundException, ConflictException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { Prisma, User } from '@prisma/client';

@Injectable()
export class UsersService {
  constructor(private prisma: PrismaService) {}

  async findAll(params: {
    page?: number;
    limit?: number;
    search?: string;
    role?: string;
  }) {
    const { page = 1, limit = 10, search, role } = params;

    const where: Prisma.UserWhereInput = {
      ...(search && {
        OR: [
          { name: { contains: search, mode: 'insensitive' } },
          { email: { contains: search, mode: 'insensitive' } },
        ],
      }),
      ...(role && { role: role as any }),
    };

    const [users, total] = await this.prisma.$transaction([
      this.prisma.user.findMany({
        where,
        skip: (page - 1) * limit,
        take: limit,
        select: {
          id: true, name: true, email: true, role: true, createdAt: true,
          // Không select password
        },
        orderBy: { createdAt: 'desc' },
      }),
      this.prisma.user.count({ where }),
    ]);

    return { users, total, page, limit };
  }

  async findById(id: string) {
    const user = await this.prisma.user.findUnique({
      where: { id },
      include: {
        profile: true,
        _count: { select: { posts: true, orders: true } },
      },
    });

    if (!user) throw new NotFoundException('User không tồn tại');
    return user;
  }

  async create(dto: Prisma.UserCreateInput): Promise<User> {
    try {
      return await this.prisma.user.create({ data: dto });
    } catch (e) {
      if (e instanceof Prisma.PrismaClientKnownRequestError && e.code === 'P2002') {
        throw new ConflictException('Email đã tồn tại');
      }
      throw e;
    }
  }

  async update(id: string, dto: Prisma.UserUpdateInput): Promise<User> {
    return this.prisma.user.update({ where: { id }, data: dto });
  }

  async delete(id: string): Promise<void> {
    await this.prisma.user.delete({ where: { id } });
  }
}
```

## 5. Relations & Nested queries

```typescript
// Tạo order kèm items trong 1 query
async createOrder(userId: string, items: Array<{ productId: string; quantity: number; price: number }>) {
  return this.prisma.order.create({
    data: {
      userId,
      total: items.reduce((sum, i) => sum + i.price * i.quantity, 0),
      items: {
        create: items.map(item => ({
          productId: item.productId,
          quantity: item.quantity,
          price: item.price,
        })),
      },
    },
    include: {
      items: true,
      user: { select: { name: true, email: true } },
    },
  });
}

// Many-to-many: kết nối Post với Tags
async addTagsToPost(postId: string, tagNames: string[]) {
  return this.prisma.post.update({
    where: { id: postId },
    data: {
      tags: {
        connectOrCreate: tagNames.map(name => ({
          where: { name },
          create: { name },
        })),
      },
    },
    include: { tags: true },
  });
}
```

## 6. Transaction

```typescript
// $transaction — tất cả hoặc không có gì
async transferBalance(fromId: string, toId: string, amount: number) {
  return this.prisma.$transaction(async (tx) => {
    const sender = await tx.user.findUnique({ where: { id: fromId } });
    if (!sender || sender.balance < amount) throw new Error('Insufficient balance');

    await tx.user.update({
      where: { id: fromId },
      data: { balance: { decrement: amount } },
    });

    await tx.user.update({
      where: { id: toId },
      data: { balance: { increment: amount } },
    });

    return tx.transaction.create({
      data: { fromId, toId, amount },
    });
  });
}
```

## 7. Raw Queries

```typescript
// Raw SQL khi Prisma không đủ linh hoạt
const result = await this.prisma.$queryRaw<Array<{ count: bigint; date: Date }>>`
  SELECT
    DATE_TRUNC('day', created_at) as date,
    COUNT(*) as count
  FROM orders
  WHERE created_at >= ${startDate}
  GROUP BY DATE_TRUNC('day', created_at)
  ORDER BY date DESC
`;

// Convert BigInt
return result.map(r => ({ date: r.date, count: Number(r.count) }));
```

## 8. So sánh Prisma vs TypeORM

| Tiêu chí | Prisma | TypeORM |
|---------|--------|---------|
| Schema | `schema.prisma` | Decorators (@Entity) |
| Type safety | Xuất sắc | Tốt |
| Query API | Fluent, type-safe | Repository + QueryBuilder |
| Migrations | Tốt hơn | Phức tạp hơn |
| Relations | Tường minh | Implicit lazy load |
| Raw SQL | `$queryRaw` | `query()` |
| Performance | Tương đương | Tương đương |

## 9. Kết luận

- **Schema first**: Một file `schema.prisma` định nghĩa tất cả — rõ ràng, dễ đọc
- **Type safety**: Auto-generated types từ schema — không bao giờ sai type
- **`$transaction`**: ACID transactions dễ dàng
- **Prisma Studio**: GUI browser xem và edit data — cực tiện khi dev
- **`select`/`include`**: Kiểm soát chính xác field nào được trả về — tránh over-fetching

Prisma là lựa chọn tốt cho dự án mới dùng SQL (PostgreSQL, MySQL). TypeORM vẫn phù hợp nếu team đã quen.
