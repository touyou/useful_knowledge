# トランジションの基本

トランジションはReactの機能で、「優先度が低い更新」として扱われるステート更新メカニズムです。Suspenseと深い関係があります。

## Suspenseとの違い

| 状況 | 通常のSuspense | トランジション |
|------|---------------|----------------|
| サスペンド時 | フォールバック表示 | 古いUI維持 |
| ユーザー体験 | ちらつく可能性 | スムーズ |

## 基本的な使い方

### 方法1: startTransition関数

```tsx
import { startTransition } from 'react';

const handleSelectUser = (id: string) => {
  startTransition(() => {
    setUserId(id);
  });
};
```

**問題点**: ボタン押下時のフィードバックがありません。

### 方法2: useTransitionフック（推奨）

```tsx
import { useTransition } from 'react';

const App: React.FC = () => {
  const [isPending, startTransition] = useTransition();
  const [userId, setUserId] = useState('user1');

  const handleSelectUser = (id: string) => {
    startTransition(() => {
      setUserId(id);
    });
  };

  return (
    <>
      <UserSelector onSelectUser={handleSelectUser} />
      <Suspense fallback={<p>Loading...</p>}>
        <UserProfile userId={userId} isPending={isPending} />
      </Suspense>
    </>
  );
};

const UserProfile: React.FC<{ userId: string; isPending: boolean }> =
  ({ userId, isPending }) => {
  const user = useAtomValue(userAtomFamily(userId));

  return (
    <section>
      <h1>{user.name}さんのプロフィール</h1>
      {isPending && <span>(読み込み中...)</span>}
    </section>
  );
};
```

> **React公式**: `useTransition`は`isPending`フラグと`startTransition`関数を返します。`isPending`はトランジション進行中かどうかを示します。
> 参考: https://react.dev/reference/react/useTransition

## 動作フロー

```
1. ボタン押下 → isPendingがtrueに（フィードバック表示）
2. 古いUI表示継続（フォールバック非表示）
3. データ取得後 → 新UI表示、isPendingがfalseに
```

## 複数トランジションの使い分け

各UIパーツごとに独立したトランジションを使うことで、正確なフィードバックを提供できます。

```tsx
const Dashboard: React.FC = () => {
  const [isPendingUser, startTransitionUser] = useTransition();
  const [isPendingPost, startTransitionPost] = useTransition();

  const [userId, setUserId] = useState('user1');
  const [category, setCategory] = useState<Category>('tech');

  const handleSelectUser = (id: string) => {
    startTransitionUser(() => {
      setUserId(id);
    });
  };

  const handleSelectCategory = (cat: Category) => {
    startTransitionPost(() => {
      setCategory(cat);
    });
  };

  return (
    <div>
      <section>
        <h2>ユーザープロフィール {isPendingUser && '(読み込み中...)'}</h2>
        <UserSelector onSelectUser={handleSelectUser} />
        <Suspense fallback={<p>Loading user...</p>}>
          <UserProfile userId={userId} />
        </Suspense>
      </section>

      <section>
        <h2>投稿一覧 {isPendingPost && '(読み込み中...)'}</h2>
        <CategorySelector onSelectCategory={handleSelectCategory} />
        <Suspense fallback={<p>Loading posts...</p>}>
          <PostList category={category} />
        </Suspense>
      </section>
    </div>
  );
};
```

**重要**: 単一トランジションの使い回しは誤った情報をユーザーに与える可能性があります。

## トランジション内包コンポーネント

ボタンコンポーネントにトランジションを内包するパターンです。

```tsx
const MyButton: React.FC<{
  action: () => void;
  children: React.ReactNode;
}> = ({ action, children }) => {
  const [isPending, startTransition] = useTransition();

  const handleClick = () => {
    startTransition(() => {
      action();
    });
  };

  return (
    <button type="button" onClick={handleClick} disabled={isPending}>
      {isPending ? <LoadingSpinner /> : children}
    </button>
  );
};

// 使用例
<MyButton action={() => setUserId('user2')}>
  ユーザー2を選択
</MyButton>
```

**命名規則**: propsは`onClick`ではなく`action`または`clickAction`を推奨。これは「トランジション内で実行される関数」であることを示します（React 19の「アクション」概念）。

## ペンディング表示のちらつき対策

キャッシュが効いている場合、データ取得がすぐに完了してもペンディング表示が一瞬現れる問題があります。

```tsx
const UserProfile: React.FC<{ userId: string; isPending: boolean }> =
  ({ userId, isPending }) => {
  const user = useAtomValue(userAtomFamily(userId));

  return (
    <section>
      <h1>{user.name}さんのプロフィール</h1>
      <p
        aria-hidden={!isPending}
        style={{
          opacity: isPending ? 1 : 0,
          transition: 'opacity 0.3s ease',
        }}
      >
        (読み込み中...)
      </p>
    </section>
  );
};
```

**解決策**:
- CSSアニメーションによるフェードイン・フェードアウト
- `@starting-style`の活用
- タイマーを使った遅延表示

## まとめ

| ポイント | 説明 |
|----------|------|
| useTransition | isPendingでフィードバック提供 |
| 複数トランジション | 各UI部分で独立管理 |
| アクション | トランジション内の関数 |

次は[トランジションの応用](./07_transition_advanced.md)で「世界の分岐」とオプトアウトについて学びます。
