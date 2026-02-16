---
title: "LINE管理画面にリッチメッセージ配信を実装する — Next.js + Firebase Storage + Zod"
emoji: "📨"
type: "tech"
topics: ["nextjs", "line", "firebase", "typescript", "zod"]
published: true
---

## はじめに

LINE公式アカウントの管理画面で、テキストだけでなく**画像・動画・音声**を送信できるようにし、ブロードキャスト配信では**最大5件のメッセージを同時送信**できるマルチメッセージ配信を実装しました。

:::message
**対象読者**
- LINE Messaging APIで画像・動画メッセージを扱いたい方
- Next.js + Firebase Storageでメディアアップロードを実装したい方
- Zodの `discriminatedUnion` で複雑なメッセージ型を安全にバリデーションしたい方
:::

この記事で得られる知見：

- LINE Messaging APIで画像/動画/音声を送るときの**メッセージ型設計**
- Firebase Storageへの直接アップロード + **Canvas APIによるプレビュー自動生成**
- `z.discriminatedUnion` による**型安全なメッセージバリデーション**
- 後方互換性を保ったAPI拡張パターン
- マルチメッセージコンポーザーUIの設計

## 全体像

```
┌─────────────┐    Firebase Storage     ┌──────────────┐
│  管理画面     │ ──── upload ──────────▶ │  Cloud Storage │
│  (Next.js)   │                         └──────┬───────┘
│              │                                │ HTTPS URL
│              │ ──── API call ──────────▶ ┌────▼────────┐
│              │   { type, url, ... }     │  Backend     │
└─────────────┘                           │  (Express)   │
                                          └──────┬───────┘
                                                 │ push / broadcast
                                          ┌──────▼───────┐
                                          │ LINE Platform │
                                          └──────────────┘
```

**ポイント**: メディアファイルはフロントエンドからFirebase Storageに**直接アップロード**し、取得したHTTPS URLをバックエンドに渡します。バックエンドはURLをLINE Messaging APIにそのまま転送するだけなので、バックエンド側でファイルを扱う必要がありません。

## 設計のポイント

### ポイント1: メッセージ型の設計 — discriminated union

LINE Messaging APIは `type` フィールドでメッセージ種別を判別します。この構造をそのままTypeScriptの判別可能ユニオン（discriminated union）として型定義します。

```typescript
// Domain type — backend & frontend shared
type CampaignMessage =
  | { type: 'text'; text: string }
  | { type: 'image'; originalContentUrl: string; previewImageUrl: string }
  | { type: 'video'; originalContentUrl: string; previewImageUrl: string }
  | { type: 'audio'; originalContentUrl: string; duration: number };
```

:::message
**なぜ discriminated union か？**
LINE APIのメッセージ構造が `type` で分岐するため、TypeScript側でも同じ判別子を使うことで**APIリクエスト/レスポンスの型がそのまま一致**します。変換レイヤーが不要になり、バグの温床を減らせます。
:::

| 方式 | 型安全 | 変換コスト | LINE APIとの一致 |
|------|--------|-----------|-----------------|
| `discriminatedUnion` | ◎ `type` で自動narrowing | なし | 完全一致 |
| 共通interface + enumフラグ | ○ | 要変換 | 要マッピング |
| `any` + 手動チェック | × | なし | 一致するが型なし |

### ポイント2: メディアアップロードの方式選択

| 方式 | メリット | デメリット |
|------|---------|-----------|
| **フロントから直接Storage** | バックエンド不要、シンプル | Storage Rules設定が必要 |
| バックエンド経由 | 認証を一元管理 | ファイル転送のオーバーヘッド |
| 署名付きURL | セキュア | 実装が複雑 |

今回は既存のFirebase Storage直接アップロードパターンがプロジェクトにあったため、**フロントから直接Storage**方式を採用しました。

### ポイント3: プレビュー画像の自動生成

LINE Messaging APIの画像/動画メッセージには `previewImageUrl` が必須です。ユーザーに手動でプレビューを用意させるのは現実的ではないため、**クライアントサイドで自動生成**します。

```
画像 → Canvas APIでリサイズ → プレビュー用JPEG
動画 → <video>要素で最初のフレームをキャプチャ → サムネイルJPEG
音声 → <audio>要素でduration取得（プレビュー画像不要）
```

## 実装Tips

### Tip 1: Canvas APIで画像プレビューを生成する

LINE APIの画像メッセージは `originalContentUrl`（元画像）と `previewImageUrl`（サムネイル）の2つのURLが必要です。Canvas APIで240px以下にリサイズしたJPEGを自動生成します。

```typescript
const PREVIEW_SIZE = 240;
const PREVIEW_QUALITY = 0.7;

async function generateImagePreview(file: File): Promise<Blob> {
  const dataUrl = await readFileAsDataUrl(file);
  const img = await loadImage(dataUrl);

  const canvas = document.createElement('canvas');
  // Maintain aspect ratio within PREVIEW_SIZE
  const scale = Math.min(PREVIEW_SIZE / img.width, PREVIEW_SIZE / img.height, 1);
  canvas.width = Math.round(img.width * scale);
  canvas.height = Math.round(img.height * scale);

  const ctx = canvas.getContext('2d')!;
  ctx.drawImage(img, 0, 0, canvas.width, canvas.height);

  return new Promise((resolve, reject) => {
    canvas.toBlob(
      (blob) => blob ? resolve(blob) : reject(new Error('Failed')),
      'image/jpeg',
      PREVIEW_QUALITY
    );
  });
}
```

:::message
`Math.min(..., 1)` で**元画像より大きくならない**ようにしています。240px以下の画像はそのままのサイズを使います。
:::

### Tip 2: 動画サムネイルをキャプチャする

`<video>` 要素を使って動画の最初のフレームをキャプチャします。

```typescript
async function generateVideoThumbnail(file: File): Promise<Blob> {
  return new Promise((resolve, reject) => {
    const video = document.createElement('video');
    video.preload = 'metadata';
    video.muted = true;
    video.playsInline = true;

    const url = URL.createObjectURL(file);
    video.src = url;

    video.onloadeddata = () => { video.currentTime = 0.1; };
    video.onseeked = () => {
      const canvas = document.createElement('canvas');
      const scale = Math.min(240 / video.videoWidth, 240 / video.videoHeight, 1);
      canvas.width = Math.round(video.videoWidth * scale);
      canvas.height = Math.round(video.videoHeight * scale);

      const ctx = canvas.getContext('2d');
      if (!ctx) { reject(new Error('Canvas not available')); return; }
      ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
      URL.revokeObjectURL(url);
      canvas.toBlob(
        (blob) => blob ? resolve(blob) : reject(new Error('Failed')),
        'image/jpeg', 0.7
      );
    };
    video.onerror = () => {
      URL.revokeObjectURL(url);
      reject(new Error('Failed to load video'));
    };
  });
}
```

:::message
**`currentTime = 0.1`** にしているのは、`0` だと一部のブラウザで黒いフレームが返る場合があるためです。`0.1`秒（100ms）後にシークすることで、実際のコンテンツフレームを確実にキャプチャできます。
:::

### Tip 3: Zodの `discriminatedUnion` でAPIバリデーション

バックエンドではZodの `z.discriminatedUnion` を使い、`type` フィールドの値に基づいて適切なスキーマを自動選択します。

```typescript
import { z } from 'zod';

const textMessageSchema = z.object({
  type: z.literal('text'),
  text: z.string().min(1).max(5000),
});

const imageMessageSchema = z.object({
  type: z.literal('image'),
  originalContentUrl: z.string().url(),
  previewImageUrl: z.string().url(),
});

const videoMessageSchema = z.object({
  type: z.literal('video'),
  originalContentUrl: z.string().url(),
  previewImageUrl: z.string().url(),
});

const audioMessageSchema = z.object({
  type: z.literal('audio'),
  originalContentUrl: z.string().url(),
  duration: z.number().int().positive(),
});

// discriminatedUnion: 'type' の値で自動的にスキーマが選択される
const campaignMessageSchema = z.discriminatedUnion('type', [
  textMessageSchema,
  imageMessageSchema,
  videoMessageSchema,
  audioMessageSchema,
]);

// Broadcast: 1〜5件のメッセージ配列
const broadcastInputSchema = z.object({
  name: z.string().min(1),
  messages: z.array(campaignMessageSchema).min(1).max(5),
  tags: z.array(z.string()).optional(),
});
```

`z.union` との違い：

| | `z.discriminatedUnion` | `z.union` |
|---|---|---|
| パース速度 | 判別子で即座に分岐 → **高速** | 全スキーマを順に試行 → 遅い |
| エラーメッセージ | 判別子不一致を明確に報告 | 「どのスキーマにも一致しない」 |
| 制約 | 共通の判別子フィールドが必須 | 任意のスキーマを組み合わせ可 |

### Tip 4: 後方互換性を保ったAPI拡張

既存のダイレクトメッセージAPIは `{ messageText: "..." }` 形式でした。新しい形式 `{ type: "text", text: "..." }` に移行しつつ、旧形式も受け付ける必要があります。

Zodの `.transform()` を使って、旧形式を新形式に自動変換します。

```typescript
// Legacy format: { messageText: "hello" }
const legacyTextSchema = z
  .object({ messageText: z.string().min(1).max(5000) })
  .transform((data) => ({ type: 'text' as const, text: data.messageText }));

// New format: { type: "text", text: "hello" } | { type: "image", ... } | ...
const newMessageSchema = z.discriminatedUnion('type', [
  textMessageSchema,
  imageMessageSchema,
  videoMessageSchema,
  audioMessageSchema,
]);

// Accept both formats
const directMessageSchema = z.union([newMessageSchema, legacyTextSchema]);
```

:::message
`z.union` の評価順序を利用して、新形式（`type` フィールドあり）を先に試し、マッチしなければ旧形式にフォールバックします。これにより**クライアント更新前でもAPIが壊れません**。
:::

### Tip 5: 受信メディアのフォールバック表示

LINEから受信した画像のコンテンツ取得にはアクセストークンが必要なため、DBには `line://msg/{messageId}` のようなプレースホルダーURLで保存しています。このURLはブラウザで直接表示できません。

UIでは `http://` / `https://` で始まらないURLを検出して、プレースホルダーを表示します。

```typescript
function isWebUrl(url: string): boolean {
  return url.startsWith('http://') || url.startsWith('https://');
}
```

```tsx
function ImageContent({ message }) {
  const src = message.previewUrl || message.content;

  if (!isWebUrl(src)) {
    return (
      <div className="flex items-center gap-2 text-xs opacity-70">
        <PhotoIcon />
        画像を受信しました
      </div>
    );
  }

  return <img src={src} alt="画像" /* ... */ />;
}
```

### Tip 6: LINE API互換のMIMEタイプに制限する

LINE Messaging APIがサポートするフォーマットは限定的です。ファイル選択時にブラウザの `accept` 属性で制限します。

```typescript
// LINE API compatible formats only
const ACCEPT_MAP = {
  image: 'image/jpeg,image/png',      // WebPは非対応
  video: 'video/mp4',                  // QuickTime(.mov)は非対応
  audio: 'audio/mpeg,audio/mp4,audio/x-m4a',
};
```

:::message
**WebPに注意**: ブラウザでは一般的なWebPですが、LINE Messaging APIでは非対応です。アップロードはできても、LINE側で表示エラーになります。
:::

## ハマりポイント

### SWCがデストラクチャリング代入の `!` を許容しない

Next.jsのコンパイラ（SWC）では、配列デストラクチャリング代入の左辺に非nullアサーション（`!`）を使うとビルドエラーになります。

```diff
  // ❌ SWC: "Not a pattern" エラー
- [arr[i]!, arr[j]!] = [arr[j]!, arr[i]!];

  // ✅ 一時変数で回避
+ const temp = arr[i];
+ arr[i] = arr[j]!;
+ arr[j] = temp!;
```

TypeScript（tsc）では問題ないのにSWCでエラーになる、という差異があるので注意が必要です。

## まとめ

### 技術選定・設計判断の一覧

| 判断 | 選択 | 理由 |
|------|------|------|
| メッセージ型 | discriminated union | LINE APIの `type` 判別子とTypeScriptの型システムが自然に一致 |
| アップロード方式 | フロントから直接Storage | バックエンドのファイル転送不要、既存パターン踏襲 |
| プレビュー生成 | クライアントサイドCanvas API | サーバーレス、追加インフラ不要 |
| バリデーション | Zod `discriminatedUnion` | 高速パース + 明確なエラーメッセージ |
| 後方互換 | `z.union` + `.transform()` | 旧クライアントを壊さずに型を拡張 |
| MIME制限 | `accept` 属性でフィルタ | LINE API非対応フォーマットを事前排除 |

### 学び

1. **LINE APIの型構造をそのままTypeScriptの型に落とす**と、変換レイヤーが不要になりバグが減る
2. **Canvas APIによるプレビュー自動生成**は、動画のサムネイル生成にも応用できる汎用テクニック
3. **Zodの `discriminatedUnion`** は `union` より高速で、エラーメッセージも読みやすい
4. **後方互換性**は `.transform()` で旧形式→新形式変換すると、スキーマ1つで表現できてスッキリ
5. **SWCとtscの挙動差異**は実際にビルドしないと気づけない。CI/CDでのビルドチェックが重要

### 実装チェックリスト

- [ ] LINE APIがサポートするMIMEタイプのみを`accept`で許可しているか
- [ ] 画像/動画メッセージに `previewImageUrl` を必ず設定しているか
- [ ] 音声メッセージの `duration` はミリ秒単位の正の整数か
- [ ] Firebase Storage Rulesで `line-media/` パスへの書き込みを許可したか
- [ ] 受信メディア（非WebURL）に対するフォールバック表示があるか
- [ ] ブロードキャストのメッセージ数がLINE API上限（5件）以内か
- [ ] 旧形式のAPIリクエストが引き続き動作するか（後方互換性）
