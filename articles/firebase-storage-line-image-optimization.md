---
title: "Firebase Storage × LINE配信の画像表示が遅い？Cache-ControlとCanvas圧縮で解決した話"
emoji: "🖼️"
type: "tech"
topics: ["Firebase", "LINE", "JavaScript", "TypeScript", "パフォーマンス"]
published: true
---

## はじめに

Firebase Storageに保存した画像をLINE Messaging APIで配信した際、**画像の表示が明らかに遅い**という問題に遭遇しました。数秒待たないと画像が表示されないケースもあり、ユーザー体験を損なっていました。

:::message
**対象読者**
- Firebase Storageに画像をアップロードしてLINEや外部サービスから配信している方
- ブラウザ側で画像を圧縮してからアップロードしたい方
- Firebase Storageの配信パフォーマンスを改善したい方
:::

この記事では、以下の2つのアプローチで問題を解決した方法を紹介します。

- **Cache-Controlヘッダーの設定** — CDN/中間キャッシュを有効化
- **Canvas APIによるクライアントサイド画像圧縮** — アップロード前にリサイズ＋JPEG変換

## 全体像

```
ユーザー(管理画面)          Firebase Storage          LINE Server          ユーザー(LINE)
      |                          |                        |                      |
      |-- 画像選択 ------------->|                        |                      |
      |   [Canvas圧縮]          |                        |                      |
      |-- アップロード --------->|                        |                      |
      |   [Cache-Control付き]   |                        |                      |
      |                          |<-- 画像取得 ----------|                      |
      |                          |   [CDNキャッシュHIT]   |                      |
      |                          |-- レスポンス --------->|                      |
      |                          |                        |-- 画像表示 --------->|
      |                          |                        |   [軽量JPEG]         |
```

**Before**: 元画像(数MB) × キャッシュなし → LINEサーバーが毎回オリジンから取得
**After**: 圧縮画像(100-300KB) × CDNキャッシュ → 高速レスポンス

## 設計のポイント

### ポイント1: なぜCache-Controlが必要なのか

Firebase Storageにファイルをアップロードする際、`Cache-Control` を明示的に指定しないと、Cloud Storageのデフォルト挙動に委ねられます。[公式ドキュメント](https://cloud.google.com/storage/docs/metadata#cache-control)によると、未指定時のデフォルトは条件によって異なりますが、意図したキャッシュ戦略を確実に適用するには明示的な指定が必要です。

| 設定 | 共有キャッシュ（CDN等） | 挙動 |
|------|:---:|------|
| `private` | ✕ | 共有キャッシュを禁止 |
| 未指定 | 条件次第 | Cloud Storageのデフォルト挙動に依存 |
| `public, max-age=31536000` | ○ | 明示的にCDN/プロキシでのキャッシュを許可 |

LINEサーバーのような外部サービスが画像URLにアクセスする場合、共有キャッシュが効かないとオリジンへのリクエストが増加します。`public, max-age=31536000`（1年）を明示的に設定することで、キャッシュ戦略を確実にコントロールできます。

:::message
Firebase Storageの `getDownloadURL()` で取得するURLにはアクセストークンが含まれます。ただし、同一URLへのリクエストは `max-age` 期間中キャッシュから返される可能性があります。画像を差し替える場合は、download URLを再発行するか、別パスにアップロードすることでキャッシュの影響を避けられます。
:::

### ポイント2: サーバーサイドではなくクライアントサイドで圧縮する理由

画像圧縮の実行場所として、3つの選択肢を検討しました。

| 方式 | メリット | デメリット |
|------|---------|-----------|
| **Cloud Functions** | サーバー側で確実に処理 | 追加のFunctions実行コスト、レイテンシ増加 |
| **外部ライブラリ (sharp等)** | 高品質な圧縮 | バンドルサイズ増大、SSR/CSR互換性の問題 |
| **Canvas API（採用）** | ゼロ依存、ブラウザ標準API | 出力形式はブラウザ依存（JPEG/PNGは全ブラウザ対応） |

今回はLINE Messaging APIのFlex Messageで使う画像が主な対象で、**1024px以下のJPEG**に変換できれば十分です。Canvas APIはブラウザ標準で追加依存なし、`toBlob()` でJPEG品質も指定できるため、最もシンプルな選択肢でした。

## 実装Tips

### Tip 1: Firebase Storage の `uploadBytes()` に Cache-Control を渡す

Firebase SDKの `uploadBytes()` は第3引数にメタデータを受け取ります。

```typescript
import { ref, uploadBytes, getDownloadURL } from "firebase/storage";

async function uploadToStorage(
  path: string,
  file: File | Blob
): Promise<string> {
  const storageRef = ref(storage, path);
  await uploadBytes(storageRef, file, {
    cacheControl: "public, max-age=31536000",
  });
  return getDownloadURL(storageRef);
}
```

:::message
`uploadBytes` のメタデータは `UploadMetadata` 型で、`cacheControl` の他に `contentType`, `contentDisposition` なども設定できます。画像・動画・音声すべてのアップロードがこの関数を経由するなら、一箇所の変更で全メディアに適用されます。
:::

### Tip 2: Canvas APIで画像を圧縮する関数

以下の関数は、画像を最大1024pxにリサイズし、JPEG品質85%で圧縮します。

```typescript
const MAX_SIZE = 1024;
const QUALITY = 0.85;
const SKIP_THRESHOLD = 300 * 1024; // 300KB

async function compressImage(file: File): Promise<File> {
  const dataUrl = await readFileAsDataUrl(file);
  const img = await loadImage(dataUrl);

  // Skip if already small enough (JPEG, ≤300KB, ≤1024px)
  if (
    file.type === "image/jpeg" &&
    file.size <= SKIP_THRESHOLD &&
    img.width <= MAX_SIZE &&
    img.height <= MAX_SIZE
  ) {
    return file;
  }

  const scale = Math.min(MAX_SIZE / img.width, MAX_SIZE / img.height, 1);
  const width = Math.max(1, Math.round(img.width * scale));
  const height = Math.max(1, Math.round(img.height * scale));

  const canvas = document.createElement("canvas");
  canvas.width = width;
  canvas.height = height;

  const ctx = canvas.getContext("2d")!;
  // Fill white background before drawing
  ctx.fillStyle = "#ffffff";
  ctx.fillRect(0, 0, width, height);
  ctx.drawImage(img, 0, 0, width, height);

  const blob = await new Promise<Blob>((resolve, reject) => {
    canvas.toBlob(
      (b) => (b ? resolve(b) : reject(new Error("Compression failed"))),
      "image/jpeg",
      QUALITY
    );
  });

  return new File([blob], file.name.replace(/\.[^.]+$/, ".jpg"), {
    type: "image/jpeg",
  });
}
```

このコードにはいくつかの工夫が含まれています。

**スキップ判定**: 既に小さいJPEG画像（300KB以下、1024px以下）は圧縮をスキップして無駄な再エンコードを避けます。

**`Math.max(1, ...)`**: 極端なアスペクト比（例: 10000×1px）の画像で `Math.round()` が0を返すのを防ぎます。Canvasのサイズが0だと `toBlob()` が `null` を返して失敗します。

**白背景の塗りつぶし**: JPEGは透過をサポートしないため、PNG画像の透明部分がそのままだと黒くなります。

### Tip 3: 透過PNGの落とし穴

Canvas APIでJPEGに変換する際、最も見落としやすいのが**透過PNGの扱い**です。

```diff
  const ctx = canvas.getContext("2d")!;
+ // Fill white background to avoid black areas from transparent PNGs
+ ctx.fillStyle = "#ffffff";
+ ctx.fillRect(0, 0, width, height);
  ctx.drawImage(img, 0, 0, width, height);
```

`fillRect` を入れないと、透過部分がこうなります:

| | 透過部分の色 |
|---|---|
| `fillRect` なし | **黒**（Canvasのデフォルト） |
| `fillRect` あり | **白**（自然な見た目） |

ロゴやイラスト系の画像を扱う場合は特に注意が必要です。

### Tip 4: base64 APIペイロードへの適用

自前のAPIサーバーに画像をbase64で送信するケース（例: サーバー側でCloud Storageにアップロードして画像URLを返す構成）でも、事前圧縮は効果的です。

:::message
LINE Messaging API自体は画像をHTTPS URLで指定する仕様です。ここで言うbase64送信は、自前のバックエンドAPIに画像データを渡すケースを指しています。
:::

```typescript
async function uploadSurveyImage(file: File): Promise<string> {
  // Compress before base64 encoding
  const compressed = await compressImage(file);

  const reader = new FileReader();
  const base64 = await new Promise<string>((resolve, reject) => {
    reader.onload = () => {
      const result = reader.result as string;
      resolve(result.split(",")[1] ?? "");
    };
    reader.onerror = reject;
    reader.readAsDataURL(compressed);
  });

  const { url } = await api.upload(base64, "image/jpeg");
  return url;
}
```

元画像が3MBのPNGなら、base64エンコード後は約4MB。圧縮後の200KB JPEGなら約270KB。APIリクエストのペイロードが**1/15以下**になります。

## まとめ

### 対策一覧

| 対策 | Before | After | 効果 |
|------|--------|-------|------|
| Cache-Control設定 | 未指定（挙動が不確定） | `public, max-age=31536000` | 共有キャッシュを明示的に有効化 |
| 画像圧縮 | 元画像（数MB） | max 1024px JPEG 85%（100-300KB） | ダウンロード時間短縮 |
| base64圧縮 | 元画像のbase64 | 圧縮済みbase64 | APIペイロード削減 |

### 学び

1. **Firebase Storageの`Cache-Control`を明示的に設定する** — 未指定だとCloud Storageのデフォルト挙動に委ねられ、意図したキャッシュ戦略にならない場合がある。`uploadBytes` のメタデータで簡単に指定できる
2. **Canvas APIは十分実用的** — ライブラリなしで画像リサイズ＋JPEG変換＋品質調整が可能。LINE配信レベルの画質なら問題なし
3. **透過PNGのJPEG変換は白背景を忘れない** — `fillRect` 一行で防げるが、見落としやすい
4. **スキップ判定で無駄な処理を避ける** — 既に小さい画像の再エンコードは品質劣化を招くだけ

### 実装チェックリスト

- [ ] `uploadBytes()` に `cacheControl` メタデータを設定しているか
- [ ] 圧縮対象の最大サイズ・品質は用途に合っているか（LINE: 1024px, 85%）
- [ ] 既に小さい画像のスキップ判定を入れているか
- [ ] 透過PNG → JPEG変換で白背景を塗っているか
- [ ] `Math.max(1, ...)` で極端なアスペクト比に対応しているか
- [ ] base64送信時にも圧縮を適用しているか
