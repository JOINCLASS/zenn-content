---
title: "Next.js×Flutterで5プロダクトを同時運用する技術戦略"
emoji: "🏗️"
type: "tech"
topics: ["Next.js", "Flutter", "マルチプロダクト", "技術戦略", "個人開発"]
published: true
published_at: 2026-03-08 08:30
---

## はじめに：1人で5プロダクトを運用するということ

合同会社ジョインクラスでは、現在5つのプロダクトを同時に運用しています。

| プロダクト | 概要 | 技術スタック |
|-----------|------|------------|
| ShareToku | サブスク紹介プログラム横断検索 | Next.js 16 + Supabase |
| MochiQ | AI学習アプリ | Flutter + Next.js（LP） |
| Focalize | 営業自動化SaaS | Next.js 14 + Supabase + Stripe |
| 元気ボタン | 高齢者見守りアプリ | Flutter + Firebase |
| Skill Tracker | スキル管理SaaS | Next.js 15（LP）+ 本体開発中 |

開発者は自分1人。CTOでありCEOであり、カスタマーサポートでもある。

「1人でそんなに回せるのか？」と聞かれることがあります。正直に言えば、回せている日もあれば、全く手が回らない日もあります。ただ、技術選定とアーキテクチャの設計次第で、1人でも5プロダクトの運用は現実的な範囲に収まります。

この記事では、Next.jsとFlutterという2つのフレームワークを軸に、5プロダクトを同時運用するための技術戦略を実例ベースで解説します。「何を選んだか」だけでなく、「なぜその選択をしたか」「どこで失敗したか」も含めて共有します。

## なぜNext.jsとFlutterなのか

### Next.jsを選んだ理由

Webアプリケーションのフレームワーク選定で最も重視したのは、**フルスタックで完結できるかどうか**です。

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

もうひとつの決め手は**エコシステムの成熟度**です。Supabase、Stripe、Vercel、next-intlなど、Next.jsを前提に設計されたサービスやライブラリが豊富にあります。インテグレーションのドキュメントが充実しているため、新しいサービスとの連携で詰まることが少ない。1人開発では「ハマった時にどれだけ早く解決できるか」が生産性に直結します。

### Flutterを選んだ理由

モバイルアプリに関しては、**iOS/Android同時リリースが前提**でした。1人でSwiftとKotlinの両方を書くのは現実的ではありません。

クロスプラットフォームの選択肢としてはReact Nativeもありましたが、Flutterを選んだ理由は3つです。

1. **UIの一貫性**: Flutterはレンダリングエンジンを独自に持つため、iOS/Androidで見た目が完全に一致する。元気ボタンのようなシンプルなUIでは特に有効
2. **Dart言語の生産性**: TypeScriptに近い文法で、学習コストが低かった。null安全も標準で備わっている
3. **Firebaseとの親和性**: Flutterの公式プラグインでFirebase Auth、Firestore、Cloud Messaging、Analyticsまでカバーできる

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

### 「2言語」という選択のトレードオフ

TypeScript（Next.js）とDart（Flutter）の2言語体制には、当然ながらコンテキストスイッチのコストがあります。朝にFocalizeのTypeScriptを書いて、午後にMochiQのDartに切り替えると、最初の30分は頭の切り替えに使われます。

それでもこの構成を選んだのは、**「Web = Next.js」「モバイル = Flutter」という明確な境界線があれば、切り替えコストは許容範囲に収まる**と判断したからです。同じプラットフォーム内で複数のフレームワークを混在させる方が、はるかに認知負荷が高い。

## 5プロダクトのアーキテクチャ全体像

テキストベースで全体像を示します。

```
┌─────────────────────────────────────────────────────────────────┐
│                     JoinClass プロダクト群                        │
├─────────────────────────┬───────────────────────────────────────┤
│    Webプロダクト (Next.js)    │    モバイルプロダクト (Flutter)       │
│                         │                                       │
│  ┌─────────────┐        │  ┌─────────────┐                      │
│  │  ShareToku  │        │  │   MochiQ    │                      │
│  │  Next.js 16 │        │  │  Flutter    │                      │
│  │  Supabase   │        │  │  Firebase   │                      │
│  └─────────────┘        │  │  Gemini AI  │                      │
│                         │  └─────────────┘                      │
│  ┌─────────────┐        │                                       │
│  │  Focalize   │        │  ┌─────────────┐                      │
│  │  Next.js 14 │        │  │ 元気ボタン   │                      │
│  │  Supabase   │        │  │  Flutter    │                      │
│  │  Stripe     │        │  │  Firebase   │                      │
│  │  Gemini AI  │        │  └─────────────┘                      │
│  └─────────────┘        │                                       │
│                         │                                       │
│  ┌─────────────┐        │                                       │
│  │Skill Tracker│        │                                       │
│  │  Next.js 15 │        │                                       │
│  │  Static LP  │        │                                       │
│  └─────────────┘        │                                       │
├─────────────────────────┴───────────────────────────────────────┤
│                         共通基盤                                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐   │
│  │ Supabase │ │ Firebase │ │ Vercel / │ │ GitHub Actions   │   │
│  │  (BaaS)  │ │  (BaaS)  │ │ お名前.com│ │ (CI/CD)         │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘   │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                        │
│  │ Stripe   │ │ Umami    │ │ Semgrep  │                        │
│  │ (決済)   │ │(Analytics)│ │ (SAST)   │                        │
│  └──────────┘ └──────────┘ └──────────┘                        │
└─────────────────────────────────────────────────────────────────┘
```

### BaaS（Backend as a Service）の使い分け

5プロダクトのバックエンドは、SupabaseとFirebaseの2つに集約しています。

**Supabase** — Webプロダクト（ShareToku、Focalize）で使用。PostgreSQLベースなので、複雑なクエリやリレーショナルデータの扱いに強い。Focalizeでは商談パイプラインの管理にRLSを活用したマルチテナント設計を採用しています。

**Firebase** — モバイルプロダクト（MochiQ、元気ボタン）で使用。Firestoreのリアルタイム同期とCloud Messagingのプッシュ通知が、モバイルアプリのユースケースに合致しています。元気ボタンでは「ボタンが押された」イベントをリアルタイムに家族へ通知する必要があり、Firebaseの設計が自然にフィットしました。

```
選定基準:

┌──────────────┬─────────────────┬─────────────────┐
│              │    Supabase     │    Firebase     │
├──────────────┼─────────────────┼─────────────────┤
│ DB           │ PostgreSQL(RDB) │ Firestore(NoSQL)│
│ リアルタイム   │ △ (Realtime可)  │ ◎ (標準機能)    │
│ Push通知     │ × (別途必要)     │ ◎ (FCM標準)     │
│ 認証         │ ◎ (Auth)        │ ◎ (Auth)        │
│ Flutter連携  │ △ (公式あり)     │ ◎ (公式推奨)    │
│ Next.js連携  │ ◎ (SSR対応)     │ ○ (Admin SDK)   │
│ 料金         │ 無料枠大きい     │ 無料枠大きい     │
├──────────────┼─────────────────┼─────────────────┤
│ 適用先       │ Webプロダクト    │ モバイルプロダクト │
└──────────────┴─────────────────┴─────────────────┘
```

この使い分けにより、「Web = Supabase」「モバイル = Firebase」という単純なルールで判断できます。新しいプロダクトを始める際も迷いが少なくなります。

## デプロイ戦略の多様性と統一

5プロダクトのデプロイ先は実は3種類に分かれています。これは意図した設計というよりも、各プロダクトの要件とコスト制約から自然にそうなった結果です。

```
デプロイ先の使い分け:

┌─────────────┬─────────────────┬─────────────────────────┐
│ プロダクト    │ デプロイ先       │ 理由                     │
├─────────────┼─────────────────┼─────────────────────────┤
│ Focalize    │ Vercel          │ SSR必須、Stripe webhook、 │
│             │                 │ i18n、standalone出力     │
├─────────────┼─────────────────┼─────────────────────────┤
│ ShareToku   │ Vercel          │ SSR + Supabase SSR連携   │
├─────────────┼─────────────────┼─────────────────────────┤
│ MochiQ LP   │ お名前.com (FTP)│ 静的サイト、コスト最小化   │
│ Skill Tracker│ お名前.com (FTP)│ 静的エクスポート          │
├─────────────┼─────────────────┼─────────────────────────┤
│ MochiQ App  │ App Store /     │ ネイティブアプリ          │
│             │ Google Play     │                         │
├─────────────┼─────────────────┼─────────────────────────┤
│ 元気ボタン   │ App Store /     │ ネイティブアプリ          │
│             │ Google Play     │                         │
└─────────────┴─────────────────┴─────────────────────────┘
```

### SSRが必要なプロダクト → Vercel

Focalizeは`output: 'standalone'`でビルドし、Vercelにデプロイしています。SSR（Server-Side Rendering）が必要な理由は、ユーザーごとに異なるダッシュボードを表示するためです。next-intlによるi18n（日英対応）もサーバーサイドで処理しています。

### 静的サイト → お名前.com（FTP）

MochiQのランディングページとSkill TrackerのLPは、`output: "export"` でビルドした完全な静的ファイルです。Vercelにデプロイする必要はなく、お名前.comの共用サーバーにFTPで配置するだけで十分です。

MochiQ LPのデプロイはGitHub Actionsで自動化しています。

```yaml
# .github/workflows/deploy.yml（MochiQ LP）
name: Deploy to Onamae.com

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run build
      - name: Deploy via FTP
        run: |
          lftp -u "$FTP_USERNAME,$FTP_PASSWORD" \
            "ftp://$FTP_SERVER" -e "
            mirror --reverse --delete --verbose \
              out/ $FTP_REMOTE_DIR/;
            bye
          "
```

`main`にマージされると、自動でビルド → FTPデプロイが走ります。Vercelの無料枠を消費せずに、静的サイトを自動デプロイできる構成です。

### モバイルアプリ → App Store / Google Play

Flutterアプリのストアデプロイは、現時点ではFastlaneの設定を整備しつつ、リリースの頻度が高くないため手動ビルド+手動アップロードで運用しています。元気ボタンもMochiQも、Fastlaneのmetadata管理（ストア掲載文やスクリーンショットの管理）は既にリポジトリに含まれています。

## CI/CDの共通化戦略

5プロダクト全てでGitHub Actionsを使っています。ワークフローの内容はプロダクトごとに異なりますが、**設計思想は統一**しています。

### 共通のCI/CDパターン

```
┌─────────────────────────────────────────────┐
│              PR作成 / main push              │
└──────────────────────┬──────────────────────┘
                       ▼
┌──────────────────────────────────────────────┐
│  Stage 1: 静的解析                            │
│  ├── TypeScript型チェック (tsc --noEmit)      │ ← Next.jsプロダクト
│  ├── Flutter analyze                         │ ← Flutterプロダクト
│  ├── ESLint / Dart analyzer                  │
│  └── Semgrep SAST スキャン                    │ ← Focalize (CASA Tier 2)
└──────────────────────┬───────────────────────┘
                       ▼
┌──────────────────────────────────────────────┐
│  Stage 2: テスト                              │
│  ├── Jest + Testing Library                  │ ← Next.jsプロダクト
│  ├── Vitest                                  │ ← ShareToku
│  ├── flutter test                            │ ← Flutterプロダクト
│  └── カバレッジレポート → Codecov              │
└──────────────────────┬───────────────────────┘
                       ▼
┌──────────────────────────────────────────────┐
│  Stage 3: デプロイ (mainブランチのみ)           │
│  ├── Vercel自動デプロイ                       │ ← Focalize, ShareToku
│  └── FTPデプロイ                              │ ← MochiQ LP, Skill Tracker
└──────────────────────────────────────────────┘
```

### Focalizeのセキュリティスキャン

FocalizeはGoogle CASA Tier 2認証を取得しており、CIパイプラインにSemgrepによるSASTスキャンを組み込んでいます。

```yaml
# security-sast.yml (Focalize)
- name: Run Semgrep SAST scan
  run: |
    semgrep scan \
      --config=auto \
      --exclude="node_modules" \
      --exclude=".next" \
      --exclude="coverage" \
      --exclude="*.test.*" \
      --error \
      .
```

このスキャンはPR作成時とmain pushの両方で実行され、脆弱性が検出された場合はCIが失敗してマージをブロックします。

### Claude Code Actionによる自動レビュー

MochiQ LPなど一部のプロダクトでは、GitHub ActionsにClaude Code Actionを統合し、PRの自動コードレビューを実施しています。

```yaml
# claude-code-review.yml
- name: Run Claude Code Review
  uses: anthropics/claude-code-action@beta
  with:
    direct_prompt: |
      このプルリクエストをレビューして、以下の観点から
      フィードバックを日本語で提供してください：
      - コード品質とベストプラクティス
      - 潜在的なバグや問題
      - パフォーマンスの考慮事項
      - セキュリティ上の懸念
```

1人開発では「自分のコードを自分でレビューする」バイアスが避けられません。AIによる自動レビューは、見落としの検出に効果があります。

## 共通UIコンポーネントの方針

5プロダクトのUIコンポーネントは、**共有ライブラリとして切り出していません**。これは意図的な判断です。

### 共有しない理由

1. **プロダクトごとにデザインが異なる**: FocalizeはSaaS的なダッシュボードUI、元気ボタンは高齢者向けのシンプルなUI。共通化する意味がない
2. **依存関係の連鎖リスク**: 共有ライブラリを更新した際に5プロダクト全てへの影響を確認する工数は、1人では対応できない
3. **Next.jsのバージョンが異なる**: Focalizeは14系、ShareTokuは16系、Skill Trackerは15系。Reactのバージョンも18と19が混在している

### 代わりに統一しているもの

コンポーネント自体は共有せず、**設計パターンとツールチェーン**を統一しています。

```
共通パターン:

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

同じヘッドレスUIライブラリとTailwind CSSを使うことで、あるプロダクトで作ったコンポーネントを別のプロダクトに「コピー&調整」で移植できます。npm packageとして共有するほどの抽象度ではないが、コピーして少し修正すれば動く。この「ゆるい共通化」が、1人開発では最もバランスが良いと感じています。

## 1人で5プロダクトを回すための工夫

### 工夫1：AI活用の徹底

AIは「便利なツール」ではなく、**事実上のチームメンバー**です。

**コーディング**: Claude Codeを開発の中核に据え、機能実装からテスト作成、リファクタリングまで一貫して活用しています。特に、新規機能の実装では「仕様を自然言語で記述 → AIがコード生成 → 自分がレビュー・修正」というフローが定着しています。

**コードレビュー**: 前述のClaude Code ActionによるPR自動レビューに加え、ローカルでもClaude Codeに既存コードの品質チェックを依頼しています。

**プロダクト管理**: AI-CEO Frameworkという自作の経営支援フレームワークを構築し、5プロダクトの状態管理・優先順位判断・意思決定ログをAIエージェント群に委任しています。6部門（開発、マーケ、営業、財務、CS、法務）に対応するAIサブエージェントがファイルベースで情報を管理し、朝のダイジェスト生成や承認キューの管理を自動化しています。

```
AI活用のレイヤー:

┌──────────────────────────────────────┐
│  経営判断: AI-CEO Framework          │
│  (プロダクト横断の優先順位管理)        │
├──────────────────────────────────────┤
│  開発: Claude Code                   │
│  (実装、テスト、リファクタリング)      │
├──────────────────────────────────────┤
│  品質保証: Claude Code Action        │
│  (PR自動レビュー)                    │
├──────────────────────────────────────┤
│  プロダクト機能: Gemini AI            │
│  (Focalize: メール解析、MochiQ: クイズ生成) │
└──────────────────────────────────────┘
```

### 工夫2：「今週はこのプロダクト」というバッチ処理

5プロダクトを毎日均等に触るのは非効率です。コンテキストスイッチのコストが大きすぎます。

代わりに、**週単位でフォーカスするプロダクトを決める**運用にしています。

```
月間スケジュールの例:

Week 1: Focalize (主要SaaS。機能追加・バグ修正)
Week 2: MochiQ (アプリアップデート + LP改善)
Week 3: ShareToku + 元気ボタン (メンテナンス + 小規模改善)
Week 4: Skill Tracker + 全体横断 (LP改善 + インフラ更新)
```

ただし、バグ報告やエスカレーションがあればこのスケジュールは崩れます。その場合でも「今週のメインはFocalize。元気ボタンのバグ修正は2時間以内で終わらせる」と制約を設けることで、フォーカスの崩壊を最小限に抑えています。

### 工夫3：BaaSへの依存を意図的に選ぶ

SupabaseとFirebaseに認証・データベース・ストレージを丸投げしているのは、**自前でインフラを管理する時間がないから**です。

「ベンダーロックインのリスクは？」という懸念は当然あります。特にFirestoreはNoSQLなので、データの移行コストは高い。それでもBaaSを選ぶ理由は明確で、**インフラ管理に費やす1時間があれば、プロダクト開発に1時間使った方がユーザー価値が高い**からです。

SupabaseはPostgreSQL互換なので、最悪の場合はデータを別のPostgreSQLに移行できます。Firebaseはロックイン度が高いですが、モバイルアプリのリアルタイム同期+プッシュ通知をFirebase以外で同じ品質・コストで実現する代替手段が現時点では見当たりません。

## 技術選定で失敗したこと

成功事例だけでなく、失敗も共有します。

### 失敗1：Next.jsのバージョンが揃わなくなった

FocalizeはNext.js 14で構築し、ShareTokuは後からNext.js 16で始めました。Skill Tracker LPは15系です。結果として、3つのプロダクトで3つの異なるNext.jsバージョンが共存しています。

Reactのバージョンも、Focalizeは18系、ShareTokuは19系という状態です。これにより以下の問題が発生しています。

- **知識の混乱**: App Routerの細かい挙動がバージョンで異なる。FocalizeではServer Actionsの書き方がShareTokuと微妙に違う
- **依存関係の不一致**: あるライブラリがReact 19を要求し、別のプロダクトではReact 18しか使えない
- **アップグレードの負債**: Focalizeを14→15→16に上げる作業が後回しになり続けている

**学び**: 新しいプロダクトを始める際は、既存プロダクトとメジャーバージョンを揃えるか、既存プロダクトのアップグレードを先に済ませてから着手すべきでした。

### 失敗2：状態管理ライブラリがプロダクトごとに違う

FlutterプロダクトでMochiQはProvider、元気ボタンはRiverpodを使っています。元気ボタンを後から開発した際に「Riverpodの方が型安全でテスタビリティが高い」と判断して切り替えたのですが、結果として2つの状態管理パターンが共存することになりました。

コードの移植が難しくなり、片方で学んだパターンが直接活かせないという非効率が生まれています。

**学び**: クロスプラットフォームプロダクトの設計パターンは早い段階で標準を決め、新規プロダクトでも踏襲すべきです。「より良い選択肢がある」としても、統一の価値は過小評価しない方がいい。

### 失敗3：モニタリングの後回し

初期段階ではプロダクトの機能開発を優先し、モニタリングを後回しにしました。FocalizeでUmamiによるアクセス解析を導入するまで、ユーザーの行動をほとんど把握できていませんでした。

Flutterアプリでは、Firebase Analyticsを後からMochiQに追加し、元気ボタンにも同様に導入しましたが、初期のユーザー行動データは完全に失われています。

**学び**: アナリティクスは機能開発と同じ優先度で初日から入れるべきです。最小限でもページビューとユーザー数だけは計測しておく。データがなければ改善の判断ができません。

## セキュリティ対策のプロダクト横断方針

5プロダクトに共通するセキュリティ方針を設けています。

### Webプロダクト共通

Focalizeでは、next.config.mjsに包括的なセキュリティヘッダーを設定しています。この設定は他のNext.jsプロダクトにも横展開しています。

```javascript
// セキュリティヘッダーの主要項目
const securityHeaders = [
  { key: "Strict-Transport-Security",
    value: "max-age=31536000; includeSubDomains; preload" },
  { key: "X-Content-Type-Options", value: "nosniff" },
  { key: "X-Frame-Options", value: "DENY" },
  { key: "Content-Security-Policy", value: buildCspHeader() },
  { key: "Permissions-Policy",
    value: "camera=(), microphone=(), geolocation=()" },
  { key: "Cross-Origin-Embedder-Policy", value: "credentialless" },
  { key: "Cross-Origin-Opener-Policy", value: "same-origin" },
];
```

### 認証のBaaS依存

認証はSupabase AuthとFirebase Authに完全に依存しています。自前でパスワードのハッシュ化やセッション管理を実装する余裕はなく、かつBaaSプロバイダの方が自分よりもセキュリティの専門性が高い。これは合理的なトレードオフです。

### 機密情報の管理

全プロダクト共通で、APIキーや認証情報は環境変数で管理し、`.env`ファイルはgitignoreに含めています。GitHub ActionsのSecretsを活用し、CIパイプラインでも機密情報をハードコードしていません。

## まとめ：マルチプロダクト運用の原則

1人で5プロダクトを同時運用する中で見えてきた原則を整理します。

### 技術選定の原則

1. **フレームワークは最小限に**: Web = Next.js、モバイル = Flutter。これ以上増やさない
2. **BaaSを活用し、インフラ管理を手放す**: SupabaseとFirebaseで認証・DB・通知をカバー
3. **統一すべきは「パターン」であり「コード」ではない**: 共有ライブラリより設計思想の統一が実用的

### 運用の原則

4. **週単位でフォーカスするプロダクトを決める**: 毎日全部を触ろうとしない
5. **AIをチームメンバーとして扱う**: コーディング、レビュー、経営判断の全レイヤーで活用
6. **CI/CDは初日から**: 手動デプロイは事故の元。GitHub Actionsで自動化

### 品質の原則

7. **セキュリティヘッダーとSASTスキャンは全プロダクト共通**: 後から入れるのではなく最初から
8. **モニタリングも初日から**: アナリティクスなしの期間はデータ的に死んでいる
9. **バージョンは可能な限り揃える**: 揃わなくなるとアップグレード負債が指数関数的に増加する

完璧な運用はできていません。アップグレード負債は溜まるし、テストカバレッジが足りないプロダクトもあります。それでも、上記の原則に沿って設計したことで、1人でも5プロダクトが「止まらずに動いている」状態を維持できています。

重要なのは、全てを完璧にすることではなく、**限られたリソースの中で最も価値のある部分に集中する判断を繰り返すこと**です。技術選定はその判断の起点であり、一度の選択がその後の運用コストを大きく左右します。

この記事が、複数プロダクトの運用を検討している方の参考になれば幸いです。

https://joinclass.co.jp
