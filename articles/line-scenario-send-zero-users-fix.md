---
title: "LINE Messaging APIでシナリオ送信が「0人に送信」になる問題の原因と対策"
emoji: "📱"
type: "tech"
topics: ["line", "messagingapi", "nodejs", "firestore", "webhook"]
published: true
---

## はじめに

LINE Messaging APIを使ったシナリオ配信機能を開発中、**送信ボタンを押すと「0人に送信しました」と表示される**問題に遭遇しました。

調査の結果、原因は**複数のチャネル間でのユーザー管理の不整合**と**Firestoreへの書き込み時のundefined値**という2つの問題でした。

本記事では、原因の特定から修正までのプロセスを共有します。

## システム構成

- **バックエンド**: Express + TypeScript + Inversify（Clean Architecture）
- **データベース**: Firestore
- **LINE連携**: LINE Messaging API（Webhook + Push Message）
- **フロントエンド**: Next.js 15（管理画面）

シナリオ送信の流れ:
1. 管理画面から「送信」ボタンを押す
2. バックエンドが `lineUsers` コレクションから対象ユーザーを取得
3. 各ユーザーに `pushToUser` でメッセージを送信

## 問題1: 全件400エラー

### 症状

シナリオ送信時、`lineUsers` からユーザーは取得できるが、LINE APIが全件 **400エラー** を返す。

```
LINE API /message/push failed: 400 {"message":"Failed to send messages"}
```

### 原因

`lineUsers` に登録されていた2ユーザーは、旧システムの**別チャネル**（"Main", "Coaching"）に所属していました。新チャネルのアクセストークンでは、別チャネルのユーザーにメッセージを送信できません。

```json
// Firestoreのデータ — channelName が旧チャネル
{
  "userId": "U88559da...",
  "channelName": "Main",        // ← 旧チャネル
  "latestEventType": "follow"
}
```

LINE Messaging APIでは、**アクセストークンとユーザーIDのチャネルが一致**しないと送信できません。

### 対策: channelIdによるフィルタリング

`LineUser` エンティティに `channelId` フィールドを追加し、送信時にチャネルでフィルタリングするようにしました。

```typescript
// LineUser エンティティ
export interface LineUser {
  lineUserId: string;
  channelId?: string;  // 追加
  // ...
}
```

```typescript
// 送信時にchannelIdでフィルタ
const channelConfig = await this.channelConfigRepo.findDefault();
let users = await this.userRepo.listAll();
users = users.filter(u => u.channelId === channelConfig.lineChannelId);
```

## 問題2: Webhookでユーザーが登録されない

### 症状

`channelId` フィルタを追加した後、Webhook URLを設定してLINE公式アカウントをブロック→再追加しても、`lineUsers` にデータが反映されない。

### 調査プロセス

**Step 1: Cloud Runのログ確認**

```bash
gcloud logging read 'resource.type="cloud_run_revision"...'
```

Webhookリクエストは **200で成功** しているのに、Firestoreのデータが更新されていない。

**Step 2: デプロイ状態の確認**

```bash
git status  # → 多数のuncommitted changes
```

Cloud Runには `channelId` 対応の**新コードがデプロイされていない**ことが判明。旧コードのWebhook処理では `channelId` が記録されないため、フィルタで全員除外されていました。

**Step 3: ローカルでのテスト**

ngrokを使ってローカルバックエンドをLINE Webhookに接続:

```bash
ngrok http 8080
# → https://xxxx.ngrok-free.dev
```

LINE Developers ConsoleでWebhook URLをngrokのURLに変更し、ブロック→再追加→ ユーザーが `channelId` 付きで登録され、シナリオ送信が成功！

## 問題3: アンケート回答後に次の質問が送信されない

### 症状

シナリオは正常に送信されるが、ユーザーが選択肢をタップしても**次の質問が送信されない**。

### 原因

バックエンドログに以下のエラー:

```
Cannot use "undefined" as a Firestore value (found in field "displayName")
```

選択肢タップ時の postback イベントで `upsertUser` が呼ばれ、`LineUser` オブジェクトの `displayName` が `undefined` のまま Firestore に書き込もうとしてエラーになっていました。

Firestoreは `{ merge: true }` の `set` で **`undefined` 値を許容しません**。

### 修正

`upsert` メソッドで `undefined` 値を除去してからFirestoreに書き込むようにしました。

```typescript
async upsert(user: LineUser): Promise<LineUser> {
  // Strip undefined values to avoid Firestore rejection
  const data: Record<string, unknown> = {};
  for (const [key, value] of Object.entries(user)) {
    if (value !== undefined) data[key] = value;
  }
  await this.getCollection().doc(user.lineUserId).set(data, { merge: true });
  return user;
}
```

## 追加改善: フォロワー同期機能

新チャネルの既存フォロワーを `lineUsers` に一括登録するため、フォロワー同期機能も実装しました。

```typescript
// LINE Get Follower IDs APIでページネーション
async getFollowerIds(start?: string): Promise<{ userIds: string[]; next?: string }> {
  const params = new URLSearchParams();
  if (start) params.set('start', start);
  return this.request<{ userIds: string[]; next?: string }>(
    `/followers/ids${query ? `?${query}` : ''}`,
    { method: 'GET' }
  );
}
```

:::message alert
**注意**: `GET /v2/bot/followers/ids` APIは**認証済みアカウント（Verified / Premium）でのみ利用可能**です。未認証のLINE公式アカウントでは `403 Access to this API is not available for your account` エラーになります。
:::

未認証アカウントの場合は、Webhookベースでの登録（友だち追加時に自動記録）に頼る必要があります。

## まとめ

| 問題 | 原因 | 修正 |
|------|------|------|
| 全件400エラー | 旧チャネルのユーザーに新チャネルのトークンで送信 | `channelId` フィルタを追加 |
| Webhookでユーザー未登録 | 新コード未デプロイ + ngrokでローカルテスト | Webhook処理に `channelId` 記録を追加 |
| 回答後の次質問が送信されない | Firestoreに `undefined` 値を書き込もうとしてエラー | `upsert` で `undefined` 値を除去 |

### 学んだこと

1. **LINE Messaging APIのチャネル間の壁**: アクセストークンとユーザーIDは同一チャネルでないと通信できない。複数チャネルを扱う場合は `channelId` での管理が必須
2. **Firestoreの `undefined` 制約**: TypeScriptの `undefined` はFirestoreでは無効値。`set({ merge: true })` 前に必ず除去する
3. **ngrokでのWebhookテスト**: ローカル開発でLINE Webhookをテストするには ngrok が便利。デプロイ前に動作確認できる
4. **Get Follower IDs APIの制限**: 無料アカウントでは使えない。Webhookベースでの登録を基本設計にすべき
