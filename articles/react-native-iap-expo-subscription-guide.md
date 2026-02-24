---
title: "Expo + react-native-iap v12 でサブスク課金を実装する際のハマりどころと設計Tips"
emoji: "💎"
type: "tech"
topics: ["reactnative", "expo", "iap", "android", "ios"]
published: true
---

## はじめに

React Native（Expo）アプリにアプリ内課金（IAP）のサブスクリプションを実装しました。Web（Stripe）経由の課金のみだったアプリに、iOS/Android 両対応のネイティブ課金を追加する作業です。

:::message
**対象読者**
- Expo（Prebuild）環境で `react-native-iap` を導入したい方
- v14 と v12 のどちらを使うべきか迷っている方
- Android ビルドで Kotlin バージョンや Product Flavor のエラーに遭遇した方
:::

この記事で得られる知見：

- **react-native-iap v14 vs v12** — Expo 環境でどちらを選ぶべきか
- **Android ビルドエラー2つの解決法** — Kotlin バージョン不一致 / Product Flavor 衝突
- **IAPService の設計パターン** — シングルトン + コールバック方式
- **プラットフォーム別の価格表示** — Android `subscriptionOfferDetails` / iOS `localizedPrice` の違い
- **Expo Config Plugin** で Android ネイティブ設定を注入する方法

## 全体像

```
┌─────────────────────────────────────────────────┐
│  React Native App                               │
│                                                 │
│  ┌──────────┐   ┌───────────────┐              │
│  │ ShopView │──▶│ useShopHook   │              │
│  └──────────┘   └──────┬────────┘              │
│                        │                        │
│                 ┌──────▼────────┐               │
│                 │  IAPService   │  (singleton)  │
│                 │  - initialize │               │
│                 │  - purchase   │               │
│                 │  - restore    │               │
│                 │  - listeners  │               │
│                 └──────┬────────┘               │
│                        │                        │
└────────────────────────┼────────────────────────┘
                         │ httpsCallable
                 ┌───────▼────────┐
                 │ Cloud Function │
                 │ verifyReceipt  │
                 └───────┬────────┘
                         │
                 ┌───────▼────────┐
                 │   Firestore    │
                 │ users/{uid}    │
                 │ .subscription  │
                 └────────────────┘
```

**ユーザー視点のフロー**:
1. ペイウォール画面で年額/月額プランを選択
2. ストアの購入シートが表示される
3. 購入完了 → Cloud Function でレシート検証
4. Firestore の `subscriptionStatus` が更新される
5. アプリ側の Firestore リスナーが変更を検知 → 即座に Premium 機能が解放

## 設計のポイント

### ポイント1: react-native-iap v14 vs v12

react-native-iap は v14 から **Nitro Modules** ベースに書き直されました。API が刷新され、型定義もすっきりしています。しかし、Expo 環境では大きな落とし穴があります。

| 項目 | v14（Nitro） | v12（Bridge） |
|------|-------------|--------------|
| Kotlin 要件 | **2.2.0** | 2.0.21（Expo デフォルト） |
| 追加依存 | `react-native-nitro-modules` | なし |
| Android API | `fetchProducts()` / `requestPurchase()` | `getSubscriptions()` / `requestSubscription()` |
| Product Flavor | なし | `amazon` / `play` |
| Expo SDK 52 互換 | ❌ ビルド不可 | ✅ |

:::message alert
**v14 が Expo でビルドできない理由**

v14 の依存ライブラリ（`billing-ktx:8.3.0`）は Kotlin 2.2.0 でコンパイルされていますが、Expo SDK 52 の React Native Gradle Plugin は **Kotlin 2.0.21 のコンパイラを内蔵**しています。

`expo-build-properties` で `kotlinVersion` を上げても、**コンパイラ本体は React Native Gradle Plugin に組み込まれている**ため変更できません。結果として `kotlin-stdlib-2.2.0.jar` のメタデータを 2.0.x コンパイラが読めずビルドエラーになります。

```
Module was compiled with an incompatible version of Kotlin.
The binary version of its metadata is 2.2.0, expected version is 2.0.0.
```

Expo SDK が Kotlin 2.2.0 をサポートするまでは **v12 一択**です。
:::

### ポイント2: 購入状態管理 — コールバック方式

`react-native-iap` の購入フローは**非同期リスナー駆動**です。`requestSubscription()` を呼んでも、結果は別の `purchaseUpdatedListener` / `purchaseErrorListener` で返ってきます。

UI 側で「購入中」のローディング状態を管理するには、Service 層にコールバックを登録する設計が有効です。

```typescript
// IAPService.ts — コールバック登録
class IAPService {
  private onPurchaseSuccess: ((purchase: Purchase) => void) | null = null;
  private onPurchaseError: ((error: PurchaseError) => void) | null = null;

  setCallbacks(
    onSuccess: ((purchase: Purchase) => void) | null,
    onError: ((error: PurchaseError) => void) | null
  ) {
    this.onPurchaseSuccess = onSuccess;
    this.onPurchaseError = onError;
  }
}
```

```typescript
// Hook 側 — コールバックで UI 状態を更新
useEffect(() => {
  const iapService = IAPService.getInstance();

  iapService.setCallbacks(
    () => {
      // Purchase succeeded — clear loading
      setPurchasingProductId(null);
    },
    (error) => {
      // Purchase failed — clear loading, show error
      setPurchasingProductId(null);
      if (error.code !== ErrorCode.E_USER_CANCELLED) {
        setError('Purchase failed');
      }
    }
  );

  return () => iapService.setCallbacks(null, null);
}, []);
```

:::message
**なぜ `finally` ブロックで解除しないのか？**

`requestSubscription()` は購入シートを表示するだけで、購入の成否を待ちません。`try/finally` で `setPurchasingProductId(null)` すると、ユーザーがまだ購入シートを見ている最中にローディングが消えてしまいます。
:::

## 実装 Tips

### Tip 1: Android の Product Flavor 衝突を Config Plugin で解決する

react-native-iap v12 は `amazon` / `play` の Product Flavor を持っています。アプリ側にも独自の Flavor（例: `dev` / `prd`）がある場合、Gradle が variant を解決できずエラーになります。

```
Could not resolve project :react-native-iap.
Cannot choose between the following variants:
  - amazonDebugApiElements
  - playDebugApiElements
```

**解決策**: `defaultConfig` に `missingDimensionStrategy` を追加する Expo Config Plugin を作成します。

```javascript
// plugins/withIAPStoreFlavor.js
const { withAppBuildGradle } = require("expo/config-plugins");

function withIAPStoreFlavor(config) {
  return withAppBuildGradle(config, (config) => {
    if (config.modResults.language === "groovy") {
      let src = config.modResults.contents;

      if (!src.includes('missingDimensionStrategy "store"')) {
        src = src.replace(
          /defaultConfig \{/,
          `defaultConfig {\n        missingDimensionStrategy "store", "play"`
        );
      }

      config.modResults.contents = src;
    }
    return config;
  });
}

module.exports = withIAPStoreFlavor;
```

```typescript
// app.config.ts
const withIAPStoreFlavor = require('./plugins/withIAPStoreFlavor');

export default {
  plugins: [
    withIAPStoreFlavor,
    // ...
  ],
};
```

`"play"` を指定すると Google Play Billing が選択され、Amazon Appstore 向けには `"amazon"` に変更できます。

### Tip 2: プラットフォーム別の価格情報を統一的に扱う

v12 の `Subscription` 型は iOS と Android で構造が大きく異なります。

**iOS**: `price`（string）, `currency`, `localizedPrice` が直接プロパティにある
**Android**: `subscriptionOfferDetails[].pricingPhases.pricingPhaseList[]` の中にネストされている

統一的に扱うヘルパー関数を用意します。

```typescript
function extractPriceInfo(product: Subscription): {
  price: number;
  currency: string;
  localizedPrice: string;
} {
  if (Platform.OS === 'android' && 'subscriptionOfferDetails' in product) {
    const details = (product as any).subscriptionOfferDetails;
    const phases = details?.[0]?.pricingPhases?.pricingPhaseList;
    if (phases?.length) {
      // Use the last phase (base price, not trial/intro)
      const basePhase = phases[phases.length - 1];
      return {
        price: parseInt(basePhase.priceAmountMicros, 10) / 1_000_000,
        currency: basePhase.priceCurrencyCode,
        localizedPrice: basePhase.formattedPrice,
      };
    }
  }

  // iOS
  return {
    price: parseFloat((product as any).price ?? '0'),
    currency: (product as any).currency ?? 'USD',
    localizedPrice: (product as any).localizedPrice ?? '',
  };
}
```

:::message
**Android の `priceAmountMicros` に注意**

Android は価格をマイクロ単位（1,000,000 = 1通貨単位）の**文字列**で返します。`parseInt` してから `1_000_000` で割る必要があります。
:::

### Tip 3: Android 購入に必要な `offerToken` のキャッシュ

Android の `requestSubscription()` には `offerToken` が**必須**です。これは `getSubscriptions()` の戻り値に含まれる値で、事前にキャッシュしておく必要があります。

```typescript
class IAPService {
  private offerTokens: Record<string, string> = {};

  async getProducts(): Promise<Subscription[]> {
    const subscriptions = await getSubscriptions({ skus: SKU_LIST });

    // Cache offer tokens for later purchase requests
    if (Platform.OS === 'android') {
      for (const sub of subscriptions) {
        if (sub.subscriptionOfferDetails?.length) {
          this.offerTokens[sub.productId] =
            sub.subscriptionOfferDetails[0].offerToken;
        }
      }
    }

    return subscriptions;
  }

  async purchaseSubscription(sku: string): Promise<void> {
    if (Platform.OS === 'android') {
      const offerToken = this.offerTokens[sku];
      if (!offerToken) {
        throw new Error(`No offer token for ${sku}`);
      }
      await requestSubscription({
        subscriptionOffers: [{ sku, offerToken }],
      });
    } else {
      await requestSubscription({ sku });
    }
  }
}
```

### Tip 4: 通貨フォーマットは `Intl.NumberFormat` で

日額・月額換算価格の表示には `Intl.NumberFormat` を使うと、ユーザーのロケールに合わせた通貨表示ができます。

```typescript
const formatPrice = (amount: number, currency: string): string => {
  try {
    return new Intl.NumberFormat(undefined, {
      style: 'currency',
      currency,
      maximumFractionDigits: currency === 'JPY' ? 0 : 2,
    }).format(amount);
  } catch {
    return `${currency} ${amount.toFixed(2)}`;
  }
};

// Usage: daily price for yearly plan
const dailyPrice = yearlyPrice / 365;
const text = formatPrice(dailyPrice, 'JPY'); // → "¥11"
```

:::message
JPY など小数点のない通貨では `maximumFractionDigits: 0` を指定しないと `¥10.58` のような不自然な表示になります。
:::

## ハマりポイント

### Kotlin バージョン不一致の調査過程

最初 v14 を導入してビルドしたところ、以下のエラーが発生しました。

```
Module was compiled with an incompatible version of Kotlin.
The binary version of its metadata is 2.2.0, expected version is 2.0.0.
```

`expo-build-properties` で `kotlinVersion: "2.2.0"` を設定し、Config Plugin で `ext.kspVersion` も注入しましたが、**Kotlin コンパイラ自体が React Native Gradle Plugin に 2.0.21 として埋め込まれている**ため解決できませんでした。

:::details 調査で辿ったファイルパス
```
node_modules/expo-modules-core/android/ExpoModulesCorePlugin.gradle
  → kotlinVersion は rootProject.ext から取得

node_modules/expo-modules-autolinking/android/.../KSPLookup.kt
  → KSP バージョンマップに 2.2.0 がない（最大 2.1.20）

node_modules/react-native/gradle/libs.versions.toml
  → kotlin = "2.0.21" がハードコード

node_modules/@react-native/gradle-plugin/react-native-gradle-plugin/build.gradle.kts
  → libs.kotlin.gradle.plugin で 2.0.21 のコンパイラを同梱
```

結論：**コンパイラバージョンはアプリ側から変更不可** → v12 にダウングレード
:::

### Product ID の命名規則

iOS と Android で Product ID の命名規則が異なる点に注意が必要です。

```typescript
import { Platform } from 'react-native';

export const IAP_PRODUCT_IDS = {
  YEARLY: Platform.select({
    ios: 'premium.yearly.v2',      // iOS: ドット区切り
    android: 'premium_yearly_v2',  // Android: アンダースコア区切り
  }) as string,
  MONTHLY: Platform.select({
    ios: 'premium.monthly.v2',
    android: 'premium_monthly_v2',
  }) as string,
};
```

## まとめ

### 技術選定・設計判断の一覧

| 判断 | 選択 | 理由 |
|------|------|------|
| IAP ライブラリ | react-native-iap **v12** | v14 は Kotlin 2.2.0 必須で Expo SDK 52 と非互換 |
| 購入状態管理 | コールバック方式 | リスナー駆動のため `finally` では対応不可 |
| レシート検証 | Cloud Function | サーバー側で検証し Firestore に書き込む |
| Flavor 解決 | Config Plugin | `missingDimensionStrategy` を prebuild で自動注入 |
| 価格表示 | `Intl.NumberFormat` | ロケール安全、通貨別の小数点処理 |

### 学び

1. **Expo 環境の Kotlin バージョンは React Native Gradle Plugin が支配している** — `expo-build-properties` では変えられない部分がある
2. **react-native-iap の iOS / Android 型定義は別物** — 統一ヘルパーが必須
3. **Android は `offerToken` が購入に必須** — `getSubscriptions()` 時にキャッシュしておく
4. **Config Plugin は Expo のネイティブ設定を柔軟に拡張できる** — `withAppBuildGradle` で `build.gradle` を直接操作可能

### 実装チェックリスト

- [ ] `react-native-iap` のバージョンが Expo SDK の Kotlin バージョンと互換か確認
- [ ] Android: `missingDimensionStrategy` で store flavor を解決
- [ ] Android: `offerToken` を `getSubscriptions()` 時にキャッシュ
- [ ] iOS: `localizedPrice` / Android: `subscriptionOfferDetails` から価格を統一的に抽出
- [ ] 購入状態はコールバック方式で管理（`finally` ブロックではなく）
- [ ] `flushFailedPurchasesCachedAsPendingAndroid()` を初期化時に呼ぶ
- [ ] レシート検証はサーバー側（Cloud Function）で行う
- [ ] `finishTransaction()` を必ず呼ぶ（呼ばないと次回起動時にリトライされる）
