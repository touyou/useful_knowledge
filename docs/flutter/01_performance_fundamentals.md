# Flutter Performance Fundamentals

## 概要

Flutterアプリケーションは基本的にパフォーマンスが高く設計されている。一般的な落とし穴を避けることで、優れたパフォーマンスを実現できる。

## フレームバジェット

### 基本ルール

```
60Hzディスプレイ: 1フレーム ≈ 16ms
├── ビルドフェーズ: ≈ 8ms
└── レンダリングフェーズ: ≈ 8ms

120Hzディスプレイ: 1フレーム ≈ 8ms
├── ビルドフェーズ: ≈ 4ms
└── レンダリングフェーズ: ≈ 4ms
```

### ジャンク（Jank）とは

フレームのレンダリングが通常よりも大幅に時間がかかり、フレームがドロップされると発生する。アニメーションが途切れたりカクついたりする現象。

## パフォーマンス計測

### ビルドモード

```bash
# デバッグモード（開発用、パフォーマンス計測には不適切）
flutter run

# プロファイルモード（パフォーマンス計測用）
flutter run --profile

# リリースモード（本番用）
flutter run --release
```

**重要**: デバッグモードはホットリロードのためのオーバーヘッドが含まれるため、パフォーマンス計測には必ずプロファイルモードを使用する。

### DevToolsによる計測

#### パフォーマンスビュー

主要な計測項目：
- `average_frame_build_time_millis`: 平均ビルド時間
- `worst_frame_build_time_millis`: 最悪ビルド時間
- `missed_frame_build_budget_count`: フレームバジェット超過回数
- `average_frame_rasterizer_time_millis`: 平均ラスタライズ時間

#### リビルド情報の表示

```dart
// DevToolsで「Show widget rebuild information」を有効化
// または以下のフラグを設定
import 'package:flutter/rendering.dart';

void enableDebugFlags() {
  debugRepaintRainbowEnabled = true;  // リペイント領域を可視化
}
```

### インテグレーションテストでの計測

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';

void main() {
  final binding = IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('performance test', (tester) async {
    await tester.pumpWidget(MyApp());

    // パフォーマンス計測を開始
    await binding.traceAction(() async {
      // テストするアクション
      await tester.tap(find.byType(ElevatedButton));
      await tester.pumpAndSettle();
    }, reportKey: 'button_tap_performance');
  });
}
```

計測結果の例（timeline_summary.json）：

```json
{
  "average_frame_build_time_millis": 4.26,
  "worst_frame_build_time_millis": 21.0,
  "missed_frame_build_budget_count": 2,
  "average_frame_rasterizer_time_millis": 5.52,
  "worst_frame_rasterizer_time_millis": 51.0,
  "frame_count": 54
}
```

## 過剰最適化の回避

### ベンチマークファースト原則

最適化の複雑さがパフォーマンス向上に見合わない場合が多い。以下の順序で対応する：

1. **計測**: 実際にパフォーマンス問題があるか確認
2. **特定**: ボトルネックの箇所を特定
3. **最適化**: 必要な箇所のみ最適化
4. **再計測**: 改善を確認

### 一般的な誤解

```dart
// ❌ 過剰最適化の例：些細な差のために複雑なコードを書く
final cachedValue = expensiveComputation();
Widget build(BuildContext context) {
  // 実際には計算コストが低い場合、キャッシュは不要
  return Text(cachedValue.toString());
}

// ✅ シンプルなコードで十分な場合が多い
Widget build(BuildContext context) {
  return Text(simpleComputation().toString());
}
```

## 16ms未満を目指す理由

フレームバジェット内に収まっていても、さらなる最適化には意義がある：

1. **バッテリー寿命**: CPU/GPU使用率の低下
2. **発熱抑制**: デバイスのサーマル管理
3. **低スペックデバイス対応**: より広いデバイスでの安定動作
4. **120fps対応準備**: 高リフレッシュレートデバイスへの対応

## 次のステップ

- [02_widget_optimization.md](./02_widget_optimization.md): ウィジェットのリビルド最適化
- [03_rendering_optimization.md](./03_rendering_optimization.md): レンダリング最適化

## 出典

- [Flutter Performance Overview](https://docs.flutter.dev/perf)
- [Flutter UI Performance](https://docs.flutter.dev/perf/ui-performance)
- [Flutter Build Modes](https://docs.flutter.dev/testing/build-modes#profile)
