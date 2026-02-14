---
title: "LINE Messaging APIでシナリオ送信が「0人に送信」になる問題の原因と対策"
emoji: "📱"
type: "tech"
topics: ["line", "messagingapi", "nodejs", "firestore", "webhook"]
published: true
---

## はじめに

LINE Messaging APIを使ったアンケートシナリオ配信機能を開発中、**送信ボタンを押すと「0人に送信しました」と表示される**問題に遭遇しました。

調査の結果、**チャネル間のユーザーID不整合**、**未デプロイコードの見落とし**、**FirestoreのundefinedValue制約**という3つの原因が連鎖していました。

:::message
**対象読者**: LINE Messaging APIでPush Messageやシナリオ配信を実装している方。特に「送信は成功するのにユーザーに届かない」問題に心当たりのある方。
:::

## システム構成

```
┌─────────────┐    ┌──────────────────┐    ┌─────────────┐
│  Next.js 15  │───▶│  Express + TS     │───▶│  LINE API   │
│  管理画面     │    │  (Clean Arch)     │    │  Push/Webhook│
└─────────────┘    └──────┬───────────┘    └─────────────┘
                          │
                   ┌──────▼───────┐
                   │  Firestore    │
                   │  lineUsers    │
                   └──────────────┘
```

**シナリオ送信の流れ:**

1. 管理画面から「送信」ボタンを押す
2. バックエンドが `lineUsers` コレクションから対象ユーザーを取得
3. 各ユーザーに `pushToUser` でメッセージを個別送信

## 問題1: LINE APIが全件400エラーを返す

### 症状

シナリオ送信を実行すると、`lineUsers` からユーザーは取得できるが、**全ユーザーへの送信が400エラー**で失敗する。

```
LINE API /message/push failed: 400 {"message":"Failed to send messages"}
```

一方、**ブロードキャスト配信は正常に動作する**。この違いが原因特定のヒントになった。

### 原因: チャネル間のユーザーID不整合

Firestoreの `lineUsers` に登録されていた2ユーザーを確認すると、旧システムの**別チャネル**所属だった。

```json
// Firestoreのデータ
{
  "userId": "U88559da...",
  "channelName": "Main",        // ← 旧チャネル
  "latestEventType": "follow"
}
```

```json
{
  "userId": "Uea2cc84...",
  "channelName": "Coaching",    // ← 旧チャネル
  "latestEventType": "follow"
}
```

LINE Messaging APIでは、**アクセストークンとユーザーIDは同一チャネルでなければ通信できない**。新チャネル「テストチャンネル」のトークンで旧チャネルのユーザーに送信しようとして400エラーになっていた。

:::message
**ブロードキャストが動く理由**: `broadcast` APIはチャネルの全フォロワーに送信するため、`lineUsers` コレクションを参照しない。一方 `push` APIは個別のユーザーIDを指定するため、チャネル不一致で失敗する。
:::

### 修正: channelIdフィールドの追加とフィルタリング

`LineUser` エンティティに `channelId` を追加し、送信時にチャネルでフィルタリング。

```diff typescript:domain/line/entities/LineUser.ts
 export interface LineUser {
   lineUserId: string;
+  channelId?: string;
   displayName?: string;
   // ...
 }
```

```typescript:application/line/LineWebhookService.ts
// 送信時にchannelIdでフィルタ
const channelConfig = await this.channelConfigRepo.findDefault();
let users = await this.userRepo.listAll();
users = users.filter(u => u.channelId === channelConfig.lineChannelId);
```

ポイントは2つ:

- **Webhook受信時に `channelId` を記録** — フォローイベントでユーザーを登録する際にチャネルIDを保存
- **送信時にフィルタ** — デフォルトチャネルに所属するユーザーのみを対象にする

## 問題2: Webhookでユーザーが登録されない

### 症状

`channelId` フィルタを実装後、LINE公式アカウントをブロック→再追加しても、`lineUsers` に**新しいデータが反映されない**。

### 調査: Cloud Runログの分析

**Step 1: Webhookリクエストの到達確認**

```bash
gcloud logging read 'resource.type="cloud_run_revision"...'
```

```
04:11:25 POST /api/line/webhook -> 200 ✅
04:11:55 POST /api/line/webhook -> 200 ✅
04:12:03 POST /api/line/webhook -> 200 ✅
```

Webhookは **200で成功** しているのに、Firestoreのデータは更新されていない。

**Step 2: 200レスポンスの正体**

Webhook処理は `events` 配列をループし、空でも正常終了する。

```typescript
async processWebhook(payload: LineWebhookPayload): Promise<void> {
  for (const event of payload.events) {  // events が空 → ループスキップ
    // ...
  }
  // 正常終了 → 200
}
```

200レスポンスは全て **LINE Developers Consoleの「検証」ボタン**からのテストリクエスト（空の `events` 配列）だった。

**Step 3: 根本原因の特定**

```bash
git status  # → 多数のuncommitted changes
```

Cloud Runには `channelId` 対応の**新コードがデプロイされていなかった**。旧コードのWebhook処理では `channelId` が記録されないため、フィルタで全員除外されていた。

### 修正: ngrokでローカルテスト

Cloud Runへのデプロイ前に、ngrokでローカルバックエンドを直接テスト。

```bash
ngrok http 8080
# → https://xxxx.ngrok-free.dev
```

LINE Developers ConsoleでWebhook URLをngrokのURLに変更し、ブロック→再追加 → ユーザーが `channelId` 付きで登録され、シナリオ送信が成功。

## 問題3: アンケート回答後に次の質問が送信されない

### 症状

シナリオの最初の質問は正常に送信されるが、ユーザーが選択肢をタップしても**次の質問が届かない**。

### 原因: FirestoreのundefinedValue制約

バックエンドログに以下のエラーが3回連続で出力されていた。

:::details エラーログ全文
```
Webhook processing error: Error: Value for argument "data" is not a valid
Firestore document. Cannot use "undefined" as a Firestore value
(found in field "displayName").
```
:::

選択肢タップ → postback イベント → `upsertUser` → `LineUser` オブジェクトの `displayName` が `undefined` のまま Firestore に書き込み → **エラーで処理が中断** → 後続の次質問送信が実行されない。

**なぜ `displayName` が `undefined` なのか?**

Webhook の `follow` イベントではプロフィール情報を取得しないため、`displayName` は未設定のまま。TypeScript上は `string | undefined` として有効だが、Firestoreの `set({ merge: true })` は **`undefined` 値を許容しない**。

### 修正: upsertでundefined値を除去

```diff typescript:infrastructure/line/repositories/FirestoreLineUserRepository.ts
 async upsert(user: LineUser): Promise<LineUser> {
-  await this.getCollection().doc(user.lineUserId).set(user, { merge: true });
+  const data: Record<string, unknown> = {};
+  for (const [key, value] of Object.entries(user)) {
+    if (value !== undefined) data[key] = value;
+  }
+  await this.getCollection().doc(user.lineUserId).set(data, { merge: true });
   return user;
 }
```

## 補足: Get Follower IDs APIの制限

既存フォロワーの一括同期のため、`GET /v2/bot/followers/ids` APIも実装したが、テスト時に `403` エラーが発生。

```
LINE API /followers/ids failed: 403
{"message":"Access to this API is not available for your account"}
```

:::message alert
**注意**: `Get Follower IDs` APIは**認証済みアカウント（Verified / Premium）でのみ利用可能**。未認証のLINE公式アカウント（フリープラン）では使用できない。
:::

未認証アカウントでは、Webhookベースでの登録（友だち追加時に自動記録）を基本設計にすべき。

## まとめ

| 問題 | 原因 | 修正 |
|------|------|------|
| 全件400エラー | 旧チャネルのユーザーIDに新チャネルのトークンで送信 | `channelId` フィルタを追加 |
| Webhookでユーザー未登録 | 新コードが未デプロイ、200は検証リクエスト | ngrokでローカルテスト後デプロイ |
| 回答後の次質問が届かない | Firestoreに `undefined` 値を書き込みエラー | `upsert` で `undefined` 値を除去 |

### 学び

1. **LINE Messaging APIのチャネル間の壁** — アクセストークンとユーザーIDは同一チャネルでなければ通信できない。複数チャネルを運用する場合は `channelId` での管理が必須
2. **Firestoreの `undefined` 制約** — TypeScriptの `undefined` はFirestoreでは無効値。`set({ merge: true })` に渡す前に必ず除去する
3. **200レスポンスの罠** — Webhook処理が200を返しても、実際にイベントが処理されたとは限らない。空の `events` 配列でも正常終了する
4. **ngrokでのWebhookテスト** — ローカル開発でLINE Webhookをテストするには ngrok が便利。デプロイ前に動作確認できる
5. **Get Follower IDs APIの制限** — 無料アカウントでは使えない。Webhookベースでの登録を基本設計にすべき

### LINE Messaging APIデバッグチェックリスト

- [ ] `push` 送信のユーザーIDは、使用するアクセストークンのチャネルに所属しているか
- [ ] Webhook URLは正しく設定され、「Webhookの利用」トグルがONになっているか
- [ ] Firestore書き込み時に `undefined` 値を渡していないか
- [ ] Cloud Run（本番環境）に最新コードがデプロイされているか
- [ ] Webhookの200レスポンスが実際のイベント処理結果か、空の検証リクエストか
