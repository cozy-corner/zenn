---
title: "Reactアプリケーションのパフォーマンス最適化実践ガイド"
emoji: "⚡"
type: "tech"
topics:
  - "react"
  - "performance"
  - "frontend"
  - "optimization"
published: false
---

## React アプリケーションの高速化

React アプリケーションが遅くなる原因はいろいろある。これを解決するために、いくつかの方法を試した。

本記事では、実際のプロジェクトで遭遇したパフォーマンス問題とその解決方法を紹介する。

## 遭遇したパフォーマンス問題

開発中のダッシュボードアプリケーションで、ユーザーからの苦情が多かった。画面の切り替えが遅い、スクロールがカクつく、入力時の反応が悪いなど、様々な問題が報告された。

### 問題の調査

それを調査するために、Chrome DevTools の Performance タブを使用した。プロファイリングを実施した結果、以下の問題が明らかになった。

- コンポーネントの不要な再レンダリングが頻発している
- 重い計算処理が毎回実行されている
- 初期バンドルサイズが大きい
- リスト表示のパフォーマンスが悪い

## 実装した改善策

最適化を実施したところ、レンダリングが速くなった。具体的には、以下の手法を適用した。useMemo と useCallback を使ってコンポーネントの再レンダリングを抑制した。React.memo でコンポーネント全体をメモ化することで不要な再計算を防いだ。lazy loading を導入して初期表示を高速化した。react-window による仮想化でリスト表示を改善した。

### useMemo の活用

それは計算コストが高い処理をメモ化する。

```javascript
// 改善前：毎回ソート処理が実行される
function ProductList({ products }) {
  const sortedProducts = products.sort((a, b) => b.price - a.price);

  return (
    <ul>
      {sortedProducts.map(product => (
        <li key={product.id}>{product.name}</li>
      ))}
    </ul>
  );
}

// 改善後：products が変わらない限りソート結果を再利用
function ProductList({ products }) {
  const sortedProducts = useMemo(() => {
    return products.sort((a, b) => b.price - a.price);
  }, [products]);

  return (
    <ul>
      {sortedProducts.map(product => (
        <li key={product.id}>{product.name}</li>
      ))}
    </ul>
  );
}
```

このコードでは、products 配列が変更されない限り、ソート処理をスキップできる。

### useCallback の活用

これを使うことで関数の再生成を防ぐ。

```javascript
// 改善前：毎回新しい関数が生成される
function TodoList({ todos, onToggle }) {
  return (
    <ul>
      {todos.map(todo => (
        <TodoItem
          key={todo.id}
          todo={todo}
          onToggle={() => onToggle(todo.id)}
        />
      ))}
    </ul>
  );
}

// 改善後：関数インスタンスを再利用
function TodoList({ todos, onToggle }) {
  const handleToggle = useCallback((id) => {
    onToggle(id);
  }, [onToggle]);

  return (
    <ul>
      {todos.map(todo => (
        <TodoItem
          key={todo.id}
          todo={todo}
          onToggle={handleToggle}
        />
      ))}
    </ul>
  );
}
```

### React.memo によるコンポーネントのメモ化

親コンポーネントが再レンダリングされても、props が変わっていなければ子コンポーネントの再レンダリングをスキップできる。

```javascript
// 改善前
function TodoItem({ todo, onToggle }) {
  return (
    <li onClick={() => onToggle(todo.id)}>
      <input type="checkbox" checked={todo.completed} />
      {todo.text}
    </li>
  );
}

// 改善後
const TodoItem = React.memo(({ todo, onToggle }) => {
  return (
    <li onClick={() => onToggle(todo.id)}>
      <input type="checkbox" checked={todo.completed} />
      {todo.text}
    </li>
  );
});
```

### lazy loading と Suspense

初期バンドルサイズを削減するために、コード分割を実施した。

```javascript
import React, { lazy, Suspense } from 'react';

// 改善前：すべてのページを最初に読み込む
import Dashboard from './pages/Dashboard';
import Settings from './pages/Settings';
import Reports from './pages/Reports';

// 改善後：必要になったときに読み込む
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));
const Reports = lazy(() => import('./pages/Reports'));

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Routes>
        <Route path="/" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
        <Route path="/reports" element={<Reports />} />
      </Routes>
    </Suspense>
  );
}
```

### 仮想化によるリスト表示の最適化

大量のアイテムを表示する場合、react-window を使用して仮想化を実装した。

```javascript
import { FixedSizeList } from 'react-window';

function VirtualizedList({ items }) {
  const Row = ({ index, style }) => (
    <div style={style}>
      {items[index].name}
    </div>
  );

  return (
    <FixedSizeList
      height={600}
      itemCount={items.length}
      itemSize={35}
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
}
```

これにより、表示されている範囲のアイテムのみがレンダリングされる。

## パフォーマンス測定

改善前と改善後で、かなり速くなった。多くのコンポーネントで効果が確認できた。

### 測定方法

React DevTools Profiler を使用して測定を実施した。以下の手順で測定した。

1. Chrome 拡張機能の React DevTools をインストール
2. Profiler タブで記録を開始
3. 対象の操作（フィルタ変更、ページ切り替えなど）を実行
4. 記録を停止して結果を分析

### 改善結果

主要な画面での改善効果は以下の通り。

**ダッシュボード画面**

初期表示時のレンダリング時間が改善された。また、フィルタ変更時の応答性も向上した。

**設定画面**

入力時の反応速度が改善された。

**レポート画面**

大量データ表示時のパフォーマンスが改善された。

## ベストプラクティス

パフォーマンス最適化を進める上で学んだベストプラクティスを紹介する。

### 1. 測定してから最適化する

闇雲に最適化するのではなく、必ずプロファイリングツールで測定してボトルネックを特定する。

### 2. 過度な最適化を避ける

useMemo や useCallback の乱用は、かえってコードの可読性を下げる可能性がある。本当に必要な箇所にのみ適用する。

### 3. 依存配列に注意する

useMemo や useCallback の依存配列に漏れがあると、古い値を参照してバグの原因になる。

```javascript
// 悪い例：count が依存配列に含まれていない
const expensiveValue = useMemo(() => {
  return calculateExpensiveValue(count);
}, []); // count が変わっても再計算されない

// 良い例
const expensiveValue = useMemo(() => {
  return calculateExpensiveValue(count);
}, [count]);
```

### 4. React.memo の比較関数を活用する

デフォルトの shallow comparison では不十分な場合、カスタム比較関数を提供できる。

```javascript
const TodoItem = React.memo(
  ({ todo, onToggle }) => {
    // コンポーネントの実装
  },
  (prevProps, nextProps) => {
    // true を返すと再レンダリングをスキップ
    return prevProps.todo.id === nextProps.todo.id &&
           prevProps.todo.completed === nextProps.todo.completed;
  }
);
```

## トラブルシューティング

最適化の過程で遭遇した問題と解決方法を紹介する。

### 問題 1: useCallback を使っても再レンダリングが発生する

依存配列の値が毎回変わっている可能性がある。特にオブジェクトや配列を依存配列に含める場合は注意が必要。

```javascript
// 悪い例：options オブジェクトが毎回新しく生成される
function Parent() {
  const options = { sort: 'asc' }; // 毎回新しいオブジェクト

  const handleClick = useCallback(() => {
    doSomething(options);
  }, [options]); // options が毎回変わるので useCallback の意味がない
}

// 良い例
function Parent() {
  const options = useMemo(() => ({ sort: 'asc' }), []);

  const handleClick = useCallback(() => {
    doSomething(options);
  }, [options]);
}
```

### 問題 2: React.memo が効かない

props として渡している関数やオブジェクトが毎回新しく生成されていないか確認する。

## まとめ

最適化は重要です。パフォーマンスを意識した開発を心がけるべきです。

React DevTools Profiler などのツールを使って測定し、ボトルネックを特定してから対処することで、効果的な改善ができる。
