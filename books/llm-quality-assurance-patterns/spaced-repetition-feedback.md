---
title: "SM-2アルゴリズムとフィードバックループの設計"
---

## フィードバックループとは何か

前章までで解説したバリデーション（Layer 1, Layer 2）は「生成直後」に品質を担保する仕組みです。しかし、すべての品質問題を生成直後に検出することはできません。曖昧な問題、難易度の不適切さ、微妙なハルシネーションは、ユーザーが実際にクイズを解いて初めて顕在化します。

フィードバックループ（Layer 3）は、ユーザーの学習行動データを品質シグナルとして活用し、AI生成コンテンツを継続的に改善する仕組みです。

MochiQでは、SM-2アルゴリズムによるスペースドリピティション（間隔反復学習）のデータが、そのまま品質フィードバックの情報源になっています。

## SM-2アルゴリズムの基本

SM-2（SuperMemo 2）は、1987年にPiotr Wozniakが発表した間隔反復学習アルゴリズムです。MochiQではこのアルゴリズムを実装し、各問題の出題タイミングを最適化しています。

### MochiQの実装

```dart
class SpacedRepetitionService {
  static const int _fastResponseMs = 5000;   // 5秒以内 = 確信あり
  static const int _slowResponseMs = 10000;  // 10秒以内 = やや迷い
  static const double _minEaseFactor = 1.3;

  /// SM-2アルゴリズムで次回復習日を計算
  ReviewSchedule calculateNextReview({
    required ReviewSchedule? currentSchedule,
    required int quality,
    DateTime? now,
  }) {
    now ??= DateTime.now();

    final current = currentSchedule ?? ReviewSchedule();
    int repetitions = current.repetitions;
    double easeFactor = current.easeFactor;
    int interval = current.interval;

    if (quality >= 3) {
      // 正解の場合: 間隔を拡大
      if (repetitions == 0) {
        interval = 1;         // 初回: 1日後
      } else if (repetitions == 1) {
        interval = 6;         // 2回目: 6日後
      } else {
        interval = (interval * easeFactor).round();  // 以降: 前回間隔 x EF
      }
      repetitions += 1;
    } else {
      // 不正解の場合: リセット
      repetitions = 0;
      interval = 1;
    }

    // EaseFactor更新式
    // EF' = EF + (0.1 - (5 - q) * (0.08 + (5 - q) * 0.02))
    easeFactor = easeFactor +
        (0.1 - (5 - quality) * (0.08 + (5 - quality) * 0.02));
    easeFactor = max(_minEaseFactor, easeFactor);

    final nextReviewDate = now.add(Duration(days: interval));

    return ReviewSchedule(
      repetitions: repetitions,
      easeFactor: easeFactor,
      interval: interval,
      nextReviewDate: nextReviewDate,
    );
  }
}
```

### 品質評価（Quality）の決定

SM-2の品質評価は0-5のスケールです。MochiQでは正誤と回答時間から自動的に品質評価を算出しています。

```dart
/// 回答結果からSM-2品質評価を算出
int getQualityFromResult({
  required bool isCorrect,
  int? responseTimeMs,
}) {
  if (!isCorrect) {
    return 2;  // 不正解は固定で2
  }

  if (responseTimeMs == null) {
    return 4;  // 時間情報なしはデフォルト4
  }

  if (responseTimeMs <= 5000) {
    return 5;  // 5秒以内: 確信あり
  } else if (responseTimeMs <= 10000) {
    return 4;  // 10秒以内: やや迷い
  } else {
    return 3;  // それ以上: かなり迷い
  }
}
```

ここで重要なのは、**回答時間が品質シグナルとして機能する**点です。正解であっても回答に10秒以上かかった問題は、quality=3（かなり迷い）と評価され、復習間隔が短く設定されます。

## ユーザー行動データからの品質問題検出

SM-2のデータは、問題ごとの `repetitions`（繰り返し回数）、`easeFactor`（記憶の容易さ）、`interval`（復習間隔）として蓄積されます。このデータから、AI生成クイズの品質問題を検出できます。

### シグナル1: 正答率が異常に低い問題

```dart
// 3回以上学習したにもかかわらず、easeFactorが最低値に近い問題
// → 問題そのものに品質問題がある可能性
bool isSuspiciouslyDifficult(ReviewSchedule review) {
  return review.repetitions >= 3 && review.easeFactor <= 1.5;
}
```

ユーザーが何度学習しても正解できない問題は、ユーザーの記憶力の問題ではなく、問題の品質に起因する可能性があります。以下のいずれかに該当するケースが多いです。

- 正解が実は間違っている（ハルシネーション）
- 問題文が曖昧で正解を特定できない
- 難易度が高すぎる

### シグナル2: 回答時間が異常に長い問題

```dart
// 正答率は高いが、常に回答に時間がかかる問題
// → 曖昧な問題の可能性（正解はわかるが、迷ってしまう）
bool isPotentiallyAmbiguous(ReviewSchedule review, int avgResponseTimeMs) {
  return review.repetitions >= 2 &&
         review.easeFactor >= 2.0 &&  // 正解はしている
         avgResponseTimeMs > 8000;     // しかし毎回迷う
}
```

「正解できるけど、毎回迷う」問題は、曖昧な選択肢が含まれている可能性が高いです。例えば、「東京」と「東京都」のように、複数の選択肢が正解に見えるケースです。

### シグナル3: ユーザーが編集・削除した問題

ユーザーが問題を手動で編集したり削除したりする行為は、最も明確な品質シグナルです。

```typescript
// ユーザーアクションによる品質シグナル
interface UserActionSignal {
  type: "edit" | "delete" | "report";
  questionId: string;
  timestamp: Date;
  editedFields?: string[];  // どのフィールドが編集されたか
}

function analyzeUserActions(signals: UserActionSignal[]): QualityReport {
  const deleteRate = signals.filter((s) => s.type === "delete").length / signals.length;
  const editRate = signals.filter((s) => s.type === "edit").length / signals.length;

  // 正解の編集は特に重要なシグナル
  const correctEdits = signals.filter(
    (s) => s.type === "edit" && s.editedFields?.includes("correct"),
  );

  return {
    overallQuality: 1 - (deleteRate * 0.5 + editRate * 0.3),
    correctAnswerIssues: correctEdits.length,
    actionBreakdown: {
      deletes: signals.filter((s) => s.type === "delete").length,
      edits: signals.filter((s) => s.type === "edit").length,
      reports: signals.filter((s) => s.type === "report").length,
    },
  };
}
```

## 統計的な品質モニタリング

個別の問題だけでなく、クイズセット全体やプロンプトバージョン単位で品質を監視します。

### 習熟度による品質評価

MochiQの `UserQuizSet` には習熟度を計算するロジックが組み込まれています。

```dart
/// 習熟度パーセンテージを計算
///
/// SM-2アルゴリズムのrepetitionsとeaseFactorを基に習熟度を算出
/// - repetitions >= 3 かつ easeFactor >= 2.3 で「習得済み」とみなす
double get masteryPercentage {
  if (questions.isEmpty) return 0;

  int masteredCount = 0;
  for (final question in questions) {
    final review = question.review;
    if (review != null && review.repetitions >= 3 && review.easeFactor >= 2.3) {
      masteredCount++;
    }
  }

  return (masteredCount / questions.length) * 100;
}
```

この習熟度データをクイズセット横断で集計すると、以下のような品質指標が得られます。

```typescript
// クイズセット横断の品質ダッシュボード
interface QualityDashboard {
  // 全体指標
  totalQuizSets: number;
  averageMasteryRate: number;      // 平均習熟率
  averageEaseFactor: number;       // 平均EaseFactor

  // 品質問題の検出
  suspiciousQuestions: number;     // 品質問題の疑いがある問題数
  lowMasteryQuizSets: number;     // 習熟率が異常に低いクイズセット数

  // プロンプトバージョン別
  qualityByPromptVersion: Map<string, {
    avgMastery: number;
    avgEaseFactor: number;
    suspiciousRate: number;
  }>;
}

function buildQualityDashboard(
  quizSets: UserQuizSet[],
): QualityDashboard {
  const allQuestions = quizSets.flatMap((qs) => qs.questions);
  const reviewedQuestions = allQuestions.filter((q) => q.review != null);

  const suspiciousCount = reviewedQuestions.filter(
    (q) => q.review!.repetitions >= 3 && q.review!.easeFactor <= 1.5,
  ).length;

  const avgEaseFactor =
    reviewedQuestions.reduce((sum, q) => sum + q.review!.easeFactor, 0) /
    reviewedQuestions.length;

  return {
    totalQuizSets: quizSets.length,
    averageMasteryRate:
      quizSets.reduce((sum, qs) => sum + qs.masteryPercentage, 0) /
      quizSets.length,
    averageEaseFactor: avgEaseFactor,
    suspiciousQuestions: suspiciousCount,
    lowMasteryQuizSets: quizSets.filter((qs) => qs.masteryPercentage < 10).length,
    qualityByPromptVersion: new Map(),  // プロンプトバージョンごとに集計
  };
}
```

### アラート閾値の設定

品質メトリクスの異常を検知するための閾値を設定します。

| メトリクス | 正常範囲 | 警告閾値 | 異常閾値 |
|-----------|---------|---------|---------|
| 平均習熟率 | 40-80% | < 30% | < 15% |
| 平均EaseFactor | 2.0-2.8 | < 1.8 | < 1.5 |
| 品質疑い問題率 | < 5% | 5-10% | > 10% |
| ユーザー削除率 | < 3% | 3-8% | > 8% |

## フィードバックを品質改善に活かす流れ

フィードバックループの最終目標は、検出した品質問題を上流（プロンプト設計、バリデーションルール）の改善に還元することです。

```
ユーザー学習データ
    ↓
品質シグナル検出
    ↓
品質ダッシュボード集約
    ↓
┌───────────────────────────────┐
│ パターン分析                    │
│ ・特定のプロンプトバージョンで  │
│   品質低下 → プロンプト改善    │
│ ・特定の教科で品質低下         │
│   → バリデーションルール追加   │
│ ・特定の問題形式で品質低下     │
│   → 生成パラメータ調整        │
└───────────────────────────────┘
    ↓
プロンプト改善 / バリデーション強化
    ↓
品質メトリクスの改善を確認
```

この改善サイクルは手動で回すことも、一定の自動化が可能です。例えば、品質疑い問題率が閾値を超えた場合に、自動的にConfidence Scoring（第6章）を実行して問題をフィルタリングする、といった連携が考えられます。

## まとめ

SM-2アルゴリズムは本来、ユーザーの記憶定着を最適化するための仕組みです。しかし、そのデータは「AI生成コンテンツの品質を測定するセンサー」としても機能します。

- **repetitions** と **easeFactor** の組み合わせで問題の難易度を事後評価できる
- **回答時間**は曖昧な問題の検出に有効
- **ユーザーの編集・削除行為**は最も信頼性の高い品質シグナル
- 統計的な集約により、プロンプト改善の方向性を特定できる

重要なのは、フィードバックループを「監視→検出→改善」の継続的なサイクルとして運用することです。一度設計して終わりではなく、品質メトリクスを定期的にレビューし、閾値やシグナルの定義を見直していく必要があります。
