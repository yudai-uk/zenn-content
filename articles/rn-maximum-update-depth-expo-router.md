---
title: "Maximum update depth exceeded — Expo RouterとTab Navigatorの混在が原因だった"
emoji: "🔄"
type: "tech"
topics: ["reactnative", "expo", "exporouter", "debug"]
published: true
---

## はじめに

React Native (Expo) で開発中のアプリを**実機にデプロイしたら `Maximum update depth exceeded` で操作不能**になりました。ErrorOverlay が全面に表示され、アプリが一切使えない状態です。

調査の結果、**Expo Router のフックを React Navigation の Tab.Navigator 内で使っていた**ことが根本原因でした。加えて、Firestore リスナーのリークと `onLayout` の再レンダーループが重なり、問題が顕在化していました。

:::message
**対象読者**: Expo Router と `createBottomTabNavigator()` を併用しているプロジェクトで、無限レンダーやパフォーマンス問題に悩んでいる方
:::

## 問題の症状

実機でアプリを起動すると、即座に以下のエラーが表示されます。

:::details エラーログ全文
```
ERROR  Warning: Error: Maximum update depth exceeded. This can happen when a component
repeatedly calls setState inside componentWillUpdate or componentDidUpdate. React limits
the number of nested updates to prevent infinite loops.

This error is located at:

  67 | export default function HomeScreen() {
  68 |   const router = useRouter();
> 69 |   const params = useLocalSearchParams();
                                        ^
  70 |
  71 |   // First-time tutorial check
  72 |   useEffect(() => {

Call Stack
  HomeScreen (components/HomeScreen.tsx:69:38)
  TabLayout (app/(tabs)/_layout.tsx:16:74)
  AppContainer (app/index.tsx:20:57)
  ScreenContentWrapper (<anonymous>)
  RNSScreenStack (<anonymous>)
  RootLayout (app/_layout.tsx:53:37)
```
:::

ポイントは **`useLocalSearchParams()` の行でエラーが発生**していることです。

## 原因調査

調査の結果、**3つの原因**が複合的に無限レンダーを引き起こしていました。

### 原因1: Expo Router のフックを Tab.Navigator 内で使用（根本原因）

これが最も重要な原因です。アプリのナビゲーション構造を見てみます。

```
app/_layout.tsx (Expo Router の Stack)
  └─ app/index.tsx (AppContainer を直接レンダー)
       └─ TabLayout (createBottomTabNavigator)
            ├─ HomeScreen ← ここで useLocalSearchParams() を呼んでいた
            └─ MyVocabularyScreen ← ここで usePathname() を呼んでいた
```

`HomeScreen` と `MyVocabularyScreen` は **React Navigation の `createBottomTabNavigator()` の子コンポーネント**として描画されています。つまり Expo Router のナビゲーションコンテキスト外です。

にもかかわらず、これらのコンポーネントで Expo Router 専用のフックを使っていました。

```tsx:components/HomeScreen.tsx
import { useRouter, useLocalSearchParams, Stack } from 'expo-router';

export default function HomeScreen() {
  const router = useRouter();
  const params = useLocalSearchParams(); // ← 問題の行
  // ...
  return (
    <>
      <Stack.Screen options={{ headerShown: false }} /> {/* ← これも問題 */}
      {/* ... */}
    </>
  );
}
```

**なぜ無限レンダーになるのか?**

`useLocalSearchParams()` は Expo Router のナビゲーションコンテキストからパラメータを取得します。Tab.Navigator 内ではこのコンテキストが存在しないため、**毎レンダーで新しい空オブジェクトを返します**。新しいオブジェクト参照 → `useEffect` の依存配列が変化を検知 → 再レンダー → また新しいオブジェクト...という無限ループです。

`usePathname()` も同様の問題を抱えていました。

```tsx:app/my-vocabulary.tsx
import { useRouter, usePathname } from 'expo-router';
import { Stack } from 'expo-router';

export default function MyVocabularyScreen() {
  const pathname = usePathname(); // ← 同じ問題
  // ...
  return (
    <View>
      <Stack.Screen options={{ headerShown: false }} /> {/* ← Tab内では不適切 */}
      {/* ... */}
    </View>
  );
}
```

### 原因2: Firestore リスナーのリーク

`HomeScreen` の `loadData()` 関数は Firestore のリアルタイムリスナーを設定し、unsubscribe 関数を返していました。しかし、**呼び出し側で戻り値を使っていませんでした**。

```tsx
// loadData() はリスナーのクリーンアップ関数を返す
const loadData = async () => {
  const unsubscribeTags = TagsService.listenTags((tags) => setTags(tags));
  const unsubscribeNotes = NotesService.listenNotes((notes) => {
    setNotes(notes);
    setIsRefreshing(false);
  });

  return () => {         // ← この関数が返される
    unsubscribeTags();
    unsubscribeNotes();
  };
};

// しかし呼び出し側で捨てられている
auth().onAuthStateChanged((user) => {
  if (user) {
    loadData();  // ← 戻り値（unsubscribe関数）を捨てている
  }
});
```

`loadData()` が呼ばれるたびに**新しいリスナーが作成され、古いリスナーは破棄されない**。結果として複数のリスナーが同時に `setNotes()` / `setTags()` を呼び、再レンダーを加速させていました。

### 原因3: onLayout コールバックの再レンダーループ

タブインジケーターの位置計算用に `onLayout` でレイアウト情報を取得していましたが、**毎回新しいオブジェクトを作成**していました。

```tsx
onLayout={(e) => {
  const { x, width } = e.nativeEvent.layout;
  setTodayTabLayout({ x, width }); // ← 毎回新しいオブジェクト参照
}}
```

`x` と `width` が変わっていなくても `{ x, width }` は新しいオブジェクトなので、React は状態変化と判断します。これが再レンダーを引き起こし、再レンダーが `onLayout` を再発火させ...というループです。

## なぜこのアーキテクチャになったか

Expo Router はファイルベースルーティングを提供しますが、Tab ナビゲーションを `createBottomTabNavigator()` で独自に実装していました。開発初期にこの構成を採用し、その後 Expo Router のフックを「便利だから」と Tab 内のコンポーネントでも使い始めたことが原因です。

**Expo Router のフックは、Expo Router が管理するスクリーン内でのみ有効**です。React Navigation の Tab.Navigator 配下では正しく動作しません。

## 修正内容

### 修正1: Expo Router フックの除去（根本修正）

`useLocalSearchParams()` でパラメータを渡していた箇所を **AsyncStorage 経由のフラグ**に変更しました。

```diff tsx:components/HomeScreen.tsx
-import { useRouter, useLocalSearchParams, Stack } from 'expo-router';
+import { useRouter } from 'expo-router';
```

```diff tsx:components/HomeScreen.tsx
 export default function HomeScreen() {
   const router = useRouter();
-  const params = useLocalSearchParams();
```

パラメータ渡しのコード（note-editor → HomeScreen への画面遷移時にフラグを渡す処理）:

```diff tsx:app/note-editor.tsx
-  // Navigate back with parameter to trigger notification modal
-  router.push({
-    pathname: "/",
-    params: { showNotificationAfterFeedback: 'true' }
-  });
+  // Set flag to trigger notification modal on HomeScreen
+  await AsyncStorage.setItem('showNotificationAfterFeedback', 'true');
+  router.push("/");
```

受け取り側も AsyncStorage から読み取るように変更:

```diff tsx:components/HomeScreen.tsx
-  // Check for notification modal trigger from params
+  // Check for notification modal trigger via AsyncStorage flag
   useEffect(() => {
     const checkNotificationTrigger = async () => {
-      const hasConfiguredNotification = await AsyncStorage.getItem('hasConfiguredNotification');
-      if (params.showNotificationAfterFeedback === 'true' && !hasConfiguredNotification) {
+      const shouldShow = await AsyncStorage.getItem('showNotificationAfterFeedback');
+      if (shouldShow !== 'true') return;
+      // Clear the flag so it only triggers once
+      await AsyncStorage.removeItem('showNotificationAfterFeedback');
+
+      const hasConfiguredNotification = await AsyncStorage.getItem('hasConfiguredNotification');
+      if (!hasConfiguredNotification) {
         setShowNotificationScheduleModal(true);
       }
     };
     checkNotificationTrigger();
-  }, [params.showNotificationAfterFeedback, hasRatedApp]);
+  }, [hasRatedApp]);
```

不要な `Stack.Screen` コンポーネントと `usePathname()` も削除:

```diff tsx:components/HomeScreen.tsx
-      <Stack.Screen
-        options={{
-          headerShown: false,
-          animation: 'fade',
-        }}
-      />
       <RNStatusBar barStyle="light-content" />
```

```diff tsx:app/my-vocabulary.tsx
-import { useRouter, usePathname } from 'expo-router';
+import { useRouter } from 'expo-router';
-import { Stack } from 'expo-router';

 export default function MyVocabularyScreen() {
   const router = useRouter();
-  const pathname = usePathname();
```

### 修正2: Firestore リスナーの適切なクリーンアップ

`useRef` でクリーンアップ関数を管理し、新しいリスナー作成前に古いものを破棄するようにしました。

```diff tsx:components/HomeScreen.tsx
+  // Ref to track Firestore listener cleanup
+  const dataListenersRef = useRef<(() => void) | null>(null);
+
   const loadData = async () => {
+    // Clean up existing listeners before creating new ones
+    if (dataListenersRef.current) {
+      dataListenersRef.current();
+      dataListenersRef.current = null;
+    }
+
     try {
       const unsubscribeTags = TagsService.listenTags(/* ... */);
       const unsubscribeNotes = NotesService.listenNotes(/* ... */);

-      return () => {
+      dataListenersRef.current = () => {
         unsubscribeTags();
         unsubscribeNotes();
       };
     } catch (error) {
       Logger.recordError(error as Error, 'Failed to load data', 'loadData');
-      return () => {};
     }
   };

+  // Cleanup Firestore listeners on unmount
+  useEffect(() => {
+    return () => {
+      if (dataListenersRef.current) {
+        dataListenersRef.current();
+      }
+    };
+  }, []);
```

### 修正3: onLayout の比較ガード

値が変わっていない場合は `setState` をスキップし、不要な再レンダーを防止します。

```diff tsx:components/HomeScreen.tsx
 onLayout={(e) => {
   const { x, width } = e.nativeEvent.layout;
-  setTodayTabLayout({ x, width });
+  setTodayTabLayout(prev =>
+    prev.x === x && prev.width === width ? prev : { x, width }
+  );
 }}
```

## まとめ

### 学び

1. **Expo Router のフックは Expo Router のスクリーン内でのみ使う** — `createBottomTabNavigator()` など React Navigation のコンテキスト配下では `useLocalSearchParams()`、`usePathname()`、`<Stack.Screen>` は正しく動作しない
2. **Firestore リスナーは `useRef` で管理する** — `loadData()` の戻り値に頼らず、ref に保持して確実にクリーンアップする
3. **`onLayout` で `setState` する際は比較ガードを入れる** — 同じ値でも新しいオブジェクト参照は再レンダーを引き起こす

### チェックリスト

同じ問題に遭遇した場合に確認すべきポイントです。

- [ ] Expo Router のフック（`useLocalSearchParams`, `usePathname`, `useSegments`）を React Navigation のコンポーネント内で使っていないか?
- [ ] `<Stack.Screen>` を Tab.Navigator の子コンポーネント内で使っていないか?
- [ ] Firestore リスナーの `unsubscribe` が確実に呼ばれているか?
- [ ] `onLayout` や `onScroll` のコールバック内で毎回新しいオブジェクトを `setState` していないか?
- [ ] 画面間のデータ受け渡しに `useLocalSearchParams` を使う場合、その画面が本当に Expo Router で管理されているか?
