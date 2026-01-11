# jotaiの基本

jotaiは「ステートの定義」と「ステートの読み書き」を分離するシンプルな状態管理ライブラリです。

## 基本概念

| 概念 | 役割 |
|------|------|
| `atom` | ステートの定義（値の箱） |
| `useAtom` | ステートの読み書き |

## 主要API

### フック一覧

```tsx
import { useAtom, useAtomValue, useSetAtom } from 'jotai';

// 読み書き両対応（useState相当）
const [count, setCount] = useAtom(countAtom);

// 読み取り専用
const count = useAtomValue(countAtom);

// 書き込み専用
const setCount = useSetAtom(countAtom);
```

## atomの種類

### 1. プリミティブatom

最も基本的な形。直接値を保持します。

```tsx
import { atom } from 'jotai';

const countAtom = atom(0);
const nameAtom = atom('');
const userAtom = atom<User | null>(null);
```

### 2. 読み取り専用の派生atom

他のatomから計算される値を定義します。

```tsx
const countAtom = atom(0);

// countAtomの値をフォーマットして返す
const countDisplayAtom = atom((get) => {
  return get(countAtom).toLocaleString();
});
```

**重要**: 派生atomは依存先のatomが変化したときのみ再計算されます（キャッシュ機能）。

### 3. 書き込み可能な派生atom

書き込み時のロジックをカプセル化します。

```tsx
const countAtom = atom(0);

// 第1引数: read関数（nullで読み取り不可）
// 第2引数: write関数
const incrementAtom = atom(
  null,
  (get, set, step: number = 1) => {
    set(countAtom, get(countAtom) + step);
  }
);

// 使用例
const increment = useSetAtom(incrementAtom);
increment(5); // countAtomが5増加
```

### 4. 読み書き両用の派生atom

読み取りと書き込みの両方のロジックを持ちます。

```tsx
const countAtom = atom(0);

const doubleCountAtom = atom(
  (get) => get(countAtom) * 2,           // read: 2倍の値を返す
  (get, set, newValue: number) => {
    set(countAtom, newValue / 2);         // write: 半分の値を設定
  }
);
```

## 設計パターン

### カプセル化

内部atomを隠蔽し、派生atomのみを公開することで変更に強い設計が可能です。

```tsx
// 内部atom（非公開）
const _countAtom = atom(0);

// 公開API
export const countAtom = atom(
  (get) => get(_countAtom),
  (get, set, action: 'increment' | 'decrement' | 'reset') => {
    switch (action) {
      case 'increment':
        set(_countAtom, get(_countAtom) + 1);
        break;
      case 'decrement':
        set(_countAtom, get(_countAtom) - 1);
        break;
      case 'reset':
        set(_countAtom, 0);
        break;
    }
  }
);
```

### ユーティリティatom

jotaiには便利なユーティリティが用意されています。

```tsx
import { atomWithReset } from 'jotai/utils';

// リセット可能なatom
const countAtom = atomWithReset(0);

// 使用例
import { useResetAtom } from 'jotai/utils';
const resetCount = useResetAtom(countAtom);
resetCount(); // 初期値にリセット
```

## TypeScript型定義

```tsx
import { atom, Atom, WritableAtom, PrimitiveAtom } from 'jotai';

// 読み取り専用
const readOnlyAtom: Atom<number> = atom((get) => get(countAtom) * 2);

// 書き込み可能（引数なし）
const actionAtom: WritableAtom<null, [], void> = atom(null, (get, set) => {
  set(countAtom, 0);
});

// 書き込み可能（引数あり）
const setterAtom: WritableAtom<null, [number], void> = atom(
  null,
  (get, set, value: number) => {
    set(countAtom, value);
  }
);

// プリミティブ（読み書き両用の基本atom）
const primitiveAtom: PrimitiveAtom<number> = atom(0);
```

## 次のステップ

jotaiの基本を理解したら、次は[Suspenseの基本](./02_suspense_basics.md)でReactの非同期処理について学びましょう。
