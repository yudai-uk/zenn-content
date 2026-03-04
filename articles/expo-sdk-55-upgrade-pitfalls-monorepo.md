---
title: "Expo SDK 55 アップグレードで遭遇した2つの罠 — fullScreenModal フリーズと pnpm hoisting"
emoji: "🪤"
type: "tech"
topics: ["expo", "reactnative", "pnpm", "monorepo", "newarchitecture"]
published: true
---

## はじめに

pnpm モノレポで複数の Expo アプリを管理しているプロジェクトで、Expo SDK 53/54 → 55 へのアップグレードを行いました。`npx expo install --fix` で依存が揃った後、実機テストで **2つの深刻な問題** に遭遇しました。

:::message
**対象読者**
- Expo SDK 55 へのアップグレードを検討している方
- pnpm モノレポで複数の Expo アプリを管理している方
- New Architecture（Fabric）への移行で問題が起きている方
:::

## モノレポ構成

```
monorepo/
├── apps/
│   ├── app-a/          # Expo SDK 53 → 55
│   └── app-b/          # Expo SDK 54 → 55
├── packages/
│   └── shared-types/   # 共有型定義
├── services/
│   ├── backend/        # Express API
│   └── functions/      # Cloud Functions
├── pnpm-workspace.yaml
└── .npmrc              # node-linker=hoisted
```

## 問題1: fullScreenModal でアプリがフリーズする

### 症状

モーダル画面（`presentation: 'fullScreenModal'`）を閉じると、**アプリ全体がフリーズ**して一切の操作ができなくなる。クラッシュではなく、画面がそのまま固まる。コンソールにエラーは出ない。

```tsx
// _layout.tsx
<Stack.Screen
  name="practice-session"
  options={{
    presentation: 'fullScreenModal',  // ← これが原因
    headerShown: false,
  }}
/>
```

### 調査プロセス

最初はモーダル内のコンポーネント（音声サービスの停止処理、ルーターの `back()` vs `dismiss()`）を疑いましたが、すべて空振り。

**切り分けのために最小テスト画面を作成:**

```tsx
// test-screen.tsx — 閉じるボタンだけの画面
export default function TestScreen() {
  const router = useRouter();
  return (
    <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
      <Button title="Close" onPress={() => router.back()} />
    </View>
  );
}
```

この最小画面でも `fullScreenModal` で開くとフリーズが再現 → **コンポーネントの問題ではなく、ナビゲーション層の問題**と確定。

次に `presentation` を `'card'` に変えたところ、フリーズしなくなった。

### 原因

**react-native-screens 4.23 + New Architecture（Fabric）の組み合わせ**で、`fullScreenModal` の dismiss 処理にバグがある。

Expo SDK 55 では Legacy Architecture が完全に削除され、New Architecture が強制されます。SDK 54 以前では Legacy Architecture がデフォルトだったため、この問題は顕在化しませんでした。

:::message
**ポイント**: Expo SDK 55 = New Architecture 強制。これまで動いていた `fullScreenModal` が突然壊れる可能性がある。
:::

### 修正

`fullScreenModal` を `card` + `slide_from_bottom` アニメーションで代替します。見た目はほぼ同じです。

```diff
 <Stack.Screen
   name="practice-session"
   options={{
-    presentation: 'fullScreenModal',
+    presentation: 'card',
+    animation: 'slide_from_bottom',
     headerShown: false,
+    gestureEnabled: false,
   }}
 />
```

サインイン画面のように上下スライドが不要な場合は `fade` アニメーションも使えます。

```diff
 <Stack.Screen
   name="signin"
   options={{
-    presentation: 'fullScreenModal',
+    presentation: 'card',
+    animation: 'fade',
     gestureEnabled: false,
   }}
 />
```

## 問題2: pnpm hoisting で「cannot find native module」エラー

### 症状

app-a をビルドして起動すると、以下のエラーが大量に発生:

```
Uncaught Error: Cannot find native module 'ExpoStoreReview'
Uncaught Error: Cannot find native module 'ExpoImage'
Uncaught Error: Cannot find native module 'ExpoDevClient'
...
```

**app-a は `expo-store-review` も `expo-image` も使っていない。** これらは app-b の依存パッケージ。

### 原因

pnpm の `node-linker=hoisted` 設定により、app-b の依存パッケージがルートの `node_modules` にホイストされます。

```
monorepo/
├── node_modules/
│   ├── expo-store-review/   ← app-b の依存だが root にホイスト
│   ├── expo-image/          ← 同上
│   └── expo-dev-client/     ← 同上
├── apps/
│   ├── app-a/               ← expo-store-review を使わない
│   └── app-b/               ← expo-store-review を使う
```

Expo のオートリンキングシステムがルートの `node_modules` をスキャンして、app-a のネイティブバイナリにもこれらを組み込もうとします。`autolinking.exclude` でネイティブ側は除外できますが、**Metro バンドラーの JS 解決は防げません**。

:::message
**`autolinking.exclude` だけでは不十分。** ネイティブ Pod のリンクは防げるが、Metro が JS モジュールを解決してランタイムで native module を探しに行く経路は残る。
:::

### 修正: 2段構えの対策

#### Step 1: autolinking.exclude（ネイティブ側）

```json
// apps/app-a/package.json
{
  "expo": {
    "autolinking": {
      "exclude": [
        "expo-store-review",
        "expo-dev-client",
        "expo-image"
      ]
    }
  }
}
```

#### Step 2: Metro resolveRequest（JS 側）

```js
// apps/app-a/metro.config.js
const { getDefaultConfig } = require('expo/metro-config');
const config = getDefaultConfig(__dirname);

// Block hoisted packages from other apps
const blockedModules = [
  'expo-store-review',
  'expo-dev-client',
  'expo-image',
  '@expo/ui',
];

const originalResolveRequest = config.resolver.resolveRequest;
config.resolver.resolveRequest = (context, moduleName, platform) => {
  if (blockedModules.some(
    (blocked) => moduleName === blocked || moduleName.startsWith(blocked + '/')
  )) {
    return { type: 'empty' };
  }
  if (originalResolveRequest) {
    return originalResolveRequest(context, moduleName, platform);
  }
  return context.resolveRequest(context, moduleName, platform);
};

module.exports = config;
```

#### Step 3: クリーン prebuild

autolinking.exclude を変更した後は、**必ずクリーン prebuild** を実行します。既存の `ExpoModulesProvider.swift` に古い設定が残っているためです。

```bash
npx expo prebuild --clean --platform ios
```

:::details ExpoModulesProvider.swift の確認方法
prebuild 後に以下のファイルを確認すると、どのモジュールがネイティブにリンクされているか分かります:

```bash
# iOS
cat ios/Pods/Target\ Support\ Files/Pods-YourApp/ExpoModulesProvider.swift

# 除外したモジュールが含まれていないことを確認
grep -E "StoreReview|DevClient|ExpoImage" ios/Pods/.../ExpoModulesProvider.swift
```
:::

## まとめ

| 問題 | 原因 | 修正 |
|------|------|------|
| `fullScreenModal` でフリーズ | react-native-screens 4.23 + New Architecture | `card` + `slide_from_bottom` で代替 |
| 他アプリの native module エラー | pnpm hoisting + Expo autolinking | `autolinking.exclude` + Metro `resolveRequest` |

### 学び

1. **Expo SDK 55 = New Architecture 強制**。Legacy Architecture は削除されたため、これまで動いていたコードが壊れる可能性がある
2. **最小再現で切り分ける**。fullScreenModal の問題は、空のテスト画面を作ることで5分で原因を特定できた
3. **pnpm モノレポでは autolinking.exclude だけでは不十分**。Metro の `resolveRequest` でJS側もブロックする必要がある
4. **`ExpoModulesProvider.swift` を読む**。どのモジュールがネイティブにリンクされているか一目で分かる

### デバッグチェックリスト

- [ ] `fullScreenModal` を使っている画面がないか確認（SDK 55 で壊れる可能性）
- [ ] `npx expo prebuild --clean` を実行して `ExpoModulesProvider` を再生成
- [ ] pnpm モノレポで他アプリの依存がホイストされていないか確認（`ls node_modules | grep expo-`）
- [ ] Metro config の `resolveRequest` でブロックリストが設定されているか確認
