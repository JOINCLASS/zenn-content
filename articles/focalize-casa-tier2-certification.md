---
title: "CASA Tier 2認証を取得するまでの全記録 — SaaS開発者向けセキュリティガイド"
emoji: "🔒"
type: "tech"
topics: ["セキュリティ", "CASA", "SaaS", "Google", "認証"]
published: false
published_at: 2026-03-05 08:30
---

## はじめに：Google連携アプリに求められる「信頼の証明」

SaaSプロダクトを開発していて、Google OAuth連携を実装した経験がある方は多いと思います。GmailやGoogleカレンダーのデータにアクセスするアプリケーションを公開するには、Googleの審査（OAuth App Verification）を通過する必要があります。

この審査プロセスの中で、アプリケーションのセキュリティレベルに応じて要求されるのが**CASA（Cloud Application Security Assessment）認証**です。

筆者は営業自動化SaaS「[Focalize](https://focalize.jp)」の開発・運営をしています。FocalizeはGmail連携によるメール送受信、Googleカレンダー連携による日程調整自動化を提供しており、Google OAuthでsensitiveスコープのデータを扱います。そのため、CASA Tier 2認証の取得が必要でした。

この記事では、CASA認証とは何か、Tier 1〜3の違い、そしてTier 2を実際に取得するまでの準備・申請・取得の全プロセスを、開発者の視点から詳細に記録します。

## CASA認証とは何か

CASA（Cloud Application Security Assessment）は、**App Defense Alliance（ADA）**が運営するクラウドアプリケーション向けのセキュリティ評価プログラムです。App Defense AllianceはGoogleが共同設立した組織で、モバイルアプリやクラウドアプリのセキュリティ基準を策定しています。

CASAの目的は、Google APIにアクセスするサードパーティアプリケーションが、一定のセキュリティ基準を満たしていることを検証することです。

具体的には、**OWASP ASVS（Application Security Verification Standard）4.0**をベースにしたセキュリティ要件への準拠が求められます。OWASP ASVSは、Webアプリケーションのセキュリティを体系的に評価するための国際標準です。

### なぜCASA認証が重要なのか

Google連携アプリを開発している場合、CASA認証の重要性は3つの側面から理解できます。

**1. Google OAuth審査の要件**

GoogleのOAuth App Verificationでは、アプリがアクセスするスコープ（権限）のレベルに応じて、CASAの取得が求められます。sensitiveスコープやrestrictedスコープを使用するアプリは、CASAなしでは審査を通過できません。

**2. ユーザーからの信頼**

「CASA Tier 2認証取得済み」という事実は、ユーザーに対して「このアプリは第三者機関によるセキュリティ検証を受けている」というシグナルになります。特にBtoBのSaaSでは、導入時のセキュリティチェックで認証の有無を問われることがあります。

**3. セキュリティ体制の客観的な証明**

自社で「セキュリティには気を配っています」と言うのと、第三者機関の評価を受けて認証を取得しているのとでは、説得力が根本的に異なります。CASA認証は、自社のセキュリティ体制を客観的に証明する手段です。

## Tier 1・Tier 2・Tier 3の違い

CASAには3つのTierがあり、アプリケーションのリスクレベルに応じて要求されるTierが決まります。

| 項目 | Tier 1 | Tier 2 | Tier 3 |
|------|--------|--------|--------|
| 対象 | リスクが低いアプリ | 中程度のリスクのアプリ | 高リスクのアプリ |
| 評価方法 | 自己評価（Self-Assessment） | 認定ラボによる評価 | 認定ラボによる詳細評価 |
| スコープ例 | 基本的なプロフィール情報のみ | sensitiveスコープ | restrictedスコープ |
| 検証の厳密さ | アンケート回答 | 自動スキャン + レビュー | 手動ペネトレーションテスト含む |
| 期間目安 | 数日 | 2〜6週間 | 4〜12週間 |
| 費用 | 無料 | 有料（ラボによる） | 高額（ラボによる） |

### どのTierが必要かの判断

GoogleのOAuth審査プロセスで、アプリがアクセスするスコープに基づいてTierが指定されます。一般的な目安は以下のとおりです。

- **Tier 1**: `openid`、`profile`、`email` など、基本的なユーザー情報のみ
- **Tier 2**: Gmail送信（`gmail.send`）、カレンダー読み書き（`calendar.events`）など、sensitiveスコープ
- **Tier 3**: Gmail全アクセス（`gmail.readonly`）、Drive全アクセスなど、restrictedスコープ

Focalizeの場合、Gmailでのメール送信とGoogleカレンダーとの連携にsensitiveスコープを使用しているため、**Tier 2**が要求されました。

## Tier 2取得に必要な準備

ここからが実践的な内容です。Tier 2認証を取得するために事前に準備すべきことを、Focalizeでの実体験に基づいて解説します。

### 1. OWASP ASVS 4.0の要件を理解する

CASA Tier 2の評価基準は、OWASP ASVS 4.0のLevel 1をベースにしています。すべての項目が対象になるわけではなく、CASAが定めたサブセット（CWEベースの要件リスト）が評価対象です。

主要な評価カテゴリは以下のとおりです。

| カテゴリ | 概要 | 代表的なチェック項目 |
|---------|------|-------------------|
| 認証（Authentication） | ユーザー認証の安全性 | OAuth 2.0の適切な実装、トークン管理 |
| セッション管理 | セッションの安全な管理 | セッションタイムアウト、トークンの無効化 |
| アクセス制御 | 権限管理の適切さ | 認可チェック、権限昇格の防止 |
| 入力検証 | 入力データの検証 | XSS対策、SQLインジェクション対策 |
| 暗号化 | データの暗号化 | TLS 1.2以上の使用、保存データの暗号化 |
| エラー処理 | エラー情報の管理 | スタックトレースの非公開、適切なログ |
| データ保護 | 個人情報の保護 | 機密データの最小化、安全な廃棄 |

### 2. セキュリティスキャンの実施

Tier 2では、認定ラボ（Authorized Lab）がアプリケーションのセキュリティスキャンを実施します。事前に自社でもスキャンを行い、脆弱性を洗い出しておくことが重要です。

Focalizeで実際に使用したツールと対策を紹介します。

#### 静的アプリケーションセキュリティテスト（SAST）

コードレベルの脆弱性を検出するために、以下を実施しました。

```bash
# 依存パッケージの脆弱性チェック
npm audit

# ESLintのセキュリティプラグインによる静的解析
npx eslint --ext .ts,.tsx src/ --rule 'no-eval: error'
```

`npm audit` で検出された脆弱性は、すべて修正またはパッチ適用を行います。`high` や `critical` の脆弱性が残っている状態では、CASA評価をパスできません。

#### 動的アプリケーションセキュリティテスト（DAST）

実行中のアプリケーションに対する外部からのスキャンです。

```bash
# OWASP ZAP（Zed Attack Proxy）によるスキャン
# Docker経由で実行
docker run -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
  -t https://your-app-url.com
```

OWASP ZAPは無料で使えるDASTツールで、CASA評価で実際にラボが使用するツールの一つでもあります。事前に自社で実行し、検出された警告をすべて対処しておくことで、ラボ評価をスムーズに通過できます。

### 3. OAuth実装のベストプラクティス

Google OAuth連携の実装品質は、CASA評価で重点的にチェックされるポイントです。以下の項目を確認しましょう。

#### スコープの最小化

リクエストするスコープは、アプリケーションの機能に必要な最小限に絞ります。

```typescript
// 良い例：必要なスコープのみを指定
const SCOPES = [
  'https://www.googleapis.com/auth/gmail.send',        // メール送信のみ
  'https://www.googleapis.com/auth/calendar.events',    // カレンダーイベントの読み書き
  'https://www.googleapis.com/auth/userinfo.email',     // メールアドレスの取得
  'https://www.googleapis.com/auth/userinfo.profile',   // プロフィール情報の取得
];

// 悪い例：不要な広範なスコープを含む
const SCOPES = [
  'https://mail.google.com/',                           // Gmailの全アクセス権（過剰）
  'https://www.googleapis.com/auth/calendar',           // カレンダーの全アクセス権（過剰）
];
```

スコープが広いほどCASAの要求Tierが上がります。`gmail.send` で済むなら `mail.google.com` は使わない。これはセキュリティの原則であると同時に、認証取得のコストを下げる実務的な判断でもあります。

#### トークンの安全な管理

アクセストークンとリフレッシュトークンの管理は、CASA評価の重要項目です。

```typescript
// トークンの保存：暗号化して保存する
// AES-256-GCMによる暗号化の例
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto';

const ALGORITHM = 'aes-256-gcm';

function encryptToken(token: string, key: Buffer): string {
  const iv = randomBytes(16);
  const cipher = createCipheriv(ALGORITHM, key, iv);
  let encrypted = cipher.update(token, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  const authTag = cipher.getAuthTag();
  // IV + AuthTag + 暗号文を結合して保存
  return `${iv.toString('hex')}:${authTag.toString('hex')}:${encrypted}`;
}

function decryptToken(encryptedData: string, key: Buffer): string {
  const [ivHex, authTagHex, encrypted] = encryptedData.split(':');
  const iv = Buffer.from(ivHex, 'hex');
  const authTag = Buffer.from(authTagHex, 'hex');
  const decipher = createDecipheriv(ALGORITHM, key, iv);
  decipher.setAuthTag(authTag);
  let decrypted = decipher.update(encrypted, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  return decrypted;
}
```

- アクセストークンは暗号化してデータベースに保存する
- リフレッシュトークンも同様に暗号化する
- トークンの有効期限を適切に管理し、期限切れトークンを自動更新する
- ユーザーがアカウントを削除した際、トークンも確実に削除する

#### PKCE（Proof Key for Code Exchange）の実装

OAuth 2.0のAuthorization Code Flowでは、PKCEの使用が推奨されています。

```typescript
import { randomBytes, createHash } from 'crypto';

// code_verifier の生成
function generateCodeVerifier(): string {
  return randomBytes(32)
    .toString('base64url')
    .substring(0, 128);
}

// code_challenge の生成（S256方式）
function generateCodeChallenge(verifier: string): string {
  return createHash('sha256')
    .update(verifier)
    .digest('base64url');
}

// OAuth認可リクエスト時にcode_challengeを含める
const authUrl = new URL('https://accounts.google.com/o/oauth2/v2/auth');
authUrl.searchParams.set('client_id', CLIENT_ID);
authUrl.searchParams.set('redirect_uri', REDIRECT_URI);
authUrl.searchParams.set('response_type', 'code');
authUrl.searchParams.set('scope', SCOPES.join(' '));
authUrl.searchParams.set('code_challenge', codeChallenge);
authUrl.searchParams.set('code_challenge_method', 'S256');
authUrl.searchParams.set('state', state); // CSRF対策
```

### 4. 通信とデータ保護の確認

#### TLSの設定

すべての通信がTLS 1.2以上で暗号化されていることを確認します。

```bash
# SSL/TLS設定の確認（SSL Labsを使用）
# https://www.ssllabs.com/ssltest/ でドメインを入力

# コマンドラインでの確認
openssl s_client -connect your-domain.com:443 -tls1_2
```

SSL Labsの評価で**A以上**を取得していることが望ましいです。TLS 1.0や1.1が有効になっている場合は無効化しましょう。

#### HTTPセキュリティヘッダー

以下のセキュリティヘッダーが適切に設定されているかを確認します。

```typescript
// Next.jsのnext.config.jsでの設定例
const securityHeaders = [
  {
    key: 'Strict-Transport-Security',
    value: 'max-age=63072000; includeSubDomains; preload'
  },
  {
    key: 'X-Content-Type-Options',
    value: 'nosniff'
  },
  {
    key: 'X-Frame-Options',
    value: 'DENY'
  },
  {
    key: 'X-XSS-Protection',
    value: '1; mode=block'
  },
  {
    key: 'Referrer-Policy',
    value: 'strict-origin-when-cross-origin'
  },
  {
    key: 'Content-Security-Policy',
    value: "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';"
  },
];
```

#### 保存データの暗号化

データベースに保存する機密情報（OAuthトークン、個人情報など）は、AES-256-GCMなどの暗号化アルゴリズムで暗号化して保存します。平文での保存はCASA評価でNGです。

### 5. 脆弱性対策チェックリスト

CASA Tier 2で特に重点的にチェックされる脆弱性と対策をまとめます。

| 脆弱性 | CWE | 対策 |
|--------|-----|------|
| XSS（クロスサイトスクリプティング） | CWE-79 | 出力エスケープ、CSPヘッダーの設定 |
| SQLインジェクション | CWE-89 | パラメータ化クエリの使用、ORM活用 |
| CSRF（クロスサイトリクエストフォージェリ） | CWE-352 | CSRFトークンの実装、SameSite属性 |
| 認証の不備 | CWE-287 | OAuth 2.0の適切な実装、PKCE |
| セッション固定攻撃 | CWE-384 | ログイン後のセッションID再生成 |
| 機密情報の露出 | CWE-200 | エラーメッセージの最小化、ログの適切な管理 |
| 不適切な暗号化 | CWE-327 | TLS 1.2以上、AES-256-GCM |
| オープンリダイレクト | CWE-601 | リダイレクト先のホワイトリスト検証 |

## 申請から取得までの流れ

ここからは、実際のCASA Tier 2認証の申請・取得プロセスを時系列で記録します。

### Step 1：CASAポータルでの申請開始

Google OAuth審査を申請すると、Googleからアプリのリスクレベルに基づいてCASA Tierが通知されます。その通知に含まれるリンクからCASAポータル（App Defense Allianceのサイト）にアクセスし、評価プロセスを開始します。

申請時に必要な情報は以下のとおりです。

- アプリケーション名
- Google Cloud ProjectのクライアントID
- アプリケーションのURL
- 使用しているOAuthスコープの一覧
- プライバシーポリシーのURL

### Step 2：認定ラボの選択

CASA Tier 2では、App Defense Allianceが認定した**Authorized Lab**（認定セキュリティラボ）による評価が必要です。CASAポータルで利用可能なラボの一覧が表示されるので、予算とスケジュールに合ったラボを選択します。

ラボの選定で考慮すべきポイントは以下です。

- **費用**: ラボによって異なりますが、Tier 2の場合、おおよそ数百〜数千ドル程度
- **対応言語**: 日本語対応の有無（英語のみのラボが多い）
- **リードタイム**: 申込から評価開始までの待ち時間
- **評価期間**: 通常2〜4週間

### Step 3：自己評価の実施

ラボによる評価の前に、CASAが提供するセルフチェックリストに基づいて自己評価を行います。このチェックリストはOWASP ASVSのサブセットで、各項目に対して「適合」「不適合」「該当なし」を回答します。

自己評価の段階で「不適合」の項目があれば、ラボ評価の前に修正しておくべきです。ラボ評価で不合格になると、修正後に再評価が必要となり、追加の時間と費用がかかります。

### Step 4：ラボによるスキャンと評価

認定ラボが以下のプロセスでアプリケーションを評価します。

**4-1. 自動スキャン**

ラボが自動化されたセキュリティスキャンツールを実行し、既知の脆弱性を検出します。OWASP ZAPやBurp Suiteなどのツールが使用されます。

**4-2. 設定レビュー**

SSL/TLS設定、HTTPセキュリティヘッダー、OAuth実装の設定を確認します。

**4-3. 結果レポート**

スキャン結果と評価結果がレポートとして提出されます。問題が検出された場合は、修正が必要な項目と推奨対策が記載されます。

### Step 5：指摘事項への対応（該当する場合）

ラボから指摘事項があった場合は、修正して再スキャンを依頼します。Focalizeの場合、初回のスキャンでいくつかの指摘を受けました。

実際に対応した指摘事項の例を紹介します。

**指摘1: Content-Security-Policyヘッダーの強化**

```
// Before: CSPが未設定
// After: 適切なCSPを設定
Content-Security-Policy: default-src 'self';
  script-src 'self';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  connect-src 'self' https://api.example.com;
```

**指摘2: セッションCookieのセキュリティ属性**

```typescript
// Before: Secure属性のみ
cookie: {
  secure: true,
}

// After: すべてのセキュリティ属性を設定
cookie: {
  secure: true,
  httpOnly: true,
  sameSite: 'lax',
  maxAge: 60 * 60 * 24, // 24時間
}
```

**指摘3: エラーレスポンスの情報漏洩**

```typescript
// Before: スタックトレースがレスポンスに含まれる
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message, stack: err.stack });
});

// After: 本番環境ではスタックトレースを隠蔽
app.use((err, req, res, next) => {
  console.error(err); // ログには記録
  res.status(500).json({
    error: 'Internal Server Error' // 汎用メッセージのみ返す
  });
});
```

修正が完了したら、ラボに再スキャンを依頼し、すべての指摘事項がクリアされたことを確認してもらいます。

### Step 6：認証の発行

すべての評価項目をパスすると、App Defense Allianceから正式にCASA Tier 2認証が発行されます。認証情報はCASAポータル上で確認でき、Googleの審査チームにも自動的に共有されます。

これにより、Google OAuth App Verificationの審査が次のステップに進みます。

### 全体のタイムライン（Focalizeの場合）

| フェーズ | 期間 | 内容 |
|---------|------|------|
| 事前準備 | 2週間 | OWASP ASVSの理解、セルフスキャン、脆弱性修正 |
| 申請・ラボ選定 | 3日 | CASAポータルでの申請、ラボの選択 |
| 自己評価 | 3日 | チェックリストへの回答 |
| ラボ評価 | 2週間 | 自動スキャン、設定レビュー |
| 指摘事項対応 | 1週間 | 修正と再スキャン |
| 認証発行 | 2日 | 最終確認と認証の発行 |
| **合計** | **約5〜6週間** | |

事前準備にどれだけ時間をかけるかで、全体のスケジュールが大きく変わります。セルフスキャンで重大な脆弱性を事前に潰しておけば、ラボ評価はスムーズに進みます。逆に、事前準備を怠ると指摘事項が増え、修正と再評価のサイクルで数か月かかることもあります。

## CASA Tier 2取得後のメリット

### 1. Google OAuth審査の通過

最も直接的なメリットは、Google OAuth App Verificationの審査を通過できることです。CASA認証がないとsensitiveスコープを使用するアプリは公開できないため、これは必須条件です。

### 2. ユーザーの信頼獲得

BtoBのSaaSでは、導入検討時にセキュリティチェックシートへの回答を求められることが珍しくありません。CASA Tier 2認証を取得していれば、「第三者機関によるセキュリティ評価を受けている」と回答でき、導入のハードルが下がります。

Focalizeのランディングページでも、セキュリティセクションに「CASA Tier 2認証」「Google Verified」「データ暗号化」のバッジを掲載しています。これは単なるマーケティングではなく、ユーザーが安心してGoogleアカウントを連携するための根拠です。

### 3. セキュリティ体制の底上げ

CASA取得のプロセスを通じて、コードベース全体のセキュリティ品質が向上します。事前準備で行う脆弱性スキャンや設定レビューは、CASA認証のためだけでなく、プロダクト全体の品質に寄与します。

具体的にFocalizeで改善された点を挙げると、以下のとおりです。

- セキュリティヘッダーの網羅的な設定
- OAuthトークンの暗号化保存の標準化
- エラーハンドリングの統一（本番環境での情報漏洩防止）
- 依存パッケージの脆弱性を定期的にチェックする仕組みの導入
- セキュリティテストのCI/CDパイプラインへの組み込み

### 4. 認証の継続的な価値

CASA認証には有効期限があり、定期的な再評価が必要です。これは一見コストに見えますが、セキュリティ対策を継続的に見直す強制力として機能します。「取得して終わり」ではなく、セキュリティを維持・改善し続ける仕組みとして捉えるべきです。

## SaaS開発者へのアドバイス

最後に、これからCASA認証の取得を検討しているSaaS開発者に向けて、実体験からのアドバイスをまとめます。

### 開発初期からセキュリティを組み込む

CASA認証を「後から対応するもの」と考えていると、取得時に大量の修正が必要になります。OAuth実装、暗号化、セキュリティヘッダーの設定は、開発初期の段階で正しく実装しておくべきです。技術的負債としてセキュリティ対策を先送りすると、認証取得時に大きなコストになります。

### セルフスキャンを習慣化する

`npm audit` やOWASP ZAPによるスキャンを、CI/CDパイプラインに組み込みましょう。デプロイのたびに自動でスキャンが走る仕組みがあれば、脆弱性の混入を早期に検出できます。

```yaml
# GitHub Actionsでのセキュリティスキャン例
name: Security Scan
on: [push, pull_request]
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm audit --audit-level=high
```

### ドキュメントを整備する

CASAの評価プロセスでは、技術的な実装だけでなく、プライバシーポリシーや利用規約、データの取り扱いに関するドキュメントも確認されます。以下は最低限整備しておくべきドキュメントです。

- プライバシーポリシー（どのデータを、なぜ、どのように収集・保存・利用するか）
- 利用規約
- データ保持ポリシー（データをいつまで保持し、どのように削除するか）

### 余裕を持ったスケジュールを組む

CASA認証の取得には、事前準備を含めて最低でも1か月、余裕を見て2か月程度のスケジュールを確保すべきです。Google OAuth審査全体のスケジュールを考慮し、逆算して早めに着手することをおすすめします。

## まとめ

CASA Tier 2認証の取得は、Google連携SaaSを公開するための技術的な要件であると同時に、プロダクトのセキュリティ品質を客観的に証明する手段です。

取得プロセスで重要なポイントを整理します。

1. **事前準備が最も重要**。OWASP ASVSの要件を理解し、セルフスキャンで脆弱性を事前に潰す
2. **OAuthの実装品質**を重点的にチェックする。スコープの最小化、トークンの暗号化、PKCEの実装
3. **セキュリティヘッダーとTLS設定**を網羅的に確認する
4. **ラボ評価はスムーズに進めるために**、自己評価の段階で不適合項目をゼロにしておく
5. **認証取得後もセキュリティ対策を継続する**仕組みを構築する

CASA認証の取得は、一度やれば終わりの作業ではなく、セキュリティを継続的に維持・改善するための出発点です。SaaS開発者として、ユーザーのデータを預かる責任を果たすための基盤として、積極的に取り組むことをおすすめします。

---

**Focalize**は、CASA Tier 2認証を取得した営業自動化SaaSです。Google OAuth連携によるセキュアなGmail・Googleカレンダー統合で、商談管理を安全に自動化できます。

https://focalize.jp
