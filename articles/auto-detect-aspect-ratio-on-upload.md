---
title: "画像・動画のアスペクト比をアップロード時に自動検出する実装パターン"
emoji: "📐"
type: "tech"
topics: ["typescript", "javascript", "canvas", "nextjs", "line"]
published: true
---

## はじめに

画像や動画をアップロードした後、ユーザーにアスペクト比を手動で選ばせるUIは地味にストレスです。「この画像は16:9？4:3？」と迷わせる代わりに、ファイルから自動検出して初期値をセットすれば、ほとんどの場合ワンクリック省略できます。

:::message
**対象読者**: Canvas API / Video API の基本を知っていて、画像・動画アップロード機能を持つWebアプリを開発している方
:::

この記事で得られる知見：

- ブラウザ上で画像・動画の実寸サイズを取得する方法
- 任意の縦横比を「最も近いプリセット」にマッピングするアルゴリズム
- 非同期検出とUI更新を安全に組み合わせるパターン
- TypeScript の非空タプル型で空配列を型レベルで防ぐテクニック

## 全体像

```
┌──────────┐     ┌───────────────┐     ┌──────────────────┐     ┌────────────┐
│ ファイル  │────▶│ 寸法を取得     │────▶│ 最も近い比率を   │────▶│ UIに反映    │
│ 選択      │     │ (img/video)   │     │ プリセットから検索│     │ (初期値)   │
└──────────┘     └───────────────┘     └──────────────────┘     └────────────┘
                   ↓                      ↓
                 width: 1170            候補: ['16:9', '4:3', '1:1', ...]
                 height: 2532           結果: '9:19.5' (iPhone縦)
```

ユーザー視点では「画像を選んだら比率が自動で選ばれている」だけ。手動で変更したければドロップダウンから選び直せます。

## 設計のポイント

### ポイント1: GCD簡約 vs プリセットマッチング

画像の縦横比を文字列にする方法は2つあります。

| 方式 | 例（1920×1080） | メリット | デメリット |
|------|----------------|---------|-----------|
| GCD簡約 | `"16:9"` | 正確 | `1920×1007` → `"1920:1007"` になる |
| プリセットマッチング | `"16:9"` | UIの選択肢と一致する | 近似値になる |

実用上は**プリセットマッチング**が圧倒的に便利です。理由：

- UIのドロップダウンにある選択肢と一致するので、値がそのまま使える
- ユーザーが「16:9」と見て直感的に理解できる
- LINE Flex Message など外部APIが受け付ける比率は固定リストであることが多い

:::message
GCD簡約は「正確だが実用的でない」ケースが多いです。`800×601` → `"800:601"` のような比率は誰も理解できません。プリセットマッチングなら `"4:3"` になります。
:::

### ポイント2: 画像 vs 動画で取得方法が違う

画像と動画では寸法の取得方法が異なります。

| メディア | 使うAPI | 寸法プロパティ | 非同期イベント |
|---------|---------|--------------|--------------|
| 画像 | `HTMLImageElement` | `img.naturalWidth`, `img.naturalHeight` | `onload` |
| 動画 | `HTMLVideoElement` | `video.videoWidth`, `video.videoHeight` | `loadedmetadata` |

画像は `width` / `height` でなく **`naturalWidth` / `naturalHeight`** を使うことで、CSS等による表示サイズではなく本来の解像度を確実に取得できます。動画の寸法は `loadedmetadata` で取得可能です（サムネイル生成には別途 `seeked` が必要）。

## 実装Tips

### Tip 1: プリセットから最も近い比率を見つける

核となるアルゴリズムです。候補の比率文字列（`"16:9"`, `"1.91:1"` など）をパースして数値化し、実際の縦横比との差が最小のものを返します。

```typescript
function parseRatioValue(ratio: string): number {
  const [w, h] = ratio.split(':')
  return Number(w) / Number(h)
}

function findClosestAspectRatio<T extends string>(
  width: number,
  height: number,
  candidates: readonly [T, ...T[]]
): T {
  const actual = width / height
  let closest = candidates[0]
  let minDiff = Infinity
  for (const candidate of candidates) {
    const diff = Math.abs(parseRatioValue(candidate) - actual)
    if (diff < minDiff) {
      minDiff = diff
      closest = candidate
    }
  }
  return closest
}
```

使用例：

```typescript
const RATIOS = ['16:9', '4:3', '1:1', '3:4', '9:16', '9:19.5'] as const

// 1920×1080 の画像
findClosestAspectRatio(1920, 1080, RATIOS)  // → '16:9'

// iPhoneスクリーンショット (1170×2532)
findClosestAspectRatio(1170, 2532, RATIOS)  // → '9:19.5'

// 正方形に近い画像 (800×750)
findClosestAspectRatio(800, 750, RATIOS)    // → '1:1'
```

:::message
**`readonly [T, ...T[]]`（非空タプル型）がポイント。** `readonly T[]` だと空配列も許容されてしまい、`candidates[0]` が `undefined` になる可能性があります。非空タプル型にすることで `candidates[0]` が必ず `T` であることを型レベルで保証でき、`!` アサーションが不要になります。
:::

### Tip 2: 画像の寸法を取得する

ブラウザ上で画像ファイルの実寸サイズを取得するには、`FileReader` → `HTMLImageElement` の2段階を踏みます。

```typescript
function readFileAsDataUrl(file: File): Promise<string> {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()
    reader.onload = () => resolve(reader.result as string)
    reader.onerror = () => reject(new Error('Failed to read file'))
    reader.readAsDataURL(file)
  })
}

function loadImage(src: string): Promise<HTMLImageElement> {
  return new Promise((resolve, reject) => {
    const img = new Image()
    img.onload = () => resolve(img)
    img.onerror = () => reject(new Error('Failed to load image'))
    img.src = src
  })
}

// 使用例
const dataUrl = await readFileAsDataUrl(file)
const img = await loadImage(dataUrl)
console.log(img.naturalWidth, img.naturalHeight)  // → 1920, 1080
```

:::message
`File` → `DataURL` → `HTMLImageElement` という変換チェーンは、ブラウザ上の画像処理では定番パターンです。Canvas でのリサイズやサムネイル生成でも同じ流れを使います。寸法は `naturalWidth` / `naturalHeight` を使うことで、DOMに挿入せずとも本来の解像度を取得できます。
:::

### Tip 3: 動画の寸法を取得する

動画の場合は `HTMLVideoElement` を使います。**寸法だけなら `loadedmetadata` で十分**です。サムネイル生成（Canvasへの描画）が必要な場合のみ `seeked` まで待つ必要があります。

```typescript
// 寸法だけ取得する場合
function getVideoDimensions(
  file: File
): Promise<{ width: number; height: number }> {
  return new Promise((resolve, reject) => {
    const video = document.createElement('video')
    video.preload = 'metadata'

    const url = URL.createObjectURL(file)
    video.src = url

    video.onloadedmetadata = () => {
      const width = video.videoWidth
      const height = video.videoHeight
      URL.revokeObjectURL(url)
      resolve({ width, height })
    }

    video.onerror = () => {
      URL.revokeObjectURL(url)
      reject(new Error('Failed to load video'))
    }
  })
}
```

サムネイル生成も兼ねる場合は、`loadeddata` → `seek` → `seeked` → Canvas描画 の流れになります。その場合は `seeked` イベント内で `videoWidth` / `videoHeight` も一緒に取得するのが効率的です。

:::message
**`URL.revokeObjectURL()` を忘れずに。** `createObjectURL` で作成した URL はブラウザのメモリを占有し続けます。使い終わったら必ず `revokeObjectURL` で解放してください。
:::

### Tip 4: 既存のアップロード処理に寸法を組み込む

画像圧縮やアップロード処理の中で `HTMLImageElement` を既に生成しているなら、そこから寸法を取り出して戻り値に含めるのが最も効率的です。

```diff
- async function compressImage(file: File): Promise<File> {
+ async function compressImage(
+   file: File
+ ): Promise<{ file: File; width: number; height: number }> {
    const dataUrl = await readFileAsDataUrl(file)
    const img = await loadImage(dataUrl)
+   const origWidth = img.naturalWidth
+   const origHeight = img.naturalHeight

    // ... compression logic using canvas ...

-   return compressedFile
+   return { file: compressedFile, width: origWidth, height: origHeight }
  }
```

こうすることで、呼び出し側は追加の画像読み込みなしに寸法を取得できます。

```typescript
async function uploadImage(file: File) {
  const { file: compressed, width, height } = await compressImage(file)
  const url = await upload(compressed)
  const ratio = findClosestAspectRatio(width, height, ASPECT_RATIOS)
  return { url, width, height, ratio }
}
```

## ハマりポイント

### 非同期検出とUIの競合

画像選択後に非同期で比率を検出してstateを更新する場合、ユーザーが素早く別の画像を選び直すと **古い検出結果が新しい画像のstateを上書き** する危険があります。

```diff
  function handleImageSelect(index: number, file: File) {
    const previewUrl = URL.createObjectURL(file)
    // 即座にファイル情報を反映（古いpreview URLは適宜revokeする）
    updateItem(index, (current) => {
      if (current.preview) URL.revokeObjectURL(current.preview)
      return { ...current, file, preview: previewUrl }
    })

    // 非同期で比率を検出
    detectAspectRatio(file)
      .then((ratio) => {
-       // 危険: 別のファイルが既に選択されている可能性
-       updateItem(index, { aspectRatio: ratio })
+       // ガード: 同じファイルの場合のみ更新
+       updateItem(index, (current) =>
+         current.file === file ? { ...current, aspectRatio: ratio } : current
+       )
      })
      .catch(() => { /* silent fallback */ })
  }
```

:::message
**関数型の state 更新 + ファイル参照の同一性チェック** で、レースコンディションを防ぎます。`current.file === file` の参照比較により、検出開始時と完了時で同じファイルであることを保証します。
:::

### iPhone のアスペクト比は 16:9 ではない

モバイル対応で見落としがちなポイントです。

| デバイス | 解像度 | 比率 |
|---------|--------|-----|
| iPhone 16 Pro Max | 2868 × 1320 | ≈ 19.5:9 |
| iPhone 16 Pro | 2622 × 1206 | ≈ 19.5:9 |
| iPhone 16e | 2532 × 1170 | ≈ 19.5:9 |
| 一般的なPC画面 | 1920 × 1080 | 16:9 |

フルスクリーン系の iPhone（X 以降の Face ID モデル）のスクリーンショットや画面録画は約 **19.5:9**（縦なら **9:19.5**）で、`9:16` とは大きく異なります。プリセットに `9:16` しか用意していないと、iPhone の縦動画が不自然にトリミングされます。

:::message
**例外**: iPhone SE（第3世代以前）は 1334×750 で実質 **16:9** です。SE系を含むサービスでは両方の比率をプリセットに入れておくと安全です。
:::

```typescript
// プリセットにiPhone比率を追加
const ASPECT_RATIOS = [
  '20:13', '19.5:9', '1.91:1', '16:9', '4:3',
  '1:1',
  '3:4', '2:3', '9:16', '9:19.5',
] as const
```

## まとめ

### 技術選定・設計判断

| 判断 | 選択 | 理由 |
|------|------|------|
| 比率の決定方式 | プリセットマッチング | UIの選択肢と一致、外部APIの制約に適合 |
| 寸法の取得タイミング | 既存処理に組み込み | 追加の読み込みコストなし |
| 検出のUI反映 | 非同期 + ガード | ファイル選択のレスポンスを優先 |
| 型安全性 | 非空タプル型 | 空配列の `undefined` アクセスを型レベルで防止 |

### 学び

1. **GCD簡約より近似マッチングの方が実用的** — 正確さより「UIで使える」ことが重要
2. **既存処理の戻り値を拡張するのが最も効率的** — 画像の `onload` は既にどこかで発生している
3. **非同期更新にはファイル参照ガードが必須** — ユーザーの操作速度を甘く見ない
4. **モバイルのアスペクト比を忘れない** — iPhone は 16:9 ではなく 19.5:9

### 実装チェックリスト

- [ ] プリセットの比率リストにモバイル向け比率（`9:19.5` 等）を含めたか
- [ ] `findClosestAspectRatio` の候補リストは空にならないか（非空タプル型推奨）
- [ ] 動画の寸法取得で `loadedmetadata` を待っているか（サムネイル生成時は `seeked`）
- [ ] `URL.createObjectURL()` の戻り値を `revokeObjectURL()` で解放しているか
- [ ] 非同期の比率検出で、ファイル差し替え時のレースコンディションを防いでいるか
- [ ] 自動検出後もユーザーが手動で変更できるUIになっているか
