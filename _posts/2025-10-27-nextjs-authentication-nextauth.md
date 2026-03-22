---
layout: article
title: NextJS – Authentication với NextAuth.js (Auth.js)
tags: [nextjs, nextauth, authentication, oauth, session]
---
NextAuth.js (nay là Auth.js) là thư viện authentication phổ biến nhất cho Next.js, hỗ trợ OAuth (Google, GitHub, Facebook...), Credentials, Email magic link, và nhiều hơn nữa — chỉ với vài dòng config.

## 1. Cài đặt

```bash
npm install --save next-auth@beta
```

Tạo auth secret:
```bash
npx auth secret
```

Thêm vào `.env`:
```
AUTH_SECRET=your-generated-secret
AUTH_GOOGLE_ID=your-google-client-id
AUTH_GOOGLE_SECRET=your-google-client-secret
NEXTAUTH_URL=http://localhost:3000
```

## 2. Cấu hình Auth

Tạo file `auth.ts` ở root project:

```typescript
// auth.ts
import NextAuth from 'next-auth';
import Google from 'next-auth/providers/google';
import GitHub from 'next-auth/providers/github';
import Credentials from 'next-auth/providers/credentials';

export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [
    Google,
    GitHub,
    Credentials({
      credentials: {
        email: { label: 'Email', type: 'email' },
        password: { label: 'Password', type: 'password' },
      },
      async authorize(credentials) {
        // Xác thực với backend API
        const res = await fetch(`${process.env.API_URL}/auth/login`, {
          method: 'POST',
          body: JSON.stringify(credentials),
          headers: { 'Content-Type': 'application/json' },
        });

        if (!res.ok) return null;

        const user = await res.json();
        return user; // { id, name, email, token }
      },
    }),
  ],

  callbacks: {
    // Thêm custom data vào token
    async jwt({ token, user, account }) {
      if (user) {
        token.accessToken = (user as any).token;
        token.id = user.id;
      }
      return token;
    },
    // Thêm custom data vào session
    async session({ session, token }) {
      session.user.id = token.id as string;
      (session as any).accessToken = token.accessToken;
      return session;
    },
  },

  pages: {
    signIn: '/login',   // Trang login tùy chỉnh
    error: '/error',    // Trang lỗi tùy chỉnh
  },
});
```

## 3. Tạo Route Handler

```typescript
// app/api/auth/[...nextauth]/route.ts
import { handlers } from '@/auth';
export const { GET, POST } = handlers;
```

## 4. Bảo vệ Route với Middleware

```typescript
// middleware.ts
import { auth } from '@/auth';

export default auth((req) => {
  const isLoggedIn = !!req.auth;
  const isAuthPage = req.nextUrl.pathname.startsWith('/login');

  if (!isLoggedIn && !isAuthPage) {
    return Response.redirect(new URL('/login', req.url));
  }
});

export const config = {
  matcher: ['/dashboard/:path*', '/profile/:path*'],
};
```

## 5. Trang Login tùy chỉnh

```tsx
// app/login/page.tsx
'use client';

import { signIn } from 'next-auth/react';
import { useState } from 'react';
import { useRouter } from 'next/navigation';

export default function LoginPage() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const router = useRouter();

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setError('');

    const result = await signIn('credentials', {
      email,
      password,
      redirect: false,
    });

    if (result?.error) {
      setError('Email hoặc mật khẩu không đúng');
    } else {
      router.push('/dashboard');
    }
  }

  return (
    <div className="flex min-h-screen items-center justify-center">
      <form onSubmit={handleSubmit} className="w-full max-w-sm space-y-4">
        <h1 className="text-2xl font-bold">Đăng nhập</h1>

        {error && <p className="text-red-500">{error}</p>}

        <input
          type="email"
          placeholder="Email"
          value={email}
          onChange={e => setEmail(e.target.value)}
          className="w-full border rounded p-2"
          required
        />
        <input
          type="password"
          placeholder="Mật khẩu"
          value={password}
          onChange={e => setPassword(e.target.value)}
          className="w-full border rounded p-2"
          required
        />
        <button type="submit" className="w-full bg-blue-600 text-white py-2 rounded">
          Đăng nhập
        </button>

        <div className="space-y-2">
          <button
            type="button"
            onClick={() => signIn('google', { callbackUrl: '/dashboard' })}
            className="w-full border py-2 rounded flex items-center justify-center gap-2"
          >
            Đăng nhập với Google
          </button>
          <button
            type="button"
            onClick={() => signIn('github', { callbackUrl: '/dashboard' })}
            className="w-full border py-2 rounded"
          >
            Đăng nhập với GitHub
          </button>
        </div>
      </form>
    </div>
  );
}
```

## 6. Lấy session trong Server Component

```tsx
// app/dashboard/page.tsx
import { auth } from '@/auth';
import { redirect } from 'next/navigation';

export default async function DashboardPage() {
  const session = await auth();

  if (!session) redirect('/login');

  return (
    <div>
      <h1>Xin chào, {session.user?.name}!</h1>
      <p>Email: {session.user?.email}</p>
      <img src={session.user?.image ?? ''} alt="Avatar" className="w-12 h-12 rounded-full" />
    </div>
  );
}
```

## 7. Lấy session trong Client Component

```tsx
'use client';

import { useSession, signOut } from 'next-auth/react';

export function Navbar() {
  const { data: session, status } = useSession();

  if (status === 'loading') return <p>Đang tải...</p>;

  return (
    <nav className="flex items-center justify-between p-4">
      <span>My App</span>
      {session ? (
        <div className="flex items-center gap-3">
          <span>{session.user?.name}</span>
          <button onClick={() => signOut({ callbackUrl: '/login' })}>
            Đăng xuất
          </button>
        </div>
      ) : (
        <a href="/login">Đăng nhập</a>
      )}
    </nav>
  );
}
```

Bọc app với `SessionProvider`:

```tsx
// app/layout.tsx
import { SessionProvider } from 'next-auth/react';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <SessionProvider>{children}</SessionProvider>
      </body>
    </html>
  );
}
```

## 8. Kết luận

NextAuth.js giúp thêm authentication vào Next.js cực nhanh:

- **OAuth** (Google, GitHub...): Chỉ cần thêm provider và client ID/secret
- **Credentials**: Kết nối với backend API tùy chỉnh
- **Middleware**: Bảo vệ route mà không cần code trong từng page
- `auth()` trong Server Component, `useSession()` trong Client Component

Đây là giải pháp authentication đầy đủ, production-ready chỉ với vài chục dòng config.
