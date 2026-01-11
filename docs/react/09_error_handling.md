# エラーハンドリング

非同期処理には必ずエラーが発生する可能性があります。PromiseのReject機能とReactのError Boundaryを組み合わせてエラーハンドリングを実装します。

## Error Boundaryの基本

Error Boundaryはコンポーネントツリー内で発生したエラーをキャッチし、フォールバックUIを表示します。

### SuspenseとError Boundaryの共通点

| 機能 | トリガー | 対応 |
|------|----------|------|
| Suspense | Promise（未完了） | フォールバック表示 |
| Error Boundary | throw Error | フォールバック表示 |

両者とも「レンダリング失敗」という状況に対応しています。

## Error Boundaryの実装

Reactから`ErrorBoundary`コンポーネントは直接提供されないため、クラスコンポーネントで実装します。

```tsx
class MyErrorBoundary extends React.Component<
  { fallback: React.ReactNode; children: React.ReactNode },
  { hasError: boolean }
> {
  constructor(props: { fallback: React.ReactNode; children: React.ReactNode }) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error: Error) {
    return { hasError: true };
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback;
    }
    return this.props.children;
  }
}
```

### ライブラリの活用

実際のプロジェクトでは[react-error-boundary](https://github.com/bvaughn/react-error-boundary)を使うことが推奨されます。

```tsx
import { ErrorBoundary } from 'react-error-boundary';

<ErrorBoundary fallback={<div>エラーが発生しました</div>}>
  <MyComponent />
</ErrorBoundary>
```

## 非同期処理でのエラー

Promiseがrejectされると、`use`や`useAtomValue`の箇所でエラーがthrowされます。

```tsx
const userAtom = atom(async () => {
  const response = await fetch('/api/user');
  if (!response.ok) {
    throw new Error('ユーザー取得に失敗');
  }
  return response.json();
});

const UserProfile: React.FC = () => {
  const user = useAtomValue(userAtom);  // ここでエラーがthrow
  return <h1>{user.name}</h1>;
};
```

## Error Boundaryの配置

「この中だけフォールバックUIに切り替わっても不自然ではない」という範囲で配置します。

### ❌ 悪い例

```tsx
<ErrorBoundary fallback={<div>エラーが発生しました</div>}>
  <SearchBox />
  <Suspense fallback={<div>ローディング中...</div>}>
    <SearchResults />
  </Suspense>
</ErrorBoundary>
```

**問題**: エラー時に検索ボックスまで消え、ユーザーは再検索できません。

### ✅ 良い例

```tsx
<SearchBox />
<ErrorBoundary fallback={<div>エラーが発生しました</div>}>
  <Suspense fallback={<div>ローディング中...</div>}>
    <SearchResults />
  </Suspense>
</ErrorBoundary>
```

検索ボックスが残り、再試行が可能です。

## エラーからの回復（リトライ）

### react-error-boundaryを使う場合

```tsx
import { ErrorBoundary, type FallbackProps } from 'react-error-boundary';

const ArticleListFallback: React.FC<FallbackProps> = ({
  error,
  resetErrorBoundary,
}) => {
  const reloadArticles = useSetAtom(articlesAtom);

  const handleRetry = () => {
    startTransition(() => {
      resetErrorBoundary();  // Error Boundaryをリセット
      reloadArticles();       // atomを再実行
    });
  };

  return (
    <div>
      <p>エラー: {error.message}</p>
      <button type="button" onClick={handleRetry}>
        再試行
      </button>
    </div>
  );
};

// 使用例
<ErrorBoundary FallbackComponent={ArticleListFallback}>
  <Suspense fallback={<p>読み込み中...</p>}>
    <ArticleList />
  </Suspense>
</ErrorBoundary>
```

**ポイント**: `startTransition`で両方の更新をまとめることで、スムーズにローディング状態に遷移します。

## jotaiでのエラーハンドリング

### atom内でエラーをキャッチ

```tsx
const userAtom = atom(async (get) => {
  try {
    const user = await fetchUser();
    return user;
  } catch (error) {
    handleError(error);  // ログ送信など
    throw error;          // 再throw（Error Boundaryに渡す）
  }
});
```

### ユーティリティ関数

```tsx
const handleErrorAndRethrow = (error: unknown) => {
  console.error('API Error:', error);
  // Sentryなどに送信
  throw error;
};

const userAtom = atom(async (get) => {
  const user = await fetchUser().catch(handleErrorAndRethrow);
  return user;
});
```

## エラーモーダルダイアログ

jotaiでエラーUI状態を管理する方法です。

```tsx
// エラー状態のatom
const errorAtom = atom<unknown>(null);

// エラーモーダル
const ErrorModalContainer: React.FC = () => {
  const [error, setError] = useAtom(errorAtom);

  if (!error) return null;

  return (
    <ErrorModal
      error={error}
      onClose={() => setError(null)}
    />
  );
};

// Reactの外からでも設定可能
import { getDefaultStore } from 'jotai';

function handleError(error: unknown) {
  const store = getDefaultStore();
  store.set(errorAtom, error);
}
```

## 従来の方法との比較

### Before: エラーハンドリングが不十分

```tsx
const [user, setUser] = useState<User | null>(null);
useEffect(() => {
  fetchUser()
    .then((data) => setUser(data))
    .catch((error) => {
      console.error(error);  // ログだけ
      // UIは永遠にローディング中...
    });
}, []);
```

### After: Suspense + Error Boundary

```tsx
<ErrorBoundary fallback={<ErrorUI />}>
  <Suspense fallback={<LoadingUI />}>
    <UserProfile />
  </Suspense>
</ErrorBoundary>
```

**Suspenseはエラー時のUIに向き合うことを強制します。**

## 実践例: ニュース記事一覧（リトライ付き）

```tsx
// 再読み込み可能なatom
const articlesAtom = createReloadableAtom(async () => {
  const response = await fetch('/api/articles');
  if (!response.ok) throw new Error('記事の取得に失敗');
  return response.json();
});

const ArticleListFallback: React.FC<FallbackProps> = ({
  error,
  resetErrorBoundary,
}) => {
  const reloadArticles = useSetAtom(articlesAtom);

  return (
    <div>
      <p>エラー: {error.message}</p>
      <button
        onClick={() => {
          startTransition(() => {
            resetErrorBoundary();
            reloadArticles();
          });
        }}
      >
        再試行
      </button>
    </div>
  );
};

const App: React.FC = () => (
  <div>
    <h1>ニュース記事一覧</h1>
    <ErrorBoundary FallbackComponent={ArticleListFallback}>
      <Suspense fallback={<p>読み込み中...</p>}>
        <ArticleList />
      </Suspense>
    </ErrorBoundary>
  </div>
);
```

## まとめ

| ポイント | 説明 |
|----------|------|
| Error Boundary | レンダリング中のエラーをキャッチ |
| 配置 | 影響範囲を考慮して配置 |
| リトライ | resetErrorBoundary + atom再読み込み |
| jotai連携 | ストア直接操作でReact外からも設定可能 |

---

これでReact × jotaiナレッジのすべての章が完了しました。[目次に戻る](./00_index.md)
