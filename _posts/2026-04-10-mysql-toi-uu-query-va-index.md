---
layout: article
title: MySQL – Tối ưu Query và Index
tags: [mysql, database, optimization, index, sql]
---
MySQL vẫn là một trong những cơ sở dữ liệu quan hệ phổ biến nhất. Khi dữ liệu lớn dần, việc viết query đúng và sử dụng index hợp lý sẽ quyết định hiệu suất của toàn bộ ứng dụng.

## 1. EXPLAIN — Phân tích Query

Trước khi tối ưu, luôn dùng `EXPLAIN` để xem MySQL thực thi query như thế nào:

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 5 AND status = 'completed';
```

Các cột quan trọng cần chú ý:

| Cột | Ý nghĩa |
|-----|---------|
| `type` | Kiểu truy cập: `ALL` (xấu) → `index` → `range` → `ref` → `eq_ref` → `const` (tốt nhất) |
| `key` | Index đang được dùng |
| `rows` | Số hàng MySQL ước tính phải scan |
| `Extra` | `Using filesort`, `Using temporary` là dấu hiệu cần tối ưu |

```sql
-- Dùng EXPLAIN ANALYZE (MySQL 8.0+) để xem thời gian thực tế
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 5;
```

## 2. Index cơ bản

### Tạo Index

```sql
-- Single column index
CREATE INDEX idx_email ON users(email);

-- Unique index
CREATE UNIQUE INDEX idx_email_unique ON users(email);

-- Composite index (thứ tự cột rất quan trọng!)
CREATE INDEX idx_user_status ON orders(user_id, status);

-- Full-text index
CREATE FULLTEXT INDEX idx_content ON posts(title, content);
```

### Quy tắc dùng Composite Index

Index `(user_id, status, created_at)` sẽ hỗ trợ các query:
- `WHERE user_id = ?` ✅
- `WHERE user_id = ? AND status = ?` ✅
- `WHERE user_id = ? AND status = ? AND created_at > ?` ✅
- `WHERE status = ?` ❌ (không dùng được — bỏ qua cột đầu)
- `WHERE user_id = ? AND created_at > ?` ⚠️ (chỉ dùng được phần `user_id`)

> **Quy tắc "Leftmost Prefix"**: MySQL chỉ dùng được các cột từ trái sang của composite index.

### Xem Index hiện có

```sql
SHOW INDEX FROM orders;

-- Kiểm tra index không được dùng (performance_schema)
SELECT object_schema, object_name, index_name
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE index_name IS NOT NULL
  AND count_star = 0
  AND object_schema = 'mydb';
```

## 3. Các lỗi Query phổ biến

### Không dùng được Index vì function

```sql
-- ❌ Sai: MySQL không thể dùng index vì phải tính toán từng row
SELECT * FROM users WHERE YEAR(created_at) = 2024;

-- ✅ Đúng: Dùng range để tận dụng index
SELECT * FROM users
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
```

### Implicit type conversion

```sql
-- ❌ Sai: phone là VARCHAR, so sánh với số → full scan
SELECT * FROM users WHERE phone = 0123456789;

-- ✅ Đúng: so sánh cùng type
SELECT * FROM users WHERE phone = '0123456789';
```

### Wildcard ở đầu chuỗi

```sql
-- ❌ Full scan — không dùng được index
SELECT * FROM products WHERE name LIKE '%iphone%';

-- ✅ Dùng được index (wildcard chỉ ở cuối)
SELECT * FROM products WHERE name LIKE 'iphone%';

-- ✅ Dùng FULLTEXT INDEX cho full-text search
SELECT * FROM products WHERE MATCH(name) AGAINST('iphone' IN BOOLEAN MODE);
```

### SELECT *

```sql
-- ❌ Kéo toàn bộ dữ liệu không cần thiết
SELECT * FROM users WHERE city = 'Hanoi';

-- ✅ Chỉ lấy cột cần dùng → có thể dùng Covering Index
SELECT id, name, email FROM users WHERE city = 'Hanoi';
```

## 4. Covering Index

Khi tất cả cột trong SELECT và WHERE đều nằm trong index, MySQL không cần đọc table chính (heap):

```sql
-- Tạo covering index cho query này
CREATE INDEX idx_city_covering ON users(city, name, email);

-- Query này sẽ chạy thuần từ index, không đọc table
SELECT name, email FROM users WHERE city = 'Hanoi';
-- Extra: Using index ← dấu hiệu tốt
```

## 5. Tối ưu JOIN

```sql
-- ✅ Luôn có index trên cột JOIN
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- ✅ Lọc trước khi JOIN (dùng subquery hoặc CTE)
SELECT u.name, o.total
FROM users u
JOIN (
  SELECT user_id, SUM(total) AS total
  FROM orders
  WHERE status = 'completed'
  GROUP BY user_id
) o ON u.id = o.user_id;
```

## 6. Phân trang hiệu quả

```sql
-- ❌ OFFSET lớn rất chậm (MySQL phải scan và bỏ qua N hàng)
SELECT * FROM posts ORDER BY id DESC LIMIT 10 OFFSET 100000;

-- ✅ Keyset pagination (Cursor-based) — luôn nhanh
SELECT * FROM posts
WHERE id < :last_seen_id  -- ID của record cuối trang trước
ORDER BY id DESC
LIMIT 10;
```

## 7. Slow Query Log

Bật Slow Query Log để phát hiện query chậm:

```sql
-- Bật slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- Log query > 1 giây
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

-- Xem cấu hình hiện tại
SHOW VARIABLES LIKE 'slow_query%';
```

Phân tích file log:
```bash
# Tóm tắt các query chậm nhất
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log
```

## 8. Checklist tối ưu

- [ ] Dùng `EXPLAIN` trước khi deploy query mới
- [ ] Index mọi cột dùng trong `WHERE`, `JOIN ON`, `ORDER BY`
- [ ] Không dùng function trên cột có index trong `WHERE`
- [ ] Tránh `SELECT *` — chỉ lấy cột cần dùng
- [ ] Dùng Covering Index cho các query đọc nhiều
- [ ] Tránh `OFFSET` lớn — dùng keyset pagination
- [ ] Bật Slow Query Log trên production

## 9. Kết luận

Tối ưu MySQL không phải là thêm thật nhiều index mà là **thêm đúng index**. Quá nhiều index sẽ làm chậm `INSERT`/`UPDATE`. Hãy luôn đo lường với `EXPLAIN` trước và sau khi thêm index để xác nhận hiệu quả.
