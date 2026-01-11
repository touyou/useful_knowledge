# Rendering Optimization

## 概要

レンダリングパフォーマンスの最適化は、GPUスレッドでの処理を効率化することで達成される。特に`saveLayer()`、不透明度、クリッピングの最適化が重要。

## saveLayer()の理解と回避

### saveLayer()が高コストな理由

1. **オフスクリーンバッファの割り当て**: GPUメモリを追加で使用
2. **レンダーターゲットの切り替え**: モバイルGPUでは特に高コスト
3. **合成処理**: 追加のブレンド操作が必要

### saveLayer()をトリガーするウィジェット

| ウィジェット | 条件 |
|-------------|------|
| `ShaderMask` | 常に |
| `ColorFilter` | 常に |
| `Chip` | `disabledColorAlpha != 0xff`の場合 |
| `Text` | `overflowShader`使用時 |
| `Opacity` | 複数の子要素を持つ場合 |

### デバッグ方法

```dart
import 'package:flutter/rendering.dart';

void debugSaveLayer() {
  // オフスクリーンレイヤーをチェッカーボードで可視化
  debugCheckLayerBounds = true;
}
```

DevToolsのPerformance viewで「Track layers」を有効にして、`saveLayer`タイムラインイベントを確認。

## Opacityウィジェットの最適化

### 問題点

`Opacity`ウィジェットは内部的に`saveLayer()`を呼び出す可能性があり、高コスト。

### 改善方法

```dart
// ❌ Bad: Opacityウィジェット
Opacity(
  opacity: 0.5,
  child: Container(
    color: Colors.blue,
    width: 100,
    height: 100,
  ),
)

// ✅ Good: 色に透明度を直接指定
Container(
  color: Colors.blue.withOpacity(0.5),
  width: 100,
  height: 100,
)

// ✅ Good: アニメーションにはAnimatedOpacityを使用
AnimatedOpacity(
  opacity: _isVisible ? 1.0 : 0.0,
  duration: const Duration(milliseconds: 300),
  child: MyWidget(),
)

// ✅ Good: 画像のフェードにはFadeInImageを使用
FadeInImage.assetNetwork(
  placeholder: 'assets/placeholder.png',
  image: 'https://example.com/image.png',
)
```

### 複数要素の透明度

```dart
// ❌ Bad: 複数の半透明要素が重なる
Stack(
  children: [
    Opacity(opacity: 0.5, child: Shape1()),
    Opacity(opacity: 0.5, child: Shape2()),
  ],
)

// ✅ Good: CustomPaintで事前に合成
CustomPaint(
  painter: PrecalculatedShapePainter(),
)
```

## クリッピングの最適化

### 問題点

クリッピングはアニメーション中に特に高コスト。

### 改善方法

```dart
// ❌ Bad: アニメーション中のクリッピング
AnimatedBuilder(
  animation: _controller,
  builder: (context, child) {
    return ClipRRect(
      borderRadius: BorderRadius.circular(20),
      child: Transform.scale(
        scale: _controller.value,
        child: Image.asset('assets/image.png'),
      ),
    );
  },
)

// ✅ Good: 事前にクリッピングした画像を使用
AnimatedBuilder(
  animation: _controller,
  builder: (context, child) {
    return Transform.scale(
      scale: _controller.value,
      child: child,  // 事前にクリッピング済み
    );
  },
  child: ClipRRect(
    borderRadius: BorderRadius.circular(20),
    child: Image.asset('assets/image.png'),
  ),
)

// ✅ Good: 角丸にはborderRadiusを使用（クリッピングなし）
Container(
  decoration: BoxDecoration(
    borderRadius: BorderRadius.circular(12),
    color: Colors.blue,
  ),
  child: MyContent(),
)

// ✅ Good: 不透明な角で覆う
Stack(
  children: [
    MySquareWidget(),
    Positioned.fill(
      child: CustomPaint(
        painter: CornerOverlayPainter(),  // 角を不透明に塗りつぶし
      ),
    ),
  ],
)
```

## RepaintBoundaryの活用

### 問題

アニメーションウィジェットが親全体の再描画を引き起こす。

```dart
// ❌ Bad: CircularProgressIndicatorが全体を再描画
class EverythingRepaintsPage extends StatelessWidget {
  const EverythingRepaintsPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Example')),
      body: const Center(
        child: CircularProgressIndicator(),  // 全体が再描画される
      ),
    );
  }
}
```

### 解決策

```dart
// ✅ Good: RepaintBoundaryで再描画範囲を限定
class OptimizedPage extends StatelessWidget {
  const OptimizedPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Example')),
      body: const Center(
        child: RepaintBoundary(
          child: CircularProgressIndicator(),  // この部分のみ再描画
        ),
      ),
    );
  }
}
```

### デバッグ

```dart
import 'package:flutter/rendering.dart';

void highlightRepaints() {
  // 再描画される領域を虹色の枠で表示
  debugRepaintRainbowEnabled = true;
}
```

## GPUグラフの分析

### UIグラフ vs GPUグラフ

| 状況 | UIグラフ | GPUグラフ | 原因 |
|------|---------|----------|------|
| ビルドが遅い | 赤 | 正常 | ウィジェット構築が遅い |
| レンダリングが遅い | 正常 | 赤 | GPU処理が遅い |
| 両方遅い | 赤 | 赤 | 複合的な問題 |

### GPUグラフが赤い場合の確認事項

1. **不要な`saveLayer()`**: ShaderMask、ColorFilter等
2. **複雑なクリッピング**: 特にアニメーション中
3. **過剰な透明度**: 重なり合う半透明要素
4. **複雑なシャドウ**: 特定の状況下

## アニメーションのスローダウン

```dart
// DevToolsの「Slow Animations」ボタンでアニメーションを5倍遅く

// プログラムでの制御
import 'package:flutter/scheduler.dart';

void slowDownAnimations() {
  timeDilation = 5.0;  // 5倍スロー
}

void normalSpeed() {
  timeDilation = 1.0;  // 通常速度
}
```

## パフォーマンス問題の診断フロー

```
GPUグラフが赤い
    │
    ├─ 最初のフレームだけ遅い？
    │   └─ シェーダーコンパイルの可能性
    │       → ウォームアップを検討
    │
    ├─ アニメーション全体が遅い？
    │   ├─ クリッピングが原因？
    │   │   └─ 代替描画方法を検討
    │   │
    │   └─ 透明度が原因？
    │       └─ AnimatedOpacity、色の透明度を使用
    │
    └─ 静的なシーンのフェード・回転が遅い？
        └─ RepaintBoundaryを追加
```

## まとめ

| 問題 | 解決策 |
|------|--------|
| saveLayer()の多用 | ShaderMask等を避ける、事前合成 |
| Opacityウィジェット | AnimatedOpacity、Color.withOpacity() |
| アニメーション中のクリッピング | 事前クリッピング、borderRadius |
| 広範囲の再描画 | RepaintBoundary |
| 複雑な透明度の重なり | CustomPaintで事前合成 |

## 出典

- [Flutter Rendering Performance](https://docs.flutter.dev/perf/rendering-performance)
- [Flutter Performance Best Practices](https://docs.flutter.dev/perf/best-practices)
- [Flutter DevTools Inspector](https://docs.flutter.dev/tools/devtools/inspector)
