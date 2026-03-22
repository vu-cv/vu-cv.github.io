---
layout: article
title: NextJS – Server Actions & API Routes
tags: [nextjs, server-actions, api, react]
---
Next.js 14 cung cấp hai cách để xử lý logic phía server: **API Routes** (truyền thống) và **Server Actions** (mới trong App Router). Bài này phân tích cả hai và hướng dẫn khi nào dùng cái nào.

## 1. API Routes

API Routes là các file trong thư mục `app/api/` trả về HTTP response. Chúng hoạt động như một REST endpoint thông thường.

### Tạo API Route cơ bản

```typescript
// app/api/hello/route.ts
import { NextResponse } from 'next/server';

export async function GET() {
  return NextResponse.json({ message: 'Hello World' });
}

export async function POST(request: Request) {
  const body = await request.json();
  return NextResponse.json({ received: body }, { status: 201 });
}
```

Truy cập: `GET http://localhost:3000/api/hello`

### API Route với params động

```typescript
// app/api/users/[id]/route.ts
import { NextResponse } from 'next/server';

export async function GET(
  request: Request,
  { params }: { params: { id: string } },
) {
  const { id } = params;
  // Fetch từ database...
  return NextResponse.json({ id, name: 'Nguyen Van A' });
}

export async function PUT(
  request: Request,
  { params }: { params: { id: string } },
) {
  const body = await request.json();
  // Update database...
  return NextResponse.json({ id: params.id, ...body });
}

export async function DELETE(
  request: Request,
  { params }: { params: { id: string } },
) {
  // Delete from database...
  return new NextResponse(null, { status: 204 });
}
```

### Đọc Query Params và Headers

```typescript
import { NextRequest, NextResponse } from 'next/server';

export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const page = searchParams.get('page') ?? '1';
  const limit = searchParams.get('limit') ?? '10';

  const authHeader = request.headers.get('authorization');

  return NextResponse.json({ page, limit, auth: authHeader });
}
```

## 2. Server Actions

Server Actions là các async function chạy trên server nhưng được gọi trực tiếp từ component. Không cần tạo API route riêng.

### Khai báo Server Action

```typescript
// app/actions/user.actions.ts
'use server';

export async function createUser(formData: FormData) {
  const name = formData.get('name') as string;
  const email = formData.get('email') as string;

  // Lưu vào database
  // await db.users.create({ name, email });

  console.log('Created:', { name, email });
  return { success: true, name };
}
```

> Directive `'use server'` ở đầu file hoặc đầu function để đánh dấu Server Action.

### Dùng với HTML Form

```tsx
// app/create-user/page.tsx
import { createUser } from '../actions/user.actions';

export default function CreateUserPage() {
  return (
    <form action={createUser}>
      <input name="name" placeholder="Tên" required />
      <input name="email" type="email" placeholder="Email" required />
      <button type="submit">Tạo User</button>
    </form>
  );
}
```

Khi submit form, `createUser` chạy trực tiếp trên server — **không cần fetch API**.

### Dùng với useActionState (React 19 / Next.js 14)

```tsx
'use client';

import { useActionState } from 'react';
import { createUser } from '../actions/user.actions';

export function CreateUserForm() {
  const [state, action, isPending] = useActionState(createUser, null);

  return (
    <form action={action}>
      <input name="name" placeholder="Tên" />
      <input name="email" placeholder="Email" />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Đang tạo...' : 'Tạo User'}
      </button>
      {state?.success && <p>Tạo thành công: {state.name}</p>}
    </form>
  );
}
```

### Gọi Server Action từ Button

```tsx
'use client';

import { deleteUser } from '../actions/user.actions';

export function DeleteButton({ userId }: { userId: string }) {
  return (
    <form action={deleteUser.bind(null, userId)}>
      <button type="submit">Xóa</button>
    </form>
  );
}
```

```typescript
// actions/user.actions.ts
'use server';

import { revalidatePath } from 'next/cache';

export async function deleteUser(userId: string) {
  // await db.users.delete(userId);
  revalidatePath('/users'); // Refresh cache trang /users
}
```

## 3. Middleware

Middleware chạy trước mọi request, dùng để authentication, redirect, log:

```typescript
// middleware.ts (đặt ở root)
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const token = request.cookies.get('token')?.value;

  // Redirect nếu chưa đăng nhập
  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  return NextResponse.next();
}

// Chỉ áp dụng cho các path này
export const config = {
  matcher: ['/dashboard/:path*', '/profile/:path*'],
};
```

## 4. Khi nào dùng gì?

| Tình huống | Nên dùng |
|-----------|----------|
| Form submit đơn giản | Server Action |
| Mutation dữ liệu từ component | Server Action |
| Public REST API cho bên thứ 3 | API Route |
| Webhook từ bên ngoài (Stripe, GitHub) | API Route |
| Upload file | API Route hoặc Server Action |
| Fetch data khi render | Server Component (fetch trực tiếp) |

## 5. Kết luận

- **API Routes** (`app/api/`) phù hợp khi cần expose endpoint ra ngoài hoặc nhận webhook
- **Server Actions** phù hợp cho form submission và mutation dữ liệu nội bộ — code gọn hơn, không cần `fetch` phía client
- **Middleware** dùng cho authentication và redirect toàn cục

Bài tiếp theo: **NextJS – Deploy lên Vercel & AWS Amplify**.
