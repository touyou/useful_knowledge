# Swift Actor ナレッジ - 目次

Swift Concurrencyにおけるアクターモデルの概念と実践的な使い方をまとめたナレッジです。Swift 5.5で導入され、Swift 6/6.2で大きく進化したアクター分離（Actor Isolation）を中心に解説します。

## 対象読者

- Swift Concurrencyの基本（async/await）を理解している方
- データ競合を防ぎたいiOS/macOS開発者
- SwiftUIアプリでのアクター活用を学びたい方

## 目次

### 基礎編

1. [Actorの基本](01_actor_fundamentals.md)
   - Actorとは何か
   - データ競合（Data Race）の問題
   - Actor分離の仕組み
   - 基本的な使い方

2. [MainActor](02_main_actor.md)
   - MainActorの役割
   - UIスレッドとの関係
   - 使用パターンとベストプラクティス
   - Swift 6.2のデフォルトMainActor分離

3. [GlobalActor](03_global_actor.md)
   - GlobalActorプロトコル
   - カスタムGlobalActorの作成
   - 使用シナリオと制約

### 応用編

4. [Sendableとアクター境界](04_sendable_isolation.md)
   - Sendableプロトコルの概念
   - アクター境界を越えるデータ
   - nonisolatedとisolated
   - Swift 6.2の`nonisolated(nonsending)`と`@concurrent`

5. [SwiftUIとの連携](05_swiftui_integration.md)
   - ViewプロトコルとMainActor
   - @Observableマクロとアクター分離
   - ObservableObjectからの移行
   - ベストプラクティス

## Swift Evolution プロポーザル

このナレッジで参照している主要なSwift Evolutionプロポーザル：

| SE番号 | タイトル | 導入バージョン |
|--------|----------|----------------|
| [SE-0306](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md) | Actors | Swift 5.5 |
| [SE-0302](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md) | Sendable and @Sendable closures | Swift 5.5 |
| [SE-0313](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0313-actor-isolation-control.md) | Actor isolation control | Swift 5.5 |
| [SE-0316](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0316-global-actors.md) | Global actors | Swift 5.5 |
| [SE-0401](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0401-remove-property-wrapper-isolation.md) | Remove Actor Isolation Inference from Property Wrappers | Swift 6.0 |
| [SE-0434](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0434-global-actor-isolated-types-usability.md) | Usability of global-actor-isolated types | Swift 6.0 |
| [SE-0461](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md) | Async function isolation inheritance | Swift 6.2 |

## 参考資料

- [The Swift Programming Language - Concurrency](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)
- [Apple Developer Documentation - GlobalActor](https://developer.apple.com/documentation/swift/globalactor)
- [Swift Concurrency Adoption Guidelines](https://www.swift.org/documentation/server/guides/libraries/concurrency-adoption-guidelines/)
- [SwiftLee - Actor関連記事](https://www.avanderlee.com/swift/)
- [Fat Bob Man - SwiftUI Views and MainActor](https://fatbobman.com/en/posts/swiftui-views-and-mainactor/)
