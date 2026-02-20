---
title: "LLMアプリのAPI呼び出しコスト最適化"
---

## コストは品質の一部である

LLMアプリのコスト最適化は、単なる節約ではありません。コストが制御できなければ、品質保証のための追加のAPI呼び出し（Confidence Scoring、Cross-validationなど）を導入できません。コスト効率の改善は、品質投資の余地を生み出す行為です。

MochiQでは、Gemini APIの呼び出しコストを以下の戦略で最適化しています。

## 戦略1: 画像前処理による無駄なAPI呼び出しの削減

前章で紹介した `ImageQualityChecker` は、コスト最適化の観点でも重要です。API呼び出しの前に明らかに不適切な画像を弾くことで、無駄なコストを削減します。

```dart
class ImageQualityChecker {
  static const int _minFileSize = 100 * 1024;   // 100KB
  static const int _maxFileSize = 40 * 1024 * 1024;  // 40MB

  static Future<ImageQualityResult> checkFile(File file) async {
    final warnings = <ImageQualityWarning>[];
    var isAcceptable = true;

    // フォーマット検証: サポート外のフォーマットを弾く
    final pathResult = checkImagePath(file.path);
    if (!pathResult.isAcceptable) isAcceptable = false;

    // サイズ検証: 小さすぎる/大きすぎる画像を弾く
    final size = await file.length();
    if (size < _minFileSize) {
      warnings.add(ImageQualityWarning.tooSmall);
    }
    if (size > _maxFileSize) {
      isAcceptable = false;
    }

    return ImageQualityResult(
      isAcceptable: isAcceptable,
      warnings: warnings,
    );
  }
}
```

100KB未満の画像はテキストを含んでいる可能性が非常に低いため、API呼び出しをスキップします。この単純なチェックだけで、MochiQでは月間API呼び出しの約5-8%を削減できています。

## 戦略2: キャッシュ戦略

同じ教材から何度もクイズを生成するユーザーがいます。MochiQのGeminiServiceでは、分析結果をインメモリキャッシュとSharedPreferencesの2層でキャッシュしています。

```dart
class GeminiService {
  final Map<String, CachedInsight> _cache = {};
  static const Duration _cacheExpiry = Duration(hours: 6);

  Future<AILearningInsights> analyzeUserLearning({
    required String userId,
    required Map<String, dynamic> learningData,
    bool forceRefresh = false,
  }) async {
    // キャッシュチェック
    if (!forceRefresh) {
      final cached = await _getCachedInsights(userId);
      if (cached != null) {
        return cached;  // キャッシュヒット: API呼び出し不要
      }
    }

    // API呼び出し
    final response = await _model.generateContent([Content.text(prompt)]);
    final insights = _parseAIInsights(response.text!);

    // キャッシュに保存
    await _cacheInsights(userId, insights);
    return insights;
  }

  Future<AILearningInsights?> _getCachedInsights(String userId) async {
    // Layer 1: インメモリキャッシュ
    final cached = _cache[userId];
    if (cached != null && !cached.isExpired) {
      return cached.insights;
    }

    // Layer 2: SharedPreferences（永続化キャッシュ）
    final prefs = await SharedPreferences.getInstance();
    final cachedJson = prefs.getString('ai_insights_$userId');
    final cacheTime = prefs.getInt('ai_insights_time_$userId');

    if (cachedJson != null && cacheTime != null) {
      final cacheDateTime = DateTime.fromMillisecondsSinceEpoch(cacheTime);
      if (DateTime.now().difference(cacheDateTime) < _cacheExpiry) {
        final insights = AILearningInsights.fromJson(json.decode(cachedJson));
        _cache[userId] = CachedInsight(insights, cacheDateTime);
        return insights;
      }
    }

    return null;
  }
}
```

キャッシュ設計で重要なポイントは以下の通りです。

- **有効期限を設定する**: 学習分析は6時間ごとに更新すれば十分。リアルタイム性と コストのバランス
- **強制更新（forceRefresh）を用意する**: ユーザーが明示的に最新データを要求した場合に対応
- **2層キャッシュ**: インメモリ（高速・揮発）+ 永続化（低速・永続）の組み合わせ

### サーバーサイドでのキャッシュ

バックエンド側でもキャッシュを導入する場合は、コンテンツハッシュをキーにします。

```typescript
// コンテンツベースのキャッシュキー生成
import crypto from "crypto";

function generateCacheKey(imageHashes: string[], prompt: string): string {
  const content = [...imageHashes, prompt].join("|");
  return crypto.createHash("sha256").update(content).digest("hex");
}

// Redis等の分散キャッシュを活用
async function getOrGenerate(
  cacheKey: string,
  generateFn: () => Promise<QuizSet>,
  ttlSeconds: number = 3600,
): Promise<QuizSet> {
  const cached = await redis.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }

  const result = await generateFn();
  await redis.setex(cacheKey, ttlSeconds, JSON.stringify(result));
  return result;
}
```

## 戦略3: モデルの使い分け

すべてのリクエストに最高性能のモデルを使う必要はありません。タスクの重要度に応じてモデルを切り替えることで、コストを大幅に削減できます。

```dart
// MochiQでの使い分け
// クイズ生成: gemini-2.0-flash-exp（高速・低コスト）
// 学習分析: gemini-2.0-flash-exp（一貫性重視のパラメータ設定）
```

```typescript
// モデル選択の判断基準（TypeScript例）
type TaskType = "quiz_generation" | "quality_review" | "analysis" | "chat";

function selectModel(taskType: TaskType): ModelConfig {
  switch (taskType) {
    case "quiz_generation":
      // 標準的なクイズ生成: コスト重視
      return {
        model: "gemini-2.0-flash",
        temperature: 0.3,
        maxTokens: 4096,
        estimatedCostPer1K: 0.0001,
      };

    case "quality_review":
      // 品質レビュー（Confidence Scoring）: 精度重視
      return {
        model: "gemini-2.0-pro",
        temperature: 0.1,
        maxTokens: 2048,
        estimatedCostPer1K: 0.005,
      };

    case "analysis":
      // 学習データ分析: バランス重視
      return {
        model: "gemini-2.0-flash",
        temperature: 0.1,
        maxTokens: 2048,
        estimatedCostPer1K: 0.0001,
      };

    case "chat":
      // AI学習サポート: 自然さ重視
      return {
        model: "gemini-2.0-flash",
        temperature: 0.5,
        maxTokens: 2048,
        estimatedCostPer1K: 0.0001,
      };
  }
}
```

### 2026年のモデルコスト目安

| モデル | 入力コスト（/1M tokens） | 出力コスト（/1M tokens） | 用途 |
|-------|------------------------|------------------------|------|
| Gemini 2.0 Flash | $0.10 | $0.40 | 標準的なクイズ生成 |
| Gemini 2.0 Pro | $1.25 | $5.00 | 品質レビュー |
| GPT-4o | $2.50 | $10.00 | 高品質な生成 |
| GPT-4o mini | $0.15 | $0.60 | コスト重視 |

MochiQのように大量のクイズを生成するアプリでは、Flashモデルと上位モデルの差額が月間で数万円規模になることがあります。

## 戦略4: Exponential Backoff Retry

API呼び出しの失敗時にすぐリトライすると、レートリミットに引っかかってさらに失敗が増え、無駄なコストが発生します。指数バックオフでリトライ間隔を広げることが重要です。

```dart
class RetryPolicy {
  final int maxAttempts;
  final Duration initialDelay;
  final Duration maxDelay;
  final double backoffMultiplier;

  const RetryPolicy({
    this.maxAttempts = 3,
    this.initialDelay = const Duration(seconds: 1),
    this.maxDelay = const Duration(seconds: 30),
    this.backoffMultiplier = 2.0,
  });

  /// n回目のリトライの待ち時間を計算
  Duration getDelayForAttempt(int attempt) {
    if (attempt < 0) return initialDelay;

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

待ち時間の推移: 1秒 → 2秒 → 4秒（最大30秒）。3回失敗した場合はリトライを諦め、エラーをユーザーに返します。

## 戦略5: リクエストのバッチ処理

複数の画像を個別にAPI呼び出しするのではなく、1回のリクエストにまとめることでオーバーヘッドを削減します。

```typescript
// 悪い例: 画像ごとに個別API呼び出し
async function generateQuizzesNaive(images: File[]): Promise<QuizSet[]> {
  const results: QuizSet[] = [];
  for (const image of images) {
    const result = await callGeminiAPI([image]);  // N回のAPI呼び出し
    results.push(result);
  }
  return results;
}

// 良い例: 複数画像を1リクエストにまとめる
async function generateQuizzesBatched(images: File[]): Promise<QuizSet> {
  return await callGeminiAPI(images);  // 1回のAPI呼び出し
}
```

MochiQの `generateQuizFromImages` は `List<File> images` を受け取り、複数画像を1つのマルチモーダルリクエストにまとめています。教科書の見開き2ページを1回のリクエストで処理できるのは、コスト面でも体験面でもメリットがあります。

## 月間コストの目安とスケーリング計画

MochiQの実績をベースに、ユーザー規模別の月間コスト目安を示します。

| ユーザー規模 | 月間API呼び出し | 推定月間コスト | 備考 |
|------------|--------------|-------------|------|
| 100人 | 3,000回 | $3-5 | ほぼ無視できるレベル |
| 1,000人 | 30,000回 | $30-50 | Flashモデル主体 |
| 10,000人 | 300,000回 | $300-500 | キャッシュ効果が顕著に |
| 100,000人 | 3,000,000回 | $2,000-4,000 | モデル使い分け+キャッシュ必須 |

前提: 1ユーザーあたり月間30回のクイズ生成、1回あたり約500入力トークン+1500出力トークン、Gemini Flash利用。

## まとめ: コスト最適化の優先順位

1. **画像前処理**: 最も簡単で効果が高い。まず実装すべき
2. **キャッシュ**: 同じ入力の重複呼び出しを排除。2層キャッシュ推奨
3. **モデルの使い分け**: タスクの重要度に応じたモデル選択
4. **バッチ処理**: 複数画像の統合リクエスト
5. **Exponential Backoff**: 失敗時の無駄なリトライを抑制

これらの施策を組み合わせることで、品質保証のための追加API呼び出し（Confidence Scoringなど）のコスト余地を確保しつつ、全体のコストを制御可能な範囲に収めることができます。
