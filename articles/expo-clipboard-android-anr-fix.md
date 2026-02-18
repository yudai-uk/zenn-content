---
title: "expo-clipboard の ClipboardEventEmitter が引き起こす Android ANR を patch-package で修正する"
emoji: "📋"
type: "tech"
topics: ["reactnative", "expo", "android", "patchpackage", "crashlytics"]
published: true
---

## はじめに

Expo（React Native）アプリの Crashlytics で **Android ANR（Application Not Responding）** が報告されました。原因は `expo-clipboard` 内部の `ClipboardEventEmitter` がメインスレッドで同期的な Binder IPC（`getPrimaryClipDescription()`）を呼び出していたこと。

この記事では、原因の特定からパッチ適用までの手順を解説します。

:::message
**対象読者**: Expo（React Native）で Android アプリを開発しており、Crashlytics で ANR レポートを受け取ったことがある方
:::

## 問題: メインスレッドが ClipboardEventEmitter でブロックされる

### 症状

Crashlytics に以下のような ANR スタックトレースが報告されます。

```
main (blocked):
  android.os.BinderProxy.transactNative (Native method)
  android.content.ClipboardManager.getPrimaryClipDescription
  expo.modules.clipboard.ClipboardModule$ClipboardEventEmitter.<init>
```

メインスレッドが `ClipboardManager.getPrimaryClipDescription()` の同期 Binder IPC 呼び出しでブロックされ、5秒以上応答がないため ANR が発生していました。

### 原因

`expo-clipboard` の Android 実装（`ClipboardModule.kt`）には、クリップボードの変更を監視する `ClipboardEventEmitter` が組み込まれています。

```kotlin
// expo-clipboard の ClipboardModule.kt（簡略化）
OnCreate {
  clipboardEventEmitter = ClipboardEventEmitter()
  clipboardEventEmitter.attachListener()
}

private inner class ClipboardEventEmitter {
  fun attachListener() =
    clipboardManager?.addPrimaryClipChangedListener(listener)

  private val listener = OnPrimaryClipChangedListener {
    // getPrimaryClipDescription() を呼び出し
    // → 同期 Binder IPC でメインスレッドをブロック
    clipboardManager?.primaryClipDescription?.let { clip ->
      sendEvent("onClipboardChanged", ...)
    }
  }

  // コンストラクタで clipboardManager にアクセス
  // → ここでも Binder IPC が発生
  private val maybeClipboardManager =
    runCatching { clipboardManager }.getOrNull()
}
```

:::message
**ポイント**: `ClipboardEventEmitter` はモジュール初期化時（`OnCreate`）に無条件で生成・登録されます。アプリ側でクリップボード変更イベントを購読していなくても、リスナーは常に動作しています。
:::

この問題は以下の条件が重なると顕在化します:

| 条件 | 影響 |
|------|------|
| 他アプリがクリップボードを頻繁に更新 | リスナーが頻繁に発火 |
| システムのクリップボードサービスが高負荷 | Binder IPC のレイテンシ増大 |
| メインスレッドが UI 描画中 | ANR 判定の 5 秒閾値に到達しやすい |

### 修正

アプリで `Clipboard.setStringAsync()` のみ使用し、クリップボード変更イベントは不要であれば、`ClipboardEventEmitter` ごと削除するのが最もシンプルな解決策です。

#### Step 1: `buildFromSource` を設定する

`expo-clipboard` はデフォルトでプリビルド済み AAR（`.aar`）を使用するため、Kotlin ソースコードへのパッチが反映されません。`package.json` に以下を追加して、ソースからビルドするよう強制します。

```json
{
  "expo": {
    "autolinking": {
      "android": {
        "buildFromSource": ["expo-clipboard"]
      }
    }
  }
}
```

:::message alert
**重要**: この設定がないと、`patch-package` でパッチを当てても AAR が優先されるため変更が反映されません。Expo のネイティブモジュールをソース修正する際は必ず確認してください。
:::

#### Step 2: `ClipboardModule.kt` からリスナーコードを削除

`node_modules/expo-clipboard/android/src/main/java/expo/modules/clipboard/ClipboardModule.kt` を編集し、イベントリスナー関連のコードを削除します。

```diff
- import android.os.Build
  import android.text.Html
  import android.text.Html.FROM_HTML_MODE_LEGACY
  import android.text.Spanned
  import android.text.TextUtils
- import android.util.Log
- import androidx.core.os.bundleOf
  import expo.modules.core.utilities.ifNull

  private const val moduleName = "ExpoClipboard"
- private val TAG = ClipboardModule::class.java.simpleName

  const val CLIPBOARD_DIRECTORY_NAME = ".clipboard"
- const val CLIPBOARD_CHANGED_EVENT_NAME = "onClipboardChanged"
-
- private enum class ContentType(val jsName: String) {
-   PLAIN_TEXT("plain-text"),
-   HTML("html"),
-   IMAGE("image")
- }
```

`ModuleDefinition` 内のライフサイクルフック:

```diff
      clipboardManager.primaryClipDescription?.hasMimeType("image/*") == true
    }
-
-   Events(CLIPBOARD_CHANGED_EVENT_NAME)
-
-   OnCreate {
-     clipboardEventEmitter = ClipboardEventEmitter()
-     clipboardEventEmitter.attachListener()
-   }
-
-   OnDestroy {
-     clipboardEventEmitter.detachListener()
-   }
-
-   OnActivityEntersBackground {
-     clipboardEventEmitter.pauseListening()
-   }
-
-   OnActivityEntersForeground {
-     clipboardEventEmitter.resumeListening()
-   }
  }
```

`ClipboardEventEmitter` inner class 全体:

```diff
- private lateinit var clipboardEventEmitter: ClipboardEventEmitter
-
- private inner class ClipboardEventEmitter {
-   private var isListening = true
-   private var timestamp = -1L
-   fun resumeListening() { isListening = true }
-   fun pauseListening() { isListening = false }
-
-   fun attachListener() =
-     maybeClipboardManager?.addPrimaryClipChangedListener(listener)
-       .ifNull {
-         Log.e(TAG, "'CLIPBOARD_SERVICE' unavailable.")
-       }
-
-   fun detachListener() =
-     maybeClipboardManager?.removePrimaryClipChangedListener(listener)
-
-   private val listener = OnPrimaryClipChangedListener {
-     // ... イベント送信ロジック（省略）
-   }
-
-   private val maybeClipboardManager =
-     runCatching { clipboardManager }.getOrNull()
- }
```

#### Step 3: パッチファイルを生成

```bash
npx patch-package expo-clipboard
```

`patches/expo-clipboard+7.1.5.patch` が生成されます。`package.json` の `postinstall` に `patch-package` が設定されていれば、以降の `npm install` で自動適用されます。

```json
{
  "scripts": {
    "postinstall": "patch-package"
  }
}
```

#### Step 4: ビルドして検証

```bash
# prebuild（android ディレクトリを再生成）
npx expo prebuild --clean --platform android

# ビルド・実行
npx expo run:android
```

ビルドログで `expo-clipboard` がソースからコンパイルされていることを確認します:

```
> Task :expo-clipboard:compileDebugKotlin
> Task :expo-clipboard:compileDebugJavaWithJavac
```

:::message
AAR からビルドされている場合、これらのタスクは表示されません。`buildFromSource` の設定が正しく適用されていることの確認にもなります。
:::

## まとめ

| 項目 | 内容 |
|------|------|
| 問題 | `ClipboardEventEmitter` がメインスレッドで同期 Binder IPC を実行し ANR 発生 |
| 原因 | イベントリスナーが未使用でもモジュール初期化時に無条件で登録される |
| 修正 | `ClipboardEventEmitter` 関連コードを削除し `patch-package` で固定化 |

### 学び

1. **Expo モジュールの AAR ビルドに注意** — `patch-package` で Kotlin ソースを修正しても、AAR が優先されると変更が反映されない。`buildFromSource` 設定が必須
2. **使わないイベントリスナーもコストがある** — `expo-clipboard` のイベントリスナーは、アプリ側で購読しなくても Native 側で常時動作している
3. **Crashlytics の ANR スタックトレースを読む** — ANR は再現が難しいが、スタックトレースからブロッキング呼び出しの特定は可能

### 注意点

- パッチはバージョン固定（例: `expo-clipboard@7.1.5`）。SDK アップデート時にパッチの再生成が必要
- JS 側の `Clipboard.addClipboardListener()` API は残存するが、Native 側のイベント送信が削除されるため Android では no-op になる
- `setStringAsync` / `getStringAsync` / `hasStringAsync` など読み書き API は影響なし
