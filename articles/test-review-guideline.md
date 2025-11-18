---
title: "React 18のパフォーマンス最適化テクニック完全ガイド"
emoji: "⚡"
type: "tech"
topics:
  - "react"
  - "performance"
  - "frontend"
published: false
---

## はじめに

Reactのアプリケーションのパフォーマンスのパフォーマンス最適化は、ユーザー体験を向上させるために重要です。大規模なアプリケーションでは、適切な最適化を行わないと、レスポンスが遅くなり、ユーザー満足度が低下してしまいます。

この記事では、React 18で使えるパフォーマンス最適化のテクニックについて解説します。

## 前提知識

この記事を読むには、Reactの基本的な知識が必要です。

## なぜパフォーマンス最適化が必要か

パフォーマンスの問題は、主に以下の原因で発生します。

- 不要な再レンダリング
- 重い計算処理
- 大きなバンドルサイズ

これらを解決することで、アプリケーションの速度が向上します。

## useMemoによるメモ化

useMemoは、計算結果をキャッシュするフックです。これを使うことで、同じ計算を繰り返し実行することを避けられます。

```jsx
function ExpensiveComponent({ items }) {
  const sortedItems = useMemo(() => {
    return items.sort((a, b) => a.price - b.price);
  }, [items]);

  return (
    <ul>
      {sortedItems.map(item => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
}
```

上記の例では、`items`が変わらない限り、ソート処理は実行されません。これにより、パフォーマンスが向上します。

## useCallbackによる関数のメモ化

useCallbackは、関数をメモ化するために使用するフックで、子コンポーネントに関数を渡す際に不要な再レンダリングを防ぐことができ、特にReact.memoと組み合わせて使用すると効果的で、大量のリストアイテムをレンダリングする場合にパフォーマンスの改善が期待できます。

```jsx
function ParentComponent() {
  const [count, setCount] = useState(0);

  const handleClick = useCallback(() => {
    console.log('Button clicked');
  }, []);

  return (
    <div>
      <ChildComponent onClick={handleClick} />
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
    </div>
  );
}
```

これを使うと最適化できます。

## React.memoによるコンポーネントのメモ化

React.memoは、コンポーネント全体をメモ化します。

```jsx
const ExpensiveChild = React.memo(({ data }) => {
  console.log('Rendering ExpensiveChild');
  return <div>{data}</div>;
});

function Parent() {
  const [count, setCount] = useState(0);
  const data = "Static data";

  return (
    <div>
      <ExpensiveChild data={data} />
      <button onClick={() => setCount(count + 1)}>
        Re-render parent
      </button>
    </div>
  );
}
```

親コンポーネントが再レンダリングされても、propsが変わっていなければ`ExpensiveChild`は再レンダリングされません。

## 仮想化によるリスト最適化

大量のデータを表示する場合は、react-windowを使用します。

```jsx
import { FixedSizeList } from 'react-window';

function VirtualizedList({ items }) {
  return (
    <FixedSizeList
      height={600}
      itemCount={items.length}
      itemSize={35}
      width="100%"
    >
      {({ index, style }) => (
        <div style={style}>
          {items[index].name}
        </div>
      )}
    </FixedSizeList>
  );
}
```

仮想化を使うと、表示されている部分だけがレンダリングされます。

## Code Splitting

React.lazyとSuspenseを使って、コンポーネントを遅延読み込みします。

```jsx
const HeavyComponent = React.lazy(() => import('./HeavyComponent'));

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <HeavyComponent />
    </Suspense>
  );
}
```

これにより、初期ロード時のバンドルサイズが削減されます。

## パフォーマンス測定

React DevToolsのProfilerを使用して測定します。

1. React DevToolsを開く
2. Profilerタブを選択
3. 記録を開始してアプリケーションを操作する

これで、どのコンポーネントが遅いかを特定できます。

## まとめ

Reactのパフォーマンス最適化について、以下のテクニックを紹介しました。

- useMemoで計算結果をメモ化すること
- useCallbackで関数をメモ化する
- React.memoでコンポーネント全体をメモ化すること
- 仮想化で大量のリストを効率的に表示する
- Code Splittingでバンドルサイズを削減する

これらのテクニックを適切に使い分けることで、高速なReactアプリケーションを構築できます。
