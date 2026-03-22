---
layout: article
title: NextJS – Performance Optimization & Core Web Vitals
tags: [nextjs, performance, core-web-vitals, optimization, frontend]
---
Core Web Vitals là bộ chỉ số Google dùng để đánh giá UX: **LCP** (loading), **FID/INP** (interactivity), **CLS** (visual stability). NextJS có nhiều tính năng built-in giúp đạt điểm cao — bài này hướng dẫn các kỹ thuật tối ưu thực tế.

## 1. Đo lường trước khi tối ưu

```bash
# Lighthouse CLI
npm install -g lighthouse
lighthouse https://yoursite.com --output html --output-path report.html

# Web Vitals trong code
```

```typescript
// app/layout.tsx — Đo Web Vitals
'use client';
import { useReportWebVitals } from 'next/web-vitals';

export default function WebVitalsReporter() {
  useReportWebVitals((metric) => {
    console.log(metric); // { name: 'LCP', value: 1200, rating: 'good' }
    // Gửi lên analytics
    fetch('/api/metrics', {
      method: 'POST',
      body: JSON.stringify(metric),
    });
  });
  return null;
}
```

## 2. Image Optimization

`next/image` tự động: WebP/AVIF conversion, lazy loading, `srcset` responsive.

```tsx
import Image from 'next/image';

// ✅ Dùng next/image thay vì <img>
export function ProductCard({ product }) {
  return (
    <div>
      <Image
        src={product.imageUrl}
        alt={product.name}
        width={400}
        height={300}
        priority={product.isFeatured}  // LCP image → load trước
        sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
        placeholder="blur"
        blurDataURL="data:image/png;base64,..."  // Skeleton trong khi load
      />
    </div>
  );
}

// Remote images — cấu hình domain
// next.config.js
module.exports = {
  images: {
    remotePatterns: [
      { protocol: 'https', hostname: 'cdn.shopxyz.com' },
      { protocol: 'https', hostname: 'images.unsplash.com' },
    ],
  },
};
```

## 3. Font Optimization

```typescript
// app/layout.tsx
import { Inter, Be_Vietnam_Pro } from 'next/font/google';

// next/font tự host font — không request tới Google Fonts
const inter = Inter({
  subsets: ['latin'],
  display: 'swap',   // Tránh FOIT (Flash of Invisible Text)
  variable: '--font-inter',
});

const beVietnam = Be_Vietnam_Pro({
  weight: ['400', '500', '700'],
  subsets: ['vietnamese'],
  display: 'swap',
  variable: '--font-be-vietnam',
});

export default function RootLayout({ children }) {
  return (
    <html lang="vi" className={`${inter.variable} ${beVietnam.variable}`}>
      <body>{children}</body>
    </html>
  );
}
```

## 4. Server Components — Giảm JavaScript bundle

```tsx
// ✅ Server Component (default) — Không gửi JS xuống client
// app/products/page.tsx
async function ProductsPage() {
  // Fetch trực tiếp — không cần useEffect hay loading state
  const products = await fetch('https://api.shop.com/products', {
    next: { revalidate: 60 }, // ISR: cache 60 giây
  }).then(r => r.json());

  return (
    <div>
      {products.map(p => <ProductCard key={p.id} product={p} />)}
    </div>
  );
}

// ❌ Tránh 'use client' khi không cần interactivity
// Chỉ dùng 'use client' cho: useState, useEffect, event handlers, browser APIs
```

## 5. Code Splitting & Dynamic Import

```tsx
import dynamic from 'next/dynamic';

// Lazy load heavy component — không load khi page init
const RichTextEditor = dynamic(() => import('@/components/RichTextEditor'), {
  loading: () => <div className="animate-pulse h-64 bg-gray-100 rounded" />,
  ssr: false,  // Chỉ render ở client (editor dùng browser API)
});

const HeavyChart = dynamic(() => import('@/components/HeavyChart'), {
  loading: () => <p>Đang tải biểu đồ...</p>,
});

// Chỉ load khi user cần
export function ProductPage() {
  const [showReviews, setShowReviews] = useState(false);
  const ReviewsSection = dynamic(() => import('./ReviewsSection'));

  return (
    <div>
      <ProductInfo />
      <button onClick={() => setShowReviews(true)}>Xem đánh giá</button>
      {showReviews && <ReviewsSection />}
    </div>
  );
}
```

## 6. Caching Strategy

```typescript
// app/products/[id]/page.tsx
export async function generateStaticParams() {
  // Pre-generate top 100 products (SSG)
  const products = await fetch('/api/products/top-100').then(r => r.json());
  return products.map(p => ({ id: p.id }));
}

async function ProductDetailPage({ params }) {
  const product = await fetch(`/api/products/${params.id}`, {
    next: {
      revalidate: 300,  // ISR: revalidate sau 5 phút
      tags: [`product-${params.id}`],  // Tag-based revalidation
    },
  }).then(r => r.json());

  return <ProductDetail product={product} />;
}

// Revalidate theo tag (khi admin cập nhật product)
// app/api/revalidate/route.ts
import { revalidateTag } from 'next/cache';

export async function POST(req: Request) {
  const { productId } = await req.json();
  revalidateTag(`product-${productId}`);
  return Response.json({ revalidated: true });
}
```

## 7. Bundle Analyzer

```bash
npm install @next/bundle-analyzer
```

```javascript
// next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});

module.exports = withBundleAnalyzer({
  // ...config
});
```

```bash
ANALYZE=true npm run build
# Mở http://localhost:8888 — xem bundle size từng package
```

## 8. Prefetching & Navigation

```tsx
import Link from 'next/link';

// ✅ Link tự prefetch khi xuất hiện trong viewport
<Link href="/products/123" prefetch={true}>
  Xem chi tiết
</Link>

// Programmatic prefetch
import { useRouter } from 'next/navigation';
const router = useRouter();

// Prefetch khi hover
<div onMouseEnter={() => router.prefetch('/products/123')}>
  Hover to prefetch
</div>
```

## 9. Minimize CLS (Cumulative Layout Shift)

```tsx
// ❌ Gây CLS — size chưa biết trước
<img src={url} />

// ✅ Không gây CLS — đặt width/height hoặc aspect-ratio
<Image src={url} width={800} height={600} alt="..." />

// ❌ Font gây CLS
<link href="https://fonts.googleapis.com/..." rel="stylesheet" />

// ✅ next/font — không gây CLS
import { Inter } from 'next/font/google';
```

## 10. Kết luận

| Kỹ thuật | Cải thiện chỉ số |
|---------|-----------------|
| `next/image` | LCP, CLS |
| `next/font` | CLS, FID |
| Server Components | TTI, Bundle size |
| Dynamic import | TTI, Bundle size |
| ISR/SG | LCP, TTFB |
| Prefetch | FID, Navigation |

- Đo trước (Lighthouse, WebVitals), tối ưu sau — không đoán
- Server Components là công cụ mạnh nhất để giảm JS bundle
- `next/image` và `next/font` giải quyết 80% vấn đề CLS
- ISR kết hợp tag-based revalidation = UX nhanh + data fresh
