# Riverpod Best Practices & Anti-Patterns

## 概要

RiverpodはFlutterの状態管理ライブラリ。正しく使用することで、保守性の高いコードとパフォーマンスの両方を実現できる。本ドキュメントでは公式ドキュメントに基づくベストプラクティスとアンチパターンを解説する。

---

## バッドノウハウ一覧

### 1. クラス内でのプロバイダー定義

#### アンチパターン

```dart
// ❌ Bad: クラス内でプロバイダーを定義
class Example {
  // サポートされていない操作。メモリリークと予期しない動作を引き起こす可能性
  final provider = Provider<String>((ref) => 'Hello world');
}
```

#### 改善方法

```dart
// ✅ Good: トップレベルのfinal変数として定義
final provider = Provider<String>((ref) => 'Hello world');

class Example {
  // プロバイダーは外部から参照
}
```

**理由**: プロバイダーはRiverpodのライフサイクル管理システムで適切に管理される必要がある。クラス内での動的生成はこの管理を回避し、メモリリークの原因となる。

---

### 2. プロバイダー内でのサイドエフェクト実行

#### アンチパターン

```dart
// ❌ Bad: プロバイダー初期化時にHTTP POSTを実行
final submitProvider = FutureProvider((ref) async {
  final formState = ref.watch(formState);

  // プロバイダーを「書き込み」操作に使用すべきではない
  return http.post('https://my-api.com', body: formState.toJson());
});
```

#### 改善方法

```dart
// ✅ Good: Notifierを使用してサイドエフェクトを管理
@riverpod
class FormController extends _$FormController {
  @override
  FormState build() => FormState.initial();

  Future<void> submit() async {
    state = state.copyWith(isSubmitting: true);
    try {
      await http.post('https://my-api.com', body: state.toJson());
      state = state.copyWith(isSubmitting: false, isSuccess: true);
    } catch (e) {
      state = state.copyWith(isSubmitting: false, error: e.toString());
    }
  }
}

// UIから呼び出し
ref.read(formControllerProvider.notifier).submit();
```

**理由**: プロバイダーは主に「読み取り」操作とデータアクセスを表す。「書き込み」操作はNotifierのメソッドとして実装すべき。

---

### 3. ref.readを使った「最適化」

#### アンチパターン

```dart
// ❌ Bad: ref.readで変更を無視して「最適化」
Consumer(
  builder: (context, ref, _) {
    // これは最適化ではない！UIが古い状態のままになる
    final tick = ref.read(tickProvider);
    return Text('Tick: $tick');
  },
);
```

#### 改善方法

```dart
// ✅ Good: ref.watchで変更を監視
Consumer(
  builder: (context, ref, _) {
    // パフォーマンスへの影響は通常無視できる
    final tick = ref.watch(tickProvider);
    return Text('Tick: $tick');
  },
);

// ✅ Good: 特定のプロパティのみ監視（本当に必要な場合）
Consumer(
  builder: (context, ref, _) {
    final isEven = ref.watch(
      tickProvider.select((tick) => tick.isEven),
    );
    return Text('Is Even: $isEven');
  },
);
```

**理由**: `ref.read`は変更を購読しないため、プロバイダーの状態が変わってもUIが更新されない。「最適化」のつもりが、実際には壊れたUIを生む。

---

### 4. 動的プロバイダーの渡し

#### アンチパターン

```dart
// ❌ Bad: プロバイダーを動的に渡す
class Example extends ConsumerWidget {
  Example({required this.provider});
  final Provider<int> provider;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // 静的解析ができない
    ref.watch(provider);
  }
}
```

#### 改善方法

```dart
// ✅ Good: 静的に既知のプロバイダーを使用
final myProvider = Provider<int>((ref) => 42);

class Example extends ConsumerWidget {
  const Example({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // riverpod_lintが問題を検出できる
    final value = ref.watch(myProvider);
    return Text('$value');
  }
}

// ✅ Good: ファミリープロバイダーでパラメータ化
@riverpod
int myValue(MyValueRef ref, {required String id}) {
  return computeValue(id);
}

class Example extends ConsumerWidget {
  const Example({required this.id, super.key});
  final String id;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final value = ref.watch(myValueProvider(id: id));
    return Text('$value');
  }
}
```

**理由**: 静的に既知のプロバイダーはriverpod_lintによる静的解析が可能。動的に渡されたプロバイダーは解析できず、バグの検出が困難。

---

### 5. 全プロバイダーの一括リセット

#### アンチパターン

```dart
// ❌ Bad: 全プロバイダーを一括リセット（そもそもAPIが存在しない）
// 仮にあったとしても使用すべきではない
void logoutUser() {
  // ref.resetAll(); // アンチパターン！
}
```

#### 改善方法

```dart
// ✅ Good: 依存関係ベースの自動リセット
@riverpod
User? currentUser(CurrentUserRef ref) {
  // 認証状態を監視
  final authState = ref.watch(authStateProvider);
  return authState.user;
}

@riverpod
UserProfile userProfile(UserProfileRef ref) {
  final user = ref.watch(currentUserProvider);
  if (user == null) {
    throw Exception('Not authenticated');
  }
  return fetchProfile(user.id);
}

// ログアウト時
void logout() {
  ref.read(authStateProvider.notifier).logout();
  // currentUserがnullになると、userProfileも自動的にリセット/無効化
}
```

**理由**: 全プロバイダーの一括リセットは、リセットすべきでないプロバイダーにも影響を与える。依存関係に基づいた選択的リセットが安全。

---

### 6. ChangeNotifierからの不適切な移行

#### アンチパターン

```dart
// ❌ Bad: 手動での状態管理（ChangeNotifierパターン）
class TodoChangeNotifier extends ChangeNotifier {
  List<Todo> todos = [];
  bool isLoading = false;
  bool hasError = false;

  TodoChangeNotifier() {
    _init();
  }

  Future<void> _init() async {
    isLoading = true;
    notifyListeners();
    try {
      todos = await fetchTodos();
    } catch (e) {
      hasError = true;
    } finally {
      isLoading = false;
      notifyListeners();
    }
  }
}
```

#### 改善方法

```dart
// ✅ Good: AsyncNotifierを使用
@riverpod
class TodoList extends _$TodoList {
  @override
  Future<List<Todo>> build() async {
    // ローディング・エラー状態は自動管理
    return await fetchTodos();
  }

  Future<void> addTodo(Todo todo) async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(() async {
      await saveTodo(todo);
      return [...(state.value ?? []), todo];
    });
  }
}

// UIでの使用
class TodoListWidget extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final todosAsync = ref.watch(todoListProvider);

    return todosAsync.when(
      loading: () => const CircularProgressIndicator(),
      error: (err, stack) => Text('Error: $err'),
      data: (todos) => ListView.builder(
        itemCount: todos.length,
        itemBuilder: (context, index) => TodoTile(todo: todos[index]),
      ),
    );
  }
}
```

**理由**: AsyncNotifierはローディング・エラー状態を自動管理し、状態の一貫性を保証。手動管理は煩雑でバグを生みやすい。

---

## リビルド最適化

### selectによる部分監視

デフォルトでは、`ref.watch`はオブジェクトのどのプロパティが変更されてもリビルドをトリガーする。

```dart
// ❌ 過剰なリビルド: ageが変わってもリビルドされる
Consumer(
  builder: (context, ref, _) {
    final user = ref.watch(userProvider);
    return Text('Name: ${user.name}');  // nameしか使っていない
  },
);

// ✅ 最適化: nameが変わった時だけリビルド
Consumer(
  builder: (context, ref, _) {
    final name = ref.watch(userProvider.select((user) => user.name));
    return Text('Name: $name');
  },
);
```

**注意**: ベンチマーク測定なしの過剰最適化は避ける。`select`の追加の複雑さがパフォーマンス向上に見合わない場合もある。

### Consumerの深い配置

```dart
// ❌ Bad: 高レベルでConsumer
return Consumer(
  builder: (context, ref, _) {
    final count = ref.watch(counterProvider);
    return Scaffold(
      appBar: AppBar(title: const Text('Counter')),
      body: Center(child: Text('Count: $count')),
    );
  },
);

// ✅ Good: 必要な箇所でのみConsumer
return Scaffold(
  appBar: AppBar(title: const Text('Counter')),
  body: Center(
    child: Consumer(
      builder: (context, ref, _) {
        final count = ref.watch(counterProvider);
        return Text('Count: $count');
      },
    ),
  ),
);
```

---

## プロバイダーのライフサイクル

### onDisposeの活用

```dart
@riverpod
Stream<int> ticker(TickerRef ref) {
  final controller = StreamController<int>();
  var count = 0;

  final timer = Timer.periodic(const Duration(seconds: 1), (_) {
    controller.add(count++);
  });

  // プロバイダーが破棄される時にクリーンアップ
  ref.onDispose(() {
    timer.cancel();
    controller.close();
  });

  return controller.stream;
}
```

### onCancelとautoDispose

```dart
@riverpod
class MyNotifier extends _$MyNotifier {
  @override
  int build() {
    ref.onCancel(() {
      print('誰もリッスンしていない');
    });

    ref.onDispose(() {
      print('.autoDisposeで破棄された');
    });

    return 0;
  }
}
```

---

## ref.watch vs ref.listen vs ref.read

| メソッド | 用途 | リビルド |
|---------|------|---------|
| `ref.watch` | 宣言的な状態監視（推奨） | あり |
| `ref.listen` | 手動のリスナー登録 | なし（コールバック） |
| `ref.read` | 一度だけ値を取得（イベントハンドラ内） | なし |

```dart
class MyWidget extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // ✅ ビルド内ではwatch
    final count = ref.watch(counterProvider);

    // ✅ サイドエフェクトにはlisten
    ref.listen(errorProvider, (previous, next) {
      if (next != null) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text(next)),
        );
      }
    });

    return ElevatedButton(
      // ✅ イベントハンドラ内ではread
      onPressed: () => ref.read(counterProvider.notifier).increment(),
      child: Text('Count: $count'),
    );
  }
}
```

---

## まとめ

| バッドノウハウ | 改善策 |
|---------------|--------|
| クラス内でプロバイダー定義 | トップレベルfinal変数として定義 |
| プロバイダー内でサイドエフェクト | Notifierのメソッドとして実装 |
| ref.readで「最適化」 | ref.watchまたはselectを使用 |
| 動的プロバイダーの渡し | 静的プロバイダーまたはファミリー |
| 全プロバイダー一括リセット | 依存関係ベースの自動リセット |
| ChangeNotifier風の手動状態管理 | AsyncNotifierで自動管理 |
| 高レベルでのConsumer配置 | 必要な箇所でのみConsumer |
| selectなしで全プロパティ監視 | 必要なプロパティのみselect（ベンチマーク後） |

## 出典

- [Riverpod Official Documentation](https://riverpod.dev/docs)
- [Riverpod DO/DON'T](https://riverpod.dev/docs/root/do_dont)
- [Riverpod FAQ](https://riverpod.dev/docs/root/faq)
- [Riverpod Migration from ChangeNotifier](https://riverpod.dev/docs/migration/from_change_notifier)
- [Riverpod How to reduce rebuilds](https://riverpod.dev/docs/how_to/select)
