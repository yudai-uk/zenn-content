---
title: "React Native (Expo) iOS実機で画面が描画されない問題の原因と対策"
emoji: "🚀"
type: "tech"
topics: ["reactnative", "expo", "ios", "performance"]
published: true
---

## はじめに

React Native (Expo) で開発中のアプリで、**iOS実機でのみ画面遷移後に描画が大幅に遅延する**問題に遭遇しました。シミュレーターでは再現せず、実機でだけ発生するという厄介なバグです。

本記事では、原因の特定から修正までのプロセスを共有します。同じSwiftUI版のアプリが存在していたため、**SwiftUIとの比較**を通じて「なぜReact Nativeでだけ起きるのか」も深掘りします。

## 問題の症状

オンボーディングフローの「トピック提案画面」で、以下の症状が発生しました。

- 画面遷移後、**数秒間まったく描画されない**（真っ暗）
- その後、画面の**半分だけが突然描画**される
- 最終的に全体が描画されるが、アニメーションのタイミングがずれる

**再現環境:**
- iOS実機（iPhone）のみ — **シミュレーターでは再現しない**
- 画面内にLinearGradientを多用したカードコンポーネント（3枚）
- カードにはreact-native-reanimatedによるアニメーション付き

## 原因調査

調査の結果、**3つの根本原因**が複合的に描画遅延を引き起こしていることが分かりました。

### 原因1: `useEffect` と SwiftUIの `.onAppear` のタイミング差

これが**最も根本的な原因**でした。

React Nativeの `useEffect` は、Reactのcommitフェーズの後に実行されますが、**ネイティブビューが実際に画面に描画されたことを保証しません**。つまり、描画が完了する前にアニメーションが開始され、初回描画とアニメーション計算がメインスレッドを奪い合います。

一方、SwiftUIの `.onAppear` は**ビューが画面に描画された後**に発火します。

```swift
// SwiftUI — 描画完了後にアニメーション開始
.onAppear {
    startCardAnimation()
}
```

```typescript
// React Native — 描画完了を待たずにアニメーション開始
useEffect(() => {
  const timer1 = setTimeout(() => setCurrentCardIndex(1), 1500);
  const timer2 = setTimeout(() => setCurrentCardIndex(2), 3000);
  return () => { clearTimeout(timer1); clearTimeout(timer2); };
}, []);
```

### 原因2: LinearGradientのGPUシェーダーコンパイル

各カードには3つの `LinearGradient` が含まれていました。

1. 背景グラデーション（purple → teal）
2. ボーダーグラデーション（ストローク効果）
3. 「Personalized」タグのグラデーション

3枚のカード × 3グラデーション + タップインジケーター = **計10個の`CAGradientLayer`** がiOSネイティブ側で生成されます。

iOS実機では、各 `CAGradientLayer` の初回レンダリングに**GPUシェーダーのコンパイル**が必要で、これがレンダリングパイプラインをブロックします。シミュレーターはソフトウェアレンダリングを使用するため、この問題が顕在化しません。

### 原因3: `React.memo` の欠如による不要な再レンダー

カードコンポーネントに `React.memo` が適用されていなかったため、親の `currentCardIndex` が変わるたびに**全3枚のカードが再レンダー**されていました。

`react-native-reanimated` のアニメーション自体はUIスレッドで動作しますが、Reactの差分検出（reconciliation）はJSスレッドで全子コンポーネントに対して実行されるため、不要な負荷がかかっていました。

## 修正内容

最終的に採用した修正は以下の2つです。

### 修正A: `InteractionManager.runAfterInteractions` で描画完了を待つ

React Nativeの `InteractionManager.runAfterInteractions` は、進行中のアニメーションやトランジションが完了した後にコールバックを実行します。これはSwiftUIの `.onAppear` に相当する動作です。

加えて、`isReady` ステートを導入し、描画完了まで各カードのアニメーションを抑止します。

```diff:components/onboarding/TopicSuggestionView.tsx
 import {
   View,
   Text,
   StyleSheet,
   TouchableOpacity,
+  InteractionManager,
 } from 'react-native';
```

**親コンポーネント（TopicSuggestionView）:**

```diff:components/onboarding/TopicSuggestionView.tsx
 const [currentCardIndex, setCurrentCardIndex] = useState(0);
+const [isReady, setIsReady] = useState(false);

 useEffect(() => {
-  const timer1 = setTimeout(() => setCurrentCardIndex(1), 1500);
-  const timer2 = setTimeout(() => setCurrentCardIndex(2), 3000);
-
-  return () => {
-    clearTimeout(timer1);
-    clearTimeout(timer2);
-  };
+  // 初回描画の完了を待ってからアニメーションを開始
+  const handle = InteractionManager.runAfterInteractions(() => {
+    setIsReady(true);
+
+    const timer1 = setTimeout(() => setCurrentCardIndex(1), 1500);
+    const timer2 = setTimeout(() => setCurrentCardIndex(2), 3000);
+
+    handle.cancel = () => {
+      clearTimeout(timer1);
+      clearTimeout(timer2);
+    };
+  });
+
+  return () => {
+    handle.cancel();
+  };
 }, []);
```

**子コンポーネント（OnboardingCard）:**

```diff:components/onboarding/TopicSuggestionView.tsx
-const flipRotation = useSharedValue(-10);
+const flipRotation = useSharedValue(0);
 const flipScale = useSharedValue(0.95);
 const offsetX = useSharedValue(0);
-const offsetY = useSharedValue((index - currentCardIndex) * 8);
+const offsetY = useSharedValue(index * 8);

 useEffect(() => {
+  if (!isReady) return; // 描画完了まで何もしない
+
   if (isActive) {
     flipRotation.value = withTiming(0, { duration: 600 });
     // ...
   }
-}, [currentCardIndex]);
+}, [currentCardIndex, isReady]);
```

ポイントは3つです。

1. **`InteractionManager.runAfterInteractions`** で画面遷移アニメーション完了後にアニメーションを開始
2. **`isReady` ガード**により、各カードのアニメーション `useEffect` を描画完了まで抑止
3. **SharedValueの初期値を安定化** — `flipRotation` を `-10` → `0` に変更し、初期描画時のレイアウトシフトを防止

### 修正B: `React.memo` によるカードの再レンダー防止

```diff:components/onboarding/TopicSuggestionView.tsx
-const OnboardingCard: React.FC<{
+const OnboardingCardInner: React.FC<{
   card: CardData;
   index: number;
   currentCardIndex: number;
   isFinal: boolean;
   showTapIndicator: boolean;
+  isReady: boolean;
-}> = ({ card, index, currentCardIndex, isFinal, showTapIndicator }) => {
+}> = ({ card, index, currentCardIndex, isFinal, showTapIndicator, isReady }) => {
   // ...
 };
+
+const OnboardingCard = React.memo(OnboardingCardInner);
```

`React.memo` でラップすることで、propsが変化していないカードの再レンダーをスキップします。`currentCardIndex` は全カードに渡されるため完全に再レンダーが止まるわけではありませんが、Reactの差分検出コストを軽減し、初期描画の負荷を下げます。

## SwiftUI版との比較

同じ画面をSwiftUIでも実装しており、そちらでは描画遅延は発生しませんでした。両者の違いを比較します。

| 観点 | SwiftUI | React Native（修正前） |
|------|---------|----------------------|
| アニメーション開始タイミング | `.onAppear`（描画完了後に発火） | `useEffect`（描画完了を保証しない） |
| アニメーションシステム | `.animation(.spring(), value:)` 1つで制御 | SharedValue × 5 × 3枚 = 15個を個別に制御 |
| 再レンダー | SwiftUIの差分更新（変化したViewのみ） | 全カードが再レンダー（memo なし） |
| グラデーション | `CAGradientLayer`（OS最適化済み） | `expo-linear-gradient`経由で同じ`CAGradientLayer` |
| 画面遷移との連携 | OSレベルで遷移完了を管理 | JSブリッジ経由で非同期 |

SwiftUIでは `.onAppear` がビュー階層への追加後に発火するため、アニメーション開始と初回描画が競合しません。React Nativeでは `useEffect` がこの保証をしないため、`InteractionManager` で明示的に制御する必要がありました。

## まとめ

### 学び

1. **`useEffect` ≠ `.onAppear`** — React Nativeの `useEffect` は描画完了を保証しない。iOS実機でアニメーションを伴う画面では `InteractionManager.runAfterInteractions` を使って描画完了を待つべき
2. **`React.memo` はアニメーションコンポーネントで特に重要** — `react-native-reanimated` のアニメーション自体はUIスレッドで動くが、Reactのreconciliationは全子コンポーネントに波及する
3. **シミュレーターと実機の差** — `CAGradientLayer` のGPUシェーダーコンパイルはシミュレーターのソフトウェアレンダリングでは再現しない。iOS描画パフォーマンスは必ず実機で検証すべき

### 応用パターン: `isReady` ガード

今回使用した `isReady` パターンは、重いコンポーネントの初期表示を最適化する汎用的なテクニックです。

```typescript
const [isReady, setIsReady] = useState(false);

useEffect(() => {
  const handle = InteractionManager.runAfterInteractions(() => {
    setIsReady(true);
  });
  return () => handle.cancel();
}, []);

// isReady が true になるまで重い処理をスキップ
useEffect(() => {
  if (!isReady) return;
  // アニメーション開始、データフェッチなど
}, [isReady]);
```

画面遷移直後にアニメーションやデータ取得を行う場面で、この `InteractionManager` + `isReady` の組み合わせは広く活用できます。
