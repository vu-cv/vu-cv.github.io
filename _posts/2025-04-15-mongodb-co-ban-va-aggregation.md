---
layout: article
title: MongoDB – Các thao tác cơ bản và Aggregation Pipeline
tags: [mongodb, database, aggregation, nosql]
---
MongoDB là cơ sở dữ liệu NoSQL dạng document phổ biến nhất. Bài này tổng hợp các thao tác CRUD cơ bản và đặc biệt đi sâu vào **Aggregation Pipeline** — công cụ mạnh mẽ để xử lý và phân tích dữ liệu phức tạp.

## 1. Cài đặt và kết nối

### Cài MongoDB locally (macOS)

```bash
brew tap mongodb/brew
brew install mongodb-community
brew services start mongodb-community
```

### Kết nối bằng mongosh

```bash
mongosh "mongodb://localhost:27017"
```

### Tạo database và collection

```javascript
use mydb
db.createCollection('users')
```

## 2. CRUD cơ bản

### Insert

```javascript
// Insert một document
db.users.insertOne({
  name: 'Nguyen Van A',
  email: 'a@example.com',
  age: 25,
  city: 'Hanoi',
  createdAt: new Date()
})

// Insert nhiều document
db.users.insertMany([
  { name: 'Tran Thi B', email: 'b@example.com', age: 30, city: 'HCM' },
  { name: 'Le Van C',   email: 'c@example.com', age: 22, city: 'Hanoi' },
])
```

### Find

```javascript
// Lấy tất cả
db.users.find()

// Lọc theo điều kiện
db.users.find({ city: 'Hanoi' })

// Toán tử so sánh
db.users.find({ age: { $gte: 25, $lte: 35 } })

// Nhiều điều kiện (AND ngầm định)
db.users.find({ city: 'Hanoi', age: { $gte: 25 } })

// OR
db.users.find({ $or: [{ city: 'Hanoi' }, { city: 'HCM' }] })

// Chỉ lấy một số field (projection)
db.users.find({ city: 'Hanoi' }, { name: 1, email: 1, _id: 0 })

// Sắp xếp, phân trang
db.users.find().sort({ age: -1 }).skip(0).limit(10)
```

### Update

```javascript
// Update một document
db.users.updateOne(
  { email: 'a@example.com' },
  { $set: { age: 26 }, $currentDate: { updatedAt: true } }
)

// Update nhiều document
db.users.updateMany(
  { city: 'HCM' },
  { $set: { city: 'Ho Chi Minh' } }
)

// Tăng giá trị số
db.users.updateOne({ email: 'a@example.com' }, { $inc: { age: 1 } })

// Upsert — nếu không tồn tại thì tạo mới
db.users.updateOne(
  { email: 'd@example.com' },
  { $set: { name: 'Pham D', age: 28 } },
  { upsert: true }
)
```

### Delete

```javascript
db.users.deleteOne({ email: 'a@example.com' })
db.users.deleteMany({ age: { $lt: 18 } })
```

## 3. Aggregation Pipeline

Aggregation Pipeline xử lý documents qua nhiều **stage** (giai đoạn) liên tiếp, mỗi stage nhận output của stage trước.

### Cấu trúc

```javascript
db.collection.aggregate([
  { $stage1: { ... } },
  { $stage2: { ... } },
  // ...
])
```

### $match — Lọc document

```javascript
db.orders.aggregate([
  { $match: { status: 'completed', total: { $gte: 100000 } } }
])
```

Tương đương `WHERE` trong SQL.

### $group — Nhóm và tính toán

```javascript
// Đếm số đơn hàng theo trạng thái
db.orders.aggregate([
  {
    $group: {
      _id: '$status',
      count: { $sum: 1 },
      totalRevenue: { $sum: '$total' },
      avgOrder: { $avg: '$total' },
      maxOrder: { $max: '$total' },
    }
  }
])
```

### $project — Chọn và biến đổi field

```javascript
db.users.aggregate([
  {
    $project: {
      _id: 0,
      fullName: '$name',
      email: 1,
      isAdult: { $gte: ['$age', 18] },
      // Tạo field mới từ phép tính
      birthYear: { $subtract: [2026, '$age'] },
    }
  }
])
```

### $sort, $skip, $limit — Sắp xếp và phân trang

```javascript
db.orders.aggregate([
  { $match: { status: 'completed' } },
  { $sort: { total: -1 } },
  { $skip: 0 },
  { $limit: 10 },
])
```

### $lookup — JOIN giữa hai collection

```javascript
// orders: { userId, total, ... }
// users: { _id, name, email, ... }

db.orders.aggregate([
  {
    $lookup: {
      from: 'users',
      localField: 'userId',
      foreignField: '_id',
      as: 'user',
    }
  },
  { $unwind: '$user' }, // Flatten mảng user thành object
  {
    $project: {
      total: 1,
      'user.name': 1,
      'user.email': 1,
    }
  }
])
```

### Ví dụ thực tế: Doanh thu theo tháng

```javascript
db.orders.aggregate([
  { $match: { status: 'completed' } },
  {
    $group: {
      _id: {
        year:  { $year: '$createdAt' },
        month: { $month: '$createdAt' },
      },
      totalRevenue: { $sum: '$total' },
      orderCount:   { $sum: 1 },
    }
  },
  { $sort: { '_id.year': 1, '_id.month': 1 } },
  {
    $project: {
      _id: 0,
      year:  '$_id.year',
      month: '$_id.month',
      totalRevenue: 1,
      orderCount: 1,
    }
  }
])
```

### $addFields — Thêm field tính toán

```javascript
db.products.aggregate([
  {
    $addFields: {
      discountedPrice: {
        $multiply: ['$price', { $subtract: [1, '$discountRate'] }]
      }
    }
  }
])
```

## 4. Index — Tăng tốc Query

```javascript
// Tạo index đơn
db.users.createIndex({ email: 1 }, { unique: true })

// Compound index
db.orders.createIndex({ userId: 1, createdAt: -1 })

// Text index để tìm kiếm full-text
db.products.createIndex({ name: 'text', description: 'text' })
db.products.find({ $text: { $search: 'laptop gaming' } })

// Xem kế hoạch thực thi query
db.users.find({ city: 'Hanoi' }).explain('executionStats')

// Liệt kê tất cả index
db.users.getIndexes()
```

## 5. Kết luận

MongoDB mạnh ở tính linh hoạt của schema và sức mạnh của Aggregation Pipeline:

- **CRUD**: `insertOne/Many`, `find`, `updateOne/Many`, `deleteOne/Many`
- **Aggregation**: `$match` → `$group` → `$sort` → `$project` → `$lookup`
- **Index**: luôn tạo index cho các field thường xuyên query để tránh full collection scan

Bài tiếp theo: **MySQL – Tối ưu Query và Index**.
