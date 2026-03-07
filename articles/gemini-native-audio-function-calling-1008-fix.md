---
title: "Gemini Native Audio + Function Calling で WebSocket 1008 切断される問題と解決策"
emoji: "🔊"
type: "tech"
topics: ["Gemini", "LiveAPI", "WebSocket", "Rust", "音声AI"]
published: true
---

## はじめに

Gemini Live API（BidiGenerateContent）で音声対話アプリを開発中、**Function Calling（tools）を有効にすると最初の応答後に WebSocket が切断される**問題に遭遇しました。

:::message
**対象読者**
- Gemini Live API でリアルタイム音声対話を実装している方
- Native Audio モデルで Function Calling を使おうとしている方
- WebSocket 1008 エラーに悩んでいる方
:::

## システム構成

```
┌─────────────┐     WebSocket      ┌──────────────┐     WebSocket      ┌────────────┐
│  Mobile App │ ◄──────────────► │  Rust Backend │ ◄──────────────► │ Gemini API │
│ (React Native)│    PCM Audio     │   (Axum)     │  BidiGenerate   │ (Live API) │
└─────────────┘                    └──────────────┘   Content        └────────────┘
                                         │
                                         ▼
                                   ┌────────────┐
                                   │  Upstash   │
                                   │Redis/Vector│
                                   └────────────┘
```

クライアントから受け取った PCM 音声を Rust バックエンドが中継し、Gemini Live API に転送。Gemini の応答音声をクライアントに返すアーキテクチャです。RAG のために Function Calling（tools）を setup メッセージに含めています。

## 問題: 1回応答した後、2回目以降の音声が送れなくなる

### 症状

1回目の発話に対しては正常に音声応答が返る。しかし2回目以降に話しかけても**一切応答がない**。

バックエンドのログには以下のエラーが大量に出力されていました。

```
ERROR voice_backend::websocket::connection: Failed to send audio to Gemini:
  Failed to queue audio: channel closed
```

### 原因

ログを遡ると、最初の音声応答の直後に Gemini 側から接続が切断されていました。

```
ERROR voice_backend::gemini::client:
  Gemini connection closed: code=1008
  reason=Operation is not implemented, or supported, or enabled.
```

WebSocket close code `1008`（Policy Violation）で、Gemini サーバーが接続を強制終了しています。

これにより音声送信用のチャネル（`audio_tx`）が閉じられ、以降のすべての音声データが送信不能になっていました。

```
┌────────┐                    ┌────────────┐                ┌────────┐
│ Client │                    │  Backend   │                │ Gemini │
└───┬────┘                    └─────┬──────┘                └───┬────┘
    │  1. 音声送信                   │                           │
    │ ──────────────────────────► │  audio → Gemini            │
    │                              │ ──────────────────────► │
    │                              │                           │
    │                              │  ◄── 音声応答 + text     │
    │  ◄──────────────────────── │                           │
    │  (正常に聞こえる)              │                           │
    │                              │      ◄── close(1008)    │ ← ここで切断！
    │                              │   channel closed          │
    │  2. 音声送信                   │                           │
    │ ──────────────────────────► │  ✗ channel closed         │
    │  (応答なし)                    │                           │
```

:::message
**原因の特定ポイント**

ログに出ていた Gemini の思考テキストがヒントになりました：

```
Gemini text: **Greeting and Contextualizing**
I'm using the `get_user_learning_context` tool to better grasp
their recent interactions and history.
```

Gemini が音声応答後に Function Calling を実行しようとしたタイミングで、接続が切断されていました。
:::

### 使用していたモデル

```rust
// config.rs
gemini_model: env::var("GEMINI_MODEL").unwrap_or_else(|_| {
    "gemini-2.5-flash-native-audio-preview-12-2025".to_string()
}),
```

**プレビュー版**の `12-2025` モデルを使用していました。

## 調査: 利用可能なモデルの確認

Gemini API で `bidiGenerateContent`（Live API）をサポートするモデルを一覧取得しました。

```bash
curl -s "https://generativelanguage.googleapis.com/v1alpha/models?key=$API_KEY" \
  | jq '.models[]
    | select(.supportedGenerationMethods[]? == "bidiGenerateContent")
    | .name'
```

```
"models/gemini-2.0-flash-exp-image-generation"
"models/gemini-2.5-flash-native-audio-latest"
"models/gemini-2.5-flash-native-audio-preview-09-2025"
"models/gemini-2.5-flash-native-audio-preview-12-2025"
```

:::message
`gemini-2.0-flash-live-001` や `gemini-2.0-flash-exp` など、ドキュメントで見かけるモデル名は `bidiGenerateContent` をサポートしていませんでした。利用可能なモデルは API で確認するのが確実です。
:::

GitHub 上でも同様の問題が複数報告されています。

- [googleapis/python-genai #843](https://github.com/googleapis/python-genai/issues/843) — Function calling が native audio model で動作しない
- [googleapis/python-genai #1832](https://github.com/googleapis/python-genai/issues/1832) — Function calling が internal error を出す
- [googleapis/python-genai #1894](https://github.com/googleapis/python-genai/issues/1894) — 12-2025 版でツール結果返却前にハルシネーション
- [livekit/livekit #3679](https://github.com/livekit/livekit/issues/3679) — Native Audio が function calling をサポートしていない

## 修正: GA 版モデルに切り替え

`preview-12-2025` から **`latest`（GA 版）** に変更したところ、Function Calling 有効のまま正常に動作しました。

```diff
 gemini_model: env::var("GEMINI_MODEL").unwrap_or_else(|_| {
-    "gemini-2.5-flash-native-audio-preview-12-2025".to_string()
+    "gemini-2.5-flash-native-audio-latest".to_string()
 }),
```

修正後のログでは、音声応答 → `generationComplete` → `turnComplete` と正常にターンが完了し、接続が維持されました。

```
DEBUG Gemini recv: { "serverContent": { "generationComplete": true } }
DEBUG serverContent with no modelTurn (turn_complete=None)
DEBUG Gemini recv: { "serverContent": { "turnComplete": true }, "usageMetadata": { ... } }
```

1008 エラーも `channel closed` も発生せず、マルチターンの音声対話が可能になりました。

## 補足: コスト比較（Gemini vs OpenAI）

OpenAI の Realtime API は Function Calling が安定していますが、コストに大きな差があります。

| | Audio Input (per 1M tokens) | Audio Output (per 1M tokens) |
|---|---|---|
| **Gemini Native Audio** | $3.00 | $12.00 |
| **OpenAI gpt-4o-mini-realtime** | $10.00 | $20.00 |
| **OpenAI gpt-realtime** | $32.00 | $64.00 |

Gemini は OpenAI の約 1/3〜1/10 のコストです。Function Calling の安定性では OpenAI に分がありますが、GA 版で改善されたことを考えると、コスト面で Gemini を選ぶ価値は十分あります。

## まとめ

| 問題 | 原因 | 修正 |
|------|------|------|
| 1回目の応答後に接続切断 | preview 版モデルで Function Calling 未対応 | GA 版（`latest`）に変更 |
| WebSocket 1008 エラー | tools 設定 + preview モデルの組み合わせ | モデルバージョン変更のみ |

### 学び

1. **Native Audio のプレビュー版は Function Calling に非対応（またはバグあり）** — 公式ドキュメントではサポートされていると書いてあるが、実際にはプレビュー版では動作しない
2. **利用可能なモデルは API で確認する** — ドキュメント上のモデル名と実際に `bidiGenerateContent` で使えるモデルは異なる場合がある
3. **`latest` エイリアスを使う** — プレビュー版は期限切れのリスクもあるため、`gemini-2.5-flash-native-audio-latest` を使うのが安全
4. **ログに出る Gemini の「思考テキスト」が原因特定のヒントになる** — Native Audio モデルは音声応答前にテキストで思考過程を出力することがあり、そこにツール呼び出しの意図が現れる

### デバッグチェックリスト

- [ ] `bidiGenerateContent` 対応モデルを API で確認したか
- [ ] プレビュー版ではなく GA 版（`latest`）を使っているか
- [ ] WebSocket close code と reason をログに出力しているか
- [ ] Gemini からの受信メッセージ（テキスト含む）をデバッグログに出しているか
- [ ] Function Calling を無効にした場合に正常動作するか切り分けたか
