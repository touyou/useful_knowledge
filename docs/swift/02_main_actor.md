# MainActor

## MainActorとは

`@MainActor`は、メインスレッドでの実行を保証する特別なグローバルアクターです。UIKitやSwiftUIではUI更新は必ずメインスレッドで行う必要があり、`@MainActor`はこれを型システムレベルで保証します。

```swift
@MainActor
class ViewController: UIViewController {
    // このクラスのすべてのメソッドとプロパティは
    // メインスレッドで実行されることが保証される
}
```

### MainActorの定義（Swift標準ライブラリ）

```swift
@globalActor
public actor MainActor: GlobalActor {
    public static let shared = MainActor(...)
}
```

## UIスレッドとの関係

### なぜMainActorが必要か

UIフレームワークはスレッドセーフではありません。UIの更新を非メインスレッドから行うと、クラッシュや予期しない動作が発生します。

```swift
// 従来の方法（DispatchQueue）
func fetchData() {
    URLSession.shared.dataTask(with: url) { data, _, _ in
        // バックグラウンドスレッドで実行される
        DispatchQueue.main.async {
            // メインスレッドに戻す（忘れがち）
            self.label.text = String(data: data!, encoding: .utf8)
        }
    }.resume()
}

// MainActorを使った方法
@MainActor
func updateUI(with data: Data) {
    // コンパイラがメインスレッドでの実行を保証
    self.label.text = String(data: data, encoding: .utf8)
}
```

### UIKitとMainActor

Swift 5.5以降、UIKitのクラスは`@MainActor`でマークされています：

```swift
// UIKitの定義（抜粋）
@MainActor
open class UIView: UIResponder { ... }

@MainActor
open class UIViewController: UIResponder { ... }

@MainActor
open class UIWindow: UIView { ... }
```

これにより、`UIViewController`などを継承したクラスは自動的に`@MainActor`に分離されます。

## 使用パターン

### 1. クラス全体に適用

```swift
@MainActor
class ProfileViewModel {
    var name: String = ""
    var email: String = ""
    var isLoading: Bool = false

    func loadProfile() async {
        isLoading = true
        defer { isLoading = false }

        let profile = await fetchProfile()
        name = profile.name
        email = profile.email
    }

    private func fetchProfile() async -> Profile {
        // ネットワークリクエスト
        return Profile(name: "Alice", email: "alice@example.com")
    }
}
```

### 2. 個別のメソッドやプロパティに適用

```swift
class DataService {
    @MainActor var statusMessage: String = ""

    @MainActor
    func updateStatus(_ message: String) {
        statusMessage = message
    }

    // MainActorに分離されていないメソッド
    func processInBackground() async {
        // バックグラウンドで処理
        let result = await heavyComputation()

        // UIを更新するときはMainActorに切り替え
        await updateStatus("Completed: \(result)")
    }
}
```

### 3. Task内でMainActorを指定

```swift
func startBackgroundWork() {
    Task {
        // バックグラウンドで処理
        let data = await fetchData()

        // MainActorに切り替えてUI更新
        await MainActor.run {
            self.updateUI(with: data)
        }
    }
}

// または Task に直接 @MainActor を指定
func startUITask() {
    Task { @MainActor in
        // このTask全体がMainActorで実行
        let data = await fetchData()
        self.updateUI(with: data)
    }
}
```

### 4. MainActor.run の使用

```swift
func processData() async {
    // バックグラウンドで重い処理
    let result = await performHeavyTask()

    // 結果をMainActorで処理
    await MainActor.run {
        self.results = result
        self.tableView.reloadData()
    }
}

// 戻り値がある場合
func getUIState() async -> String {
    return await MainActor.run {
        return self.textField.text ?? ""
    }
}
```

## ベストプラクティス

### 1. ViewModelは@MainActorでマーク

```swift
@MainActor
final class SearchViewModel: ObservableObject {
    @Published var searchText: String = ""
    @Published var results: [SearchResult] = []
    @Published var isSearching: Bool = false

    func search() async {
        isSearching = true
        defer { isSearching = false }

        results = await searchService.search(query: searchText)
    }
}
```

### 2. nonisolatedで必要に応じてオプトアウト

```swift
@MainActor
class UserManager {
    var currentUser: User?

    // MainActorから外れる必要がある場合
    nonisolated func createId() -> UUID {
        return UUID()  // 状態にアクセスしないのでどこからでも呼べる
    }

    // 非同期でMainActorから外れる
    nonisolated func hashPassword(_ password: String) async -> String {
        // 重い計算をバックグラウンドで
        return await computeHash(password)
    }
}
```

### 3. アクター分離エラーの解決パターン

```swift
// エラー: Call to main actor-isolated instance method 'update()'
// in a synchronous nonisolated context

// 解決策1: 呼び出し元を@MainActorにする
@MainActor
func runUpdate(model: MyModel) {
    model.update()
}

// 解決策2: Taskで@MainActorを指定
func runUpdate(model: MyModel) {
    Task { @MainActor in
        model.update()
    }
}

// 解決策3: async関数にしてawaitを使う
func runUpdate(model: MyModel) async {
    await model.update()
}
```

## Swift 6.2のデフォルトMainActor分離

### 背景

Swift 6.2（WWDC 2025）で導入された「Default Actor Isolation」により、新規プロジェクトではコードがデフォルトで`@MainActor`に分離されるようになりました。

### 設定方法（Xcode 26以降）

Build Settingsで設定可能：

```
// Swift Compiler - General
Default Actor Isolation = MainActor
```

### 動作の変更

```swift
// Swift 6.2 (Default Actor Isolation = MainActor)

// 明示的なアノテーションなしでMainActorに分離
class MyViewModel {
    var data: [String] = []  // MainActor分離

    func loadData() {        // MainActor分離
        // ...
    }
}

// 明示的にMainActorから外す場合
nonisolated class BackgroundProcessor {
    // ...
}
```

### 影響

1. **新規プロジェクト**: デフォルトでMainActor分離が有効
2. **既存プロジェクト**: 従来の動作を維持（nonisolated）
3. **移行**: 段階的に有効化可能

### メリット

- UI関連コードでの`@MainActor`アノテーションが不要に
- データ競合エラーの早期発見
- SwiftUIとの親和性向上

### 注意点

```swift
// Default Actor Isolation = MainActor の場合

// バックグラウンド処理には明示的な指定が必要
@concurrent  // Swift 6.2の新しい属性
func performHeavyTask() async {
    // バックグラウンドで実行
}

// または nonisolated を使用
nonisolated func computeHash(_ input: String) -> String {
    // どのアクターからも呼び出し可能
}
```

## MainActor vs DispatchQueue.main

| 観点 | MainActor | DispatchQueue.main |
|------|-----------|-------------------|
| 型安全性 | コンパイル時チェック | ランタイムのみ |
| 忘れにくさ | コンパイルエラーで検出 | バグになりやすい |
| async/await統合 | ネイティブ対応 | 別の仕組み |
| パフォーマンス | 最適化されている | オーバーヘッドあり |

## まとめ

| 概念 | 説明 |
|------|------|
| @MainActor | メインスレッド実行を保証するグローバルアクター |
| MainActor.run | 任意の場所からMainActorで実行 |
| nonisolated | MainActorから外れる明示的な指定 |
| Default Actor Isolation | Swift 6.2のデフォルトMainActor分離 |

## 次のステップ

- [GlobalActor](03_global_actor.md) - カスタムグローバルアクターの作成
