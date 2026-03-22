---
layout: article
title: Docker & Docker Compose cho NestJS
tags: [docker, nestjs, devops, container, deployment]
---
Docker giúp đóng gói ứng dụng và mọi dependency vào một container, đảm bảo chạy nhất quán trên mọi môi trường. Bài này hướng dẫn containerize ứng dụng NestJS và chạy cùng MongoDB, Redis bằng Docker Compose.

## 1. Dockerfile cho NestJS

Tạo file `Dockerfile` tại root project:

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:20-alpine AS production

WORKDIR /app

# Chỉ copy những gì cần thiết
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

COPY --from=builder /app/dist ./dist

# Tạo user non-root cho bảo mật
RUN addgroup -g 1001 -S nodejs && adduser -S nestjs -u 1001
USER nestjs

EXPOSE 3000
CMD ["node", "dist/main"]
```

`.dockerignore`:
```
node_modules
dist
.env
*.md
.git
coverage
```

Build và chạy:
```bash
docker build -t my-nestjs-app .
docker run -p 3000:3000 --env-file .env my-nestjs-app
```

## 2. Docker Compose — Full Stack

Tạo `docker-compose.yml` để chạy NestJS + MongoDB + Redis cùng nhau:

```yaml
services:
  # NestJS App
  api:
    build:
      context: .
      target: production
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
      MONGODB_URI: mongodb://mongo:27017/mydb
      REDIS_HOST: redis
      REDIS_PORT: 6379
    env_file:
      - .env
    depends_on:
      mongo:
        condition: service_healthy
      redis:
        condition: service_started
    restart: unless-stopped
    networks:
      - app-network

  # MongoDB
  mongo:
    image: mongo:7
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: password
      MONGO_INITDB_DATABASE: mydb
    volumes:
      - mongo_data:/data/db
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  # Redis
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
    networks:
      - app-network

  # Mongo Express — UI quản lý MongoDB (chỉ dùng dev)
  mongo-express:
    image: mongo-express
    ports:
      - "8081:8081"
    environment:
      ME_CONFIG_MONGODB_ADMINUSERNAME: root
      ME_CONFIG_MONGODB_ADMINPASSWORD: password
      ME_CONFIG_MONGODB_URL: mongodb://root:password@mongo:27017/
    depends_on:
      - mongo
    networks:
      - app-network
    profiles:
      - dev  # Chỉ khởi động khi chạy với --profile dev

volumes:
  mongo_data:
  redis_data:

networks:
  app-network:
    driver: bridge
```

## 3. Docker Compose cho Development

Tạo `docker-compose.dev.yml` riêng để dev với hot reload:

```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - .:/app               # Mount source code để hot reload
      - /app/node_modules    # Giữ node_modules trong container
    environment:
      NODE_ENV: development
      MONGODB_URI: mongodb://mongo:27017/mydb
      REDIS_HOST: redis
    command: npm run start:dev
    depends_on:
      - mongo
      - redis

  mongo:
    image: mongo:7
    ports:
      - "27017:27017"
    volumes:
      - mongo_data_dev:/data/db

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  mongo_data_dev:
```

`Dockerfile.dev`:
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["npm", "run", "start:dev"]
```

## 4. Các lệnh thường dùng

```bash
# Khởi động toàn bộ stack
docker compose up -d

# Dev mode với hot reload
docker compose -f docker-compose.dev.yml up

# Khởi động kèm profile dev (Mongo Express)
docker compose --profile dev up -d

# Xem logs
docker compose logs -f api

# Restart một service
docker compose restart api

# Scale NestJS lên 3 instance
docker compose up -d --scale api=3

# Dừng và xóa container (giữ volume)
docker compose down

# Dừng và xóa cả volume (reset sạch DB)
docker compose down -v

# Rebuild image sau khi thay đổi code
docker compose up -d --build api

# Chạy lệnh trong container
docker compose exec api sh
docker compose exec mongo mongosh
```

## 5. Environment Variables an toàn

Không đưa `.env` vào Docker image. Dùng secrets hoặc env_file:

```yaml
# docker-compose.yml
services:
  api:
    env_file:
      - .env.production  # File này không commit lên git
```

Với production, dùng Docker Secrets:
```yaml
services:
  api:
    secrets:
      - db_password
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password

secrets:
  db_password:
    external: true  # Được tạo bằng: docker secret create db_password -
```

## 6. Health Check cho NestJS

Thêm endpoint health check trong app:

```typescript
// src/app.controller.ts
@Get('health')
health() {
  return { status: 'ok', timestamp: new Date().toISOString() };
}
```

Cấu hình trong Docker Compose:
```yaml
api:
  healthcheck:
    test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 40s
```

## 7. Kết luận

Docker + Docker Compose giải quyết bài toán "chạy được trên máy tôi":

- **Multi-stage build** — image production nhỏ, không chứa dev dependencies
- **Docker Compose** — quản lý nhiều service (app + DB + cache) như một unit
- **Volume** — dữ liệu tồn tại khi container restart
- **Network** — các service nói chuyện với nhau qua tên service (không cần IP)
- **Profile** — bật/tắt service tùy môi trường (dev tools chỉ chạy khi dev)
