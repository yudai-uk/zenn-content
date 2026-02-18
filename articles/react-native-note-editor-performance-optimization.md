---
title: "3000行超のReact Nativeコンポーネントを4段階で高速化した実践記録"
emoji: "⚡"
type: "tech"
topics: ["reactnative", "expo", "performance", "react", "typescript"]
published: true
---

## はじめに

React Native（Expo）アプリで、ノートエディタ画面の表示が体感で遅いという問題がありました。原因を調べると、1ファイル3,000行超のモノリシックコンポーネントが、マウント時にFirebase呼び出しを直列実行し、50以上のuseState、15以上のモーダルを即時インポートしていました。

この記事では、エディタのTime-to-Interactive（操作可能になるまでの時間）を改善するために行った4段階の最適化を解説します。

:::message
**対象読者**: React Native（Expo）で画面表示の遅さに悩んでいる方、大規模コンポーネントのパフォーマンス改善に興味がある方
:::

**この記事で得られる知見:**
- データ読み込みをクリティカルパスとバックグラウンドに分離する設計パターン
- `InteractionManager` で画面遷移アニメーション後に処理を遅延させる方法
- `React.lazy()` + `Suspense` によるモーダルの遅延読み込みの実践
- `React.memo` / `useMemo` の効果的な適用箇所の見極め方
- 実装時に遭遇するstale closure問題とその対策

## 全体像

最適化は以下の4段階で実施しました。上から順にインパクトが大きいです。

```
┌──────────────────────────────────────────────────────┐
│  ユーザーが「新規作成」をタップ                         │
└──────────────────┬───────────────────────────────────┘
                   ▼
┌──────────────────────────────────────────────────────┐
│  Phase 1: データ読み込みの分離                         │
│  ┌────────────────┐  ┌─────────────────────────────┐ │
│  │ Phase A (即時)  │  │ Phase B (バックグラウンド)    │ │
│  │ ノートデータ取得 │  │ ユーザー情報 / タグ一覧取得  │ │
│  │ → isLoading=false│ │ → 表示には影響しない         │ │
│  └────────────────┘  └─────────────────────────────┘ │
└──────────────────┬───────────────────────────────────┘
                   ▼
┌──────────────────────────────────────────────────────┐
│  Phase 2: 非クリティカル処理の遅延実行                  │
│  InteractionManager.runAfterInteractions で            │
│  チュートリアル / 深夜アラート / メニュー設定を遅延      │
└──────────────────┬───────────────────────────────────┘
                   ▼
┌──────────────────────────────────────────────────────┐
│  Phase 3: モーダルの遅延読み込み                        │
│  React.lazy() + Suspense で 10個のモーダルを分割       │
└──────────────────┬───────────────────────────────────┘
                   ▼
┌──────────────────────────────────────────────────────┐
│  Phase 4: 子コンポーネントのメモ化                      │
│  React.memo / useMemo で不要な再レンダリングを削減      │
└──────────────────────────────────────────────────────┘
```

## 設計のポイント

### ポイント1: データ読み込みの2段階分離

エディタ画面を開く際、もともとは以下の3つのAPI呼び出しをすべて完了してから `isLoading = false` にしていました。

| データ | 表示に必要か | 取得時間の目安 |
|--------|:----------:|:------------:|
| ノートデータ | **必須** | 200-500ms |
| ユーザー情報 | 不要（言語設定等） | 100-300ms |
| タグ一覧 | 不要（タグ選択時のみ） | 100-200ms |

:::message
**ポイント**: エディタの表示に本当に必要なデータはノートデータだけ。ユーザー情報やタグはバックグラウンドで取得しても体験に影響しない。
:::

```typescript
// components/screens/EditorScreen.tsx
useEffect(() => {
  let isCancelled = false;

  const loadData = async () => {
    try {
      // Phase A: エディタ表示に必須のデータだけ取得
      if (noteId) {
        const fetchedNote = await NotesService.fetchNote(noteId);
        if (isCancelled) return;
        setNote(fetchedNote);
      } else {
        // New note: no fetch needed
        setNote(createEmptyNote());
      }
      if (isCancelled) return;
      setIsLoading(false); // ← ここでエディタ表示

      // Phase B: 残りをバックグラウンドで並列取得
      Promise.allSettled([
        UserService.fetchUser(),
        TagsService.fetchTags(),
      ]).then(([userResult, tagsResult]) => {
        if (isCancelled) return;
        if (userResult.status === 'fulfilled' && userResult.value) {
          setUser(userResult.value);
        }
        if (tagsResult.status === 'fulfilled') {
          setTags(tagsResult.value);
        }
      });
    } catch (error) {
      if (!isCancelled) setIsLoading(false);
    }
  };

  loadData();
  return () => { isCancelled = true; };
}, [noteId]);
```

`Promise.all` ではなく **`Promise.allSettled`** を使っているのがポイントです。`Promise.all` だとタグ取得が失敗した場合にユーザー情報も失われますが、`Promise.allSettled` なら各結果を個別にハンドリングできます。

### ポイント2: InteractionManager で画面遷移を邪魔しない

React Nativeでは、画面遷移のアニメーション中にJavaScriptの重い処理が走るとフレームドロップが発生します。`InteractionManager.runAfterInteractions` を使うと、アニメーション完了後まで処理を遅延できます。

```typescript
import { InteractionManager } from 'react-native';

useEffect(() => {
  let isCancelled = false;

  // Animation completes first, then run non-critical checks
  const task = InteractionManager.runAfterInteractions(() => {
    if (isCancelled) return;
    checkTutorialStatus();
  });

  return () => {
    isCancelled = true;
    task.cancel();
  };
}, []);
```

:::message
`InteractionManager` のタスクは必ず `cancel()` で後片付けしましょう。コンポーネントがアンマウントされた後にコールバックが実行されると、setState on unmounted component の警告が出ます。
:::

## 実装Tips

### Tip 1: React.lazy() でモーダルを遅延読み込み

エディタ画面には添削モーダル、翻訳モーダル、AI質問モーダルなど15以上のモーダルがありました。これらは画面表示時には不要なので、`React.lazy()` で初回オープン時に読み込むよう変更します。

```tsx
// Before: すべてトップレベルで即時 import
import { CorrectionModal } from '../components/CorrectionModal';
import { TranslationModal } from '../components/TranslationModal';
import { ChatModal } from '../components/ChatModal';
// ... 10個以上のモーダル

// After: React.lazy() で必要時にロード
const CorrectionModal = React.lazy(() =>
  import('../components/CorrectionModal').then(m => ({ default: m.CorrectionModal }))
);
const TranslationModal = React.lazy(() =>
  import('../components/TranslationModal').then(m => ({ default: m.TranslationModal }))
);
const ChatModal = React.lazy(() =>
  import('../components/ChatModal')
);
```

JSXでは `Suspense` で囲みます。

```tsx
return (
  <View style={styles.container}>
    {/* Non-lazy components render normally */}
    <EditorContent />
    <Toolbar />

    {/* Lazy-loaded modals wrapped in Suspense */}
    <Suspense fallback={null}>
      {showCorrection && <CorrectionModal onClose={handleClose} />}
      {showTranslation && <TranslationModal onClose={handleClose} />}
      {showChat && <ChatModal onClose={handleClose} />}
    </Suspense>
  </View>
);
```

:::message
**注意**: `Suspense` は遅延読み込みされるコンポーネントだけを囲みましょう。非lazyなコンポーネントまで含めると、lazy読み込み中にそれらも非表示になってしまいます。
:::

### Tip 2: バレルimport経由の lazy は避ける

`React.lazy()` でバレルファイル（`index.ts`）経由のインポートをすると、遅延読み込みの効果が薄れます。

```typescript
// ❌ Bad: barrel import causes entire index to be loaded
const AddWordSheet = React.lazy(() =>
  import('../components').then(m => ({ default: m.AddWordSheet }))
);

// ✅ Good: direct path loads only the target module
const AddWordSheet = React.lazy(() =>
  import('../components/AddWordSheet')
);
```

バレルファイルは多数のコンポーネントを re-export しているため、1つのモーダルを読み込むつもりが全コンポーネントのパースが走ります。直接パスを指定することで、そのモーダルのコードだけが読み込まれます。

### Tip 3: React.memo の型定義

React.memo を使うとき、`React.FC` との組み合わせで型エラーが起きることがあります。

```tsx
// ❌ TS error: React.FC<Props> と React.memo の戻り値型が合わない
export const Toolbar: React.FC<ToolbarProps> = React.memo(({ items }) => {
  // ...
});

// ✅ Generic パラメータで Props を渡す
export const Toolbar = React.memo<ToolbarProps>(({ items }) => {
  // ...
});
```

### Tip 4: useMemo の依存配列を正しく設定する

ツールバーのアイテム一覧を `useMemo` でメモ化する際、依存配列に `content`（テキスト入力値）を含めてしまうと、1文字タイプするたびにツールバーが再生成されます。

```tsx
// ❌ Bad: content が変わるたびに再生成（毎タイプで実行される）
const toolbarItems = useMemo(() =>
  createToolbarItems(handleCorrect, handleTranslate, handleSpeak),
  [showTutorial, content, noteId]
);

// ✅ Good: content を除外（ハンドラー内では ref.current で最新値を取得）
const toolbarItems = useMemo(() =>
  createToolbarItems(handleCorrect, handleTranslate, handleSpeak),
  // eslint-disable-next-line react-hooks/exhaustive-deps
  [showTutorial, noteId, targetLanguage]
);
```

ハンドラー内でテキスト内容が必要な場合は、`useRef` で最新値を保持し、ハンドラー実行時に `ref.current` から取得します。これにより依存配列から `content` を外せます。

## ハマりポイント

### Stale Closure: InteractionManager 内で古い state を参照する

`InteractionManager.runAfterInteractions` のコールバック内で `useState` の値を使うと、実行時にはすでに古い値になっている可能性があります。

```tsx
// ❌ Bad: note は InteractionManager 実行時に古い可能性がある
InteractionManager.runAfterInteractions(() => {
  if (shouldUpdateDate) {
    const yesterday = new Date();
    yesterday.setDate(yesterday.getDate() - 1);
    setNote({ ...note, date: yesterday }); // note が stale!
  }
});
```

```diff
// ✅ Good: functional update で最新の state を使う
InteractionManager.runAfterInteractions(() => {
  if (shouldUpdateDate) {
    const yesterday = new Date();
    yesterday.setDate(yesterday.getDate() - 1);
-   setNote({ ...note, date: yesterday });
+   setNote(prev => prev ? { ...prev, date: yesterday } : prev);
  }
});
```

:::message
`InteractionManager`、`setTimeout`、`Alert.alert` のボタンコールバックなど、非同期的に実行されるクロージャ内では、常に **functional update** (`setState(prev => ...)`) を使いましょう。
:::

### useRef vs useState: 即座に参照したい値

チュートリアルの自動入力とユーザーの手動入力が競合するケースがありました。ユーザーが既にタイプを開始していた場合、チュートリアルのサンプルテキストで上書きしないようにするガードが必要です。

```tsx
// ❌ Bad: useState は非同期更新なので、チェック時に反映されていない
const [hasUserTyped, setHasUserTyped] = useState(false);

const handleContentChange = (text: string) => {
  setHasUserTyped(true); // ← 次の render まで反映されない
  // ...
};

// Tutorial check (別の useEffect)
if (!hasUserTyped) {
  setContent(tutorialSampleText); // hasUserTyped がまだ false の可能性
}
```

```tsx
// ✅ Good: useRef なら即座に値が反映される
const hasUserTypedRef = useRef(false);

const handleContentChange = (text: string) => {
  hasUserTypedRef.current = true; // ← 即座に反映
  // ...
};

// Tutorial check
if (!hasUserTypedRef.current) {
  setContent(tutorialSampleText); // 正しく判定できる
}
```

:::message
**使い分け**: レンダリングに影響する値は `useState`、レンダリングに関係なく即座に参照したいフラグは `useRef` を使いましょう。
:::

### useEffect クリーンアップ関数が async 関数内に埋まる

`useEffect` 内で `async` 関数を定義し、その中で `return` したクリーンアップ関数は、useEffect のクリーンアップとして認識されません。

```tsx
// ❌ Bad: async 関数の return は useEffect に渡らない
useEffect(() => {
  const loadData = async () => {
    const data = await fetchData();
    setData(data);
    const unsubscribe = listenForUpdates(setData);
    return () => unsubscribe(); // ← これは useEffect のクリーンアップにならない!
  };
  loadData();
}, []);

// ✅ Good: リスナーの管理を useEffect 直下で行う
useEffect(() => {
  let unsubscribe: (() => void) | null = null;

  const loadData = async () => {
    const data = await fetchData();
    setData(data);
    unsubscribe = listenForUpdates(setData);
  };
  loadData();

  return () => {
    if (unsubscribe) unsubscribe(); // ← useEffect のクリーンアップとして正しく機能
  };
}, []);
```

## まとめ

### 最適化手法の一覧

| Phase | 手法 | 対象 | 効果 |
|-------|------|------|------|
| 1 | データ読み込み分離 | useEffect 内の API 呼び出し | エディタ即時表示（最大効果） |
| 2 | InteractionManager | チュートリアル / アラート / メニュー設定 | 遷移アニメーションのスムーズ化 |
| 3 | React.lazy + Suspense | 10個のモーダルコンポーネント | 初期バンドルサイズ削減 |
| 4 | React.memo / useMemo | ツールバー / マイクボタン / 日付表示 | 編集中の不要な再レンダリング削減 |

### 学び

1. **表示に必須のデータだけ先に取得する**のが最もインパクトが大きい。API呼び出しの並列化よりも、そもそも何を待つかの設計が重要
2. **InteractionManager** は画面遷移のカクつきを解消する強力なツール。ただしクリーンアップ（`.cancel()`）を忘れずに
3. **React.lazy()** はバレルimport経由だと効果が薄れる。直接パスでインポートすること
4. **useMemo の依存配列**に入力値（content等）を含めると毎タイプで再計算される。ref で最新値を保持するパターンと組み合わせる
5. **stale closure** は InteractionManager / setTimeout / Alert 内で特に起きやすい。functional update (`setState(prev => ...)`) を習慣にする

### 実装チェックリスト

- [ ] `isLoading` の解除に本当にすべてのAPI完了を待つ必要があるか確認
- [ ] 画面遷移直後に走る非クリティカル処理を `InteractionManager` でラップ
- [ ] 初期表示で不要なモーダルを `React.lazy()` に変換（バレル経由を避ける）
- [ ] `Suspense` の範囲が lazy コンポーネントだけを囲んでいるか確認
- [ ] `useMemo` の依存配列に頻繁に変わる値が混入していないか確認
- [ ] 非同期コールバック内の `setState` が functional update を使っているか確認
- [ ] `useEffect` のクリーンアップで `isCancelled` フラグと `InteractionManager.cancel()` を設定
