---
title: "useAnimatedKeyboard() が Android のキーボード回避を壊す — ツールバーが画面中央に浮く問題の根本原因と修正"
emoji: "⌨️"
type: "tech"
topics: ["reactnative", "reanimated", "android", "expo", "keyboard"]
published: true
---

## はじめに

React Native（Expo）アプリで `react-native-reanimated` の `useAnimatedKeyboard()` を使ってキーボード上にツールバーを配置していたところ、**Android でキーボードを閉じた後にツールバーが画面中央に浮いたまま戻らない**という問題が発生しました。

この記事では、2 回の修正が失敗した経緯と、根本原因の特定、最終的な解決策を解説します。

:::message
**対象読者**: React Native で `useAnimatedKeyboard()` を使ってキーボード連動 UI を実装しており、Android で stale な高さが残る問題に遭遇した方
:::

## システム構成

```
┌─────────────────────────────────────┐
│  React Native (Expo Router)         │
│                                     │
│  ┌───────────────────────────────┐  │
│  │  Note Editor Screen           │  │
│  │                               │  │
│  │  ┌─────────────────────────┐  │  │
│  │  │  Content Area           │  │  │
│  │  └─────────────────────────┘  │  │
│  │  ┌─────────────────────────┐  │  │
│  │  │  Toolbar                │  │  │  ← paddingBottom で位置制御
│  │  └─────────────────────────┘  │  │
│  └───────────────────────────────┘  │
│                                     │
│  ┌───────────────────────────────┐  │
│  │  Keyboard (native)            │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

ツールバーは `ReanimatedAnimated.View` に `keyboardStyle`（`paddingBottom`）を適用し、キーボードの高さに追従させる設計です。

## 問題: キーボードを閉じてもツールバーが中央に浮く

### 症状

Android で以下の操作を行うとツールバーが画面中央付近に浮いたまま戻らなくなります。

1. ノートエディタを開く
2. テキスト入力でキーボードを表示
3. キーボードを閉じる
4. **別の画面に遷移して戻る**
5. ツールバーが画面中央に浮いている

特にナビゲーションスタック内で画面が unmount されずに残る場合に再現しやすい問題でした。

### 失敗した修正 v1: KeyboardState チェック

最初の修正では、`useAnimatedKeyboard()` の `state` が `CLOSED` / `UNKNOWN` のとき `paddingBottom` を 0 にするゲートを追加しました。

```typescript
// v1: KeyboardState によるゲート
const keyboard = useAnimatedKeyboard();

const keyboardStyle = useAnimatedStyle(() => {
  if (
    keyboard.state.value === KeyboardState.CLOSED ||
    keyboard.state.value === KeyboardState.UNKNOWN
  ) {
    return { paddingBottom: 0 };
  }
  return {
    paddingBottom: Math.max(0, keyboard.height.value - insets.bottom),
  };
});
```

**結果**: `state` が `OPEN` のまま更新されないケースがあり、stale な高さが残り続けました。

### 失敗した修正 v2: useAnimatedReaction + useFocusEffect

次に、`useAnimatedReaction` で状態遷移を追跡し、`useFocusEffect` でフォーカス復帰時にリセットする方式を試みました。

```typescript
// v2: ゲート変数 + useAnimatedReaction
const keyboard = useAnimatedKeyboard();
const keyboardShowing = useSharedValue(false);

useAnimatedReaction(
  () => keyboard.state.value,
  (state) => {
    if (state === KeyboardState.OPENING || state === KeyboardState.OPEN) {
      keyboardShowing.value = true;
    } else if (state === KeyboardState.CLOSED || state === KeyboardState.UNKNOWN) {
      keyboardShowing.value = false;
    }
  }
);

useFocusEffect(
  useCallback(() => {
    keyboardShowing.value = false;
    return () => { keyboardShowing.value = false; };
  }, [keyboardShowing])
);

const keyboardStyle = useAnimatedStyle(() => {
  if (!keyboardShowing.value || keyboard.height.value <= 0) {
    return { paddingBottom: 0 };
  }
  return {
    paddingBottom: Math.max(0, keyboard.height.value - insets.bottom),
  };
});
```

**結果**: JS 側で `keyboardShowing` を `false` にリセットしても、ネイティブ側の WindowInsets コールバックが stale な値で上書きしてしまい、レース条件で修正が無効化されました。

## 根本原因: `setDecorFitsSystemWindows(false)`

`useAnimatedKeyboard()` の Android 実装を調査したところ、**フックの初期化時に `setDecorFitsSystemWindows(false)` を呼び出している**ことが判明しました。

:::message
**ポイント**: `setDecorFitsSystemWindows(false)` は Android の **`adjustResize`** を無効化します。これにより、キーボード表示時のレイアウト調整がネイティブで行われなくなり、代わりに `WindowInsetsCompat` のコールバックで JS 側に高さが伝達されます。
:::

問題の因果関係は以下の通りです:

```
useAnimatedKeyboard() 初期化
  ↓
setDecorFitsSystemWindows(false) を呼び出し
  ↓
ネイティブの adjustResize が無効化される
  ↓
WindowInsets コールバック経由でキーボード高さを JS に伝達
  ↓
画面遷移で WindowInsets コールバックが stale な値を報告
  ↓
JS 側でどんなリセットを試みてもネイティブ側が上書き
  ↓
paddingBottom が残り続け、ツールバーが浮く
```

| 要因 | 詳細 |
|------|------|
| `setDecorFitsSystemWindows(false)` | `adjustResize` を無効化 |
| ナビゲーションスタック | 画面が unmount されず残る |
| WindowInsets コールバック | stale な値を報告し続ける |
| JS 側リセット | ネイティブからの上書きでレース条件発生 |

## 修正: `useAnimatedKeyboard()` を完全に削除する

根本原因がフック自体のネイティブ副作用にあるため、**`useAnimatedKeyboard()` を完全に削除**し、プラットフォーム別の方式に置き換えました。

### 修正方針

| | Android | iOS |
|---|---|---|
| キーボード回避 | ネイティブ `adjustResize` | `paddingBottom` + `withTiming` |
| イベント | `keyboardDidShow/DidHide` | `keyboardWillShow/WillHide` |
| `paddingBottom` | 常に 0 | アニメーション駆動 |

**Android** では `useAnimatedKeyboard()` を削除するだけで `adjustResize`（AndroidManifest で設定済み）がネイティブにキーボード回避を処理します。`paddingBottom` は不要です。

:::message
Expo 管理プロジェクトの場合、`app.config.ts` で `android.softwareKeyboardLayoutMode: "resize"` を明示しておくと、`adjustResize` 前提が設定として見える化できます。将来の設定変更による意図しない挙動変化も防げます。
:::

**iOS** では `keyboardWillShow/WillHide` イベントと `withTiming` を組み合わせてスムーズなアニメーションを実現します。

### Step 1: インポートの変更

```diff
 import ReanimatedAnimated, {
-  KeyboardState,
-  useAnimatedKeyboard,
-  useAnimatedReaction,
   useAnimatedStyle,
   useSharedValue,
+  withTiming,
 } from 'react-native-reanimated';
```

`useAnimatedKeyboard` とその関連型を削除し、`withTiming` を追加します。

### Step 2: キーボードトラッキングの置換

```diff
- const keyboard = useAnimatedKeyboard();
- const keyboardShowing = useSharedValue(false);
-
- useAnimatedReaction(
-   () => keyboard.state.value,
-   (state) => {
-     if (state === KeyboardState.OPENING || state === KeyboardState.OPEN) {
-       keyboardShowing.value = true;
-     } else if (state === KeyboardState.CLOSED || state === KeyboardState.UNKNOWN) {
-       keyboardShowing.value = false;
-     }
-   }
- );
-
- const keyboardStyle = useAnimatedStyle(() => {
-   if (!keyboardShowing.value || keyboard.height.value <= 0) {
-     return { paddingBottom: 0 };
-   }
-   return {
-     paddingBottom: Math.max(0, keyboard.height.value - insets.bottom),
-   };
- });
+ // iOS only: animated paddingBottom
+ // Android: adjustResize handles this natively, paddingBottom stays 0
+ const keyboardAnimatedHeight = useSharedValue(0);
+
+ const keyboardStyle = useAnimatedStyle(() => ({
+   paddingBottom: keyboardAnimatedHeight.value,
+ }));
```

Android では `keyboardAnimatedHeight` は常に 0 のため、`paddingBottom` は適用されません。

### Step 3: Keyboard.addListener をプラットフォーム別に分岐

```typescript
useEffect(() => {
  const showEvent =
    Platform.OS === 'ios' ? 'keyboardWillShow' : 'keyboardDidShow';
  const hideEvent =
    Platform.OS === 'ios' ? 'keyboardWillHide' : 'keyboardDidHide';

  const showListener = Keyboard.addListener(showEvent, (event) => {
    setKeyboardVisible(true);
    setKeyboardHeight(event.endCoordinates.height);

    // iOS only: animate paddingBottom
    if (Platform.OS === 'ios') {
      keyboardAnimatedHeight.value = withTiming(
        Math.max(0, event.endCoordinates.height - insets.bottom),
        { duration: event.duration || 250 }
      );
    }
  });

  const hideListener = Keyboard.addListener(hideEvent, () => {
    setKeyboardVisible(false);
    setKeyboardHeight(0);

    if (Platform.OS === 'ios') {
      keyboardAnimatedHeight.value = withTiming(0, { duration: 250 });
    }
  });

  return () => {
    showListener.remove();
    hideListener.remove();
  };
}, [insets.bottom, keyboardAnimatedHeight]);
```

:::message
iOS では `keyboardWillShow/WillHide` を使い、イベントに含まれる `duration` で `withTiming` のアニメーション速度をネイティブのキーボードアニメーションに合わせています。Android では `keyboardDidShow/DidHide` を使いますが、`paddingBottom` の更新は行いません。
:::

### Step 4: useFocusEffect でセーフティネット

```typescript
useFocusEffect(
  useCallback(() => {
    keyboardAnimatedHeight.value = 0;
    return () => {
      keyboardAnimatedHeight.value = 0;
    };
  }, [keyboardAnimatedHeight])
);
```

ナビゲーション遷移中にキーボードが開いていた場合のエッジケースに対応します。Android では `keyboardAnimatedHeight` が常に 0 のため、この処理は no-op です。

## まとめ

| 問題 | 原因 | 修正 |
|------|------|------|
| キーボードを閉じてもツールバーが浮く | `useAnimatedKeyboard()` が `setDecorFitsSystemWindows(false)` で `adjustResize` を無効化 | `useAnimatedKeyboard()` を削除し、JS ドリブンの `Keyboard.addListener` に置換 |
| JS 側のリセットが効かない | ネイティブ WindowInsets が stale な値で上書き | Android はネイティブ `adjustResize` に委任（`paddingBottom` 不要） |
| iOS のアニメーションが途切れる | — | `keyboardWillShow/WillHide` + `withTiming` でスムーズに追従 |

### 学び

1. **`useAnimatedKeyboard()` はネイティブの副作用を伴う** — 単に値を読むだけでなく、`setDecorFitsSystemWindows(false)` でシステムのレイアウト挙動を変更する。ドキュメントでは目立たないが、Android のキーボード回避全体に影響する
2. **JS 側のゲートではネイティブの上書きに勝てない** — `useAnimatedReaction` や `useFocusEffect` でリセットしても、ネイティブ側からの値更新がレース条件で上書きする
3. **プラットフォーム別の最適解を選ぶ** — Android は `adjustResize` がネイティブで優秀、iOS は `keyboardWillShow/WillHide` がアニメーション duration を提供。無理に統一せず、それぞれの強みを活かすのが正解

### デバッグチェックリスト

- [ ] `useAnimatedKeyboard()` を使っていないか確認（Android で `adjustResize` が無効化されていないか）
- [ ] AndroidManifest の `android:windowSoftInputMode="adjustResize"` が設定されているか
- [ ] ナビゲーション遷移後にキーボード高さが stale になっていないか
- [ ] iOS では `keyboardWillShow` / Android では `keyboardDidShow` を使い分けているか
- [ ] `useFocusEffect` でフォーカス復帰時にアニメーション値をリセットしているか
