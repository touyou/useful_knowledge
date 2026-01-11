# jotai-eagerを使う

jotai-eagerは「不要な非同期処理を避け、サスペンドを減らす」ためのユーティリティです。

## 問題: 不要なサスペンド

ユーザー一覧をチェックボックスで選択するアプリケーションを考えます。

```tsx
// ユーザー一覧（非同期取得）
const usersAtom = atom(async () => fetchUsers());

// 選択状態（null = 全選択、Set = 個別選択）
const internalSelectedUserIdsAtom = atom<Set<string> | null>(null);

// 選択されたユーザーID（派生atom）
const selectedUserIdsAtom = atom(async (get): Promise<Set<string>> => {
  const selectedUserIds = get(internalSelectedUserIdsAtom);
  if (selectedUserIds !== null) {
    return selectedUserIds;  // 個別選択の場合はそのまま返す
  }

  // 全選択の場合はユーザー一覧から生成
  const users = await get(usersAtom);
  return new Set(users.map((user) => user.id));
});
```

**問題**: チェックボックスを操作するたびに`selectedUserIdsAtom`が再計算され、サスペンドが発生します。

**気づき**: 「2回目以降は実際には非同期処理が不要では？」

- 初回: ユーザーデータが必要（非同期）
- 2回目以降: データはメモリに存在（同期で十分）

## jotai-eagerの解決策

`eagerAtom`は、非同期処理が本当に必要な場合のみPromiseを返します。

```tsx
import { eagerAtom } from 'jotai-eager';

const selectedUserIdsAtom = eagerAtom((get) => {
  const selectedUserIds = get(internalSelectedUserIdsAtom);
  if (selectedUserIds !== null) {
    return selectedUserIds;  // 同期的に返す
  }

  // getがPromiseを解決済みなら同期的に返す
  const users = get(usersAtom);
  return new Set(users.map((user) => user.id));
});
```

### 通常のatomとの違い

| 特徴 | 通常のatom | eagerAtom |
|------|------------|-----------|
| 読み取り関数 | async可能 | 同期関数のみ |
| getの戻り値 | `Promise<T>` | `T`（use風） |
| 戻り値の型 | `Promise<T>` | `T \| Promise<T>` |

### eagerAtomのgetの動作

```tsx
const eagerDerivedAtom = eagerAtom((get) => {
  // 依存先がPromise未解決 → ここでサスペンド
  // 依存先がPromise解決済み → 同期的に値を取得
  const data = get(asyncAtom);
  return processData(data);
});
```

これはReactの`use`フックに似た動作です。

## Promiseキャッシング

jotai-eagerは「一度解決したPromiseの結果をキャッシュ」します。

```
初回読み取り: get(usersAtom) → Promise → サスペンド
2回目以降: get(usersAtom) → キャッシュされた値 → 同期的に返す
```

これが「2回目以降のチェックボックス操作時にサスペンドが発生しない」理由です。

## 実践例: テーマ切り替え

ユーザー設定から初期テーマを取得し、手動で切り替える機能です。

```tsx
type Theme = 'light' | 'dark';

// ユーザー設定（非同期取得）
const userPreferencesAtom = atom(async () => {
  return fetchUserPreferences();  // { preferredTheme: 'dark', ... }
});

// 内部状態（null = 初期値未設定）
const internalThemeAtom = atom<Theme | null>(null);

// 現在のテーマ
const themeAtom = eagerAtom((get): Theme => {
  const internalTheme = get(internalThemeAtom);
  if (internalTheme !== null) {
    return internalTheme;  // 手動設定があればそれを使う
  }

  // 初期値はユーザー設定から取得
  const preferences = get(userPreferencesAtom);
  return preferences.preferredTheme;
});

// テーマ切り替えアクション
const toggleThemeAtom = atom(null, async (get, set) => {
  const currentTheme = await get(themeAtom);
  const newTheme: Theme = currentTheme === 'light' ? 'dark' : 'light';
  set(internalThemeAtom, newTheme);
});
```

**動作**:
1. 初回: `userPreferencesAtom`を待つ（サスペンド）
2. テーマ表示後: トグルボタンで即座に切り替え（サスペンドなし）

## いつeagerAtomを使うか

| 状況 | 推奨 |
|------|------|
| 常に非同期処理が必要 | 通常のasync atom |
| 初回のみ非同期、以降は同期 | eagerAtom |
| 条件によって非同期/同期が変わる | eagerAtom |

## 注意点

- `eagerAtom`の読み取り関数は同期関数
- `async/await`は使えない
- 非同期処理は`get`が内部で処理

## まとめ

| ポイント | 説明 |
|----------|------|
| 目的 | 不要なサスペンドを削減 |
| 仕組み | Promiseのキャッシュと条件付き同期処理 |
| 使いどころ | 初回のみ非同期が必要なケース |

次は[エラーハンドリング](./09_error_handling.md)でError Boundaryとリトライを学びます。
