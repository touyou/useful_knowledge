# SuspenseとjotaiToの組み合わせ

jotaiの派生atomはキャッシュ機能を持つため、Suspenseの「Promiseをコンポーネント外に保管する」制約を自然に解決できます。

## 基本パターン: 非同期派生atom

```tsx
import { atom, useAtomValue } from 'jotai';

// 非同期の派生atom
const userAtom = atom(async (): Promise<User> => {
  const user = await fetchUser();
  return user;
});

const App: React.FC = () => {
  return (
    <Suspense fallback={<p>Loading...</p>}>
      <UserProfile />
    </Suspense>
  );
};

const UserProfile: React.FC = () => {
  // useAtomValueは内蔵のuseによりPromiseを自動処理
  const user: User = useAtomValue(userAtom);

  return (
    <section>
      <h1>{user.name}さんのプロフィール</h1>
    </section>
  );
};
```

**ポイント**:
- `useAtomValue`は非同期atomからPromiseを解決した値を返す
- 計算は最初の読み取り時に実行され、キャッシュされる
- 依存先がないため再計算されない

## パラメータ付き非同期処理

### 方法1: パラメータatomに依存する派生atom

単一パラメータを切り替える場合に適しています。

```tsx
// パラメータを保持するatom
const userIdAtom = atom<string | null>(null);

// パラメータに依存する派生atom
const userAtom = atom(async (get): Promise<User | null> => {
  const userId = get(userIdAtom);
  if (userId === null) return null;

  const user = await fetchUser(userId);
  return user;
});
```

**使用例**:

```tsx
const App: React.FC = () => {
  const setUserId = useSetAtom(userIdAtom);

  return (
    <>
      <UserSelector onSelectUser={(id) => setUserId(id)} />
      <Suspense fallback={<p>Loading...</p>}>
        <UserProfile />
      </Suspense>
    </>
  );
};

const UserProfile: React.FC = () => {
  const user = useAtomValue(userAtom);

  if (user === null) {
    return <p>ユーザーが選択されていません。</p>;
  }

  return <h1>{user.name}さんのプロフィール</h1>;
};
```

**設計思想**: 「非同期処理を実行する」のではなく「パラメータを更新する」ことで、Reactの `UI = f(state)` の原則に沿います。

### 方法2: atomファクトリー（atomFamily）

複数パラメータを同時に扱う場合や、キャッシュを独立させたい場合に適しています。

```tsx
import { atomFamily } from 'jotai/utils';

// パラメータごとにatomを生成
const userAtomFamily = atomFamily((userId: string) =>
  atom(async (): Promise<User> => {
    const user = await fetchUser(userId);
    return user;
  })
);

const UserProfile: React.FC<{ userId: string }> = ({ userId }) => {
  const userAtom = userAtomFamily(userId);
  const user = useAtomValue(userAtom);

  return <h1>{user.name}さんのプロフィール</h1>;
};
```

**メリット**: 各パラメータのキャッシュが独立しており、`user1` → `user2` → `user1` の遷移時も効率的（user1のキャッシュが残る）。

### 手動実装版（atomFamilyなし）

```tsx
const createUserAtom = (userId: string) =>
  atom(async (): Promise<User> => {
    const user = await fetchUser(userId);
    return user;
  });

const userAtoms = new Map<string, ReturnType<typeof createUserAtom>>();

const getUserAtom = (userId: string) => {
  let userAtom = userAtoms.get(userId);
  if (!userAtom) {
    userAtom = createUserAtom(userId);
    userAtoms.set(userId, userAtom);
  }
  return userAtom;
};
```

## メモリ管理の注意点

atomファクトリーパターンでは、不要なatomをMapから削除しないとメモリリークが発生します。

- 大量のパラメータを扱う場合は要注意
- サーバー環境では特に注意が必要
- `atomFamily`にはタイムスタンプベースの自動削除機能あり

## 設計パラダイムの変化

### 従来のReact

```
ボタンクリック → 非同期処理 → ステート更新 → UI更新
```

### Suspense + jotai

```
ボタンクリック → ステート更新 → 非同期処理（サスペンド） → UI更新
```

**重要**: Promiseそのものをステートとして保持する発想により:
- ステート数が削減される
- Reactの責務に非同期処理が統合される
- 宣言的なコードが書ける

## 実践例: 商品検索

```tsx
const searchKeywordAtom = atom('');

const searchResultsAtom = atom(async (get): Promise<Product[]> => {
  const keyword = get(searchKeywordAtom);
  if (keyword === '') return [];

  const products = await searchProducts(keyword);
  return products;
});

const SearchBox: React.FC = () => {
  const setKeyword = useSetAtom(searchKeywordAtom);

  return (
    <form action={(formData) => {
      const keyword = formData.get('keyword') as string;
      setKeyword(keyword);
    }}>
      <input name="keyword" />
      <button type="submit">検索</button>
    </form>
  );
};

const SearchResults: React.FC = () => {
  const results = useAtomValue(searchResultsAtom);

  return (
    <ul>
      {results.map((product) => (
        <li key={product.id}>
          {product.name} - ¥{product.price}
        </li>
      ))}
    </ul>
  );
};
```

## まとめ

| パターン | 用途 |
|----------|------|
| パラメータatom依存 | 単一パラメータの切り替え |
| atomFamily | 複数パラメータ、独立キャッシュ |

次は[AbortSignalを使った非同期処理の中断](./04_abort_signal.md)で、不要になったリクエストをキャンセルする方法を学びます。
