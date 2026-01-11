# Flutter Performance Optimization Guide

このドキュメントは、Flutter公式ドキュメント（2025年11月時点）およびRiverpod公式ドキュメントに基づいて作成されたパフォーマンス最適化ガイドです。

## ドキュメント構成

| ファイル | 内容 | 対象読者 |
|---------|------|---------|
| [01_performance_fundamentals.md](./01_performance_fundamentals.md) | パフォーマンスの基礎概念、フレームバジェット、計測方法 | 全員 |
| [02_widget_optimization.md](./02_widget_optimization.md) | ウィジェットのリビルド最適化、const活用、状態管理 | 全員 |
| [03_rendering_optimization.md](./03_rendering_optimization.md) | レンダリング最適化、saveLayer回避、RepaintBoundary | 中級者以上 |
| [04_list_grid_optimization.md](./04_list_grid_optimization.md) | リスト・グリッドの遅延読み込み、intrinsic操作の回避 | 全員 |
| [05_riverpod_best_practices.md](./05_riverpod_best_practices.md) | Riverpodのベストプラクティスとアンチパターン | Riverpod使用者 |

## クイックリファレンス

### 最重要ルール

1. **16msルール**: 60Hzディスプレイでは1フレーム約16ms（ビルド8ms + レンダリング8ms）
2. **const活用**: 可能な限り`const`コンストラクタを使用
3. **遅延ビルド**: `ListView.builder`等のlazyビルダーを使用
4. **状態の局所化**: `setState()`は最小のサブツリーで呼び出し
5. **過剰最適化の回避**: ベンチマーク測定なしの最適化は避ける

### よくあるアンチパターン

| アンチパターン | 改善方法 |
|--------------|---------|
| `Opacity`ウィジェットの多用 | `AnimatedOpacity`または`Color.withOpacity()`を使用 |
| `ListView(children: [...])` | `ListView.builder()`を使用 |
| Widget上で`operator ==`のオーバーライド | constウィジェットに依存 |
| AnimatedBuilder内で高コストウィジェットをビルド | `child`パラメータを活用 |

## 出典

- [Flutter Performance Best Practices](https://docs.flutter.dev/perf/best-practices) - 2025年11月更新
- [Flutter Rendering Performance](https://docs.flutter.dev/perf/rendering-performance)
- [Riverpod Official Documentation](https://riverpod.dev/docs)
- Flutter 3.38.1 / Dart 3.5 時点の情報
