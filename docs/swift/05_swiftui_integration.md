# SwiftUIとの連携

## ViewプロトコルとMainActor

### WWDC 2024の重要な変更

WWDC 2024で、SwiftUIの`View`プロトコル全体が`@MainActor`でマークされました。これにより、`View`に準拠するすべての型は自動的に`@MainActor`に分離されます。

```swift
// SwiftUIの定義（概念的）
@MainActor
public protocol View {
    associatedtype Body: View
    @ViewBuilder var body: Body { get }
}

// すべてのViewは自動的に@MainActor
struct ContentView: View {
    // @MainActorが自動的に適用される
    var body: some View {
        Text("Hello, World!")
    }
}
```

### Swift 6以前との違い

Swift 5.x/6.0では、`@StateObject`や`@ObservedObject`のプロパティラッパーがViewに`@MainActor`を推論していました（SE-0401で廃止）：

```swift
// Swift 5.x - プロパティラッパーからの推論
struct OldStyleView: View {
    @StateObject var viewModel = MyViewModel()  // MainActorを推論

    var body: some View {
        Text(viewModel.text)
    }
}

// Swift 6.2+ - Viewプロトコル自体が@MainActor
struct NewStyleView: View {
    // Viewに準拠した時点で@MainActor
    var body: some View {
        Text("Hello")
    }
}
```

## @Observableマクロとアクター分離

### @Observableの基本

iOS 17/macOS 14で導入された`@Observable`マクロは、SwiftUIの新しい状態管理方法です：

```swift
@Observable
class CounterModel {
    var count = 0

    func increment() {
        count += 1
    }
}

struct CounterView: View {
    var model = CounterModel()

    var body: some View {
        VStack {
            Text("Count: \(model.count)")
            Button("Increment") {
                model.increment()
            }
        }
    }
}
```

### @Observableと@MainActorの組み合わせ

`@Observable`マクロ自体はアクター分離を推論しません。UI更新を行うモデルには明示的に`@MainActor`を付ける必要があります：

```swift
@MainActor
@Observable
class UserViewModel {
    var name: String = ""
    var isLoading: Bool = false
    var error: String?

    func loadUser() async {
        isLoading = true
        defer { isLoading = false }

        do {
            let user = try await fetchUser()
            name = user.name
        } catch {
            self.error = error.localizedDescription
        }
    }
}
```

### @Observable使用時の注意点

```swift
// 問題: @MainActorなしでUI更新
@Observable
class ProblematicModel {
    var data: [String] = []

    func loadData() async {
        // バックグラウンドスレッドで実行される可能性
        data = await fetchData()  // ⚠️ UIスレッドでない可能性
    }
}

// 解決策: @MainActorを明示
@MainActor
@Observable
class SafeModel {
    var data: [String] = []

    func loadData() async {
        // MainActor上で実行されることが保証
        data = await fetchData()  // ✅ 安全
    }
}
```

## ObservableObjectからの移行

### 従来のパターン（ObservableObject）

```swift
class LegacyViewModel: ObservableObject {
    @Published var items: [Item] = []
    @Published var isLoading = false

    @MainActor
    func loadItems() async {
        isLoading = true
        defer { isLoading = false }
        items = await fetchItems()
    }
}

struct LegacyView: View {
    @StateObject var viewModel = LegacyViewModel()

    var body: some View {
        List(viewModel.items) { item in
            Text(item.name)
        }
        .task {
            await viewModel.loadItems()
        }
    }
}
```

### 新しいパターン（@Observable）

```swift
@MainActor
@Observable
class ModernViewModel {
    var items: [Item] = []
    var isLoading = false

    func loadItems() async {
        isLoading = true
        defer { isLoading = false }
        items = await fetchItems()
    }
}

struct ModernView: View {
    @State var viewModel = ModernViewModel()

    var body: some View {
        List(viewModel.items) { item in
            Text(item.name)
        }
        .task {
            await viewModel.loadItems()
        }
    }
}
```

### 移行時の主な変更点

| 項目 | ObservableObject | @Observable |
|------|------------------|-------------|
| プロパティラッパー | @Published | 不要 |
| View側の宣言 | @StateObject / @ObservedObject | @State |
| MainActor推論 | @StateObjectが推論 | 明示が必要 |
| 環境オブジェクト | @EnvironmentObject | @Environment |

### 移行時のエラーと解決策

```swift
// エラー: Call to main actor-isolated initializer in a synchronous nonisolated context

// 問題のあるコード
@Observable
class ViewModel {
    // ...
}

struct MyView: View {
    @State var viewModel = ViewModel()  // エラー
}

// 解決策1: ViewModelを@MainActorにする
@MainActor
@Observable
class ViewModel {
    // ...
}

// 解決策2: Viewも明示的に@MainActorにする（通常は不要）
@MainActor
struct MyView: View {
    @State var viewModel = ViewModel()
}
```

## SwiftUIでのベストプラクティス

### 1. ViewModelは常に@MainActor

```swift
@MainActor
@Observable
final class SearchViewModel {
    var query: String = ""
    var results: [SearchResult] = []
    var isSearching = false
    var errorMessage: String?

    private let searchService: SearchService

    init(searchService: SearchService = .shared) {
        self.searchService = searchService
    }

    func search() async {
        guard !query.isEmpty else {
            results = []
            return
        }

        isSearching = true
        errorMessage = nil
        defer { isSearching = false }

        do {
            results = try await searchService.search(query)
        } catch {
            errorMessage = error.localizedDescription
            results = []
        }
    }
}
```

### 2. 重い処理はバックグラウンドで

```swift
@MainActor
@Observable
class ImageProcessor {
    var processedImage: UIImage?
    var isProcessing = false

    func processImage(_ input: UIImage) async {
        isProcessing = true
        defer { isProcessing = false }

        // 重い処理はバックグラウンドで実行
        let result = await Task.detached(priority: .userInitiated) {
            return self.applyFilters(to: input)
        }.value

        // 結果の反映はMainActorで
        processedImage = result
    }

    nonisolated private func applyFilters(to image: UIImage) -> UIImage {
        // CPU集約的な処理
        return image
    }
}
```

### 3. @Environmentでの依存注入

```swift
@MainActor
@Observable
class AppState {
    var currentUser: User?
    var isAuthenticated: Bool { currentUser != nil }

    func signIn(email: String, password: String) async throws {
        currentUser = try await AuthService.shared.signIn(
            email: email,
            password: password
        )
    }

    func signOut() {
        currentUser = nil
    }
}

// App.swift
@main
struct MyApp: App {
    @State private var appState = AppState()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(appState)
        }
    }
}

// 使用側
struct ProfileView: View {
    @Environment(AppState.self) private var appState

    var body: some View {
        if let user = appState.currentUser {
            Text("Welcome, \(user.name)")
        } else {
            Text("Please sign in")
        }
    }
}
```

### 4. SwiftUIでActorを使わない理由

SwiftUIのデータモデルには通常のアクターではなく、`@MainActor`付きクラスを使うべきです：

```swift
// ❌ 避けるべき: SwiftUIモデルにactor
actor BadViewModel {
    var items: [Item] = []

    func loadItems() async {
        items = await fetchItems()
    }
}

struct BadView: View {
    let viewModel = BadViewModel()

    var body: some View {
        // エラー: actorプロパティへの同期アクセス不可
        List(viewModel.items) { item in  // ❌
            Text(item.name)
        }
    }
}

// ✅ 推奨: @MainActorクラス
@MainActor
@Observable
class GoodViewModel {
    var items: [Item] = []

    func loadItems() async {
        items = await fetchItems()
    }
}

struct GoodView: View {
    @State var viewModel = GoodViewModel()

    var body: some View {
        List(viewModel.items) { item in  // ✅
            Text(item.name)
        }
    }
}
```

**理由**:
- SwiftUIのViewはMainActor上で動作
- actorプロパティへのアクセスは常にawaitが必要
- Viewのbody内でawaitは使用不可

## Swift 6.2での注意点

### Default Actor Isolationの影響

Swift 6.2でDefault Actor Isolation = MainActorを有効にした場合：

```swift
// すべてのコードがデフォルトでMainActorに分離

// 明示的なアノテーションなしでMainActor
@Observable
class ViewModel {  // 暗黙的に@MainActor
    var data: [String] = []
}

// バックグラウンド処理には明示的指定が必要
nonisolated func backgroundTask() async {
    // MainActorから外れる
}

@concurrent
func parallelWork() async {
    // グローバルエグゼキュータで実行
}
```

## まとめ

| パターン | 推奨される使い方 |
|----------|------------------|
| ViewModel | `@MainActor @Observable class` |
| View | `View`プロトコルで自動的に`@MainActor` |
| 重い処理 | `Task.detached`でバックグラウンド実行 |
| 状態共有 | `@Environment`で依存注入 |
| データモデル | `actor`ではなく`@MainActor`クラス |

## 参考リンク

- [Apple Developer - Observation](https://developer.apple.com/documentation/observation)
- [SwiftLee - Default Actor Isolation](https://www.avanderlee.com/concurrency/default-actor-isolation-in-swift-6-2/)
- [Fat Bob Man - SwiftUI Views and MainActor](https://fatbobman.com/en/posts/swiftui-views-and-mainactor/)
- [Hacking with Swift - Do not use actor for SwiftUI data models](https://www.hackingwithswift.com/quick-start/concurrency/important-do-not-use-an-actor-for-your-swiftui-data-models)
