---
title: "LINE リッチメニューの高さを自由に設定できるようにする — プリセット＋カスタムの設計パターン"
emoji: "📐"
type: "tech"
topics: ["LINE", "React", "NextJS", "TypeScript", "useReducer"]
published: true
---

## はじめに

LINE Messaging API のリッチメニューを管理画面から作成できるようにしていたのですが、高さが「標準（2500×1686）」と「コンパクト（2500×843）」の2択しかありませんでした。

LINE の公式リファレンスでは height の最小値が 250、アスペクト比が 1.45 以上という制約が記載されていますが、LINE 公式管理画面や SDK のサンプルでは 1686 と 843 のプリセットしか提示されていません。この記事では、**プリセット2択にカスタムオプションを追加**して、250〜1724px の範囲で自由に高さを設定できるようにした実装を紹介します。

:::message
**対象読者**
- LINE リッチメニューの管理画面を実装している方
- `useReducer` でウィザード型UIの状態管理をしている方
- プリセット＋カスタム入力の UI パターンに興味がある方
:::

**この記事で得られる知見:**
- LINE API のリッチメニューサイズの実際の制約
- プリセット＋カスタムを型安全に共存させる設計
- `useReducer` にアクションを追加する際のエッジケース対策
- Zod スキーマをプリセットから範囲バリデーションに変更する方法

## 全体像

```
ユーザーの操作フロー:

[サイズ選択] ─── 標準 (2500×1686)
      │
      ├──── コンパクト (2500×843)
      │
      └──── カスタム ──→ [スライダー + 数値入力]
                              │
                              ▼
                         250 ~ 1724 px
                              │
                              ▼
                    [テンプレート選択]
                              │
                              ▼
                   [以降のウィザードステップ]
```

フロントエンドは React + Next.js のウィザード形式UI、バックエンドは Express + Zod で構成しています。変更は以下の5ファイルで完結し、既存のプリセット機能はそのまま維持されます。

| ファイル | 変更内容 |
|---------|---------|
| `types.ts` | `MenuSize` 型拡張、定数追加 |
| `useRichMenuWizard.ts` | reducer にアクション追加、寸法取得ロジック変更 |
| `templates.ts` | 関数シグネチャを `(width, height)` に変更 |
| `LayoutTemplateSelector.tsx` | カスタム高さ UI 追加 |
| `richMenuSchema.ts`（バックエンド） | Zod バリデーションを範囲指定に変更 |

## 設計のポイント

### ポイント1: LINE API の実際のサイズ制約

LINE Messaging API のリッチメニューサイズは、以下の制約があります：

| パラメータ | 制約 |
|-----------|------|
| width | 800〜2500 |
| height | 250 以上 |
| アスペクト比 (width / height) | 1.45 以上 |

LINE 公式管理画面や SDK のサンプルでは 2500×1686 と 2500×843 のプリセットしか提示されていませんが、API 自体はこの範囲内であれば任意の値を受け入れます。

:::message
本アプリでは width を 2500 に固定しているため、最大 height は `floor(2500 / 1.45)` = **1724px** になります。width を変えるなら、最大 height も `floor(width / 1.45)` で再計算が必要です。
:::

### ポイント2: Union 型でプリセット＋カスタムを共存させる

カスタムサイズを追加するにあたり、2つのアプローチを検討しました。

| アプローチ | メリット | デメリット |
|-----------|---------|-----------|
| A: `MenuSize` に `"custom"` を追加 | 既存のプリセット処理がそのまま使える。Union 型に1つ追加するだけで済む | `customHeight` の追加管理が必要 |
| B: `menuSize` を廃止して常に数値で管理 | 型がシンプル | プリセットの UX が失われる。既存コード全体の書き換えが必要 |

**アプローチ A** を採用しました。型定義はこうなります：

```typescript
// Before
type MenuSize = "standard" | "compact";

// After
type MenuSize = "standard" | "compact" | "custom";
```

`WizardState` に `customHeight` フィールドを追加し、`menuSize === "custom"` の場合のみ参照します：

```typescript
interface WizardState {
  menuSize: MenuSize;
  customHeight: number; // "custom" 選択時のみ使用
  // ... other fields
}
```

:::message
**なぜ `customHeight` を常に保持するか？**
ユーザーがカスタムで高さを調整した後、一度「標準」に切り替えて戻ってきたとき、前回の値が残っている方が UX が良いためです。
:::

### ポイント3: テンプレート生成を width/height ベースに変更

もともとテンプレート生成は `MenuSize` を受け取って内部で寸法に変換していました。カスタムサイズ対応にあたり、直接 `width` と `height` を受け取るように変更します：

```typescript
// Before
function getTemplatesForSize(menuSize: MenuSize): LayoutTemplate[] {
  const { width, height } = SIZES[menuSize];
  // ...
}

// After
function getTemplatesForSize(width: number, height: number): LayoutTemplate[] {
  // width, height を直接使ってグリッドを生成
}
```

これにより、呼び出し側がサイズの解決を担い、テンプレート生成は純粋な寸法計算に専念できます。

## 実装 Tips

### Tip 1: getMenuDimensions のシグネチャ変更

`menuSize` だけでは `customHeight` にアクセスできないため、`WizardState` 全体を渡すように変更します：

```typescript
// Before
function getMenuDimensions(size: MenuSize) {
  return MENU_SIZES[size];
}

// After
function getMenuDimensions(state: WizardState) {
  if (state.menuSize === "custom") {
    return { width: 2500, height: state.customHeight };
  }
  return MENU_SIZES[state.menuSize];
}
```

reducer 内の `ADD_AREA` や `DUPLICATE_AREA` など、寸法を参照するすべてのケースで呼び出しを更新します。

### Tip 2: スライダー + 数値入力の UI

カスタム選択時に、スライダー（`range` input）と数値入力を組み合わせて表示します。両方を同じ dispatch に繋ぐだけで双方向に同期します：

```tsx
{menuSize === "custom" && (
  <div className="space-y-3 p-4 border rounded-lg bg-gray-50">
    <div className="flex items-center justify-between">
      <label className="text-sm font-medium">高さ</label>
      <span className="text-sm text-gray-500">
        {canvasWidth} × {canvasHeight}
      </span>
    </div>
    <input
      type="range"
      min={MIN_HEIGHT}
      max={MAX_HEIGHT}
      value={customHeight}
      onChange={(e) => onCustomHeightChange(Number(e.target.value))}
    />
    <div className="flex items-center gap-2">
      <input
        type="number"
        min={MIN_HEIGHT}
        max={MAX_HEIGHT}
        value={customHeight}
        onChange={(e) => onCustomHeightChange(Number(e.target.value))}
        className="w-24 text-center text-sm border rounded-md px-2 py-1"
      />
      <span className="text-xs text-gray-400">
        {MIN_HEIGHT} ~ {MAX_HEIGHT} px
      </span>
    </div>
  </div>
)}
```

:::message
`range` と `number` の2つの input が同じ `customHeight` 値にバインドされているため、スライダーを動かすと数値が追従し、数値を直接入力するとスライダーが追従します。特別な同期ロジックは不要です。
:::

### Tip 3: Zod スキーマをリテラル固定から範囲バリデーションに変更

バックエンドのバリデーションも柔軟にする必要があります：

```typescript
// Before
size: z.object({
  width: z.literal(2500),
  height: z.union([z.literal(1686), z.literal(843)]),
})

// After
size: z.object({
  width: z.literal(2500),
  height: z.number().int().min(250).max(1724),
})
```

`z.literal` の Union から `z.number()` のチェーンに変えるだけで、範囲内の任意の整数を受け入れるようになります。

## ハマりポイント

### NaN が伝播する問題

`<input type="number">` で値を全消しすると `e.target.value` が空文字になり、`Number("")` は `0` ではなく... 実は `0` です。しかし、入力途中で `"-"` や `"e"` だけの状態だと `NaN` になります。

```
Number("")    // → 0
Number("-")   // → NaN
Number("1e")  // → NaN
```

この `NaN` が reducer に渡ると、`customHeight` が `NaN` になり、キャンバスサイズ計算やテンプレート生成に伝播して描画が壊れます。

対策として、reducer 側で `Number.isFinite` ガードを入れます：

```typescript
case "SET_CUSTOM_HEIGHT": {
  if (!Number.isFinite(action.value)) return state;
  const clamped = Math.min(MAX_HEIGHT, Math.max(MIN_HEIGHT, Math.round(action.value)));
  // ...
}
```

### 不要なレイアウトリセットの問題

高さを変更するとテンプレートとエリアをクリアする設計にしていますが、範囲外の値を入力してクランプした結果が現在の値と同じ場合、エリアが無駄にリセットされてしまいます。

```typescript
// Bad: 毎回リセット
case "SET_CUSTOM_HEIGHT": {
  const clamped = clamp(action.value);
  return { ...state, customHeight: clamped, areas: [], selectedTemplateId: null };
}

// Good: 値が変わらないならスキップ
case "SET_CUSTOM_HEIGHT": {
  if (!Number.isFinite(action.value)) return state;
  const clamped = clamp(action.value);
  if (clamped === state.customHeight) return state; // Early return
  return { ...state, customHeight: clamped, areas: [], selectedTemplateId: null };
}
```

:::message
useReducer で「副作用のあるアクション」（エリア全消しなど）を扱う場合、**実際に値が変わるかどうかを先にチェック**する癖をつけると、意図しないデータ消失を防げます。
:::

## まとめ

### 技術判断の一覧

| 判断 | 選択 | 理由 |
|------|------|------|
| カスタムサイズの追加方法 | Union 型に `"custom"` を追加 | 既存プリセットの処理をそのまま活用できる |
| 高さの取得方法 | `getMenuDimensions(state)` に変更 | `customHeight` へのアクセスが必要 |
| テンプレート生成 | `(width, height)` を直接渡す | `MenuSize` への依存を排除 |
| バックエンドバリデーション | `z.number().int().min().max()` | 範囲内の任意の整数を受け入れ |
| NaN 対策 | `Number.isFinite` ガード | input 中間状態からの伝播を防止 |

### 学び

1. **LINE API は公式管理画面のプリセットより柔軟な値を受け入れる**。SDK サンプルの 1686/843 にとらわれず、API リファレンスの制約（height >= 250、aspect ratio >= 1.45）を直接活用することで、より自由度の高い実装が可能になる
2. **プリセット＋カスタムは Union 型の拡張で実現できる**。既存の型に1つリテラルを追加するだけで、破壊的変更を最小限に抑えられる
3. **`<input type="number">` は NaN を生む**。reducer に数値を渡す前に `Number.isFinite` で検証する
4. **副作用のあるアクションでは値の同一性チェックを先に行う**。不要なリセットを防ぎ、ユーザーの作業を守る

### 実装チェックリスト

- [ ] `MenuSize` 型にカスタムオプションを追加
- [ ] `WizardState` に `customHeight` フィールドを追加
- [ ] 寸法取得関数を `WizardState` ベースに変更
- [ ] テンプレート生成関数を `(width, height)` シグネチャに変更
- [ ] スライダー + 数値入力の UI を実装
- [ ] `Number.isFinite` ガードと同一値スキップを reducer に追加
- [ ] バックエンドの Zod スキーマを範囲バリデーションに変更
- [ ] 既存プリセット（標準・コンパクト）が従来通り動作することを確認
