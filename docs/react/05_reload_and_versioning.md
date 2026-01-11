# 派生atomの再読み込みとUIバージョニング

派生atomはキャッシュされるため、同じPromiseを返し続けます。しかし、ユーザー操作による再取得が必要な場合があります。

## 再読み込みキーパターン

### 基本形式

```tsx
// 再読み込み用のキーatom
const refetchKeyAtom = atom(0);

// キーに依存する派生atom
const userAtom = atom(async (get) => {
  get(refetchKeyAtom);  // 再読み込みキーを参照
  const user = await fetchUser();
  return user;
});
```

**原理**: 派生atomは依存しているatomが変化したときのみ再計算されます。

### リロードボタンの実装

```tsx
const ReloadButton: React.FC = () => {
  const setRefetchKey = useSetAtom(refetchKeyAtom);

  const handleReload = () => {
    setRefetchKey((key) => key + 1);  // キーをインクリメント
  };

  return (
    <button type="button" onClick={handleReload}>
      再読み込み
    </button>
  );
};
```

## インターフェースの改善

キーatomを直接公開するのではなく、操作用atomを提供します。

```tsx
// 内部: キーatom
const refetchKeyAtom = atom(0);

// 公開: データatom
const userAtom = atom(async (get) => {
  get(refetchKeyAtom);
  return fetchUser();
});

// 公開: 再読み込みatom
const reloadUserAtom = atom(null, (get, set) => {
  set(refetchKeyAtom, (key) => key + 1);
});

// 使用例
const ReloadButton: React.FC = () => {
  const reloadUser = useSetAtom(reloadUserAtom);
  return (
    <button type="button" onClick={() => reloadUser()}>
      再読み込み
    </button>
  );
};
```

## 再利用可能なユーティリティ

```tsx
import { atom, Getter, Atom, WritableAtom } from 'jotai';

function createReloadableAtom<T>(
  getter: (get: Getter) => T
): WritableAtom<T, [], void> {
  const refetchKeyAtom = atom(0);

  return atom(
    (get) => {
      get(refetchKeyAtom);
      return getter(get);
    },
    (get, set) => {
      set(refetchKeyAtom, (key) => key + 1);
    }
  );
}
```

**使用例**:

```tsx
const userAtom = createReloadableAtom(async () => {
  return fetchUser();
});

const UserProfile: React.FC = () => {
  const user = useAtomValue(userAtom);
  const reloadUser = useSetAtom(userAtom);  // 同じatomで再読み込み

  return (
    <section>
      <h1>{user.name}さんのプロフィール</h1>
      <button type="button" onClick={() => reloadUser()}>
        再読み込み
      </button>
    </section>
  );
};
```

## UIバージョニングの考え方

再読み込みキーは単なる「無意味な数値」ではなく、**UIのバージョン識別子**として解釈できます。

### 整数を使う理由

- 各バージョンが一意に識別できる
- 真偽値より「バージョン」という概念が明確
- デバッグ時に何回目の読み込みかわかる

### 宣言的UIとの整合性

Reactの視点では:
- UIの計算は純粋でなければならない
- データ取得もUIの計算の一部
- 同じパラメータに対しては同じ結果が得られるべき

再読み込みキーは「パラメータ自体の変化」と解釈することで、純粋性の要件を満たします。

```
UI = f(state)
↓
UI = f(userId, refetchKey)  // refetchKeyもパラメータの一部
```

## 別解: 分離型ユーティリティ

データatomと再読み込みatomを分離する方法もあります。

```tsx
function createReloadableAtom<T>(
  getter: (get: Getter) => T
): {
  dataAtom: Atom<T>;
  reloadAtom: WritableAtom<null, [], void>;
} {
  const refetchKeyAtom = atom(0);

  const dataAtom = atom((get) => {
    get(refetchKeyAtom);
    return getter(get);
  });

  const reloadAtom = atom(null, (get, set) => {
    set(refetchKeyAtom, (key) => key + 1);
  });

  return { dataAtom, reloadAtom };
}

// 使用例
const { dataAtom: userAtom, reloadAtom: reloadUserAtom } = createReloadableAtom(
  async () => fetchUser()
);
```

## まとめ

| 概念 | 説明 |
|------|------|
| 再読み込みキー | 依存先の変化で再計算をトリガー |
| UIバージョン | キーをバージョン識別子として解釈 |
| カプセル化 | キーatomを隠蔽し、操作atomを公開 |

次は[トランジションの基本](./06_transition_basics.md)でスムーズなUI遷移を学びます。
