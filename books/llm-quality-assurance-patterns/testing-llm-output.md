---
title: "LLM出力のテスト戦略"
---

## 従来のテストとLLMテストの違い

従来のソフトウェアテストは「入力Xに対して出力Yが返ること」をアサーションします。LLMの出力は非決定論的なので、この前提が成り立ちません。同じプロンプトに対して毎回異なる出力が返る以上、テスト戦略を根本から見直す必要があります。

本章では、LLM出力のテストを5つの手法に分類し、それぞれの実装方法とCI/CDへの組み込み方を解説します。

## 手法1: 決定論的テスト（構造バリデーション）

LLM出力のうち、決定論的に検証できる部分を対象にしたテストです。第4章のLayer 1（構造バリデーション）をテストケースとして記述します。

```dart
// Dart（Flutter）でのテスト例
import 'package:test/test.dart';

void main() {
  group('QuizResponseParser', () {
    test('正常なJSONをパースできること', () {
      final response = '''
      {
        "title": "テストクイズ",
        "questions": [
          {
            "question": "1+1は？",
            "answers": ["1", "2", "3", "4"],
            "correct": "2",
            "type": "multiple_choice"
          }
        ],
        "sourceText": "算数の基礎"
      }
      ''';

      final result = QuizResponseParser.parse(response);
      expect(result, isNotNull);
      expect(result!.title, equals('テストクイズ'));
      expect(result.questions.length, equals(1));
      expect(result.questions[0].correct, equals('2'));
    });

    test('マークダウンコードブロック内のJSONをパースできること', () {
      final response = '''
      以下のクイズを生成しました：
      ```json
      {
        "title": "テスト",
        "questions": [
          {
            "question": "水の化学式は？",
            "answers": ["H2O", "CO2", "O2", "N2"],
            "correct": "H2O",
            "type": "multiple_choice"
          }
        ],
        "sourceText": "化学の基礎"
      }
      ```
      ''';

      final result = QuizResponseParser.parse(response);
      expect(result, isNotNull);
      expect(result!.questions[0].correct, equals('H2O'));
    });

    test('空のquestionsはnullを返すこと', () {
      final response = '{"title": "空", "questions": [], "sourceText": ""}';
      final result = QuizResponseParser.parse(response);
      expect(result, isNull);
    });

    test('不正なJSONはnullを返すこと', () {
      final response = 'これはJSONではありません';
      final result = QuizResponseParser.parse(response);
      expect(result, isNull);
    });
  });

  group('StructuralValidator', () {
    test('正解が選択肢に含まれていない場合にcriticalエラーを返すこと', () {
      final question = GeneratedQuestion(
        question: 'テスト問題',
        answers: ['A', 'B', 'C', 'D'],
        correct: 'E',  // 選択肢にない
        type: QuestionType.multipleChoice,
      );

      final errors = StructuralValidator.validateQuestion(question, index: 0);
      expect(errors.any((e) => e.severity == Severity.critical), isTrue);
    });

    test('4択問題で選択肢が3つの場合にcriticalエラーを返すこと', () {
      final question = GeneratedQuestion(
        question: 'テスト問題',
        answers: ['A', 'B', 'C'],  // 3つしかない
        correct: 'A',
        type: QuestionType.multipleChoice,
      );

      final errors = StructuralValidator.validateQuestion(question, index: 0);
      expect(errors.any((e) => e.field.contains('answers')), isTrue);
    });
  });
}
```

決定論的テストのポイントは以下の通りです。

- **パーサーのロバスト性をテストする**: 正常系だけでなく、マークダウン囲み、不正JSON、空データなどのエッジケースを網羅
- **バリデーションルールを個別にテストする**: 「正解が選択肢に含まれるか」「選択肢の数が正しいか」をそれぞれ独立してテスト
- **CI/CDで毎回実行する**: 決定論的テストは高速で安定しているため、プルリクエストごとに実行

## 手法2: 統計的テスト

LLM出力を複数回生成し、品質指標の分布が許容範囲内に収まるかを検証します。

```typescript
// 統計的テスト（TypeScript / Node.js）
import { describe, it, expect } from "vitest";

describe("Quiz Generation Quality", () => {
  // このテストはAPIを呼び出すため、CI/CDでは定期実行（daily）に限定
  it("構造エラー率が5%以下であること", async () => {
    const iterations = 20;
    let structuralErrors = 0;

    for (let i = 0; i < iterations; i++) {
      const result = await generateQuiz(testImageBuffer, {
        quizType: "multiple_choice",
        questionCount: 5,
      });

      if (result.isFailure) {
        structuralErrors++;
        continue;
      }

      const validation = validateQuizStructure(result.value.questions);
      if (validation.some((e) => e.severity === "critical")) {
        structuralErrors++;
      }
    }

    const errorRate = structuralErrors / iterations;
    expect(errorRate).toBeLessThanOrEqual(0.05);
  }, 120_000); // タイムアウト: 2分

  it("ハルシネーション率が10%以下であること", async () => {
    const iterations = 10;
    const sourceText = "テスト用のソーステキスト...";
    let hallucinationCount = 0;

    for (let i = 0; i < iterations; i++) {
      const result = await generateQuiz(testImageBuffer, {
        quizType: "multiple_choice",
        questionCount: 5,
      });

      if (result.isFailure) continue;

      const verification = verifySourceAlignment(
        result.value.questions,
        sourceText,
      );
      hallucinationCount += verification.filter(
        (v) => v.isLikelyHallucination,
      ).length;
    }

    const totalQuestions = iterations * 5;
    const hallucinationRate = hallucinationCount / totalQuestions;
    expect(hallucinationRate).toBeLessThanOrEqual(0.1);
  }, 300_000); // タイムアウト: 5分
});
```

統計的テストの注意点として以下を挙げます。

- **実行回数と信頼性のトレードオフ**: 20回の実行でも統計的に有意とは言い切れないが、コストとのバランスで現実的な回数を設定
- **CI/CDでの実行頻度**: 毎コミットではなく、日次または週次のスケジュール実行が適切
- **閾値の段階的な引き下げ**: 最初は構造エラー率10%をターゲットにし、改善が進んだら5%に引き下げる

## 手法3: ゴールデンデータセット

人手でキュレーションした「正解のクイズセット」を用意し、LLMの生成結果と比較します。

```typescript
// ゴールデンデータセット
const goldenDataset: GoldenQuizSet[] = [
  {
    sourceDescription: "中学理科_光合成",
    sourceImagePath: "test/fixtures/biology_photosynthesis.jpg",
    expectedTopics: ["光合成", "二酸化炭素", "酸素", "葉緑体"],
    expectedDifficulty: 2,  // 中学生レベル
    minimumQuestions: 3,
    forbiddenTopics: ["化学反応式", "ATP"],  // 中学レベルでは不適切
  },
  {
    sourceDescription: "小学算数_分数",
    sourceImagePath: "test/fixtures/math_fractions.jpg",
    expectedTopics: ["分数", "足し算", "引き算"],
    expectedDifficulty: 1,  // 小学生レベル
    minimumQuestions: 3,
    forbiddenTopics: ["微分", "積分", "行列"],
  },
];

describe("Golden Dataset Validation", () => {
  for (const golden of goldenDataset) {
    it(`${golden.sourceDescription}から適切なクイズが生成されること`, async () => {
      const imageBuffer = await readFile(golden.sourceImagePath);
      const result = await generateQuiz(imageBuffer, { questionCount: 5 });

      // 最低問題数の検証
      expect(result.questions.length).toBeGreaterThanOrEqual(
        golden.minimumQuestions,
      );

      // 期待されるトピックが含まれていること
      const allText = result.questions
        .map((q) => q.question + q.answers.join(" "))
        .join(" ");

      const coveredTopics = golden.expectedTopics.filter((topic) =>
        allText.includes(topic),
      );
      const topicCoverage = coveredTopics.length / golden.expectedTopics.length;
      expect(topicCoverage).toBeGreaterThanOrEqual(0.5);

      // 禁止トピックが含まれていないこと
      for (const forbidden of golden.forbiddenTopics) {
        expect(allText).not.toContain(forbidden);
      }
    });
  }
});
```

ゴールデンデータセットの運用ポイントは以下の通りです。

- **多様な教科・学年をカバーする**: 国語、算数、理科、社会、英語 x 小中高 の組み合わせ
- **定期的に更新する**: プロンプト変更や新教科対応に合わせてデータセットも更新
- **禁止トピックの設定**: 学年にそぐわない高度な内容がハルシネーションとして混入していないかの検証

## 手法4: Snapshot Testing の応用

LLMの出力そのものをスナップショットとして保存し、出力の傾向が大きく変わった場合に検出します。完全一致ではなく、構造の類似度やトピックの一貫性をチェックします。

```typescript
// スナップショットベースの回帰テスト
interface OutputSnapshot {
  promptVersion: string;
  generatedAt: Date;
  structure: {
    questionCount: number;
    questionTypes: string[];
    hasExplanations: boolean;
    averageQuestionLength: number;
    averageAnswerLength: number;
  };
  topics: string[];
}

function createSnapshot(quizSet: QuizSet): OutputSnapshot {
  return {
    promptVersion: getCurrentPromptVersion(),
    generatedAt: new Date(),
    structure: {
      questionCount: quizSet.questions.length,
      questionTypes: [...new Set(quizSet.questions.map((q) => q.type))],
      hasExplanations: quizSet.questions.some((q) => q.explanation),
      averageQuestionLength:
        quizSet.questions.reduce((sum, q) => sum + q.question.length, 0) /
        quizSet.questions.length,
      averageAnswerLength:
        quizSet.questions.reduce(
          (sum, q) => sum + q.answers.join("").length / q.answers.length,
          0,
        ) / quizSet.questions.length,
    },
    topics: extractTopics(quizSet),
  };
}

function compareSnapshots(
  current: OutputSnapshot,
  baseline: OutputSnapshot,
): SnapshotDiff {
  const diffs: string[] = [];

  // 問題数の大幅な変動を検出
  const countDiff = Math.abs(
    current.structure.questionCount - baseline.structure.questionCount,
  );
  if (countDiff > 2) {
    diffs.push(
      `問題数が${baseline.structure.questionCount}から${current.structure.questionCount}に変動`,
    );
  }

  // 問題文の平均長さの大幅な変動を検出
  const lengthRatio =
    current.structure.averageQuestionLength /
    baseline.structure.averageQuestionLength;
  if (lengthRatio < 0.5 || lengthRatio > 2.0) {
    diffs.push(`問題文の平均長さが大幅に変動（比率: ${lengthRatio.toFixed(2)}）`);
  }

  return {
    hasDrifted: diffs.length > 0,
    diffs,
  };
}
```

## 手法5: CI/CDでのLLMテスト自動化

上記の手法をCI/CDパイプラインに統合します。

```yaml
# .github/workflows/llm-quality.yml
name: LLM Quality Tests

on:
  pull_request:
    paths:
      - 'lib/services/quiz_generation_service.dart'
      - 'lib/services/gemini_service.dart'
      - 'prompts/**'
  schedule:
    - cron: '0 2 * * *'  # 毎日午前2時（JST 11:00）

jobs:
  deterministic-tests:
    name: Deterministic Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run parser and validator tests
        run: flutter test test/services/quiz_generation_test.dart

  statistical-tests:
    name: Statistical Quality Tests
    runs-on: ubuntu-latest
    if: github.event_name == 'schedule'  # 日次のみ
    steps:
      - uses: actions/checkout@v4
      - name: Run statistical tests
        env:
          GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }}
        run: |
          npm run test:statistical -- --timeout 600000
      - name: Upload quality report
        uses: actions/upload-artifact@v4
        with:
          name: quality-report
          path: reports/quality-*.json

  golden-dataset:
    name: Golden Dataset Validation
    runs-on: ubuntu-latest
    if: github.event_name == 'schedule'  # 日次のみ
    steps:
      - uses: actions/checkout@v4
      - name: Run golden dataset tests
        env:
          GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }}
        run: |
          npm run test:golden
      - name: Notify on failure
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: 'LLM品質テスト失敗',
              body: 'ゴールデンデータセットテストが失敗しました。品質メトリクスを確認してください。',
              labels: ['quality', 'ai']
            })
```

### テスト種別と実行タイミング

| テスト種別 | 実行タイミング | 実行時間 | APIコスト |
|-----------|-------------|---------|----------|
| 決定論的テスト | 毎PR | 数秒 | なし |
| 統計的テスト | 日次 | 5-10分 | $0.5-2 |
| ゴールデンデータ | 日次 | 2-5分 | $0.3-1 |
| スナップショット | プロンプト変更時 | 1-3分 | $0.1-0.5 |

## 品質メトリクスとアラート閾値

テストの結果を品質メトリクスとして収集し、閾値を超えた場合にアラートを発報します。

```typescript
interface QualityMetrics {
  timestamp: Date;
  promptVersion: string;

  // 構造品質
  structuralErrorRate: number;     // 目標: < 2%
  parseFailureRate: number;        // 目標: < 1%

  // コンテンツ品質
  hallucinationRate: number;       // 目標: < 5%
  topicCoverageRate: number;       // 目標: > 60%
  difficultyMatchRate: number;     // 目標: > 70%

  // ユーザー品質（フィードバックループ）
  averageEaseFactor: number;       // 目標: > 2.0
  userDeleteRate: number;          // 目標: < 5%
  userEditRate: number;            // 目標: < 10%
}
```

## まとめ: テスト戦略のピラミッド

LLM出力のテストも、従来のテストピラミッドの考え方が適用できます。

```
           /\
          /  \   ゴールデンデータ + 統計テスト（少数・高コスト・日次）
         /    \
        /------\
       /        \  スナップショット + Confidence Scoring（中程度）
      /          \
     /------------\
    /              \  決定論的テスト：パーサー・バリデーション（大量・低コスト・毎CI）
   /________________\
```

下層（決定論的テスト）は大量に、高速に、低コストで実行します。上層（統計的テスト、ゴールデンデータ）は少数を、定期的に、API呼び出しを伴って実行します。この構成により、コストを抑えながら包括的な品質保証が実現できます。
