---
title: "TypeScriptで型安全なAPIクライアントを実装する実践ガイド"
emoji: "🔒"
type: "tech"
topics:
  - "typescript"
  - "api"
  - "型安全性"
published: false
---

## はじめに

Web 開発において、API との通信は避けて通れない要素です。しかし、型の不一致によるランタイムエラーに悩まされた経験がある方も多いのではないでしょうか。

この記事では、TypeScript を活用して型安全な API クライアントを実装する方法を紹介します。実際のプロジェクトで使える実践的なパターンを、コード例とともに解説していきます。

## 前提知識

TypeScript の基本的な文法と、ジェネリクスの概念を理解している方を対象としています。

## 型安全なAPIクライアントが必要な理由

従来の JavaScript や any 型を多用した TypeScript では、以下のような問題が発生します。

- API のレスポンス構造が変更されても、コンパイル時に検出できない
- 存在しないプロパティへのアクセスがランタイムまで分からない
- リファクタリング時の影響範囲が把握しづらい

これらの問題を解決するには、型システムを最大限活用することが重要です。

## 基本的な実装パターン

まず、シンプルな API クライアントから始めましょう。

### レスポンス型の定義

```typescript
// ユーザー情報のレスポンス型
interface User {
  id: number;
  name: string;
  email: string;
  createdAt: string;
}

// API レスポンスの共通型
interface ApiResponse<T> {
  data: T;
  status: number;
  message?: string;
}
```

### 型安全なfetch関数

```typescript
async function fetchApi<T>(
  url: string,
  options?: RequestInit
): Promise<ApiResponse<T>> {
  const response = await fetch(url, options);

  if (!response.ok) {
    throw new Error(`HTTP error! status: ${response.status}`);
  }

  const data = await response.json();
  return data;
}
```

これを使うことで、型推論が効きます。

```typescript
// 使用例
const result = await fetchApi<User>('/api/users/1');
console.log(result.data.name); // 型安全にアクセス可能
```

## より高度な実装：APIクライアントクラス

実際のプロジェクトでは、クラスベースの実装が便利です。

```typescript
class ApiClient {
  private baseUrl: string;

  constructor(baseUrl: string) {
    this.baseUrl = baseUrl;
  }

  async get<T>(endpoint: string): Promise<T> {
    const response = await fetch(`${this.baseUrl}${endpoint}`);
    return response.json();
  }

  async post<T, U>(endpoint: string, body: U): Promise<T> {
    const response = await fetch(`${this.baseUrl}${endpoint}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
    });
    return response.json();
  }
}
```

### 使用例

```typescript
const api = new ApiClient('https://api.example.com');

// GETリクエスト
const user = await api.get<User>('/users/1');

// POSTリクエスト
interface CreateUserRequest {
  name: string;
  email: string;
}

const newUser = await api.post<User, CreateUserRequest>('/users', {
  name: 'John Doe',
  email: 'john@example.com',
});
```

## エラーハンドリングの型安全化

エラーハンドリングも型安全にできます。

```typescript
interface ApiError {
  code: string;
  message: string;
  details?: Record<string, string[]>;
}

class ApiClient {
  async get<T>(endpoint: string): Promise<T> {
    try {
      const response = await fetch(`${this.baseUrl}${endpoint}`);

      if (!response.ok) {
        const error: ApiError = await response.json();
        throw new ApiClientError(error);
      }

      return response.json();
    } catch (error) {
      if (error instanceof ApiClientError) {
        throw error;
      }
      throw new Error('Network error');
    }
  }
}

class ApiClientError extends Error {
  constructor(public error: ApiError) {
    super(error.message);
    this.name = 'ApiClientError';
  }
}
```

## Zodによるランタイムバリデーション

TypeScript の型チェックはコンパイル時のみ有効です。ランタイムでの検証には Zod が有効です。

```typescript
import { z } from 'zod';

// スキーマ定義
const UserSchema = z.object({
  id: z.number(),
  name: z.string(),
  email: z.string().email(),
  createdAt: z.string(),
});

type User = z.infer<typeof UserSchema>;

async function fetchUser(id: number): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  const data = await response.json();

  // ランタイムバリデーション
  return UserSchema.parse(data);
}
```

これにより、予期しないデータ構造からアプリケーションを守ることができます。

## まとめ

型安全な API クライアントを実装することで、以下のメリットが得られます。

- コンパイル時のエラー検出
- IDE の補完が効いて開発効率が向上する
- リファクタリングが安全になる
- バグの早期発見

実際のプロジェクトでは、これらのパターンを組み合わせて、プロジェクトの要件に合わせてカスタマイズしてください。型安全性を意識することで、より堅牢なアプリケーションを構築できます。
