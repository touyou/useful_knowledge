# GlobalActor

## GlobalActorとは

`GlobalActor`は、プログラム全体で一意なアクターを定義するためのプロトコルです。`@MainActor`も`GlobalActor`の一種であり、同じ仕組みを使ってカスタムのグローバルアクターを作成できます。

### GlobalActorプロトコルの定義

```swift
public protocol GlobalActor {
    associatedtype ActorType: Actor
    static var shared: ActorType { get }
}
```

要件は単純で、**常に同じインスタンスを返す`shared`プロパティ**を提供するだけです。

## カスタムGlobalActorの作成

### 基本的な定義

```swift
@globalActor
actor ImageProcessingActor: GlobalActor {
    static let shared = ImageProcessingActor()
}
```

`@globalActor`属性をつけることで、この型を属性として使用できるようになります。

### 使用例：画像処理アクター

```swift
@globalActor
actor ImageProcessor: GlobalActor {
    static let shared = ImageProcessor()

    private var cache: [String: UIImage] = [:]

    func process(image: UIImage) -> UIImage {
        // 画像処理ロジック
        return image
    }

    func getCached(key: String) -> UIImage? {
        return cache[key]
    }

    func cache(image: UIImage, key: String) {
        cache[key] = image
    }
}

// 使用
@ImageProcessor
func processUserAvatar(_ image: UIImage) async -> UIImage {
    // ImageProcessorアクター上で実行される
    return await ImageProcessor.shared.process(image: image)
}
```

### 使用例：データベースアクター

```swift
@globalActor
actor DatabaseActor: GlobalActor {
    static let shared = DatabaseActor()
}

@DatabaseActor
class UserRepository {
    private var users: [User] = []

    func save(_ user: User) {
        users.append(user)
    }

    func findAll() -> [User] {
        return users
    }

    func findById(_ id: Int) -> User? {
        return users.first { $0.id == id }
    }
}

// 使用
let repo = UserRepository()

Task { @DatabaseActor in
    repo.save(User(id: 1, name: "Alice"))
    let users = repo.findAll()
    print(users)
}
```

## グローバルアクターの適用範囲

### 関数とプロパティ

```swift
@DatabaseActor var globalConfig: DatabaseConfig?

@DatabaseActor
func initializeDatabase() {
    globalConfig = DatabaseConfig()
}
```

### クラスと構造体

```swift
@DatabaseActor
class DatabaseManager {
    var connection: Connection?

    func connect() {
        // すべてのメソッドがDatabaseActor上で実行
    }

    func disconnect() {
        connection = nil
    }
}
```

### プロトコル適合

```swift
protocol DataStore {
    func save(_ data: Data)
    func load() -> Data?
}

@DatabaseActor
class SecureStore: DataStore {
    private var storage: Data?

    // プロトコルメソッドもDatabaseActorに分離
    func save(_ data: Data) {
        storage = data
    }

    func load() -> Data? {
        return storage
    }
}
```

## 使用シナリオ

### 1. 特定リソースへの排他アクセス

```swift
@globalActor
actor FileSystemActor: GlobalActor {
    static let shared = FileSystemActor()
}

@FileSystemActor
class LogManager {
    private let fileHandle: FileHandle

    init(path: String) throws {
        fileHandle = try FileHandle(forWritingTo: URL(fileURLWithPath: path))
    }

    func write(_ message: String) {
        guard let data = message.data(using: .utf8) else { return }
        fileHandle.write(data)
    }
}
```

### 2. サードパーティライブラリのラップ

スレッドセーフでないライブラリをアクターでラップ：

```swift
@globalActor
actor LegacyAPIActor: GlobalActor {
    static let shared = LegacyAPIActor()
}

@LegacyAPIActor
class LegacyAPIWrapper {
    private let legacyClient = LegacyThirdPartyClient()

    func fetchData() -> Data {
        return legacyClient.getData()  // スレッドセーフでないAPI
    }

    func sendData(_ data: Data) {
        legacyClient.send(data)
    }
}
```

### 3. 共有キャッシュの管理

```swift
@globalActor
actor CacheActor: GlobalActor {
    static let shared = CacheActor()
}

@CacheActor
final class ResponseCache {
    private var cache: [URL: CachedResponse] = [:]
    private let maxSize: Int = 100

    func get(_ url: URL) -> CachedResponse? {
        guard let cached = cache[url] else { return nil }
        if cached.isExpired {
            cache.removeValue(forKey: url)
            return nil
        }
        return cached
    }

    func set(_ response: CachedResponse, for url: URL) {
        if cache.count >= maxSize {
            evictOldest()
        }
        cache[url] = response
    }

    private func evictOldest() {
        // 最も古いエントリを削除
    }
}
```

## 制約と注意点

### 1. 複数のグローバルアクター属性は不可

```swift
// エラー: 複数のグローバルアクター属性
@MainActor @DatabaseActor
func invalidFunction() { }  // コンパイルエラー
```

### 2. インスタンスアクターとの組み合わせ不可

```swift
actor MyActor {
    // エラー: アクターのメソッドにグローバルアクター属性は不可
    @MainActor
    func invalidMethod() { }  // コンパイルエラー
}
```

### 3. クラス継承時の制約

サブクラスは親クラスと同じグローバルアクターに分離される必要があります：

```swift
@MainActor
class Parent { }

// OK: 親と同じグローバルアクター
@MainActor
class Child: Parent { }

// エラー: 異なるグローバルアクター
@DatabaseActor
class InvalidChild: Parent { }  // コンパイルエラー
```

### 4. nonisolatedでオプトアウト

```swift
@DatabaseActor
class DataManager {
    let id: UUID = UUID()  // 不変なのでnonisolatedでも安全

    nonisolated var identifier: String {
        return id.uuidString  // グローバルアクター外からも同期的にアクセス可能
    }

    nonisolated func generateKey() -> String {
        return UUID().uuidString  // 状態にアクセスしないので安全
    }
}
```

## GlobalActorの推論規則

アクター分離は以下のルールで推論されます：

1. **サブクラス**: スーパークラスから推論
2. **オーバーライド**: オーバーライドする宣言から推論
3. **プロトコル適合**: 同一ソースファイル内で定義された場合、プロトコルから推論

```swift
@MainActor
protocol UIUpdatable {
    func updateUI()
}

// UIUpdatableに適合 → @MainActorが推論される
class MyView: UIUpdatable {
    func updateUI() {
        // @MainActorに分離されている
    }
}
```

## MainActorとカスタムGlobalActorの使い分け

| ユースケース | 推奨 |
|-------------|------|
| UI更新 | @MainActor |
| データベースアクセス | カスタムGlobalActor |
| ファイルI/O | カスタムGlobalActor |
| ネットワーキング | 通常のactor or カスタムGlobalActor |
| サードパーティライブラリ | カスタムGlobalActor |

## まとめ

| 概念 | 説明 |
|------|------|
| GlobalActor | プログラム全体で一意なアクター |
| @globalActor | カスタムグローバルアクターを定義する属性 |
| shared | グローバルアクターのシングルトンインスタンス |
| 推論規則 | 継承やプロトコル適合からの自動推論 |

## 次のステップ

- [Sendableとアクター境界](04_sendable_isolation.md) - アクター間でデータを安全に渡す方法
