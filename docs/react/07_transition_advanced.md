# トランジションの応用

トランジションの「世界の分岐」概念と、意図的にフォールバックを表示する「オプトアウト」について解説します。

## 世界の分岐

Reactがトランジション実行時、**2つの異なる世界のUIを同時にレンダリング**しています。

```
古い世界: userId="user1", isPending=true  ← 画面に表示
新しい世界: userId="user2", サスペンド中   ← バックグラウンド
```

トランジションは「優先度が低い更新」のため、Reactは「完全なUIができあがるまで待つ」ことを選択します。

## 古いUIが維持される条件

Reactは自動的に「古いUIを維持するか」「フォールバックを表示するか」を判断します。

**判断基準**: 「すでにマウント済みのSuspenseが再サスペンドしたかどうか」

| 状況 | 動作 |
|------|------|
| 既存Suspenseが再サスペンド | 古いUI維持 |
| 新規Suspenseがサスペンド | フォールバック表示 |

### 例: 新しいSuspenseがサスペンドする場合

```tsx
function UserPage() {
  const [showPosts, setShowPosts] = useState(false);

  return (
    <div>
      <UserProfile />
      <button onClick={() => {
        startTransition(() => {
          setShowPosts(true);
        });
      }}>
        投稿を見る
      </button>

      {showPosts && (
        <Suspense fallback={<LoadingSpinner />}>
          <UserPosts />
        </Suspense>
      )}
    </div>
  );
}
```

**動作**: `showPosts`が`false`のときSuspenseはマウントされていないため、ボタン押下時は新しいSuspenseがサスペンドし、フォールバックが表示されます。

### Suspenseの配置による違い

```tsx
// パターンA: 条件付きレンダリングがSuspenseの外
{showPosts && (
  <Suspense fallback={<LoadingSpinner />}>
    <UserPosts />
  </Suspense>
)}

// パターンB: 条件付きレンダリングがSuspenseの中
<Suspense fallback={<LoadingSpinner />}>
  {showPosts && <UserPosts />}
</Suspense>
```

**パターンA**: 新規Suspense → フォールバック表示
**パターンB**: 既存Suspenseの再サスペンド → 古いUI維持

## オプトアウト: 意図的にフォールバックを表示

トランジション使用時でも、あえてフォールバックUIを表示したい場合があります。

### タブUIの問題

```tsx
const App: React.FC = () => {
  const [activeTab, setActiveTab] = useState<'A' | 'B'>('A');

  return (
    <div>
      <button onClick={() => startTransition(() => setActiveTab('A'))}>
        タブA {activeTab === 'A' ? '(選択中)' : ''}
      </button>
      <button onClick={() => startTransition(() => setActiveTab('B'))}>
        タブB {activeTab === 'B' ? '(選択中)' : ''}
      </button>

      <Suspense fallback={<LoadingSpinner />}>
        {activeTab === 'A' ? <TabAContent /> : <TabBContent />}
      </Suspense>
    </div>
  );
};
```

**問題**: タブBを押してもすぐに「タブB（選択中）」にならず、古い内容が維持されます。

### 解決策1: Suspenseを分割

```tsx
{activeTab === 'A' && (
  <Suspense fallback={<LoadingSpinner />}>
    <TabAContent />
  </Suspense>
)}
{activeTab === 'B' && (
  <Suspense fallback={<LoadingSpinner />}>
    <TabBContent />
  </Suspense>
)}
```

異なるタブ用のSuspenseは別々のインスタンスになるため、タブ切り替え時は新しいSuspenseがサスペンドし、フォールバックUIが表示されます。

### 解決策2: key属性を使用（推奨）

```tsx
<Suspense key={activeTab} fallback={<LoadingSpinner />}>
  {activeTab === 'A' ? <TabAContent /> : <TabBContent />}
</Suspense>
```

`key`が変わるたびにSuspenseのインスタンスが変わるため、フォールバックUIが表示されます。

> **参考**: [Reactのkey属性テクニック](https://zenn.dev/uhyo/articles/react-key-techniques)

## ネストしたトランジション

```tsx
const [isPendingA, startTransitionA] = useTransition();
const [isPendingB, startTransitionB] = useTransition();

startTransitionA(() => {
  startTransitionB(() => {
    setState(...);
  });
});
```

**仕様** (v19.2時点):
- 複数トランジションの同時並行処理はサポートされていない
- 内部的には1つのトランジションに統合される
- サスペンド時はすべてのpending状態が`true`になる

## トランジションにできないステート更新

### 制御コンポーネントの制限

```tsx
const [name, setName] = useState('');

<input
  type="text"
  value={name}
  onChange={(e) => {
    // ❌ これはトランジションにできない
    setName(e.target.value);
  }}
/>
```

**理由**:
- 制御コンポーネントの場合、ユーザー入力時点でDOM上の状態が既に変更
- ステート更新は「後追い」で整合性を保つもの
- トランジション化するとサスペンド時に入力が反映されない

## 実践例: ページナビゲーション

```tsx
const App: React.FC = () => {
  const [currentPage, setCurrentPage] = useState<PageId>('home');

  const handleNavigate = (pageId: PageId) => {
    startTransition(() => {
      setCurrentPage(pageId);
    });
  };

  return (
    <div>
      <nav>
        <button onClick={() => handleNavigate('home')}>
          ホーム {currentPage === 'home' ? '(現在)' : ''}
        </button>
        <button onClick={() => handleNavigate('profile')}>
          プロフィール {currentPage === 'profile' ? '(現在)' : ''}
        </button>
      </nav>

      {/* keyでオプトアウト: ページ切り替え時はフォールバック表示 */}
      <Suspense key={currentPage} fallback={<p>ページを読み込み中...</p>}>
        <PageContent pageId={currentPage} />
      </Suspense>
    </div>
  );
};
```

## まとめ

| 概念 | 説明 |
|------|------|
| 世界の分岐 | 古いUIと新しいUIを並行レンダリング |
| 自動判断 | 再サスペンドか新規かで動作が変わる |
| key属性 | 意図的にフォールバックを表示 |

次は[jotai-eagerを使う](./08_jotai_eager.md)で非同期処理の最適化を学びます。
