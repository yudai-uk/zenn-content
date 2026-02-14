---
title: "1500行のモノリスを75行に — LINE リッチメニュービルダーをウィザード形式にリファクタリングした話"
emoji: "🧩"
type: "tech"
topics: ["React", "NextJS", "LINE", "リファクタリング", "TypeScript"]
published: true
---

## はじめに

LINE Messaging API のリッチメニューを管理する画面を作っていたら、気づけば **1つのファイルが1,557行** になっていました。キャンバス操作、フォーム管理、API呼び出し、領域描画ロジックが全部1コンポーネントに同居している状態です。

この記事では、そのモノリスを **14コンポーネント + 75行のオーケストレータ** に分割し、5ステップのウィザード形式UIに再構成した過程を紹介します。

:::message
**対象読者**
- React / Next.js で管理画面を作っている方
- 巨大コンポーネントのリファクタリングに悩んでいる方
- LINE リッチメニューの管理画面を実装したい方
:::

**この記事で得られる知見:**
- `useReducer` でウィザード全体の状態をフラットに管理するパターン
- キャンバスエディタを分離する際のイベント委譲設計
- LINE API の画像アップロードで踏む落とし穴と対処法
- 非エンジニアにも使えるUI改善の具体的アプローチ

## 全体像

### Before: モノリス構成

```
page.tsx (1,557行)
├── 型定義 (50行)
├── ヘルパー関数 (200行)
├── コンポーネント本体 (1,300行)
│   ├── 状態管理 (useState x 20個)
│   ├── キャンバスイベントハンドラ
│   ├── フォーム送信ロジック
│   └── JSX (800行)
```

### After: ウィザード構成

```
page.tsx (75行 — オーケストレータ)
├── RichMenuList          # メニュー一覧テーブル
├── RichMenuCreator       # ウィザードコンテナ
│   ├── LayoutTemplateSelector  # Step 1: レイアウト選択
│   ├── CanvasEditor            # Step 2-3: 画像 + エリア編集
│   ├── AreaToolbar             # ツールバー
│   ├── AreaInspector           # サイドパネル
│   │   └── ActionConfigurator  # アクション設定カード
│   └── CreateTestMenuButton    # テストメニューワンクリック
├── SetDefaultForm        # デフォルト設定 (ドロップダウン)
└── LinkToUserForm        # ユーザー紐付け (ドロップダウン)

共有基盤:
├── types.ts              # 型定義
├── utils.ts              # ヘルパー関数
├── templates.ts          # レイアウトテンプレート
└── useRichMenuWizard.ts  # useReducer ベースの状態管理
```

ウィザードのフローは以下の5ステップです:

```
[レイアウト] → [画像] → [アクション] → [基本設定] → [確認・作成]
```

## 設計のポイント

### ポイント1: なぜ useState x 20 ではなく useReducer か

元のコードでは `useState` が20個並んでいました:

```tsx
// Before: 状態が散らばっていて関連性が見えない
const [formName, setFormName] = useState("");
const [formAlias, setFormAlias] = useState("");
const [areas, setAreas] = useState<DraftArea[]>([]);
const [selectedAreaId, setSelectedAreaId] = useState<string | null>(null);
const [interaction, setInteraction] = useState<InteractionState>({ mode: "idle" });
const [zoom, setZoom] = useState(1);
// ... あと14個
```

| 方式 | メリット | デメリット |
|------|---------|-----------|
| `useState` x N | シンプル、個別更新が楽 | 状態間の整合性が保証できない |
| `useReducer` | 状態遷移が明示的、整合性を保証可能 | ボイラープレートが増える |
| 外部ライブラリ (Zustand等) | 柔軟 | この規模では過剰 |

:::message
**`useReducer` を選んだ決定的な理由**: サイズ変更時に `areas` と `selectedAreaId` を同時にリセットする必要がある。`useState` だと片方だけ更新して不整合が起きるリスクがある。
:::

```tsx
// After: 状態遷移がアトミック
function reducer(state: WizardState, action: WizardAction): WizardState {
  switch (action.type) {
    case "SET_MENU_SIZE":
      // サイズ変更 → テンプレート・エリア・選択を同時リセット
      return {
        ...state,
        menuSize: action.size,
        selectedTemplateId: null,
        areas: [],
        selectedAreaId: null,
      };
    // ...
  }
}
```

### ポイント2: キャンバスエディタの分離戦略

キャンバス操作（ドラッグで領域作成・移動・リサイズ）は最も複雑なロジックです。分離のポイントは**イベントハンドリングをコンポーネント内に閉じ、外部にはコールバックだけ公開する**ことです。

```tsx
// CanvasEditor のインターフェース
interface CanvasEditorProps {
  // 入力 (読み取り専用データ)
  areas: DraftArea[];
  selectedAreaId: string | null;
  interaction: InteractionState;
  imagePreviewUrl: string;
  canvasWidth: number;
  canvasHeight: number;

  // 出力 (コールバック)
  onAreasChange: (areas: DraftArea[]) => void;
  onSelectedAreaIdChange: (id: string | null) => void;
  onInteractionChange: (state: InteractionState) => void;
}
```

PointerEvent の処理（座標変換、スナップ、リサイズ計算）はすべて `CanvasEditor` 内に閉じます。親コンポーネントは「エリア配列が変わった」「選択が変わった」というコールバックを受け取るだけです。

:::message
キャンバスのような複雑なインタラクションコンポーネントを分離するコツは、**内部状態を最小限にして、残りは props + callback で外部管理**にすること。`ref` だけは内部で持ちます。
:::

### ポイント3: テンプレート選択でワンクリック設定

元のUIでは、ユーザーが Shift+ドラッグでエリアを1つずつ描画する必要がありました。非エンジニアにとってはかなりのハードルです。

テンプレート選択を導入し、**ワンクリックでレイアウトが完成する**ようにしました:

```tsx
// テンプレートからエリアを生成するヘルパー
function generateGrid(cols: number, rows: number, width: number, height: number) {
  const cellW = Math.floor(width / cols);
  const cellH = Math.floor(height / rows);
  const areas = [];

  for (let row = 0; row < rows; row++) {
    for (let col = 0; col < cols; col++) {
      areas.push({
        x: col * cellW,
        y: row * cellH,
        // 最後の列/行は端数を吸収
        width: col === cols - 1 ? width - col * cellW : cellW,
        height: row === rows - 1 ? height - row * cellH : cellH,
      });
    }
  }
  return areas;
}
```

テンプレートには SVG プレビューを付けて、選択前にレイアウトがわかるようにしています。

## 実装Tips

### Tip 1: LINE画像アップロードのドメイン問題

LINE Messaging API でリッチメニュー画像をアップロードする際、**通常のAPIとはドメインが異なります**。

```diff
- const LINE_API_BASE = 'https://api.line.me/v2/bot';
+ const LINE_API_BASE = 'https://api.line.me/v2/bot';
+ const LINE_DATA_API_BASE = 'https://api-data.line.me/v2/bot';
```

| エンドポイント | ドメイン |
|--------------|---------|
| リッチメニュー作成 | `api.line.me` |
| **画像アップロード** | `api-data.line.me` |
| エイリアス登録 | `api.line.me` |

通常のドメインに画像を送ると **404** が返ります。LINE公式ドキュメントにも書いてありますが、見落としがちなポイントです。

### Tip 2: 画像の自動リサイズで413エラーを防ぐ

LINE API の画像アップロードには **1MB** のサイズ制限があります。ユーザーがスマホで撮った写真をそのままアップロードすると確実に超えます。

クライアント側で Canvas API を使い、**メニューサイズに合わせてリサイズ + JPEG圧縮** を行います:

```tsx
async function resizeImageForMenu(
  file: File,
  targetWidth: number,
  targetHeight: number
): Promise<{ dataUrl: string; base64: string; contentType: string }> {
  const img = await loadImage(file);
  const canvas = document.createElement("canvas");
  canvas.width = targetWidth;
  canvas.height = targetHeight;
  canvas.getContext("2d")!.drawImage(img, 0, 0, targetWidth, targetHeight);

  // JPEG品質を下げながら1MB以下になるまで試行
  for (let quality = 0.92; quality >= 0.3; quality -= 0.1) {
    const jpeg = canvas.toDataURL("image/jpeg", quality);
    const base64 = jpeg.split(",")[1] ?? "";
    if (Math.ceil(base64.length * 0.75) <= 1_000_000) {
      return { dataUrl: jpeg, base64, contentType: "image/jpeg" };
    }
  }
  // フォールバック
  const jpeg = canvas.toDataURL("image/jpeg", 0.3);
  return { dataUrl: jpeg, base64: jpeg.split(",")[1] ?? "", contentType: "image/jpeg" };
}
```

:::message
元の画像フォーマットが PNG でも、LINE API 向けには JPEG で送るのがサイズ効率的です。リッチメニュー画像は写真やイラストが多く、JPEG の圧縮アーティファクトが目立ちにくいためです。
:::

### Tip 3: Tailwind CSS の動的クラスに注意

リサイズハンドルで `cursor-${handle}-resize` のように動的にクラス名を生成すると、**Tailwind が CSS を生成してくれません**。

```tsx
// NG: 動的クラスはビルド時に検出されない
className={`cursor-${handle}-resize`}

// OK: 静的クラスを直接書く
<button className="cursor-nw-resize ..." />
<button className="cursor-ne-resize ..." />
<button className="cursor-sw-resize ..." />
<button className="cursor-se-resize ..." />
```

Tailwind はビルド時にソースコードを静的解析してクラス名を抽出するため、テンプレートリテラルで組み立てたクラス名は認識できません。`safelist` に追加するか、静的に書く必要があります。

### Tip 4: アクション設定をカード形式にして選択ミスを防ぐ

元のUIでは、アクション種別が `<select>` の中に `message`, `uri`, `postback` と英語で並んでいました。非エンジニアには何が何だかわかりません。

**カード形式UI**に変更し、各アクションに日本語ラベル・説明文・アイコンを付けました:

```tsx
const ACTION_TYPE_META = [
  {
    type: "message",
    label: "メッセージ送信",
    description: "タップ時にテキストメッセージを送信します",
    icon: "💬",
  },
  {
    type: "uri",
    label: "URLを開く",
    description: "タップ時に指定したURLをブラウザで開きます",
    icon: "🔗",
  },
  // ...
];
```

技術的な名称をそのまま出すのではなく、**ユーザーが「何が起きるか」を想像できるラベル** にすることが大事です。

## ハマりポイント

### useReducer でオプショナルフィールドの扱い

アクション設定で入力欄を空にしたとき、`value || undefined` で `undefined` に変換していました:

```tsx
// NG: 必須フィールドまで undefined になる
function updateField(key: string, value: string) {
  onUpdate({ ...action, [key]: value || undefined });
}
```

バリデーションで `.trim()` を呼ぶと `Cannot read properties of undefined` でクラッシュします。

```diff
- function updateField(key: string, value: string) {
-   onUpdate({ ...action, [key]: value || undefined });
- }
+ const OPTIONAL_FIELDS = new Set(["label", "displayText"]);
+
+ function updateField(key: string, value: string) {
+   const resolved = OPTIONAL_FIELDS.has(key) ? value || undefined : value;
+   onUpdate({ ...action, [key]: resolved });
+ }
```

**必須フィールドは空文字を許容し、バリデーションで弾く**。`undefined` にするのはオプショナルフィールドだけです。

## まとめ

### 設計判断の一覧

| 判断 | 選択 | 理由 |
|------|------|------|
| 状態管理 | `useReducer` | 複数状態の同時更新が必要 |
| ウィザードステップ管理 | reducer内のステップ遷移 | バリデーションと状態を一元管理 |
| キャンバス分離 | props + callback パターン | 内部ロジックを閉じ、外部は宣言的に利用 |
| アクション設定UI | カード形式 | 非エンジニアの理解しやすさ |
| 画像処理 | クライアント側リサイズ | LINE API 1MB制限の事前回避 |
| ID入力 | ドロップダウン選択 | 入力ミス防止 |

### 学び

1. **1500行超えたら赤信号**。`useState` が10個超えたらリファクタリングのサイン
2. **`useReducer` は「複数状態の整合性」が必要な場面で真価を発揮する**
3. **キャンバス系コンポーネントは「内部にロジック、外部にコールバック」で分離する**
4. **LINE API は通常ドメインとデータドメインが別**。画像系は `api-data.line.me`
5. **非エンジニアが使うUIでは、技術用語をそのまま出さない**

### 実装チェックリスト

- [ ] `useState` が10個以上ある → `useReducer` への移行を検討
- [ ] 複数の状態を同時に更新する箇所がある → reducer のアクションで一括更新
- [ ] キャンバス/ドラッグ操作がある → イベント処理をコンポーネント内に閉じる
- [ ] LINE API で画像をアップロードする → `api-data.line.me` を使用
- [ ] ユーザーがアップロードする画像 → クライアント側でリサイズ
- [ ] Tailwind で動的クラス名を生成している → 静的クラスに置き換え
