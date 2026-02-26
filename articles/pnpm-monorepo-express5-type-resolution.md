---
title: "pnpm モノレポ移行で Express 5 の型エラー ― @types/express-serve-static-core 解決差異"
emoji: "📦"
type: "tech"
topics: ["pnpm", "monorepo", "express", "typescript", "npm"]
published: true
---

## はじめに

既存の Express 5 + TypeScript プロジェクトを pnpm ワークスペースのモノレポに移行したところ、元リポジトリでは通っていた `tsc` が数百件の型エラーで失敗しました。

ソースコードは一切変更していません。原因は **lockfile を引き継がずに移行したことで、間接依存の型定義パッケージが破壊的な型変更を含むバージョンに更新された** ことでした。

:::message
**対象読者**: npm で管理していたプロジェクトを pnpm ワークスペース（モノレポ）に移行しようとしている方。特に Express 5 を使っている場合に該当しやすい内容です。
:::

この記事で得られる知見：

- lockfile を引き継がない移行で型エラーが出る根本原因
- `@types/express-serve-static-core` 5.0.x → 5.1.x の破壊的型変更の内容
- 間接依存のバージョンを制御する方法（ピン止め / pnpm overrides）

## 移行の全体像

```
移行前                          移行後
┌──────────────────┐           ┌──────────────────────────────┐
│ my-backend/      │           │ my-org/ (monorepo)           │
│  package.json    │  ──────>  │  services/my-backend/        │
│  package-lock    │  git      │    package.json              │
│  src/            │  archive  │    src/                      │
│  ...             │           │  pnpm-workspace.yaml         │
└──────────────────┘           │  pnpm-lock.yaml              │
    npm install                └──────────────────────────────┘
                                   pnpm install
```

`git archive` でファイルをコピーし、`package-lock.json` は削除（pnpm が `pnpm-lock.yaml` を生成するため）。`package.json` の `name` をスコープ付きに変更し、`pnpm install` → `pnpm build` を実行する、という流れです。

## 問題: tsc が数百件の型エラーで失敗する

### 症状

`pnpm install` は正常に完了。しかし `tsc` を実行すると以下のエラーが大量に発生しました。

```
src/controllers/coachingController.ts(404,12): error TS2345:
  Argument of type 'string | string[]' is not assignable
  to parameter of type 'string'.
    Type 'string[]' is not assignable to type 'string'.
```

`req.params` や `req.query` の値を `string` として扱っている箇所すべてでエラーになります。元のリポジトリでは `npm run build` が問題なく通るのに、モノレポ側では失敗する状態です。

### 原因の調査

ソースコードは同一なので、依存パッケージのバージョンが異なるはず。比較してみると：

| パッケージ | npm（元リポ） | pnpm（モノレポ） |
|-----------|:------------:|:---------------:|
| `express` | 5.1.0 | 5.2.1 |
| `@types/express` | 5.0.3 | 5.0.6 |
| `@types/express-serve-static-core` | **5.0.7** | **5.1.1** |

`package.json` には `"express": "^5.1.0"` のようにキャレット範囲で指定していました。元リポジトリでは `package-lock.json` がバージョンを固定していましたが、モノレポ移行時に lockfile は引き継がず `pnpm install` で新規生成したため、semver 範囲内の最新バージョンが解決されました。

:::message
これは npm vs pnpm の問題ではなく、**lockfile を引き継がなかったこと** が根本原因です。npm でも lockfile なしで `npm install` すれば同様に新しいバージョンが解決されます。
:::

### 根本原因: @types/express-serve-static-core の破壊的型変更

`@types/express-serve-static-core` の `5.0.x` → `5.1.x` で、`ParamsDictionary`（`req.params` の型）の定義が変わっています。

```diff
  // @types/express-serve-static-core/index.d.ts
  export interface ParamsDictionary {
-   [key: string]: string;           // 5.0.7
+   [key: string]: string | string[];  // 5.1.1 — ワイルドカードルート対応
+   [key: number]: string;
  }
```

5.1.x では Express 5 のワイルドカードルート（`/user/*id`）でパラメータが配列になるケースに対応するため、`ParamsDictionary` の値型に `string[]` が追加されました。この結果、`req.params.id` の型が `string` から `string | string[]` に広がり、`string` を期待する関数への引数でコンパイルエラーが発生します。

:::message
`@types/express@5.0.3` は `@types/express-serve-static-core@^5.0.0` に依存しています。lockfile で `5.0.7` に固定されていたものが、新規インストールで `^5.0.0` を満たす最新の `5.1.1` に解決されました。semver 上は互換（minor update）でも、型定義の変更はビルドを壊し得ます。
:::

## 修正: 依存バージョンのピン止め

Phase 1（モノレポ移行）ではソースコードを変更しない方針のため、元の npm lockfile と同じバージョンにピン止めしました。

```diff:services/my-backend/package.json
 {
   "dependencies": {
-    "express": "^5.1.0",
+    "express": "5.1.0",
   },
   "devDependencies": {
-    "@types/express": "^5.0.3",
+    "@types/express": "5.0.3",
+    "@types/express-serve-static-core": "5.0.7",
   }
 }
```

ポイントは3つです。

1. **`express` 本体もピン止め** — 元リポジトリで実績のある組み合わせを固定する
2. **`@types/express` のキャレットを外す** — 間接依存の解決範囲を制限
3. **`@types/express-serve-static-core` を明示的に追加** — 元は `@types/express` の間接依存だが、直接指定することでバージョンを確実に制御できる

:::message
**代替手段**: pnpm では `package.json` の `pnpm.overrides` で間接依存のバージョンを制御することもできます。

```json
{
  "pnpm": {
    "overrides": {
      "@types/express-serve-static-core": "5.0.7"
    }
  }
}
```

ただしモノレポルートの `package.json` に書く必要があり、他のワークスペースにも影響する点に注意してください。
:::

```bash
pnpm install
pnpm --filter @my-org/my-backend build  # tsc 成功
```

## 代替案: ソースコードを修正する

将来的には、Express 5 の正しい型に合わせてソースコードを修正するのが望ましいです。

```typescript
// Before: string として直接使用（5.0.x では動く）
const userId = req.params.userId;
someFunction(userId); // Error: string | string[] は string に代入不可

// After: 型ガードで安全に取り出す
const raw = req.params.userId;
const userId = typeof raw === "string" ? raw : raw?.[0];
if (userId) {
  someFunction(userId);
}
```

ただし、移行フェーズでは「まず動かす」ことを優先し、型修正は後続タスクにするのが現実的です。

## まとめ

### 問題と対応の一覧

| 問題 | 原因 | 対応 |
|------|------|------|
| tsc で `string \| string[]` エラー | `@types/express-serve-static-core` が 5.0.x → 5.1.x に更新された | バージョンをピン止め |
| 元リポジトリでは出ないエラー | lockfile を引き継がず新規解決したため間接依存が更新された | 間接依存も含めて明示的に指定 or `pnpm.overrides` |

### 学び

1. **lockfile を引き継がない移行では、型定義パッケージのバージョンを必ず確認する**。semver 互換でも型の破壊的変更はあり得る
2. **間接依存のバージョンを制御するには、直接 `devDependencies` に追加してピン止めするか、`pnpm.overrides` を使う**
3. **移行フェーズではソースコード変更を最小限にし、設定で解決する**。ロジック修正と環境移行を同時にやると問題の切り分けが困難になる

### 移行チェックリスト

- [ ] 元リポジトリの lockfile から主要パッケージの実バージョンを確認
- [ ] 型定義パッケージ（`@types/*`）のバージョンを比較
- [ ] 間接依存（`@types/express-serve-static-core` 等）もチェック
- [ ] `pnpm install` → `tsc --noEmit` でビルド検証
- [ ] CI ワークフローで同じビルドが通ることを確認
