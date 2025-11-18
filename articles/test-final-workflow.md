---
title: "Next.js App Routerでの認証実装パターン"
emoji: "🔐"
type: "tech"
topics:
  - "nextjs"
  - "authentication"
  - "react"
published: false
---

## はじめに

Next.js 13 で導入された App Router は、従来の Pages Router とは異なるアーキテクチャを持っています。それに伴い、認証の実装方法も変更が必要になりました。

この記事では、App Router での認証実装パターンを、実際のコード例とともに解説します。セッション管理、ミドルウェアの活用、Server Components との組み合わせなど、実践的な内容を扱います。

## 前提知識

Next.js の基本的な概念（Server Components、Client Components、ルーティング）を理解していることを前提としています。

## App Router における認証の考え方

従来の Pages Router では、`getServerSideProps` や `_app.tsx` で認証状態を管理していましたが、App Router では以下の方法を組み合わせます。

- Middleware による認証チェック
- Server Components でのセッション取得
- Server Actions による認証処理

これらを適切に組み合わせることで、安全で効率的な認証を実装できます。

## 基本的な実装：NextAuth.js を使う

NextAuth.js は App Router に対応しており、比較的簡単に認証を実装できます。

### インストール

```bash
npm install next-auth@beta
```

App Router 対応の beta 版を使用します。

### 設定ファイルの作成

```typescript
// app/api/auth/[...nextauth]/route.ts
import NextAuth from "next-auth";
import GithubProvider from "next-auth/providers/github";

const handler = NextAuth({
  providers: [
    GithubProvider({
      clientId: process.env.GITHUB_ID!,
      clientSecret: process.env.GITHUB_SECRET!,
    }),
  ],
  callbacks: {
    async session({ session, token }) {
      if (session.user) {
        session.user.id = token.sub!;
      }
      return session;
    },
  },
});

export { handler as GET, handler as POST };
```

### セッション情報の取得

Server Components では、セッション情報を直接取得できます。

```typescript
// app/dashboard/page.tsx
import { getServerSession } from "next-auth";
import { redirect } from "next/navigation";

export default async function DashboardPage() {
  const session = await getServerSession();

  if (!session) {
    redirect("/login");
  }

  return (
    <div>
      <h1>ダッシュボード</h1>
      <p>ようこそ、{session.user?.name}さん</p>
    </div>
  );
}
```

これにより、サーバー側で認証チェックができます。

## Middleware による認証ガード

特定のパスへのアクセスを制限するには、Middleware を使用します。

```typescript
// middleware.ts
import { withAuth } from "next-auth/middleware";

export default withAuth({
  callbacks: {
    authorized({ req, token }) {
      // /admin配下は管理者のみアクセス可能
      if (req.nextUrl.pathname.startsWith("/admin")) {
        return token?.role === "admin";
      }
      // その他の保護されたルートはログイン必須
      return !!token;
    },
  },
});

export const config = {
  matcher: ["/dashboard/:path*", "/admin/:path*"],
};
```

## カスタム認証の実装

NextAuth.js を使わずに、独自の認証システムを実装できます。

### セッション管理

```typescript
// lib/session.ts
import { cookies } from "next/headers";
import { SignJWT, jwtVerify } from "jose";

const secretKey = process.env.SESSION_SECRET!;
const key = new TextEncoder().encode(secretKey);

export async function createSession(userId: string) {
  const token = await new SignJWT({ userId })
    .setProtectedHeader({ alg: "HS256" })
    .setIssuedAt()
    .setExpirationTime("24h")
    .sign(key);

  cookies().set("session", token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    maxAge: 60 * 60 * 24, // 24時間
    path: "/",
  });
}

export async function getSession() {
  const session = cookies().get("session")?.value;

  if (!session) {
    return null;
  }

  try {
    const { payload } = await jwtVerify(session, key);
    return payload;
  } catch (error) {
    return null;
  }
}
```

### ログイン処理

Server Actions を使ってログイン処理を実装します。

```typescript
// app/login/actions.ts
"use server";

import { createSession } from "@/lib/session";
import { redirect } from "next/navigation";

export async function login(formData: FormData) {
  const email = formData.get("email") as string;
  const password = formData.get("password") as string;

  // ユーザー認証（実際にはデータベースと照合）
  const user = await authenticateUser(email, password);

  if (!user) {
    return { error: "メールアドレスまたはパスワードが正しくありません" };
  }

  await createSession(user.id);
  redirect("/dashboard");
}
```

## クライアント側での認証状態の利用

Client Components で認証状態を使う場合は、Context を利用します。

```typescript
// app/providers/auth-provider.tsx
"use client";

import { createContext, useContext } from "react";
import { Session } from "next-auth";

const AuthContext = createContext<{ session: Session | null }>({
  session: null,
});

export function AuthProvider({
  children,
  session,
}: {
  children: React.ReactNode;
  session: Session | null;
}) {
  return (
    <AuthContext.Provider value={{ session }}>
      {children}
    </AuthContext.Provider>
  );
}

export const useAuth = () => useContext(AuthContext);
```

## まとめ

App Router での認証実装について、以下のポイントを解説しました。

- NextAuth.js を使った基本的な実装
- Middleware による認証ガード
- カスタム認証システムの実装
- Client Components での認証状態の利用

App Router の特性を活かした認証実装により、セキュアで高速なアプリケーションを構築できます。実際のプロジェクトでは、これらのパターンを組み合わせて、要件に合わせた認証システムを構築してください。
