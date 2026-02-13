---
title: "React Nativeでflex:1がボタンとテキストの重なりを引き起こす仕組みと対処法"
emoji: "🃏"
type: "tech"
topics: ["reactnative", "flexbox", "layout", "css"]
published: true
---

## はじめに

React Nativeでカード型UIを実装した際、**カード内のテキストとボタンが重なって表示される**レイアウト崩れに遭遇しました。

原因は `flex: 1` の誤用でした。Webの CSS Flexbox と同じ感覚で使っていたのですが、React Native の固定高さコンテナ内では思わぬ挙動になります。

:::message
**対象読者**: React NativeでFlexboxレイアウトを使っている方。特にカードUIやオーバーレイで「要素が重なる」「はみ出す」問題に心当たりのある方。
:::

## 問題の症状

スタック型カードUIの最前面カードに、質問テキスト・サンプル文・タップボタンを縦に並べて表示するデザインでした。

**期待するレイアウト:**
```
┌──────────────────┐
│                  │
│       🌍        │
│  質問テキスト      │
│  サンプル文        │
│                  │
│  [タップして書く]   │ ← ボタン
│                  │
└──────────────────┘
```

**実際のレイアウト:**
```
┌──────────────────┐
│                  │
│       🌍        │
│  質問テキスト      │
│  サンプル文テキスト  │ ← テキストとボタンが重なる
│  [タップして書く]   │ ← ボタンがテキスト領域に食い込む
│                  │
│                  │
└──────────────────┘
```

テキストの量が多いカードほど顕著に重なりが発生し、特に2行にわたる質問文+サンプル文の組み合わせで再現しました。

## 原因: 固定高さコンテナ内での `flex: 1` の挙動

### 問題のコード

```tsx:CardComponent.tsx
const styles = StyleSheet.create({
  cardOverlay: {
    flex: 1,
    backgroundColor: 'rgb(24, 48, 52)',
    padding: 16,
  },
  cardContent: {
    flex: 1,           // ← これが原因
    justifyContent: 'center',
    alignItems: 'center',
  },
  buttonContainer: {
    marginTop: 12,
    alignItems: 'center',
  },
});
```

### なぜ重なるのか

`flex: 1` はFlexboxの**残り空間をすべて占有する**指定です。ここで問題になるのは、親コンテナの高さが固定されている（例: `minHeight: 350`）場合の挙動です。

```
カード全体（高さ350px）
├── cardOverlay（flex: 1 → 350px）
│   ├── cardContent（flex: 1 → 残り全部を取ろうとする）
│   │   ├── 絵文字（54px）
│   │   ├── 質問テキスト（~60px）
│   │   └── サンプル文（~44px）
│   │   計 ~160px だが、flex:1 で ~300px に引き伸ばされる
│   └── buttonContainer（marginTop: 12）
│       └── ボタン（~50px）
│       計 ~62px
│
│   合計: 300 + 62 = 362px > 350px → はみ出し＆重なり
```

**`flex: 1` を指定した `cardContent` が利用可能な全スペースを占有しようとする**ため、後続の `buttonContainer` のスペースが圧迫されます。実際のコンテンツ高さ（~160px）よりも多くの空間を確保し、結果としてボタンとの間に余白ではなく**重なり**が生じます。

### SwiftUI では起きない理由

参照実装の SwiftUI 版では同等のレイアウトを `VStack` で実装していましたが、この問題は発生しません。

```swift
VStack(spacing: 24) {
    // コンテンツ部分
    VStack {
        Text(emoji).font(.system(size: 54))
        Text(questionText).font(.title2)
        Text(sampleAnswer).font(.body)
    }

    // ボタン
    Button("タップして書く") { ... }
}
.frame(maxHeight: .infinity)
```

SwiftUI の `VStack` は子ビューの**自然なサイズ**を尊重し、`spacing: 24` で均等な余白を確保します。`.frame(maxHeight: .infinity)` は「利用可能な最大サイズまで広がれる」という意味で、子ビューを引き伸ばすわけではありません。

一方、React Native の `flex: 1` は**利用可能なスペースを積極的に埋めに行く**動作です。この違いが重なりの原因でした。

## 修正

### 修正1: `cardContent` から `flex: 1` を削除

```diff tsx
 const styles = StyleSheet.create({
   cardContent: {
-    flex: 1,
     justifyContent: 'center',
     alignItems: 'center',
   },
 });
```

`flex: 1` を削除すると、`cardContent` は子要素の自然なサイズ（~160px）だけを占有します。ボタンのためのスペースが確保され、重なりが解消されます。

### 修正2: 親コンテナに `justifyContent: 'center'` を追加

```diff tsx
 const styles = StyleSheet.create({
   cardOverlay: {
     flex: 1,
+    justifyContent: 'center',
     backgroundColor: 'rgb(24, 48, 52)',
     padding: 16,
   },
 });
```

`cardContent` が自然な高さになったことで、カード内でコンテンツが上詰めになります。`justifyContent: 'center'` を親に追加して、コンテンツ全体（テキスト＋ボタン）を縦方向の中央に配置します。

### 修正3: ボタンの余白を調整

```diff tsx
 const styles = StyleSheet.create({
   buttonContainer: {
-    marginTop: 12,
-    marginBottom: Platform.OS === 'ios' ? 32 : 24,
+    marginTop: 24,
     alignItems: 'center',
   },
 });
```

- `marginTop` を `12` → `24` に増やし、テキストとボタンの間に十分な余白を確保
- `marginBottom` は親のパディングで十分なため削除（プラットフォーム分岐も不要に）

### 修正4: コンテナの最小高さを調整

```diff tsx
 const styles = StyleSheet.create({
   cardStackContainer: {
-    minHeight: 350,
+    minHeight: 380,
   },
   stackContainer: {
-    minHeight: 300,
+    minHeight: 340,
   },
 });
```

コンテンツが自然な高さで収まるよう、コンテナの `minHeight` を調整します。

### 修正後のレイアウト構造

```
カード全体（高さ380px）
├── cardOverlay（flex: 1, justifyContent: 'center'）
│   ├── cardContent（自然な高さ ~160px）
│   │   ├── 絵文字（54px）
│   │   ├── 質問テキスト（~60px）
│   │   └── サンプル文（~44px）
│   └── buttonContainer（marginTop: 24）
│       └── ボタン（~50px）
│
│   合計: 160 + 24 + 50 = 234px（380pxに余裕を持って収まる）
│   上下の余白が均等に分配される（justifyContent: 'center'）
```

## `flex: 1` を使うべき場面・避けるべき場面

| 場面 | `flex: 1` | 自然な高さ |
|------|-----------|-----------|
| 画面全体を埋めるコンテナ | ✅ | - |
| ScrollView の内容 | - | ✅ |
| 固定高さ内の複数セクション | 慎重に | ✅（推奨） |
| カード内のコンテンツ | - | ✅ |
| 兄弟要素と残りスペースを分け合う | ✅ | - |

### 判断基準

**`flex: 1` を使うべきケース:**
- 画面全体を埋める必要がある（ルートコンテナ、SafeAreaView直下など）
- 兄弟要素と明示的にスペースを分割したい（`flex: 1` と `flex: 2` の比率指定など）

**`flex: 1` を避けるべきケース:**
- コンテンツの自然なサイズで表示したい
- 固定高さの親コンテナ内で、コンテンツがはみ出す可能性がある
- `justifyContent: 'center'` で中央配置したいだけの場合

:::message
`flex: 1` で中央配置する代わりに、**親要素に `justifyContent: 'center'` を指定**する方が安全です。子要素のサイズを引き伸ばさずに中央配置を実現できます。
:::

## まとめ

### 学び

1. **`flex: 1` は「残りスペースを全て占有する」指定** — コンテンツの自然なサイズとは無関係にスペースを確保しに行く
2. **固定高さコンテナ内での `flex: 1` は重なりの原因になりやすい** — 子要素の合計サイズがコンテナを超えると要素が重なる
3. **SwiftUI の `VStack` と React Native の `flex: 1` は同じではない** — SwiftUI は子の自然なサイズを尊重するが、`flex: 1` は積極的にスペースを埋める
4. **中央配置は親の `justifyContent: 'center'` で実現する** — 子に `flex: 1` を付けて `justifyContent: 'center'` するのは過剰

### Flexbox レイアウトチェックリスト

- [ ] `flex: 1` を使う前に「本当にスペースを埋める必要があるか」を確認
- [ ] 固定高さコンテナ内の子要素に `flex: 1` を使っていないか
- [ ] 中央配置が目的なら `flex: 1` ではなく親の `justifyContent` で実現しているか
- [ ] `Platform.OS` による余白分岐がある場合、それは対症療法ではないか
- [ ] コンテンツの自然なサイズ合計がコンテナに収まるか計算で確認
