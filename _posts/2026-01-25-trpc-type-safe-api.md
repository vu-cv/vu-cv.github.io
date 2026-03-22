---
layout: article
title: tRPC – Type-safe API không cần schema cho NextJS full-stack
tags: [trpc, typescript, nextjs, type-safe, api, fullstack]
---
tRPC cho phép gọi server function từ client với **full type safety** — không cần viết schema, không cần code generation. Nếu bạn làm full-stack TypeScript với NextJS, tRPC thay thế REST hoàn toàn trong internal API.

## 1. tRPC hoạt động như thế nào?

```
Server định nghĩa procedure:
  getUser(id: string) → Promise<User>

Client gọi như gọi function thật:
  const user = await trpc.getUser.query('123')
  // TypeScript biết user có type User!
```

Không cần: Swagger, axios types, API docs — IDE tự hiểu.

## 2. Cài đặt (NextJS App Router)

```bash
npm install @trpc/server @trpc/client @trpc/react-query @trpc/next
npm install @tanstack/react-query zod
```

## 3. Server — Định nghĩa router

```typescript
// src/server/trpc.ts
import { initTRPC, TRPCError } from '@trpc/server';
import { getServerSession } from 'next-auth';
import { authOptions } from '@/lib/auth';
import superjson from 'superjson';
import { ZodError } from 'zod';

// Context — truyền vào mọi procedure
export async function createContext() {
  const session = await getServerSession(authOptions);
  return { session };
}
export type Context = Awaited<ReturnType<typeof createContext>>;

const t = initTRPC.context<Context>().create({
  transformer: superjson,  // Hỗ trợ Date, Map, Set trong JSON
  errorFormatter({ shape, error }) {
    return {
      ...shape,
      data: {
        ...shape.data,
        zodError: error.cause instanceof ZodError ? error.cause.flatten() : null,
      },
    };
  },
});

export const router = t.router;
export const publicProcedure = t.procedure;

// Middleware kiểm tra auth
const isAuthed = t.middleware(({ ctx, next }) => {
  if (!ctx.session?.user) {
    throw new TRPCError({ code: 'UNAUTHORIZED' });
  }
  return next({ ctx: { ...ctx, user: ctx.session.user } });
});

export const protectedProcedure = t.procedure.use(isAuthed);
```

```typescript
// src/server/routers/users.ts
import { z } from 'zod';
import { router, publicProcedure, protectedProcedure } from '../trpc';

export const usersRouter = router({
  // Query — đọc data
  getAll: publicProcedure
    .input(z.object({
      page: z.number().default(1),
      limit: z.number().default(10),
      search: z.string().optional(),
    }))
    .query(async ({ input }) => {
      const { page, limit, search } = input;
      const users = await db.user.findMany({
        where: search ? { name: { contains: search } } : undefined,
        skip: (page - 1) * limit,
        take: limit,
      });
      const total = await db.user.count();
      return { users, total, page, limit };
    }),

  getById: publicProcedure
    .input(z.string().uuid())
    .query(async ({ input: id }) => {
      const user = await db.user.findUnique({ where: { id } });
      if (!user) throw new TRPCError({ code: 'NOT_FOUND', message: 'User không tồn tại' });
      return user;
    }),

  // Mutation — thay đổi data
  create: publicProcedure
    .input(z.object({
      name: z.string().min(2),
      email: z.string().email(),
      password: z.string().min(6),
    }))
    .mutation(async ({ input }) => {
      const hashedPw = await bcrypt.hash(input.password, 10);
      return db.user.create({
        data: { ...input, password: hashedPw },
      });
    }),

  update: protectedProcedure
    .input(z.object({
      id: z.string().uuid(),
      name: z.string().min(2).optional(),
      avatar: z.string().url().optional(),
    }))
    .mutation(async ({ input, ctx }) => {
      if (input.id !== ctx.user.id) {
        throw new TRPCError({ code: 'FORBIDDEN' });
      }
      const { id, ...data } = input;
      return db.user.update({ where: { id }, data });
    }),

  delete: protectedProcedure
    .input(z.string().uuid())
    .mutation(async ({ input: id, ctx }) => {
      await db.user.delete({ where: { id } });
      return { success: true };
    }),
});
```

```typescript
// src/server/routers/posts.ts
export const postsRouter = router({
  getAll: publicProcedure
    .input(z.object({ authorId: z.string().optional() }))
    .query(async ({ input }) => {
      return db.post.findMany({
        where: input.authorId ? { authorId: input.authorId } : undefined,
        include: { author: { select: { id: true, name: true, avatar: true } } },
        orderBy: { createdAt: 'desc' },
      });
    }),

  create: protectedProcedure
    .input(z.object({
      title: z.string().min(1).max(200),
      content: z.string().min(10),
    }))
    .mutation(async ({ input, ctx }) => {
      return db.post.create({
        data: { ...input, authorId: ctx.user.id },
      });
    }),
});

// src/server/root.ts
import { router } from './trpc';
import { usersRouter } from './routers/users';
import { postsRouter } from './routers/posts';

export const appRouter = router({
  users: usersRouter,
  posts: postsRouter,
});

export type AppRouter = typeof appRouter;
```

## 4. NextJS API Handler

```typescript
// app/api/trpc/[trpc]/route.ts
import { fetchRequestHandler } from '@trpc/server/adapters/fetch';
import { appRouter } from '@/server/root';
import { createContext } from '@/server/trpc';

const handler = (req: Request) =>
  fetchRequestHandler({
    endpoint: '/api/trpc',
    req,
    router: appRouter,
    createContext,
  });

export { handler as GET, handler as POST };
```

## 5. Client Setup

```typescript
// src/trpc/client.ts
import { createTRPCReact } from '@trpc/react-query';
import type { AppRouter } from '@/server/root';

export const trpc = createTRPCReact<AppRouter>();

// src/providers.tsx
'use client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { httpBatchLink } from '@trpc/client';
import superjson from 'superjson';
import { useState } from 'react';
import { trpc } from '@/trpc/client';

export function TRPCProvider({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => new QueryClient());
  const [trpcClient] = useState(() =>
    trpc.createClient({
      links: [
        httpBatchLink({
          url: '/api/trpc',
          transformer: superjson,
        }),
      ],
    })
  );

  return (
    <trpc.Provider client={trpcClient} queryClient={queryClient}>
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    </trpc.Provider>
  );
}
```

## 6. Dùng trong Client Components

```tsx
'use client';
import { trpc } from '@/trpc/client';

export function UsersList() {
  // Giống React Query — có loading, error, data
  const { data, isLoading, error } = trpc.users.getAll.useQuery({
    page: 1,
    limit: 10,
    search: 'nguyen',
  });

  if (isLoading) return <p>Loading...</p>;
  if (error) return <p>Error: {error.message}</p>;

  return (
    <ul>
      {data?.users.map(user => (
        <li key={user.id}>{user.name} — {user.email}</li>
      ))}
    </ul>
  );
}

export function CreateUserForm() {
  const createUser = trpc.users.create.useMutation({
    onSuccess: () => {
      // Invalidate cache
      trpc.useUtils().users.getAll.invalidate();
    },
  });

  async function handleSubmit(data: FormData) {
    await createUser.mutateAsync({
      name: data.get('name') as string,
      email: data.get('email') as string,
      password: data.get('password') as string,
    });
  }

  return (
    <form action={handleSubmit}>
      <input name="name" placeholder="Tên" />
      <input name="email" type="email" placeholder="Email" />
      <input name="password" type="password" placeholder="Mật khẩu" />
      <button type="submit" disabled={createUser.isPending}>
        {createUser.isPending ? 'Đang tạo...' : 'Tạo user'}
      </button>
    </form>
  );
}
```

## 7. Server-side Caller

```typescript
// app/users/page.tsx — Server Component
import { createCaller } from '@/server/root';
import { createContext } from '@/server/trpc';

export default async function UsersPage() {
  // Gọi trực tiếp không qua HTTP — nhanh hơn
  const caller = createCaller(await createContext());
  const { users, total } = await caller.users.getAll({ page: 1, limit: 20 });

  return (
    <div>
      <h1>Users ({total})</h1>
      {users.map(u => <div key={u.id}>{u.name}</div>)}
    </div>
  );
}
```

## 8. Kết luận

- **End-to-end type safety**: Đổi tên field ở server → IDE báo lỗi ngay ở client
- **Zod validation**: Input validation tự động, error message rõ ràng
- **Batching**: Nhiều queries gộp thành 1 HTTP request
- **Server Caller**: Gọi procedure từ Server Component không qua network
- **Giới hạn**: Chỉ dùng được trong TypeScript full-stack — không phù hợp public API cho bên ngoài

tRPC là lựa chọn tốt nhất cho NextJS full-stack — nhanh hơn REST, type-safe hơn GraphQL, ít boilerplate nhất.
