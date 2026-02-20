---
title: "AIが生成するクイズの品質をどう担保するか — EdTech開発者のための実践ガイド"
emoji: "✅"
type: "tech"
topics: ["AI", "品質保証", "EdTech", "LLM", "テスト"]
published: true
published_at: 2026-03-06 08:30
---

## はじめに：AI生成コンテンツの「正しさ」は誰が保証するのか

LLMを使えば、教材からクイズを自動生成することは技術的に難しくありません。Gemini APIやOpenAI APIにテキストを投げて「クイズを作って」と頼めば、それらしい問題と選択肢が返ってきます。

問題はその先です。

**返ってきたクイズが「正しい」かどうかを、どう保証するか。**

AI学習アプリ「MochiQ」の開発を通じて、私たちはこの問いに正面から向き合いました。MochiQは教材の写真を撮影するだけでAIがクイズを自動生成するアプリです。SM-2アルゴリズムによる間隔反復学習と組み合わせて、効率的な記憶定着を実現します。

しかし、学習アプリで提供するクイズに誤りがあれば、ユーザーは間違った知識を「正しい」と思い込んで覚えてしまう。これはSNSの投稿やチャットボットの回答とは次元の異なるリスクです。

本記事では、AI生成クイズの品質を担保するために私たちが構築したバリデーションパイプラインの設計と実装を、コード例を交えて解説します。EdTechに限らず、LLMの出力品質を保証する必要があるすべてのプロダクトに応用できる内容です。

## AI生成クイズで発生する品質問題の分類

実際の運用で遭遇した品質問題を整理すると、大きく4つのカテゴリに分類できます。

### 1. Hallucination（事実と異なる内容の生成）

最も深刻な問題です。LLMが教材に書かれていない「もっともらしいが誤った情報」を生成するケースです。

**実例：**
- 教材に「鎌倉幕府は1185年に成立」と書かれているのに、クイズの正解が「1192年」になっている
- 化学式の問題で、存在しない化合物を選択肢に含めている
- 英単語の意味を微妙に間違えた選択肢を「正解」としている

LLMは教材のテキストを「参考にしつつ」、学習データに基づいて回答を生成します。教材と学習データの間に矛盾がある場合、学習データ側の知識が優先されることがある。これがHallucinationの主な原因です。

### 2. 構造的な不整合

JSONレスポンスのパース段階で発覚する問題です。

- 正解が選択肢のリストに含まれていない
- 4択問題なのに選択肢が3つしかない
- 同一の選択肢が重複している
- 問題文が空、または意味をなさない

### 3. 難易度の不適切さ

教材の内容と生成されるクイズの難易度が一致しない問題です。

- 小学校3年生向けの教材から、高校レベルの専門用語を使った問題が生成される
- 逆に、高校生向けの教材から「○か×か」レベルの単純すぎる問題しか生成されない

### 4. 曖昧な問題文

正解が一意に定まらない問題が生成されるケースです。

- 「日本で最も有名な山は？」のような主観的な問い
- 複数の正解がありえる問題で、選択肢の1つだけを正解としている

## バリデーションパイプラインの全体設計

これらの問題に対処するため、私たちは3層のバリデーションパイプラインを設計しました。

```
[Gemini API] → [Layer 1: 構造バリデーション] → [Layer 2: 内容バリデーション] → [Layer 3: フィードバックループ]
                     ↓ NG                          ↓ NG                         ↓ 報告
                  再生成 or エラー               問題を除外                    品質改善に反映
```

それぞれの層の責務を明確に分離することで、保守性と拡張性を確保しています。

## Layer 1：構造バリデーション — JSONレスポンスの検証

まず、LLMの出力が期待するデータ構造に合致しているかを検証します。これはプログラムで完全に自動化できるレイヤーです。

### レスポンスパーサーの実装

MochiQでは、Gemini APIからのレスポンスをパースする専用クラスを用意しています。

```dart
class QuizResponseParser {
  static GeneratedQuizSet? parse(String response) {
    try {
      String jsonString = response;

      // Markdownコードブロック内のJSONを抽出
      final codeBlockMatch =
          RegExp(r'```(?:json)?\s*([\s\S]*?)```').firstMatch(response);
      if (codeBlockMatch != null) {
        jsonString = codeBlockMatch.group(1)!.trim();
      } else {
        // レスポンス内のJSONオブジェクトを検出
        final jsonMatch = RegExp(r'\{[\s\S]*\}').firstMatch(response);
        if (jsonMatch != null) {
          jsonString = jsonMatch.group(0)!;
        }
      }

      final jsonData = json.decode(jsonString) as Map<String, dynamic>;

      // 必須フィールドの検証
      if (!jsonData.containsKey('questions') ||
          (jsonData['questions'] as List).isEmpty) {
        return null;
      }

      return GeneratedQuizSet.fromJson(jsonData);
    } catch (e) {
      debugPrint('Failed to parse quiz response: $e');
      return null;
    }
  }
}
```

ここで重要なのは、LLMのレスポンスが「純粋なJSON」であるとは限らない点です。Markdownのコードブロックで囲まれたり、前後に説明文がついたりするケースがあるため、正規表現でJSON部分を抽出する処理が不可欠です。

### 構造バリデーションのチェック項目

パース成功後、個々の問題に対して以下の構造チェックを実行します。

```typescript
interface QuizQuestion {
  question: string;
  answers: string[];
  correct: string;
  type: "multiple_choice" | "true_false";
  explanation?: string;
}

function validateQuestionStructure(q: QuizQuestion): ValidationResult {
  const errors: string[] = [];

  // 1. 問題文の存在チェック
  if (!q.question || q.question.trim().length === 0) {
    errors.push("問題文が空です");
  }

  // 2. 選択肢数の検証
  if (q.type === "multiple_choice" && q.answers.length !== 4) {
    errors.push(`4択問題の選択肢が${q.answers.length}個です`);
  }
  if (q.type === "true_false" && q.answers.length !== 2) {
    errors.push(`○×問題の選択肢が${q.answers.length}個です`);
  }

  // 3. 正解が選択肢に含まれているか
  if (!q.answers.includes(q.correct)) {
    errors.push("正解が選択肢に含まれていません");
  }

  // 4. 選択肢の重複チェック
  const uniqueAnswers = new Set(q.answers);
  if (uniqueAnswers.size !== q.answers.length) {
    errors.push("選択肢に重複があります");
  }

  // 5. 選択肢の長さチェック（空文字の排除）
  if (q.answers.some((a) => a.trim().length === 0)) {
    errors.push("空の選択肢があります");
  }

  return {
    isValid: errors.length === 0,
    errors,
  };
}
```

構造バリデーションで不合格となった問題は、クイズセットから除外します。除外後に問題数がゼロになった場合は、ユーザーにエラーを表示して再生成を促します。

MochiQの実装では、構造バリデーションの不合格率は全体の約3〜5%でした。割合は低いですが、このチェックがなければユーザーに壊れた問題が表示されることになります。

## Layer 2：内容バリデーション — 品質の検証

構造的に正しくても、内容が不適切なクイズは存在します。このレイヤーでは、クイズの「中身」を検証します。

### 正解と選択肢の関係性チェック

最も効果的だったのが、正解と不正解選択肢の「距離」を検証するアプローチです。

```typescript
function validateAnswerQuality(q: QuizQuestion): ValidationResult {
  const errors: string[] = [];
  const warnings: string[] = [];

  // 1. 正解が極端に短い/長い場合を警告
  const avgLength =
    q.answers.reduce((sum, a) => sum + a.length, 0) / q.answers.length;
  const correctLength = q.correct.length;

  if (correctLength > avgLength * 2.5 || correctLength < avgLength * 0.3) {
    warnings.push("正解の長さが他の選択肢と著しく異なります");
    // 長さで正解が推測できてしまう問題
  }

  // 2. 「すべて正解」パターンの検出
  //    選択肢が「AとB」「BとC」「AとC」「AとBとC」のように
  //    包含関係にある場合を検出
  const containsAll = q.answers.some(
    (a) => q.answers.filter((b) => a !== b).every((b) => a.includes(b))
  );
  if (containsAll) {
    warnings.push("選択肢間に包含関係があります");
  }

  // 3. ○×問題の正解偏りチェック（バッチ単位）
  //    これはクイズセット全体で検証

  return {
    isValid: errors.length === 0,
    errors,
    warnings,
  };
}
```

### 難易度の妥当性チェック

MochiQではクイズ生成時にユーザーの学年情報をプロンプトに含めています。

```dart
static String build({
  required String quizType,
  required int questionCount,
  String? gradeLevel,
}) {
  final gradeInstruction = gradeLevel != null
      ? '対象学年: $gradeLevel の生徒向けに適切な難易度で出題してください。\n'
      : '';

  return '''
画像に含まれるテキストを読み取り、そのテキストの内容に基づいてクイズを生成してください。

$gradeInstruction
問題形式: ...
問題数: $questionCount問
...
''';
}
```

しかし、プロンプトで指示するだけでは難易度が適切になる保証はありません。そこで、生成後に以下のヒューリスティクスで難易度の妥当性を検証します。

```typescript
function estimateDifficulty(q: QuizQuestion): "easy" | "medium" | "hard" {
  let score = 0;

  // 問題文の長さ（長い問題は一般的に難しい）
  if (q.question.length > 100) score += 2;
  else if (q.question.length > 50) score += 1;

  // 選択肢間の類似度（似ている選択肢ほど難しい）
  const similarities = calculatePairwiseSimilarity(q.answers);
  if (similarities.average > 0.7) score += 2;
  else if (similarities.average > 0.4) score += 1;

  // 専門用語の含有率
  const technicalTermRatio = countTechnicalTerms(q.question) / q.question.length;
  if (technicalTermRatio > 0.3) score += 2;

  if (score >= 4) return "hard";
  if (score >= 2) return "medium";
  return "easy";
}
```

推定された難易度が対象学年に対して不適切な場合（小学3年生に `hard` レベルが多い等）、その問題にフラグを立てます。ただし、この段階では自動除外ではなくログ記録にとどめ、後述のフィードバックループで統計的に改善していく方針を取りました。

## Layer 3：フィードバックループ — ユーザー行動からの品質改善

バリデーションの最終層は、実際のユーザー行動データに基づく品質検証です。プログラムだけでは検出できない問題を、ユーザーの回答パターンから発見します。

### 回答データから品質問題を検出する

MochiQではSM-2アルゴリズムで復習スケジュールを管理しています。このデータには品質のシグナルが含まれています。

```dart
int getQualityFromResult({
  required bool isCorrect,
  int? responseTimeMs,
}) {
  if (!isCorrect) {
    return 2; // 不正解は固定で2
  }

  if (responseTimeMs == null) {
    return 4; // 時間情報なしはデフォルト4
  }

  if (responseTimeMs <= 5000) {
    return 5; // 高速回答 = 確信あり
  } else if (responseTimeMs <= 10000) {
    return 4; // 中速回答 = やや迷い
  } else {
    return 3; // 低速回答 = かなり迷い
  }
}
```

この品質スコアを集計すると、以下の異常パターンを検出できます。

**パターンA：全員が間違える問題**

ある問題の正答率が極端に低い場合（例：10%以下）、その問題自体に品質問題がある可能性が高い。正解が間違っているか、問題文が曖昧かのどちらかです。

```typescript
function detectProblematicQuestions(
  questionStats: Map<string, QuestionStats>
): string[] {
  const problematic: string[] = [];

  for (const [questionId, stats] of questionStats) {
    // 十分なサンプル数がある場合のみ判定
    if (stats.totalAttempts < 10) continue;

    // 正答率が極端に低い
    if (stats.correctRate < 0.1) {
      problematic.push(questionId);
      continue;
    }

    // 正答率が極端に高い（簡単すぎる = 学習効果が低い）
    if (stats.correctRate > 0.98 && stats.averageResponseTime < 2000) {
      problematic.push(questionId);
    }
  }

  return problematic;
}
```

**パターンB：回答時間の異常**

正解しているのに回答時間が異常に長い問題は、問題文が曖昧で迷わせている可能性があります。逆に、全員が瞬時に正解する問題は、正解が明白すぎて学習効果がない可能性があります。

**パターンC：ユーザーからの直接フィードバック**

MochiQでは、生成されたクイズを保存前にプレビュー画面で確認・編集できる仕組みを提供しています。ユーザーがどの問題を編集したか、どの問題を削除したかのデータも、品質改善の重要なシグナルになります。

## プロンプトエンジニアリング：品質問題を「発生させない」ための工夫

バリデーションは「発生した問題を検出する」仕組みですが、プロンプトの設計で「問題を発生させにくくする」ことも同様に重要です。

### 出力形式の厳密な指定

あいまいな指示は、あいまいな出力を生みます。MochiQでは、期待するJSONの完全な構造をプロンプトに含めています。

```
以下のJSON形式で回答してください：
{
  "title": "クイズのタイトル（内容に基づいて自動生成）",
  "questions": [
    {
      "question": "問題文",
      "answers": ["選択肢1", "選択肢2", "選択肢3", "選択肢4"],
      "correct": "正解の選択肢",
      "type": "multiple_choice",
      "explanation": "解説（任意）"
    }
  ],
  "sourceText": "画像から抽出したテキストの要約"
}
```

さらに、制約条件を明示的に列挙します。

```
重要な指示：
- 必ず上記のJSON形式のみで回答してください
- 正解は必ず選択肢の中から選んでください
- 4択問題の場合、選択肢は必ず4つにしてください
- ○×問題の場合、answersは["○", "×"]にしてください
```

一見冗長に感じるかもしれませんが、「正解は必ず選択肢の中から選んでください」という指示を入れるだけで、正解と選択肢の不一致問題は大幅に減少しました。

### Temperature設定の最適化

クイズ生成では、創造性よりも正確性が重要です。MochiQではTemperatureを低めに設定しています。

```dart
_model = GenerativeModel(
  model: 'gemini-2.0-flash-exp',
  apiKey: _apiKey,
  generationConfig: GenerationConfig(
    temperature: 0.3,  // 創造性よりも一貫性を優先
    topP: 0.9,
    topK: 20,
    maxOutputTokens: 4096,
  ),
);
```

Temperature 0.3は経験的に最適だった値です。0.1まで下げると選択肢のバリエーションが乏しくなり、0.7以上に上げるとHallucinationの発生率が上がりました。

ここで注意すべきは、学習分析のような自由度が求められるタスクでは、異なるTemperature設定を使い分けている点です。

```dart
// 学習分析用 — より一貫性のある出力が求められる
GenerationConfig(
  temperature: 0.1,
  topP: 0.8,
  topK: 10,
  maxOutputTokens: 2048,
)
```

タスクの性質に応じてモデルパラメータを使い分けることが、全体の品質を引き上げるポイントです。

### ソーステキストの参照指示

Hallucinationを抑制する最も効果的な手法は、「教材に書かれている内容からのみ出題すること」を明示的に指示することです。

```
画像に含まれるテキストを読み取り、**そのテキストの内容に基づいて**クイズを生成してください。
```

「そのテキストの内容に基づいて」という制約を入れることで、LLMが学習データから勝手に情報を補完する傾向を抑制できます。完全にゼロにはなりませんが、この指示の有無で体感的にHallucination率が大きく変わりました。

## 実際に遭遇した品質問題と対策

開発中に遭遇した具体的な品質問題と、それに対してどう対処したかを紹介します。

### 問題1：正解が選択肢に含まれない

**現象：** 正解フィールドの文字列が、選択肢の文字列と微妙に異なるケースが頻発。

```json
{
  "answers": ["葉緑体", "ミトコンドリア", "核", "リボソーム"],
  "correct": "葉緑体（ようりょくたい）"
}
```

正解に読み仮名が付加されており、完全一致で比較すると不一致になります。

**対策：** 正規化処理を挟んだ上で比較するバリデーションを追加。

```typescript
function normalizeAnswer(answer: string): string {
  return answer
    .replace(/（.*?）/g, "")    // 括弧内の補足を除去
    .replace(/\(.*?\)/g, "")    // 半角括弧も対応
    .replace(/\s+/g, "")        // 空白を除去
    .trim();
}

function isCorrectInAnswers(q: QuizQuestion): boolean {
  const normalizedCorrect = normalizeAnswer(q.correct);
  return q.answers.some(
    (a) => normalizeAnswer(a) === normalizedCorrect
  );
}
```

### 問題2：○×問題の正解偏り

**現象：** 生成されるクイズセット内の○×問題で、正解が「○」に偏る傾向があった。5問中4問が「○」で正解というケースが多発。

**対策：** クイズセット全体の○×バランスを検証し、偏りが大きい場合はプロンプトを調整して再生成。

```typescript
function checkTrueFalseBalance(questions: QuizQuestion[]): boolean {
  const tfQuestions = questions.filter((q) => q.type === "true_false");
  if (tfQuestions.length < 2) return true; // チェック不要

  const trueCount = tfQuestions.filter((q) => q.correct === "○").length;
  const ratio = trueCount / tfQuestions.length;

  // 30%〜70%の範囲に収まっていればOK
  return ratio >= 0.3 && ratio <= 0.7;
}
```

### 問題3：画像のOCR品質による連鎖的な品質低下

**現象：** ぼやけた画像や影のかかった画像から生成されたクイズは、問題文自体が意味不明になるケースがあった。

**対策：** クイズ生成の前段階で画像品質チェックを実装。

```dart
class ImageQualityChecker {
  static const int _minFileSize = 100 * 1024;   // 100KB
  static const int _warnFileSize = 20 * 1024 * 1024; // 20MB
  static const int _maxFileSize = 40 * 1024 * 1024;  // 40MB

  static Future<ImageQualityResult> checkFile(File file) async {
    final warnings = <ImageQualityWarning>[];
    var isAcceptable = true;

    // フォーマットチェック
    final pathResult = checkImagePath(file.path);
    warnings.addAll(pathResult.warnings);
    if (!pathResult.isAcceptable) isAcceptable = false;

    // ファイルサイズチェック
    try {
      final size = await file.length();
      final sizeResult = checkFileSize(size);
      warnings.addAll(sizeResult.warnings);
      if (!sizeResult.isAcceptable) isAcceptable = false;
    } catch (_) {}

    return ImageQualityResult(
      isAcceptable: isAcceptable,
      warnings: warnings,
    );
  }
}
```

入力の品質を保証することが、出力の品質保証の第一歩です。ゴミを入れれば、ゴミが出てくる（Garbage In, Garbage Out）。この原則はLLMでもまったく同じです。

### 問題4：タイムアウトによる不完全なレスポンス

**現象：** ネットワーク遅延やAPIの負荷により、JSONの途中でレスポンスが切れるケースがあった。途切れたJSONをパースすると予期しないエラーになる。

**対策：** タイムアウト処理と指数バックオフリトライを実装。

```dart
// タイムアウト付きAPI呼び出し
final response = await _model
    .generateContent([
      Content.multi([
        TextPart(prompt),
        ...imageParts,
      ])
    ])
    .timeout(const Duration(seconds: 30));
```

```dart
class RetryPolicy {
  final int maxAttempts;
  final Duration initialDelay;
  final double backoffMultiplier;

  Duration getDelayForAttempt(int attempt) {
    final delayMs = initialDelay.inMilliseconds *
        pow(backoffMultiplier, attempt).toInt();
    return Duration(
      milliseconds: min(delayMs, maxDelay.inMilliseconds),
    );
  }

  bool shouldRetry(int currentAttempt) {
    return currentAttempt < maxAttempts;
  }
}
```

## 品質メトリクスの計測と監視

品質保証は「実装して終わり」ではありません。継続的に計測し、改善し続ける仕組みが必要です。

### 追跡すべきメトリクス

| メトリクス | 計測方法 | アラート条件 |
|-----------|---------|------------|
| 構造バリデーション不合格率 | パース失敗数 / 総生成数 | > 10% |
| 正解不一致率 | 正解が選択肢にない問題数 / 総問題数 | > 5% |
| ユーザー編集率 | 生成後に編集された問題数 / 総問題数 | > 30% |
| ユーザー削除率 | 生成後に削除された問題数 / 総問題数 | > 20% |
| 極端な正答率の問題数 | 正答率 < 10% or > 98% の問題数 | 増加傾向 |
| 平均回答時間の異常値 | 全体平均の3倍以上かかる問題数 | 増加傾向 |

これらのメトリクスを定期的にレビューし、異常値が検出された場合はプロンプトの調整やバリデーションルールの追加で対処します。

### コスト意識

品質チェックを厚くすればするほど、API呼び出し回数とレイテンシは増えます。すべてのチェックを毎回実行するのではなく、問題の重大度に応じて層を分けることが重要です。

- **必須チェック（Layer 1）:** 構造バリデーション。API呼び出し不要。全リクエストに適用
- **推奨チェック（Layer 2）:** 内容バリデーション。ローカルで実行可能なものに限定
- **統計的チェック（Layer 3）:** 十分なデータが蓄積された後にバッチ処理で実行

## まとめ：AI品質保証の設計原則

AI生成コンテンツの品質保証で学んだ教訓を整理します。

**1. LLMの出力を信頼しない**

LLMは非常に有用なツールですが、出力の正しさを保証するものではありません。特に学習コンテンツのように「間違いが直接的な害を生む」ドメインでは、多層的なバリデーションが不可欠です。

**2. 入力の品質を保証する**

出力の品質は入力の品質に依存します。画像品質チェック、テキスト抽出の精度検証、適切な前処理。LLMに渡す前のパイプラインに投資することが、結果的に最もコスト効率の良い品質改善策です。

**3. プロンプトは「仕様書」として設計する**

「クイズを作って」ではなく、出力形式、制約条件、禁止事項を明示的に記述する。プロンプトの設計品質が、バリデーションの負荷を直接的に左右します。

**4. ユーザー行動データを品質シグナルとして活用する**

プログラムだけでは検出できない品質問題がある。実際のユーザーの回答パターン、編集行動、フィードバックを品質改善のループに組み込むことで、時間とともに品質は向上します。

**5. コストと品質のバランスを意識する**

完璧な品質保証を目指すと、レイテンシとコストが際限なく増大します。「許容可能なリスク」を定義し、段階的なチェック体制を敷くことが現実的なアプローチです。

---

MochiQは、教材を撮影するだけでAIがクイズを自動生成し、SM-2アルゴリズムで最適な復習タイミングを管理する学習アプリです。本記事で解説した品質保証の仕組みにより、信頼性の高い学習体験を提供しています。

https://mochiq.joinclass.jp
