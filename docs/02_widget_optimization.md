# Widget Optimization

## 概要

Flutterのウィジェットリビルドは高速だが、不要なリビルドを減らすことでパフォーマンスを向上できる。

## build()メソッドの最適化

### 原則

`build()`メソッドは頻繁に呼び出される。祖先ウィジェットがリビルドされるたびに呼び出される可能性がある。

### ベストプラクティス

#### 1. build()内での繰り返し処理を避ける

```dart
// ❌ Bad: build()内で毎回計算
class MyWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final items = List.generate(1000, (i) => expensiveComputation(i));
    return ListView(children: items.map((e) => Text(e)).toList());
  }
}

// ✅ Good: 必要な時だけ計算
class MyWidget extends StatefulWidget {
  @override
  State<MyWidget> createState() => _MyWidgetState();
}

class _MyWidgetState extends State<MyWidget> {
  late final List<String> items;

  @override
  void initState() {
    super.initState();
    items = List.generate(1000, (i) => expensiveComputation(i));
  }

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: items.length,
      itemBuilder: (context, index) => Text(items[index]),
    );
  }
}
```

#### 2. 大きなウィジェットを分割する

```dart
// ❌ Bad: 巨大な単一ウィジェット
class HugeWidget extends StatefulWidget {
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // 数百行のウィジェットツリー
        HeaderSection(),
        BodySection(),
        FooterSection(),
      ],
    );
  }
}

// ✅ Good: 責務ごとに分割
class OptimizedWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        const HeaderSection(),  // constで不変を明示
        BodySection(),          // 状態を持つ部分
        const FooterSection(),  // constで不変を明示
      ],
    );
  }
}
```

## constコンストラクタの活用

### 効果

`const`コンストラクタはウィジェットの不変性を示し、Flutterがリビルドをスキップできる。

```dart
// ✅ Good: constを積極的に使用
class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Scaffold(
        appBar: AppBar(
          title: const Text('My App'),  // const
        ),
        body: const Center(             // const
          child: Text(
            'Hello',
            style: TextStyle(fontSize: 24),  // constコンテキスト内
          ),
        ),
      ),
    );
  }
}
```

### constが使えない場合

```dart
// 動的な値を含む場合はconstは使えない
Text(widget.dynamicTitle)  // OK、constなし
Text('Static Title')       // constを付けるべき → const Text('Static Title')
```

## StatelessWidget vs ヘルパー関数

### ヘルパー関数の問題

```dart
// ❌ Bad: ヘルパー関数
class MyWidget extends StatelessWidget {
  Widget _buildHeader() {
    return Container(
      child: Text('Header'),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        _buildHeader(),  // 毎回新しいインスタンスが生成される
        _buildBody(),
      ],
    );
  }
}

// ✅ Good: 別のStatelessWidgetに分割
class HeaderWidget extends StatelessWidget {
  const HeaderWidget({super.key});

  @override
  Widget build(BuildContext context) {
    return Container(
      child: const Text('Header'),
    );
  }
}

class MyWidget extends StatelessWidget {
  const MyWidget({super.key});

  @override
  Widget build(BuildContext context) {
    return const Column(
      children: [
        HeaderWidget(),  // constで再利用可能
        BodyWidget(),
      ],
    );
  }
}
```

## setStateの局所化

### 原則

`setState()`は影響を受けるサブツリーが最小になるウィジェットで呼び出す。

```dart
// ❌ Bad: ルートレベルでsetState
class BadExample extends StatefulWidget {
  @override
  State<BadExample> createState() => _BadExampleState();
}

class _BadExampleState extends State<BadExample> {
  int _counter = 0;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(
        children: [
          const ExpensiveWidget(),      // これもリビルドされる！
          const AnotherExpensiveWidget(), // これもリビルドされる！
          Text('Count: $_counter'),
        ],
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => setState(() => _counter++),
        child: const Icon(Icons.add),
      ),
    );
  }
}

// ✅ Good: カウンター部分を分離
class GoodExample extends StatelessWidget {
  const GoodExample({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(
        children: [
          const ExpensiveWidget(),        // リビルドされない
          const AnotherExpensiveWidget(), // リビルドされない
          const CounterWidget(),          // この中でsetState
        ],
      ),
    );
  }
}

class CounterWidget extends StatefulWidget {
  const CounterWidget({super.key});

  @override
  State<CounterWidget> createState() => _CounterWidgetState();
}

class _CounterWidgetState extends State<CounterWidget> {
  int _counter = 0;

  @override
  Widget build(BuildContext context) {
    return Row(
      children: [
        Text('Count: $_counter'),
        IconButton(
          onPressed: () => setState(() => _counter++),
          icon: const Icon(Icons.add),
        ),
      ],
    );
  }
}
```

## Consumerウィジェットの配置（Provider/Riverpod）

### 原則

状態を監視するウィジェットは、できるだけ深い位置に配置する。

```dart
// ❌ Bad: 高レベルでConsumer
return Consumer<CartModel>(
  builder: (context, cart, child) {
    return HumongousWidget(
      child: AnotherMonstrousWidget(
        child: Text('Total: ${cart.totalPrice}'),
      ),
    );
  },
);

// ✅ Good: 必要な箇所でのみConsumer
return HumongousWidget(
  child: AnotherMonstrousWidget(
    child: Consumer<CartModel>(
      builder: (context, cart, child) {
        return Text('Total: ${cart.totalPrice}');
      },
    ),
  ),
);
```

### childパラメータの活用

```dart
// ✅ Good: 高コストウィジェットをchildで渡す
return Consumer<CartModel>(
  builder: (context, cart, child) => Stack(
    children: [
      child!,  // リビルドされない
      Text('Total: ${cart.totalPrice}'),
    ],
  ),
  child: const SomeExpensiveWidget(),  // 一度だけビルド
);
```

## operator==のオーバーライドを避ける

### 問題点

Widget上での`operator ==`のオーバーライドはO(N²)の動作を引き起こす。

```dart
// ❌ Don't: Widget上でoperator==をオーバーライド
class MyWidget extends StatelessWidget {
  final String value;

  const MyWidget({required this.value, super.key});

  @override
  bool operator ==(Object other) =>
      identical(this, other) ||
      other is MyWidget && runtimeType == other.runtimeType && value == other.value;

  @override
  int get hashCode => value.hashCode;

  @override
  Widget build(BuildContext context) => Text(value);
}

// ✅ Do: constコンストラクタに依存
class MyWidget extends StatelessWidget {
  final String value;

  const MyWidget({required this.value, super.key});

  @override
  Widget build(BuildContext context) => Text(value);
}

// 使用時にconstを使う
const MyWidget(value: 'fixed')
```

## AnimatedBuilderの最適化

```dart
// ❌ Bad: アニメーションに依存しないウィジェットもbuilder内に
AnimatedBuilder(
  animation: _controller,
  builder: (context, child) {
    return Column(
      children: [
        Transform.rotate(
          angle: _controller.value * 2 * pi,
          child: const Icon(Icons.refresh),
        ),
        const ExpensiveWidget(),  // 毎フレームリビルドされる！
      ],
    );
  },
);

// ✅ Good: childパラメータを活用
AnimatedBuilder(
  animation: _controller,
  builder: (context, child) {
    return Column(
      children: [
        Transform.rotate(
          angle: _controller.value * 2 * pi,
          child: const Icon(Icons.refresh),
        ),
        child!,  // リビルドされない
      ],
    );
  },
  child: const ExpensiveWidget(),  // 一度だけビルド
);
```

## StringBufferの使用

```dart
// ❌ Bad: 文字列結合の連続
String buildText(List<String> items) {
  String result = '';
  for (final item in items) {
    result += '$item\n';  // 毎回新しいStringオブジェクトを生成
  }
  return result;
}

// ✅ Good: StringBufferを使用
String buildText(List<String> items) {
  final buffer = StringBuffer();
  for (final item in items) {
    buffer.writeln(item);  // 効率的に追加
  }
  return buffer.toString();  // 最後に一度だけ文字列を生成
}
```

## まとめ

| テクニック | 効果 |
|-----------|------|
| constコンストラクタ | リビルドスキップ |
| ウィジェット分割 | 局所的リビルド |
| setState局所化 | リビルド範囲縮小 |
| Consumer深配置 | リビルド範囲縮小 |
| childパラメータ | 高コストウィジェット保護 |
| StatelessWidget化 | 再利用・const化 |

## 出典

- [Flutter Performance Best Practices](https://docs.flutter.dev/perf/best-practices)
- [Flutter State Management](https://docs.flutter.dev/data-and-backend/state-mgmt/simple)
