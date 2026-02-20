---
title: "CASA Tier 2認証 — SaaSのセキュリティ認証を1人で取得する"
---

## Google連携アプリに求められる「信頼の証明」

SaaSプロダクトを開発していて、Google OAuth連携を実装した経験がある方は多いと思います。GmailやGoogleカレンダーのデータにアクセスするアプリケーションを公開するには、Googleの審査（OAuth App Verification）を通過する必要があります。

この審査プロセスの中で、アプリケーションのセキュリティレベルに応じて要求されるのが**CASA（Cloud Application Security Assessment）認証**です。

Focalizeは、Gmail連携によるメール送受信とGoogleカレンダー連携による日程調整自動化を提供しており、Google OAuthでsensitiveスコープのデータを扱います。そのため、CASA Tier 2認証の取得が必要でした。

この章では、CASA認証の概要から、Tier 2を実際に取得するまでの準備・申請・取得の全プロセスを、マルチプロダクト運用の文脈で解説します。1プロダクトで得た知見を他のプロダクトにどう横展開したかも含めて共有します。

## CASA認証とは何か

CASA（Cloud Application Security Assessment）は、**App Defense Alliance（ADA）**が運営するクラウドアプリケーション向けのセキュリティ評価プログラムです。App Defense AllianceはGoogleが共同設立した組織で、Google APIにアクセスするサードパーティアプリケーションが、一定のセキュリティ基準を満たしていることを検証します。

具体的には、**OWASP ASVS（Application Security Verification Standard）4.0**をベースにしたセキュリティ要件への準拠が求められます。

### Tier 1・Tier 2・Tier 3の違い

| 項目 | Tier 1 | Tier 2 | Tier 3 |
|------|--------|--------|--------|
| 対象 | リスクが低いアプリ | 中程度のリスクのアプリ | 高リスクのアプリ |
| 評価方法 | 自己評価 | 認定ラボによる評価 | 認定ラボによる詳細評価 |
| スコープ例 | 基本的なプロフィール情報のみ | sensitiveスコープ | restrictedスコープ |
| 検証の厳密さ | アンケート回答 | 自動スキャン + レビュー | 手動ペネトレーションテスト含む |
| 期間目安 | 数日 | 2〜6週間 | 4〜12週間 |
| 費用 | 無料 | 有料（ラボによる） | 高額（ラボによる） |

Focalizeは`gmail.send`と`calendar.events`というsensitiveスコープを使用しているため、**Tier 2**が要求されました。

### どのTierが必要かの判断

- **Tier 1**: `openid`、`profile`、`email` など、基本的なユーザー情報のみ
- **Tier 2**: Gmail送信（`gmail.send`）、カレンダー読み書き（`calendar.events`）など
- **Tier 3**: Gmail全アクセス（`gmail.readonly`）、Drive全アクセスなど

スコープの選択は技術的な判断であると同時に、認証取得のコストに直結する経営判断でもあります。Focalizeでは「メール送信のみ」で`gmail.send`に限定し、「メール受信の全履歴」にアクセスする`gmail.readonly`は使わない設計にしています。これにより、Tier 3ではなくTier 2で済んでいます。

## Tier 2取得に必要な準備

### 1. OWASP ASVSの要件を理解する

CASA Tier 2の評価基準は、OWASP ASVS 4.0のLevel 1をベースにしています。主要な評価カテゴリは以下のとおりです。

| カテゴリ | 概要 | 代表的なチェック項目 |
|---------|------|-------------------|
| 認証 | ユーザー認証の安全性 | OAuth 2.0の適切な実装、トークン管理 |
| セッション管理 | セッションの安全な管理 | タイムアウト、トークンの無効化 |
| アクセス制御 | 権限管理の適切さ | 認可チェック、権限昇格の防止 |
| 入力検証 | 入力データの検証 | XSS対策、SQLインジェクション対策 |
| 暗号化 | データの暗号化 | TLS 1.2以上、保存データの暗号化 |
| エラー処理 | エラー情報の管理 | スタックトレースの非公開、適切なログ |
| データ保護 | 個人情報の保護 | 機密データの最小化、安全な廃棄 |

### 2. セキュリティスキャンの事前実施

Tier 2では認定ラボがセキュリティスキャンを実施しますが、事前に自社でもスキャンを行い、脆弱性を洗い出しておくことが重要です。

**SAST（静的解析）:**

```bash
# 依存パッケージの脆弱性チェック
npm audit

# Semgrepによる静的解析
semgrep scan --config=auto --error .
```

`npm audit` で検出された`high`や`critical`の脆弱性が残っている状態では、CASA評価をパスできません。

**DAST（動的解析）:**

```bash
# OWASP ZAPによるスキャン
docker run -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
  -t https://your-app-url.com
```

OWASP ZAPはCASA評価でラボが実際に使用するツールの一つでもあります。事前に自社で実行し、検出された警告をすべて対処しておくことで、ラボ評価をスムーズに通過できます。

### 3. OAuth実装のベストプラクティス

OAuth実装の品質は、CASA評価で重点的にチェックされるポイントです。

**スコープの最小化:**

```typescript
// 良い例：必要なスコープのみを指定
const SCOPES = [
  'https://www.googleapis.com/auth/gmail.send',
  'https://www.googleapis.com/auth/calendar.events',
  'https://www.googleapis.com/auth/userinfo.email',
  'https://www.googleapis.com/auth/userinfo.profile',
];
```

**トークンの暗号化保存:**

```typescript
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto';

const ALGORITHM = 'aes-256-gcm';

function encryptToken(token: string, key: Buffer): string {
  const iv = randomBytes(16);
  const cipher = createCipheriv(ALGORITHM, key, iv);
  let encrypted = cipher.update(token, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  const authTag = cipher.getAuthTag();
  return `${iv.toString('hex')}:${authTag.toString('hex')}:${encrypted}`;
}
```

アクセストークンとリフレッシュトークンは、AES-256-GCMで暗号化してデータベースに保存します。平文でのトークン保存はCASA評価でNGです。

**PKCE（Proof Key for Code Exchange）:**

```typescript
import { randomBytes, createHash } from 'crypto';

function generateCodeVerifier(): string {
  return randomBytes(32).toString('base64url').substring(0, 128);
}

function generateCodeChallenge(verifier: string): string {
  return createHash('sha256').update(verifier).digest('base64url');
}
```

OAuth 2.0のAuthorization Code FlowでPKCEを使用することで、認可コードの傍受攻撃を防止します。

### 4. 脆弱性対策チェックリスト

CASA Tier 2で特に重点的にチェックされる項目です。

| 脆弱性 | CWE | Focalizeでの対策 |
|--------|-----|-----------------|
| XSS | CWE-79 | CSPヘッダー、React自動エスケープ |
| SQLインジェクション | CWE-89 | Supabase ORM、パラメータ化クエリ |
| CSRF | CWE-352 | SameSite Cookie、CSRFトークン |
| 認証の不備 | CWE-287 | OAuth 2.0 + PKCE |
| セッション固定攻撃 | CWE-384 | ログイン後のセッションID再生成 |
| 機密情報の露出 | CWE-200 | 本番エラーメッセージの最小化 |
| 不適切な暗号化 | CWE-327 | TLS 1.2以上、AES-256-GCM |
| オープンリダイレクト | CWE-601 | リダイレクト先のホワイトリスト検証 |

## 申請から取得までの流れ

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

### ラボから受けた指摘と対応

実際にFocalizeがラボから受けた指摘事項の例です。

**指摘1: CSPヘッダーの強化**

初期の段階ではContent-Security-Policyが不十分でした。ラボの指摘を受けて、接続先の外部サービスをすべてホワイトリストに明示する厳密なCSPを設定しました。

**指摘2: セッションCookieのセキュリティ属性**

```typescript
// Before: Secure属性のみ
cookie: { secure: true }

// After: すべてのセキュリティ属性を設定
cookie: {
  secure: true,
  httpOnly: true,
  sameSite: 'lax',
  maxAge: 60 * 60 * 24,  // 24時間
}
```

**指摘3: エラーレスポンスの情報漏洩**

本番環境でスタックトレースがレスポンスに含まれていた箇所を修正し、汎用的なエラーメッセージのみを返すようにしました。ログには詳細を記録しつつ、ユーザーには最小限の情報だけを返す。

## マルチプロダクト運用の文脈での価値

### 他プロダクトへの横展開

CASA Tier 2取得のために実装したセキュリティ対策は、Focalize固有のものではありません。以下の対策は他のNext.jsプロダクトにそのまま横展開しています。

- **セキュリティヘッダーのテンプレート**: next.config.mjsの設定をコピー&調整
- **CIパイプラインのSASTスキャン**: Semgrepの設定をShareTokuにも適用
- **エラーハンドリングの方針**: 本番環境での情報漏洩防止パターン
- **依存パッケージの脆弱性チェック**: `npm audit`のCI組み込み

1つのプロダクトで認証取得に投資した時間が、他の4プロダクトのセキュリティ品質を無料で底上げしてくれる。これはマルチプロダクト運用の大きなメリットです。

### セキュリティをCI/CDに組み込む習慣

CASA認証の取得プロセスを通じて、「セキュリティスキャンをCIの一部として自動実行する」という習慣が定着しました。この習慣は認証の有無に関係なく、すべてのプロダクトで価値があります。

```yaml
# 全Next.jsプロダクト共通のセキュリティチェック
- run: npm audit --audit-level=high
- run: npx tsc --noEmit
- run: semgrep scan --config=auto --error .
```

### 認証の継続的な価値

CASA認証には有効期限があり、定期的な再評価が必要です。これは一見コストですが、セキュリティ対策を継続的に見直す強制力として機能します。技術的負債としてセキュリティ対策を先送りすることを防いでくれる外部の規律です。

## SaaS開発者へのアドバイス

### 開発初期からセキュリティを組み込む

CASA認証を「後から対応するもの」と考えていると、取得時に大量の修正が必要になります。OAuth実装、暗号化、セキュリティヘッダーの設定は、開発初期の段階で正しく実装しておくべきです。

### セルフスキャンを習慣化する

`npm audit`やSemgrepによるスキャンを、CI/CDパイプラインに組み込みましょう。デプロイのたびに自動でスキャンが走る仕組みがあれば、脆弱性の混入を早期に検出できます。

### ドキュメントを整備する

CASAの評価プロセスでは、技術的な実装だけでなく、以下のドキュメントも確認されます。

- プライバシーポリシー（データの収集・保存・利用方法）
- 利用規約
- データ保持ポリシー（データの保持期間と削除方法）

### 余裕を持ったスケジュール

事前準備を含めて最低でも1か月、余裕を見て2か月程度のスケジュールを確保すべきです。Google OAuth審査全体のスケジュールを考慮し、逆算して早めに着手することをおすすめします。

次の章では、シニア向けアプリ開発のUX原則とFlutter実装テクニックを解説します。
