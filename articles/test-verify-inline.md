---
title: "React のパフォーマンス最適化テクニック"
emoji: "⚡"
type: "tech"
topics:
  - "react"
  - "performance"
  - "optimization"
published: false
---

## はじめに

React アプリケーションのパフォーマンス最適化は重要です。この記事では実践的な最適化手法を紹介します。メモ化、コンポーネント分割、仮想化など、様々なテクニックを解説し、実際のコード例とともに説明します。

## メモ化によるパフォーマンス改善

useMemo と useCallback を使うことで、不必要な再レンダリングを防ぐことができます。これによりアプリケーションの応答性が向上します。さらにユーザー体験が改善され、メモリ使用量も削減されます。

```typescript
const MemoizedComponent = React.memo(({ data }) => {
  const processedData = useMemo(() => {
    return data.map(item => item.value * 2);
  }, [data]);

  return <div>{processedData}</div>;
});
```

上記のコードでメモ化を実現できます。

## useCallback の活用

useCallback を使うことで関数の再生成を防げます。これにより子コンポーネントの不要な再レンダリングを防止でき、アプリケーション全体のパフォーマンスが向上し、メモリ使用量も最適化され、ユーザー体験が改善されます。

```typescript
function ParentComponent() {
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []);

  return <ChildComponent onClick={handleClick} />;
}
```

このコード例により、関数のメモ化が実現され、パフォーマンスの問題を解決できます。

## コンポーネント分割

大きなコンポーネントは分割することで、再レンダリングの範囲を最小化できます。状態管理を適切に行い、不要な props の受け渡しを避けることで、パフォーマンスが向上します。

```typescript
function LargeComponent() {
  const [count, setCount] = useState(0);
  const [text, setText] = useState('');

  return (
    <div>
      <Counter count={count} setCount={setCount} />
      <TextInput text={text} setText={setText} />
    </div>
  );
}
```

## リスト描画の最適化

リストを描画する際は、必ず key を指定してください。これにより React が効率的に DOM を更新できます。

```typescript
function ItemList({ items }) {
  return (
    <ul>
      {items.map((item, index) => (
        <li key={index}>{item.name}</li>
      ))}
    </ul>
  );
}
```

## 仮想化の導入

長いリストを扱う場合、react-window や react-virtualized を使うことで、表示されている要素のみをレンダリングできます。

```typescript
import { FixedSizeList } from 'react-window';

function VirtualizedList({ items }) {
  return (
    <FixedSizeList
      height={400}
      itemCount={items.length}
      itemSize={35}
    >
      {({ index, style }) => (
        <div style={style}>{items[index].name}</div>
      )}
    </FixedSizeList>
  );
}
```

これでパフォーマンスが大幅に改善されます。

## まとめ

適切な最適化により React アプリケーションは高速に動作します。実際のプロジェクトでは、これらのテクニックを組み合わせて使うことで、ユーザー体験を向上させることができます。パフォーマンス測定ツールを使って実際の効果を確認することも重要です。
