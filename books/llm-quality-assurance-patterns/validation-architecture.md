---
title: "3層バリデーションアーキテクチャ"
---

## バリデーションを層に分ける理由

LLMの出力に対するバリデーションを単一のチェック関数で行おうとすると、すぐに肥大化して保守困難になります。MochiQでは、バリデーションを3つの層に分離し、それぞれ異なる関心事に対処する設計を採用しています。

- **Layer 1: 構造バリデーション** — JSONパース、フィールド検証、型チェック
- **Layer 2: コンテンツバリデーション** — 回答品質、難易度推定、偏り検出
- **Layer 3: フィードバックループ** — ユーザー行動データによる品質改善

各層は独立して動作し、Layer 1で弾かれたデータはLayer 2に到達しません。パイプラインのフィルタとして機能します。

## Layer 1: 構造バリデーション

構造バリデーションは「プログラムとして処理できるか」を検証する層です。LLMの出力が期待するデータ構造に合致しているかを確認します。

### JSONパースとフォーマット正規化

まず、LLMのレスポンスからJSON部分を抽出します。前章で紹介した `QuizResponseParser` がこの役割を担いますが、さらに詳細なバリデーションを追加します。

```dart
/// Layer 1: 構造バリデーション
class StructuralValidator {
  /// クイズセット全体の構造を検証
  static ValidationResult validateQuizSet(GeneratedQuizSet quizSet) {
    final errors = <ValidationError>[];

    // タイトルの検証
    if (quizSet.title.isEmpty) {
      errors.add(ValidationError(
        field: 'title',
        message: 'タイトルが空です',
        severity: Severity.warning,
      ));
    }

    // 問題数の検証
    if (quizSet.questions.isEmpty) {
      errors.add(ValidationError(
        field: 'questions',
        message: '問題が1つも含まれていません',
        severity: Severity.critical,
      ));
      return ValidationResult(errors: errors);
    }

    // 各問題の構造を検証
    for (var i = 0; i < quizSet.questions.length; i++) {
      final questionErrors = validateQuestion(quizSet.questions[i], index: i);
      errors.addAll(questionErrors);
    }

    // 重複問題の検出
    final duplicates = _detectDuplicateQuestions(quizSet.questions);
    errors.addAll(duplicates);

    return ValidationResult(errors: errors);
  }

  /// 個別の問題を検証
  static List<ValidationError> validateQuestion(
    GeneratedQuestion question, {
    required int index,
  }) {
    final errors = <ValidationError>[];
    final prefix = 'questions[$index]';

    // 問題文が空でないか
    if (question.question.trim().isEmpty) {
      errors.add(ValidationError(
        field: '$prefix.question',
        message: '問題文が空です',
        severity: Severity.critical,
      ));
    }

    // 選択肢の数を検証
    if (question.type == QuestionType.multipleChoice) {
      if (question.answers.length != 4) {
        errors.add(ValidationError(
          field: '$prefix.answers',
          message: '4択問題の選択肢が${question.answers.length}個です',
          severity: Severity.critical,
        ));
      }
    } else if (question.type == QuestionType.trueFalse) {
      if (question.answers.length != 2) {
        errors.add(ValidationError(
          field: '$prefix.answers',
          message: '○×問題の選択肢が${question.answers.length}個です',
          severity: Severity.critical,
        ));
      }
    }

    // 正解が選択肢に含まれているか
    if (!question.answers.contains(question.correct)) {
      errors.add(ValidationError(
        field: '$prefix.correct',
        message: '正解「${question.correct}」が選択肢に含まれていません',
        severity: Severity.critical,
      ));
    }

    // 選択肢の重複を検出
    final uniqueAnswers = question.answers.toSet();
    if (uniqueAnswers.length != question.answers.length) {
      errors.add(ValidationError(
        field: '$prefix.answers',
        message: '選択肢に重複があります',
        severity: Severity.high,
      ));
    }

    return errors;
  }

  /// 重複問題を検出
  static List<ValidationError> _detectDuplicateQuestions(
    List<GeneratedQuestion> questions,
  ) {
    final errors = <ValidationError>[];
    final seen = <String>{};

    for (var i = 0; i < questions.length; i++) {
      final normalized = questions[i].question.trim().toLowerCase();
      if (seen.contains(normalized)) {
        errors.add(ValidationError(
          field: 'questions[$i]',
          message: '重複した問題です',
          severity: Severity.high,
        ));
      }
      seen.add(normalized);
    }

    return errors;
  }
}
```

TypeScriptでの同等の実装は以下のようになります。

```typescript
// Layer 1: 構造バリデーション（TypeScript版）
interface QuizQuestion {
  question: string;
  answers: string[];
  correct: string;
  type: "multiple_choice" | "true_false";
  explanation?: string;
}

interface ValidationError {
  field: string;
  message: string;
  severity: "critical" | "high" | "warning";
}

function validateQuizStructure(questions: QuizQuestion[]): ValidationError[] {
  const errors: ValidationError[] = [];

  if (questions.length === 0) {
    errors.push({
      field: "questions",
      message: "問題が1つも含まれていません",
      severity: "critical",
    });
    return errors;
  }

  for (const [i, q] of questions.entries()) {
    // 正解が選択肢に含まれているか
    if (!q.answers.includes(q.correct)) {
      errors.push({
        field: `questions[${i}].correct`,
        message: `正解「${q.correct}」が選択肢に含まれていません`,
        severity: "critical",
      });
    }

    // 4択問題の選択肢数チェック
    if (q.type === "multiple_choice" && q.answers.length !== 4) {
      errors.push({
        field: `questions[${i}].answers`,
        message: `4択問題の選択肢が${q.answers.length}個です`,
        severity: "critical",
      });
    }

    // 選択肢の重複チェック
    const unique = new Set(q.answers);
    if (unique.size !== q.answers.length) {
      errors.push({
        field: `questions[${i}].answers`,
        message: "選択肢に重複があります",
        severity: "high",
      });
    }
  }

  return errors;
}
```

### 自動修復（Auto-fix）の導入

構造的不整合の一部は、自動修復が可能です。バリデーションで問題を検出した後、修復できるものは修復し、修復不可能なものだけを棄却します。

```dart
class StructuralAutoFixer {
  /// 修復可能な構造問題を自動修正
  static GeneratedQuizSet autoFix(GeneratedQuizSet quizSet) {
    final fixedQuestions = <GeneratedQuestion>[];

    for (final question in quizSet.questions) {
      final fixed = _fixQuestion(question);
      if (fixed != null) {
        fixedQuestions.add(fixed);
      }
      // 修復不可能な問題はスキップ（棄却）
    }

    return GeneratedQuizSet(
      title: quizSet.title.isEmpty ? 'AI生成クイズ' : quizSet.title,
      questions: fixedQuestions,
      sourceText: quizSet.sourceText,
    );
  }

  static GeneratedQuestion? _fixQuestion(GeneratedQuestion question) {
    var answers = List<String>.from(question.answers);
    var correct = question.correct;

    // 正解が選択肢に含まれていない → 正解を選択肢に追加
    if (!answers.contains(correct)) {
      if (question.type == QuestionType.multipleChoice && answers.length >= 4) {
        answers[answers.length - 1] = correct;  // 最後の選択肢を置換
      } else if (answers.length < 4) {
        answers.add(correct);
      } else {
        return null;  // 修復不可能
      }
    }

    // 選択肢の重複を除去
    final uniqueAnswers = answers.toSet().toList();
    if (uniqueAnswers.length < 2) return null;  // 修復不可能

    // 全角/半角の正規化
    answers = uniqueAnswers.map(_normalizeText).toList();
    correct = _normalizeText(correct);

    return GeneratedQuestion(
      question: question.question,
      answers: answers,
      correct: correct,
      type: question.type,
      explanation: question.explanation,
    );
  }

  static String _normalizeText(String text) {
    return text
        .replaceAll('\u3000', ' ')  // 全角スペース→半角
        .replaceAll(RegExp(r'\s+'), ' ')  // 連続スペースを1つに
        .trim();
  }
}
```

## Layer 2: コンテンツバリデーション

構造的に正しいデータが、内容としても適切かどうかを検証する層です。ここではLLM自身を「バリデータ」として活用するパターンも含まれます。

### 回答品質チェック

生成されたクイズの質を評価するために、別のLLM呼び出しでクロスチェックを行います。

```typescript
// Layer 2: コンテンツバリデーション（TypeScript版）
async function validateQuizContent(
  questions: QuizQuestion[],
  sourceText: string,
): Promise<ContentValidationResult> {
  const issues: ContentIssue[] = [];

  for (const [i, q] of questions.entries()) {
    // ソーステキストとの関連性チェック
    const relevance = await checkRelevanceToSource(q, sourceText);
    if (relevance.score < 0.5) {
      issues.push({
        questionIndex: i,
        type: "hallucination_risk",
        message: `ソーステキストとの関連性が低い (${relevance.score})`,
        confidence: relevance.score,
      });
    }

    // 曖昧性チェック: 正解以外の選択肢が正解になり得ないか
    const ambiguity = checkAnswerAmbiguity(q);
    if (ambiguity.isAmbiguous) {
      issues.push({
        questionIndex: i,
        type: "ambiguous_answer",
        message: `選択肢「${ambiguity.alternativeCorrect}」も正解の可能性`,
        confidence: ambiguity.confidence,
      });
    }
  }

  return { issues, isAcceptable: issues.filter(i => i.type === "hallucination_risk").length === 0 };
}

function checkAnswerAmbiguity(question: QuizQuestion): AmbiguityResult {
  // 正解と他の選択肢の類似度を計算
  for (const answer of question.answers) {
    if (answer === question.correct) continue;

    const similarity = calculateSimilarity(answer, question.correct);
    if (similarity > 0.8) {
      return {
        isAmbiguous: true,
        alternativeCorrect: answer,
        confidence: similarity,
      };
    }
  }

  return { isAmbiguous: false, alternativeCorrect: null, confidence: 0 };
}
```

### 難易度推定

問題文と選択肢の語彙レベルから、問題の難易度を推定します。

```dart
class DifficultyEstimator {
  /// 問題の推定難易度を返す（1-5のスケール）
  static int estimateDifficulty(GeneratedQuestion question) {
    int score = 0;

    // 問題文の長さ
    if (question.question.length > 100) score += 1;
    if (question.question.length > 200) score += 1;

    // 選択肢の類似度（高いほど難しい）
    final avgSimilarity = _averageChoiceSimilarity(question.answers);
    if (avgSimilarity > 0.6) score += 1;
    if (avgSimilarity > 0.8) score += 1;

    // 専門用語の含有率
    final technicalTermRatio = _technicalTermRatio(question.question);
    if (technicalTermRatio > 0.2) score += 1;

    return (score + 1).clamp(1, 5);
  }

  static double _averageChoiceSimilarity(List<String> answers) {
    if (answers.length < 2) return 0;
    double total = 0;
    int count = 0;
    for (var i = 0; i < answers.length; i++) {
      for (var j = i + 1; j < answers.length; j++) {
        total += _stringSimilarity(answers[i], answers[j]);
        count++;
      }
    }
    return count > 0 ? total / count : 0;
  }

  static double _stringSimilarity(String a, String b) {
    final setA = a.split('').toSet();
    final setB = b.split('').toSet();
    final intersection = setA.intersection(setB);
    final union = setA.union(setB);
    return union.isEmpty ? 0 : intersection.length / union.length;
  }

  static double _technicalTermRatio(String text) {
    // 簡易的な専門用語検出（漢字4文字以上の連続を専門用語とみなす）
    final technicalTerms = RegExp(r'[\u4e00-\u9fff]{4,}').allMatches(text);
    final totalChars = text.length;
    if (totalChars == 0) return 0;
    final termChars = technicalTerms.fold<int>(
      0, (sum, match) => sum + match.group(0)!.length,
    );
    return termChars / totalChars;
  }
}
```

## Layer 3: フィードバックループ

Layer 1とLayer 2は「生成直後」の検証です。Layer 3は「ユーザーが実際に使った後」のデータを用いて品質を改善する仕組みです。詳細は第7章で解説しますが、概要をここで紹介します。

### ユーザー行動シグナルによる品質検出

```dart
class QualitySignalDetector {
  /// ユーザーの回答パターンから品質問題のシグナルを検出
  static List<QualitySignal> detectSignals(UserQuizSet quizSet) {
    final signals = <QualitySignal>[];

    for (final question in quizSet.questions) {
      final review = question.review;
      if (review == null) continue;

      // シグナル1: 正答率が異常に低い問題
      // repetitionsが3以上なのにeaseFactorが最低値に近い
      if (review.repetitions >= 3 && review.easeFactor <= 1.5) {
        signals.add(QualitySignal(
          questionId: question.id,
          type: SignalType.suspiciouslyDifficult,
          message: '繰り返し学習しても習得されない問題',
          confidence: 0.7,
        ));
      }

      // シグナル2: easeFactorが急激に低下した問題
      // （最初は正解できたが、後から間違えるようになった → 曖昧な問題の可能性）
      if (review.easeFactor < 1.8 && review.repetitions >= 2) {
        signals.add(QualitySignal(
          questionId: question.id,
          type: SignalType.possiblyAmbiguous,
          message: 'easeFactorが低下傾向の問題',
          confidence: 0.5,
        ));
      }
    }

    return signals;
  }
}
```

## 3層を統合するバリデーションパイプライン

3つの層を統合して、一つのバリデーションパイプラインとして動作させます。

```dart
class QuizValidationPipeline {
  /// バリデーションを実行し、品質保証済みのクイズセットを返す
  static Future<ValidationPipelineResult> validate(
    GeneratedQuizSet rawQuizSet, {
    required String sourceText,
    String? gradeLevel,
  }) async {
    // Layer 1: 構造バリデーション
    final structuralResult = StructuralValidator.validateQuizSet(rawQuizSet);

    if (structuralResult.hasCriticalErrors) {
      // 自動修復を試みる
      final fixed = StructuralAutoFixer.autoFix(rawQuizSet);
      final revalidation = StructuralValidator.validateQuizSet(fixed);

      if (revalidation.hasCriticalErrors) {
        return ValidationPipelineResult.rejected(
          reason: '構造的な問題を修復できませんでした',
          errors: revalidation.errors,
        );
      }
      rawQuizSet = fixed;
    }

    // Layer 2: コンテンツバリデーション（非同期処理）
    final contentResult = await ContentValidator.validate(
      rawQuizSet,
      sourceText: sourceText,
      gradeLevel: gradeLevel,
    );

    // 品質スコアの算出
    final qualityScore = _calculateQualityScore(
      structuralResult,
      contentResult,
    );

    return ValidationPipelineResult.accepted(
      quizSet: rawQuizSet,
      qualityScore: qualityScore,
      warnings: [...structuralResult.warnings, ...contentResult.warnings],
    );
  }

  static double _calculateQualityScore(
    ValidationResult structural,
    ContentValidationResult content,
  ) {
    double score = 1.0;
    score -= structural.warnings.length * 0.05;
    score -= content.issues.length * 0.1;
    return score.clamp(0.0, 1.0);
  }
}
```

## まとめ: 各層の責務

| 層 | 検証対象 | 実行タイミング | 自動修復 |
|----|---------|--------------|---------|
| Layer 1 | JSONの構造、型、フィールド | 生成直後（同期） | 可能 |
| Layer 2 | コンテンツの正確性、難易度、曖昧さ | 生成直後（非同期） | 一部可能 |
| Layer 3 | ユーザー行動に基づく品質シグナル | 学習セッション後 | 問題のフラグ付け |

この3層構造により、生成直後に検出できる問題は即座に対処し、使用後にしかわからない問題はフィードバックループで段階的に改善するという戦略が実現できます。
