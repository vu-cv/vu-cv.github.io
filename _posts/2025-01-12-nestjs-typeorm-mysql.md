---
layout: article
title: NestJS – CRUD với TypeORM & MySQL
tags: [nestjs, typeorm, mysql, crud, database]
---
TypeORM là ORM phổ biến nhất trong hệ sinh thái NestJS, hỗ trợ MySQL, PostgreSQL, SQLite và nhiều database khác. Bài này hướng dẫn xây dựng CRUD API hoàn chỉnh với NestJS + TypeORM + MySQL.

## 1. Cài đặt

```bash
npm install --save @nestjs/typeorm typeorm mysql2
```

## 2. Cấu hình kết nối

```typescript
// app.module.ts
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'mysql',
      host: process.env.DB_HOST ?? 'localhost',
      port: Number(process.env.DB_PORT ?? 3306),
      username: process.env.DB_USER ?? 'root',
      password: process.env.DB_PASS ?? 'password',
      database: process.env.DB_NAME ?? 'mydb',
      entities: [__dirname + '/**/*.entity{.ts,.js}'],
      synchronize: true,   // Chỉ dùng trong dev — tự tạo/update bảng
      logging: process.env.NODE_ENV !== 'production',
    }),
  ],
})
export class AppModule {}
```

## 3. Tạo Entity

```typescript
// src/products/product.entity.ts
import {
  Entity, PrimaryGeneratedColumn, Column,
  CreateDateColumn, UpdateDateColumn, Index,
} from 'typeorm';

@Entity('products')
export class Product {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Index()
  @Column({ length: 255 })
  name: string;

  @Column('text', { nullable: true })
  description: string;

  @Column('decimal', { precision: 10, scale: 2 })
  price: number;

  @Column({ default: 0 })
  stock: number;

  @Index()
  @Column({ nullable: true })
  category: string;

  @Column({ default: true })
  isActive: boolean;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

## 4. Quan hệ giữa các bảng

```typescript
// src/orders/order.entity.ts
import {
  Entity, PrimaryGeneratedColumn, Column,
  ManyToOne, OneToMany, JoinColumn,
  CreateDateColumn,
} from 'typeorm';
import { User } from '../users/user.entity';
import { OrderItem } from './order-item.entity';

@Entity('orders')
export class Order {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @ManyToOne(() => User, user => user.orders, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'user_id' })
  user: User;

  @Column({ name: 'user_id' })
  userId: string;

  @OneToMany(() => OrderItem, item => item.order, { cascade: true })
  items: OrderItem[];

  @Column('decimal', { precision: 12, scale: 2, default: 0 })
  total: number;

  @Column({ default: 'pending' })
  status: string;

  @CreateDateColumn()
  createdAt: Date;
}

// src/orders/order-item.entity.ts
@Entity('order_items')
export class OrderItem {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @ManyToOne(() => Order, order => order.items)
  order: Order;

  @ManyToOne(() => Product)
  @JoinColumn({ name: 'product_id' })
  product: Product;

  @Column()
  quantity: number;

  @Column('decimal', { precision: 10, scale: 2 })
  price: number;
}
```

## 5. Repository Pattern

```typescript
// src/products/products.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, Like, Between } from 'typeorm';
import { Product } from './product.entity';

@Injectable()
export class ProductsService {
  constructor(
    @InjectRepository(Product)
    private productRepo: Repository<Product>,
  ) {}

  async findAll(query: {
    page?: number; limit?: number;
    search?: string; category?: string;
    minPrice?: number; maxPrice?: number;
  }) {
    const { page = 1, limit = 10, search, category, minPrice, maxPrice } = query;

    const qb = this.productRepo.createQueryBuilder('p')
      .where('p.isActive = :active', { active: true });

    if (search) qb.andWhere('p.name LIKE :search', { search: `%${search}%` });
    if (category) qb.andWhere('p.category = :category', { category });
    if (minPrice) qb.andWhere('p.price >= :minPrice', { minPrice });
    if (maxPrice) qb.andWhere('p.price <= :maxPrice', { maxPrice });

    const [items, total] = await qb
      .orderBy('p.createdAt', 'DESC')
      .skip((page - 1) * limit)
      .take(limit)
      .getManyAndCount();

    return { items, total, page, limit, totalPages: Math.ceil(total / limit) };
  }

  async findOne(id: string): Promise<Product> {
    const product = await this.productRepo.findOne({ where: { id } });
    if (!product) throw new NotFoundException(`Sản phẩm #${id} không tồn tại`);
    return product;
  }

  async create(data: Partial<Product>): Promise<Product> {
    const product = this.productRepo.create(data);
    return this.productRepo.save(product);
  }

  async update(id: string, data: Partial<Product>): Promise<Product> {
    await this.findOne(id); // Kiểm tra tồn tại
    await this.productRepo.update(id, data);
    return this.findOne(id);
  }

  async remove(id: string): Promise<void> {
    await this.findOne(id);
    await this.productRepo.softDelete(id); // Soft delete nếu có deletedAt column
  }
}
```

## 6. Transaction

```typescript
import { DataSource } from 'typeorm';

@Injectable()
export class OrdersService {
  constructor(private dataSource: DataSource) {}

  async createOrder(userId: string, items: Array<{ productId: string; quantity: number }>) {
    return this.dataSource.transaction(async (manager) => {
      let total = 0;
      const orderItems: Partial<OrderItem>[] = [];

      for (const item of items) {
        const product = await manager.findOne(Product, {
          where: { id: item.productId },
          lock: { mode: 'pessimistic_write' }, // Lock row để tránh race condition
        });

        if (!product || product.stock < item.quantity) {
          throw new BadRequestException(`Sản phẩm ${product?.name} không đủ hàng`);
        }

        await manager.decrement(Product, { id: item.productId }, 'stock', item.quantity);

        const subtotal = Number(product.price) * item.quantity;
        total += subtotal;
        orderItems.push({ product, quantity: item.quantity, price: product.price });
      }

      const order = manager.create(Order, { userId, total, status: 'pending' });
      await manager.save(order);

      const savedItems = orderItems.map(i => manager.create(OrderItem, { ...i, order }));
      await manager.save(savedItems);

      return order;
    });
  }
}
```

## 7. Migration

Trong production không dùng `synchronize: true`, thay bằng migration:

```bash
# Tạo migration
npx typeorm migration:generate src/migrations/CreateProducts -d src/data-source.ts

# Chạy migration
npx typeorm migration:run -d src/data-source.ts

# Rollback
npx typeorm migration:revert -d src/data-source.ts
```

`src/data-source.ts`:
```typescript
import { DataSource } from 'typeorm';

export default new DataSource({
  type: 'mysql',
  host: process.env.DB_HOST,
  port: Number(process.env.DB_PORT),
  username: process.env.DB_USER,
  password: process.env.DB_PASS,
  database: process.env.DB_NAME,
  entities: ['src/**/*.entity.ts'],
  migrations: ['src/migrations/*.ts'],
});
```

## 8. Kết luận

TypeORM + NestJS là bộ đôi mạnh mẽ cho SQL database:

- **Entity** = định nghĩa bảng với decorator
- **Repository** = query builder fluent, type-safe
- **Transaction** = đảm bảo ACID khi thao tác nhiều bảng
- **Migration** = quản lý schema thay đổi an toàn trong production
- Hỗ trợ `@OneToMany`, `@ManyToOne`, `@ManyToMany` để map quan hệ
