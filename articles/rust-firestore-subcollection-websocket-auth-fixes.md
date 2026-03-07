---
title: "RustでFirestoreサブコレクション操作時にハマった3つの罠【パス構築・WebSocket認証・部分更新】"
emoji: "🦀"
type: "tech"
topics: ["Rust", "Firestore", "WebSocket", "axum", "Firebase"]
published: true
---

## はじめに

Rust（axum）で構築したリアルタイム音声対話バックエンドで、**Firestore のサブコレクション操作**と **WebSocket 認証**に関する3つのバグに連続して遭遇しました。いずれも**コンパイルは通るがランタイムで失敗する**タイプで、原因特定に時間がかかったものです。

:::message
**対象読者**
- Rust で Firestore（`firestore-rs` crate）を使っている方
- axum で WebSocket エンドポイントを実装している方
- サブコレクション（`users/{uid}/notes` 等）を操作するバックエンドを書いている方
:::

## システム構成

```
┌─────────────┐     WebSocket      ┌──────────────┐      gRPC        ┌────────────┐
│  Mobile App │ ◄──────────────► │  Rust Backend │ ◄──────────────► │  Firestore │
│(React Native)│  ?token=xxx      │   (axum)     │   subcollection  │            │
└─────────────┘                    └──────────────┘   operations     └────────────┘
```

モバイルアプリから WebSocket で接続し、Firebase Auth トークンで認証後、ユーザーごとのサブコレクション（`users/{uid}/notes`, `users/{uid}/voiceSessions` 等）を読み書きする構成です。

## 問題1: WebSocket 接続で認証が効かず anonymous になる

### 症状

Firebase Auth で認証済みのユーザーなのに、バックエンド側のログでは常に `anonymous` として扱われていました。

```
WARN voice_backend: WebSocket auth failed: Missing Authorization header
INFO voice_backend: WebSocket upgrade request user_id="anonymous"
```

開発環境では anonymous フォールバックがあるため接続自体は成功しますが、ユーザー固有のデータにアクセスできない状態でした。

### 原因

**ブラウザや React Native の WebSocket API は、接続時にカスタム HTTP ヘッダーを設定できない**という仕様上の制約が原因です。

```
// Browser/React Native の WebSocket API
const ws = new WebSocket(url);
// ← headers を指定するオプションがない！
```

クライアント側はこの制約を知っていたため、トークンをクエリパラメータとして送信していました。

```typescript
// Client side
const wsUrl = `${baseUrl}?token=${encodeURIComponent(token)}`;
const ws = new WebSocket(wsUrl);
```

しかしバックエンド側は `Authorization` ヘッダーからのみトークンを読み取る実装になっていたため、トークンが無視されていました。

:::message alert
WebSocket の認証は HTTP の世界とルールが異なります。`Authorization` ヘッダーが使えないため、**クエリパラメータ**、**プロトコルヘッダー（Sec-WebSocket-Protocol）**、**接続後の最初のメッセージ**のいずれかでトークンを渡す必要があります。
:::

### 修正

axum の `Query` エクストラクターを追加し、クエリパラメータを優先的に確認するようにしました。

```rust
use axum::extract::{Query, State, WebSocketUpgrade};

#[derive(Deserialize, Default)]
pub struct WsQuery {
    token: Option<String>,
}

pub async fn ws_handler(
    ws: WebSocketUpgrade,
    State(state): State<AppState>,
    Query(query): Query<WsQuery>,
    headers: axum::http::HeaderMap,
) -> Response {
    // Query param first (WebSocket API cannot set custom headers),
    // then fall back to Authorization header.
    let user_id = if let Some(token) = &query.token {
        verify_firebase_token(token, &config.firebase_project_id).await
    } else {
        verify_token_from_headers(&headers, &config).await
    };

    // ...
}
```

WebSocket クライアントからはクエリパラメータ、REST クライアント（テスト等）からは `Authorization` ヘッダー、どちらでも認証できるようになりました。

## 問題2: Firestore サブコレクションのパス構築で InvalidArgument

### 症状

認証が通るようになった後、Firestore への読み書きが以下のエラーで全滅しました。

```
Firestore query failed: InvalidArgument:
  Collection id "users/abc123/voiceSessions" is invalid
  because it contains "/"
```

### 原因

`firestore-rs` crate の fluent API で、サブコレクションのパスを**スラッシュ結合した文字列**で渡していたことが原因です。

```rust
// NG: collection ID にスラッシュが含まれてしまう
self.db.fluent()
    .select()
    .from(&format!("users/{user_id}/notes"))  // ← これがコレクション ID として解釈される
    .obj::<NoteDocument>()
    .query()
    .await
```

Firestore の API では、コレクション ID はスラッシュを含まない単一のセグメント（`notes`, `voiceSessions` 等）であり、親ドキュメントのパスは別途指定する必要があります。

:::message
`firestore-rs` crate では、サブコレクションへのアクセスに `parent_path()` メソッドで親パスを構築し、`.parent()` でクエリに渡すパターンを使います。Firebase のコンソール URL（`users/abc/notes/xyz`）をそのままコードに持ち込むと失敗します。
:::

### 修正

`parent_path()` + `.parent()` パターンに統一しました。

```diff
+ let parent = self.db
+     .parent_path("users", user_id)
+     .map_err(|e| format!("Failed to construct parent path: {e}"))?;

  self.db.fluent()
      .select()
-     .from(&format!("users/{user_id}/notes"))
+     .from("notes")
+     .parent(&parent)
      .obj::<NoteDocument>()
      .query()
      .await
```

同じパターンを select / insert / update すべてに適用します。

ここで注意が必要なのは、**insert と update では `.parent()` を呼べるタイミングがビルダーチェーンの特定の位置に限定される**ことです。

```rust
// insert: .generate_document_id() の後に .parent()
self.db.fluent()
    .insert()
    .into("voiceSessions")
    .generate_document_id()
    .parent(&parent)           // ← ここ
    .object(&session)
    .execute::<VoiceSessionDocument>()
    .await

// update: .document_id() の後に .parent()
self.db.fluent()
    .update()
    .in_col("voiceSessions")
    .document_id(doc_id)
    .parent(&parent)           // ← ここ
    .object(&session)
    .execute::<VoiceSessionDocument>()
    .await
```

`.parent()` の呼び出し位置を間違えるとコンパイルエラーになります（ビルダーパターンで型が変わるため）。crate のソースコードを読んで正しい位置を確認する必要がありました。

:::details firestore-rs の builder chain と .parent() が使える位置

```
select:
  .fluent().select().from(collection).parent(&parent)  → OK
  .fluent().select().by_id_in(collection).parent(&parent)  → OK

insert:
  .fluent().insert().into(collection).generate_document_id().parent(&parent)  → OK
  .fluent().insert().into(collection).parent(&parent)  → NG (コンパイルエラー)

update:
  .fluent().update().in_col(collection).document_id(id).parent(&parent)  → OK
  .fluent().update().in_col(collection).parent(&parent)  → NG (コンパイルエラー)
```

これは各ステップで返される型が異なり、`.parent()` メソッドが特定の型にしか実装されていないためです。
:::

## 問題3: 部分更新の応答デシリアライズでフィールド欠落エラー

### 症状

セッション終了時にサマリーだけを更新する処理で、以下のエラーが発生していました。

```
Failed to update session summary:
  missing field `userId` at line X column Y
```

Firestore への書き込み自体は成功しているのに、**レスポンスのデシリアライズ**で失敗するという状況です。

### 原因

`firestore-rs` の `.fields()` で更新フィールドを限定すると、**Firestore のレスポンスにもそのフィールドしか含まれない**ことが原因でした。

```rust
// NG: summary だけ更新するが、レスポンスを VoiceSessionDocument として
//     デシリアライズしようとする
self.db.fluent()
    .update()
    .fields(paths!(VoiceSessionDocument::summary))  // summary だけ更新
    .in_col("voiceSessions")
    .document_id(session_doc_id)
    .parent(&parent)
    .object(&session)
    .execute::<VoiceSessionDocument>()  // ← 全フィールド必要な型でデシリアライズ
    .await
```

Firestore の UpdateDocument API はフィールドマスクを指定すると、レスポンスにもマスクされたフィールドのみを返します。`VoiceSessionDocument` には `user_id: String`（必須フィールド）があるため、レスポンスに `userId` が含まれずデシリアライズが失敗します。

:::message
これは「書き込みは成功しているがレスポンスの解釈で失敗する」パターンです。Firestore コンソールで確認するとデータは正しく更新されているので、混乱しやすいバグです。
:::

### 修正

レスポンスの型を `serde_json::Value` に変更し、部分的なレスポンスでも受け取れるようにしました。

```diff
  self.db.fluent()
      .update()
      .fields(paths!(VoiceSessionDocument::summary))
      .in_col("voiceSessions")
      .document_id(session_doc_id)
      .parent(&parent)
      .object(&session)
-     .execute::<VoiceSessionDocument>()
+     .execute::<serde_json::Value>()  // Partial response - don't deserialize to full type
      .await
```

他の選択肢として、`.fields()` を外して全フィールドを更新する方法もありますが、不要なフィールドの上書きリスクがあるため、`serde_json::Value` で受ける方が安全です。

## まとめ

| 問題 | 原因 | 修正 |
|------|------|------|
| WebSocket 認証が効かない | WebSocket API はカスタムヘッダーを送れない | クエリパラメータからトークンを読む |
| サブコレクション操作で InvalidArgument | パスをスラッシュ結合してコレクション ID に渡していた | `parent_path()` + `.parent()` パターンに変更 |
| 部分更新でデシリアライズエラー | `.fields()` 指定時のレスポンスにフィールドが欠落 | `.execute::<serde_json::Value>()` で受ける |

### 学び

1. **WebSocket の認証は HTTP REST と異なる** — `Authorization` ヘッダーが使えないことを前提に設計する
2. **Firestore のサブコレクションパスはコレクション ID と親パスを分離する** — URL 風のスラッシュ結合は使えない
3. **crate の fluent API はソースコードを読んでビルダーチェーンの型遷移を理解する** — ドキュメントだけでは分からないことがある
4. **部分更新時のレスポンス型に注意** — フィールドマスクはレスポンスにも影響する
5. **「コンパイルが通る ≠ 正しい」** — 特に文字列パスやジェネリック型パラメータは実行時まで問題が露呈しない

### デバッグチェックリスト

- [ ] WebSocket エンドポイントでクエリパラメータからの認証をサポートしているか
- [ ] Firestore サブコレクション操作で `parent_path()` + `.parent()` を使っているか
- [ ] insert の `.parent()` は `.generate_document_id()` の後に呼んでいるか
- [ ] update の `.parent()` は `.document_id()` の後に呼んでいるか
- [ ] `.fields()` で部分更新する際、`.execute::<T>()` の `T` が部分レスポンスに対応できるか
