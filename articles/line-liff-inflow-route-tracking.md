---
title: "LIFF SDKで実現するLINE友だち追加の流入経路トラッキング設計と実装"
emoji: "📊"
type: "tech"
topics: ["line", "liff", "nextjs", "firestore", "typescript"]
published: true
---

## はじめに

LINE公式アカウントを運用していると、「この友だちはどこから来たのか」を知りたくなります。Instagram経由？YouTube？店舗QRコード？

**Lステップ**のような外部ツールを使えば流入経路分析は可能ですが、自前で実装すれば仕組みを完全にコントロールでき、既存システムとの統合も自由自在です。

本記事では、**LIFF SDK**を使って友だち追加の流入経路を確実にトラッキングする仕組みの設計と実装Tipsを紹介します。

:::message
**対象読者**: LINE Messaging APIを使ったシステムに流入経路トラッキングを追加したい方。特にLIFFの活用方法やFirestoreでの分析データ設計に興味がある方。
:::

**この記事で得られる知見:**

- LIFF方式を採用した理由と、リダイレクト方式との比較
- LIFFページをNext.jsで実装する際のSSR回避テクニック
- Firestoreでのアトミックカウンター設計（`FieldValue.increment`）
- 公開APIエンドポイントのセキュリティ設計
- next-intl等のミドルウェアとLIFFパスの共存方法

## 全体像

```
1. 管理者が「流入経路」を作成
   → LIFF URLが生成: https://liff.line.me/{LIFF_ID}?route=instagram-bio

2. ユーザーがLIFF URLをクリック（SNS、広告、店舗QRなど）
   → LINEアプリ内でLIFFページが開く
   → LIFF SDKでログイン → LINE User IDを取得
   → 追跡データをバックエンドAPIに送信
   → 未友だち → 友だち追加URLにリダイレクト

3. 友だち追加イベント（follow webhook）到着
   → LINE User IDで追跡記録を検索（確実な紐付け）
   → 経路タグ付与 + ウェルカムメッセージ送信
```

```
┌─────────────┐    ┌──────────────────┐    ┌─────────────┐
│  LIFFページ   │───▶│  Express Backend  │───▶│  LINE API   │
│  (Next.js)   │    │  (追跡記録保存)    │    │  (Webhook)  │
└─────────────┘    └──────┬───────────┘    └──────┬──────┘
                          │                       │
                   ┌──────▼───────┐         ┌─────▼──────┐
                   │  Firestore    │◀────────│ follow event│
                   │  追跡記録     │  マッチング │ (webhook)  │
                   └──────────────┘         └────────────┘
```

## 設計のポイント

### ポイント1: なぜLIFF方式を選んだか

LINE友だち追加の流入経路をトラッキングする方法は主に2つあります。

| 方式 | 仕組み | ユーザーID取得 | 精度 |
|------|--------|---------------|------|
| **リダイレクト方式** | 中継ページ → 友だち追加URL | Webhookの時刻マッチング | 低（複数人同時で誤帰属） |
| **LIFF方式** | LIFFページ → LIFF SDK | `liff.getProfile()` で直接取得 | 高（LINE User IDで確実に紐付け） |

**リダイレクト方式の問題点**は、中継ページではLINEユーザーIDを取得できないことです。「中継ページを踏んだ直後にfollow webhookが来たら同一人物とみなす」という時刻ベースのマッチングに頼るため、複数人が同時にアクセスすると誤帰属が発生します。

:::message
**LIFF方式の決定的な利点**: LIFFページはLINEアプリ内で開くため、`liff.getProfile()` でLINE User IDを直接取得できます。このIDはfollow webhookの `source.userId` と同一なので、**100%正確な紐付け**が可能です。
:::

### ポイント2: 追跡データと統計の分離

データモデルは「生データ」と「集計データ」を分離する設計にしました。

```
┌─────────────────┐
│  inflow_routes   │ ← 経路の定義（名前、タグ、URL等）
│  clickCount      │ ← 累計クリック数（即時カウンター）
│  followCount     │ ← 累計友だち追加数
└────────┬────────┘
         │ 1:N
┌────────▼────────┐
│ inflow_tracking  │ ← 個別の追跡記録（誰が・いつ・どの経路から）
│ lineUserId       │ ← LIFF SDKで取得したユーザーID
│ utmSource, ...   │ ← UTMパラメータ
│ matched          │ ← follow webhookとマッチ済みか
└─────────────────┘
         │ N:1
┌────────▼────────┐
│ daily_stats      │ ← 日別の集計（グラフ表示用）
│ routeId + date   │ ← 複合キー
│ clickCount       │
│ followCount      │
└─────────────────┘
```

**なぜ分離するのか？**

- **経路定義のカウンター**（`clickCount` / `followCount`）: ダッシュボードで即座に表示する総数。アトミックにインクリメント
- **追跡記録**: 個別ユーザーの行動ログ。「誰がいつどの経路から来たか」を後から分析
- **日別統計**: 時系列グラフ用。「この経路は今週伸びているか」を可視化

一つのコレクションに全部持たせると、集計のためにフルスキャンが必要になります。分離しておけば、ダッシュボード表示は経路ドキュメント1件の読み取りだけで済みます。

### ポイント3: Webhookマッチングの設計

友だち追加イベント（follow webhook）が到着したとき、追跡記録とマッチングする流れです。

```
follow event (lineUserId: "U1234...")
  │
  ├─ lineUserIdで未マッチの追跡記録を検索
  │   → Firestoreクエリ: lineUserId == "U1234..." && matched == false
  │
  ├─ マッチあり:
  │   ├─ matched = true に更新
  │   ├─ followCount++ (経路 + 日別統計)
  │   ├─ autoTagsをユーザーに付与
  │   └─ ウェルカムメッセージ送信
  │
  └─ マッチなし:
      └─ 通常のfollow処理を実行
```

:::message
マッチングロジックは**Webhook処理内（認証済みコンテキスト）でのみ実行**します。LIFFページからの公開APIではタグ付与やメッセージ送信を行いません（後述のセキュリティ設計を参照）。
:::

## 実装Tips

### Tip 1: Next.jsでLIFF SDKをSSR回避して使う

LIFF SDKはブラウザ環境専用のため、Next.jsのサーバーサイドレンダリングで読み込むとエラーになります。`dynamic import` + `useEffect` で回避します。

```tsx
// app/liff/track/page.tsx
"use client";

import { useState, useEffect } from "react";

export default function LiffTrackPage() {
  const [status, setStatus] = useState<"loading" | "done" | "error">("loading");

  useEffect(() => {
    (async () => {
      // Dynamic import to avoid SSR issues
      const liff = (await import("@line/liff")).default;
      await liff.init({ liffId: process.env.NEXT_PUBLIC_LIFF_ID! });

      if (!liff.isLoggedIn()) {
        liff.login(); // Redirect to LINE login
        return;
      }

      const profile = await liff.getProfile();
      const friendship = await liff.getFriendship();

      // Send tracking data to backend
      await fetch("/api/track", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          lineUserId: profile.userId,
          routeCode: new URLSearchParams(location.search).get("route"),
          alreadyFriend: friendship.friendFlag,
        }),
      });

      if (!friendship.friendFlag) {
        // Redirect to friend add URL
        window.location.href = "https://line.me/R/ti/p/@your-bot-id";
      } else {
        setStatus("done");
      }
    })();
  }, []);

  if (status === "loading") return <p>読み込み中...</p>;
  return <p>ご登録ありがとうございます</p>;
}
```

**ポイント:**

- `(await import("@line/liff")).default` で動的インポート。`import liff from "@line/liff"` をトップレベルで書くとSSRで失敗する
- `liff.getFriendship()` で友だち追加済みかを事前チェック。既に友だちなら友だち追加URLへのリダイレクトをスキップ

:::message
`liff.getFriendship()` を使うには、LINE DevelopersコンソールのLIFF設定で「友だち追加オプション」を **ON（aggressive）** にする必要があります。
:::

### Tip 2: Firestoreアトミックカウンターで集計の競合を防ぐ

クリック数や友だち追加数のカウンターを「読み取り → +1 → 書き込み」で実装すると、同時アクセスでカウントが失われます。

```typescript
// ❌ Race condition: 2人が同時にアクセスすると1カウント消失
const doc = await docRef.get();
const current = doc.data()?.clickCount ?? 0;
await docRef.update({ clickCount: current + 1 });
```

Firestoreの `FieldValue.increment()` を使えば、サーバー側でアトミックにインクリメントできます。

```typescript
import { FieldValue } from "firebase-admin/firestore";

// ✅ Atomic increment: 同時アクセスでもカウント消失なし
await docRef.update({
  clickCount: FieldValue.increment(1),
});
```

**日別統計のupsert**にはさらに工夫が必要です。ドキュメントが存在しない場合は作成し、存在する場合はカウンターだけ更新したい。`set({ merge: true })` と組み合わせます。

```typescript
// Upsert: ドキュメントの有無を問わずアトミックに更新
const id = `${routeId}_${date}`; // e.g. "route123_2026-02-14"
await collection.doc(id).set(
  {
    id,
    routeId,
    date,
    clickCount: FieldValue.increment(1),
    updatedAt: new Date().toISOString(),
    createdAt: new Date().toISOString(), // 既存ドキュメントではmergeで無視される
  },
  { merge: true }
);
```

:::message
`set({ merge: true })` は既存フィールドを上書きしますが、`FieldValue.increment()` はmerge時でも**加算として動作**します。`createdAt` は初回作成時のみ書き込まれ、2回目以降は既存値が保持されます。
:::

### Tip 3: 公開APIエンドポイントのセキュリティ設計

LIFFページからの追跡データ送信は**認証なしの公開エンドポイント**です。LIFFページ自体はLINEログインで保護されますが、APIエンドポイント自体は誰でも叩けます。

**やってはいけない設計:**

```typescript
// ❌ 公開APIでタグ付与やメッセージ送信を行う
app.post("/api/track", async (req, res) => {
  const { lineUserId, routeCode, alreadyFriend } = req.body;
  await saveTracking(lineUserId, routeCode);

  if (alreadyFriend) {
    // 攻撃者がalreadyFriend=trueで任意のlineUserIdを送ると
    // 他人のアカウントにタグが付与されてしまう！
    await applyTags(lineUserId, route.autoTags);
    await sendMessage(lineUserId, route.welcomeMessage);
  }
});
```

**安全な設計:**

```typescript
// ✅ 公開APIではデータ記録のみ。アクションはWebhook（認証済み）で実行
app.post("/api/track", async (req, res) => {
  const { lineUserId, routeCode, alreadyFriend } = req.body;
  // Record tracking data only (no tag mutations or message sends)
  await saveTracking(lineUserId, routeCode, alreadyFriend);

  if (alreadyFriend) {
    // Count only - no side effects
    await incrementFollowCount(routeId);
  }

  res.json({ success: true });
});

// Webhook handler (LINE signature verified)
async function handleFollowEvent(lineUserId: string) {
  const tracking = await findUnmatchedTracking(lineUserId);
  if (tracking) {
    await markAsMatched(tracking.id);
    // Safe to apply actions here - webhook is verified
    await applyTags(lineUserId, route.autoTags);
    await sendMessage(lineUserId, route.welcomeMessage);
  }
}
```

:::message
**原則: 公開エンドポイントでは「記録」のみ、「アクション」はWebhook（署名検証済み）で実行**。これにより、攻撃者が偽のリクエストを送っても、タグの不正付与やスパムメッセージ送信を防げます。
:::

### Tip 4: next-intlミドルウェアとLIFFパスの共存

`next-intl` 等のi18nミドルウェアを使っている場合、`/liff/track?route=xxx` にアクセスすると `/ja/liff/track` にリダイレクトされてしまいます。LIFFページはlocale不要なので、ミドルウェアでバイパスします。

```diff typescript:middleware.ts
 export default function middleware(request: NextRequest) {
   const { pathname } = request.nextUrl;

   // Skip locale redirect for auth and LIFF pages
-  if (pathname.startsWith('/auth')) {
+  if (pathname.startsWith('/auth') || pathname.startsWith('/liff')) {
     return NextResponse.next();
   }

   // Apply locale detection for all other routes
   return intlMiddleware(request);
 }
```

LIFFページは `app/liff/track/page.tsx` に配置し、`[locale]` ディレクトリの外に置きます。こうすることで、LIFFから開かれたとき余計なリダイレクトなしにページが表示されます。

### Tip 5: Firestoreの複合インデックス設計

追跡記録のクエリには複合インデックスが必要です。設計を間違えるとクエリ実行時にエラーになります。

```json
// firestore.indexes.json
{
  "indexes": [
    {
      "collectionGroup": "inflow_tracking",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "lineUserId", "order": "ASCENDING" },
        { "fieldPath": "matched", "order": "ASCENDING" },
        { "fieldPath": "clickedAt", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "inflow_tracking",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "routeId", "order": "ASCENDING" },
        { "fieldPath": "clickedAt", "order": "DESCENDING" }
      ]
    }
  ]
}
```

**1つ目のインデックス**は、follow webhook到着時のマッチングクエリ用:

```typescript
// lineUserIdで未マッチの追跡記録を検索
collection
  .where("lineUserId", "==", userId)
  .where("matched", "==", false)
  .orderBy("clickedAt", "desc")
  .limit(1);
```

**2つ目のインデックス**は、管理画面での経路別ユーザー一覧用:

```typescript
// 特定経路の追跡記録を新しい順に取得
collection
  .where("routeId", "==", routeId)
  .orderBy("clickedAt", "desc")
  .limit(200);
```

:::message
Firestoreは `where` + `orderBy` の組み合わせに複合インデックスを要求します。デプロイは `firebase deploy --only firestore:indexes` で実行できます。
:::

## ハマりポイント

### TypeScriptの `exactOptionalPropertyTypes` とFirestore

TypeScriptの `exactOptionalPropertyTypes: true` を有効にしている場合、オプショナルプロパティの型定義に注意が必要です。

```typescript
// ❌ exactOptionalPropertyTypes: true の場合エラー
interface TrackingRecord {
  utmSource?: string; // undefined の代入が型エラーになる
}

const record: TrackingRecord = {
  utmSource: params.get("utm_source") ?? undefined, // Error!
};
```

```typescript
// ✅ 明示的に | undefined を追加
interface TrackingRecord {
  utmSource?: string | undefined;
}
```

`exactOptionalPropertyTypes` は「`?` は値を省略できるが、`undefined` を明示的に代入するのは別」という厳格なルールです。Firestoreのデータモデルのように `undefined` を扱う場面が多いエンティティでは、`| undefined` を明記する癖をつけましょう。

### LIFF App作成にはLINE Loginチャネルが必要

LIFFアプリを作成するには、**Messaging APIチャネルではなくLINE Loginチャネル**が必要です。

| チャネルタイプ | LIFF作成 | Webhook受信 | Push Message |
|--------------|----------|------------|-------------|
| Messaging API | ❌ | ✅ | ✅ |
| LINE Login | ✅ | ❌ | ❌ |

LINE DevelopersコンソールでLINE Loginチャネルを新規作成し、「LIFF」タブからアプリを追加します。既存のMessaging APIチャネルとは別物です。

## まとめ

### 技術選定・設計判断の一覧

| 判断 | 選択 | 理由 |
|------|------|------|
| トラッキング方式 | LIFF SDK | LINE User IDを直接取得でき、100%正確な帰属 |
| カウンター更新 | `FieldValue.increment` + `set({ merge: true })` | 同時アクセスでのカウント消失を防止 |
| タグ付与のタイミング | Webhook処理内 | 公開APIからの不正リクエストを防止 |
| LIFF SDK読み込み | Dynamic import | Next.jsのSSRエラーを回避 |
| LIFFページのパス | `[locale]` 外に配置 | i18nミドルウェアのリダイレクトを回避 |
| データモデル | 経路定義 / 追跡記録 / 日別統計を分離 | クエリ効率とダッシュボード表示速度の最適化 |

### 学び

1. **LIFF SDKはLINE User IDの取得に最適** — リダイレクト方式の時刻マッチングと比べて精度が段違い
2. **公開APIでは「記録のみ」が原則** — 副作用（タグ付与、メッセージ送信）は認証済みコンテキスト（Webhook）で実行する
3. **Firestoreのアトミック操作を活用する** — `FieldValue.increment` + `set({ merge: true })` でread-then-writeの競合を排除
4. **Next.jsのミドルウェアとLIFFの共存は要注意** — i18nミドルウェアがLIFFパスをリダイレクトしないよう明示的にバイパスする
5. **`exactOptionalPropertyTypes` とFirestoreの相性** — オプショナルプロパティには `| undefined` を明記する

### 実装チェックリスト

- [ ] LINE DevelopersコンソールでLINE Loginチャネルを作成し、LIFFアプリを登録したか
- [ ] LIFF設定で「友だち追加オプション」をON（aggressive）にしたか
- [ ] `NEXT_PUBLIC_LIFF_ID` を環境変数に設定したか
- [ ] LIFF SDKをdynamic importしているか（SSRエラー回避）
- [ ] i18nミドルウェアで `/liff` パスをバイパスしているか
- [ ] 公開APIエンドポイントでタグ付与やメッセージ送信を行っていないか
- [ ] Firestoreの複合インデックスをデプロイしたか
- [ ] カウンター更新に `FieldValue.increment` を使っているか
