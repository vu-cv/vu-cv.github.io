---
layout: article
title: NextJS – Deploy lên Vercel & AWS Amplify
tags: [nextjs, deploy, vercel, aws, amplify]
---
Sau khi xây dựng xong ứng dụng Next.js, bước tiếp theo là deploy lên môi trường production. Bài này hướng dẫn hai cách phổ biến nhất: **Vercel** (nhanh, zero-config) và **AWS Amplify** (linh hoạt, tích hợp AWS ecosystem).

## 1. Deploy lên Vercel

Vercel là công ty tạo ra Next.js nên việc deploy có thể nói là seamless nhất.

### Bước 1: Chuẩn bị repository

Đảm bảo project đã được push lên GitHub, GitLab, hoặc Bitbucket.

### Bước 2: Tạo tài khoản Vercel

Truy cập [vercel.com](https://vercel.com){:target="_blank"} và đăng ký bằng tài khoản GitHub.

### Bước 3: Import project

1. Chọn **Add New → Project**
2. Kết nối GitHub repository
3. Vercel tự detect Next.js và cấu hình sẵn — chỉ cần nhấn **Deploy**

### Bước 4: Cấu hình Environment Variables

Trên Vercel Dashboard:
- Vào **Settings → Environment Variables**
- Thêm các biến như `DATABASE_URL`, `JWT_SECRET`, v.v.

```bash
# Các biến production thường gặp
NEXTAUTH_URL=https://your-domain.vercel.app
DATABASE_URL=mongodb+srv://...
JWT_SECRET=production-secret
```

### Bước 5: Custom Domain

- Vào **Settings → Domains**
- Thêm domain của bạn và cập nhật DNS theo hướng dẫn

### Deploy tự động (CI/CD)

Vercel tự động deploy khi bạn push lên `main` branch. Mỗi Pull Request sẽ có preview URL riêng — rất tiện để review.

```bash
# Hoặc deploy thủ công qua CLI
npm install -g vercel
vercel login
vercel --prod
```

## 2. Deploy lên AWS Amplify

AWS Amplify phù hợp khi bạn cần tích hợp sâu với các service AWS khác (S3, RDS, Cognito...).

### Bước 1: Cài đặt Amplify CLI

```bash
npm install -g @aws-amplify/cli
amplify configure
```

Lệnh `amplify configure` sẽ yêu cầu đăng nhập AWS Console và tạo IAM user.

### Bước 2: Khởi tạo Amplify trong project

```bash
cd my-next-app
amplify init
```

Trả lời các câu hỏi:
```
? Enter a name for the project: mynextapp
? Choose the type of app that you're building: javascript
? What javascript framework are you using: react
? Source Directory Path: .
? Distribution Directory Path: .next
? Build Command: npm run build
? Start Command: npm start
```

### Bước 3: Thêm Hosting

```bash
amplify add hosting
```

Chọn **Managed hosting with custom domains (Amplify Console)** và **Continuous deployment**.

### Bước 4: Publish

```bash
amplify publish
```

Lệnh này sẽ:
1. Build project (`npm run build`)
2. Upload lên AWS
3. Trả về URL production

### Bước 5: Cấu hình qua AWS Console

Truy cập [AWS Amplify Console](https://console.aws.amazon.com/amplify){:target="_blank"}:

- Kết nối GitHub repository để tự động deploy khi push code
- Cấu hình **Environment Variables** trong phần **App settings**
- Thêm **Custom Domain** và Amplify sẽ tự cấu hình SSL

### amplify.yml — Cấu hình build

Tạo file `amplify.yml` ở root project:

```yaml
version: 1
frontend:
  phases:
    preBuild:
      commands:
        - npm ci
    build:
      commands:
        - npm run build
  artifacts:
    baseDirectory: .next
    files:
      - '**/*'
  cache:
    paths:
      - node_modules/**/*
      - .next/cache/**/*
```

## 3. So sánh Vercel vs AWS Amplify

| Tiêu chí | Vercel | AWS Amplify |
|---------|--------|-------------|
| Độ phức tạp setup | Rất đơn giản | Trung bình |
| Free tier | Tốt (100GB bandwidth) | 1000 build phút/tháng |
| Tích hợp AWS | Hạn chế | Hoàn toàn |
| Custom server | Không (serverless) | Có |
| Edge Network | Tốt | CloudFront |
| Preview deployments | Có sẵn | Có sẵn |
| Phù hợp khi | Chỉ cần Next.js đơn giản | Dùng nhiều AWS service |

## 4. Tối ưu trước khi deploy

### Kiểm tra bundle size

```bash
npm run build
# Xem kết quả phân tích bundle trong output
```

Thêm vào `next.config.js` để phân tích chi tiết:
```javascript
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});

module.exports = withBundleAnalyzer({});
```

```bash
ANALYZE=true npm run build
```

### Cấu hình next.config.js cho production

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'standalone',          // Tối ưu cho Docker/server deployment
  compress: true,                // Gzip compression
  images: {
    domains: ['your-cdn.com'],   // Cho phép load ảnh từ domain ngoài
  },
  headers: async () => [
    {
      source: '/(.*)',
      headers: [
        { key: 'X-Frame-Options', value: 'DENY' },
        { key: 'X-Content-Type-Options', value: 'nosniff' },
      ],
    },
  ],
};

module.exports = nextConfig;
```

## 5. Kết luận

- **Vercel**: Lựa chọn hàng đầu cho dự án Next.js — setup 5 phút, CI/CD tự động, free tier tốt
- **AWS Amplify**: Lựa chọn khi dự án nằm trong hệ sinh thái AWS — tích hợp S3, Cognito, API Gateway

Cả hai đều hỗ trợ preview URL cho mỗi PR, automatic HTTPS, và global CDN.
