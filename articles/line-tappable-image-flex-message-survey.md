---
title: "LINE Flex Messageで「タップ画像配信」を実装する — 設計判断と実装Tips"
emoji: "👆"
type: "tech"
topics: ["LINE", "MessagingAPI", "FlexMessage", "NextJS", "TypeScript"]
published: true
---

## はじめに

LINE公式アカウント（未認証）では、フォロワーIDの一括取得（`/v2/bot/followers/ids`）が使えないため、**ユーザーからのアクション（メッセージ送信やフォロー）が発生するまで個別のユーザーIDを取得できない** という制約があります。ブロードキャストは全友だちに送れますが、特定ユーザーへのターゲティング配信やアンケートの個別送信にはユーザーIDが必要です。

この記事では、**タップ可能な画像（Flex Message）** を配信し、ユーザーのタップ（postback）でユーザーIDを取得・タグ付けすることで、この制約を解消した実装について解説します。

:::message
**対象読者**
- LINE Messaging API でボットやアンケートを実装している方
- ユーザーIDの取得・ターゲティング配信に課題を感じている方
- Flex Message の実践的な活用例を探している方
:::

**この記事で得られること：**
- Flex Message で全面タップ可能な画像を実装する方法
- postback アクションを活用したユーザー行動トラッキング
- 「画像のみ」アンケート質問タイプの設計パターン
- フロントエンド管理画面での動的メッセージタイプの拡張方法

## 全体像

今回実装した機能のフローは以下の通りです。

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  管理画面     │     │   Backend    │     │   LINE API   │
│ (Next.js)    │────▶│  (Express)   │────▶│  broadcast/  │
│              │     │              │     │  multicast   │
└──────────────┘     └──────┬───────┘     └──────┬───────┘
                            │                     │
                            │                     ▼
                            │              ┌──────────────┐
                            │              │   ユーザー    │
                            │              │  タップ画像   │
                            │              │   を受信      │
                            │              └──────┬───────┘
                            │                     │
                            │               タップ（postback）
                            │                     │
                            ▼                     ▼
                     ┌──────────────┐     ┌──────────────┐
                     │  Webhook     │◀────│  LINE API    │
                     │  Handler     │     │  postback    │
                     └──────┬───────┘     └──────────────┘
                            │
                    ┌───────┴───────┐
                    ▼               ▼
             認知タグ付与     アンケート開始
```

**ユーザー視点のフロー：**
1. LINE でタップ可能な画像を受け取る
2. 画像をタップする
3. (a) 認知のみ → ユーザーに `image_tapped` タグが付く
4. (b) アンケート開始 → 最初のアンケート質問が自動送信される

## 設計のポイント

### ポイント1: なぜ Flex Message を選んだか

LINE Messaging API でタップ可能な画像を送る方法はいくつかあります。

| 方式 | タップ検知 | 画像表示 | postback対応 |
|------|:--------:|:-------:|:-----------:|
| Image Message | × | 全面 | × |
| Imagemap | ○ | 全面 | × (uri/message/clipboard) |
| Template (buttons) | ○ | サムネイル | ○ |
| **Flex Message** | **○** | **全面** | **○** |

:::message
**Flex Message が最適な理由：** Imagemap はタップ領域を座標で定義でき、`uri`・`message`・`clipboard` アクションに対応していますが、**postback アクションには非対応** です。`message` アクションでも Webhook でユーザーIDは取得できますが、ユーザーのチャットにテキストが表示されてしまい、構造化データの受け渡しにも不向きです。Template Message は postback に対応していますが、画像がサムネイルサイズに制限されます。Flex Message なら **画像を全面表示しつつ postback アクションを設定** できます。
:::

### ポイント2: タップ時アクションを管理者が選べる設計

タップ画像の用途は大きく2つあります。

1. **認知のみ** — 画像を見せて、タップしたユーザーにタグを付けたい（広告やお知らせ）
2. **アンケート開始** — タップをトリガーにアンケートフローを開始したい

これを管理者がブロードキャスト作成時に選べるよう、postback データを以下のフォーマットで設計しました。

```
image_tap:recognition          → 認知のみ
image_tap:start_survey:{id}   → アンケート開始（シナリオID付き）
```

Webhook ハンドラーはこの prefix で処理を分岐します。

## 実装 Tips

### Tip 1: Flex Message で全面タップ画像を作る

Flex Message で画像全体をタップ可能にするポイントは、**bubble の body に image コンポーネントを直接配置** し、`paddingAll: "0px"` で余白を消すことです。

```typescript
// Flex Message bubble structure
const flexContents = {
  type: "bubble",
  body: {
    type: "box",
    layout: "vertical",
    contents: [
      {
        type: "image",
        url: imageUrl,
        size: "full",
        aspectMode: "cover",
        aspectRatio: "1:1",
        action: {
          type: "postback",
          label: "タップ",
          data: "image_tap:recognition",
          displayText: "タップしました",
        },
      },
    ],
    paddingAll: "0px",  // Remove all padding for full-bleed image
  },
};
```

:::message
**`aspectRatio` の選び方：** `1:1` は正方形で LINE のチャット画面にバランスよく収まります。縦長の画像なら `1:1.5`、横長なら `3:2` など用途に合わせて調整してください。
:::

:::message alert
**画像URL・postback の制約：**
- 画像URLは **HTTPS** 必須（TLS 1.2以上）、JPEG または PNG、最大 10 MB（推奨 1 MB 以下）
- `postback.data` は最大 **300文字** — シナリオIDが長い場合は注意
- `postback.displayText` は最大 **300文字**
- `altText` は最大 **1500文字**
:::

### Tip 2: CampaignMessage → BroadcastMessage の変換レイヤー

管理画面のデータモデル（`flex_image`）とLINE APIに送るデータモデル（`flex`）は異なります。保存時は管理しやすい形式で、送信時にLINE APIフォーマットに変換する設計にしました。

```typescript
// Domain model (stored in DB)
type CampaignMessage =
  | { type: "text"; text: string }
  | { type: "image"; originalContentUrl: string; previewImageUrl: string }
  | { type: "flex_image"; imageUrl: string; altText: string; action: FlexImageAction };

// Gateway model (sent to LINE API)
type BroadcastMessage =
  | { type: "text"; text: string }
  | { type: "image"; originalContentUrl: string; previewImageUrl: string }
  | { type: "flex"; altText: string; contents: Record<string, unknown> };
```

変換関数で `flex_image` → `flex` に変換します。

```typescript
function toGatewayMessages(messages: CampaignMessage[]): BroadcastMessage[] {
  return messages.map((msg) => {
    if (msg.type !== "flex_image") return msg;

    const postbackData =
      msg.action.type === "recognition"
        ? "image_tap:recognition"
        : `image_tap:start_survey:${msg.action.scenarioId}`;

    return {
      type: "flex",
      altText: msg.altText,
      contents: buildFlexBubble(msg.imageUrl, postbackData),
    };
  });
}
```

:::message
**なぜ変換レイヤーを挟むか：** `flex_image` というドメイン固有の概念を LINE API の `flex` メッセージ仕様から切り離すことで、将来的に他のFlex Message バリエーション（カルーセル画像、ボタン付き画像など）を追加しやすくなります。
:::

### Tip 3: 画像のみアンケートの実装パターン

既存のアンケート（survey）は「質問文 + 選択肢」で構成されていますが、「画像タップで次に進む」形式を追加するために **`isImageOnly` フラグ** を導入しました。

なお、`questionText` はスキーマ上は必須のままです。画像のみモードでは管理画面から自動的に `"(画像のみ)"` が設定され、Flex Message の `altText`（通知テキストや画像を表示できない環境でのフォールバック）として使われます。

```typescript
// Survey node schema (zod)
const surveyNodeSchema = z.object({
  id: z.string().min(1),
  type: z.literal("survey"),
  data: z
    .object({
      questionText: z.string().min(1),
      imageUrl: z.string().url().optional(),
      isImageOnly: z.boolean().optional(),
      choices: z.array(surveyChoiceSchema).min(1),
    })
    .refine((d) => d.isImageOnly || d.choices.length >= 2, {
      message: "Non-image-only surveys require at least 2 choices",
      path: ["choices"],
    }),
});
```

:::message
**後方互換性のポイント：** `choices` の最小値を一律 `min(1)` に下げると、既存の通常アンケートで1選択肢のみのデータが通ってしまいます。`.refine()` で **`isImageOnly` のときだけ 1 選択肢を許可** し、それ以外は従来通り2以上を要求することで後方互換性を維持しています。
:::

画像のみアンケートの送信は、通常の template メッセージではなく Flex Message を使います。

```typescript
async function sendSurvey(runtime, node, lineUserId) {
  // Image-only: send as tappable Flex Message
  if (node.data.isImageOnly && node.data.imageUrl) {
    const firstChoice = node.data.choices[0];
    const postbackData = buildSurveyAnswerPostbackData({
      scenarioId: runtime.id,
      nodeId: node.id,
      choiceValue: firstChoice.value,
    });

    await gateway.pushToUser(lineUserId, [
      {
        type: "flex",
        altText: node.data.questionText,
        contents: buildFlexBubble(node.data.imageUrl, postbackData),
      },
    ]);
    return;
  }

  // Normal survey: existing template message logic
  // ...
}
```

既存の `survey_answer:*` postback パイプラインをそのまま再利用できるのがこの設計の利点です。画像のみアンケートのタップも、通常の選択肢クリックと同じ `handleSurveyAnswer` で処理されます。

:::message alert
**重複タップに注意：** ユーザーが同じ画像を複数回タップすると、タグ付与やアンケート開始が重複実行される可能性があります。本番運用では、ユーザーごと・ノードごとの完了フラグをチェックする冪等性ガードの追加を検討してください。
:::

### Tip 4: フロントエンドでの条件付き UI 切り替え

管理画面で「画像のみモード」をONにすると、質問文入力と選択肢セクションを非表示にし、代わりに遷移先セレクターを表示します。

```tsx
{/* Image-only toggle */}
<input
  type="checkbox"
  checked={question.isImageOnly}
  onChange={(e) =>
    updateQuestion(index, (prev) => ({
      ...prev,
      isImageOnly: e.target.checked,
      // Reset choices when switching modes
      choices: e.target.checked
        ? [createDefaultChoice()]
        : [createDefaultChoice(), createDefaultChoice()],
    }))
  }
/>
<label>画像タップのみ（テキスト・選択肢なし）</label>

{/* Conditionally show question text or transition selector */}
{!question.isImageOnly && <QuestionTextEditor ... />}
{question.isImageOnly && <TransitionSelector ... />}
{!question.isImageOnly && <ChoicesEditor ... />}
```

:::message
**モード切替時の注意：** `isImageOnly` をONにしたとき、選択肢を1つにリセットしています。OFFに戻したときは2つに戻します。これにより、バリデーションルール（通常: 2以上、画像のみ: 1）との整合性が保たれます。
:::

## ハマりポイント

### `isComplete` でアクション固有の必須チェックを忘れる

ブロードキャスト送信ボタンの有効/無効は `isComplete` 関数で制御しています。最初、`flex_image` の完了判定を画像URLとaltTextだけでチェックしていました。

```diff
  case "flex_image":
-   return !!draft.imageUrl && !!draft.altText.trim();
+   if (!draft.imageUrl || !draft.altText.trim()) return false;
+   if (draft.action.type === "start_survey" && !draft.action.scenarioId) return false;
+   return true;
```

`start_survey` アクションを選んだのにシナリオが未選択のまま送信できてしまう問題がありました。**アクションタイプごとの必須フィールドも完了判定に含める** 必要があります。

## まとめ

### 技術選定・設計判断の一覧

| 判断 | 選択 | 理由 |
|------|------|------|
| メッセージ形式 | Flex Message | 画像全面表示 + postback 対応 |
| タップアクション | postback | Webhook で受信可能、データ付与可能 |
| アクション種別 | prefix ベース分岐 | `image_tap:recognition` / `image_tap:start_survey:{id}` |
| 画像のみアンケート | `isImageOnly` フラグ | 既存スキーマへの最小限の変更で対応 |
| choices 最小値 | `.refine()` で条件分岐 | 後方互換性を維持 |
| ドメインモデル | `flex_image` → `flex` 変換 | 保存形式とAPI形式の分離 |

### 学び

1. **Flex Message は LINE Messaging API の中で最も柔軟なメッセージ形式** — Imagemap や Template では対応できないユースケースでも、Flex Message なら実現できることが多い
2. **postback データ設計は拡張性を意識する** — prefix ベースの分岐にしておくと新しいアクションタイプの追加が容易
3. **後方互換性は refine で守る** — zod の `.refine()` を使えば条件付きバリデーションを既存スキーマに安全に追加できる
4. **変換レイヤーはドメインモデルを守る** — 外部APIの仕様変更の影響をアプリケーション層に閉じ込められる
5. **完了判定はアクションの種類ごとに検証する** — Union 型の各バリアントが持つ固有の必須フィールドを見落としやすい

### 実装チェックリスト

- [ ] Flex Message の `altText` を設定したか（画像を表示できない環境向け）
- [ ] postback データに適切な prefix を付けたか
- [ ] Webhook ハンドラーで新しい postback prefix を処理しているか
- [ ] `isImageOnly` 時の選択肢数バリデーションが既存に影響しないか
- [ ] フロントエンドの完了判定にアクション固有の必須チェックを含めたか
- [ ] 既存のアンケート機能が壊れていないか（回帰テスト）
