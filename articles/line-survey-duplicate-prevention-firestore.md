---
title: "LINE Botアンケートの重複回答を防ぐ — Firestoreアトミック操作と3層ガード設計"
emoji: "🛡️"
type: "tech"
topics: ["LINE", "Firestore", "TypeScript", "MessagingAPI", "webhook"]
published: true
---

## はじめに

LINE Messaging APIでアンケート（ボタンテンプレート）を配信すると、ユーザーが**選択肢を複数回タップ**できてしまいます。タップごとにpostbackイベントが発火し、異なる選択肢のタグがすべて付与される、回答が複数件保存されるといった問題が起きます。

さらに、回答後にユーザーが「はい。」などのテキストを手打ち送信すると、messageイベント経由でシナリオが再起動してしまうケースもあります。

この記事では、**Firestoreの`create()`によるアトミック重複防止**と、**3つのエントリポイントへのガード配置**でこれらを解決した設計と実装を解説します。

:::message
**対象読者**
- LINE Messaging API でアンケートやシナリオ配信を実装している方
- Firestore でデータの重複書き込みを防ぎたい方
- webhook の冪等性やレース条件に課題を感じている方
:::

**この記事で得られること：**
- Firestore `create()` を使ったアトミックな重複防止パターン
- SHA-256ハッシュによる決定的ドキュメントIDの設計
- レガシーデータとの後方互換を保つ移行戦略
- LINE webhookの3つのエントリポイントに対するガード設計

## 全体像

重複回答が発生する3つの経路と、それぞれに配置したガードの全体像です。

```
ユーザーのアクション           webhookイベント        ガード箇所
─────────────────────────────────────────────────────────────
① 選択肢を複数回タップ  →  postback(survey_answer) → handleSurveyAnswer()
② Flex画像を再タップ    →  postback(flex_tap)      → handleFlexImageTap()
③「はい。」を手打ち送信  →  message                 → triggerScenariosByEvent()
                                                         ↓
                                                   Firestore存在チェック
                                                   回答済み → スキップ
                                                   未回答   → 処理続行
```

## 設計のポイント

### ポイント1: なぜ `create()` を使うのか

Firestoreで重複書き込みを防ぐ方法はいくつかあります。

| 方式 | メリット | デメリット |
|------|---------|-----------|
| `set()` + 事前クエリ | 実装がシンプル | レース条件に脆弱（2つのリクエストが同時にクエリ→両方書き込み） |
| トランザクション | 強整合性 | パフォーマンスコスト、リトライロジックが必要 |
| **`create()` + 決定的ID** | **アトミック、高速** | **ドキュメントIDの設計が必要** |
| Security Rules | クライアント側で制御可能 | サーバー（Admin SDK）のみが書き込む構成では不要 |

:::message
**`create()` の特性**: 指定したドキュメントIDが既に存在する場合、Firestoreはエラーコード `6`（ALREADY_EXISTS）を返します。`set()`と違い上書きしないため、**アプリケーション側でロックやトランザクションを使わずに排他制御**ができます。
:::

### ポイント2: 決定的ドキュメントIDの設計

重複判定に使うキーは `scenarioId + nodeId + lineUserId` の組み合わせです。これをFirestoreのドキュメントIDとして使えれば、`create()`の排他制御が効きます。

ただし、単純な文字列結合ではIDの区切りが衝突する可能性があります。

```
// 危険な例: 区切り文字 "_" がID内にも存在しうる
"scenario_1" + "_" + "node_2" + "_" + "user_3"
// ↓ と区別できない
"scenario" + "_" + "1_node_2" + "_" + "user_3"
```

そこで、SHA-256ハッシュで決定的なIDを生成します。

```typescript
import { createHash } from 'crypto';

function buildDeterministicDocId(
  scenarioId: string,
  nodeId: string,
  lineUserId: string
): string {
  // NUL文字で区切ることで、各フィールドの境界が一意に決まる
  const payload = [scenarioId, nodeId, lineUserId].join('\0');
  return createHash('sha256').update(payload).digest('hex').slice(0, 40);
}
```

:::message
**なぜNUL文字（`\0`）で区切るか**: 各IDにNUL文字が含まれない前提で、`"abc" + "\0" + "def"` と `"ab" + "\0" + "cdef"` が異なるハッシュになることを保証します。scenarioId・nodeId・lineUserIdは通常の文字列なのでこの前提は安全です。ハッシュ化した16進文字列はFirestoreのドキュメントID制限にも適合します。
:::

### ポイント3: レガシーデータとの後方互換

既存のデータはランダムID（nanoid）で保存されています。新しい決定的IDに一括移行するのはリスクが高いため、**読み取り時に両方をチェック**する戦略を取ります。

```
保存フロー:
  1. レガシーデータのクエリ確認（scenarioId + nodeId + lineUserId）
  2. 見つかれば → false（既に回答済み）
  3. 見つからなければ → 決定的IDで create() を実行
  4. ALREADY_EXISTS なら → false
  5. 成功なら → true

存在チェック:
  1. 決定的IDで doc.get()（単一ドキュメント参照、高速パス）
  2. 存在すれば → true
  3. なければ → レガシークエリでフォールバック
```

## 実装Tips

### Tip 1: `saveIfNotExists` — アトミック保存パターン

「保存できたかどうか」をbooleanで返すインターフェースにすることで、呼び出し側のロジックがシンプルになります。

```typescript
// Repository interface
interface ISurveyResponseRepository {
  save(response: SurveyResponse): Promise<void>;
  saveIfNotExists(response: SurveyResponse): Promise<boolean>;
  // 特定ノードの回答存在チェック（高速パス + レガシーフォールバック）
  existsByScenarioNodeAndUser(
    scenarioId: string,
    nodeId: string,
    lineUserId: string
  ): Promise<boolean>;
  // シナリオ全体の回答存在チェック（再トリガー防止用）
  existsByScenarioAndUser(
    scenarioId: string,
    lineUserId: string
  ): Promise<boolean>;
}
```

実装側では、Firestoreの `create()` が投げるエラーコードで分岐します。

```typescript
async saveIfNotExists(response: SurveyResponse): Promise<boolean> {
  const col = this.getCollection();

  // Step 1: Check for legacy (random-ID) documents
  const legacySnap = await col
    .where('scenarioId', '==', response.scenarioId)
    .where('nodeId', '==', response.nodeId)
    .where('lineUserId', '==', response.lineUserId)
    .limit(1)
    .get();
  if (!legacySnap.empty) return false;

  // Step 2: Atomic create with deterministic ID
  const docId = buildDeterministicDocId(
    response.scenarioId,
    response.nodeId,
    response.lineUserId
  );
  try {
    await col.doc(docId).create(response);
    return true;
  } catch (err: unknown) {
    if (
      typeof err === 'object' &&
      err !== null &&
      'code' in err &&
      (err as { code: number }).code === 6
    ) {
      return false; // Already exists
    }
    throw err;
  }
}
```

:::message
**エラーコード 6 の判定**: Firestore Admin SDKのエラーオブジェクトは `code` プロパティに gRPC ステータスコードを持ちます。`6` は `ALREADY_EXISTS` に対応します。`instanceof` で特定のエラークラスを判定するより、このコード判定のほうがSDKバージョン間で安定しています。
:::

### Tip 2: 存在チェックの高速パス

`existsByScenarioNodeAndUser` は、まず決定的IDで `get()` を試みます。これはクエリより低コストな単一ドキュメント参照なので、高速です。

```typescript
async existsByScenarioNodeAndUser(
  scenarioId: string,
  nodeId: string,
  lineUserId: string
): Promise<boolean> {
  const col = this.getCollection();

  // Fast path: single document get (cheaper than query)
  const docId = buildDeterministicDocId(scenarioId, nodeId, lineUserId);
  const docSnap = await col.doc(docId).get();
  if (docSnap.exists) return true;

  // Slow path: query for legacy documents
  const snap = await col
    .where('scenarioId', '==', scenarioId)
    .where('nodeId', '==', nodeId)
    .where('lineUserId', '==', lineUserId)
    .limit(1)
    .get();
  return !snap.empty;
}
```

新しいデータが蓄積されるにつれて、高速パスのヒット率が上がり、レガシークエリの実行頻度は自然に減っていきます。

### Tip 3: 3層ガードの配置

アンケートへの重複アクセスは3つの経路から起きるため、それぞれにガードを配置します。

**① postback回答ハンドラ（最重要）**

`save()` を `saveIfNotExists()` に置き換えるだけで、タグ付与と次ノード実行を条件付きにできます。

```diff
  async handleSurveyAnswer(lineUserId, answer) {
    // ... validate scenario/node/choice ...

-   await this.upsertUser({ lineUserId, tags: [surveyTag, ...customTags] });
-   await this.surveyResponseRepo.save(response);
+   const saved = await this.surveyResponseRepo.saveIfNotExists(response);
+   if (!saved) return; // Already answered — skip tag assignment
+
+   await this.upsertUser({ lineUserId, tags: [surveyTag, ...customTags] });

    const nextNodeId = findNextNodeId(runtime, node.id, choice.value);
    if (nextNodeId) {
      await this.executeFromNode(runtime, nextNodeId, lineUserId);
    }
  }
```

:::message
**saveを先に移動する理由**: タグ付与より先に `saveIfNotExists` を実行することで、2つ目以降のタップが来ても最初の回答のタグのみが付与されることを保証します。
:::

**② Flex画像タップハンドラ**

回答済みのアンケートをFlex画像経由で再タップされた場合、シナリオの再起動をスキップします。

```diff
  async handleFlexImageTap(lineUserId, postbackData) {
    const payload = postbackData.replace('flex_tap:', '');
    if (payload.startsWith('start_survey:')) {
      const scenarioId = payload.replace('start_survey:', '');
+     const alreadyAnswered = await this.surveyResponseRepo
+       .existsByScenarioAndUser(scenarioId, lineUserId);
+     if (!alreadyAnswered) {
        // ... start scenario ...
+     }
    }
    // flex_image_tap tag is still applied regardless
    await this.upsertUser({ lineUserId, tags: ['flex_image_tap'] });
  }
```

**③ メッセージイベントトリガー**

テキスト送信でシナリオが再起動するのを防ぎます。ただし、**すべてのシナリオにチェックを入れるとパフォーマンスに影響する**ため、surveyノードを含むシナリオのみチェックします。

`hasSurveyNode` はシナリオのノード配列にsurveyタイプが含まれるかを返す簡易ヘルパーです（`runtime.nodes.some(n => n.type === 'survey')`）。

```diff
  async triggerScenariosByEvent(runtimes, eventType, lineUserId) {
    for (const runtime of runtimes) {
      const triggers = findTriggerNodes(runtime, eventType);
      if (triggers.length === 0) continue;

+     if (hasSurveyNode(runtime)) {
+       const answered = await this.surveyResponseRepo
+         .existsByScenarioAndUser(runtime.id, lineUserId);
+       if (answered) continue;
+     }

      for (const triggerNode of triggers) {
        // ... execute from next node ...
      }
    }
  }
```

:::message
**`hasSurveyNode` による条件分岐**: メッセージノードのみのシナリオ（お知らせ通知など）は重複チェック不要です。`hasSurveyNode(runtime)` でフィルタリングすることで、不要なFirestoreクエリを避けています。
:::

## ハマりポイント

### Firestore `create()` のエラー型

Firestore Admin SDK（内部的に `@google-cloud/firestore`）のエラーは `Error` 互換ですが、`instanceof` による型判定はSDKバージョン間で挙動が変わりうるため、gRPC ステータスコードの `code` プロパティで判定するのが安定します。

```typescript
// ❌ instanceof は SDK バージョンで挙動が変わりうる
catch (err) {
  if (err instanceof FirestoreError && err.code === 'already-exists') { ... }
}

// ✅ gRPC ステータスコードで判定（Admin SDK）
catch (err: unknown) {
  if (
    typeof err === 'object' &&
    err !== null &&
    'code' in err &&
    (err as { code: number }).code === 6
  ) {
    return false;
  }
  throw err;
}
```

### 管理者手動送信はガードしない

管理者が管理画面から「このシナリオを全ユーザーに送信」する機能は、意図的にガードの対象外にしています。再送したいケースがあるためです。ガードの対象はあくまで **ユーザー起因のwebhookイベント** に限定しています。

## まとめ

### 設計判断の一覧

| 判断 | 選択 | 理由 |
|------|------|------|
| 重複防止の方式 | `create()` + 決定的ID | アトミック、ロック不要、高速 |
| ドキュメントIDの生成 | SHA-256ハッシュ（NUL区切り） | 衝突回避、Firestore ID制限に適合 |
| レガシーデータ対応 | 読み取り時に両方チェック | 一括移行のリスクを回避 |
| ガード範囲 | webhook経由の3エントリポイント | 管理者手動送信は対象外 |
| チェック対象のフィルタ | `hasSurveyNode()` | 不要なDBアクセスを回避 |

### 学び

1. **Firestoreの`create()`は軽量な排他制御として優秀** — トランザクションより軽く、レース条件に強い
2. **ハッシュベースの決定的IDは移行期に有効** — 既存データと共存しつつ段階的に移行できる
3. **ガードは「入口」に置く** — 処理の途中ではなく、各エントリポイントの最初でチェックすることで、ロジックがシンプルに保てる
4. **すべてにガードを入れない** — `hasSurveyNode()` のように、チェックが必要なケースだけに絞ることでパフォーマンスを維持

### 実装チェックリスト

- [ ] `create()` のエラーハンドリングで gRPC コード 6 をキャッチしているか
- [ ] 決定的IDの生成ロジックでセパレータの衝突が起きないか
- [ ] レガシーデータ（ランダムID）のフォールバッククエリが含まれているか
- [ ] 管理者手動送信など、ガードすべきでない経路を除外しているか
- [ ] surveyノードを含まないシナリオに不要なチェックが走らないか
