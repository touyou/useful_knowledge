# Suspenseの基本

SuspenseはReactの非同期データ取得機能で、宣言的UIの原則を保ちながら非同期処理を扱います。

## 従来の方法との違い

### Before: isLoadingパターン

```tsx
const UserProfile: React.FC = () => {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    fetchUser().then((data) => {
      setUser(data);
      setIsLoading(false);
    });
  }, []);

  if (isLoading) return <p>Loading...</p>;
  return <h1>{user?.name}のプロフィール</h1>;
};
```

### After: Suspense + use

```tsx
const UserProfile: React.FC<{ userPromise: Promise<User> }> = ({ userPromise }) => {
  const user = use(userPromise);
  return <h1>{user.name}のプロフィール</h1>;
};

const App: React.FC = () => {
  const userPromise = useMemo(() => fetchUser(), []);

  return (
    <Suspense fallback={<p>Loading...</p>}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  );
};
```

> **React公式**: `use` APIを使用したコンポーネントは、Promise解決まで「サスペンド」し、`Suspense`のフォールバックが表示されます。
> 参考: https://react.dev/reference/react/use

## 動作フロー

```
1. 初回レンダリング → コンポーネントがサスペンド（中断）
2. Suspenseがフォールバック表示 → "Loading..."
3. Promise完了 → 自動的に再レンダリング
4. データ表示 → useがPromiseの値を返す
```

## 重要な制約: Promiseの保管場所

### ❌ うまく動かないパターン

```tsx
const UserProfile: React.FC = () => {
  // コンポーネント内でPromiseを作成
  const userPromise = useMemo(() => fetchUser(), []);
  const user = use(userPromise);
  return <h1>{user.name}さんのプロフィール</h1>;
};
```

**問題**: 無限ループが発生します。

**理由**:
1. サスペンド発生
2. 再レンダリング試行
3. 新しいPromise生成（useMemoが効かない）
4. 再度サスペンド
5. 1に戻る...

**なぜuseMemoが効かない?**
- useMemoは「前回のレンダリング」の値を再利用
- サスペンド中はレンダリングが完了していない
- 記憶領域が確保されていないため機能しない

### ✅ 正しいパターン

**Promiseはコンポーネント外に保管する必要があります。**

```tsx
// 親コンポーネントでPromiseを作成
const App: React.FC = () => {
  const userPromise = useMemo(() => fetchUser(), []);

  return (
    <Suspense fallback={<p>Loading...</p>}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  );
};

// 子コンポーネントはPromiseを受け取るだけ
const UserProfile: React.FC<{ userPromise: Promise<User> }> = ({ userPromise }) => {
  const user = use(userPromise);
  return <h1>{user.name}さんのプロフィール</h1>;
};
```

## Suspense境界（Suspense Boundary）

### 基本概念

Suspense境界は「サスペンドの影響範囲を区切る」ための仕組みです。try-catchに似た動作をします。

```tsx
<Suspense fallback={<p>Loading outer...</p>}>
  <ComponentA />
  <Suspense fallback={<p>Loading inner...</p>}>
    <ComponentB />  {/* ここでサスペンドしても外には影響しない */}
  </Suspense>
</Suspense>
```

### 複数のSuspense境界

読み込み時間が異なるデータを段階的に表示できます。

```tsx
const UserProfilePage: React.FC = () => {
  const userPromise = useMemo(() => fetchUser(), []);      // 500ms
  const postsPromise = useMemo(() => fetchPosts(), []);    // 1500ms

  return (
    <Suspense fallback={<p>Loading user...</p>}>
      <UserInfo userPromise={userPromise} />
      <h2>投稿一覧</h2>
      <Suspense fallback={<p>Loading posts...</p>}>
        <PostList postsPromise={postsPromise} />
      </Suspense>
    </Suspense>
  );
};
```

**表示順序**:
1. 最初: 全体が "Loading user..."
2. 500ms後: ユーザー情報表示、投稿は "Loading posts..."
3. 1500ms後: 投稿一覧も表示

### 設計のポイント

- 読み込みに時間がかかる部分を独立したSuspense内に配置
- その部分だけサスペンドしてページ他部分は表示継続
- 「どこが消えたら困るか」を考えてSuspense境界を配置

## まとめ

| ポイント | 説明 |
|----------|------|
| Promiseの保管 | コンポーネント外に保持が必須 |
| Suspense境界 | 影響範囲を区切る |
| 段階的表示 | ネストしたSuspenseで実現 |

次は[SuspenseとjotaiToの組み合わせ](./03_suspense_plus_jotai.md)で、jotaiを使ってPromise保管の問題を解決する方法を学びます。
