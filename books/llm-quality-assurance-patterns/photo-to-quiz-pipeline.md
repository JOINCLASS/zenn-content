---
title: "写真→クイズ生成パイプラインの全体像"
free: true
---

## MochiQのコアパイプライン

MochiQは「教材を撮影するだけで、AIがクイズを自動生成する」アプリです。このシンプルな体験の裏側には、複数のステージからなるパイプラインが存在します。

```
カメラ撮影 → 画像品質チェック → Gemini API（マルチモーダル） → JSONパース → バリデーション → 保存 → 出題
```

各ステージで異なる種類のエラーが発生し得るため、ステージごとに適切な品質保証の仕組みが必要です。本章ではパイプライン全体を俯瞰し、各ステージの役割と課題を整理します。

## ステージ1: 画像の取得と前処理

ユーザーがスマホのカメラで教材を撮影するところからパイプラインが始まります。この段階での品質問題は「そもそもAIが読み取れない画像が入力される」ことです。

MochiQでは、API呼び出しの前に `ImageQualityChecker` で画像の品質を検証しています。

```dart
class ImageQualityChecker {
  static const _supportedFormats = ['jpg', 'jpeg', 'png', 'webp', 'gif'];
  static const int _minFileSize = 100 * 1024;   // 100KB
  static const int _warnFileSize = 20 * 1024 * 1024; // 20MB
  static const int _maxFileSize = 40 * 1024 * 1024;  // 40MB

  static Future<ImageQualityResult> checkFile(File file) async {
    final warnings = <ImageQualityWarning>[];
    var isAcceptable = true;

    // フォーマット検証
    final pathResult = checkImagePath(file.path);
    warnings.addAll(pathResult.warnings);
    if (!pathResult.isAcceptable) isAcceptable = false;

    // サイズ検証
    final size = await file.length();
    final sizeResult = checkFileSize(size);
    warnings.addAll(sizeResult.warnings);
    if (!sizeResult.isAcceptable) isAcceptable = false;

    return ImageQualityResult(
      isAcceptable: isAcceptable,
      warnings: warnings,
    );
  }
}
```

ポイントは、**API呼び出しの前にフィルタリングすること**です。100KB未満の画像はテキストを含んでいる可能性が低く、40MBを超える画像は処理コストに見合いません。この段階で弾くことで、無駄なAPI呼び出しとユーザーの待ち時間を削減できます。

## ステージ2: マルチモーダルAIによる解析と生成

画像品質チェックを通過した画像は、Gemini APIに送信されます。MochiQでは `gemini-2.0-flash-exp` モデルを使用し、画像とプロンプトを同時に送るマルチモーダルリクエストを行っています。

```dart
class QuizGenerationService {
  static const String _modelName = 'gemini-2.0-flash-exp';
  static const Duration _defaultTimeout = Duration(seconds: 30);

  late final GenerativeModel _model;

  QuizGenerationService() {
    _model = GenerativeModel(
      model: _modelName,
      apiKey: _apiKey,
      generationConfig: GenerationConfig(
        temperature: 0.3,  // クイズ生成は低めのtemperatureで一貫性を重視
        topP: 0.9,
        topK: 20,
        maxOutputTokens: 4096,
      ),
    );
  }

  Future<Result<GeneratedQuizSet, QuizGenerationError>> generateQuizFromImages({
    required List<File> images,
    String quizType = 'mixed',
    int questionCount = 5,
    String? gradeLevel,
    Duration? timeout,
  }) async {
    // Phase 1: 画像をバイナリに変換
    final imageParts = <DataPart>[];
    for (final image in images) {
      final bytes = await image.readAsBytes();
      final mimeType = _getMimeType(image.path);
      imageParts.add(DataPart(mimeType, bytes));
    }

    // Phase 2: プロンプト構築
    final prompt = QuizGenerationPrompt.build(
      quizType: quizType,
      questionCount: questionCount,
      gradeLevel: gradeLevel,
    );

    // Phase 3: API呼び出し（タイムアウト付き）
    final response = await _model
        .generateContent([
          Content.multi([TextPart(prompt), ...imageParts])
        ])
        .timeout(timeout ?? _defaultTimeout);

    // Phase 4: レスポンスのパース
    final quizSet = QuizResponseParser.parse(response.text!);
    return Result.success(quizSet);
  }
}
```

ここで重要な設計判断がいくつかあります。

- **Temperature 0.3**: クイズ生成は創造性より一貫性を重視するため低めに設定。分析用途ではさらに低い0.1を使用
- **タイムアウト30秒**: モバイルアプリではユーザーの待ち時間が体験に直結するため、厳密に制御
- **複数画像対応**: 教科書の見開きなど、複数ページを一度に撮影するケースに対応

## ステージ3: レスポンスのパースとバリデーション

LLMからのレスポンスは自由形式のテキストです。JSONを要求しても、マークダウンのコードブロックで囲んでくることもあれば、前後に説明文を付けてくることもあります。

```dart
class QuizResponseParser {
  static GeneratedQuizSet? parse(String response) {
    try {
      String jsonString = response;

      // マークダウンのコードブロック内のJSONを抽出
      final codeBlockMatch =
          RegExp(r'```(?:json)?\s*([\s\S]*?)```').firstMatch(response);
      if (codeBlockMatch != null) {
        jsonString = codeBlockMatch.group(1)!.trim();
      } else {
        // 生のJSONオブジェクトを探索
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

このパーサーは「防御的」に設計されています。LLMの出力フォーマットのばらつきを吸収し、可能な限りJSONを抽出する試みを行います。それでもパースに失敗した場合は `null` を返し、呼び出し元でリトライや別のエラーハンドリングに回します。

## ステージ4: エラーハンドリングとリカバリ

パイプラインの各ステージで発生し得るエラーを `QuizGenerationError` として分類し、ユーザーに適切なリカバリアクションを提示します。

```dart
enum QuizGenerationError {
  noTextDetected,    // テキストが検出できない
  generationFailed,  // 生成に失敗
  timeout,           // タイムアウト
  networkError,      // ネットワークエラー
}
```

各エラーに対して、ユーザーが取れるアクション（再撮影、リトライ、接続確認）を明示するのが重要です。LLMアプリでは「何かうまくいかなかった」が頻繁に起きるため、ユーザーにとって次のステップが明確であることが体験の質を左右します。

## ステージ5: 保存と出題

バリデーションを通過したクイズはFirestoreに保存され、SM-2アルゴリズムによるスケジューリングのもとでユーザーに出題されます。各問題には `ReviewSchedule` が紐づき、学習の進行に応じて出題間隔が最適化されます。

```dart
class UserQuestion {
  final String id;
  final String question;
  final List<String> answers;
  final String correct;
  final QuestionType type;
  final String? explanation;
  final ReviewSchedule? review;  // SM-2スケジュール
}
```

## パイプラインの全体設計で意識すべきこと

このパイプラインから読み取れる設計原則をまとめます。

1. **早期フィルタリング**: 品質チェックはパイプラインのできるだけ上流で行う。API呼び出しの後で弾くのは無駄が大きい
2. **防御的パース**: LLMの出力フォーマットは信用しない。複数のパターンに対応するパーサーを書く
3. **段階的な進捗通知**: 生成に時間がかかるため、ユーザーには「今何をしているか」をリアルタイムで伝える
4. **明確なエラーリカバリ**: エラーが起きた時に「次に何をすべきか」をユーザーに提示する
5. **タイムアウトの厳密な設定**: モバイルアプリでは30秒が限界。それ以上待たせるなら非同期処理を検討する

次章以降では、このパイプラインの各ステージにおける品質問題を深掘りし、具体的な対策パターンを解説していきます。
