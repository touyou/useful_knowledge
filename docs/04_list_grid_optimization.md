# List & Grid Optimization

## 概要

大量のアイテムを表示するリストやグリッドでは、遅延読み込み（lazy loading）が重要。画面に表示されている部分のみをビルドすることで、メモリ使用量とビルド時間を削減する。

## 遅延ビルダーの使用

### ListView

```dart
// ❌ Bad: 全アイテムを一度にビルド
ListView(
  children: List.generate(
    10000,
    (index) => ListTile(
      title: Text('Item $index'),
    ),
  ),
)

// ✅ Good: 遅延ビルダーを使用
ListView.builder(
  itemCount: 10000,
  itemBuilder: (context, index) {
    return ListTile(
      title: Text('Item $index'),
    );
  },
)
```

### GridView

```dart
// ❌ Bad: 全アイテムを一度にビルド
GridView.count(
  crossAxisCount: 2,
  children: items.map((item) => ItemCard(item: item)).toList(),
)

// ✅ Good: 遅延ビルダーを使用
GridView.builder(
  gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount: 2,
  ),
  itemCount: items.length,
  itemBuilder: (context, index) {
    return ItemCard(item: items[index]);
  },
)
```

### CustomScrollView with Slivers

```dart
// ✅ Good: 複数のスクロール可能要素を組み合わせる場合
CustomScrollView(
  slivers: [
    SliverAppBar(
      title: const Text('My App'),
      floating: true,
    ),
    SliverList.builder(
      itemCount: items.length,
      itemBuilder: (context, index) {
        return ListTile(title: Text(items[index]));
      },
    ),
    SliverGrid.builder(
      gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
        crossAxisCount: 3,
      ),
      itemCount: gridItems.length,
      itemBuilder: (context, index) {
        return GridTile(child: Image.network(gridItems[index]));
      },
    ),
  ],
)
```

## Intrinsic操作の回避

### 問題点

Intrinsic操作（`IntrinsicWidth`、`IntrinsicHeight`等）は、全ての子要素のサイズを問い合わせる追加のレイアウトパスを実行する。大きなグリッドでは特に高コスト。

### 発生する状況

- 全てのセルを均一なサイズにしたい場合
- 最大の子要素に合わせてサイズを決定する場合

### デバッグ方法

DevToolsのPerformance viewで「Track layouts」を有効にし、`'$runtimeType intrinsics'`というラベルのタイムラインイベントを確認。

### 解決策

```dart
// ❌ Bad: IntrinsicHeightでセルの高さを揃える
GridView.builder(
  gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount: 2,
  ),
  itemBuilder: (context, index) {
    return IntrinsicHeight(
      child: MyVariableHeightWidget(data: items[index]),
    );
  },
)

// ✅ Good: 固定のアスペクト比を使用
GridView.builder(
  gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount: 2,
    childAspectRatio: 1.0,  // 正方形のセル
  ),
  itemBuilder: (context, index) {
    return MyWidget(data: items[index]);
  },
)

// ✅ Good: 固定の高さを使用
GridView.builder(
  gridDelegate: const SliverGridDelegateWithMaxCrossAxisExtent(
    maxCrossAxisExtent: 200,
    mainAxisExtent: 150,  // 固定の高さ
  ),
  itemBuilder: (context, index) {
    return MyWidget(data: items[index]);
  },
)
```

## ページネーションとインクリメンタルロード

### 基本パターン

```dart
class PaginatedListWidget extends StatefulWidget {
  const PaginatedListWidget({super.key});

  @override
  State<PaginatedListWidget> createState() => _PaginatedListWidgetState();
}

class _PaginatedListWidgetState extends State<PaginatedListWidget> {
  final List<Item> _items = [];
  bool _isLoading = false;
  bool _hasMore = true;
  final ScrollController _scrollController = ScrollController();

  @override
  void initState() {
    super.initState();
    _loadMore();
    _scrollController.addListener(_onScroll);
  }

  @override
  void dispose() {
    _scrollController.dispose();
    super.dispose();
  }

  void _onScroll() {
    if (_scrollController.position.pixels >=
        _scrollController.position.maxScrollExtent - 200) {
      _loadMore();
    }
  }

  Future<void> _loadMore() async {
    if (_isLoading || !_hasMore) return;

    setState(() => _isLoading = true);

    final newItems = await fetchItems(offset: _items.length, limit: 20);

    setState(() {
      _items.addAll(newItems);
      _isLoading = false;
      _hasMore = newItems.length == 20;
    });
  }

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      controller: _scrollController,
      itemCount: _items.length + (_isLoading ? 1 : 0),
      itemBuilder: (context, index) {
        if (index == _items.length) {
          return const Center(child: CircularProgressIndicator());
        }
        return ItemTile(item: _items[index]);
      },
    );
  }
}
```

## ローディング状態の表示

```dart
class DataListWidget extends StatefulWidget {
  const DataListWidget({super.key});

  @override
  State<DataListWidget> createState() => _DataListWidgetState();
}

class _DataListWidgetState extends State<DataListWidget> {
  List<Map<String, Object?>> _data = [];
  bool _isLoading = true;

  @override
  void initState() {
    super.initState();
    _loadData();
  }

  Future<void> _loadData() async {
    final response = await http.get(
      Uri.parse('https://api.example.com/items'),
    );

    setState(() {
      _data = (jsonDecode(response.body) as List).cast<Map<String, Object?>>();
      _isLoading = false;
    });
  }

  @override
  Widget build(BuildContext context) {
    if (_isLoading) {
      return const Center(child: CircularProgressIndicator());
    }

    return ListView.builder(
      itemCount: _data.length,
      itemBuilder: (context, index) {
        return ListTile(
          title: Text(_data[index]['title'] as String),
        );
      },
    );
  }
}
```

## アイテムのキャッシュ

### AutomaticKeepAliveの使用

スクロールで画面外に出たアイテムの状態を保持する場合：

```dart
class KeepAliveListItem extends StatefulWidget {
  final int index;

  const KeepAliveListItem({required this.index, super.key});

  @override
  State<KeepAliveListItem> createState() => _KeepAliveListItemState();
}

class _KeepAliveListItemState extends State<KeepAliveListItem>
    with AutomaticKeepAliveClientMixin {
  @override
  bool get wantKeepAlive => true;  // 状態を保持

  @override
  Widget build(BuildContext context) {
    super.build(context);  // 必須
    return ExpensiveWidget(index: widget.index);
  }
}
```

**注意**: `AutomaticKeepAlive`はメモリ使用量を増加させる。必要な場合のみ使用。

## SliverAppBarとの組み合わせ

```dart
// ✅ Good: NestedScrollViewで複雑なスクロール
NestedScrollView(
  headerSliverBuilder: (context, innerBoxIsScrolled) {
    return [
      SliverAppBar(
        expandedHeight: 200,
        floating: false,
        pinned: true,
        flexibleSpace: FlexibleSpaceBar(
          title: const Text('My List'),
          background: Image.network(
            'https://example.com/header.jpg',
            fit: BoxFit.cover,
          ),
        ),
      ),
    ];
  },
  body: ListView.builder(
    itemCount: items.length,
    itemBuilder: (context, index) {
      return ListTile(title: Text(items[index]));
    },
  ),
)
```

## パフォーマンスチェックリスト

| チェック項目 | 対応 |
|-------------|------|
| 大量アイテムを一度にビルドしている | `.builder()`を使用 |
| IntrinsicWidth/Heightを使用している | 固定サイズまたはアスペクト比を使用 |
| スクロール時にフレームドロップがある | アイテムの複雑さを軽減 |
| 無限スクロールでメモリが増加し続ける | ページネーションを実装 |
| アイテムの状態が失われる | AutomaticKeepAliveを検討（慎重に） |

## まとめ

| テクニック | 効果 |
|-----------|------|
| ListView.builder | 遅延ビルド、メモリ削減 |
| GridView.builder | 遅延ビルド、メモリ削減 |
| 固定サイズ指定 | Intrinsic操作の回避 |
| ページネーション | メモリ使用量の制限 |
| Sliver系ウィジェット | 柔軟なスクロール構成 |

## 出典

- [Flutter Performance Best Practices](https://docs.flutter.dev/perf/best-practices)
- [Flutter Cookbook - Long Lists](https://docs.flutter.dev/cookbook/lists/long-lists)
- [Flutter for UIKit Developers](https://docs.flutter.dev/get-started/flutter-for/uikit-devs)
