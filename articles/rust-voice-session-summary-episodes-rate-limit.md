---
title: "Rust音声AIサーバーにセッション要約・エピソード記憶抽出・レート制限を実装する"
emoji: "🧠"
type: "tech"
topics: ["Rust", "GeminiAPI", "Redis", "WebSocket", "AI"]
published: true
---

## はじめに

[前回の記事](https://zenn.dev/yudai_uk/articles/rust-opus-codec-goaway-reconnect-gemini-live)では、Gemini Live API を使った音声会話サーバーに Opus コーデックと GoAway 自動再接続を実装しました。今回は **セッション終了後の自動処理** と **利用制限** を追加します。

:::message
**対象読者**
- リアルタイム音声 AI アプリのバックエンドを開発している方
- LLM を使った会話データのポストプロセッシングに興味がある方
- WebSocket セッションにレート制限を適用したい方
:::

この記事で扱う実装は以下の 3 つです。

1. **セッション要約生成** — 会話終了後に Gemini Text API で 3〜5 文の要約を自動生成
2. **エピソード記憶抽出** — 会話から体験・予定・好み・感情を構造化 JSON で抽出し、ベクトル DB にインデックス
3. **レート制限** — Redis で同時 1 セッション、1 日 10 回、1 日 60 分を強制

## 全体像

```
セッション中                              セッション終了後（非同期）
┌──────────┐    ┌──────────┐    ┌──────────┐
│ Mobile   │◄──►│  Rust    │◄──►│ Gemini   │
│ App      │ WS │ Backend  │ WS │ Live API │
└──────────┘    └────┬─────┘    └──────────┘
                     │
              ┌──────┴──────┐
              │ Rate Limit  │ ← Redis (SET NX)
              │ Check       │
              └─────────────┘

                     ▼ セッション終了時

              ┌─────────────┐    ┌───────────┐
              │ Conversation│───►│ Gemini    │
              │ Turns       │    │ Text API  │
              └─────────────┘    │(Flash)    │
                                 └─────┬─────┘
                                       │
                          ┌────────────┼────────────┐
                          ▼            ▼            ▼
                    ┌──────────┐ ┌──────────┐ ┌──────────┐
                    │ Summary  │ │ Episodes │ │ Vector   │
                    │→Firestore│ │→Firestore│ │→Upstash  │
                    └──────────┘ └──────────┘ └──────────┘
```

ポイントは **ポストプロセッシングを `tokio::spawn` で非同期実行** していること。クライアントはセッション終了のレスポンスを即座に受け取り、要約生成・エピソード抽出はバックグラウンドで走ります。

## 設計のポイント

### ポイント 1: Live API と Text API の使い分け

Gemini には 2 種類の API があり、用途に応じて使い分けます。

| API | プロトコル | 用途 | モデル |
|-----|----------|------|--------|
| Live API | WebSocket（双方向ストリーム） | リアルタイム音声会話 | gemini-2.5-flash-native-audio |
| Text API | REST（generateContent） | 要約・抽出などバッチ処理 | gemini-2.5-flash |

:::message
Live API は音声のリアルタイム処理に最適化されていますが、会話終了後のテキスト解析には REST の Text API（`generateContent`）を使う方がシンプルで安価です。
:::

Text API クライアントは `responseMimeType: "application/json"` を指定して **構造化 JSON 出力** を強制します。これにより、レスポンスのパースが安定します。

```rust
let body = json!({
    "contents": [{
        "parts": [{"text": prompt}]
    }],
    "systemInstruction": {
        "parts": [{"text": system_instruction}]
    },
    "generationConfig": {
        "responseMimeType": "application/json",
        "temperature": 0.3,
    }
});
```

### ポイント 2: ポストプロセッシングの非同期実行

セッション終了時の処理は **クライアントへのレスポンスをブロックしない** ことが重要です。要約生成に数秒かかるため、`tokio::spawn` で非同期タスクとして実行します。

```
クライアント                    サーバー                      Gemini Text API
    │                              │                              │
    │──── session_end ────────────►│                              │
    │                              │── save_session ──► Firestore │
    │◄── session_ended ───────────│                              │
    │                              │                              │
    │  (接続クローズ)              │── spawn ──────────────────►│
    │                              │                   generate_summary
    │                              │                   extract_episodes
    │                              │◄────────── results ─────────│
    │                              │── save to Firestore/Vector  │
```

実装上の工夫として、**要約とエピソード抽出を直列で実行** しています。並列にすることもできますが、Gemini API のレート制限に引っかかるリスクを避けるためです。

```rust
// セッション終了時のクリーンアップ（簡略化）
tokio::spawn(async move {
    // 1. 会話履歴を保存
    let doc_id = rag.save_session(&user_id, &doc).await?;

    // 2. 要約を生成して保存
    if let Ok(summary) = generate_summary(&text_client, &doc.turns).await {
        rag.update_session_summary(&user_id, &doc_id, &summary).await.ok();
    }

    // 3. エピソードを抽出して保存 + ベクトルインデックス
    if let Ok(episodes) = extract_episodes(&text_client, &doc.turns).await {
        rag.save_episodes(&user_id, &doc_id, &episodes).await.ok();
    }
});
```

### ポイント 3: エピソード記憶の構造化抽出

エピソード記憶は、ユーザーの発言から **体験・予定・好み・感情** を自動抽出し、次回のセッションでパーソナライズに使います。

```json
{
  "episodes": [
    {
      "content": "先週東京タワーに行った",
      "episode_type": "experience",
      "entities": ["東京タワー"],
      "date_referenced": "2026-02-28T00:00:00Z"
    },
    {
      "content": "来月大阪に出張する予定",
      "episode_type": "plan",
      "entities": ["大阪"],
      "date_referenced": "2026-04-01T00:00:00Z"
    }
  ]
}
```

抽出されたエピソードは Firestore に保存すると同時に、Upstash Vector にもインデックスします。次回のセッション開始時にベクトル検索でヒットし、システムプロンプトに注入されます。

:::message
**なぜ episode_type を分けるか？**
タイプ別に分けることで、「前回の旅行の話をしよう」（experience 検索）「来週の予定は？」（plan 検索）のようにコンテキストに応じた検索が可能になります。
:::

## 実装 Tips

### Tip 1: Gemini JSON モードで安定したパースを実現する

Gemini の Text API で構造化出力を得るには `responseMimeType: "application/json"` が有効です。ただし、レスポンスのテキストは `candidates[0].content.parts[0].text` に文字列として格納されているため、2 段階のパースが必要です。

```rust
// Gemini API レスポンスから JSON テキストを取り出す
let text = response["candidates"][0]["content"]["parts"][0]["text"]
    .as_str()
    .ok_or("No text in response")?;

// JSON テキストをデシリアライズ
let result: T = serde_json::from_str(text)?;
```

:::message
`responseMimeType` を指定しないと、Gemini がマークダウンのコードブロック（` ```json ... ``` `）で囲んで返すことがあり、パースに失敗します。必ず指定しましょう。
:::

### Tip 2: Redis SET NX で同時セッション制限を原子的に実装する

「1 ユーザー 1 セッション」の制約は、Redis の `SET key value NX EX ttl` で原子的に実装できます。

```
SET voice:active:{user_id} "1" NX EX 1200
```

- **NX**（Not eXists）: キーが存在しない場合のみ SET → 2 つ目のセッション開始を防ぐ
- **EX**: TTL を設定 → セッションが異常終了してもキーが残り続けない

```rust
pub async fn check_and_acquire(&self, user_id: &str) -> Result<(), RateLimitError> {
    let active_key = format!("voice:active:{user_id}");
    match self.redis.set_nx(&active_key, "1", self.session_ttl_secs).await {
        Ok(true)  => { /* 取得成功 */ }
        Ok(false) => return Err(RateLimitError::ConcurrentSession),
        Err(_)    => { /* Redis 障害時はフェイルオープン */ }
    }

    // 日次制限チェック（count >= 10 or total_seconds >= 3600）
    let daily_key = format!("voice:daily:{user_id}:{today}");
    if let Ok(Some(usage)) = self.redis.get::<DailyUsage>(&daily_key).await {
        if usage.count >= 10 { /* 解放してエラー */ }
        if usage.total_seconds >= 3600 { /* 解放してエラー */ }
    }
    Ok(())
}
```

:::message
**フェイルオープン設計**: Redis が落ちている場合、レート制限チェックを通過させます。レート制限は「あると嬉しい」機能であり、Redis 障害でサービス全体が止まるのは本末転倒だからです。
:::

### Tip 3: 日次使用量の追跡は GET→SET で十分

日次のセッション回数と合計時間は、JSON オブジェクトとして Redis に保存します。

```
voice:daily:{user_id}:2026-03-07 → {"count": 3, "total_seconds": 1200}
```

原子性が必要なら `HINCRBY` や Lua スクリプトを使うべきですが、同時 1 セッション制約があるため **同一ユーザーの並行更新は発生しません**。単純な GET→更新→SET で問題ありません。

| 方式 | 原子性 | 実装コスト | 今回の判断 |
|------|--------|-----------|-----------|
| Lua スクリプト | ✅ | 高（デバッグ困難） | 不要 |
| HINCRBY | ✅ | 中（Hash 型） | 不要 |
| GET → SET | ❌ | 低 | ✅ 採用（並行更新なし） |

### Tip 4: セッション最大時間を tokio::select! で強制する

WebSocket のメインループに 15 分タイマーを追加し、セッション時間を制限します。`tokio::select!` にタイムアウトの分岐を追加するだけです。

```rust
let mut session_start: Option<tokio::time::Instant> = None;

loop {
    // セッション開始済みならデッドラインを計算、未開始なら永久待機
    let timeout = async {
        match session_start {
            Some(start) => {
                let deadline = start + Duration::from_secs(900);
                tokio::time::sleep_until(deadline).await;
            }
            None => std::future::pending::<()>().await,
        }
    };

    tokio::select! {
        msg = ws_receiver.next() => { /* WebSocket メッセージ処理 */ }
        event = event_rx.recv() => { /* Gemini イベント処理 */ }
        _ = timeout => {
            // 15分超過 → クライアントに通知して終了
            send(ServerMessage::SessionEnded {
                reason: "max_duration_reached".into()
            }).await;
            break;
        }
    }
}
```

:::message
`std::future::pending::<()>()` は **永遠に完了しない Future** です。セッション未開始時にタイムアウト分岐が発火しないようにするために使います。セッション開始後は `sleep_until(deadline)` に切り替わり、ループを回るたびに残り時間を再計算します。
:::

## まとめ

### 技術選定・設計判断の一覧

| 判断 | 選択 | 理由 |
|------|------|------|
| ポストプロセッシング API | Gemini Text API (REST) | Live API は音声用。テキスト処理は REST が安価・シンプル |
| 要約+抽出の実行 | 直列・非同期 spawn | API レート制限回避 + クライアント非ブロック |
| 同時セッション制限 | Redis SET NX | 原子的な排他制御。TTL で異常終了時も自動解放 |
| 日次カウンター | GET → SET | 同時 1 セッション保証があるため原子性不要 |
| セッション時間制限 | tokio::select! + sleep_until | メインループに自然に統合できる |
| Redis 障害時 | フェイルオープン | レート制限よりサービス可用性を優先 |

### 学び

1. **ポストプロセッシングはユーザー体験に影響しない形で実行する** — `tokio::spawn` で非同期化し、セッション終了のレスポンスを即返す
2. **構造化出力は `responseMimeType` で強制する** — プロンプトだけに頼ると出力形式が安定しない
3. **レート制限はフェイルオープンで設計する** — 補助的な機能が主要機能を止めてはいけない
4. **既存の制約を活用して実装を簡略化する** — 「同時 1 セッション」の保証があれば日次カウンターの原子性は不要
5. **`std::future::pending()` は条件付きタイマーの実装に便利** — select! の分岐を無効化するイディオム

### 実装チェックリスト

- [ ] ポストプロセッシングは `tokio::spawn` で非同期実行しているか
- [ ] Gemini JSON モードで `responseMimeType` を指定しているか
- [ ] SET NX に TTL を付けて異常終了時のデッドロックを防いでいるか
- [ ] Redis 障害時にフェイルオープンしているか
- [ ] レート制限エラーをクライアントに適切なエラーコードで返しているか
- [ ] セッション最大時間を超過した場合にクライアントへ通知しているか
