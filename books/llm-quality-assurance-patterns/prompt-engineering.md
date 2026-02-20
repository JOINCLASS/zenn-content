---
title: "クイズ生成に特化したプロンプトエンジニアリング"
---

## プロンプトがLLMアプリの品質を決める

LLMアプリの品質保証において、プロンプト設計は最もレバレッジの高い施策です。バリデーションやエラーハンドリングは「問題が起きた後の対処」ですが、プロンプトの改善は「問題の発生そのものを減らす」ことができます。

本章では、MochiQのクイズ生成で実際に使用しているプロンプト設計のパターンを解説します。

## パターン1: 出力フォーマットの厳密な指定

LLMに自由形式で出力させると、構造的不整合の温床になります。出力フォーマットは可能な限り厳密に指定します。

### MochiQのプロンプト設計

```dart
class QuizGenerationPrompt {
  static String build({
    required String quizType,
    required int questionCount,
    String? gradeLevel,
  }) {
    final typeInstruction = _getTypeInstruction(quizType);
    final gradeInstruction = gradeLevel != null
        ? '対象学年: $gradeLevel の生徒向けに適切な難易度で出題してください。\n'
        : '';

    return '''
画像に含まれるテキストを読み取り、そのテキストの内容に基づいてクイズを生成してください。

$gradeInstruction
問題形式: $typeInstruction
問題数: $questionCount問

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

重要な指示：
- 必ず上記のJSON形式のみで回答してください
- 問題文と選択肢は日本語で作成してください
- 正解は必ず選択肢の中から選んでください
- 4択問題の場合、選択肢は必ず4つにしてください
- ○×問題の場合、answersは["○", "×"]にしてください
- explanationは省略可能ですが、あると学習効果が高まります
''';
  }
}
```

このプロンプトにはいくつかの設計上の工夫があります。

1. **JSONのサンプルを含める**: 「JSON形式で」と指示するだけでなく、実際の構造をサンプルとして示す
2. **制約条件を明示する**: 「正解は必ず選択肢の中から選ぶ」「選択肢は必ず4つ」など、構造バリデーションで検出される問題を先回りして防ぐ
3. **ソーステキストの抽出を要求する**: `sourceText` フィールドにより、AIが画像からどのテキストを読み取ったかを確認できる（ハルシネーション検出の手がかり）

### 2026年のベストプラクティス: Structured Output

2026年現在、Gemini APIとOpenAI APIはStructured Output（構造化出力）をサポートしています。JSON Schemaを直接指定することで、出力フォーマットの不整合を大幅に低減できます。

```typescript
// OpenAI APIのStructured Output
import OpenAI from "openai";

const openai = new OpenAI();

const quizSchema = {
  type: "object",
  properties: {
    title: { type: "string" },
    questions: {
      type: "array",
      items: {
        type: "object",
        properties: {
          question: { type: "string" },
          answers: {
            type: "array",
            items: { type: "string" },
            minItems: 4,
            maxItems: 4,
          },
          correct: { type: "string" },
          type: {
            type: "string",
            enum: ["multiple_choice", "true_false"],
          },
          explanation: { type: "string" },
        },
        required: ["question", "answers", "correct", "type"],
      },
    },
    sourceText: { type: "string" },
  },
  required: ["title", "questions", "sourceText"],
} as const;

const response = await openai.chat.completions.create({
  model: "gpt-4o",
  messages: [
    { role: "system", content: systemPrompt },
    { role: "user", content: userPrompt },
  ],
  response_format: {
    type: "json_schema",
    json_schema: {
      name: "quiz_generation",
      schema: quizSchema,
      strict: true,
    },
  },
});
```

Structured Outputを使えばJSONの構造的な不整合はほぼゼロにできますが、「正解が選択肢に含まれているか」「選択肢に重複がないか」といった意味的な制約は依然としてプロンプトで制御する必要があります。

## パターン2: Temperature設定の最適化

Temperatureは出力のランダム性を制御するパラメータです。用途に応じて適切な値を設定することが品質に直結します。

### MochiQでの使い分け

```dart
// クイズ生成用: temperature 0.3
// 一貫性のある問題生成を重視。ただし0にすると問題のバリエーションが減る
GenerationConfig(
  temperature: 0.3,
  topP: 0.9,
  topK: 20,
  maxOutputTokens: 4096,
)

// 分析・評価用: temperature 0.1
// 学習データ分析では再現性と一貫性を最大限に重視
GenerationConfig(
  temperature: 0.1,
  topP: 0.8,
  topK: 10,
  maxOutputTokens: 2048,
)
```

Temperature設定のガイドラインを整理します。

| 用途 | 推奨Temperature | 理由 |
|------|---------------|------|
| クイズ生成 | 0.2 - 0.4 | 正確性と多様性のバランス |
| 学習分析 | 0.0 - 0.1 | 再現性と一貫性を重視 |
| 解説文生成 | 0.3 - 0.5 | 自然な文章表現を許容 |
| クリエイティブ系 | 0.7 - 1.0 | 多様な出力が望ましい場合 |

教育コンテンツの生成では、temperatureを上げすぎないことが鉄則です。多様性よりも正確性が優先されるためです。

## パターン3: ソーステキスト参照によるハルシネーション抑制

ハルシネーションを減らす最も効果的な方法は、LLMに「この情報だけを使って回答せよ」と明示的に指示することです。

```
以下のテキスト内容に**のみ**基づいてクイズを生成してください。
テキストに記載されていない情報を問題に含めないでください。

ソーステキスト:
"""
{ここに教材から抽出されたテキスト}
"""

生成ルール:
1. 問題の回答は、必ずソーステキスト内に記載されている情報から導出できるものにすること
2. ソーステキストに記載のない固有名詞、数値、事実を問題や選択肢に使用しないこと
3. 不正解の選択肢も、ソーステキストの文脈から妥当なものを選ぶこと
```

画像入力のマルチモーダルモデルの場合、画像から抽出したテキストをプロンプトに含めることはできません。しかし、レスポンスに `sourceText` フィールドを要求することで、AIが何を読み取ったかを事後的に検証できます。

## パターン4: Few-shot例の効果的な使い方

Few-shot（少数例の提示）は、出力品質を安定させる強力なテクニックです。ただし、例の選び方次第で逆効果になることもあります。

### 良いFew-shot例の条件

```
## 例1: 良い問題の例
入力テキスト: 「織田信長は1582年の本能寺の変で明智光秀に討たれた」
生成されたクイズ:
{
  "question": "織田信長が討たれた事件の名前は？",
  "answers": ["本能寺の変", "応仁の乱", "関ヶ原の戦い", "大坂の陣"],
  "correct": "本能寺の変",
  "type": "multiple_choice",
  "explanation": "1582年、明智光秀が本能寺に滞在していた織田信長を襲撃した事件です。"
}

## 例2: 悪い問題の例（生成してはいけないもの）
入力テキスト: 「織田信長は1582年の本能寺の変で明智光秀に討たれた」
悪い生成例:
{
  "question": "織田信長の好きな食べ物は？",
  "answers": ["味噌", "米", "肉", "魚"],
  "correct": "味噌"
}
理由: ソーステキストに食べ物に関する情報が含まれていないため、ハルシネーション。
```

「悪い例」を明示的に示すことで、LLMが陥りやすい失敗パターンを回避できます。これは Negative Few-shot と呼ばれるテクニックです。

## パターン5: プロンプトのバージョン管理

プロンプトはコードと同様にバージョン管理すべきです。MochiQでは、プロンプトをリソースファイルとして管理し、変更履歴を追跡しています。

```
prompts/
├── quiz_generation/
│   ├── v1.0.txt     # 初期バージョン
│   ├── v1.1.txt     # ハルシネーション対策追加
│   ├── v2.0.txt     # Few-shot例追加
│   └── v2.1.txt     # 学年別指示の強化
└── analysis/
    ├── v1.0.txt     # 学習分析プロンプト
    └── v1.1.txt     # パーソナライズ対応
```

プロンプトの変更は、品質メトリクスの変動と紐づけて評価します。

```typescript
// プロンプトバージョンと品質メトリクスの追跡
interface PromptVersion {
  version: string;
  deployedAt: Date;
  metrics: {
    structuralErrorRate: number;   // 構造エラー率
    hallucinationRate: number;     // ハルシネーション率
    userSatisfaction: number;      // ユーザー満足度
    averageLatency: number;        // 平均レイテンシ
  };
}
```

## Gemini API vs OpenAI API の比較

MochiQではGemini APIをメインで使用していますが、OpenAI APIとの比較検証も行っています。

| 観点 | Gemini API | OpenAI API |
|------|-----------|------------|
| マルチモーダル | ネイティブ対応（画像+テキスト） | GPT-4oで対応 |
| 日本語品質 | 良好 | 非常に良好 |
| Structured Output | 対応 | JSON Mode / Strict JSON Schema |
| コスト（Flashモデル） | 低い | 比較的高い |
| レイテンシ | 高速（特にFlashモデル） | 安定 |
| 教育コンテンツ適性 | 良好 | 良好 |

教育アプリの場合、日本語の自然さとコストのバランスが重要です。Gemini Flashモデルはコストパフォーマンスに優れているため、大量のクイズ生成に適しています。一方、品質が最重要の場面（例: 入試対策コンテンツ）では、GPT-4oやGemini Proのような上位モデルの使用を検討します。

## プロンプト設計のチェックリスト

プロンプトを本番投入する前に、以下のチェックリストで検証します。

- [ ] 出力フォーマットが具体例（JSONサンプル等）とともに指定されているか
- [ ] 制約条件（選択肢の数、正解の包含等）が明示されているか
- [ ] ソーステキストへの参照を強制しているか（ハルシネーション対策）
- [ ] 対象ユーザーのレベルが指定されているか
- [ ] 「してはいけないこと」が明記されているか
- [ ] Temperature等の生成パラメータが用途に合っているか
- [ ] バージョンが管理され、変更前後の品質比較が可能か
