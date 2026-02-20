---
title: "ハルシネーション検出の実装パターン"
---

## ハルシネーションはなぜ起きるのか

LLMのハルシネーション（幻覚）とは、モデルが学習データに基づいて「もっともらしいが事実ではない」情報を生成する現象です。教育AIアプリにおいては、教材に書かれていない情報でクイズが作られるケースが該当します。

根本的には、LLMは「次のトークンとして最も確率が高いものを出力する」仕組みであり、出力が事実かどうかを自ら検証する機能を持ちません。つまり、ハルシネーションは現在のLLMアーキテクチャに内在する問題であり、完全な排除はできません。

しかし、検出と軽減は可能です。本章では3つのパターンを実装レベルで解説します。

## パターン1: ソース照合法

最も直接的なアプローチです。生成されたクイズの内容が、元の教材テキストに含まれているかを検証します。

### 基本的な実装

MochiQでは、クイズ生成時にLLMから `sourceText`（画像から抽出されたテキストの要約）を取得しています。この `sourceText` と生成された問題・回答を照合します。

```dart
class SourceVerifier {
  /// ソーステキストとクイズの整合性を検証
  static SourceVerificationResult verify({
    required GeneratedQuizSet quizSet,
    required String sourceText,
  }) {
    final results = <QuestionVerification>[];
    final sourceNormalized = _normalize(sourceText);
    final sourceTokens = _tokenize(sourceNormalized);

    for (final question in quizSet.questions) {
      // 正解がソーステキストに含まれているか
      final correctNormalized = _normalize(question.correct);
      final correctInSource = _containsAnswer(sourceTokens, correctNormalized);

      // 問題文のキーワードがソーステキストに含まれているか
      final questionKeywords = _extractKeywords(question.question);
      final keywordOverlap = _calculateOverlap(sourceTokens, questionKeywords);

      final confidence = _calculateConfidence(correctInSource, keywordOverlap);

      results.add(QuestionVerification(
        question: question,
        correctInSource: correctInSource,
        keywordOverlap: keywordOverlap,
        confidence: confidence,
        isLikelyHallucination: confidence < 0.3,
      ));
    }

    return SourceVerificationResult(verifications: results);
  }

  /// テキストの正規化
  static String _normalize(String text) {
    return text
        .replaceAll(RegExp(r'[\s　]+'), ' ')    // 空白の正規化
        .replaceAll(RegExp(r'[（(].*?[）)]'), '') // 括弧内の読み仮名を除去
        .replaceAll(RegExp(r'[、。，．]'), '')    // 句読点の除去
        .toLowerCase()
        .trim();
  }

  /// トークン分割（日本語対応）
  static Set<String> _tokenize(String text) {
    final tokens = <String>{};
    // N-gram（2-gram, 3-gram）でトークン化
    for (var n = 2; n <= 4; n++) {
      for (var i = 0; i <= text.length - n; i++) {
        tokens.add(text.substring(i, i + n));
      }
    }
    // 単語単位でも追加
    tokens.addAll(text.split(' ').where((t) => t.isNotEmpty));
    return tokens;
  }

  /// ソーステキストに回答が含まれるか検証
  static bool _containsAnswer(Set<String> sourceTokens, String answer) {
    if (answer.length <= 2) {
      // 短い回答（○、×、数字など）はソース照合をスキップ
      return true;
    }
    final answerTokens = _tokenize(answer);
    final overlap = answerTokens.intersection(sourceTokens);
    return overlap.length / answerTokens.length > 0.5;
  }

  /// キーワードの重複率を計算
  static double _calculateOverlap(
    Set<String> sourceTokens,
    Set<String> keywords,
  ) {
    if (keywords.isEmpty) return 0;
    final overlap = keywords.intersection(sourceTokens);
    return overlap.length / keywords.length;
  }

  /// 信頼度スコアを算出
  static double _calculateConfidence(bool correctInSource, double keywordOverlap) {
    double score = 0;
    if (correctInSource) score += 0.5;
    score += keywordOverlap * 0.5;
    return score.clamp(0.0, 1.0);
  }

  /// 問題文からキーワードを抽出
  static Set<String> _extractKeywords(String question) {
    // 助詞・助動詞・疑問詞を除外した意味語を抽出
    final stopWords = {'は', 'が', 'の', 'を', 'に', 'で', 'と', 'か', 'も',
                       'する', 'ある', 'いる', 'なる', 'できる',
                       'どれ', 'どの', '何', 'いつ', 'どこ', 'なぜ', 'どう',
                       '次', 'うち', '正しい', '選び', 'なさい', 'ください'};
    final words = question.split(RegExp(r'[\s、。？?！!]+'));
    return words.where((w) => w.length >= 2 && !stopWords.contains(w)).toSet();
  }
}
```

### 回答の正規化が鍵

ソース照合で最もハマるのが、テキスト表現の揺れです。以下のようなケースでは、単純な文字列一致では検出できません。

| ソーステキスト | 生成された回答 | 一致判定 |
|-------------|-------------|---------|
| 「エビングハウス」 | 「ヘルマン・エビングハウス」 | 部分一致で対応 |
| 「1185年」 | 「1185」 | 数値の正規化が必要 |
| 「二酸化炭素(にさんかたんそ)」 | 「二酸化炭素」 | 括弧内除去で対応 |
| 「ＮＡＴＯの設立」 | 「NATOの設立」 | 全角/半角の正規化 |

```dart
/// 回答テキストの正規化
class AnswerNormalizer {
  static String normalize(String text) {
    var result = text;

    // 全角英数字 → 半角
    result = result.replaceAllMapped(
      RegExp(r'[Ａ-Ｚａ-ｚ０-９]'),
      (m) => String.fromCharCode(m.group(0)!.codeUnitAt(0) - 0xFEE0),
    );

    // 括弧内の読み仮名を除去
    result = result.replaceAll(RegExp(r'[（(][ぁ-ん、]+[）)]'), '');

    // 全角スペース → 半角
    result = result.replaceAll('　', ' ');

    // 前後の空白除去
    result = result.trim();

    return result;
  }
}
```

## パターン2: Cross-validation（交差検証）

同じ教材からクイズを複数回生成し、結果の一貫性を検証するパターンです。ハルシネーションは非決定論的に発生するため、複数回の生成で再現しない情報はハルシネーションの可能性が高いと判断できます。

```typescript
// Cross-validation パターン（TypeScript版）
interface CrossValidationResult {
  question: string;
  consistencyScore: number;
  appearances: number;
  totalGenerations: number;
  isConsistent: boolean;
}

async function crossValidateQuiz(
  images: Buffer[],
  generationCount: number = 3,
): Promise<CrossValidationResult[]> {
  // 同じ入力で複数回生成
  const generations: QuizQuestion[][] = [];
  for (let i = 0; i < generationCount; i++) {
    const result = await generateQuiz(images);
    generations.push(result.questions);
  }

  // 全生成結果の問題をフラットに収集
  const allQuestions = generations.flat();

  // 問題文の類似度でクラスタリング
  const clusters = clusterBySimilarity(allQuestions, threshold: 0.7);

  // 各クラスタの一貫性を評価
  return clusters.map((cluster) => {
    const appearances = cluster.length;
    const consistencyScore = appearances / generationCount;

    // 正解の一致も確認
    const correctAnswers = new Set(cluster.map((q) => q.correct));
    const answerConsistency = 1 / correctAnswers.size;

    return {
      question: cluster[0].question,
      consistencyScore: consistencyScore * answerConsistency,
      appearances,
      totalGenerations: generationCount,
      isConsistent: consistencyScore >= 0.66 && answerConsistency === 1,
    };
  });
}

function clusterBySimilarity(
  questions: QuizQuestion[],
  threshold: number,
): QuizQuestion[][] {
  const clusters: QuizQuestion[][] = [];

  for (const question of questions) {
    let assigned = false;
    for (const cluster of clusters) {
      const similarity = calculateTextSimilarity(
        question.question,
        cluster[0].question,
      );
      if (similarity >= threshold) {
        cluster.push(question);
        assigned = true;
        break;
      }
    }
    if (!assigned) {
      clusters.push([question]);
    }
  }

  return clusters;
}
```

### Cross-validationのトレードオフ

Cross-validationは検出精度が高い反面、API呼び出し回数が増えるため、コストとレイテンシが倍増します。以下の場面での使用を推奨します。

- **入試対策や資格試験の教材**: 品質が最重要で、コスト増が許容される場合
- **バッチ処理**: リアルタイム性が求められない、夜間バッチでの品質検証
- **品質モニタリング**: サンプリングで一定割合のクイズをCross-validationする

通常のユーザー操作（カメラ撮影→即座にクイズ表示）では、レイテンシの制約からCross-validationは適用困難です。その場合はパターン1のソース照合を主体とし、パターン3のConfidence scoringを補助的に使います。

## パターン3: Confidence Scoring（確信度スコア）

LLMの生成結果に対して、別のLLM呼び出しで確信度を評価するパターンです。「セルフレビュー」とも呼ばれます。

```dart
class ConfidenceScorer {
  final GenerativeModel _reviewModel;

  ConfidenceScorer()
      : _reviewModel = GenerativeModel(
          model: 'gemini-2.0-flash-exp',
          apiKey: Environment.geminiApiKey,
          generationConfig: GenerationConfig(
            temperature: 0.1,  // レビューは一貫性重視
            maxOutputTokens: 1024,
          ),
        );

  /// 生成されたクイズの確信度を評価
  Future<List<QuestionConfidence>> scoreConfidence({
    required List<GeneratedQuestion> questions,
    required String sourceText,
  }) async {
    final prompt = '''
以下のクイズが、ソーステキストの内容に基づいて正しく作成されているか評価してください。

ソーステキスト:
"""
$sourceText
"""

クイズ:
${questions.asMap().entries.map((e) => '''
問題${e.key + 1}: ${e.value.question}
正解: ${e.value.correct}
選択肢: ${e.value.answers.join(', ')}
''').join('\n')}

各問題について、以下のJSON形式で評価してください:
{
  "evaluations": [
    {
      "questionIndex": 0,
      "confidence": 0.95,
      "issues": [],
      "reasoning": "ソーステキストに明記されている内容"
    },
    {
      "questionIndex": 1,
      "confidence": 0.3,
      "issues": ["hallucination"],
      "reasoning": "ソーステキストに該当する記述がない"
    }
  ]
}

confidenceは0.0-1.0で、1.0が最も確信度が高いことを意味します。
issuesには該当するものを含めてください: hallucination, ambiguous, wrong_answer, difficulty_mismatch
''';

    final response = await _reviewModel.generateContent([Content.text(prompt)]);
    return _parseConfidenceScores(response.text!);
  }

  List<QuestionConfidence> _parseConfidenceScores(String responseText) {
    try {
      final jsonMatch = RegExp(r'\{[\s\S]*\}', dotAll: true)
          .firstMatch(responseText);
      if (jsonMatch == null) return [];

      final data = json.decode(jsonMatch.group(0)!) as Map<String, dynamic>;
      final evaluations = data['evaluations'] as List;

      return evaluations.map((e) => QuestionConfidence(
        questionIndex: e['questionIndex'] as int,
        confidence: (e['confidence'] as num).toDouble(),
        issues: List<String>.from(e['issues'] as List),
        reasoning: e['reasoning'] as String? ?? '',
      )).toList();
    } catch (e) {
      debugPrint('Confidence score parse error: $e');
      return [];
    }
  }
}
```

### 確信度スコアの活用方法

確信度スコアの閾値に応じて、以下のアクションを取ります。

| 確信度 | アクション |
|-------|----------|
| 0.8以上 | そのまま出題 |
| 0.5-0.8 | 出題するが、ユーザーに「AI生成」マークを強調表示 |
| 0.3-0.5 | 出題せず、別の問題で差し替え |
| 0.3未満 | 棄却。ログに記録してプロンプト改善の材料に |

## 実際にMochiQで遭遇したハルシネーション事例

MochiQの運用で実際に観測した代表的なハルシネーション事例を紹介します。

### 事例1: 教科外の知識の混入

入力: 中学の英語教科書の写真（"I have a pen."を含むページ）
生成: 「"I have a pen."はどの文法パターンか？→ SVO / SVC / SVOO / SVOC」

教科書にはSVOなどの文法用語は書かれておらず、LLMが自身の知識から補完してしまった事例です。ソース照合法で「SVO」「SVC」等の用語がソーステキストに含まれないことを検出し、フィルタリングできました。

### 事例2: 数値の微妙な改変

入力: 「日本の面積は約37.8万km2」
生成: 「日本の面積は約何万km2か？→ 37.5 / 37.8 / 38.0 / 38.5」

正解は37.8で問題ないのですが、不正解の選択肢に「37.5」が含まれています。これ自体はハルシネーションではありませんが、数値の微妙な改変は教育コンテンツでは混乱を招きます。このパターンは、不正解の選択肢が正解に近すぎないかをチェックする「誤答品質検証」で対処します。

### 事例3: 同義語による誤判定

入力: 「エビングハウスの忘却曲線」に関する記述
生成: 「忘却曲線を発見した心理学者は？→ エビングハウス」

ソーステキストには「ヘルマン・エビングハウス」と記載されていたため、「エビングハウス」との単純な文字列一致では不一致と判定されてしまいました。この問題は回答の正規化（部分一致検出、同義語辞書）で解決しています。

## まとめ: ハルシネーション対策の組み合わせ

単一の手法ではハルシネーションを完全に検出することはできません。複数のパターンを組み合わせることで、検出率を高めます。

| パターン | 検出できるもの | コスト | レイテンシ |
|---------|-------------|-------|----------|
| ソース照合 | ソース外情報の混入 | 低（API不要） | 低 |
| Cross-validation | 非一貫的な生成内容 | 高（N回のAPI呼び出し） | 高 |
| Confidence Scoring | 全般的な品質問題 | 中（1回のAPI呼び出し） | 中 |

リアルタイム生成ではソース照合を必須とし、重要なコンテンツにはConfidence Scoringを追加する。Cross-validationはバッチ処理や品質モニタリングで活用する、というのが実用的な組み合わせです。
