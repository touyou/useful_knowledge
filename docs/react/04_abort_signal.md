# AbortSignalを使った非同期処理の中断

AbortSignalは「中断を伝える仕組み」で、不要になった非同期処理を効率的に止められます。

## ユースケース

- user1のデータ取得中にuser2に切り替えた → user1の処理を中断
- 検索キーワード入力中に新しいキーワードが入力された → 古い検索を中断
- コンポーネントがアンマウントされた → 進行中のリクエストを中断

## jotaiでの実装

派生atomの読み取り関数は自動的にAbortSignalを受け取ります。

```tsx
const userAtom = atom(async (get, { signal }): Promise<User> => {
  const userId = get(userIdAtom);

  // signalをfetchに渡すだけ
  const response = await fetch(`/api/users/${userId}`, { signal });
  return await response.json();
});
```

**ポイント**: `signal`を非同期処理に渡すだけで対応完了です。

## 中断時の動作

中断されると`AbortError`が発生し、以降のコードは実行されません。

```tsx
const dataAtom = atom(async (get, { signal }): Promise<Data> => {
  const response = await fetch('/api/data', { signal });
  // ↑ここで中断された場合、以降は実行されない

  console.log('This will not be logged if aborted');
  return await response.json();
});
```

**特徴**:
- 中断エラーはキャッシュされない
- UIにも影響しない（Error Boundaryに渡らない）
- 単に「なかったこと」になる

## 複数の非同期処理

複数の非同期処理がある場合、同じsignalを渡します。

```tsx
const userDataAtom = atom(async (get, { signal }) => {
  const userId = get(userIdAtom);

  // Promise.allで並列実行
  const [user, posts] = await Promise.all([
    fetchUser(userId, signal),
    fetchPosts(userId, signal),
  ]);

  return { user, posts };
});
```

## カスタムfetch関数での対応

```tsx
async function fetchUser(userId: string, signal?: AbortSignal): Promise<User> {
  const response = await fetch(`/api/users/${userId}`, { signal });

  if (!response.ok) {
    throw new Error(`HTTP error! status: ${response.status}`);
  }

  return await response.json();
}
```

## atomファクトリーとの違い

atomファクトリーでは別々のatomが作成されるため、パラメータ変更時に中断は発生しません。

| パターン | 中断発生 |
|----------|----------|
| パラメータatom依存 | ✅ 発生する（同じatomの再計算） |
| atomFamily | ❌ 発生しない（別のatom） |

```tsx
// パラメータatom依存: user1 → user2 で user1の処理が中断される
const userAtom = atom(async (get, { signal }) => {
  const userId = get(userIdAtom);  // userIdAtomが変わると再計算
  return fetchUser(userId, signal);
});

// atomFamily: user1 → user2 でも user1の処理は継続
const userAtomFamily = atomFamily((userId: string) =>
  atom(async () => fetchUser(userId))  // 別のatom
);
```

## 手動でAbortControllerを使う場合

特殊なケースでは手動でAbortControllerを作成することもあります。

```tsx
const searchAtom = atom(async (get, { signal }) => {
  const keyword = get(searchKeywordAtom);

  // デバウンス処理
  await new Promise((resolve, reject) => {
    const timeoutId = setTimeout(resolve, 300);

    // 中断されたらタイムアウトをキャンセル
    signal.addEventListener('abort', () => {
      clearTimeout(timeoutId);
      reject(new DOMException('Aborted', 'AbortError'));
    });
  });

  return searchProducts(keyword, signal);
});
```

## まとめ

| ポイント | 説明 |
|----------|------|
| 自動提供 | jotaiの派生atomはsignalを自動受信 |
| 渡すだけ | fetch等にsignalを渡すだけで対応完了 |
| エラー処理不要 | 中断エラーはUIに影響しない |

次は[派生atomの再読み込みとUIバージョニング](./05_reload_and_versioning.md)でデータの再取得パターンを学びます。
