---
title: "技術選定 — Next.jsとFlutterを選んだ理由"
---

## フレームワーク選定の判断基準

1人で5プロダクトを運用する場合、技術選定で最も重要なのは「最先端かどうか」ではありません。**学習・運用・トラブルシューティングの総コストが最小になるかどうか**です。

フレームワークの選定にあたって、以下の4つの基準を設けました。

1. **フルスタック完結性** — フロントエンドとバックエンドを1つのフレームワーク内で扱えるか
2. **エコシステムの成熟度** — 連携サービスのドキュメントが充実しているか。ハマった時にすぐ解決できるか
3. **クロスプラットフォーム対応** — 1つのコードベースでiOS/Android両方をカバーできるか（モバイルの場合）
4. **言語の親和性** — 複数言語を使う場合、切り替えコストが許容範囲に収まるか

この基準に照らして、Webは**Next.js**、モバイルは**Flutter**という選択に至りました。

## なぜNext.jsなのか

### フルスタックで完結できる

1人で5プロダクトを運用する場合、フロントエンドとバックエンドで別のフレームワークを使い分ける余裕はありません。Next.jsのApp Routerは、Server ComponentsとRoute Handlersの組み合わせで、フロントエンドからAPI、SSR/SSGまでを1つのプロジェクト内で完結できます。

```
プロダクトごとのNext.js構成:

ShareToku (Next.js 16)
├── App Router + Server Components
├── Supabase (認証・DB)
├── Recharts (データ可視化)
└── Radix UI (UIコンポーネント)

Focalize (Next.js 14)
├── App Router + Server Components
├── Supabase (認証・DB) + PostgreSQL (pg直接)
├── Stripe (決済)
├── next-intl (i18n: 日英対応)
├── next-auth (認証)
├── Google APIs (カレンダー連携)
├── nodemailer + web-push (通知)
├── Gemini AI (@google/genai)
└── Radix UI + dnd-kit (UI)

Skill Tracker LP (Next.js 15)
├── Static Export (output: "export")
├── Headless UI + Heroicons
└── FTPデプロイ (お名前.com)
```

Focalizeのように、OAuth認証・決済・メール送信・プッシュ通知・AI連携まで含む複雑なSaaSも、Next.js単体のプロジェクト構成で実現できています。別途Express.jsやNestJSのAPIサーバーを立てる必要がない。これはインフラコストと運用コストの両面で大きなメリットです。

### エコシステムの成熟度

もうひとつの決め手は**エコシステムの成熟度**です。Supabase、Stripe、Vercel、next-intlなど、Next.jsを前提に設計されたサービスやライブラリが豊富にあります。インテグレーションのドキュメントが充実しているため、新しいサービスとの連携で詰まることが少ない。

1人開発では「ハマった時にどれだけ早く解決できるか」が生産性に直結します。Stack OverflowやGitHub Issuesの情報量も、Next.jsは圧倒的に多い。マイナーなフレームワークを選んで、ドキュメントもコミュニティもない状態でデバッグに半日費やす、という事態は避けたい。

### Next.jsを選ばない方がよいケース

公平を期すために、Next.jsが適さないケースも挙げておきます。

- **完全な静的サイトのみ**: Astroの方がビルドサイズが小さく、パフォーマンスが良い
- **リアルタイム性が最重要**: WebSocketを多用するならRemixやSvelteKitの方が扱いやすい場合がある
- **サーバーサイド処理が不要**: React単体（Vite）で十分なケースもある

ジョインクラスの場合、Focalizeのようなフルスタック要件とShareTokuのようなSSR要件が混在しているため、Next.jsの「なんでもできる」汎用性が最もフィットしました。

## なぜFlutterなのか

### iOS/Android同時リリースが前提

モバイルアプリに関しては、**iOS/Android同時リリースが前提**でした。1人でSwiftとKotlinの両方を書くのは現実的ではありません。

クロスプラットフォームの選択肢としてはReact Nativeもありましたが、Flutterを選んだ理由は3つです。

**1. UIの一貫性。** Flutterはレンダリングエンジンを独自に持つため、iOS/Androidで見た目が完全に一致します。元気ボタンのようなシンプルなUIでは特に有効で、プラットフォーム間のUI差異をデバッグする時間がゼロになります。

**2. Dart言語の生産性。** TypeScriptに近い文法で、学習コストが低い。null安全も標準で備わっています。TypeScriptを日常的に書いているエンジニアなら、Dartへの移行は1〜2週間で実用レベルに到達できます。

**3. Firebaseとの親和性。** Flutterの公式プラグインでFirebase Auth、Firestore、Cloud Messaging、Analyticsまでカバーできます。元気ボタンのようなリアルタイム通知が中核のアプリでは、この統合の質が開発速度に直結しました。

```
Flutterプロダクトの構成:

元気ボタン (Flutter 3.38 + Dart 3.10)
├── Firebase (Auth, Firestore, Messaging, Analytics)
├── Riverpod + freezed (状態管理)
├── go_router (ナビゲーション)
├── mobile_scanner + qr_flutter (QRコード)
├── flutter_local_notifications (リマインダー)
└── iOS / Android 両対応

MochiQ (Flutter 3.x + Dart 3.7)
├── Firebase (Auth, Firestore, Analytics)
├── Provider (状態管理)
├── Gemini AI (google_generative_ai)
├── image_picker (カメラ撮影)
├── in_app_purchase (サブスクリプション)
├── flutter_local_notifications (復習リマインダー)
├── app_links (ディープリンク / クイズ共有)
└── iOS / Android 両対応
```

### React Nativeではなく Flutterを選んだ理由

React Nativeを選べば、TypeScript一本で統一できたのではないか。これは正当な問いです。

React Nativeを選ばなかった最大の理由は、**ネイティブUIへの依存度**です。React Nativeはプラットフォームのネイティブコンポーネントをブリッジして使うため、iOS/Androidで微妙にレンダリングが異なるケースがあります。1人開発では、プラットフォーム間の差異を調査・修正する余裕がありません。

Flutterの独自レンダリングエンジン（Skia/Impeller）は「書いたとおりに表示される」保証があり、プラットフォーム差異のデバッグ工数がほぼゼロです。この予測可能性は、1人開発において非常に価値があります。

## TypeScript と Dart の2言語体制

### コンテキストスイッチのコスト

TypeScript（Next.js）とDart（Flutter）の2言語体制には、当然ながらコンテキストスイッチのコストがあります。朝にFocalizeのTypeScriptを書いて、午後にMochiQのDartに切り替えると、最初の30分は頭の切り替えに使われます。

それでもこの構成を選んだのは、**「Web = Next.js」「モバイル = Flutter」という明確な境界線があれば、切り替えコストは許容範囲に収まる**と判断したからです。同じプラットフォーム内で複数のフレームワークを混在させる方が、はるかに認知負荷が高い。

### 2言語間の共通パターン

TypeScriptとDartは文法的に類似点が多く、以下のパターンは両言語でほぼ同じ思考で書けます。

- **非同期処理**: `async/await` は両言語で同一の構文
- **null安全**: TypeScriptの `?` とDartの `?` は同じセマンティクス
- **型推論**: 両言語とも強力な型推論を備えている
- **クラスベースOOP**: 継承、インターフェース、ミックスインの概念が共通

実質的に「方言が違う同じ言語」という感覚で書けるため、2言語であることのコストは想像より小さいです。

### UIライブラリの統一方針

コンポーネント自体はプロダクト間で共有していませんが、**設計パターンとツールチェーン**を統一しています。

```
Next.jsプロダクト共通:
  UIライブラリ: Radix UI or Headless UI (ヘッドレス)
  スタイリング: Tailwind CSS 4
  ユーティリティ: clsx + tailwind-merge (class-variance-authority)
  アイコン: Lucide React or Heroicons
  型安全: TypeScript strict mode

Flutterプロダクト共通:
  フォント: google_fonts
  通知: flutter_local_notifications
  共有: share_plus
  ストレージ: shared_preferences
```

同じヘッドレスUIライブラリとTailwind CSSを使うことで、あるプロダクトで作ったコンポーネントを別のプロダクトに「コピー&調整」で移植できます。npm packageとして共有するほどの抽象度ではないが、コピーして少し修正すれば動く。この「ゆるい共通化」が、1人開発では最もバランスが良い方法です。

## 技術選定のまとめ

| 判断基準 | Next.js (Web) | Flutter (モバイル) |
|---------|--------------|-------------------|
| フルスタック | Server Components + Route Handlers | Firebase連携でバックエンド不要 |
| エコシステム | Supabase, Stripe, Vercel等が充実 | Firebase公式プラグインが充実 |
| クロスプラットフォーム | SSR/SSG/Static Exportで柔軟 | iOS/Android完全一致 |
| 言語の学習コスト | TypeScript（広く普及） | Dart（TypeScriptからの移行が容易） |

フレームワーク選定は「一度選んだら数年間付き合う結婚のようなもの」です。最新・最先端よりも、**長期的な運用コストが最小になる選択**を意識することが、マルチプロダクト運用では特に重要です。

次の章では、この2つのフレームワークで構築した5プロダクトのアーキテクチャ全体像を見ていきます。
