---
title: "Firebase Auth直後のFirestore書き込みでpermission-deniedになる原因と対策【React Native】"
emoji: "🔐"
type: "tech"
topics: ["Firebase", "Firestore", "ReactNative", "Expo", "認証"]
published: true
---

## はじめに

React Native（Expo）アプリで新規ユーザー登録直後に Firestore の別コレクションへ書き込むと、`firestore/permission-denied` エラーが発生する問題に遭遇しました。

セキュリティルール自体は正しく設定されているのに、**新規登録直後だけ** 失敗するという厄介なケースです。

:::message
**対象読者**
- Firebase Auth + Firestore を使っている React Native / Expo 開発者
- 新規登録フロー直後に Firestore 書き込みエラーが出て困っている方
- `permission-denied` なのにルールは合っている、という状況の方
:::

## システム構成

```
┌──────────────┐     ┌──────────────────┐     ┌────────────────┐
│  React Native │────▶│  Firebase Auth   │────▶│  Firestore     │
│  (Expo)       │     │  (認証トークン)    │     │  (セキュリティ  │
│               │     │                  │     │   ルール評価)   │
└──────────────┘     └──────────────────┘     └────────────────┘
                              │
                              ▼
                     クライアント側で Auth 状態が
                     確定する前に書き込みが先行する
                     レースコンディション
```

Firestore のセキュリティルールは以下のように設定されています。

```
match /accounts/{accountId} {
  allow read, write: if request.auth != null
                     && request.auth.uid == accountId;
}
```

このルールは正しいのですが、**クライアント側で Auth の認証状態が Firestore に利用可能になる前に書き込みが実行されると**、`request.auth` が `null` と評価されて `permission-denied` になります。

## 問題：新規登録直後の Firestore 書き込みが permission-denied になる

### 症状

新規ユーザーのサインアップ完了後、ユーザー情報を Firestore の `accounts` コレクションに保存する処理が `firestore/permission-denied` で失敗していました。

Crashlytics には以下のような non-fatal エラーが記録されていました。

```
[firestore/permission-denied] The caller does not have permission
  at accounts/{userId}
```

ポイントは以下の3つです。

- **既存ユーザーのログインでは発生しない**（新規登録時のみ）
- セキュリティルールは正しく `request.auth.uid == accountId` と設定済み
- `users` コレクションへの書き込みは成功している

### 原因

根本原因は **クライアント側での Auth 状態遷移中に Firestore 書き込みが先行するレースコンディション** と **fire-and-forget パターンの組み合わせ** でした。

:::message alert
これは「必ず失敗する」バグではなく、**タイミングによって発生するレースコンディション** です。端末の性能やネットワーク状況によって再現率が変わるため、開発中は気づきにくく、本番の Crashlytics で初めて発覚するケースが多いです。
:::

```typescript
// User creation logic
async function createUser(userId: string, isFromSignup: boolean) {
  // ... user document creation ...

  if (isFromSignup) {
    // ❌ await なし = fire-and-forget
    WelcomeService.saveAccountInfo();
  }

  return true; // ← saveAccountInfo() の完了を待たずに return
}
```

`saveAccountInfo()` の中では `getIdToken(true)` でトークンをリフレッシュしてから Firestore に書き込む設計でしたが、**呼び出し元が `await` していないため、`createUser` が `return true` した後にトークンリフレッシュと書き込みが非同期で走っていました。**

:::message
**なぜ `users` コレクションは成功するのか？**

`users` コレクションへの書き込みは `createUser()` 内で直接 `await` して実行されています。この時点では Firebase Auth の初期トークンが有効で、`users` コレクションのセキュリティルール評価に通ります。

問題は `accounts` コレクションへの書き込みが別の非同期フロー（fire-and-forget）で実行されるため、**クライアント側で Auth の認証状態が Firestore に利用可能になる前に書き込みが先行する** ことがある点です。
:::

### 時系列で見る問題の発生メカニズム

```
時間 →

[Firebase Auth]  サインアップ完了 ──── トークン発行 ──── Firestore に反映
                     │                                      │
[createUser]    users書き込み(✅)                            │
                     │                                      │
[fire-and-forget]    └─── accounts書き込み(❌ permission-denied)
                              ↑
                    Auth 状態が Firestore クライアントに
                    まだ利用可能になっていない
```

## 修正

### 修正1：await を追加して実行順序を保証

最もシンプルかつ重要な修正です。

```diff typescript
  if (isFromSignup) {
-   WelcomeService.saveAccountInfo();
+   await WelcomeService.saveAccountInfo();
  }
```

`saveAccountInfo()` 内部で `getIdToken(true)` を呼んでトークンをリフレッシュしているため、`await` することでトークンが確実に更新された後に Firestore 書き込みが実行されるようになります。

:::message
`getIdToken(true)` は Firebase Auth に強制的にトークンリフレッシュを要求するメソッドです。`true` を渡すことでキャッシュを無視して新しいトークンを取得します。
:::

### 修正2：permission-denied に対するリトライロジック（フォールバック）

`await` + `getIdToken(true)` でほぼ解決しますが、フォールバックとしてリトライロジックを追加します。

:::message
**より根本的な対策としては** `onAuthStateChanged` や `onIdTokenChanged` で Auth 状態の確定を待ってから Firestore 操作を行うアプローチもあります。ただし、既存のサインアップフローに大きな変更を加えずに対処したい場合は、以下のリトライパターンが現実的です。
:::

```typescript
async function saveAccountWithRetry(
  userId: string,
  email: string
): Promise<void> {
  const maxAttempts = 2;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      const accountRef = firestore()
        .collection("accounts")
        .doc(userId);
      const accountDoc = await accountRef.get();

      const data: Record<string, any> = {
        userId,
        email,
        platform: Platform.OS === "ios" ? "iOS" : "Android",
        updatedAt: firestore.FieldValue.serverTimestamp(),
      };

      // Only set createdAt for new documents
      if (!accountDoc.exists) {
        data.createdAt = firestore.FieldValue.serverTimestamp();
      }

      await accountRef.set(data, { merge: true });
      return; // Success
    } catch (error: any) {
      const isPermissionDenied =
        error?.code === "firestore/permission-denied";

      if (isPermissionDenied && attempt < maxAttempts) {
        // Wait for auth token to propagate
        await new Promise((r) => setTimeout(r, 1000));
        continue;
      }

      // Log the final failure
      Logger.recordError(error, `Failed (attempt ${attempt}/${maxAttempts})`);
    }
  }
}
```

**設計ポイント：**

| 項目 | 値 | 理由 |
|------|-----|------|
| リトライ回数 | 1回（計2試行） | トークン伝播は通常1秒以内に完了するため |
| 待機時間 | 1秒 | Auth → Firestore のトークン反映に十分な時間 |
| リトライ対象 | `permission-denied` のみ | 他のエラー（ネットワーク切断等）は即失敗 |

### 修正3：エラーログの適切な管理

`console.error` は本番ビルドでもパフォーマンスに影響を与える可能性があります。Crashlytics の `Logger.recordError()` に置き換えることで、エラーの追跡と分析が適切に行えるようになります。

```diff typescript
- console.error('Failed to save account:', error);
+ Logger.recordError(
+   error instanceof Error ? error : new Error(String(error)),
+   'Failed to save account',
+   'WelcomeService.saveAccountWithRetry'
+ );
```

### 修正4：キャッシュのクリーンアップ

セッション内で一度処理したユーザーを `Set` でキャッシュしている場合、エラー発生時にキャッシュをクリアしないと **同一セッション中のリトライが永久にスキップされます。**

```diff typescript
  } catch (error) {
+   // Clear cache so this can be retried
+   sentCache.delete(userId);
    Logger.recordError(error, 'Failed to process');
  }
```

## まとめ

| 問題 | 原因 | 修正 |
|------|------|------|
| 新規登録直後に `permission-denied` | fire-and-forget で Auth トークン反映前に書き込み | `await` で実行順序を保証 |
| ネットワーク遅延時も失敗 | トークン伝播にラグがある | permission-denied 時に1秒待機してリトライ |
| エラーが追跡できない | `console.error` で記録 | `Logger.recordError()` で Crashlytics に記録 |
| リトライ不可 | キャッシュがクリアされない | catch 節でキャッシュをクリア |

### 学び

1. **fire-and-forget は認証直後のフローでは使わない。** Auth トークンの Firestore への反映は即座ではない
2. **`getIdToken(true)` でリフレッシュしても、Firestore クライアントへの反映には追加の時間が必要な場合がある。** リトライロジックがセーフティネットになる
3. **セキュリティルールが正しくても `permission-denied` になる場合、タイミングの問題を疑う。** 特に新規登録フロー、トークンリフレッシュ直後に注意
4. **エラーキャッシュの管理を忘れない。** エラー時にキャッシュをクリアしないと、リトライ機会を永久に失う

### この対策では防げないケース

- **セキュリティルール自体の誤り**: ルールのロジックが間違っている場合はリトライしても解決しない
- **Custom Claims の未反映**: Admin SDK で設定した Custom Claims は `getIdToken(true)` でリフレッシュしないと反映されない。Claims に依存するルールの場合は別途対応が必要
- **Firestore のクォータ超過**: プロジェクトレベルのクォータ制限による拒否は `permission-denied` とは異なるエラーコードになる

### デバッグチェックリスト

- [ ] Firebase Auth のサインアップ / ログイン直後に Firestore 書き込みをしていないか？
- [ ] fire-and-forget（await なし）で認証依存の処理を呼んでいないか？
- [ ] `getIdToken(true)` を呼んだ後、十分な時間を空けて Firestore に書き込んでいるか？
- [ ] エラー時のキャッシュクリアが実装されているか？
- [ ] `console.error` ではなく適切なエラー追跡ツール（Crashlytics 等）を使っているか？
