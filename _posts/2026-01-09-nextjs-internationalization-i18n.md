---
layout: article
title: NextJS – Internationalization (i18n) đa ngôn ngữ
tags: [nextjs, i18n, internationalization, localization, frontend]
---
Internationalization (i18n) giúp ứng dụng hỗ trợ nhiều ngôn ngữ. NextJS App Router có hỗ trợ i18n built-in thông qua routing. Bài này hướng dẫn implement i18n với **next-intl** — thư viện phổ biến nhất.

## 1. Cài đặt

```bash
npm install next-intl
```

## 2. Cấu trúc thư mục

```
src/
  app/
    [locale]/           ← Dynamic segment cho locale
      layout.tsx
      page.tsx
      products/
        page.tsx
  messages/
    vi.json             ← Tiếng Việt
    en.json             ← Tiếng Anh
  middleware.ts
  i18n.ts
```

## 3. Cấu hình

```typescript
// i18n.ts
import { notFound } from 'next/navigation';
import { getRequestConfig } from 'next-intl/server';

export const locales = ['vi', 'en'] as const;
export type Locale = (typeof locales)[number];
export const defaultLocale: Locale = 'vi';

export default getRequestConfig(async ({ locale }) => {
  if (!locales.includes(locale as Locale)) notFound();

  return {
    messages: (await import(`./messages/${locale}.json`)).default,
  };
});
```

```typescript
// middleware.ts
import createMiddleware from 'next-intl/middleware';
import { locales, defaultLocale } from './i18n';

export default createMiddleware({
  locales,
  defaultLocale,
  localePrefix: 'as-needed', // /vi/... cho vi, /en/... cho en, / cho default
});

export const config = {
  matcher: ['/((?!api|_next|.*\\..*).*)'],
};
```

## 4. Translation Files

```json
// messages/vi.json
{
  "common": {
    "search": "Tìm kiếm",
    "loading": "Đang tải...",
    "error": "Đã có lỗi xảy ra",
    "retry": "Thử lại",
    "save": "Lưu",
    "cancel": "Hủy",
    "delete": "Xóa"
  },
  "nav": {
    "home": "Trang chủ",
    "products": "Sản phẩm",
    "orders": "Đơn hàng",
    "profile": "Hồ sơ"
  },
  "products": {
    "title": "Danh sách sản phẩm",
    "count": "{count} sản phẩm",
    "addToCart": "Thêm vào giỏ",
    "outOfStock": "Hết hàng",
    "price": "{price}₫"
  },
  "auth": {
    "login": "Đăng nhập",
    "register": "Đăng ký",
    "email": "Email",
    "password": "Mật khẩu",
    "forgotPassword": "Quên mật khẩu?",
    "loginSuccess": "Đăng nhập thành công!",
    "error": {
      "invalidCredentials": "Email hoặc mật khẩu không đúng",
      "emailRequired": "Email là bắt buộc"
    }
  }
}
```

```json
// messages/en.json
{
  "common": {
    "search": "Search",
    "loading": "Loading...",
    "error": "Something went wrong",
    "retry": "Retry",
    "save": "Save",
    "cancel": "Cancel",
    "delete": "Delete"
  },
  "nav": {
    "home": "Home",
    "products": "Products",
    "orders": "Orders",
    "profile": "Profile"
  },
  "products": {
    "title": "Products",
    "count": "{count} products",
    "addToCart": "Add to Cart",
    "outOfStock": "Out of Stock",
    "price": "${price}"
  },
  "auth": {
    "login": "Login",
    "register": "Register",
    "email": "Email",
    "password": "Password",
    "forgotPassword": "Forgot password?",
    "loginSuccess": "Logged in successfully!",
    "error": {
      "invalidCredentials": "Invalid email or password",
      "emailRequired": "Email is required"
    }
  }
}
```

## 5. Layout với locale

```typescript
// app/[locale]/layout.tsx
import { NextIntlClientProvider } from 'next-intl';
import { getMessages, getLocale } from 'next-intl/server';
import { notFound } from 'next/navigation';
import { locales } from '@/i18n';

export function generateStaticParams() {
  return locales.map(locale => ({ locale }));
}

export default async function LocaleLayout({
  children,
  params: { locale },
}: {
  children: React.ReactNode;
  params: { locale: string };
}) {
  if (!locales.includes(locale as any)) notFound();

  const messages = await getMessages();

  return (
    <html lang={locale}>
      <body>
        <NextIntlClientProvider messages={messages}>
          {children}
        </NextIntlClientProvider>
      </body>
    </html>
  );
}
```

## 6. Dùng trong Server Component

```tsx
// app/[locale]/products/page.tsx
import { useTranslations } from 'next-intl';
import { getTranslations } from 'next-intl/server';

// Server Component
export async function generateMetadata({ params: { locale } }) {
  const t = await getTranslations({ locale, namespace: 'products' });
  return { title: t('title') };
}

export default async function ProductsPage() {
  const t = await getTranslations('products');
  const products = await fetchProducts();

  return (
    <div>
      <h1>{t('title')}</h1>
      <p>{t('count', { count: products.length })}</p>
      <div className="grid grid-cols-3 gap-4">
        {products.map(p => (
          <div key={p.id}>
            <h3>{p.name}</h3>
            <p>{t('price', { price: p.price.toLocaleString() })}</p>
          </div>
        ))}
      </div>
    </div>
  );
}
```

## 7. Dùng trong Client Component

```tsx
'use client';
import { useTranslations, useLocale } from 'next-intl';

export function AddToCartButton({ productId, inStock }) {
  const t = useTranslations('products');
  const locale = useLocale();

  return (
    <button disabled={!inStock}>
      {inStock ? t('addToCart') : t('outOfStock')}
    </button>
  );
}
```

## 8. Language Switcher

```tsx
'use client';
import { useRouter, usePathname } from 'next/navigation';
import { useLocale } from 'next-intl';

export function LanguageSwitcher() {
  const locale = useLocale();
  const router = useRouter();
  const pathname = usePathname();

  const switchLocale = (newLocale: string) => {
    // Thay đổi locale trong URL
    const newPath = pathname.replace(`/${locale}`, `/${newLocale}`);
    router.push(newPath);
  };

  return (
    <div className="flex gap-2">
      <button
        onClick={() => switchLocale('vi')}
        className={locale === 'vi' ? 'font-bold' : 'text-gray-500'}
      >
        🇻🇳 Tiếng Việt
      </button>
      <button
        onClick={() => switchLocale('en')}
        className={locale === 'en' ? 'font-bold' : 'text-gray-500'}
      >
        🇬🇧 English
      </button>
    </div>
  );
}
```

## 9. Format số, ngày tháng, tiền tệ

```tsx
import { useFormatter, useNow } from 'next-intl';

export function ProductPrice({ price, date }) {
  const format = useFormatter();

  return (
    <div>
      {/* Format tiền tệ theo locale */}
      <p>{format.number(price, { style: 'currency', currency: 'VND' })}</p>
      {/* → 1.500.000 ₫ (vi) | ₫1,500,000 (en) */}

      {/* Format ngày */}
      <p>{format.dateTime(date, { dateStyle: 'medium' })}</p>
      {/* → 25 thg 12, 2025 (vi) | Dec 25, 2025 (en) */}

      {/* Relative time */}
      <p>{format.relativeTime(date)}</p>
      {/* → 3 ngày trước (vi) | 3 days ago (en) */}
    </div>
  );
}
```

## 10. Kết luận

- **`[locale]` segment**: Routing tự động theo ngôn ngữ (`/vi/products`, `/en/products`)
- **Middleware**: Tự redirect user đến đúng locale dựa trên browser language
- **Server/Client Component**: `getTranslations()` (server), `useTranslations()` (client)
- **Format**: Dùng `useFormatter()` cho số, ngày, tiền — không hard-code format
- **Namespace**: Chia messages thành namespace (auth, products, nav) để dễ quản lý

next-intl là lựa chọn tốt nhất cho App Router — TypeScript support tốt, không client bundle lớn.
