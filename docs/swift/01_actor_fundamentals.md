# Actorの基本

## Actorとは何か

Actorは、Swift 5.5で導入された並行処理のための参照型です。クラスと似ていますが、**データ競合（Data Race）を防ぐ仕組み**が組み込まれています。

```swift
actor BankAccount {
    private var balance: Int = 0

    func deposit(amount: Int) {
        balance += amount
    }

    func withdraw(amount: Int) -> Bool {
        guard balance >= amount else { return false }
        balance -= amount
        return true
    }

    func getBalance() -> Int {
        return balance
    }
}
```

### Actorの特徴

1. **参照型**: クラスと同様に参照セマンティクスを持つ
2. **分離された状態**: 内部状態は外部から直接アクセスできない
3. **直列化されたアクセス**: 一度に一つのタスクのみがアクター内のコードを実行
4. **Sendableに自動準拠**: アクターは自動的にSendableプロトコルに準拠

## データ競合（Data Race）の問題

データ競合は、複数のスレッドが同じメモリ位置に同時にアクセスし、少なくとも1つが書き込みを行う場合に発生します。

### 従来の問題（クラスの場合）

```swift
// 危険: データ競合の可能性あり
class UnsafeCounter {
    var count = 0

    func increment() {
        count += 1  // 読み取り → 加算 → 書き込み の非原子操作
    }
}

let counter = UnsafeCounter()

// 複数のタスクから同時にアクセス
Task { counter.increment() }
Task { counter.increment() }
// 結果が不定になる可能性
```

### Actorによる解決

```swift
actor SafeCounter {
    var count = 0

    func increment() {
        count += 1  // アクター内で直列化されるため安全
    }
}

let counter = SafeCounter()

// awaitが必要 - アクセスが直列化される
Task { await counter.increment() }
Task { await counter.increment() }
// 常に正しい結果
```

## Actor分離（Actor Isolation）の仕組み

### 分離ドメイン

各アクターは独自の**分離ドメイン（Isolation Domain）**を持ちます。アクターの状態（プロパティ）とメソッドは、そのアクターに「分離」されています。

```swift
actor ImageCache {
    private var cache: [String: Data] = [:]

    // アクター分離されたメソッド
    func store(image: Data, for key: String) {
        cache[key] = image
    }

    // アクター分離されたメソッド
    func retrieve(for key: String) -> Data? {
        return cache[key]
    }
}
```

### 外部からのアクセス

アクター外部からアクター分離された状態やメソッドにアクセスする場合、`await`が必要です：

```swift
let imageCache = ImageCache()

// 外部からのアクセスにはawaitが必要
func cacheImage(_ data: Data, key: String) async {
    await imageCache.store(image: data, for: key)
}

func getCachedImage(key: String) async -> Data? {
    return await imageCache.retrieve(for: key)
}
```

### アクター内部からのアクセス

アクター内部では、同じアクターの状態やメソッドに`await`なしでアクセスできます：

```swift
actor DataProcessor {
    private var processedCount = 0
    private var results: [String] = []

    func process(data: String) -> String {
        let result = data.uppercased()
        results.append(result)       // 同期的にアクセス可能
        processedCount += 1          // awaitは不要
        return result
    }

    func getStats() -> (count: Int, results: [String]) {
        return (processedCount, results)  // 内部からは直接アクセス
    }
}
```

## 基本的な使い方

### Actorの定義

```swift
actor UserSession {
    private var user: User?
    private var token: String?
    private var lastActivity: Date = Date()

    func login(user: User, token: String) {
        self.user = user
        self.token = token
        self.lastActivity = Date()
    }

    func logout() {
        user = nil
        token = nil
    }

    func isLoggedIn() -> Bool {
        return user != nil && token != nil
    }

    func getCurrentUser() -> User? {
        lastActivity = Date()
        return user
    }
}
```

### 非同期コンテキストからの使用

```swift
let session = UserSession()

// 非同期関数内で使用
func performLogin() async {
    let user = User(id: 1, name: "Alice")
    await session.login(user: user, token: "abc123")

    if await session.isLoggedIn() {
        print("ログイン成功")
    }
}

// Taskを使用
func startLogin() {
    Task {
        await performLogin()
    }
}
```

### Actor間の連携

複数のアクターが連携する場合、それぞれのアクセスに`await`が必要です：

```swift
actor AuthService {
    func validateToken(_ token: String) async -> Bool {
        // トークン検証ロジック
        return true
    }
}

actor UserService {
    private let auth: AuthService
    private var currentUser: User?

    init(auth: AuthService) {
        self.auth = auth
    }

    func authenticateUser(token: String) async -> User? {
        // 別のアクターへのアクセス
        guard await auth.validateToken(token) else {
            return nil
        }

        // 自分自身の状態を更新
        let user = User(id: 1, name: "User")
        currentUser = user
        return user
    }
}
```

## 再入可能性（Reentrancy）

Actorメソッドは再入可能です。`await`でサスペンドしている間に、他のタスクがアクターにアクセスできます。

```swift
actor Account {
    var balance: Int = 100

    func transfer(amount: Int, to other: Account) async {
        guard balance >= amount else { return }

        balance -= amount
        // この await 中に別のタスクが self にアクセス可能
        await other.deposit(amount: amount)
        // await 後、balance の状態が変わっている可能性がある
    }

    func deposit(amount: Int) {
        balance += amount
    }
}
```

### 再入可能性への対処

```swift
actor Account {
    var balance: Int = 100

    func safeTransfer(amount: Int, to other: Account) async -> Bool {
        // await 前に必要なチェックと状態変更を完了
        guard balance >= amount else { return false }
        balance -= amount

        // await 後は self の状態に依存しない
        await other.deposit(amount: amount)
        return true
    }
}
```

## まとめ

| 概念 | 説明 |
|------|------|
| Actor | データ競合を防ぐ参照型 |
| Actor分離 | アクター内の状態への排他的アクセス |
| await | アクター境界を越えるアクセスに必要 |
| 再入可能性 | await中に他のタスクがアクターにアクセス可能 |

## 次のステップ

- [MainActor](02_main_actor.md) - UIスレッドと連携するための特別なアクター
