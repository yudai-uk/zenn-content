---
title: "useReducerベースのエディタモーダルに「再編集」機能を追加する3つの実装ポイント"
emoji: "🔄"
type: "tech"
topics: ["React", "TypeScript", "useReducer", "設計パターン"]
published: true
---

## はじめに

React で「新規作成モーダル」を作ったあと、**既存データの編集**にも対応させたくなることはよくあります。

テンプレート選択 → フォーム入力 → 完了、というステップ型のエディタモーダルを `useReducer` で状態管理している場合、「再編集」対応はちょっとした工夫が必要です。

:::message
**対象読者**
- `useReducer` でモーダルの状態管理をしている方
- 「新規作成」専用モーダルを「編集」にも対応させたい方
- ステップ型 UI（選択画面 → 編集画面）を持つコンポーネントの設計に悩んでいる方
:::

この記事で得られる知見:

- `useReducer` に `INITIALIZE` アクションを追加して既存データで状態を復元する方法
- 1 つのモーダルで「新規作成」と「編集」を安全に切り替えるステート設計
- `useRef` で `useEffect` の二重初期化を防ぐテクニック

## 全体像

今回扱うのは、以下のようなステップ型エディタモーダルです。

```
┌─────────────────────────────────────────┐
│  親コンポーネント（アイテム一覧）         │
│                                         │
│  [アイテムA] [編集]  ← 既存データで開く  │
│  [アイテムB] [編集]                      │
│                                         │
│  [+ 新規追加]       ← 空で開く          │
└────────────┬────────────────────────────┘
             │ open + initialData?
             ▼
┌─────────────────────────────────────────┐
│  エディタモーダル                        │
│                                         │
│  initialData なし → ステップ1: 選択画面  │
│  initialData あり → ステップ2: 編集画面  │
│                                         │
│  useReducer で状態管理                   │
└─────────────────────────────────────────┘
```

**ポイント**: モーダルの開閉状態に加えて「どのアイテムを編集中か」を管理し、完了時に新規追加と更新を出し分けます。

## 実装ポイント 1: useReducer に INITIALIZE アクションを追加する

### 課題

`useReducer` でステップ型の状態管理をしていると、初期状態は常に「選択画面」です。既存データで編集画面を直接開く手段がありません。

```typescript
// Before: 常に選択画面から開始
const initialState: EditorState = {
  step: "select",
  formData: null,
};

type EditorAction =
  | { type: "SELECT_TEMPLATE"; templateId: string }
  | { type: "SET_FIELD"; field: string; value: string }
  | { type: "RESET" };
```

### 解決策

`INITIALIZE` アクションを追加し、任意のデータで `edit` ステップに直接遷移できるようにします。

```diff typescript
 type EditorAction =
   | { type: "SELECT_TEMPLATE"; templateId: string }
   | { type: "SET_FIELD"; field: string; value: string }
+  | { type: "INITIALIZE"; formData: EditorFormData }
   | { type: "RESET" };

 function reducer(state: EditorState, action: EditorAction): EditorState {
   switch (action.type) {
     case "SELECT_TEMPLATE": {
       const template = TEMPLATES.find((t) => t.id === action.templateId);
       if (!template) return state;
       return { step: "edit", formData: { ...template.defaultData } };
     }
+    case "INITIALIZE":
+      return { step: "edit", formData: action.formData };
     case "RESET":
       return initialState;
     // ...other cases
   }
 }
```

カスタムフックからは安定したコールバックとして公開します。

```typescript
export function useEditor() {
  const [state, dispatch] = useReducer(reducer, initialState);

  const initialize = useCallback(
    (formData: EditorFormData) => dispatch({ type: "INITIALIZE", formData }),
    []
  );
  const reset = useCallback(() => dispatch({ type: "RESET" }), []);

  return { state, initialize, reset, /* ...other callbacks */ };
}
```

:::message
**なぜ `INITIALIZE` を別アクションにするのか？**

`SELECT_TEMPLATE` はテンプレートの `defaultData` を元に状態を作りますが、`INITIALIZE` は完全な既存データをそのまま復元します。責務が異なるため、別アクションにした方が reducer のロジックがシンプルに保てます。
:::

:::message alert
**`INITIALIZE` で返す状態に注意**

reducer の状態に `errors` や `selectedTemplate` など他のフィールドがある場合、`INITIALIZE` の返り値にも含めるか、意図的に初期値にリセットするかを明確にしましょう。`{ step: "edit", formData: action.formData }` は他のフィールドを暗黙的に落とします。
:::

## 実装ポイント 2: モーダルの開閉状態に「編集対象インデックス」を持たせる

### 課題

モーダルの開閉を `boolean` で管理していると、「新規作成」と「既存アイテムの編集」を区別できません。

```typescript
// Before: 新規追加しかできない
const [showEditor, setShowEditor] = useState(false);

const handleComplete = (data: EditorFormData) => {
  // 常に新規追加される
  onChange([...items, data]);
  setShowEditor(false);
};
```

### 解決策

`boolean` を拡張し、「開いているか」と「編集中のインデックス」を 1 つのステートにまとめます。`open` フィールドを残しているのは、`editIndex: null` が「新規作成モードで開いている」と「閉じている」の両方を意味してしまうのを防ぐためです。

```diff typescript
-const [showEditor, setShowEditor] = useState(false);
+const [editorState, setEditorState] = useState<{
+  open: boolean;
+  editIndex: number | null;
+}>({ open: false, editIndex: null });
```

これにより、呼び出し側で明確にモードを指定できます。

```typescript
// New item
const handleAddNew = () => {
  setEditorState({ open: true, editIndex: null });
};

// Edit existing item
const handleEdit = (index: number) => {
  setEditorState({ open: true, editIndex: index });
};
```

完了ハンドラでは `editIndex` に応じて新規追加と更新を出し分けます。

```typescript
const handleComplete = (data: ItemData) => {
  const idx = editorState.editIndex;
  if (idx !== null && items[idx]?.type === "target_type") {
    // Update existing
    const updated = [...items];
    updated[idx] = data;
    onChange(updated);
  } else {
    // Append new
    onChange([...items, data]);
  }
  setEditorState({ open: false, editIndex: null });
};
```

:::message
**ガード条件が重要**

`editIndex` が有効でも、モーダルが開いている間に外部で `items` が変更される可能性があります。`items[idx]?.type === "target_type"` のようなガード条件を入れることで、stale なインデックスによる誤更新を防げます。
:::

:::message alert
**インデックス vs ID**

この例ではインデックスで編集対象を追跡していますが、モーダルが開いている間にリストの並び替え・挿入・削除が起こりうる場合は、アイテムの `id` で追跡する方が安全です。インデックスベースの管理は「モーダルが開いている間はリストが変更されない」という前提のもとで使いましょう。
:::

モーダルには `initialData` を条件付きで渡します。

```tsx
<EditorModal
  open={editorState.open}
  onClose={() => setEditorState({ open: false, editIndex: null })}
  onComplete={handleComplete}
  initialData={
    editorState.editIndex !== null &&
    items[editorState.editIndex]?.type === "target_type"
      ? items[editorState.editIndex]
      : undefined
  }
/>
```

## 実装ポイント 3: useRef で二重初期化を防ぐ

### 課題

モーダルコンポーネント側で `initialData` を受け取って `useEffect` で初期化する場合、`useEffect` の依存配列の扱いに注意が必要です。

```typescript
// Problem: initialData や editor が変わるたびに再初期化される
useEffect(() => {
  if (open && initialData) {
    editor.initialize(initialData);
  }
}, [open, initialData, editor]);
```

依存配列の変化で `useEffect` が複数回発火すると、ユーザーの編集途中のデータが上書きされてしまいます。また、開発環境では React の Strict Mode が effect を 2 回実行するため、この問題がより顕在化します（本番ビルドでは 1 回のみ）。

### 解決策

`useRef` で「今回のオープンセッションで初期化済みか」を追跡します。

```typescript
const initializedRef = useRef(false);
const { initialize } = editor; // Destructure for stable reference

useEffect(() => {
  if (open && !initializedRef.current) {
    initializedRef.current = true;
    if (initialData) {
      initialize(initialData);
    }
  }
  if (!open) {
    initializedRef.current = false;
  }
}, [open, initialData, initialize]);
```

この実装のポイント:

1. `open` が `true` になった**最初の 1 回だけ**初期化が走る（"1 オープンサイクル 1 初期化" パターン。モーダルが開いたまま編集対象を切り替えたい場合は、一度閉じて再オープンするか ref をリセットする必要があります）
2. `initialData` がなければ初期化をスキップ（テンプレート選択画面から開始）
3. `open` が `false` になったらフラグをリセット → 次回オープン時に再初期化可能
4. `editor` オブジェクトではなく `initialize` を個別に依存配列に入れることで、不要な再実行を防ぐ

:::message alert
**前提: `initialData` はモーダルを開くときに同期的に渡す**

このパターンは、`initialData` がモーダルオープンと同時に（同じレンダーで）確定する前提です。`initialData` を非同期で取得する場合は、`initializedRef` のロックにより後から到着したデータが反映されません。その場合は、データ取得完了後に `open` を `true` にするか、別のアプローチを検討してください。
:::

:::message
**`editor` を依存配列に入れない理由**

`useReducer` + `useCallback` で作ったカスタムフックは、返すオブジェクトの参照が毎レンダーで変わります。`editor` をそのまま依存配列に入れると、毎レンダーで effect が再実行されます。`useCallback` で安定化された個別のコールバック（`initialize`）を依存に入れるのがベストプラクティスです。
:::

## まとめ

| ポイント | 課題 | 解決策 |
|---------|------|--------|
| INITIALIZE アクション | reducer が常に初期状態から開始 | 既存データで `edit` ステップに直接遷移するアクションを追加 |
| editIndex ステート | `boolean` では新規/編集を区別できない | `{ open, editIndex }` で 1 つのステートにまとめる |
| useRef ガード | useEffect の二重初期化 | `initializedRef` で 1 セッション 1 回の初期化を保証 |

### 学び

1. **reducer に `INITIALIZE` を持たせる**のは、ステップ型 UI の「途中から開始」を実現する汎用的なパターン
2. **モーダルの開閉と編集対象を 1 つのステートにまとめる**ことで、状態の不整合を防げる
3. **`useRef` で初期化フラグを管理する**ことで、`useEffect` の再実行に対して堅牢になる
4. **依存配列にはオブジェクトではなく安定したコールバックを入れる**ことで、不要な再実行を避けられる

### 実装チェックリスト

- [ ] reducer に `INITIALIZE` アクションが追加されているか
- [ ] モーダルの完了ハンドラで `editIndex` の有効性をガード条件でチェックしているか
- [ ] `useRef` で二重初期化を防いでいるか
- [ ] 依存配列にオブジェクトではなく個別コールバックを入れているか
- [ ] モーダルを閉じる際に `editIndex` を `null` にリセットしているか
