---
title: "Rust中継サーバーにOpusコーデックとGoAway自動再接続を実装した設計メモ"
emoji: "🎙️"
type: "tech"
topics: ["Rust", "WebSocket", "Opus", "GeminiAPI", "リアルタイム音声"]
published: true
---

## はじめに

Gemini Live API を使ったリアルタイム音声アプリを開発しています。構成は **モバイルアプリ ↔ Rust 中継サーバー ↔ Gemini Live API** の 3 層で、中継サーバーが認証・RAG・セッション管理を担当しています。

:::message
**対象読者**
- Gemini Live API（Multimodal Live API）を使った音声アプリに興味がある方
- WebSocket 中継サーバーの設計パターンを知りたい方
- Rust で Opus コーデックを扱いたい方
:::

この記事では、以下の 2 つの機能を実装した際の設計判断とハマりポイントをまとめます。

- **Opus コーデック対応** — PCM 生データ直送（~80KB/秒）を Opus 圧縮（~14KB/秒）にして帯域 82% 削減
- **GoAway 自動再接続** — Gemini が定期的に送る GoAway シグナル受信後、クライアントに透過的に再接続

## 全体像

```
┌─────────────┐         ┌──────────────────┐         ┌───────────────┐
│  Mobile App │◄──WS───►│  Rust Backend    │◄──WS───►│ Gemini Live   │
│             │  Opus   │                  │  PCM    │ API           │
│  (16kHz)    │  or PCM │  Decode / Encode │  16kHz  │               │
│             │         │  Auth / RAG      │  24kHz  │               │
└─────────────┘         └──────────────────┘         └───────────────┘

入力: App → [Opus 16kHz] → Backend → [PCM 16kHz] → Gemini
出力: Gemini → [PCM 24kHz] → Backend → [Opus 24kHz] → App
```

ポイントは **コーデック変換をサーバー側に閉じ込めている** こと。クライアントが Opus に対応していなくても PCM モードにフォールバックできます。

## 設計のポイント

### ポイント 1: コーデックネゴシエーション

クライアントの Opus 対応状況はプラットフォームによって異なります（React Native で Opus を扱うには native module が必要）。そこで、セッション開始時にネゴシエーションする方式にしました。

| 方式 | メリット | デメリット |
|------|---------|-----------|
| サーバー側で固定 | 実装シンプル | Opus 非対応クライアントが使えない |
| クライアント指定 | 柔軟 | サーバー側の対応確認が必要 |
| **ネゴシエーション** | 柔軟 + フォールバック可能 | プロトコル設計が少し複雑 |

セッション開始メッセージに `codec` フィールドを追加し、サーバーが対応可能な場合のみ `"opus"` で応答、失敗時は `"pcm"` にフォールバックします。

```json
// Client → Server
{ "type": "session_start", "codec": "opus" }

// Server → Client（Opus 対応時）
{ "type": "session_started", "session_id": "...", "codec": "opus" }

// Server → Client（Opus 初期化失敗時 = フォールバック）
{ "type": "session_started", "session_id": "...", "codec": "pcm" }
```

:::message
`codec` フィールドは `Option` にして省略可能にしています。既存クライアントが `codec` を送らなくても `"pcm"` がデフォルトで選ばれるため、**後方互換性を壊しません**。
:::

### ポイント 2: Opus のフレームバッファリング

Opus エンコーダは固定フレームサイズ（60ms 分のサンプル）を要求します。しかし Gemini から返ってくる PCM チャンクのサイズは不定です。

```
Gemini から: 320B → 640B → 480B → ...（不定長の PCM チャンク）
Opus が要求: 1440 samples (60ms @ 24kHz) = 2880B ずつ
```

そこで、**内部バッファにサンプルを蓄積し、フレームサイズに達したら encode する**方式にしました。

```rust
pub struct OpusEncoder {
    encoder: audiopus::coder::Encoder, // 24kHz mono
    buffer: Vec<i16>,                  // PCM sample accumulator
    frame_size: usize,                 // 1440 samples (60ms @ 24kHz)
}

impl OpusEncoder {
    /// PCM bytes を受け取り、完全なフレームが溜まったら Opus に encode。
    /// 0 個以上の Opus パケットを返す。
    pub fn encode(&mut self, pcm_bytes: &[u8]) -> Result<Vec<Vec<u8>>, String> {
        // bytes → i16 samples に変換してバッファに追加
        for chunk in pcm_bytes.chunks_exact(2) {
            let sample = i16::from_le_bytes([chunk[0], chunk[1]]);
            self.buffer.push(sample);
        }

        let mut packets = Vec::new();
        while self.buffer.len() >= self.frame_size {
            let frame: Vec<i16> = self.buffer.drain(..self.frame_size).collect();
            let mut output = vec![0u8; 4000];
            let len = self.encoder
                .encode(&frame[..], &mut output[..])
                .map_err(|e| format!("Opus encode error: {e}"))?;
            output.truncate(len);
            packets.push(output);
        }
        Ok(packets)
    }
}
```

:::message
`encode()` の戻り値が `Vec<Vec<u8>>`（0 個以上のパケット）なのがポイントです。小さい PCM チャンクが来た場合は 0 個（バッファに蓄積のみ）、大きいチャンクなら複数個のパケットを返します。
:::

### ポイント 3: エンコーダの共有方法

Opus エンコーダは Gemini からの受信タスク（`tokio::spawn` 内）で使われます。Rust の所有権ルールから、タスク間で共有するには `Arc<Mutex<>>` が必要です。

```rust
// Connection 側でエンコーダを作成
let encoder = Arc::new(Mutex::new(OpusEncoder::new()?));

// Gemini クライアントに渡す
let client = GeminiClient::connect(
    api_key, model, resumption_handle,
    client_tx, event_tx,
    Some(encoder.clone()), // Arc::clone で共有
).await?;
```

ここで `tokio::sync::Mutex` ではなく **`std::sync::Mutex`** を使っています。Opus エンコードは CPU バウンドで高速（マイクロ秒オーダー）なため、`.await` を跨がない `std::sync::Mutex` の方が適切です。

## 実装 Tips

### Tip 1: audiopus クレートのセットアップ

Rust で Opus を扱うには `audiopus` クレートが便利です。ただし、ネイティブの `libopus` が必要です。

```toml
# Cargo.toml
[dependencies]
audiopus = "0.2"
```

macOS では `brew install opus pkg-config` が必要です。`pkg-config` がないと `audiopus_sys` がシステムライブラリを見つけられず、ソースからビルドしようとして `autoreconf: command not found` で失敗します。

:::message alert
Docker / CI 環境では `apt-get install libopus-dev pkg-config` を忘れずに。
:::

### Tip 2: デコーダとエンコーダでサンプルレートが異なる

Gemini Live API の音声は**入力 16kHz / 出力 24kHz** と非対称です。デコーダとエンコーダで異なるサンプルレートを指定する必要があります。

```rust
// 入力: クライアント → Gemini（16kHz）
let decoder = Decoder::new(SampleRate::Hz16000, Channels::Mono)?;

// 出力: Gemini → クライアント（24kHz）
let encoder = Encoder::new(SampleRate::Hz24000, Channels::Mono, Application::Voip)?;
```

リサンプリングは不要です。各方向でサンプルレートが一貫しているため、デコーダ / エンコーダのコンストラクタで正しいレートを指定するだけで済みます。

### Tip 3: GoAway → Status メッセージへの変換

GoAway 受信時に即座に `session_ended` を送ると、クライアントがセッションを破棄してしまいます。代わりに `status` メッセージで「もうすぐ切れるけど再接続するよ」と通知します。

```rust
// Before: クライアントがセッション終了と解釈してしまう
InternalEvent::GoAway { seconds } => {
    send(ServerMessage::SessionEnded {
        reason: format!("goaway:{seconds}s"),
    });
}

// After: 情報通知のみ、セッションは継続
InternalEvent::GoAway { seconds } => {
    send(ServerMessage::Status {
        state: format!("goaway:{seconds}s"),
    });
}
```

クライアント側は `status` メッセージを受け取っても音声パイプラインを破棄せず、UI 表示だけ更新します。

## GoAway 自動再接続の設計

Gemini Live API は長時間セッションで定期的に GoAway シグナルを送り、接続を切断します。Session Resumption ハンドルを使えば、状態を引き継いで再接続できます。

```
時系列:
  [GoAway受信] → Status("goaway:30s") → クライアントに通知
  [Gemini切断] → GeminiDisconnected イベント発火
  → Status("reconnecting") → クライアントに通知
  → Redis から resumption handle 取得
  → 1秒待機 → 再接続試行 (最大2回)
  → 成功: Status("reconnected")
  → 失敗: SessionEnded("reconnection_failed")
```

### 再接続回数の制限

無限ループを防ぐために、セッション全体で最大 3 回までの再接続に制限しています。

```rust
const MAX_RECONNECTIONS: u32 = 3;

// GeminiDisconnected ハンドラ内
if total_reconnections >= MAX_RECONNECTIONS {
    send(ServerMessage::SessionEnded {
        reason: "reconnection_limit".to_string(),
    });
    return;
}
```

:::message
1 回の切断につき最大 2 回リトライ × セッション全体で 3 サイクルまで。10 分ごとに GoAway が来ても 30 分は持つ計算です。
:::

## ハマりポイント

### session_end 後の不要な再接続

テスト中に発見した問題です。クライアントが `session_end` を送った後、Gemini の受信タスクが切断を検知して `GeminiDisconnected` イベントが発火し、**意図しない再接続が走りました**。

```
session_end → session_ended → (Gemini切断検知) → reconnecting → reconnected
                                ↑ これは不要！
```

修正はシンプルで、`session_end` 処理時にフラグを立てて再接続を抑止します。

```diff
+ let mut session_ended_by_client = false;

  // session_end ハンドラ内
  ClientMessage::SessionEnd => {
+     session_ended_by_client = true;
      gemini_client.take().close().await;
      // ...
  }

  // GeminiDisconnected ハンドラ内
  InternalEvent::GeminiDisconnected { .. } => {
+     if session_ended_by_client {
+         continue; // 再接続しない
+     }
      // 再接続ロジック...
  }
```

:::message
WebSocket 中継サーバーでは「誰が切断を起こしたか」を常に追跡する必要があります。クライアント起因の切断とサーバー起因の切断で振る舞いを分けないと、予期しないループが発生します。
:::

## まとめ

### 設計判断の一覧

| 判断 | 選択 | 理由 |
|------|------|------|
| コーデック方式 | ネゴシエーション | 後方互換性 + フォールバック |
| Opus フレーム処理 | 内部バッファ蓄積 | Gemini のチャンクサイズが不定 |
| Mutex の種類 | `std::sync::Mutex` | CPU バウンド処理、`.await` を跨がない |
| GoAway 通知 | `Status` メッセージ | セッション継続のため `SessionEnded` を避ける |
| 再接続上限 | 3 回 / セッション | 無限ループ防止 |
| 切断種別の追跡 | フラグ管理 | クライアント起因の切断で再接続を抑止 |

### 学び

1. **コーデック変換はサーバー側に閉じ込める** — クライアントの対応状況に依存しない設計になる
2. **Opus のフレームバッファリングは必須** — 入力チャンクサイズが不定の場合、蓄積→エンコードのパターンが有効
3. **GoAway は「まもなく切断」の予告** — 即座にセッション終了ではなく、再接続の準備期間として活用する
4. **切断の発生源を追跡する** — 中継サーバーでは「誰が切断を起こしたか」で振る舞いを分ける必要がある
5. **`audiopus` は `libopus` + `pkg-config` が前提** — CI/Docker で忘れがち

### 実装チェックリスト

- [ ] `libopus` と `pkg-config` がビルド環境にインストールされているか
- [ ] デコーダ（16kHz）とエンコーダ（24kHz）のサンプルレートが正しいか
- [ ] `codec` フィールド省略時にデフォルト `"pcm"` で動作するか
- [ ] `session_end` 後に再接続が走らないか
- [ ] 再接続上限に達した場合に `session_ended` が送信されるか
- [ ] Docker/CI の Dockerfile に `libopus-dev` が含まれているか
