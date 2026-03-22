---
layout: article
title: PostgreSQL – Giới thiệu và so sánh với MySQL
tags: [postgresql, mysql, database, nodejs, sql]
---
PostgreSQL và MySQL là hai database quan hệ phổ biến nhất. Bài này so sánh điểm mạnh/yếu và hướng dẫn dùng PostgreSQL với NodeJS/NestJS qua TypeORM.

## 1. So sánh PostgreSQL vs MySQL

| Tiêu chí | PostgreSQL | MySQL |
|---------|-----------|-------|
| **JSON** | JSONB (indexed, queryable) | JSON (basic) |
| **Array** | Native array type | Không có |
| **Full-text search** | Mạnh, built-in | Cơ bản |
| **Window functions** | Đầy đủ | Hạn chế (MySQL 8+) |
| **CTE (WITH)** | Đầy đủ | MySQL 8+ |
| **ACID** | Strict | Phụ thuộc storage engine |
| **Replication** | Streaming + Logical | Binary log |
| **Extensions** | PostGIS, pgvector, ... | Ít hơn |
| **Performance write** | Tốt | Rất tốt (InnoDB) |
| **Performance read** | Tốt | Tốt |

**Chọn PostgreSQL khi**: Cần JSON phức tạp, GIS, Full-text search, complex queries.
**Chọn MySQL khi**: Ecosystem quen thuộc, cần tốc độ write cao, simple queries.

## 2. Cài đặt với Docker

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
      POSTGRES_DB: mydb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

## 3. NestJS + TypeORM + PostgreSQL

```bash
npm install @nestjs/typeorm typeorm pg
```

```typescript
// app.module.ts
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: process.env.DB_HOST,
      port: Number(process.env.DB_PORT ?? 5432),
      username: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      database: process.env.DB_NAME,
      entities: [__dirname + '/**/*.entity{.ts,.js}'],
      synchronize: process.env.NODE_ENV !== 'production',
      ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : false,
    }),
  ],
})
export class AppModule {}
```

## 4. Entity với PostgreSQL-specific features

```typescript
// src/products/product.entity.ts
import {
  Entity, PrimaryGeneratedColumn, Column,
  Index, CreateDateColumn, UpdateDateColumn
} from 'typeorm';

@Entity('products')
export class Product {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Index()
  @Column({ length: 255 })
  name: string;

  @Column('decimal', { precision: 10, scale: 2 })
  price: number;

  @Column({ default: 0 })
  stock: number;

  // JSONB — lưu attributes linh hoạt
  @Column({ type: 'jsonb', nullable: true })
  attributes: Record<string, any>;

  // Array type — PostgreSQL native
  @Column({ type: 'text', array: true, default: [] })
  tags: string[];

  // Full-text search vector
  @Column({
    type: 'tsvector',
    nullable: true,
    select: false,  // Không trả về mặc định
  })
  searchVector: any;

  @Column({ nullable: true })
  categoryId: string;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

## 5. JSONB Queries

```typescript
// Tìm sản phẩm có color = 'red' trong attributes JSONB
const products = await this.productRepo
  .createQueryBuilder('p')
  .where("p.attributes->>'color' = :color", { color: 'red' })
  .getMany();

// Tìm với nested JSON
const products = await this.productRepo
  .createQueryBuilder('p')
  .where("p.attributes->'specs'->>'ram' = :ram", { ram: '8GB' })
  .getMany();

// Dùng @> operator (contains)
const products = await this.productRepo
  .createQueryBuilder('p')
  .where('p.attributes @> :attrs', { attrs: JSON.stringify({ brand: 'Samsung' }) })
  .getMany();
```

## 6. Array Operations

```typescript
// Tìm sản phẩm có tag 'sale'
const products = await this.productRepo
  .createQueryBuilder('p')
  .where(':tag = ANY(p.tags)', { tag: 'sale' })
  .getMany();

// Tìm sản phẩm có tất cả tags ['sale', 'new']
const products = await this.productRepo
  .createQueryBuilder('p')
  .where('p.tags @> :tags', { tags: ['sale', 'new'] })
  .getMany();
```

## 7. Full-text Search

```typescript
// Tạo index và trigger để tự động update search vector
// migration
await queryRunner.query(`
  ALTER TABLE products ADD COLUMN search_vector tsvector;

  CREATE INDEX products_search_idx ON products USING gin(search_vector);

  CREATE OR REPLACE FUNCTION update_products_search_vector()
  RETURNS TRIGGER AS $$
  BEGIN
    NEW.search_vector =
      setweight(to_tsvector('simple', COALESCE(NEW.name, '')), 'A') ||
      setweight(to_tsvector('simple', COALESCE(NEW.description, '')), 'B');
    RETURN NEW;
  END;
  $$ LANGUAGE plpgsql;

  CREATE TRIGGER products_search_update
  BEFORE INSERT OR UPDATE ON products
  FOR EACH ROW EXECUTE FUNCTION update_products_search_vector();
`);

// Query full-text search
const results = await this.productRepo
  .createQueryBuilder('p')
  .where("p.search_vector @@ plainto_tsquery('simple', :query)", { query: keyword })
  .orderBy("ts_rank(p.search_vector, plainto_tsquery('simple', :query))", 'DESC')
  .setParameter('query', keyword)
  .getMany();
```

## 8. Window Functions

```typescript
// Rank sản phẩm theo doanh thu trong từng category
const ranked = await this.dataSource.query(`
  SELECT
    id, name, category_id,
    SUM(quantity) as total_sold,
    RANK() OVER (
      PARTITION BY category_id
      ORDER BY SUM(quantity) DESC
    ) as rank_in_category
  FROM order_items oi
  JOIN products p ON p.id = oi.product_id
  GROUP BY p.id, p.name, p.category_id
`);

// Tính running total
const runningTotal = await this.dataSource.query(`
  SELECT
    date,
    revenue,
    SUM(revenue) OVER (ORDER BY date) as cumulative_revenue
  FROM daily_revenue
  ORDER BY date
`);
```

## 9. pgvector — Vector Database trong PostgreSQL

```bash
# Extension cho AI/embedding similarity search
```

```sql
-- Kích hoạt extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Tạo column vector
ALTER TABLE documents ADD COLUMN embedding vector(1536);

-- Tạo index
CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops);

-- Similarity search
SELECT id, content, 1 - (embedding <=> '[0.1, 0.2, ...]') as similarity
FROM documents
ORDER BY embedding <=> '[0.1, 0.2, ...]'
LIMIT 5;
```

Dùng pgvector thay vì Qdrant khi đã có PostgreSQL và không cần scale vector search riêng.

## 10. Kết luận

PostgreSQL mạnh hơn MySQL ở:
- **JSONB**: Lưu và query JSON phức tạp với index
- **Array**: Native array type tiện lợi
- **Full-text search**: Built-in, không cần Elasticsearch cho use case đơn giản
- **Window functions**: Analytics queries mạnh mẽ
- **pgvector**: Tích hợp AI embedding trực tiếp vào database

Với NodeJS/NestJS, TypeORM hỗ trợ tốt cả hai — migrate từ MySQL sang PostgreSQL khá dễ nếu dùng TypeORM.
