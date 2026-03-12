---
title: "RAGで取得したユーザー記憶をホーム画面の吹き出しに反映する実装と、Reanimatedのstale closure問題"
emoji: "💬"
type: "tech"
topics: ["reactnative", "rust", "rag", "gemini", "reanimated"]
published: true
---

## はじめに

AI会話アプリを開発していると、「キャラクターがユーザーのことを覚えている感」をどう演出するかが重要になります。本記事では、既存のRAGインフラを活用してホーム画面の吹き出しフレーズをパーソナライズする機能をフルスタックで実装した際の設計判断と、React Native Reanimated特有のstale closure問題に遭遇して解決した話をまとめます。

:::message
**対象読者**
- RAGの基本概念を理解している方
- React Native + Reanimatedでアニメーションを書いたことがある方
- Rust（Axum）でAPIを書いたことがある方
:::

**この記事で得られる知見:**
- 既存RAGインフラを新機能に最小コストで再利用する設計パターン
- Reanimatedの`runOnJS`とstale closureの落とし穴と解決策
- パーソナライズ機能のフォールバック戦略

## 全体像

```
┌─────────────────────────────────────────────────────┐
│  Mobile App (React Native)                          │
│                                                     │
│  HomeScreen                                         │
│    └─ useBubblePhrases()                            │
│         ├─ onAuthStateChanged → fetch              │
│         ├─ BubblePhrasesUseCase (4h cache)         │
│         └─ fallback: default phrases               │
│                        │                            │
│  CharacterBubble       │                            │
│    └─ useRef(phrases) ←┘ ← stale closure対策       │
│    └─ Reanimated animation loop                    │
└────────────────────────┬────────────────────────────┘
                         │ GET /bubble-phrases
                         ▼
┌─────────────────────────────────────────────────────┐
│  Backend (Rust / Axum)                              │
│                                                     │
│  bubble_phrases_handler                             │
│    ├─ Firebase Auth token verification              │
│    ├─ PromptService.build_system_prompt()           │
│    │    └─ Firestore: episodes, sessions,           │
│    │       notes, learner profile                   │
│    └─ GeminiTextClient.generate_json::<Vec<String>>│
│         └─ "10個の短いフレーズを生成"               │
└─────────────────────────────────────────────────────┘
```

ユーザーがアプリを開くと、バックエンドがFirestoreからユーザーの会話履歴・エピソード記憶・ノートを取得し、それをコンテキストとしてGeminiに渡して「友達が呟くような短い日本語フレーズ」を10個生成します。

## 設計のポイント

### ポイント1: 既存RAGインフラの再利用

新しいエンドポイントのために専用のRAGパイプラインを組む必要はありません。既にチャット用に構築されていた`PromptService.build_system_prompt()`をそのまま呼ぶだけで、ユーザーのエピソード記憶・セッション要約・ノート・学習者プロファイルがすべて含まれたコンテキストが手に入ります。

| 方式 | メリット | デメリット |
|------|---------|-----------|
| **専用RAGパイプライン構築** | 吹き出し向けに最適化可能 | 開発コスト大、メンテ二重化 |
| **既存system prompt再利用** | 実装最小限、情報量十分 | 不要な情報も含まれる |

:::message
既存のRAGインフラが十分な情報を返してくれるなら、新機能のためにパイプラインを新設する必要はありません。「system promptとして組み立てられた文字列をそのまま別のLLM呼び出しに渡す」だけで、パーソナライズ機能が低コストで実現できます。
:::

### ポイント2: フロントエンドのフォールバック戦略

パーソナライズは「あれば嬉しいが、なくても致命的ではない」機能です。そのため、あらゆる失敗ケースでデフォルトフレーズにフォールバックする設計にしました。

```
未認証 ──→ APIスキップ ──→ デフォルトフレーズ
API失敗 ──→ catch ──→ デフォルトフレーズ維持
Gemini失敗 ──→ backend側で空配列 ──→ デフォルトフレーズ維持
```

## 実装Tips

### Tip 1: Axumハンドラで既存のsystem promptを再利用する

バックエンド側の実装は驚くほどシンプルです。既存の`PromptService`と`GeminiTextClient`を組み合わせるだけです。

```rust
pub async fn bubble_phrases_handler(
    State(state): State<AppState>,
    headers: HeaderMap,
) -> Result<Json<BubblePhrasesResponse>, (StatusCode, Json<Value>)> {
    let user_id = verify_token(&headers, &state.config).await?;

    // Reuse existing RAG context builder
    let system_prompt = state
        .prompt_service
        .build_system_prompt(&user_id, None)
        .await;

    let prompt = r#"Based on the user context provided in the system prompt,
generate 10 short Japanese phrases that a close friend might casually say.
These should feel personal — reference things you know about the user.
Output as a JSON array of strings."#;

    let phrases: Vec<String> = state
        .text_client
        .generate_json(prompt, &system_prompt)
        .await
        .unwrap_or_else(|e| {
            warn!(user_id = %user_id, error = %e, "Failed to generate");
            vec![] // Fallback to empty → frontend uses defaults
        });

    Ok(Json(BubblePhrasesResponse { phrases }))
}
```

:::message
`unwrap_or_else`で空配列を返すことで、Geminiのエラーがユーザー体験を壊さないようにしています。フロントエンドは空配列を受け取ったらデフォルトフレーズを維持します。
:::

### Tip 2: UseCaseでメモリキャッシュと重複排除

吹き出しフレーズは頻繁に変わる必要がないため、4時間のTTLキャッシュを設けました。また、同時に複数回フェッチが走らないよう、inflight promiseで重複排除しています。

```typescript
class BubblePhrasesUseCase {
  private cachedPhrases: string[] | null = null;
  private cachedAt = 0;
  private inflight: Promise<string[]> | null = null;

  async getPhrases(token: string): Promise<string[]> {
    const CACHE_TTL = 4 * 60 * 60 * 1000; // 4 hours

    if (this.cachedPhrases && Date.now() - this.cachedAt < CACHE_TTL) {
      return this.cachedPhrases;
    }

    // Deduplicate concurrent requests
    if (this.inflight) return this.inflight;

    this.inflight = this.api
      .getBubblePhrases(token)
      .then((phrases) => {
        if (phrases.length > 0) {
          this.cachedPhrases = phrases;
          this.cachedAt = Date.now();
        }
        return phrases;
      })
      .finally(() => { this.inflight = null; });

    return this.inflight;
  }
}
```

### Tip 3: onAuthStateChangedで認証完了を待つ

React Nativeでは、コンポーネントのマウント時に`auth.currentUser`がまだ`null`のケースがあります。`useEffect`内で直接`currentUser`を見ると、認証済みなのにフェッチがスキップされてしまいます。

```diff
  useEffect(() => {
-   const user = auth.currentUser;
-   if (!user) return; // ← 認証初期化前はnull!
+   const unsubscribe = auth.onAuthStateChanged((user) => {
+     if (!user || fetchedRef.current) return;
+     fetchedRef.current = true;
+     user.getIdToken()
+       .then((token) => useCase.getPhrases(token))
+       .then((result) => {
+         if (result.length > 0) setPhrases(result);
+       })
+       .catch(() => {});
+   });
+   return () => unsubscribe();
  }, []);
```

:::message
Firebase Authの初期化は非同期です。`auth.currentUser`はアプリ起動直後は`null`で、`onAuthStateChanged`が発火して初めてユーザーオブジェクトが取得できます。認証状態に依存する処理は必ずリスナー経由で書きましょう。
:::

## ハマりポイント: Reanimatedの`runOnJS`とstale closure

今回一番ハマったのがこの問題です。APIからパーソナライズフレーズを取得できているのに、画面にはデフォルトフレーズしか表示されませんでした。

### 症状

- `useBubblePhrases`フックはAPIから10個のフレーズを正常に取得
- `setPhrases`で状態も更新されている
- しかし`CharacterBubble`のアニメーションで表示されるのは常にデフォルトフレーズ

### 原因

問題は`CharacterBubble`内のアニメーションループにありました。

```typescript
// ❌ 問題のあるコード
function CharacterBubble({ phrases }: Props) {
  const [phrase, setPhrase] = useState(
    () => phrases[Math.floor(Math.random() * phrases.length)]
  );

  const pickNextPhrase = () => {
    setPhrase(phrases[Math.floor(Math.random() * phrases.length)]);
  };

  useEffect(() => {
    const animate = () => {
      opacity.value = withSequence(
        withTiming(1, { duration: 400 }),
        withDelay(6000,
          withTiming(0, { duration: 400 }, () => {
            runOnJS(pickNextPhrase)(); // ← ここが問題！
          })
        ),
      );
    };
    animate();
    const interval = setInterval(animate, 9000);
    return () => clearInterval(interval);
  }, [opacity, translateY]); // ← phrasesが依存配列にない
```

Reanimatedの`runOnJS`はUIスレッドからJSスレッドへのブリッジです。`useEffect`が初回マウント時に`pickNextPhrase`をキャプチャした後、**propsの`phrases`が更新されても、アニメーションコールバック内の`pickNextPhrase`は古いクロージャのまま**です。

```
マウント時:
  phrases = ["デフォルト1", "デフォルト2", ...]
  pickNextPhrase → デフォルトを参照 ✓
  useEffect → animate() → runOnJS(pickNextPhrase) をキャプチャ

API完了後:
  phrases = ["パーソナライズ1", "パーソナライズ2", ...]  ← 更新！
  pickNextPhrase → パーソナライズを参照 ✓（新しい関数）
  でもアニメーション内のrunOnJSは古いpickNextPhraseを持ったまま ✗
```

### 修正: useRefで最新のpropsを参照する

```diff
  function CharacterBubble({ phrases }: Props) {
+   const phrasesRef = useRef(phrases);
+   phrasesRef.current = phrases;

    const pickNextPhrase = () => {
-     setPhrase(phrases[Math.floor(Math.random() * phrases.length)]);
+     const current = phrasesRef.current;
+     setPhrase(current[Math.floor(Math.random() * current.length)]);
    };
```

`useRef`は毎レンダーで同一のオブジェクト参照を返すので、`pickNextPhrase`がキャプチャする`phrasesRef`は常に同じrefオブジェクトです。そしてrefの`.current`プロパティは毎レンダーで最新のpropsに更新されるため、アニメーションコールバックから呼ばれた時点で最新のフレーズ配列を参照できます。

:::message
**Reanimated + setIntervalのルール**: `runOnJS`で呼ぶ関数がpropsやstateに依存する場合、`useRef`経由で最新値を参照すること。`useEffect`の依存配列にpropsを入れてアニメーションを再セットアップするのは、ちらつきの原因になるので避けた方が良いです。
:::

## まとめ

### 技術選定・設計判断の一覧

| 判断 | 選択 | 理由 |
|------|------|------|
| RAGコンテキスト取得 | 既存`build_system_prompt`再利用 | 新規パイプライン不要、情報量十分 |
| LLMレスポンス形式 | `generate_json::<Vec<String>>` | 型安全なパース、バリデーション不要 |
| フロントキャッシュ | メモリ内4時間TTL | 頻繁な更新不要、オフライン耐性 |
| 認証待ち | `onAuthStateChanged`リスナー | `currentUser`直接参照はrace condition |
| Reanimated stale closure | `useRef`でpropsを橋渡し | 依存配列変更はアニメーション再起動を伴う |
| エラーハンドリング | 全レイヤーでデフォルトフォールバック | パーソナライズは"nice to have" |

### 学び

1. **既存インフラの再利用が最強の設計判断** — RAG用に構築したsystem promptは、そのまま別のLLMタスクのコンテキストに転用できる
2. **Reanimatedの`runOnJS`はクロージャを凍結する** — `useEffect`内で定義した関数をアニメーションコールバックに渡すと、propsの変更が反映されない
3. **`useRef`はクロージャの壁を越える唯一の橋** — 参照の同一性を利用して、古いクロージャから最新の値にアクセスできる
4. **パーソナライズ機能のエラーハンドリングは「何も起きない」が正解** — ユーザーに見えるエラーを出すより、静かにデフォルトにフォールバックする方が体験が良い

### 実装チェックリスト

- [ ] 既存のRAGインフラで再利用できる部分がないか確認したか
- [ ] LLMのエラー時にバックエンドが安全な値（空配列等）を返すか
- [ ] フロントエンドで認証状態の初期化タイミングを考慮しているか
- [ ] Reanimatedのコールバックで参照する値にstale closureの懸念がないか
- [ ] ネットワーク切断時にデフォルト表示が維持されるか
