---
layout: article
title: NestJS – E2E Testing (End-to-End Testing)
tags: [nestjs, testing, e2e, jest, supertest, nodejs]
---
E2E Testing kiểm tra toàn bộ luồng từ HTTP request → Controller → Service → Database — đảm bảo các layer phối hợp đúng. NestJS dùng **Supertest** để test HTTP endpoints thực tế.

## 1. Cấu trúc E2E test

```
test/
  app.e2e-spec.ts     ← Test file
  jest-e2e.json       ← Jest config cho E2E
```

```json
// jest-e2e.json
{
  "moduleFileExtensions": ["js", "json", "ts"],
  "rootDir": ".",
  "testEnvironment": "node",
  "testRegex": ".e2e-spec.ts$",
  "transform": { "^.+\\.(t|j)s$": "ts-jest" }
}
```

```bash
npm run test:e2e
```

## 2. Setup cơ bản

```typescript
// test/app.e2e-spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';

describe('AppController (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();

    // Cấu hình giống main.ts
    app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
    app.setGlobalPrefix('api');

    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  it('GET /api/health', () => {
    return request(app.getHttpServer())
      .get('/api/health')
      .expect(200)
      .expect({ status: 'ok' });
  });
});
```

## 3. Test Authentication Flow

```typescript
// test/auth.e2e-spec.ts
describe('Auth (e2e)', () => {
  let app: INestApplication;
  let accessToken: string;

  beforeAll(async () => {
    // Setup app...
  });

  describe('POST /api/auth/register', () => {
    it('nên đăng ký thành công', async () => {
      const res = await request(app.getHttpServer())
        .post('/api/auth/register')
        .send({ name: 'Test User', email: 'test@e2e.com', password: '123456' })
        .expect(201);

      expect(res.body).toMatchObject({
        _id: expect.any(String),
        name: 'Test User',
        email: 'test@e2e.com',
      });
      expect(res.body.password).toBeUndefined(); // Password không được trả về
    });

    it('nên trả về 409 khi email đã tồn tại', () => {
      return request(app.getHttpServer())
        .post('/api/auth/register')
        .send({ name: 'Dup', email: 'test@e2e.com', password: '123456' })
        .expect(409);
    });

    it('nên trả về 400 khi thiếu field bắt buộc', () => {
      return request(app.getHttpServer())
        .post('/api/auth/register')
        .send({ email: 'missing-name@test.com' })
        .expect(400);
    });
  });

  describe('POST /api/auth/login', () => {
    it('nên login thành công và trả về token', async () => {
      const res = await request(app.getHttpServer())
        .post('/api/auth/login')
        .send({ email: 'test@e2e.com', password: '123456' })
        .expect(200);

      expect(res.body.access_token).toBeDefined();
      accessToken = res.body.access_token; // Lưu để dùng ở test sau
    });

    it('nên trả về 401 khi sai mật khẩu', () => {
      return request(app.getHttpServer())
        .post('/api/auth/login')
        .send({ email: 'test@e2e.com', password: 'wrong' })
        .expect(401);
    });
  });

  describe('GET /api/auth/me (protected)', () => {
    it('nên trả về profile khi có token hợp lệ', () => {
      return request(app.getHttpServer())
        .get('/api/auth/me')
        .set('Authorization', `Bearer ${accessToken}`)
        .expect(200)
        .expect(res => {
          expect(res.body.email).toBe('test@e2e.com');
        });
    });

    it('nên trả về 401 khi không có token', () => {
      return request(app.getHttpServer())
        .get('/api/auth/me')
        .expect(401);
    });
  });
});
```

## 4. Test CRUD với Database thật

```typescript
// test/users.e2e-spec.ts
describe('Users (e2e)', () => {
  let app: INestApplication;
  let adminToken: string;
  let createdUserId: string;

  beforeAll(async () => {
    const moduleFixture = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
    await app.init();

    // Đăng nhập admin
    const loginRes = await request(app.getHttpServer())
      .post('/api/auth/login')
      .send({ email: process.env.ADMIN_EMAIL, password: process.env.ADMIN_PASSWORD });
    adminToken = loginRes.body.access_token;
  });

  afterAll(async () => {
    // Cleanup: xóa user test
    if (createdUserId) {
      await request(app.getHttpServer())
        .delete(`/api/users/${createdUserId}`)
        .set('Authorization', `Bearer ${adminToken}`);
    }
    await app.close();
  });

  it('POST /api/users — tạo user mới', async () => {
    const res = await request(app.getHttpServer())
      .post('/api/users')
      .set('Authorization', `Bearer ${adminToken}`)
      .send({ name: 'E2E User', email: 'e2e@test.com', password: 'test123' })
      .expect(201);

    createdUserId = res.body._id;
    expect(res.body.name).toBe('E2E User');
  });

  it('GET /api/users/:id — lấy user vừa tạo', async () => {
    await request(app.getHttpServer())
      .get(`/api/users/${createdUserId}`)
      .set('Authorization', `Bearer ${adminToken}`)
      .expect(200)
      .expect(res => expect(res.body._id).toBe(createdUserId));
  });

  it('PATCH /api/users/:id — cập nhật user', async () => {
    await request(app.getHttpServer())
      .patch(`/api/users/${createdUserId}`)
      .set('Authorization', `Bearer ${adminToken}`)
      .send({ name: 'Updated Name' })
      .expect(200)
      .expect(res => expect(res.body.name).toBe('Updated Name'));
  });

  it('DELETE /api/users/:id — xóa user', async () => {
    await request(app.getHttpServer())
      .delete(`/api/users/${createdUserId}`)
      .set('Authorization', `Bearer ${adminToken}`)
      .expect(200);

    createdUserId = ''; // Đã xóa, không cần cleanup

    // Verify đã xóa
    await request(app.getHttpServer())
      .get(`/api/users/${createdUserId}`)
      .set('Authorization', `Bearer ${adminToken}`)
      .expect(404);
  });
});
```

## 5. Dùng In-Memory Database

Để test nhanh hơn mà không cần MongoDB thật:

```bash
npm install -D mongodb-memory-server
```

```typescript
// test/setup.ts
import { MongoMemoryServer } from 'mongodb-memory-server';
import mongoose from 'mongoose';

let mongod: MongoMemoryServer;

beforeAll(async () => {
  mongod = await MongoMemoryServer.create();
  const uri = mongod.getUri();
  await mongoose.connect(uri);
});

afterAll(async () => {
  await mongoose.disconnect();
  await mongod.stop();
});

afterEach(async () => {
  // Xóa toàn bộ data sau mỗi test
  const collections = mongoose.connection.collections;
  for (const key in collections) {
    await collections[key].deleteMany({});
  }
});
```

```json
// jest-e2e.json
{
  "globalSetup": "./test/setup.ts",
  "testEnvironment": "node"
}
```

## 6. Test File Upload

```typescript
it('POST /api/upload — upload file', async () => {
  const res = await request(app.getHttpServer())
    .post('/api/upload')
    .set('Authorization', `Bearer ${accessToken}`)
    .attach('file', Buffer.from('test file content'), 'test.txt')
    .expect(201);

  expect(res.body.url).toMatch(/uploads\/.*\.txt/);
});
```

## 7. GitHub Actions — Chạy E2E tự động

```yaml
# .github/workflows/e2e.yml
- name: Run E2E tests
  env:
    MONGODB_URI: mongodb://localhost:27017/testdb
    JWT_SECRET: test-secret
    NODE_ENV: test
  run: npm run test:e2e
  services:
    mongodb:
      image: mongo:6
      ports:
        - 27017:27017
```

## 8. Kết luận

- **Supertest**: Test HTTP endpoints thực tế, không cần start server
- **`beforeAll`/`afterAll`**: Setup và teardown app một lần cho cả describe block
- **Sequential test**: Lưu ID từ POST để dùng cho GET/PATCH/DELETE
- **In-memory MongoDB**: Nhanh hơn, không cần Docker khi CI/CD đơn giản
- **Cleanup**: Xóa test data sau mỗi test để tránh interference

E2E test bắt được lỗi mà unit test bỏ qua — luôn có E2E cho critical flows (auth, payment, order).
