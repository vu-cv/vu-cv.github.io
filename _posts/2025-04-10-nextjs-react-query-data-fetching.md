---
layout: article
title: NextJS – Data Fetching với TanStack Query (React Query)
tags: [nextjs, react-query, tanstack, data-fetching, swr]
---
TanStack Query (React Query) là thư viện quản lý server state mạnh mẽ nhất cho React. Kết hợp với Next.js App Router, nó xử lý caching, refetching, loading states và error handling một cách tự động.

## 1. Cài đặt

```bash
npm install --save @tanstack/react-query @tanstack/react-query-devtools
```

## 2. Cấu hình Provider

```tsx
// app/providers.tsx
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import { useState } from 'react';

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000,     // Data được coi là fresh trong 1 phút
        retry: 2,                  // Retry 2 lần khi lỗi
        refetchOnWindowFocus: false,
      },
    },
  }));

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

```tsx
// app/layout.tsx
import { Providers } from './providers';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

## 3. useQuery — Đọc dữ liệu

```tsx
'use client';

import { useQuery } from '@tanstack/react-query';

interface Product {
  id: string;
  name: string;
  price: number;
}

async function fetchProducts(page: number, category?: string): Promise<{
  items: Product[];
  total: number;
}> {
  const params = new URLSearchParams({ page: String(page) });
  if (category) params.set('category', category);

  const res = await fetch(`/api/products?${params}`);
  if (!res.ok) throw new Error('Lỗi khi tải sản phẩm');
  return res.json();
}

export function ProductList() {
  const [page, setPage] = useState(1);
  const [category, setCategory] = useState<string>();

  const { data, isLoading, isError, error, isFetching } = useQuery({
    queryKey: ['products', page, category], // Cache key — thay đổi key = fetch lại
    queryFn: () => fetchProducts(page, category),
    placeholderData: (prev) => prev, // Giữ data cũ khi fetch trang mới
  });

  if (isLoading) return <p>Đang tải...</p>;
  if (isError) return <p>Lỗi: {error.message}</p>;

  return (
    <div>
      {isFetching && <p className="text-sm text-gray-400">Đang cập nhật...</p>}

      <div className="grid grid-cols-3 gap-4">
        {data?.items.map(p => (
          <div key={p.id} className="border rounded p-4">
            <h3>{p.name}</h3>
            <p>{p.price.toLocaleString('vi-VN')}đ</p>
          </div>
        ))}
      </div>

      <div className="flex gap-2 mt-4">
        <button onClick={() => setPage(p => p - 1)} disabled={page === 1}>Trước</button>
        <span>Trang {page}</span>
        <button onClick={() => setPage(p => p + 1)}>Sau</button>
      </div>
    </div>
  );
}
```

## 4. useMutation — Tạo / Cập nhật / Xóa

```tsx
'use client';

import { useMutation, useQueryClient } from '@tanstack/react-query';

async function createProduct(data: Partial<Product>) {
  const res = await fetch('/api/products', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  if (!res.ok) throw new Error('Tạo sản phẩm thất bại');
  return res.json();
}

async function deleteProduct(id: string) {
  const res = await fetch(`/api/products/${id}`, { method: 'DELETE' });
  if (!res.ok) throw new Error('Xóa thất bại');
}

export function CreateProductForm() {
  const queryClient = useQueryClient();
  const [name, setName] = useState('');

  const { mutate, isPending, isError, error } = useMutation({
    mutationFn: createProduct,
    onSuccess: () => {
      // Invalidate cache → tự động refetch danh sách
      queryClient.invalidateQueries({ queryKey: ['products'] });
      setName('');
      alert('Tạo thành công!');
    },
    onError: (err) => {
      console.error('Lỗi:', err);
    },
  });

  return (
    <form onSubmit={e => { e.preventDefault(); mutate({ name, price: 100000 }); }}>
      <input value={name} onChange={e => setName(e.target.value)} placeholder="Tên sản phẩm" />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Đang tạo...' : 'Tạo sản phẩm'}
      </button>
      {isError && <p className="text-red-500">{error.message}</p>}
    </form>
  );
}
```

## 5. Optimistic Update — UI phản hồi ngay không cần đợi API

```tsx
const { mutate } = useMutation({
  mutationFn: (id: string) => deleteProduct(id),
  onMutate: async (deletedId) => {
    // Hủy refetch đang chạy để tránh overwrite
    await queryClient.cancelQueries({ queryKey: ['products'] });

    // Snapshot state cũ để rollback nếu lỗi
    const previous = queryClient.getQueryData(['products']);

    // Cập nhật UI ngay lập tức
    queryClient.setQueryData(['products'], (old: any) => ({
      ...old,
      items: old.items.filter((p: Product) => p.id !== deletedId),
    }));

    return { previous };
  },
  onError: (err, id, context) => {
    // Rollback nếu API thất bại
    queryClient.setQueryData(['products'], context?.previous);
  },
  onSettled: () => {
    // Luôn refetch để đồng bộ với server
    queryClient.invalidateQueries({ queryKey: ['products'] });
  },
});
```

## 6. Prefetch — Tải trước dữ liệu

```tsx
// Prefetch trong Server Component để hydrate client
import { HydrationBoundary, QueryClient, dehydrate } from '@tanstack/react-query';

export default async function ProductsPage() {
  const queryClient = new QueryClient();

  // Prefetch trên server
  await queryClient.prefetchQuery({
    queryKey: ['products', 1],
    queryFn: () => fetchProducts(1),
  });

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <ProductList /> {/* Client component nhận ngay data từ server */}
    </HydrationBoundary>
  );
}
```

## 7. useInfiniteQuery — Infinite Scroll

```tsx
const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useInfiniteQuery({
  queryKey: ['products', 'infinite'],
  queryFn: ({ pageParam = 1 }) => fetchProducts(pageParam),
  getNextPageParam: (lastPage, pages) =>
    lastPage.items.length === 10 ? pages.length + 1 : undefined,
  initialPageParam: 1,
});

const allProducts = data?.pages.flatMap(p => p.items) ?? [];
```

## 8. Kết luận

TanStack Query giải quyết hoàn toàn bài toán server state:

- `useQuery` — fetch, cache, auto-refetch tự động
- `useMutation` — create/update/delete với invalidate cache
- **Optimistic Update** — UI responsive ngay, rollback khi lỗi
- **Prefetch** — kết hợp Server Component để zero loading state
- `useInfiniteQuery` — infinite scroll đơn giản

Không cần `useEffect` + `useState` để fetch data nữa — code sạch hơn, ít bug hơn.
