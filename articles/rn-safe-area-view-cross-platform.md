---
title: "React NativeのSafeAreaViewがAndroidで効かない — importの落とし穴"
emoji: "📱"
type: "tech"
topics: ["reactnative", "expo", "android", "ios"]
published: true
---

## はじめに

React Native（Expo）で開発中のアプリで、**iOSでは上部に余計な余白が生まれ、Androidではノッチにコンテンツが被る**という問題に遭遇しました。

調査の結果、原因は `SafeAreaView` の **import元の違い** でした。`react-native` と `react-native-safe-area-context` の2つのパッケージに同名のコンポーネントが存在し、それぞれ挙動がまったく異なります。

:::message
**対象読者**: React Native（Expo）でクロスプラットフォーム開発をしている方。特に「iOSでは動くのにAndroidでレイアウトが崩れる」問題に心当たりのある方。
:::

## 問題の症状

### iOS での症状

ホーム画面の上部に**不自然な余白**が生まれていました。SafeAreaViewによる自動インセットに加えて、Android対策で追加した `paddingTop` が二重に効いていたためです。

```
┌──────────────────┐
│   ステータスバー    │
├──────────────────┤
│                  │ ← SafeAreaView の自動インセット
│                  │ ← paddingTop による余白（二重！）
│   コンテンツ       │
└──────────────────┘
```

### Android での症状

`SafeAreaView` が**まったく機能せず**、ステータスバーやノッチの下にコンテンツが潜り込んでいました。仕方なく `Platform.OS === 'android'` で `paddingTop` を追加するハック的な対処をしていました。

```
┌──────────────────┐
│ ステータスバー(被る)│ ← SafeAreaView が効いていない
│   コンテンツ       │
│                  │
└──────────────────┘
```

## 原因: `react-native` の SafeAreaView は iOS 専用

React Native には **2つの SafeAreaView** が存在します。

| 項目 | `react-native` | `react-native-safe-area-context` |
|------|----------------|----------------------------------|
| import | `import { SafeAreaView } from 'react-native'` | `import { SafeAreaView } from 'react-native-safe-area-context'` |
| iOS対応 | ✅ | ✅ |
| Android対応 | ❌（通常のViewと同じ） | ✅ |
| ノッチ/ダイナミックアイランド | 部分的 | ✅ 完全対応 |
| カスタマイズ性 | edges指定不可 | `edges` propで制御可能 |

**`react-native` の `SafeAreaView` は iOS専用**です。Androidでは通常の `View` と同じ動作をするため、セーフエリアのインセットが一切適用されません。

### なぜこの罠にハマるのか

1. **エディタの自動補完が `react-native` を優先する** — `SafeAreaView` と入力すると、多くのエディタが `react-native` からのimportを最初に提案します
2. **iOSでは正常に動作する** — iOSだけでテストしていると問題に気づけません
3. **同じコンポーネント名** — 名前が同一のため、importを意識しないと見落とします

```tsx
// ❌ エディタが自動補完しがちなimport（iOS専用）
import { SafeAreaView } from 'react-native';

// ✅ クロスプラットフォームで動作するimport
import { SafeAreaView } from 'react-native-safe-area-context';
```

## よくある対症療法とその問題

`react-native` の `SafeAreaView` を使ったまま、Android側だけ手動でパディングを追加するパターンをよく見かけます。

### パターン1: `Platform.OS` で分岐

```tsx
import { SafeAreaView, Platform, StatusBar } from 'react-native';

<SafeAreaView style={{
  paddingTop: Platform.OS === 'android' ? StatusBar.currentHeight : 0,
}}>
```

**問題点:**
- `StatusBar.currentHeight` はステータスバーの高さだけを返す
- ノッチやダイナミックアイランドのインセットは考慮されない
- iOSでは SafeAreaView の自動インセットと `paddingTop: 0` が共存するため一見動くが、意図が不明確

### パターン2: 固定値でハードコード

```tsx
<SafeAreaView style={{
  paddingTop: Platform.OS === 'android' ? 40 : 0,
}}>
```

**問題点:**
- デバイスごとにステータスバーやノッチの高さは異なる
- 将来のデバイスでレイアウトが崩れるリスク

これらの対症療法は**技術的負債**を積み上げるだけです。

## 正しい修正: `react-native-safe-area-context` に統一

### 修正パターン1: 単純なimport切り替え

最もシンプルなケースです。`SafeAreaView` のimport元を変更するだけで修正できます。

```diff tsx
-import { SafeAreaView, View, Text } from 'react-native';
+import { View, Text } from 'react-native';
+import { SafeAreaView } from 'react-native-safe-area-context';

 export default function HomeScreen() {
   return (
     <SafeAreaView style={{ flex: 1 }}>
       <Text>ホーム画面</Text>
     </SafeAreaView>
   );
 }
```

### 修正パターン2: Platform固有パディングの削除

Android対策で追加していた `Platform.OS` 分岐と `StatusBar.currentHeight` が不要になります。

```diff tsx
-import { SafeAreaView, Platform, StatusBar, View, Text } from 'react-native';
+import { View, Text } from 'react-native';
+import { SafeAreaView } from 'react-native-safe-area-context';

 export default function ProfileScreen() {
   return (
-    <SafeAreaView style={{
-      flex: 1,
-      paddingTop: Platform.OS === 'android' ? StatusBar.currentHeight : 0,
-    }}>
+    <SafeAreaView style={{ flex: 1 }}>
       <Text>プロフィール</Text>
     </SafeAreaView>
   );
 }
```

`react-native-safe-area-context` の `SafeAreaView` は両プラットフォームで適切なインセットを自動計算するため、手動のパディング指定が不要になります。

### 修正パターン3: edges の活用

画面によっては、特定の辺だけセーフエリアを適用したい場合があります。`react-native-safe-area-context` の `edges` propを使えば細かく制御できます。

```tsx
import { SafeAreaView } from 'react-native-safe-area-context';

// 上部だけセーフエリアを適用（タブバーがある画面など）
<SafeAreaView edges={['top']} style={{ flex: 1 }}>
  <Text>タブ画面</Text>
</SafeAreaView>

// 左右と上部のみ（下部にカスタムタブバーがある場合）
<SafeAreaView edges={['top', 'left', 'right']} style={{ flex: 1 }}>
  <Text>カスタムタブバー画面</Text>
</SafeAreaView>
```

:::message
モーダル画面では `edges` の指定に注意が必要です。フルスクリーンモーダルでは全辺、ハーフモーダルでは上部のみなど、表示形式に応じて使い分けましょう。
:::

## プロジェクト全体の一括精査方法

1つのファイルを修正して終わりではなく、プロジェクト全体で `react-native` の `SafeAreaView` を使っている箇所を洗い出すことが重要です。

### Step 1: 該当ファイルの洗い出し

```bash
# react-native から SafeAreaView を import しているファイルを検索
grep -rn "from 'react-native'" --include="*.tsx" --include="*.ts" | grep "SafeAreaView"
```

### Step 2: ファイルごとの影響調査

各ファイルについて、以下の観点で確認します。

- [ ] `SafeAreaView` が `react-native` からimportされているか
- [ ] `Platform.OS` による `paddingTop` の分岐があるか
- [ ] `StatusBar.currentHeight` を使用しているか
- [ ] モーダル画面か通常画面か（`edges` 指定の要否）
- [ ] `SafeAreaView` 以外のimportが `react-native` から必要か（`View`, `Text` など）

### Step 3: 修正の実施

:::details 修正チェックリスト（ファイルごとに実施）
1. `SafeAreaView` のimport元を `react-native-safe-area-context` に変更
2. `react-native` のimport文から `SafeAreaView` を削除
3. `Platform.OS` による `paddingTop` 分岐を削除
4. `StatusBar.currentHeight` の参照を削除（他で使用していなければ `StatusBar` のimportも削除）
5. 不要になった `Platform` のimportを削除
6. 必要に応じて `edges` propを追加
7. iOS / Android 両方で動作確認
:::

## まとめ

### 学び

1. **`react-native` の `SafeAreaView` はiOS専用** — Androidでは通常の `View` と同じ動作をする
2. **エディタの自動補完を信用しすぎない** — `SafeAreaView` と入力したとき、`react-native` からのimportが優先されがち
3. **`Platform.OS` 分岐は対症療法** — import元を正しく選べば、プラットフォーム分岐は不要になる
4. **修正は1ファイルで終わらない** — プロジェクト全体を `grep` で精査して一括修正する

### SafeAreaView 移行チェックリスト

プロジェクトに適用する際の確認項目です。

- [ ] `react-native-safe-area-context` がインストール済みか
- [ ] ルートに `SafeAreaProvider` が配置されているか（Expo Routerの場合は自動）
- [ ] 全ファイルで `SafeAreaView` のimport元が `react-native-safe-area-context` になっているか
- [ ] `Platform.OS` による `paddingTop` ハックを削除したか
- [ ] `StatusBar.currentHeight` への依存を削除したか
- [ ] モーダル画面に適切な `edges` が設定されているか
- [ ] iOS / Android 両方の実機で表示確認を行ったか
