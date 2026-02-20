---
title: "デプロイ戦略とCI/CD — 3種類のデプロイ先を統一管理する"
---

## デプロイ先の使い分け

5プロダクトのデプロイ先は3種類に分かれています。これは意図した設計というよりも、各プロダクトの要件とコスト制約から自然にそうなった結果です。

| プロダクト | デプロイ先 | 理由 |
|-----------|----------|------|
| Focalize | Vercel | SSR必須、Stripe webhook、i18n、standalone出力 |
| ShareToku | Vercel | SSR + Supabase SSR連携 |
| MochiQ LP | お名前.com (FTP) | 静的サイト、コスト最小化 |
| Skill Tracker LP | お名前.com (FTP) | 静的エクスポート |
| MochiQ App | App Store / Google Play | ネイティブアプリ |
| 元気ボタン | App Store / Google Play | ネイティブアプリ |

## Vercel — SSRが必要なプロダクト

### Focalizeのデプロイ構成

Focalizeは`output: 'standalone'`でビルドし、Vercelにデプロイしています。SSRが必要な理由は、ユーザーごとに異なるダッシュボードを表示するためです。next-intlによるi18n（日英対応）もサーバーサイドで処理しています。

Vercelとの連携で特に重要なのは、以下の3点です。

**環境変数の管理。** Supabaseの接続情報、Stripeのシークレットキー、Google OAuthのクレデンシャルなど、多数の環境変数をVercelのダッシュボードで管理しています。Production/Preview/Developmentで値を分けることで、本番とステージングで異なる設定を安全に扱えます。

**Stripe Webhookのエンドポイント。** StripeのWebhookはRoute Handlerとして実装しています。VercelのServerless Functionsとして自動的にデプロイされるため、別途Webhookサーバーを立てる必要がありません。

**自動プレビューデプロイ。** PRを作成すると、Vercelが自動でプレビュー環境をデプロイしてくれます。1人開発でも、本番にマージする前にプレビュー環境で動作確認できるのは安心です。

### ShareTokuのSSR構成

ShareTokuもVercelにデプロイしていますが、Focalizeほど複雑な構成ではありません。Supabaseの`@supabase/ssr`パッケージを使ったServer ComponentsでのSSRが主な用途です。

## お名前.com (FTP) — 静的サイト

### なぜVercelを使わないのか

MochiQのランディングページとSkill TrackerのLPは、`output: "export"` でビルドした完全な静的ファイルです。Vercelにデプロイすることもできますが、お名前.comの共用サーバーにFTPで配置するだけで十分です。

Vercelの無料枠には月間のBandwidth制限があります。静的サイトをVercelから外すことで、FocalizeとShareTokuのために無料枠を温存できます。年間で数百円の共用サーバー費用で静的サイトをホスティングできるため、コスト効率は圧倒的に良い。

### GitHub ActionsによるFTPデプロイの自動化

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

`main`にマージされると、自動でビルド → FTPデプロイが走ります。`lftp`の`mirror --reverse --delete`は、ローカルのビルド出力をリモートに同期し、リモートにしか存在しないファイルは削除します。これにより、デプロイのたびにクリーンな状態が保証されます。

## App Store / Google Play — モバイルアプリ

### 現在の運用

Flutterアプリのストアデプロイは、現時点ではFastlaneの設定を整備しつつ、リリースの頻度が高くないため手動ビルド+手動アップロードで運用しています。元気ボタンもMochiQも、Fastlaneのmetadata管理（ストア掲載文やスクリーンショットの管理）は既にリポジトリに含まれています。

### Fastlane metadata管理

ストアの掲載文やスクリーンショットをGitで管理しておくことは、手動デプロイであっても価値があります。「前回のストア説明文は何だったか」がリポジトリの履歴から辿れるため、A/Bテストの記録としても機能します。

```
元気ボタンのFastlane構成:

ios/fastlane/metadata/
├── ja/
│   ├── name.txt
│   ├── subtitle.txt
│   ├── description.txt
│   ├── keywords.txt
│   └── release_notes.txt
└── en-US/
    ├── name.txt
    ├── subtitle.txt
    ├── description.txt
    ├── keywords.txt
    └── release_notes.txt

android/fastlane/metadata/android/
├── ja-JP/
│   ├── full_description.txt
│   ├── short_description.txt
│   └── title.txt
└── en-US/
    ├── full_description.txt
    ├── short_description.txt
    └── title.txt
```

将来的にリリース頻度が上がれば、GitHub Actions + Fastlaneでの自動デプロイに移行する計画です。

## CI/CDの共通パターン

### 3ステージ構成

5プロダクト全てでGitHub Actionsを使っています。ワークフローの内容はプロダクトごとに異なりますが、**3ステージ構成**という設計思想は統一しています。

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
│  └── Semgrep SAST スキャン                    │ ← Focalize
└──────────────────────┬───────────────────────┘
                       ▼
┌──────────────────────────────────────────────┐
│  Stage 2: テスト                              │
│  ├── Jest + Testing Library                  │ ← Next.jsプロダクト
│  ├── Vitest                                  │ ← ShareToku
│  ├── flutter test                            │ ← Flutterプロダクト
│  └── カバレッジレポート                        │
└──────────────────────┬───────────────────────┘
                       ▼
┌──────────────────────────────────────────────┐
│  Stage 3: デプロイ (mainブランチのみ)           │
│  ├── Vercel自動デプロイ                       │ ← Focalize, ShareToku
│  └── FTPデプロイ                              │ ← MochiQ LP, Skill Tracker
└──────────────────────────────────────────────┘
```

### Semgrep SASTスキャン

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

`--config=auto`はSemgrepが自動的にプロジェクトの言語とフレームワークを検出し、適切なルールセットを適用します。`--error`フラグにより、脆弱性が検出された場合はCIが失敗してマージをブロックします。

このスキャンはFocalize固有の要件（CASA認証）から導入しましたが、セキュリティスキャン自体は他のWebプロダクトにも横展開すべきです。実際にShareTokuにも同様のスキャンを導入済みです。

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

1人開発では「自分のコードを自分でレビューする」バイアスが避けられません。AIによる自動レビューは万能ではありませんが、明らかなミスやセキュリティの見落としを検出するセーフティネットとして機能します。

特に有効だったケースとして、APIキーの誤コミット検出、未使用のimport文の指摘、エラーハンドリングの欠如の警告などがあります。

## セキュリティヘッダーの共通テンプレート

Webプロダクトで共通して適用しているセキュリティヘッダーのテンプレートです。Focalizeで最初に構築し、他のNext.jsプロダクトにも横展開しています。

```javascript
// next.config.mjs のセキュリティヘッダー
const securityHeaders = [
  {
    key: "Strict-Transport-Security",
    value: "max-age=31536000; includeSubDomains; preload"
  },
  { key: "X-Content-Type-Options", value: "nosniff" },
  { key: "X-Frame-Options", value: "DENY" },
  {
    key: "Content-Security-Policy",
    value: buildCspHeader()  // プロダクトごとにカスタマイズ
  },
  {
    key: "Permissions-Policy",
    value: "camera=(), microphone=(), geolocation=()"
  },
  { key: "Cross-Origin-Embedder-Policy", value: "credentialless" },
  { key: "Cross-Origin-Opener-Policy", value: "same-origin" },
];
```

Content-Security-Policyだけはプロダクトごとに内容が異なりますが（連携する外部サービスが違うため）、それ以外のヘッダーは全プロダクトで同一です。

## 機密情報の管理

全プロダクト共通で、APIキーや認証情報は環境変数で管理し、`.env`ファイルはgitignoreに含めています。GitHub ActionsのSecretsを活用し、CIパイプラインでも機密情報をハードコードしていません。

```
機密情報の管理ルール:

1. .envファイルは絶対にコミットしない (.gitignore必須)
2. GitHub Secrets にCI/CD用の値を登録
3. Vercelの環境変数にProduction用の値を登録
4. .env.example に必要な環境変数のキー名のみを記載（値は空）
5. 新しい環境変数を追加したら .env.example も同時に更新
```

このルールは5プロダクト全てで徹底しています。過去に`.env`ファイルの取り扱いでヒヤリとした経験があり、それ以降は`.env`関連のルールを最も厳格に運用しています。

次の章では、FocalizeでのCASA Tier 2認証取得の全プロセスを解説します。
