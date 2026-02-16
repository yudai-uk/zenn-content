---
title: "LINE管理画面をテーブル→2ペインチャットUIにリデザインした設計と実装Tips"
emoji: "💬"
type: "tech"
topics: ["nextjs", "react", "typescript", "firebase", "line"]
published: true
---

## はじめに

LINE公式アカウントの管理画面で「友だち一覧」をテーブル形式から**LINEアプリ風の2ペインチャットUI**にリデザインしました。左パネルに友だちリスト、右パネルにメッセージ履歴＋送信フォームという構成です。

:::message
**対象読者**
- Next.js + Firestore でチャット系UIを実装したい方
- テーブル→2ペインレイアウトへのリデザインを検討中の方
- ポーリングベースのメッセージ同期を設計したい方
:::

この記事で得られる知見:

- 2ペインレイアウトのレスポンシブ設計パターン
- Firestoreでメッセージ履歴を永続化する際の複合インデックス設計
- ポーリング時のリクエスト競合・ユーザー切り替えのガード手法
- 過去メッセージ読み込み時のスクロール位置保持テクニック
- LINE風メッセージバブルのCSS設計

:::message
本記事のコード例は説明に必要な部分のみ抜粋しています。import文やreturn文、変数宣言など自明な部分は省略しています。
:::

## 全体像

```
┌─────────────────────────────────────────────────┐
│  Admin Layout (sidebar)                         │
│  ┌──────────┬──────────────────────────────────┐│
│  │ Friends  │  Chat Panel                      ││
│  │ List     │  ┌────────────────────────────┐  ││
│  │          │  │ Chat Header (name, badges) │  ││
│  │ [Search] │  ├────────────────────────────┤  ││
│  │ [Filter] │  │                            │  ││
│  │          │  │  Message Area (scrollable) │  ││
│  │ User A ◀─┼──│  ┌──────┐                 │  ││
│  │ User B   │  │  │bubble│     ┌──────┐    │  ││
│  │ User C   │  │  └──────┘     │bubble│    │  ││
│  │ ...      │  │               └──────┘    │  ││
│  │          │  ├────────────────────────────┤  ││
│  │          │  │ Message Input [Send]       │  ││
│  │          │  └────────────────────────────┘  ││
│  └──────────┴──────────────────────────────────┘│
└─────────────────────────────────────────────────┘
```

**データフロー:**

1. 管理者が友だちリストからユーザーを選択
2. そのユーザーのメッセージ履歴をAPIから取得（Firestore）
3. 10秒間隔のポーリングで新着メッセージを自動取得
4. 管理者がメッセージ送信 → LINE Messaging API経由でpush → Firestoreに永続化

## 設計のポイント

### ポイント1: メッセージ永続化の設計

管理画面でメッセージ履歴を閲覧するには、送受信メッセージをFirestoreに保存する必要があります。

| 方式 | メリット | デメリット |
|------|---------|-----------|
| **LINE APIのメッセージ取得** | 実装不要 | そもそもLINE APIにそういう機能がない |
| **Webhook受信時にFirestoreへ保存** | リアルタイム、冪等性制御可能 | 過去データなし |
| **外部ストレージ(S3等)** | 大量データ対応 | オーバースペック、コスト増 |

LINE Messaging APIにはメッセージ履歴の取得APIがないため、**Webhook受信時にFirestoreへ保存**する方式を採用しました。

```
┌─────────┐   Webhook    ┌──────────┐    save     ┌───────────┐
│  LINE   │─────────────▶│ Backend  │────────────▶│ Firestore │
│  User   │              │ (Express)│             │line_messages│
└─────────┘              └──────────┘             └───────────┘
                              │                        ▲
                              │ push + save            │ read
                              ▼                        │
                         ┌──────────┐            ┌───────────┐
                         │  LINE    │            │  Admin    │
                         │  API     │            │  Frontend │
                         └──────────┘            └───────────┘
```

:::message
**冪等性の考慮**: LINE Webhookは再送される場合があります。inboundメッセージのドキュメントIDに `inbound_{webhookMessageId}` を使うことで、同じメッセージが重複保存されるのを防いでいます。
:::

### ポイント2: Firestoreの複合インデックス

メッセージ履歴の取得クエリは以下のような形になります:

```typescript
// "このユーザーのメッセージを新しい順で取得"
collection('line_messages')
  .where('lineUserId', '==', lineUserId)
  .orderBy('createdAt', 'desc')
  .limit(50)
```

`where` と `orderBy` が**異なるフィールド**を参照する場合、Firestoreは複合インデックスを要求します。これがないと `FAILED_PRECONDITION` エラーが発生します。

```json:firestore.indexes.json
{
  "indexes": [
    {
      "collectionGroup": "line_messages",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "lineUserId", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    }
  ]
}
```

```bash
# デプロイを忘れずに
firebase deploy --only firestore:indexes --project your-project-id
```

:::message
`firestore.indexes.json` にインデックスを追加しても、`firebase deploy` しなければ反映されません。コードレビューでは通るのに本番でエラーになる、というパターンに注意してください。
:::

### ポイント3: 2ペインレイアウトのレスポンシブ設計

デスクトップでは左右2ペイン、モバイルでは1画面ずつ切り替える設計にしました。

| 画面サイズ | 表示 | 操作 |
|-----------|------|------|
| **md以上** | 左右同時表示 | リストクリックで右パネル切り替え |
| **md未満** | 1パネルのみ | リスト→チャット / 戻るボタンで戻る |

```tsx
// 左パネル: モバイルでチャット表示中は非表示
<div className={`w-full md:w-80 border-r ${
  mobileShowChat ? "hidden md:flex md:flex-col" : "flex flex-col"
}`}>
  <FriendsList />
</div>

// 右パネル: モバイルでは選択時のみ表示
<div className={`flex-1 min-w-0 ${
  mobileShowChat ? "flex" : "hidden md:flex"
}`}>
  {selectedUser ? <ChatPanel /> : <EmptyState />}
</div>
```

state管理は `mobileShowChat` フラグ1つで制御します。ユーザー選択時に `true`、戻るボタンで `false` にするだけです。

## 実装Tips

### Tip 1: ポーリング時のユーザー切り替えガード

10秒間隔でメッセージを取得するポーリングでは、**ユーザー切り替え中にレスポンスが返ってくる**ケースを考慮する必要があります。

ユーザーAのメッセージを取得中にユーザーBに切り替えると、ユーザーAのレスポンスがユーザーBのチャット画面に混入してしまいます。

```tsx
function useMessageHistory(lineUserId: string | null) {
  const [messages, setMessages] = useState<Message[]>([]);
  const activeUserRef = useRef(lineUserId);
  activeUserRef.current = lineUserId;

  // ユーザー切り替え時にメッセージをクリア + 初回取得
  useEffect(() => {
    if (!lineUserId) { setMessages([]); return; }
    setMessages([]); // 前ユーザーのメッセージを破棄
    api.getMessageHistory(lineUserId, { limit: 50 }).then((data) => {
      if (activeUserRef.current !== lineUserId) return;
      setMessages(data.reverse());
    });
  }, [lineUserId]);

  // ポーリング: 最新メッセージを定期取得し、未表示分を追加
  useEffect(() => {
    if (!lineUserId) return;
    const targetUser = lineUserId;

    const poll = async () => {
      // ガード: ユーザーが切り替わっていたらスキップ
      if (activeUserRef.current !== targetUser) return;

      try {
        const data = await api.getMessageHistory(targetUser, { limit: 50 });

        // ガード: fetch中にユーザーが変わっていたら破棄
        if (activeUserRef.current !== targetUser) return;

        // APIはDESC順で返すのでreverseして時系列順にする
        const reversed = data.reverse();
        setMessages((prev) => {
          const existingIds = new Set(prev.map((m) => m.id));
          const newMessages = reversed.filter((m) => !existingIds.has(m.id));
          if (newMessages.length === 0) return prev;
          return [...prev, ...newMessages];
        });
      } catch (err) {
        console.error("Polling error:", err);
      }
    };

    const interval = setInterval(poll, 10_000);
    return () => clearInterval(interval);
  }, [lineUserId]);
}
```

:::message
`useRef` でアクティブなユーザーIDを追跡し、**リクエスト送信前**と**レスポンス受信後**の2箇所でガードするのがポイントです。`useEffect` のクリーンアップだけではin-flightリクエストのレスポンスは防げません。なお、A→B→Aのように同じユーザーに素早く戻る場合はIDベースのガードでは弾けないため、より厳密にはバージョンカウンターとの併用が有効です。
:::

### Tip 2: 過去メッセージ読み込み時のスクロール位置保持

「過去のメッセージを読み込む」ボタンで古いメッセージを先頭に追加すると、新しいDOMが上に挿入されてスクロール位置がずれます。

解決策は、**読み込み開始時にスクロール状態をスナップショット**し、**DOM更新直後（`useLayoutEffect`）にオフセットを計算して復元**するアプローチです。

```tsx
const scrollSnapshotRef = useRef<{
  scrollHeight: number;
  scrollTop: number;
} | null>(null);
const prevMessagesRef = useRef<Message[]>([]);

// 読み込み開始時にスナップショット
useEffect(() => {
  if (loadingOlder && containerRef.current) {
    scrollSnapshotRef.current = {
      scrollHeight: containerRef.current.scrollHeight,
      scrollTop: containerRef.current.scrollTop,
    };
  }
}, [loadingOlder]);

// DOM更新直後にスクロール位置を復元
useLayoutEffect(() => {
  const container = containerRef.current;
  const prev = prevMessagesRef.current;
  if (!container || messages.length <= prev.length) {
    prevMessagesRef.current = messages;
    return;
  }

  // 先頭のメッセージIDが変わっていたら prepend と判定
  const isPrepend = prev[0] && messages[0] && messages[0].id !== prev[0].id;

  if (isPrepend && scrollSnapshotRef.current) {
    const { scrollHeight: oldH, scrollTop: oldTop } = scrollSnapshotRef.current;
    container.scrollTop = oldTop + (container.scrollHeight - oldH);
    scrollSnapshotRef.current = null;
  }

  prevMessagesRef.current = messages;
}, [messages]);
```

:::message
`MutationObserver` で同じことを実現するアプローチもありますが、ローディングスピナーの表示切り替えなど**意図しないDOM変更**でobserverが発火してしまう問題があります。`useLayoutEffect` + スナップショットの方が確実です。
:::

### Tip 3: リクエスト競合の防止（バージョニング）

フィルタ条件を素早く変更すると、古いリクエストのレスポンスが新しいリクエストより後に届き、UIが古い結果で上書きされることがあります。

```tsx
function useDataList(filterTags: string[]) {
  const [data, setData] = useState<User[]>([]);
  const fetchVersionRef = useRef(0);
  // ref経由で最新値を参照し、useCallbackの再生成を防ぐ
  const filterTagsRef = useRef(filterTags);
  filterTagsRef.current = filterTags;

  const fetchData = useCallback(async () => {
    const version = ++fetchVersionRef.current;

    const result = await api.listUsers({ tags: filterTagsRef.current });

    // 古いリクエストのレスポンスを破棄
    if (version !== fetchVersionRef.current) return;

    setData(result.data);
  }, []);
}
```

:::message
`filterTags` を直接 `useCallback` の依存配列に入れると、タグ変更のたびにコールバックが再生成されます。`useRef` 経由で最新値を参照することで、コールバックの参照を安定させつつ常に最新のフィルタ条件でリクエストできます。
:::

`AbortController` を使う方法もありますが、バージョンカウンターの方がシンプルで、loading状態の管理とも相性が良いです。

### Tip 4: LINE風メッセージバブルのCSS

LINE風のバブルデザインは、方向（送信/受信）によって配色と配置を切り替えるだけで実現できます。

```tsx
<div className={`flex items-end gap-1.5 ${
  isOutbound ? "justify-end" : "justify-start"
}`}>
  <div className={`px-3 py-2 rounded-2xl text-sm max-w-[70%] ${
    isOutbound
      ? "bg-[#06C755] text-white rounded-br-md"   // LINE green
      : "bg-white text-gray-900 border rounded-bl-md"
  }`}>
    {message.content}
  </div>
</div>
```

ポイントは `rounded-2xl` で全体を丸くしつつ、**吹き出しの根元だけ `rounded-br-md`（送信）/ `rounded-bl-md`（受信）で角を立てる**ことです。LINEの `#06C755` はブランドカラーとして使えます。

## ハマりポイント

### Firestoreインデックスのデプロイ忘れ

`firestore.indexes.json` に複合インデックスを追加して型チェックもlintも通り、コードレビューもOK。しかし実行すると:

```
Error: 9 FAILED_PRECONDITION: The query requires an index.
```

**原因**: `firebase deploy --only firestore:indexes` を実行していなかった。

```diff
# CI/CDパイプラインに入れておくと安心
+ firebase deploy --only firestore:indexes --project $PROJECT_ID
```

### exactOptionalPropertyTypesとの戦い

TypeScriptの `exactOptionalPropertyTypes: true` が有効な環境では、optional propertyに `undefined` を明示的に代入できません。`source?: string` と宣言したプロパティには、値を省略するか具体的な文字列を渡すかの二択で、`source: undefined` は型エラーになります。

```typescript
interface Message {
  source?: 'admin' | 'scenario' | 'broadcast';  // optional
}

// NG: undefined を明示的に代入すると型エラー
const value: Message['source'] | undefined = Math.random() > 0.5 ? 'admin' : undefined;
const msg: Message = {
  source: value,  // Type 'string | undefined' is not assignable
};

// OK: source を省略する
const msg: Message = {};  // source フィールドなし

// OK: 引数の型で undefined を除外して受け取る
function sendMessage(
  text: string,
  source: NonNullable<Message['source']>  // 'admin' | 'scenario' | 'broadcast'
) {
  const msg: Message = { source };  // 必ず値がある
}
```

## まとめ

### 設計判断の一覧

| 判断 | 選択 | 理由 |
|------|------|------|
| メッセージ永続化 | Webhook時にFirestore保存 | LINE APIに履歴取得機能がない |
| ドキュメントID | `inbound_{webhookMsgId}` | Webhook再送時の冪等性確保 |
| リアルタイム更新 | 10秒ポーリング | Firestoreリスナーより実装がシンプル、管理画面の用途に十分 |
| レスポンシブ | `md:` ブレークポイントで切り替え | 友だちリストとチャットの1パネル切り替え |
| 競合防止 | バージョンカウンター + activeUserRef | AbortControllerより状態管理と相性が良い |

### 学び

1. **Firestoreの複合インデックスはコードと一緒にデプロイする**。`firestore.indexes.json` の変更はCI/CDに組み込むと安全
2. **ポーリングの非同期ガードは2段階**。リクエスト前（staleチェック）+ レスポンス後（activeチェック）の両方が必要
3. **スクロール位置保持はuseLayoutEffect**。MutationObserverは関係ないDOM変更でも発火するため不安定
4. **2ペインのレスポンシブは状態フラグ1つで十分**。複雑なメディアクエリよりTailwindの `hidden md:flex` + booleanフラグの方がシンプル

### 実装チェックリスト

- [ ] Firestoreの複合インデックスを定義しデプロイしたか
- [ ] Webhook再送時の冪等性を確保しているか
- [ ] ポーリング中のユーザー切り替えでデータが混入しないか
- [ ] 過去メッセージ読み込みでスクロールが飛ばないか
- [ ] モバイルで1パネル切り替えが正しく動作するか
- [ ] メッセージ送信後に楽観的UIが即座に反映されるか
