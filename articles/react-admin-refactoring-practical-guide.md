---
title: "877行のReactコンポーネントを150行にした7つのリファクタリング手法"
emoji: "🔧"
type: "tech"
topics: ["React", "TypeScript", "NextJS", "リファクタリング", "パフォーマンス"]
published: true
---

## はじめに

Next.js の管理画面を開発していたら、気づけば1つのページコンポーネントが **877行** に膨れ上がっていました。型定義、バリデーション、グラフ構築ロジック、フォーム状態管理、UIが全て1ファイルに混在する状態です。

:::message
**対象読者**: 「動いているけど読みにくい」Reactコンポーネントの整理に悩んでいる方。Next.js App Router + TypeScript 環境を想定していますが、テクニック自体はReact全般に適用できます。
:::

この記事で得られる知見:

- 巨大コンポーネントを**カスタムフック + 子コンポーネント**に分割する判断基準
- `URL.createObjectURL` の**メモリリーク防止**パターン
- `React.memo` が**効かない罠**と正しい適用方法
- インデックスベースIDの**衝突バグ**と UUID による解決
- APIポーリングの**差分取得 + Exponential Backoff** 実装
- `useMemo` による**線形検索の排除**
- ユーティリティ関数の**共通化判断**

## 全体像

リファクタリング前後の構成を示します。

```
【Before】
page.tsx (877行)
├── 型定義 (SurveyChoice, SurveyQuestion)
├── バリデーション関数
├── グラフ構築ロジック
├── フォーム状態管理 (13個のuseState)
├── イベントハンドラ (~20個)
└── UI (JSX 400行)

【After】
page.tsx (150行) ── UIのみ
├── types.ts ── 型定義 + ヘルパー
├── validateQuestions.ts ── バリデーション
├── buildGraphPayload.ts ── グラフ構築
├── useFormState.ts ── カスタムフック
└── components/
    ├── ItemList.tsx ── 一覧テーブル
    ├── QuestionEditor.tsx ── 質問カード
    └── ChoiceEditor.tsx ── 選択肢行
```

## 手法1: カスタムフックへのロジック抽出

### なぜ分けるか

1ファイルに13個の `useState` と20個のハンドラがあると、「この状態はどのハンドラから更新されるか」の追跡が困難になります。

:::message
**分割の判断基準**: 「このファイルを開いて、目的の関数を見つけるまで何秒かかるか？」5秒以上なら分割を検討する価値があります。
:::

### 実装

状態とハンドラをカスタムフックにまとめ、ページコンポーネントはUIのみに専念させます。

```typescript:hooks/useFormState.ts
export function useFormState() {
  const [items, setItems] = useState<Item[]>([createDefaultItem()]);
  const [name, setName] = useState("");
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [status, setStatus] = useState("");

  const updateItem = useCallback(
    (index: number, updater: (item: Item) => Item) => {
      setItems((prev) => prev.map((q, i) => (i === index ? updater(q) : q)));
    },
    []
  );

  const addItem = useCallback(() => {
    setItems((prev) => [...prev, createDefaultItem()]);
  }, []);

  const removeItem = useCallback((index: number) => {
    setItems((prev) => prev.filter((_, i) => i !== index));
  }, []);

  // ... other handlers

  return { items, name, setName, isSubmitting, status, updateItem, addItem, removeItem };
}
```

```typescript:page.tsx
export default function EditorPage() {
  const { items, name, setName, isSubmitting, status, ...handlers } = useFormState();

  return (
    <div className="p-8 max-w-4xl mx-auto">
      <h1>エディタ</h1>
      <form onSubmit={handlers.onSubmit}>
        {items.map((item, index) => (
          <QuestionEditor key={item.id} item={item} index={index} {...handlers} />
        ))}
      </form>
    </div>
  );
}
```

**877行 → 150行**。ページコンポーネントの責務が「レイアウトとデータの受け渡し」に限定されました。

## 手法2: URL.createObjectURL のメモリリーク防止

### 問題

画像プレビューで `URL.createObjectURL()` を使うと、明示的に `revokeObjectURL()` を呼ばない限りメモリが解放されません。管理画面で繰り返し画像を選択・変更すると、メモリリークが蓄積します。

### 解決パターン: Ref による追跡 + 3箇所での解放

```typescript:hooks/useFormState.ts
// Track all created preview URLs
const previewUrlsRef = useRef<Set<string>>(new Set());

// 1. Unmount cleanup
useEffect(() => {
  const urls = previewUrlsRef.current;
  return () => {
    urls.forEach((url) => URL.revokeObjectURL(url));
  };
}, []);

// 2. Revoke on replacement
function handleImageSelect(index: number, file: File) {
  const previewUrl = URL.createObjectURL(file);
  previewUrlsRef.current.add(previewUrl);

  updateItem(index, (item) => {
    // Revoke the old preview URL before replacing
    if (item.imagePreviewUrl) {
      URL.revokeObjectURL(item.imagePreviewUrl);
      previewUrlsRef.current.delete(item.imagePreviewUrl);
    }
    return { ...item, imageFile: file, imagePreviewUrl: previewUrl };
  });
}

// 3. Revoke on form reset after save
async function onSubmit(event: FormEvent) {
  // ... save logic ...

  // Clean up all tracked URLs before resetting state
  previewUrlsRef.current.forEach((url) => URL.revokeObjectURL(url));
  previewUrlsRef.current.clear();
  setItems([createDefaultItem()]);
}
```

:::message
**ポイント**: `useEffect` のクリーンアップで `previewUrlsRef.current` を直接参照すると、React の exhaustive-deps ルールで警告が出ます。ローカル変数にコピーしてから使いましょう。
```typescript
useEffect(() => {
  const urls = previewUrlsRef.current; // Copy to local variable
  return () => { urls.forEach((url) => URL.revokeObjectURL(url)); };
}, []);
```
:::

### アンマウント時のステート参照問題

メッセージ入力コンポーネントでも同様の問題があります。`useEffect` のクリーンアップは初回レンダリング時のクロージャを保持するため、最新の `attachment` を参照できません。

```diff typescript:components/MessageInput.tsx
+ const attachmentRef = useRef(attachment);
+ attachmentRef.current = attachment;
+
  useEffect(() => {
    return () => {
-     if (attachment?.previewUrl) {
-       URL.revokeObjectURL(attachment.previewUrl);
+     if (attachmentRef.current?.previewUrl) {
+       URL.revokeObjectURL(attachmentRef.current.previewUrl);
      }
    };
  }, []);
```

## 手法3: インデックスベースIDの衝突バグ修正

### 問題の再現

質問IDを `q_{index + 1}` のようにインデックスから生成していると、削除→追加で衝突が起きます。

```
初期状態: q_1, q_2, q_3
q_2を削除: q_1, q_3
新規追加: q_1, q_3, q_3  ← 衝突！
```

選択肢の遷移先が `q_3` を参照していた場合、意図しない質問にジャンプするバグになります。

### 修正

```diff typescript:types.ts
  export function createItemId(): string {
-   return `q_${index + 1}`;
+   return `q_${crypto.randomUUID().slice(0, 8)}`;
  }
```

`crypto.randomUUID()` はブラウザ・Node.js 両方で利用可能です。先頭8文字に切り詰めても、管理画面の質問数（最大10-20個）では衝突確率は無視できるレベルです。

:::message
**バリデーション側も強化**: 万一の衝突に備え、保存時に重複IDチェックと参照先存在チェックを追加しておくと安全です。
```typescript
const ids = items.map((q) => q.id);
if (new Set(ids).size !== ids.length) {
  throw new Error("IDが重複しています。項目を追加し直してください");
}
```
:::

## 手法4: React.memo が効かない罠と正しい適用

### よくある失敗

リストアイテムに `React.memo` を適用しても、親が毎レンダリングで新しいコールバックを渡していると効果がありません。

```typescript:components/UserList.tsx
// ❌ Every render creates a new function reference
{users.map((user) => (
  <UserItem
    key={user.id}
    user={user}
    onClick={() => onSelectUser(user.id)}  // New function every time!
  />
))}
```

### 正しいパターン

子コンポーネント側でIDを受け取り、内部でクリック時にIDを渡すようにします。

```typescript:components/UserItem.tsx
export const UserItem = memo(function UserItem({
  user,
  selected,
  onSelect,
}: {
  user: User;
  selected: boolean;
  onSelect: (userId: string) => void;
}) {
  return (
    <li
      onClick={() => onSelect(user.id)}
      className={selected ? "bg-blue-50" : ""}
    >
      {user.name}
    </li>
  );
});
```

```typescript:components/UserList.tsx
// ✅ Stable callback reference with useCallback
const handleSelect = useCallback((userId: string) => {
  setSelectedUserId(userId);
}, []);

{users.map((user) => (
  <UserItem
    key={user.id}
    user={user}
    selected={user.id === selectedUserId}
    onSelect={handleSelect}  // Same reference every render
  />
))}
```

| パターン | memo効果 | 理由 |
|---------|---------|------|
| `onClick={() => handler(id)}` を親で生成 | なし | 毎回新しい関数参照 |
| `onSelect={stableCallback}` + 子でID付与 | あり | 参照が安定 |

## 手法5: useMemo による線形検索の排除

ユーザー一覧から選択中のユーザーを `find()` で検索するのは、リスト画面で頻繁に再レンダリングされる場合に無駄です。

```diff typescript:page.tsx
+ const usersMap = useMemo(
+   () => new Map(users.map((u) => [u.id, u])),
+   [users]
+ );

- const selectedUser = users.find((u) => u.id === selectedId);
+ const selectedUser = selectedId ? usersMap.get(selectedId) : undefined;
```

**O(n) → O(1)** の改善。ユーザー数が数百〜数千の管理画面では体感差が出ます。

## 手法6: APIポーリングの差分取得 + Exponential Backoff

### Before: 10秒ごとに全件再取得

```typescript
// Every 10 seconds, fetch the latest 50 messages
const data = await api.getMessageHistory(userId, { limit: 50 });
```

全件を再取得して差分を計算するのは帯域の無駄です。

### After: カーソルベースの差分取得

```
Client                          Server
  |                               |
  |-- GET ?limit=50 ------------->|  (Initial fetch)
  |<-- [msg1, msg2, ..., msg50] --|
  |                               |
  |   (10s later)                 |
  |-- GET ?after=msg50.createdAt->|  (Differential)
  |<-- [msg51, msg52] ------------|  (Only new messages)
```

```typescript:hooks/useMessageHistory.ts
const POLL_INTERVAL = 10_000;
const MAX_BACKOFF_MULTIPLIER = 6; // max 60s

const latestCursorRef = useRef<string | null>(null);

// Polling with exponential backoff
useEffect(() => {
  let consecutiveErrors = 0;

  const scheduleNext = () => {
    const backoff = Math.min(Math.pow(2, consecutiveErrors), MAX_BACKOFF_MULTIPLIER);
    pollRef.current = setTimeout(runPoll, POLL_INTERVAL * backoff);
  };

  const runPoll = async () => {
    try {
      const cursor = latestCursorRef.current;
      const data = await api.getMessages(userId, {
        limit: 50,
        ...(cursor ? { after: cursor } : {}),
      });

      consecutiveErrors = 0; // Reset on success

      if (data.length > 0) {
        setMessages((prev) => {
          const existingIds = new Set(prev.map((m) => m.id));
          const fresh = data.filter((m) => !existingIds.has(m.id));
          if (fresh.length === 0) return prev;
          fresh.sort((a, b) => a.createdAt.localeCompare(b.createdAt));
          return [...prev, ...fresh];
        });

        // Update cursor to the newest message
        const sorted = [...data].sort((a, b) =>
          b.createdAt.localeCompare(a.createdAt)
        );
        const newest = sorted[0];
        if (newest && (!latestCursorRef.current ||
            newest.createdAt > latestCursorRef.current)) {
          latestCursorRef.current = newest.createdAt;
        }
      }
    } catch {
      consecutiveErrors++;
    }
    scheduleNext();
  };

  pollRef.current = setTimeout(runPoll, POLL_INTERVAL);
  return () => { if (pollRef.current) clearTimeout(pollRef.current); };
}, [userId]);
```

:::message
**楽観的更新との併用時の注意**: `addOptimisticMessage()` でカーソルを更新してはいけません。クライアント時刻がサーバーより先に進んでいると、カーソルが未来を指してしまい、サーバーに保存された実メッセージを取りこぼします。カーソル更新はサーバーレスポンスからのみ行いましょう。
:::

| 方式 | 通信量 | エラー耐性 | 実装複雑度 |
|------|--------|-----------|-----------|
| 全件再取得 | 大（毎回N件） | 高 | 低 |
| カーソルベース差分 | 小（新規のみ） | 中 | 中 |
| WebSocket | 最小 | 要再接続処理 | 高 |

管理画面のチャットにはカーソルベース差分がバランスが良い選択肢です。

## 手法7: ユーティリティ関数の共通化判断

2箇所以上で同じロジックが書かれていたら共通化を検討します。今回は音声の再生時間フォーマットが2つのコンポーネントに重複していました。

```typescript:utils/formatDuration.ts
export function formatDurationMs(ms: number): string {
  const totalSec = Math.round(ms / 1000);
  const min = Math.floor(totalSec / 60);
  const sec = totalSec % 60;
  return `${min}:${sec.toString().padStart(2, "0")}`;
}
```

:::message
**単位に注意**: 片方はミリ秒、片方は秒で受け取っていた、というバグの温床になりやすいパターンです。関数名に単位を含める（`formatDurationMs`）ことで、呼び出し側のミスを防ぎます。
:::

## まとめ

### 手法一覧

| 手法 | 解決する問題 | 効果 |
|------|------------|------|
| カスタムフック抽出 | 巨大コンポーネント | 877行→150行 |
| ObjectURL追跡Set | メモリリーク | 3箇所での確実な解放 |
| UUID ID生成 | インデックス衝突 | 削除→追加でも安全 |
| memo + コールバック設計 | 不要な再レンダリング | 親から安定した参照を渡す |
| useMemo Map化 | 線形検索 | O(n) → O(1) |
| カーソルベース差分ポーリング | 帯域の無駄 | 新規メッセージのみ取得 |
| ユーティリティ共通化 | コード重複 | 単位ミスの防止 |

### 学び

1. **分割の粒度は「5秒ルール」** — 目的のコードを見つけるまで5秒以上かかるなら分割する
2. **`URL.createObjectURL` は作成・置換・アンマウントの3箇所で解放** — Setで追跡すると漏れない
3. **`React.memo` は props の安定性とセット** — 子のインターフェース設計が鍵
4. **楽観的更新のカーソルはサーバーレスポンスからのみ更新** — クライアント時刻は信用しない
5. **関数名に単位を含める** — `formatDuration` ではなく `formatDurationMs`

### リファクタリングチェックリスト

- [ ] 1ファイル300行以上のコンポーネントはないか
- [ ] `URL.createObjectURL` に対応する `revokeObjectURL` があるか
- [ ] IDの生成にインデックスを使っていないか
- [ ] `React.memo` 適用先のpropsに毎回新しいオブジェクト/関数を渡していないか
- [ ] APIポーリングで全件再取得していないか
- [ ] 同じロジックが2箇所以上にコピーされていないか
