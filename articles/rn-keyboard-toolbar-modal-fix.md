---
title: "React Nativeのモーダル画面でKeyboardAvoidingViewが予測変換バーを無視する問題の解決法"
emoji: "⌨️"
type: "tech"
topics: ["reactnative", "expo", "ios", "reanimated"]
published: true
---

## はじめに

React Native（Expo）で開発中のアプリで、**モーダル画面のテキストエディタでツールバーが予測変換バーの下に隠れる**問題に遭遇しました。

`KeyboardAvoidingView`（KAV）はキーボード回避の定番コンポーネントですが、iOSのモーダル画面（`presentation: 'modal'`）では**予測変換バー分の高さを正しく計算できない**ケースがあります。

本記事では、この問題の根本原因と、`react-native-reanimated` の `useAnimatedKeyboard` を使った解決法を紹介します。

:::message
**対象読者**: React Native（Expo）でモーダル画面内にテキスト入力＋ツールバーを配置している方。特に「通常画面では動くのにモーダルだとツールバーが隠れる」問題に心当たりのある方。
:::

## 問題の症状

テキストエディタ画面をモーダル（`presentation: 'modal'`）で表示し、キーボードの上にツールバーを配置していました。

**期待する表示:**
```
┌──────────────────┐
│   テキスト入力     │
│                  │
├──────────────────┤
│   ツールバー       │ ← キーボードの直上
├──────────────────┤
│   予測変換バー     │ ← 約44px
├──────────────────┤
│   キーボード       │
└──────────────────┘
```

**実際の表示:**
```
┌──────────────────┐
│   テキスト入力     │
│                  │
├──────────────────┤
│   予測変換バー     │ ← ツールバーが予測変換の下に隠れる！
│  ┄┄ツールバー┄┄   │ ← 見えない
├──────────────────┤
│   キーボード       │
└──────────────────┘
```

予測変換バーが**約44px**分ツールバーを覆い隠し、操作不能になっていました。

## 試行錯誤した手法

解決に至るまで、複数のアプローチを試しました。

| 手法 | 概要 | 結果 |
|------|------|------|
| KAV `behavior="padding"` | 標準的なキーボード回避 | ❌ 予測変換バーに隠れる |
| `InputAccessoryView` | RN標準のキーボード付属ビュー | ❌ モーダル画面で全消え（[React Native Issue #21363](https://github.com/facebook/react-native/issues/21363)） |
| `keyboardVerticalOffset` 調整 | KAVのオフセット値を変更 | ❌ 端末・OSバージョンで高さが異なり、固定値では対応不可 |

## 原因: KAVがモーダルのコンテキストでキーボード座標を誤計算する

`KeyboardAvoidingView` の内部実装を追ったところ、根本原因が判明しました。

### KAVの高さ計算ロジック

KAVは以下のように動作します：

1. `keyboardWillChangeFrame` イベントからキーボードの `endCoordinates.screenY` を取得
2. 自身の View フレームの `y + height` を取得
3. 両者の差分をパディング（またはheight）として適用

```
キーボード回避量 = viewFrame.y + viewFrame.height - keyboard.screenY
```

### モーダルでの座標系のずれ

通常画面では、View フレームのY座標はスクリーン座標系と一致するため正確に計算できます。

しかし**モーダル画面**では状況が異なります：

- iOSのモーダルは独自の `UIViewController` として表示される
- View フレームの座標が**モーダルのローカル座標系**で返される場合がある
- キーボードのフレームは常に**スクリーン座標系**
- この座標系のずれにより、予測変換バー分（約44px）のパディングが不足する

```
通常画面:
  viewFrame.y → スクリーン座標 → ✅ 正確

モーダル画面:
  viewFrame.y → ローカル座標 → ❌ ずれる
  keyboard.screenY → スクリーン座標 → ✅ 正確
  → 差分計算が不正確に
```

:::message
予測変換バーの高さは端末やOSバージョンによって異なります（通常36〜55px程度）。`keyboardVerticalOffset` に固定値を入れてもすべての環境で正確に動作させることはできません。
:::

## 解決: `useAnimatedKeyboard` でKAVを置き換える

`react-native-reanimated` v3 の `useAnimatedKeyboard()` を使うことで、この問題を根本的に解決できます。

### なぜ useAnimatedKeyboard で解決するのか

| 比較項目 | `KeyboardAvoidingView` | `useAnimatedKeyboard` |
|----------|----------------------|----------------------|
| 座標取得方法 | View frame vs キーボード frame の差分 | `NSNotificationCenter` でキーボード frame を直接取得 |
| View hierarchy 依存 | ✅ 依存（座標系ずれの原因） | ❌ 非依存 |
| モーダル対応 | ❌ 座標系ずれが発生 | ✅ スクリーン座標から直接計算 |
| 予測変換バー | ❌ ずれる場合がある | ✅ `screenHeight - keyboardFrame.origin.y` で正確 |

`useAnimatedKeyboard` は View hierarchy に依存せず、`NSNotificationCenter` の `keyboardWillChangeFrame` 通知から**スクリーン座標系のキーボードフレーム**を直接取得します。そのため、モーダルの座標系ずれの影響を受けません。

### 実装

#### 修正後のレイアウト構造

```
SafeAreaView (edges=['bottom'])
  View (header)
  [iOS]  ReanimatedAnimated.View (paddingBottom = keyboard.height - insets.bottom)
  [Android] KeyboardAvoidingView (behavior="height")
    ScrollView (flex:1)
      TextInput (title)
      TextInput (content)
    View (toolbarsContainer)
      EditorToolbar
      KeyboardToolbar (キーボード非表示時は非表示)
```

#### コード変更

**1. import の追加**

```diff tsx
 import {
   View,
   Text,
   TextInput,
   KeyboardAvoidingView,
   Platform,
   ScrollView,
-  Animated,
-  PanResponder,
 } from 'react-native';
+import ReanimatedAnimated, {
+  useAnimatedKeyboard,
+  useAnimatedStyle,
+} from 'react-native-reanimated';
```

:::details react-native-reanimated のバージョンについて
`useAnimatedKeyboard` は reanimated v3 以降で利用可能です。Expo SDK 50+ であれば reanimated v3 がデフォルトで含まれているため、追加インストールは不要です。

```bash
# バージョン確認
npx expo install --check react-native-reanimated
```
:::

**2. フックの追加**

```tsx
const insets = useSafeAreaInsets();

const keyboard = useAnimatedKeyboard();
const iosKeyboardStyle = useAnimatedStyle(() => {
  if (Platform.OS !== 'ios') return {};
  return {
    paddingBottom: Math.max(0, keyboard.height.value - insets.bottom),
  };
});
```

**`insets.bottom` を引く理由**: `SafeAreaView` が既に safe area bottom（約34px）を適用済みのため、`keyboard.height`（スクリーン底辺からの高さ）からこの値を引かないと34px分余計に押し上げられます。

```
keyboard.height = 291px（スクリーン底辺からキーボード上端まで）
insets.bottom = 34px（SafeAreaViewが適用済み）
paddingBottom = 291 - 34 = 257px ← 正確な値
```

**3. Platform条件分岐**

```tsx
{Platform.OS === 'ios' ? (
  <ReanimatedAnimated.View style={[styles.container, iosKeyboardStyle]}>
    {renderEditorContent()}
  </ReanimatedAnimated.View>
) : (
  <KeyboardAvoidingView
    behavior="height"
    style={styles.container}
    keyboardVerticalOffset={0}
  >
    {renderEditorContent()}
  </KeyboardAvoidingView>
)}
```

**4. renderEditorContent ヘルパー**

ScrollView + ツールバーを関数に抽出して、iOS/Android の分岐で重複を回避します。

```tsx
const renderEditorContent = () => (
  <>
    <ScrollView
      style={styles.scrollView}
      contentContainerStyle={styles.scrollViewContent}
      keyboardShouldPersistTaps="handled"
    >
      {/* テキスト入力フィールド */}
    </ScrollView>
    <View style={styles.toolbarsContainer}>
      <EditorToolbar />
      <KeyboardToolbar isKeyboardVisible={keyboardVisible} />
    </View>
  </>
);
```

### なぜ Android は KAV のままなのか

Androidでは `KeyboardAvoidingView` がモーダル画面でも正常に動作します。これは Android のキーボード表示メカニズムが iOS と異なり、`windowSoftInputMode` によるレイアウト調整が View hierarchy の座標系に依存しないためです。

変更を最小限にとどめ、Androidの既存動作を壊さないように、iOS のみ `useAnimatedKeyboard` に切り替えています。

## SwiftUI版との比較

参考として、SwiftUI版のアプリではこの問題はそもそも発生しません。

```swift
// SwiftUI — UIKit の inputAccessoryView を直接使用
textView.inputAccessoryView = toolbarView
```

SwiftUI（UIKit）の `inputAccessoryView` はキーボードに**物理的に付属する**ビューで、キーボードフレームの一部として扱われます。そのため、座標計算の必要がなく、モーダルでも予測変換バーの有無に関係なく常に正しい位置に表示されます。

React Native にも `InputAccessoryView` コンポーネントがありますが、[モーダル画面では動作しない既知の問題](https://github.com/facebook/react-native/issues/21363)があるため、今回は `useAnimatedKeyboard` による代替手法を採用しました。

## まとめ

### 学び

1. **`KeyboardAvoidingView` はモーダル画面で座標系がずれる** — View frame とキーボード frame の座標系が一致しないことが原因
2. **`InputAccessoryView` はモーダルで使えない** — React Native の既知の問題（[#21363](https://github.com/facebook/react-native/issues/21363)）
3. **`useAnimatedKeyboard` は View hierarchy に非依存** — `NSNotificationCenter` から直接キーボード座標を取得するため、モーダルでも正確
4. **予測変換バーの高さは端末依存** — 固定値のオフセットでは対応不可。動的に追従する仕組みが必要

### モーダル画面でのキーボード対応チェックリスト

- [ ] `KeyboardAvoidingView` がモーダル内で正しく動作しているか確認
- [ ] 予測変換バーの ON/OFF 両方でツールバーが見えるか確認
- [ ] 問題がある場合は `useAnimatedKeyboard`（reanimated v3）の使用を検討
- [ ] `SafeAreaView` のインセットと `keyboard.height` の二重計算を避けているか
- [ ] Android では KAV が正常動作するか別途確認（Platform分岐を推奨）
- [ ] iOS / Android 両方の実機で表示確認を行ったか
