# React × jotai ナレッジ

本ドキュメントは [uhyo氏「jotaiによるReact再入門」](https://zenn.dev/uhyo/books/learn-react-with-jotai) をベースに、React公式ドキュメントの補足を加えてまとめたものです。

## 対象読者

- ReactでのSuspense以後の設計に不慣れな方
- jotaiを使った状態管理と非同期処理を学びたい方
- モダンなReactパターンを身につけたい方

## 目次

1. [jotaiの基本](./01_jotai_basics.md) - atom、useAtom、派生atomの基礎
2. [Suspenseの基本](./02_suspense_basics.md) - 非同期処理と宣言的UI
3. [SuspenseとjotaiToの組み合わせ](./03_suspense_plus_jotai.md) - 実践的な非同期処理パターン
4. [AbortSignalを使った非同期処理の中断](./04_abort_signal.md) - 不要なリクエストのキャンセル
5. [派生atomの再読み込みとUIバージョニング](./05_reload_and_versioning.md) - 再取得パターン
6. [トランジションの基本](./06_transition_basics.md) - スムーズなUI遷移
7. [トランジションの応用](./07_transition_advanced.md) - 世界の分岐とオプトアウト
8. [jotai-eagerを使う](./08_jotai_eager.md) - 非同期処理の最適化
9. [エラーハンドリング](./09_error_handling.md) - Error Boundaryとリトライ

## 核心となる考え方

### Suspense以後の設計思想

```
従来: ボタン押下 → 非同期処理 → ステート更新 → UI更新
新方式: ボタン押下 → ステート更新 → 非同期処理（サスペンド） → UI更新
```

**Promiseそのものをステートとして保持する**発想により、Reactの責務（ステート変化に対応したUI更新）に非同期処理が統合されます。

### 3つの柱

| 機能 | 役割 |
|------|------|
| **Suspense** | 非同期処理中のフォールバックUI表示 |
| **Transition** | UIのちらつき防止と操作フィードバック |
| **Error Boundary** | エラー発生時のフォールバックUI表示 |

## 参考リンク

- [jotai 公式ドキュメント](https://jotai.org/)
- [React 公式ドキュメント - Suspense](https://react.dev/reference/react/Suspense)
- [React 公式ドキュメント - useTransition](https://react.dev/reference/react/useTransition)
- [原著: jotaiによるReact再入門](https://zenn.dev/uhyo/books/learn-react-with-jotai)
