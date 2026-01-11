# Sendableとアクター境界

## Sendableプロトコルとは

`Sendable`は、型の値がアクター境界を安全に越えられることを示すマーカープロトコルです。Swift Concurrencyにおけるデータ競合防止の中核となる概念です。

```swift
// Sendableプロトコル（マーカープロトコル）
public protocol Sendable { }
```

### なぜSendableが必要か

アクター間でデータを渡すとき、そのデータが並行アクセスに対して安全である必要があります：

```swift
actor DataProcessor {
    func process(_ item: SomeType) {
        // itemがSendableでない場合、データ競合の危険
    }
}

let processor = DataProcessor()
let item = SomeType()
await processor.process(item)  // itemはアクター境界を越える
```

## 自動的にSendableになる型

### 値型（条件付き）

構造体と列挙型は、すべての格納プロパティがSendableの場合、自動的にSendableになります：

```swift
// 自動的にSendable
struct Point {
    var x: Int
    var y: Int
}

// 自動的にSendable
enum Status {
    case pending
    case completed(result: String)
    case failed(error: String)
}

// Sendableではない（配列内の型がSendableでない場合）
struct Container {
    var items: [NonSendableClass]  // Sendableでない
}
```

### Actor

すべてのアクターは自動的にSendableです：

```swift
actor Counter {
    var count = 0
}

let counter: Sendable = Counter()  // OK
```

### 基本型

`Int`、`String`、`Bool`などの基本型はすべてSendableです。

## Sendable適合の明示

### 構造体・列挙型

```swift
// 明示的にSendableを宣言
struct UserData: Sendable {
    let id: Int
    let name: String
    let createdAt: Date
}
```

### クラス（制限付き）

クラスがSendableになるには、以下の条件を満たす必要があります：

```swift
// 1. finalクラスである
// 2. すべての格納プロパティがimmutable（let）
// 3. すべてのプロパティの型がSendable
// 4. 親クラスがないか、NSObjectのみ継承

final class ImmutableConfig: Sendable {
    let apiKey: String
    let timeout: TimeInterval

    init(apiKey: String, timeout: TimeInterval) {
        self.apiKey = apiKey
        self.timeout = timeout
    }
}
```

### @unchecked Sendable（注意して使用）

コンパイラチェックをバイパスしたい場合（自分でスレッド安全性を保証する場合）：

```swift
// 内部でロックを使用してスレッド安全性を保証
final class ThreadSafeCache: @unchecked Sendable {
    private var cache: [String: Any] = [:]
    private let lock = NSLock()

    func get(_ key: String) -> Any? {
        lock.lock()
        defer { lock.unlock() }
        return cache[key]
    }

    func set(_ value: Any, for key: String) {
        lock.lock()
        defer { lock.unlock() }
        cache[key] = value
    }
}
```

**注意**: `@unchecked Sendable`は最後の手段です。可能な限り避けましょう。

## @Sendableクロージャ

クロージャがアクター境界を越える場合、`@Sendable`が必要です：

```swift
actor TaskRunner {
    func run(_ work: @Sendable () async -> Void) async {
        await work()
    }
}

let runner = TaskRunner()

// @Sendableクロージャ
await runner.run {
    print("Running on TaskRunner")
}
```

### @Sendableクロージャの制約

```swift
var mutableState = 0

// エラー: mutableなキャプチャは@Sendableクロージャで不可
Task {
    mutableState += 1  // Capture of 'mutableState' with non-sendable type
}

// 解決策1: letを使う
let immutableValue = 42
Task {
    print(immutableValue)  // OK
}

// 解決策2: Sendableな型をキャプチャ
let sendableData = SendableStruct(value: 42)
Task {
    print(sendableData.value)  // OK
}
```

## nonisolatedとisolated

### nonisolated

`nonisolated`は、アクターの分離から外れることを明示します：

```swift
actor BankAccount {
    let accountNumber: String  // 不変
    var balance: Int = 0       // 可変

    init(accountNumber: String) {
        self.accountNumber = accountNumber
    }

    // 不変プロパティへのアクセスはnonisolatedでOK
    nonisolated var formattedAccountNumber: String {
        return "Account: \(accountNumber)"
    }

    // 状態にアクセスしないメソッド
    nonisolated func generateTransactionId() -> String {
        return UUID().uuidString
    }

    // 可変状態にはnonisolatedでアクセス不可
    // nonisolated var currentBalance: Int {
    //     return balance  // エラー
    // }
}

let account = BankAccount(accountNumber: "123")

// awaitなしで呼び出し可能
let formatted = account.formattedAccountNumber
let txId = account.generateTransactionId()
```

### isolated パラメータ

関数のパラメータにアクター分離を指定できます：

```swift
func performOperation(on account: isolated BankAccount) {
    // accountのアクター分離されたメンバーに直接アクセス可能
    account.balance += 100  // awaitなしでOK
}

// 使用時
let account = BankAccount(accountNumber: "123")
await performOperation(on: account)
```

## Swift 6.2の新機能

### nonisolated(nonsending)

SE-0461で導入された`nonisolated(nonsending)`は、非同期関数が呼び出し元のアクター上で実行されることを指定します：

```swift
class Person {
    var name: String = ""

    // 呼び出し元のアクター上で実行（アクター境界を越えない）
    nonisolated(nonsending)
    func printNameConcurrently() async {
        print(name)  // 安全にアクセス可能
    }
}

@MainActor
func onMainActor(person: Person) async {
    // personはMainActor上で実行される
    await person.printNameConcurrently()
}
```

### @concurrent

`@concurrent`は、関数が常にグローバル並行エグゼキュータ上で実行されることを指定します：

```swift
struct Processor: Sendable {
    func performSync() {}

    // 呼び出し元のアクター上で実行（デフォルト、Swift 6.2以降）
    nonisolated(nonsending)
    func performAsync() async {}

    // 常にグローバルエグゼキュータにスイッチ
    @concurrent
    func alwaysSwitch() async {}
}

actor MyActor {
    let processor = Processor()

    func call() async {
        processor.performSync()        // アクター上で実行

        await processor.performAsync() // アクター上で実行

        await processor.alwaysSwitch() // グローバルエグゼキュータで実行
    }
}
```

### NonisolatedNonsendingByDefault

Swift 6.2以降、この機能を有効にすると、nonisolatedなasync関数はデフォルトで呼び出し元のアクター上で実行されます：

```swift
// NonisolatedNonsendingByDefault が有効な場合

struct S: Sendable {
    // 暗黙的に nonisolated(nonsending)
    func performAsync() async {}

    // 明示的にグローバルエグゼキュータを使用
    @concurrent
    func backgroundWork() async {}
}
```

## アクター分離エラーの解決パターン

### パターン1: データ競合エラー

```swift
class MyModel {
    var count: Int = 0

    func perform() {
        Task {
            self.update()  // エラー: データ競合のリスク
        }
    }

    func update() { count += 1 }
}

// 解決策: @MainActorで分離
@MainActor
class MyModel {
    var count: Int = 0

    func perform() {
        Task {
            self.update()  // OK: MainActor上で直列化
        }
    }

    func update() { count += 1 }
}
```

### パターン2: Sendableでない型のアクター境界越え

```swift
// エラーが出る状況
class NonSendableData {
    var value: Int = 0
}

actor Processor {
    func process(_ data: NonSendableData) {
        // NonSendableDataはアクター境界を越えられない
    }
}

// 解決策1: Sendableな型を使う
struct SendableData: Sendable {
    let value: Int
}

// 解決策2: 値をコピーして渡す
actor Processor {
    func process(value: Int) {
        // プリミティブ型はSendable
    }
}
```

### パターン3: 分離された適合（Isolated Conformance）

```swift
protocol DataProvider {
    func getData() -> Data
}

// MainActor分離された適合
@MainActor
struct MainActorProvider: @MainActor DataProvider {
    func getData() -> Data {
        // MainActorでのみ使用可能
        return Data()
    }
}
```

## まとめ

| 概念 | 説明 |
|------|------|
| Sendable | アクター境界を安全に越えられる型 |
| @Sendable | クロージャがアクター境界を越えられることを示す |
| nonisolated | アクター分離から外れる |
| isolated | パラメータのアクター分離を指定 |
| nonisolated(nonsending) | 呼び出し元のアクター上で実行（Swift 6.2） |
| @concurrent | グローバルエグゼキュータで実行（Swift 6.2） |

## 次のステップ

- [SwiftUIとの連携](05_swiftui_integration.md) - SwiftUIアプリでのアクター活用
