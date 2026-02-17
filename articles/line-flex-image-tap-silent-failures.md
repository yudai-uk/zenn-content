---
title: "LINE Flex Messageのタップが反応しない — 200 OKなのに動かない3つの原因"
emoji: "🔇"
type: "tech"
topics: ["LINE", "MessagingAPI", "Node.js", "webhook", "デバッグ"]
published: true
---

## はじめに

LINE Messaging API で Flex Message（タップ可能な画像）をブロードキャスト配信し、タップしたらアンケートを開始する機能を実装しました。配信は成功、webhook も 200 OK を返す。**しかしアンケートが始まらない。**

エラーログもなし。すべてが正常に見える中で、3つの「サイレント失敗」を突き止めた記録です。

ここで重要な前提として、**webhook の 200 OK は「LINE Platform がイベントを受信した」ことを意味するだけで、「業務ロジックが成功した」ことは意味しません**。この区別を見落とすと、今回のようなサイレント失敗に気づけなくなります。

:::message
**対象読者**
- LINE Messaging API で postback イベントを扱っている方
- webhook が 200 OK を返すのに期待通りの動作をしない問題に悩んでいる方
- Flex Message でインタラクティブな機能を実装している方
:::

## システム構成

```
┌──────────┐     POST /broadcast     ┌──────────────┐
│  管理画面  │ ──────────────────────→ │  バックエンド   │
│ (Next.js) │                        │  (Express)    │
└──────────┘                        └──────┬───────┘
                                           │ Flex Message 送信
                                           ▼
                                    ┌──────────────┐
                                    │  LINE Platform │
                                    └──────┬───────┘
                                           │ ユーザーが画像をタップ
                                           ▼
┌──────────────┐    postback event   ┌──────────────┐
│  アンケート    │ ←───────────────── │  Webhook      │
│  フロー実行   │                    │  エンドポイント │
└──────────────┘                    └──────────────┘
```

## 問題1: postback の prefix が未ハンドル

:::message
**前提**: この機能では Flex Message の画像に `postback` アクション（`uri` や `message` アクションではなく）を設定しています。ユーザーが画像をタップすると、LINE Platform から webhook に `postback` イベントが送信されます。
:::

### 症状

Flex Message の画像をタップしても何も起きない。webhook エンドポイントのログを確認すると、POST リクエストは届いており、すべて **200 OK** を返している。エラーログは一切なし。

### 原因

webhook ハンドラが postback イベントを処理する際、`data` フィールドの prefix でハンドラを振り分けていました。

```typescript
// webhook ハンドラの postback 処理（修正前）
if (event.type === 'postback' && event.postback?.data) {
  if (data.startsWith('send_msg:')) {
    await this.handleSendMessage(userId, data);
    continue;
  }
  if (data.startsWith('survey_answer:')) {
    await this.handleSurveyAnswer(userId, data);
    continue;
  }
  // ← ここで flex_tap: が来ても何も処理されない
  // → continue もないので次の判定に落ちるが、該当なしでスキップ
}
```

Flex Message のタップアクションで送信される postback data は `flex_tap:start_survey:{scenarioId}` という形式でしたが、`flex_tap:` に対応するハンドラが**存在しませんでした**。

:::message
**ポイント**: webhook は LINE Platform に 200 OK を返すため（返さないとリトライされる）、未ハンドルの postback があってもエラーにならない。ログにも残らない。これが「サイレント失敗」の原因です。
:::

### 修正

`flex_tap:` prefix のハンドラを追加しました。

```diff typescript
  // webhook ハンドラの postback 処理
+ if (event.type === 'postback' && event.postback?.data.startsWith('flex_tap:')) {
+   await this.handleFlexImageTap(userId, event.postback.data);
+   continue;
+ }
+
  if (event.type === 'postback' && event.postback?.data) {
    if (data.startsWith('send_msg:')) {
```

```typescript
// handleFlexImageTap の実装
private async handleFlexImageTap(userId: string, postbackData: string): Promise<void> {
  const payload = postbackData.replace('flex_tap:', '');

  if (payload.startsWith('start_survey:')) {
    const scenarioId = payload.replace('start_survey:', '');
    const flow = await this.scenarioRepo.findById(scenarioId);
    if (flow && flow.isActive) {
      const runtime = toScenarioRuntime(flow);
      // シナリオの最初のノードから実行開始
      const startNodeId = findNextNodeId(runtime, triggerNode.id);
      if (startNodeId) {
        await this.executeFromNode(runtime, startNodeId, userId);
      }
    }
  }
}
```

## 問題2: アンケートのバリデーションが厳しすぎる

### 症状

問題1を修正した後、タップ→アンケート開始の処理は動くようになったが、**画像のみの質問**（選択肢が1つ）でバリデーションエラーが発生し、アンケートフローが実行されなかった。

### 原因

アンケートノードの Zod バリデーションスキーマで、選択肢の最小数が `min(2)` になっていました。

```typescript
// アンケートノードのスキーマ
const surveyNodeSchema = z.object({
  type: z.literal('survey'),
  data: z.object({
    questionText: z.string().min(1),
    imageUrl: z.string().url().optional(),
    choices: z.array(surveyChoiceSchema).min(2), // ← 2つ以上必須
  }),
});
```

通常のテキスト選択式アンケートでは問題ありませんが、**画像をタップさせるだけの質問**（例: 「この画像を見て回答してください」→ 1つのボタン）では選択肢が1つしかなく、バリデーションに失敗していました。

### 修正

今回のシステムでは、画像のみの質問（確認ボタン1つ）が正当なユースケースとして存在するため、`min(1)` に緩和しました。

```diff typescript
  data: z.object({
    questionText: z.string().min(1),
    imageUrl: z.string().url().optional(),
-   choices: z.array(surveyChoiceSchema).min(2),
+   choices: z.array(surveyChoiceSchema).min(1),
  }),
```

:::message
**注意**: `min(1)` への変更はすべてのケースで適切とは限りません。設問タイプごとにバリデーションを分けるのがより堅牢な設計です。今回は画像タップ型の設問が他のアンケート設問と同じスキーマを共有していたため、最小限の修正として `min(1)` を選択しました。
:::

## 問題3: 配信履歴が表示されない

### 症状

管理画面のブロードキャスト配信ページで、過去に送信した配信の履歴が一切表示されない。新しく送信した配信はその場で表示されるが、ページをリロードすると消える。

### 原因

この問題は2つの欠落が重なっていました。

1. **バックエンド**: キャンペーン一覧を取得する GET エンドポイントが存在しない
2. **フロントエンド**: ページマウント時にデータを取得する処理がない

配信履歴は `useState` で管理されており、送信成功時にローカルの state に追加する処理しかありませんでした。

```
フロントエンド: campaigns state = [] （初期値が空配列のまま）
                        ↑
           バックエンドから取得する処理がない
                        ↑
           GET /broadcast エンドポイントが存在しない
```

### 修正

**バックエンド**: 一覧取得エンドポイントを追加。

```typescript
// ルート定義
router.get('/broadcast', (req, res) => controller.listCampaigns(req, res));

// コントローラー
async listCampaigns(_req: Request, res: Response): Promise<Response> {
  const campaigns = await this.campaignService.listRecent();
  return res.json({ data: campaigns });
}
```

**フロントエンド**: マウント時に取得する `useEffect` を追加。

```typescript
useEffect(() => {
  const api = getLineAdminAPI();
  api.listCampaigns().then(loaded => {
    setCampaigns(prev => {
      if (prev.length === 0) return loaded;
      // 送信直後のキャンペーンと履歴をIDベースでマージ
      const existingIds = new Set(prev.map(c => c.id));
      const merged = [...prev, ...loaded.filter(c => !existingIds.has(c.id))];
      return merged.sort((a, b) => b.createdAt.localeCompare(a.createdAt));
    });
  }).catch(() => {});
}, []);
```

:::message
**なぜ単純な `setCampaigns(loaded)` ではないのか？**

ページを開いた直後に配信を送信し、その後に初期ロードのレスポンスが返ってくるケースがあります。単純に上書きすると、たった今送信したキャンペーンが消えてしまいます。IDベースでマージすることで、この競合状態を回避しています。
:::

## まとめ

| 問題 | 原因 | 修正 |
|------|------|------|
| タップしても反応しない | `flex_tap:` postback のハンドラがない | prefix ハンドラを追加 |
| アンケートが開始されない | `choices.min(2)` で1選択肢がバリデーション失敗 | `min(1)` に緩和 |
| 配信履歴が表示されない | GET エンドポイントと初期ロードがない | エンドポイント + useEffect 追加 |

### 学び

1. **webhook が 200 OK でも安心しない** — LINE webhook は必ず 200 を返す設計のため、未ハンドルの postback がサイレントに無視される。新しい postback prefix を導入したら、ハンドラの追加を忘れないこと
2. **postback の prefix を一覧管理する** — ハンドラが `if-else` チェーンで分散していると、新しい prefix の追加漏れが起きやすい。prefix とハンドラのマッピングを一元管理することで見通しが良くなる
3. **バリデーションはエッジケースを考慮する** — 「最低2つ」という制約は一般的には正しいが、画像タップのような単一選択肢のユースケースを見落としていた
4. **「書き込みはできるが読み取りがない」パターンに注意** — CRUD の C（Create）だけ実装して R（Read）を忘れるのは意外とよくある

### デバッグチェックリスト

- [ ] 新しい postback prefix を導入したら、webhook ハンドラに対応する分岐があるか？
- [ ] webhook のレスポンスコードだけでなく、実際にハンドラが実行されたかログで確認しているか？
- [ ] バリデーションスキーマは正当なエッジケース（最小構成）で通るか？
- [ ] データを作成する API があるなら、一覧取得する API もあるか？
- [ ] フロントエンドで state の初期値が空の場合、マウント時にデータを取得しているか？
