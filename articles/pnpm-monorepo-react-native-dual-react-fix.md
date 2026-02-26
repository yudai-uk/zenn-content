---
title: "pnpm モノレポで React Native が \"Invalid hook call\" — React が2重バンドルされる原因と修正"
emoji: "🔀"
type: "tech"
topics: ["reactnative", "pnpm", "monorepo", "metro", "expo"]
published: true
---

## はじめに

pnpm モノレポに React Native（Expo）アプリと Next.js アプリを同居させたところ、React Native アプリが **"Invalid hook call"** / **"Cannot read property 'useMemo' of null"** で起動しなくなりました。

Metro のキャッシュクリアでは直らず、バンドル内部を調査した結果、**2つの異なる React バージョンが同時にバンドルされていた**ことが原因でした。

:::message
**対象読者**: pnpm モノレポで React Native（Metro バンドラー）と Web アプリを共存させている方。特に、モノレポ内の複数アプリで React のバージョンが異なるケースに該当します。
:::

## システム構成

```
monorepo/                      # pnpm workspace
├── apps/
│   ├── mobile/                # React Native (Expo) — react@19.0.0
│   └── web/                   # Next.js — react@19.0.0-rc
├── packages/                  # shared libraries
├── pnpm-workspace.yaml
└── package.json
```

2つのアプリが**微妙に異なる React バージョン**を使っている点がポイントです。

## 問題: アプリが起動しない

### 症状

React Native アプリを起動すると、スプラッシュスクリーンのまま先に進まず、`adb logcat` に以下のエラーが出力されます。

```
E ReactNativeJS: 'Invalid hook call. Hooks can only be called inside
  of the body of a function component. This could happen for one of
  the following reasons:
  1. You might have mismatching versions of React and the renderer
  2. You might be breaking the Rules of Hooks
  3. You might have more than one copy of React in the same app
  See https://react.dev/link/invalid-hook-call for tips ...'

W ReactNativeJS: Warning: TypeError: Cannot read property 'useMemo' of null
```

Hook のルール違反はしていないし、`react` と `react-native` のバージョンは正しい。**3番目の「React が複数コピーある」** が怪しいと判断しました。

### 原因の調査

Metro が配信するバンドルをダウンロードして調べます。

```bash
# Metro が配信中のバンドルを取得
curl -s "http://localhost:8081/entry.bundle?platform=android&dev=true" \
  -o /tmp/bundle.js

# バンドル内の React バージョンを確認
grep -oE "react@[0-9][^/\"]*" /tmp/bundle.js | sort -u
```

```
react@19.0.0
react@19.0.0-rc-66855b96-20241106
```

:::message alert
**2つの React がバンドルに含まれていた。** `react@19.0.0`（mobile 用）と `react@19.0.0-rc`（web 用）が同時にバンドルされています。
:::

### 根本原因: pnpm の `.pnpm` ストア構造と Metro の解決順序

pnpm はすべてのパッケージを `.pnpm` ストアにフラットに格納し、シンボリンクで繋ぎます。

```
node_modules/.pnpm/
├── react@19.0.0/node_modules/react/        # mobile 用
├── react@19.0.0-rc-.../node_modules/react/ # web 用（Next.js が要求）
└── expo-router@.../node_modules/
    └── (react のシンボリンクなし → 親をたどる)
```

**Metro バンドラーがシンボリンクを辿って依存を解決する際、pnpm ストア内で「隣にある」RC 版 React を一部のパッケージが見つけてしまう**のが原因です。

具体的にどのモジュールが RC 版を参照しているかは、バンドル内のモジュール ID を追跡すると分かります。

```bash
# モジュール ID 640 = react@19.0.0-rc とする場合
# そのモジュールに依存しているモジュールを検索
grep -n "\[640\]" /tmp/bundle.js | grep -v "react@19.0.0-rc"
```

```
# 結果例:
expo-modules-core/src/Refs.ts
expo-router/build/global-state/storeContext.js
react-native-draggable-flatlist/src/hooks/useStableCallback.ts
```

一部のサードパーティパッケージが RC 版 React を参照してしまい、アプリ本体が使う `react@19.0.0` と衝突していました。

## 修正: `resolveRequest` で React の解決先を固定する

### 試したが効かなかったこと

| アプローチ | 結果 |
|-----------|------|
| `rm -rf $TMPDIR/metro-*` + Metro キャッシュクリア | キャッシュの問題ではないので解消しない |
| `extraNodeModules` で React パスを指定 | pnpm ストア内部の解決には効かない |
| アプリのアンインストール + 再ビルド | 埋め込みバンドルにも同じ問題がある |

### 解決策: `resolveRequest` でインターセプト

Metro の `resolveRequest` を使い、**`react` の import を常に mobile アプリの `node_modules/react` に強制的に解決**させます。

```js:metro.config.js
const { getDefaultConfig } = require('expo/metro-config');
const path = require('path');

const config = getDefaultConfig(__dirname);

// Monorepo: pin React to this app's version
// Prevents pnpm from resolving react@RC (used by web app)
const appNodeModules = path.resolve(__dirname, 'node_modules');
const defaultResolveRequest = config.resolver.resolveRequest;

config.resolver.resolveRequest = (context, moduleName, platform) => {
  if (
    moduleName === 'react' ||
    moduleName === 'react/jsx-runtime' ||
    moduleName === 'react/jsx-dev-runtime'
  ) {
    const subpath = moduleName === 'react'
      ? ''
      : moduleName.replace('react', '');
    const filePath = path.join(
      appNodeModules,
      'react',
      subpath ? subpath + '.js' : 'index.js'
    );
    return { type: 'sourceFile', filePath };
  }
  if (defaultResolveRequest) {
    return defaultResolveRequest(context, moduleName, platform);
  }
  return context.resolveRequest(context, moduleName, platform);
};

module.exports = config;
```

:::message
**なぜ `extraNodeModules` ではダメなのか？**
`extraNodeModules` は「通常の解決で見つからなかった場合のフォールバック」です。pnpm のシンボリンク構造では通常の解決で RC 版が先に見つかるため、フォールバックに到達しません。`resolveRequest` はすべての解決リクエストをインターセプトできるので確実です。
:::

### 修正の確認

```bash
# Metro キャッシュクリア & 再起動
rm -rf $TMPDIR/metro-*
npx expo start --dev-client --clear

# バンドルを再取得して検証
curl -s "http://localhost:8081/entry.bundle?platform=android&dev=true" \
  -o /tmp/bundle-fixed.js
grep -c "react@19.0.0-rc" /tmp/bundle-fixed.js
# => 0  (RC 版が完全に排除された)
```

## おまけ: New Architecture での `setLayoutAnimationEnabledExperimental` 警告

React Native の New Architecture（Fabric）を有効にしている場合、以下の警告が出ます。

```
W ReactNativeJS: 'setLayoutAnimationEnabledExperimental is currently
  a no-op in the New Architecture.'
```

これは New Architecture では `LayoutAnimation` がネイティブ側で自動的にサポートされるため、旧来のワークアラウンドが不要になったことを示しています。

```diff:components/HomeScreen.tsx
  import {
    View,
    Text,
    Platform,
-   UIManager,
  } from 'react-native';

- // Enable LayoutAnimation for Android
- if (Platform.OS === 'android' && UIManager.setLayoutAnimationEnabledExperimental) {
-   UIManager.setLayoutAnimationEnabledExperimental(true);
- }
```

`newArchEnabled=true`（`android/gradle.properties`）の場合はこのコードを削除するだけで警告が消えます。

## まとめ

| 問題 | 原因 | 修正 |
|------|------|------|
| "Invalid hook call" / "useMemo null" | pnpm ストア内で React が2バージョン解決される | `resolveRequest` で React の解決先を固定 |
| `setLayoutAnimationEnabledExperimental` 警告 | New Architecture では不要な API | 該当コードの削除 |

### 学び

1. **pnpm モノレポ + Metro の組み合わせでは、React のバージョン統一が最重要。** 同一ストアに異なるバージョンが存在するだけで、Metro が誤解決する可能性がある
2. **「キャッシュクリアで直る」と思い込まない。** バンドル内容を実際にダウンロードして `grep` するのが最速の診断方法
3. **`extraNodeModules` は pnpm のシンボリンク解決には無力。** `resolveRequest` によるインターセプトが必要
4. **`resolveRequest` は Metro の解決を完全に制御できる強力なフック。** モノレポ固有の問題はここで解決するのが定石

### デバッグチェックリスト

- [ ] `grep -oE "react@[0-9][^/\"]*" bundle.js | sort -u` でバンドル内の React バージョンを確認
- [ ] `pnpm list react --depth=0 -r` でモノレポ内の全 React バージョンを確認
- [ ] 異なるバージョンがある場合 → `resolveRequest` で固定 or `pnpm.overrides` で統一
- [ ] New Architecture 有効時 → `setLayoutAnimationEnabledExperimental` を削除
