---
title: "pnpm モノレポで React Native を複数バージョン共存させる Metro 設定"
emoji: "🧱"
type: "tech"
topics: ["reactnative", "pnpm", "monorepo", "metro", "expo"]
published: true
---

## はじめに

pnpm モノレポに **React Native のバージョンが異なる2つの Expo アプリ** を同居させたところ、ビルド時に `SyntaxError` や `Invalid hook call` が連鎖的に発生しました。

原因は Metro バンドラーが pnpm の `.pnpm` ストアから **別アプリ用の react-native や react を誤って解決** していたことでした。本記事では、3つの Metro 設定（`extraNodeModules`・`blockList`・`resolveRequest`）を組み合わせて各アプリを完全に分離した方法を紹介します。

:::message
**対象読者**: pnpm モノレポで複数の React Native アプリ（異なる RN バージョン）を管理している方。Expo + Metro バンドラー環境を想定しています。
:::

## システム構成

```
monorepo/                          # pnpm workspace
├── apps/
│   ├── app-a/                     # Expo — react-native@0.79.x, react@19.0.0
│   └── app-b/                     # Expo — react-native@0.81.x, react@19.1.0
├── packages/                      # shared libraries
├── pnpm-workspace.yaml
└── package.json
```

2つのアプリが **異なる React Native バージョンと React バージョン** を使っている点が今回のポイントです。

## 問題1: SyntaxError — 新しい RN の構文が古い RN のアプリで読まれる

### 症状

app-a（RN 0.79.x）を Android でビルドすると、Metro バンドル時に以下のエラーが発生します。

```
SyntaxError: /node_modules/.pnpm/react-native@0.81.5.../
  Libraries/Components/View/VirtualView.js:
  Unexpected token (42:9)

  40 |
  41 | function VirtualView({mode, ...}: Props) {
> 42 |   match (mode) {
     |          ^
  43 |     'static' => return <StaticView {...} />;
```

RN 0.81 で導入された新しい構文（`match` 式）を、RN 0.79 用の Babel 設定では解析できません。

### 原因

Metro が **app-a の `react-native` ではなく、pnpm ストア内の app-b 用 `react-native@0.81.x` を解決** していました。

pnpm はすべてのパッケージを `node_modules/.pnpm/` にフラットに格納します。Metro がシンボリンクを辿って依存解決する過程で、本来参照すべきでない別バージョンの RN を発見してしまいます。

```
node_modules/.pnpm/
├── react-native@0.79.6.../    # app-a 用（正しい）
├── react-native@0.81.5.../    # app-b 用（誤って参照される）
└── some-library@.../
    └── node_modules/
        └── react-native → ??? （pnpm のリンクが 0.81 を指す場合がある）
```

## 問題2: Invalid hook call — React が複数バージョンバンドルされる

### 症状

app-b（RN 0.81.x）を起動すると、画面が真っ白のまま `logcat` に以下が出力されます。

```
E ReactNativeJS: 'Invalid hook call. Hooks can only be called inside
  of the body of a function component.
  ...
  3. You might have more than one copy of React in the same app'
```

### 原因

モノレポ内に **3つの React バージョン** が存在していました。

```bash
pnpm list react --depth=0 -r
# app-a:  react@19.0.0
# app-b:  react@19.1.0
# (pnpm store にはさらに react@19.0.0-rc も存在)
```

Metro がバンドル時にこれらを混在させ、コンポーネントの `useState` と `useEffect` が異なる React インスタンスを参照してしまいます。

## 修正: 3層の Metro 分離設定

解決には Metro の3つの仕組みを組み合わせます。

| 設定 | 役割 |
|------|------|
| `extraNodeModules` | import の解決先をアプリ直下の `node_modules` に固定 |
| `blockList` | 別アプリ用のバージョンをファイルシステムレベルで除外 |
| `resolveRequest` | `react` の import を強制的にインターセプトして固定 |

:::message
**なぜ3つ必要なのか？**
`extraNodeModules` は「通常の解決で見つからなかったときのフォールバック」なので、pnpm が先に別バージョンを見つけると効きません。`blockList` は強力ですが、正規表現の対象外の経路で漏れる可能性があります。`resolveRequest` で確実にインターセプトし、三重に防御します。
:::

### app-a の metro.config.js（RN 0.79.x 側）

```js:metro.config.js
const { getDefaultConfig } = require('expo/metro-config');
const path = require('path');

const config = getDefaultConfig(__dirname);
const appNodeModules = path.resolve(__dirname, 'node_modules');

// 1. extraNodeModules: react / react-native をアプリ直下に固定
config.resolver.extraNodeModules = {
  ...config.resolver.extraNodeModules,
  'react-native': path.resolve(appNodeModules, 'react-native'),
  'react': path.resolve(appNodeModules, 'react'),
};

// 2. blockList: 別アプリの RN バージョンをバンドル対象から除外
config.resolver.blockList = [
  ...(config.resolver.blockList ? [config.resolver.blockList] : []),
  /node_modules\/\.pnpm\/react-native@0\.81\./,
];

// 3. resolveRequest: react の import を完全にインターセプト
const defaultResolveRequest = config.resolver.resolveRequest;

config.resolver.resolveRequest = (context, moduleName, platform) => {
  if (
    moduleName === 'react' ||
    moduleName === 'react/jsx-runtime' ||
    moduleName === 'react/jsx-dev-runtime'
  ) {
    const subpath = moduleName === 'react'
      ? '' : moduleName.replace('react', '');
    const filePath = path.join(
      appNodeModules, 'react',
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

### app-b の metro.config.js（RN 0.81.x 側）

```js:metro.config.js
const { getDefaultConfig } = require('expo/metro-config');
const path = require('path');

const config = getDefaultConfig(__dirname);
const appNodeModules = path.resolve(__dirname, 'node_modules');

// 1. extraNodeModules
config.resolver.extraNodeModules = {
  ...config.resolver.extraNodeModules,
  'react-native': path.resolve(appNodeModules, 'react-native'),
  'react': path.resolve(appNodeModules, 'react'),
};

// 2. blockList: 別アプリの RN + 古い React を除外
config.resolver.blockList = [
  ...(config.resolver.blockList || []),
  /node_modules\/\.pnpm\/react-native@0\.79\./,
  /node_modules\/\.pnpm\/react@19\.0\.0[^-]/,
  /node_modules\/\.pnpm\/react@19\.0\.0-rc/,
];

// 3. resolveRequest
const defaultResolveRequest = config.resolver.resolveRequest;

config.resolver.resolveRequest = (context, moduleName, platform) => {
  if (
    moduleName === 'react' ||
    moduleName === 'react/jsx-runtime' ||
    moduleName === 'react/jsx-dev-runtime'
  ) {
    const subpath = moduleName === 'react'
      ? '' : moduleName.replace('react', '');
    const filePath = path.join(
      appNodeModules, 'react',
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
**`blockList` の正規表現について**

`/react@19\.0\.0[^-]/` は `react@19.0.0` にマッチし、`react@19.0.0-rc` にはマッチしません。RC 版も別途ブロックするために `/react@19\.0\.0-rc/` を追加しています。バージョンの境界を正確に指定しないと、意図しないパッケージまで除外してしまうので注意してください。
:::

## 修正の確認

```bash
# Metro キャッシュクリア & 再起動
rm -rf $TMPDIR/metro-*
npx expo start --dev-client --clear

# バンドルを取得して RN バージョンを確認
curl -s "http://localhost:8081/entry.bundle?platform=android&dev=true" \
  -o /tmp/bundle.js

grep -oE "react-native@[0-9][^/\"]*" /tmp/bundle.js | sort -u
# => react-native@0.79.6 のみ（app-a の場合）

grep -oE "react@[0-9][^/\"]*" /tmp/bundle.js | sort -u
# => react@19.0.0 のみ（app-a の場合）
```

バンドル内に想定外のバージョンが含まれていなければ成功です。

## まとめ

| 問題 | 原因 | 修正 |
|------|------|------|
| SyntaxError（新構文） | pnpm ストア内で別アプリ用の RN が解決される | `extraNodeModules` + `blockList` で正しい RN に固定 |
| Invalid hook call | React が複数バージョンバンドルされる | `resolveRequest` で react import をインターセプト |

### 学び

1. **pnpm モノレポ + Metro では、アプリごとに metro.config.js で依存を明示的に分離する必要がある。** npm/yarn と違い、pnpm のストア構造は Metro の想定外
2. **`extraNodeModules` だけでは不十分。** pnpm のシンボリンク解決で別バージョンが先に見つかる場合、フォールバックに到達しない
3. **`blockList` で不要なバージョンをファイルシステムレベルで除外するのが最も直接的。** 正規表現で `.pnpm/react-native@X.Y.` を指定する
4. **`resolveRequest` は最終防衛線。** すべての解決リクエストをインターセプトできるため、漏れがない
5. **バンドル内容を `curl` + `grep` で検証する習慣をつける。** 推測ではなくバンドルの中身を確認するのが最速の診断方法

### デバッグチェックリスト

- [ ] `pnpm list react react-native --depth=0 -r` でモノレポ内の全バージョンを確認
- [ ] `grep -oE "react(-native)?@[0-9][^/\"]*" bundle.js | sort -u` でバンドル内のバージョンを確認
- [ ] 各アプリの `metro.config.js` に `extraNodeModules` + `blockList` + `resolveRequest` を設定
- [ ] `blockList` の正規表現がバージョン境界を正確に指定しているか確認
- [ ] Metro キャッシュクリア後に再検証（`rm -rf $TMPDIR/metro-*`）
