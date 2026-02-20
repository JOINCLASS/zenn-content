---
title: "バックエンド戦略 — SupabaseとFirebaseの使い分け"
---

## BaaS選定の判断フレームワーク

1人で5プロダクトを運用する場合、自前でバックエンドサーバーを立てて管理する余裕はありません。BaaS（Backend as a Service）に認証・データベース・ストレージ・通知を丸投げするのは、リソース制約から導かれる合理的な判断です。

問題は「どのBaaSを選ぶか」です。ジョインクラスでは、**Supabase**と**Firebase**の2つを使い分けています。「どちらか一方に統一すべきでは？」と思われるかもしれません。実際、統一できるならその方が運用コストは下がります。しかし、両者には明確な得意領域の違いがあり、プロダクトの要件に応じて使い分ける方が総合的なコストが低いと判断しました。

## 選定基準の比較表

```
┌──────────────┬─────────────────┬─────────────────┐
│              │    Supabase     │    Firebase     │
├──────────────┼─────────────────┼─────────────────┤
│ DB           │ PostgreSQL(RDB) │ Firestore(NoSQL)│
│ クエリの柔軟性│ ◎ SQL完全対応   │ △ 制約が多い    │
│ リアルタイム   │ △ (Realtime可)  │ ◎ (標準機能)    │
│ Push通知     │ × (別途必要)     │ ◎ (FCM標準)     │
│ 認証         │ ◎ (Auth)        │ ◎ (Auth)        │
│ Flutter連携  │ △ (公式あり)     │ ◎ (公式推奨)    │
│ Next.js連携  │ ◎ (SSR対応)     │ ○ (Admin SDK)   │
│ RLS          │ ◎ (PostgreSQL)  │ ○ (Security Rules)│
│ マイグレーション│ ◎ (SQL)        │ × (手動)        │
│ ベンダーロックイン│ 低 (PostgreSQL)│ 高 (独自形式)   │
│ 料金         │ 無料枠大きい     │ 無料枠大きい     │
├──────────────┼─────────────────┼─────────────────┤
│ 適用先       │ Webプロダクト    │ モバイルプロダクト │
└──────────────┴─────────────────┴─────────────────┘
```

この比較から導き出したシンプルなルールは、**「Web = Supabase」「モバイル = Firebase」**です。

## Supabaseを選ぶプロダクト

### ShareToku — リレーショナルデータの扱いが中核

ShareTokuはサブスクリプションサービスの紹介プログラムを横断検索するサービスです。データ構造は典型的なリレーショナルモデルで、「サービス」「紹介プログラム」「ユーザー」「レビュー」がテーブル間でリレーションを持ちます。

このようなデータに対して「月額1000円以下のサービスで、紹介特典が最も高いものをランキング表示する」といった複雑なクエリを実行する必要があります。SQLが使えるSupabaseなら、このようなクエリを直感的に書けます。

```sql
-- ShareTokuでの複雑なクエリ例
SELECT
  s.name AS service_name,
  rp.referral_bonus,
  rp.referee_bonus,
  s.monthly_price,
  COUNT(r.id) AS review_count,
  AVG(r.rating) AS avg_rating
FROM services s
JOIN referral_programs rp ON s.id = rp.service_id
LEFT JOIN reviews r ON rp.id = r.referral_program_id
WHERE s.monthly_price <= 1000
  AND rp.is_active = true
GROUP BY s.id, s.name, rp.referral_bonus, rp.referee_bonus, s.monthly_price
ORDER BY rp.referral_bonus DESC
LIMIT 20;
```

Firestoreでこのクエリを実現しようとすると、複数のコレクションからデータを取得して、クライアントサイドでJOINとソートを行う必要があります。パフォーマンスも悪く、コードも複雑になります。

### Focalize — RLSによるマルチテナント設計

Focalizeは営業自動化SaaSで、複数のユーザーがそれぞれ自分の商談データにアクセスします。SupabaseのRow Level Security（RLS）を使うことで、データベースレベルでテナント分離を実現しています。

```sql
-- FocalizeのRLSポリシー例
CREATE POLICY "Users can only access their own deals" ON deals
  FOR ALL
  USING (auth.uid() = user_id);

CREATE POLICY "Users can only access their own contacts" ON contacts
  FOR ALL
  USING (auth.uid() = user_id);
```

RLSのメリットは、**アプリケーションコードに認可ロジックを書く必要がない**ことです。データベースレベルでアクセス制御が強制されるため、コードのバグで他のユーザーのデータが漏洩するリスクが構造的に排除されます。1人開発でコードレビューが十分にできない環境では、この構造的な安全性は非常に重要です。

### Supabase + Next.jsのSSR統合

SupabaseはNext.jsのSSR（Server-Side Rendering）との統合が優れています。`@supabase/ssr` パッケージを使うことで、Server Componentsからデータベースにアクセスし、認証状態を含むページをサーバーサイドでレンダリングできます。

```typescript
// Server ComponentでのSupabaseアクセス例
import { createClient } from '@/lib/supabase/server';

export default async function DealsPage() {
  const supabase = await createClient();
  const { data: deals } = await supabase
    .from('deals')
    .select('*, contacts(*)')
    .order('updated_at', { ascending: false });

  return <DealsList deals={deals} />;
}
```

RLSが有効なので、このコードは自動的に現在ログインしているユーザーの商談データのみを返します。認可チェックのコードを一切書いていないにもかかわらず、データの分離が保証されている。これがRLSの強力さです。

## Firebaseを選ぶプロダクト

### 元気ボタン — リアルタイム通知が中核

元気ボタンの最も重要な要件は「親がボタンを押したら、子どものスマホにすぐ通知が届く」ことです。この「すぐ」は、遅延が数秒以内であることを意味します。

Firebaseは、FirestoreのリアルタイムリスナーとCloud Messaging（FCM）が標準で統合されています。Firestoreにデータが書き込まれたタイミングでCloud Functionsをトリガーし、FCMでプッシュ通知を送信する。このフロー全体がFirebase内で完結します。

```dart
// 元気ボタン: Firestoreへのチェックイン記録
Future<void> checkIn() async {
  final user = FirebaseAuth.instance.currentUser;
  if (user == null) return;

  await FirebaseFirestore.instance
    .collection('checkins')
    .add({
      'userId': user.uid,
      'familyId': _familyId,
      'checkedAt': FieldValue.serverTimestamp(),
      'type': 'daily',
    });
}
```

この書き込みをトリガーに、Cloud Functionsが家族メンバーのFCMトークンを取得してプッシュ通知を送信します。Supabaseでこの一連のフローを実現しようとすると、プッシュ通知の部分を別途構築する必要があります（SupabaseにはFCMに相当する機能がありません）。

### MochiQ — Firebase Authの匿名認証

MochiQでは、アプリの初回起動時にユーザー登録を求めずに使い始められるようにしています。Firebase Authの**匿名認証（Anonymous Auth）**は、この要件にぴったりです。

```dart
// MochiQ: 匿名認証で即座に利用開始
Future<void> signInAnonymously() async {
  final userCredential = await FirebaseAuth.instance.signInAnonymously();
  // ユーザーはすぐにクイズ生成を開始できる
  // 後からメールアドレス登録でアカウントを永続化
}
```

匿名ユーザーとしてクイズを生成・学習した後、有料プランに移行する際にメールアドレスを登録する。この段階的なオンボーディングは、学習アプリのコンバージョン率を大きく左右します。Supabase Authにも匿名認証はありますが、Flutter公式プラグインの統合度ではFirebase Authの方が安定しています。

### Firestoreのスキーマレス設計

MochiQのクイズデータは、形式がクイズの種類によって異なります。選択式、記述式、画像付きなど、スキーマが動的に変わるデータの格納にはFirestoreのスキーマレスな構造が適しています。

```
// MochiQのFirestoreデータ構造
quizzes/
  {quizId}/
    title: "日本史 - 江戸時代"
    type: "multiple_choice"
    questions: [
      {
        text: "関ヶ原の戦いは何年？",
        options: ["1598年", "1600年", "1603年", "1615年"],
        correctIndex: 1,
        explanation: "..."
      }
    ]
    spaced_repetition: {
      nextReviewAt: Timestamp,
      interval: 3,  // 日数
      easeFactor: 2.5
    }
```

PostgreSQLのJSONB型でも同様のことは可能ですが、Firestoreの方がFlutterクライアントからの読み書きが自然です。`withConverter` を使えば型安全なデータアクセスも実現できます。

## BaaS依存のトレードオフ

### ベンダーロックインのリスク

SupabaseとFirebaseに認証・データベース・ストレージを丸投げしているのは、**自前でインフラを管理する時間がないから**です。このトレードオフは意識的に受け入れています。

**Supabaseのロックイン度は低い。** SupabaseはPostgreSQL互換なので、最悪の場合はデータを別のPostgreSQLインスタンスに移行できます。RLSポリシーもSQLベースなので、移行先でもそのまま適用可能です。

**Firebaseのロックイン度は高い。** FirestoreはGoogleの独自NoSQLであり、データ構造と問い合わせ方法がFirestore固有です。移行する場合、データモデルの再設計が必要になります。それでもFirebaseを選ぶ理由は、モバイルアプリのリアルタイム同期+プッシュ通知をFirebase以外で同じ品質・コストで実現する代替手段が現時点では見当たらないからです。

### 障害時の影響

BaaSに依存しているということは、SupabaseやFirebaseに障害が発生した場合、自分では何もできないことを意味します。実際に、SupabaseのPostgreSQLがメンテナンスで一時的に接続できなくなったことがあります。

この場合の対策は「待つ」しかありません。ただし、SupabaseとFirebaseを分けていることで、片方に障害が起きても全プロダクトが止まるわけではありません。元気ボタンとMochiQはFirebase側なので、Supabaseの障害には影響されません。

### コストの予測可能性

BaaSのコストは、ユーザー数とデータ量に応じてスケールします。5プロダクトの現在の規模では、SupabaseもFirebaseも無料枠の範囲内で収まっています。詳細なコスト分析は第9章で解説します。

## 選定の意思決定フローチャート

新しいプロダクトでBaaSを選ぶ際の判断フレームワークをまとめます。

```
新しいプロダクトの開始
  │
  ├── Webアプリ? → YES → リレーショナルデータ? → YES → Supabase
  │                                             → NO  → 要検討
  │
  ├── モバイルアプリ? → YES → プッシュ通知が必要? → YES → Firebase
  │                                              → NO  → Supabase
  │                                                       (Flutter連携を許容できるなら)
  │
  └── 静的サイト? → BaaS不要。Static Exportで十分
```

このフレームワークは完璧ではありませんが、「毎回ゼロから検討する」コストを削減できます。80%のケースはこの判断で正しい選択ができ、残りの20%は個別に検討すればよい。

## PostgreSQLを直接使うケース

Focalizeでは、Supabase経由のアクセスに加えて、`pg` パッケージでPostgreSQLに直接接続するケースがあります。これはSupabaseのクライアントライブラリでは表現しにくい複雑なクエリ（サブクエリ、CTE、ウィンドウ関数など）を実行する場合です。

```typescript
// Focalizeでの直接クエリ例
import { Pool } from 'pg';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

// 月別の商談成約率を計算する複雑なクエリ
const result = await pool.query(`
  WITH monthly_deals AS (
    SELECT
      DATE_TRUNC('month', closed_at) AS month,
      COUNT(*) FILTER (WHERE status = 'won') AS won,
      COUNT(*) FILTER (WHERE status IN ('won', 'lost')) AS total
    FROM deals
    WHERE user_id = $1
      AND closed_at >= NOW() - INTERVAL '12 months'
    GROUP BY DATE_TRUNC('month', closed_at)
  )
  SELECT
    month,
    won,
    total,
    ROUND(won::numeric / NULLIF(total, 0) * 100, 1) AS win_rate
  FROM monthly_deals
  ORDER BY month DESC
`, [userId]);
```

Supabaseのクライアントライブラリは基本的なCRUDには十分ですが、分析系のクエリでは直接SQLを書く方が表現力が高いです。ただし、直接接続を使う場合はRLSが適用されない（service_roleキーで接続するため）ので、アプリケーションコード側で必ず`user_id`の検証を行う必要があります。

次の章では、5プロダクトのデプロイ戦略とCI/CDの共通化について解説します。
