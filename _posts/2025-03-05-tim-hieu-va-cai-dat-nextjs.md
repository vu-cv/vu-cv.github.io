---
layout: article
title: Tìm hiểu & Cài đặt NextJS 14
tags: [nextjs, react, frontend, typescript]
---
Next.js là framework React phổ biến nhất hiện nay, được phát triển bởi Vercel. Phiên bản 14 giới thiệu App Router, Server Components, và Server Actions — những tính năng thay đổi cách chúng ta xây dựng ứng dụng web hiện đại.

## 1. Giới thiệu

**Next.js 14** nổi bật với:
- **App Router** — kiến trúc routing mới dựa trên thư mục `app/`
- **React Server Components (RSC)** — render phía server mặc định, giảm JavaScript phía client
- **Server Actions** — gọi server function trực tiếp từ component mà không cần API route
- **Partial Prerendering** — kết hợp static và dynamic rendering trong cùng một trang
- **Turbopack** — bundler mới nhanh hơn Webpack nhiều lần

## 2. Yêu cầu

- **Node.js** >= 18.17.0
- **npm**, **yarn**, hoặc **pnpm**

## 3. Tạo project mới

```bash
npx create-next-app@latest my-next-app
```

Trả lời các câu hỏi cấu hình:
```
✔ Would you like to use TypeScript? → Yes
✔ Would you like to use ESLint? → Yes
✔ Would you like to use Tailwind CSS? → Yes
✔ Would you like to use `src/` directory? → No
✔ Would you like to use App Router? (recommended) → Yes
✔ Would you like to customize the default import alias? → No
```

## 4. Cấu trúc project (App Router)

```
my-next-app/
├── app/
│   ├── layout.tsx       ← Root layout (wrapper cho toàn bộ app)
│   ├── page.tsx         ← Trang chủ (/)
│   ├── globals.css
│   ├── about/
│   │   └── page.tsx     ← Trang /about
│   └── blog/
│       ├── page.tsx     ← Trang /blog
│       └── [slug]/
│           └── page.tsx ← Trang /blog/:slug (dynamic route)
├── components/
├── public/
├── next.config.js
├── tailwind.config.ts
└── tsconfig.json
```

## 5. Các khái niệm cốt lõi

### Server Component (mặc định)

Mọi component trong `app/` đều là **Server Component** theo mặc định — chúng chạy trên server, có thể `async/await`, truy cập database trực tiếp:

```tsx
// app/page.tsx — Server Component
async function HomePage() {
  // Fetch data trực tiếp, không cần useEffect
  const data = await fetch('https://api.example.com/posts');
  const posts = await data.json();

  return (
    <main>
      <h1>Danh sách bài viết</h1>
      {posts.map((post: any) => (
        <div key={post.id}>{post.title}</div>
      ))}
    </main>
  );
}

export default HomePage;
```

### Client Component

Khi cần interactivity (state, event, browser APIs), thêm `'use client'` ở đầu file:

```tsx
'use client';

import { useState } from 'react';

export function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      Đã nhấn {count} lần
    </button>
  );
}
```

### Layout

Layout bao bọc các trang con và **giữ nguyên state** khi navigate:

```tsx
// app/layout.tsx
export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="vi">
      <body>
        <header>Header</header>
        <main>{children}</main>
        <footer>Footer</footer>
      </body>
    </html>
  );
}
```

### Dynamic Routes

```tsx
// app/blog/[slug]/page.tsx
export default function BlogPost({ params }: { params: { slug: string } }) {
  return <h1>Bài viết: {params.slug}</h1>;
}
```

## 6. Navigation

Dùng component `Link` của Next.js để navigate (có prefetch tự động):

```tsx
import Link from 'next/link';

export function Nav() {
  return (
    <nav>
      <Link href="/">Trang chủ</Link>
      <Link href="/about">Giới thiệu</Link>
      <Link href="/blog">Blog</Link>
    </nav>
  );
}
```

## 7. Metadata

Thêm SEO metadata cho từng trang:

```tsx
// app/page.tsx
import { Metadata } from 'next';

export const metadata: Metadata = {
  title: 'Trang chủ | My App',
  description: 'Mô tả trang chủ',
  openGraph: {
    title: 'My App',
    images: ['/og-image.png'],
  },
};

export default function HomePage() {
  return <h1>Trang chủ</h1>;
}
```

## 8. Chạy project

```bash
cd my-next-app

# Development
npm run dev

# Build production
npm run build

# Chạy production
npm start
```

Truy cập [http://localhost:3000](http://localhost:3000){:target="_blank"}

## 9. Tối ưu Image

Next.js có component `Image` tích hợp sẵn lazy loading và tối ưu định dạng:

```tsx
import Image from 'next/image';

export function Avatar() {
  return (
    <Image
      src="/avatar.jpg"
      alt="Avatar"
      width={100}
      height={100}
      priority // Ưu tiên load sớm cho LCP image
    />
  );
}
```

## 10. Kết luận

Next.js 14 với App Router mang đến mô hình phát triển mạnh mẽ:

- **Server Components** — fetch data trực tiếp, không cần API
- **Client Components** — dùng khi cần state/event
- **Layouts** — share UI giữa các trang
- **Dynamic Routes** — URL params linh hoạt
- **Built-in optimizations** — Image, Font, Script

Bài tiếp theo: **NextJS – Server Actions & API Routes**.
