---
title: "React Nativeの1200行モノリシックコンポーネントを22ファイルに分割した設計と実装"
emoji: "🧩"
type: "tech"
topics: ["reactnative", "react", "typescript", "refactoring", "expo"]
published: true
---

## はじめに

React Native（Expo）で開発中のアプリで、アカウント画面のコンポーネントが **1200行超のモノリシックファイル** に成長してしまいました。30個以上の `useState`、12個のサブスクリーン、9個のモーダル、4つの `useEffect` が1ファイルに集中し、変更のたびに全体を読み解く必要がある状態です。

この記事では、このコンポーネントを **22ファイル** に分割したリファクタリングの設計判断と実装テクニックを紹介します。

:::message
**対象読者**
- React / React Native で巨大コンポーネントに悩んでいる方
- boolean の乱立を Union Type で整理したい方
- `React.memo` が効かない問題に直面したことがある方
:::

**この記事で得られる知見：**
- boolean 乱立 → Union Type で状態を排他制御するパターン
- カスタムフックの責務分割の判断基準
- `React.memo` を実際に効かせるための `useCallback` 戦略
- `React.lazy` によるサブスクリーンの遅延読み込み

## リファクタリング前の状況

まず、どれだけ大変だったか見てみましょう。

```tsx
// AccountScreen.tsx — 1200行超のモノリシックコンポーネント
export const AccountScreen: React.FC<Props> = ({ onClose }) => {
  // 30+ の useState...
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const [showFAQScreen, setShowFAQScreen] = useState(false);
  const [showContactUsScreen, setShowContactUsScreen] = useState(false);
  const [showMessageScreen, setShowMessageScreen] = useState(false);
  const [showDebugMenu, setShowDebugMenu] = useState(false);
  const [showSignInModal, setShowSignInModal] = useState(false);
  const [showNotificationModal, setShowNotificationModal] = useState(false);
  const [showTagEditorModal, setShowTagEditorModal] = useState(false);
  // ...あと20個以上続く

  // 4つの useEffect...
  useEffect(() => { /* ユーザーデータ監視 */ }, []);
  useEffect(() => { /* LINE連携チェック */ }, []);
  useEffect(() => { /* 未読メッセージチェック */ }, []);
  useEffect(() => { /* 認証状態チェック */ }, [dep]);

  // 12個の if 文でサブスクリーン分岐...
  if (showDebugMenu) return <DebugMenuScreen onClose={() => setShowDebugMenu(false)} />;
  if (showFAQScreen) return <FAQScreen onClose={() => setShowFAQScreen(false)} />;
  if (showContactUsScreen) return <ContactUsScreen onClose={...} />;
  // ...あと9個続く

  // メイン画面（300行の IIFE パターン）
  return (
    <SafeAreaView>
      {(() => {
        // 700行のJSXがここに...
        // 9個のモーダルもここに...
      })()}
    </SafeAreaView>
  );
};
```

**問題点：**
| 問題 | 影響 |
|------|------|
| 30+ の boolean state | 排他的なはずの画面遷移が同時に `true` になりうる |
| 1ファイル 1200行 | 変更箇所の特定が困難、レビュー負荷が高い |
| IIFE パターン | JSX 内でローカル変数を宣言するためのワークアラウンド |
| useEffect が4つ混在 | どの副作用がどの状態に影響するか追いにくい |

## 全体像

リファクタリング後のファイル構成です。

```
types/
  account.ts                          # Union 型定義

hooks/account/
  useAccountUser.ts                   # ユーザーデータ listener
  useAccountAuth.ts                   # 認証状態 + signOut
  useLineLink.ts                      # LINE連携チェック
  useUnreadMessages.ts                # 未読メッセージ
  useAccountNavigation.ts             # 画面遷移管理

components/account/
  AccountScreen.tsx                   # メインコンテナ（hooks 組み立て）
  AccountHeader.tsx                   # ヘッダー
  AccountLoadingView.tsx              # ローディング
  AccountSubScreenRenderer.tsx        # サブスクリーンルーティング
  AccountModals.tsx                   # 9個のモーダル統合
  styles.ts                           # 共有スタイル
  sections/
    LineLinkSection.tsx               # LINE連携バナー
    MessagesSection.tsx               # メッセージ
    SubscriptionSection.tsx           # サブスクリプション
    SettingsSection.tsx               # 設定（11項目）
    OtherSection.tsx                  # FAQ / 利用規約等
    UserInfoSection.tsx               # ユーザー情報
    ReferralCodeSection.tsx           # 紹介コード
    DebugSection.tsx                  # デバッグ
```

```
┌─────────────────────────────────────────────┐
│              AccountScreen.tsx               │
│  ┌─────────────┐  ┌──────────────────────┐  │
│  │    Hooks     │  │     Components       │  │
│  │ ┌─────────┐  │  │ ┌────────────────┐   │  │
│  │ │ User    │  │  │ │ Header         │   │  │
│  │ │ Auth    │  │  │ │ SubScreen      │   │  │
│  │ │ LineLink│  │  │ │ Renderer       │   │  │
│  │ │ Unread  │  │  │ │ Sections (×8)  │   │  │
│  │ │ Nav     │  │  │ │ Modals         │   │  │
│  │ └─────────┘  │  │ └────────────────┘   │  │
│  └─────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────┘
```

## 設計のポイント

### ポイント1: boolean 乱立 → Union Type で排他制御

元コードでは12個のサブスクリーンを12個の boolean で管理していました。

```tsx
// ❌ Before: 排他的なはずの画面を独立した boolean で管理
const [showFAQScreen, setShowFAQScreen] = useState(false);
const [showContactUs, setShowContactUs] = useState(false);
const [showDebugMenu, setShowDebugMenu] = useState(false);
// ...あと9個
```

これだと **複数の画面が同時に `true` になりうる** という構造的な問題があります。実際にはどのサブスクリーンも排他的（一度に1つしか開かない）なので、Union Type で表現できます。

```tsx
// ✅ After: Union Type で排他的な状態を型レベルで保証
type AccountSubScreen =
  | 'none'
  | 'faq'
  | 'contactUs'
  | 'debugMenu'
  | 'messageScreen'
  | 'linkLine'
  | 'onboarding'
  | 'userNameEdit'
  | 'accountManagement'
  // ...

type AccountModalType =
  | 'none'
  | 'signIn'
  | 'notification'
  | 'tagEditor'
  | 'speech'
  | 'targetLanguage'
  // ...
```

:::message
**なぜサブスクリーンとモーダルを分けるか？**
サブスクリーンは全画面を置き換える（`if (...) return <SubScreen />`）のに対し、モーダルはメイン画面の上にオーバーレイします。ライフサイクルが異なるため、別の状態として管理します。
:::

これにより、遷移管理のフックが非常にシンプルになります。

```tsx
// hooks/account/useAccountNavigation.ts
export function useAccountNavigation() {
  const [activeSubScreen, setActiveSubScreen] = useState<AccountSubScreen>('none');
  const [activeModal, setActiveModal] = useState<AccountModalType>('none');

  const openSubScreen = useCallback((screen: AccountSubScreen) => {
    setActiveSubScreen(screen);
  }, []);

  const closeSubScreen = useCallback(() => {
    setActiveSubScreen('none');
  }, []);

  const openModal = useCallback((modal: AccountModalType) => {
    setActiveModal(modal);
  }, []);

  const closeModal = useCallback(() => {
    setActiveModal('none');
  }, []);

  return {
    activeSubScreen, activeModal,
    openSubScreen, closeSubScreen,
    openModal, closeModal,
  };
}
```

### ポイント2: ドメイン別フック分割の判断基準

1ファイルに混在していた4つの `useEffect` を、**データソースの独立性**を基準にフックを分割しました。

| Hook | 責務 | データソース |
|------|------|-------------|
| `useAccountUser` | ユーザーデータ監視 + プラン判定 | Firestore listener |
| `useAccountAuth` | 認証状態判定 + signOut | Firebase Auth |
| `useLineLink` | LINE連携チェック | Firestore (accounts) |
| `useUnreadMessages` | 未読メッセージチェック | MessageService |
| `useAccountNavigation` | 画面遷移管理 | ローカル state のみ |

```tsx
// hooks/account/useAccountUser.ts
export function useAccountUser() {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const [planStatus, setPlanStatus] = useState('');

  useEffect(() => {
    const unsubscribe = UserService.listenUser(async (userData) => {
      setUser(userData);
      setIsLoading(false);
      if (userData) {
        // Plan status calculation
        const isPremium = UserUtils.isPremiumUser(userData);
        setPlanStatus(isPremium ? 'Active' : 'Unsubscribed');
      }
    });
    return () => unsubscribe();
  }, []);

  return { user, isLoading, planStatus };
}
```

:::message
**分割の判断基準**: 「このフックを別の画面でも使い回せるか？」を自問します。`useLineLink` は LINE 連携がある画面ならどこでも使えますし、`useUnreadMessages` もバッジ表示で再利用できます。
:::

### ポイント3: サブスクリーンを switch 文 + React.lazy で整理

12個の `if` 文の連鎖は、`switch` 文に置き換えてルーティングコンポーネントに分離しました。

```tsx
// ❌ Before: if 文の連鎖
if (showDebugMenu) return <DebugMenuScreen onClose={() => setShowDebugMenu(false)} />;
if (showFAQScreen) return <FAQScreen onClose={() => setShowFAQScreen(false)} />;
if (showContactUs) return <ContactUsScreen onClose={() => setShowContactUs(false)} />;
// ...あと9個
```

```tsx
// ✅ After: switch 文 + React.lazy
const LazyDebugMenu = React.lazy(() =>
  import('@/components/DebugMenuScreen').then(m => ({ default: m.DebugMenuScreen }))
);
const LazyOnboarding = React.lazy(() =>
  import('@/components/onboarding/OnboardingFlow').then(m => ({ default: m.OnboardingFlow }))
);

export function SubScreenRenderer({ activeSubScreen, onClose }: Props) {
  switch (activeSubScreen) {
    case 'debugMenu':
      return (
        <Suspense fallback={<LoadingView />}>
          <LazyDebugMenu onClose={onClose} />
        </Suspense>
      );
    case 'faq':
      return <FAQScreen onClose={onClose} />;
    case 'contactUs':
      return <ContactUsScreen onClose={onClose} />;
    // ...
    case 'none':
      return null;
  }
}
```

:::message
**React.lazy の効果**: React Native (Metro) ではバンドルは既に読み込まれているため、Web のようなネットワーク遅延はありません。ただし、コンポーネントの初期化を遅延させることで、**アカウント画面を開いた時点ではまだ使わない重量級コンポーネントのメモリ確保を回避**できます。
:::

## 実装Tips

### Tip 1: React.memo を「本当に」効かせる useCallback 戦略

セクションコンポーネントを `React.memo` でラップしても、親から **毎レンダーで新しい関数参照** を渡していたら無意味です。

```tsx
// ❌ React.memo が効かない: 毎レンダーで新しい関数が作られる
<SettingsSection
  onOpenTextSize={() => openSubScreen('textSize')}
  onOpenWidget={() => openSubScreen('widgetColor')}
/>
```

```tsx
// ✅ React.memo が効く: useCallback で参照を安定化
const openTextSize = useCallback(() => openSubScreen('textSize'), [openSubScreen]);
const openWidget = useCallback(() => openSubScreen('widgetColor'), [openSubScreen]);

<SettingsSection
  onOpenTextSize={openTextSize}
  onOpenWidget={openWidget}
/>
```

ポイントは、`openSubScreen` 自体も `useCallback` で安定化されていること。依存チェーンが全て安定していないと意味がありません。

```tsx
// useAccountNavigation.ts 内
const openSubScreen = useCallback((screen: AccountSubScreen) => {
  setActiveSubScreen(screen);  // setState は常に安定した参照
}, []);
```

### Tip 2: 後方互換性を re-export で維持

巨大コンポーネントを分割すると、インポートしている側のファイルを全て修正する必要が出てきます。re-export で回避できます。

```tsx
// components/AccountScreen.tsx（元のファイル）
// 1200行 → 1行に
export { AccountScreen } from './account/AccountScreen';
```

これにより、`components/index.ts` の `export * from './AccountScreen'` や、他のファイルからのインポートは **一切変更不要** です。

### Tip 3: セクションコンポーネントの粒度

セクションの分割粒度は「**ScrollView 内の視覚的なまとまり**」で判断しました。

```tsx
// AccountScreen.tsx（メインコンテナ）は組み立てるだけ
<ScrollView>
  {showLineLinkBanner && <LineLinkSection onOpen={openLinkLine} />}
  <MessagesSection hasUnread={hasUnread} onOpen={openMessages} />
  {user && <SubscriptionSection user={user} planStatus={planStatus} />}
  <SettingsSection onOpenModal={openModal} />
  <OtherSection onOpenFAQ={openFAQ} onOpenContactUs={openContactUs} />
  {user?.id && <UserInfoSection user={user} />}
  <ReferralCodeSection onOpen={openReferralCode} />
  {isDev && <DebugSection onSignOut={handleSignOut} />}
</ScrollView>
```

各セクションは `React.memo` でラップし、自身に関係のある props が変わらない限り再レンダーされません。

```tsx
export const SettingsSection = React.memo(function SettingsSection({
  onOpenModal,
  onOpenTextSize,
  onOpenWidget,
}: Props) {
  return (
    <View>
      <Text>Settings</Text>
      <Card>
        <MenuItem testID="tags_button" icon="local-offer" label="Tags"
          onPress={() => { logAnalytics('tagEditor'); onOpenModal('tagEditor'); }} />
        <MenuItem testID="notification_button" icon="notifications" label="Notification"
          onPress={() => onOpenModal('notification')} />
        {/* ...11項目 */}
      </Card>
    </View>
  );
});
```

## ハマりポイント: IIFE パターンの除去

元コードには、JSX 内でローカル変数を宣言するために IIFE（即時実行関数式）が使われていました。

```tsx
// ❌ Before: IIFE パターン — 300行が return 内の関数に閉じ込められている
return (
  <SafeAreaView>
    {(() => {
      const currentUser = auth().currentUser;
      const userId = currentUser?.uid || 'N/A';
      const email = currentUser?.email;
      return (
        <>
          <Header />
          <ScrollView>
            {/* 700行のJSX... */}
          </ScrollView>
          {/* 9個のモーダル... */}
        </>
      );
    })()}
  </SafeAreaView>
);
```

```tsx
// ✅ After: ローカル変数はコンポーネント本体で宣言
const userId: string = currentUser?.uid || t('unavailable');
const userEmail = currentUser?.email ?? undefined;

return (
  <SafeAreaView>
    <AccountHeader onClose={onClose} />
    <ScrollView>
      {/* セクションコンポーネントに分割済み */}
    </ScrollView>
    <AccountModals activeModal={activeModal} onClose={closeModal} />
  </SafeAreaView>
);
```

IIFE が必要だった理由は「`return` 内で `const` を使いたかった」だけなので、コンポーネント本体に移動すれば自然に解消します。

## まとめ

### 設計判断の一覧

| 判断 | 選択 | 理由 |
|------|------|------|
| boolean 30個の管理 | Union Type 2つ | 排他状態を型レベルで保証 |
| useEffect の分割基準 | データソース別 | 独立した副作用を独立したフックに |
| サブスクリーン分岐 | switch + React.lazy | 12個の if 連鎖を型安全なルーティングに |
| セクション分割粒度 | 視覚的まとまり | ScrollView 内の各セクションが1コンポーネント |
| 既存インポートとの互換 | re-export | 変更ファイル数を最小化 |
| パフォーマンス最適化 | React.memo + useCallback | 全セクションの不要な再レンダーを防止 |

### 学び

1. **排他的な状態は boolean ではなく Union Type で表現する** — 「同時に true になりえない」という制約を型で強制できる
2. **React.memo は useCallback とセットで初めて効く** — 依存チェーン全体が安定していないと意味がない
3. **re-export は大規模リファクタリングの味方** — インポート元を変えずにファイル構成だけ変えられる
4. **IIFE パターンは設計の歪みのサイン** — JSX 内で変数宣言が必要なら、コンポーネント構成を見直すタイミング

### リファクタリングチェックリスト

- [ ] 排他的な boolean が3つ以上あれば Union Type に統合する
- [ ] 各 `useEffect` が独立したデータソースを持っているか確認する
- [ ] `React.memo` を使うなら、props の関数が全て `useCallback` で安定しているか確認する
- [ ] 元のファイルを re-export に変更し、既存インポートが壊れないか確認する
- [ ] 全ての `testID` が移動先で保持されているか grep で確認する
