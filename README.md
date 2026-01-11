# Useful Knowledge

フロントエンド開発に関する実践的なナレッジを集めたリポジトリです。

## 構成

```
docs/
├── flutter/     # Flutter パフォーマンス最適化ガイド
└── react/       # React × jotai ナレッジ
```

## Flutter パフォーマンス最適化

Flutterアプリケーションのパフォーマンス最適化に関する包括的なガイドです。

### 目次

- [00. 目次](docs/flutter/00_index.md)
- [01. パフォーマンスの基礎](docs/flutter/01_performance_fundamentals.md)
- [02. Widget最適化](docs/flutter/02_widget_optimization.md)
- [03. レンダリング最適化](docs/flutter/03_rendering_optimization.md)
- [04. リスト・グリッドの最適化](docs/flutter/04_list_grid_optimization.md)
- [05. Riverpodベストプラクティス](docs/flutter/05_riverpod_best_practices.md)

## React × jotai ナレッジ

[uhyo氏「jotaiによるReact再入門」](https://zenn.dev/uhyo/books/learn-react-with-jotai) をベースに、React公式ドキュメントの補足を加えてまとめたものです。Suspense以後のReact設計を学びます。

### 目次

- [00. 目次](docs/react/00_index.md)
- [01. jotaiの基本](docs/react/01_jotai_basics.md)
- [02. Suspenseの基本](docs/react/02_suspense_basics.md)
- [03. SuspenseとjotaiToの組み合わせ](docs/react/03_suspense_plus_jotai.md)
- [04. AbortSignalを使った非同期処理の中断](docs/react/04_abort_signal.md)
- [05. 派生atomの再読み込みとUIバージョニング](docs/react/05_reload_and_versioning.md)
- [06. トランジションの基本](docs/react/06_transition_basics.md)
- [07. トランジションの応用](docs/react/07_transition_advanced.md)
- [08. jotai-eagerを使う](docs/react/08_jotai_eager.md)
- [09. エラーハンドリング](docs/react/09_error_handling.md)

## ライセンス

このリポジトリの内容は個人的な学習目的でまとめたものです。

### 参考・引用元

- **Flutter**: Flutter公式ドキュメント、各種技術ブログ
- **React**: [jotaiによるReact再入門](https://zenn.dev/uhyo/books/learn-react-with-jotai) by uhyo、React公式ドキュメント
