---
title: "LINE Messaging APIで送信者のアイコン・名前を切り替える機能を実装する"
emoji: "🎭"
type: "tech"
topics: ["line", "typescript", "nextjs", "cleanarchitecture", "firebase"]
published: true
---

## はじめに

LINE公式アカウントからメッセージを送るとき、送信者のアイコンと表示名をメッセージごとにカスタマイズできる機能を実装しました。Lステップなどのツールにある「送信者アイコン変更」に相当する機能です。

:::message
**対象読者**
- LINE Messaging APIの`sender`プロパティを使いたい方
- メッセージ単位でsenderを切り替える設計パターンを知りたい方
- Clean Architecture + DI（Inversify）のバックエンドで新機能を追加する流れを知りたい方
:::

この記事で得られる知見：

- LINE Messaging APIの**アイコン・表示名カスタマイズ**の仕様と制約
- メッセージ単位のsender指定を支える**型設計とデータフロー**
- senderProfileId → MessageSender の**遅延解決パターン**
- チャネル境界を跨がせない**セキュリティ設計**
- 実装中に踏んだ**ハマりポイント3選**

## 全体像

```
┌──────────────┐  senderProfileId   ┌───────────────┐
│  管理画面      │ ─── API call ────▶ │  Backend       │
│  (Next.js)    │  + messages[]      │  (Express)     │
└──────────────┘                     └───────┬───────┘
                                             │ 1. senderProfileId → sender解決
                                             │ 2. messages に sender を付与
                                             │ 3. push / broadcast
                                     ┌───────▼───────┐
                                     │ LINE Platform   │
                                     │                 │
                                     │ "田中 from      │
                                     │  '公式アカウント'" │
                                     └─────────────────┘
```

**ポイント**: フロントエンドは `senderProfileId`（参照ID）だけを送り、バックエンドが送信直前にプロフィール情報を解決して LINE API の `sender` プロパティに変換します。

## LINE Messaging APIのsender仕様

LINE Messaging APIでは、メッセージオブジェクトに `sender` プロパティを指定することで、送信者のアイコンと表示名をカスタマイズできます。

```json
{
  "type": "text",
  "text": "こんにちは！",
  "sender": {
    "name": "田中",
    "iconUrl": "https://example.com/icon.png"
  }
}
```

:::message
**LINE APIの制約**
- `name`: 最大20文字
- `iconUrl`: HTTPS URL、**PNG形式のみ**、1:1比率、1MB以下
- 表示形式: 必ず **「表示名 from 'アカウント名'」** と表示される（この `from '...'` 部分は消せない）
- メッセージオブジェクトの種類に制限なし（text, image, video, audio, flex 等すべて対応）
- push / multicast / narrowcast / broadcast / reply すべてのAPIで利用可能
:::

`from 'アカウント名'` が常に付くのはLINEプラットフォーム側の仕様で、公式アカウントの識別・なりすまし防止のためです。これを非表示にする方法はありません。

## 設計のポイント

### ポイント1: 保存時はID参照、送信時に実体解決

送信者プロフィールの扱い方として2つの方式を比較しました：

| 方式 | メリット | デメリット |
|------|---------|-----------|
| **メッセージに `name` + `iconUrl` を埋め込む** | 送信時の解決が不要 | プロフィール変更が既存メッセージに反映されない |
| **`senderProfileId` で参照し送信時に解決** | プロフィール変更が即座に反映 | 送信時にDB参照が必要 |

**後者を採用**しました。ブロードキャストのテンプレートを保存した後にプロフィール画像を差し替えたい、というユースケースに対応するためです。

```typescript
// Domain entity — メッセージには参照IDだけ持つ
interface CampaignMessage {
  type: "text" | "image" | "video" | "audio" | "flex";
  senderProfileId?: string; // Save reference, resolve at send time
  // ... other fields
}
```

### ポイント2: senderMapによる重複排除解決

1つのブロードキャストには最大5件のメッセージがあり、それぞれ異なる送信者を設定できます。メッセージごとにDBを叩くのではなく、ユニークなIDを抽出して重複を排除し、必要最小限のクエリで解決します。

```typescript
// Application service
async function buildSenderMap(
  messages: CampaignMessage[],
  channelId?: string
): Promise<Record<string, MessageSender>> {
  // Collect unique senderProfileIds to avoid duplicate queries
  const ids = new Set<string>();
  for (const msg of messages) {
    if (msg.senderProfileId) ids.add(msg.senderProfileId);
  }
  if (ids.size === 0) return {};

  // Resolve all unique IDs (at most 5 for broadcast messages)
  const entries = await Promise.all(
    [...ids].map(async (id) => {
      const profile = await profileRepo.findById(id);
      if (profile && (!channelId || profile.channelId === channelId)) {
        return [id, { name: profile.name, iconUrl: profile.iconUrl }] as const;
      }
      return null;
    })
  );
  const resolved = entries.filter(
    (e): e is NonNullable<typeof e> => e != null
  );
  return Object.fromEntries(resolved);
}
```

```typescript
// Message converter — sender付与
function toLineMessages(
  messages: CampaignMessage[],
  senderMap: Record<string, MessageSender> = {}
): LineMessage[] {
  return messages.map((msg) => {
    const sender = msg.senderProfileId
      ? senderMap[msg.senderProfileId]
      : undefined;

    if (msg.type === "text") {
      return { type: "text", text: msg.text, ...(sender && { sender }) };
    }
    if (msg.type === "image") {
      return {
        type: "image",
        originalContentUrl: msg.originalContentUrl,
        previewImageUrl: msg.previewImageUrl,
        ...(sender && { sender }),
      };
    }
    // ... other types
  });
}
```

### ポイント3: チャネル境界の適用

マルチチャネル運用（1つのシステムで複数のLINE公式アカウントを管理）に対応するため、送信者プロフィールにはチャネルスコープを設けています。

```typescript
// Repository — save時にチャネル境界チェック
async function save(
  input: SenderProfileInput,
  channelId: string
): Promise<SenderProfile> {
  if (input.id) {
    const existing = await this.findById(input.id);
    // Prevent overwriting another channel's profile
    if (existing && existing.channelId !== channelId) {
      throw new Error("Cannot update profile of another channel");
    }
  }
  // ... save logic
}
```

:::message
**なぜチャネル境界が重要か**
マルチテナント的な運用では、チャネルAの管理者がチャネルBのプロフィールを読み込んだり上書きしたりできてはいけません。`findById` → `save` の一連の操作で、必ずチャネルIDの一致を確認します。
:::

## 実装Tips

### Tip 1: spread演算子によるフィールドリークを防ぐ

メッセージ変換時に `{ ...msg, sender }` としてしまうと、`senderProfileId` などの内部フィールドがLINE APIに送信されてしまいます。

```diff
  // ❌ Bad: internal fields leak to LINE API
- return { ...msg, ...(sender && { sender }) };
+ // ✅ Good: explicitly construct only LINE API fields
+ return {
+   type: "text",
+   text: msg.text,
+   ...(sender && { sender }),
+ };
```

LINE APIは未知のフィールドを無視してくれることが多いですが、将来的にバリデーションが厳しくなる可能性もあります。明示的にフィールドを構築するのが安全です。

### Tip 2: Firestoreの複合インデックス問題を回避する

`where("channelId", "==", id).orderBy("updatedAt", "desc")` のようなクエリは、Firestoreの複合インデックスが必要です。新しいコレクションを追加するたびにインデックスを作成するのは手間なので、レコード数が少ない場合はインメモリソートで回避できます。

```typescript
// Firestore query — avoid composite index requirement
async function list(channelId: string): Promise<SenderProfile[]> {
  const snapshot = await db
    .collection("sender_profiles")
    .where("channelId", "==", channelId)
    .get();

  const profiles = snapshot.docs.map((doc) => doc.data() as SenderProfile);
  // Sort in memory instead of orderBy (avoids composite index)
  // Note: updatedAt is stored as ISO 8601 string, so localeCompare works
  return profiles.sort(
    (a, b) => (b.updatedAt ?? "").localeCompare(a.updatedAt ?? "")
  );
}
```

:::message
レコード数が数十〜数百程度であればインメモリソートで十分です。数千件以上になる場合は複合インデックスを作成した方がパフォーマンス上有利です。
:::

### Tip 3: SenderProfileSelectorの条件付き表示

プロフィールが1件も登録されていない状態でセレクターを表示すると、ユーザーを混乱させます。「プロフィールが存在するときだけ表示」にすることでUIをクリーンに保ちます。

```tsx
function SenderProfileSelector({
  value,
  onChange,
}: {
  value?: string;
  onChange: (id: string | undefined) => void;
}) {
  const { profiles, loaded } = useSenderProfiles();

  // Hide completely when no profiles exist
  if (!loaded || profiles.length === 0) return null;

  return (
    <select
      value={value ?? ""}
      onChange={(e) => onChange(e.target.value || undefined)}
    >
      <option value="">デフォルト（アカウント設定）</option>
      {profiles.map((p) => (
        <option key={p.id} value={p.id}>
          {p.name}
        </option>
      ))}
    </select>
  );
}
```

## ハマりポイント

### 1. テキスト編集時にsenderProfileIdが消える

ブロードキャストのメッセージコンポーザーで、テキスト入力のたびにメッセージオブジェクトを再生成していたところ、`senderProfileId` が消失する問題が発生しました。

```diff
  const handleTextChange = (text: string) => {
    setMessages((prev) =>
      prev.map((msg, i) =>
-       i === index ? { type: "text", text } : msg
+       i === index
+         ? { type: "text", text, senderProfileId: msg.senderProfileId }
+         : msg
      )
    );
  };
```

`prev` コールバック内の `msg` から直接 `senderProfileId` を取得することで、stale stateの問題も回避しています。

同様に、画像アップロードの状態遷移（uploading → complete）でも `senderProfileId` を引き継ぐ必要があります。状態を再構築するすべてのパスで、付随フィールドが保持されているか確認しましょう。

### 2. channelId未指定時のsender解決

個別メッセージ送信でチャネルIDが省略された場合（デフォルトチャネルを使うケース）、`senderProfileId` だけ指定されると解決時にチャネル境界チェックが行えません。

```typescript
// Resolve effective channelId before sender resolution
const effectiveChannelId =
  channelId ?? (await channelConfigService.getDefault())?.id;

if (senderProfileId && !effectiveChannelId) {
  throw new Error("Cannot resolve sender without channel context");
}

const sender = await senderProfileService.resolveSender(
  senderProfileId,
  effectiveChannelId
);
```

### 3. Firestoreの`where` + `orderBy`で500エラー

`list()` メソッドに `where` + `orderBy` を使ったところ、複合インデックスがないため500エラーが発生しました。解決策は Tip 2 を参照してください。

:::details エラーメッセージの例
```
Error: 9 FAILED_PRECONDITION: The query requires an index.
You can create it here: https://console.firebase.google.com/v1/...
```
:::

## まとめ

### 技術選定・設計判断の一覧

| 判断 | 選択 | 理由 |
|------|------|------|
| sender情報の保持方式 | IDで参照、送信時に解決 | プロフィール変更の即時反映 |
| sender解決の粒度 | メッセージ単位 | 1配信内で異なるsenderを使いたい |
| sender解決のタイミング | 重複排除（senderMap） | ユニークIDだけ解決しDB呼び出し最小化 |
| チャネル境界 | 全操作（CRUD + resolve）で適用 | マルチチャネルのセキュリティ |
| Firestore一覧取得 | `where` のみ + インメモリソート | 複合インデックス不要 |
| Selector表示条件 | プロフィール0件なら非表示 | UIのノイズ軽減 |

### 学び

1. **LINE APIの `from '...'` は消せない** — ユーザーへの事前説明が必要
2. **spread演算子でのオブジェクト構築は内部フィールドリークの温床** — LINE APIに送るオブジェクトは明示的に構築する
3. **状態の再構築時に付随フィールドが消える問題は地味だが頻出** — テスト時にsender指定→テキスト編集→sender消失のパターンを必ず確認
4. **Firestoreの複合インデックスは小規模コレクションならインメモリソートで回避可能**

### 実装チェックリスト

- [ ] `sender` プロパティの `name` が20文字以内か
- [ ] `iconUrl` がHTTPS・PNG・1:1比率・1MB以下か
- [ ] メッセージ変換時にspreadで内部フィールドがリークしていないか
- [ ] チャネル境界チェックがCRUD全操作 + resolve時に適用されているか
- [ ] 状態更新のすべてのパスで `senderProfileId` が保持されているか
- [ ] プロフィール0件時にSelectorが非表示になるか

## 参考

- [アイコンと表示名をカスタマイズする | LINE Developers](https://developers.line.biz/ja/docs/messaging-api/icon-nickname-switch/)
- [Messaging APIの概要 | LINE Developers](https://developers.line.biz/ja/docs/messaging-api/overview/)
