---
title: "AI-DLCのルールファイルを全部読んでみた——awslabs/aidlc-workflowsの仕組みを深掘りする"
emoji: "🗂️"
type: "tech"
topics: ["AIDLC", "AmazonQ", "AIコーディング", "AWS", "ソフトウェア開発"]
published: false
---

> **この記事について**
> AI-DLCを試してみたいエンジニア向けに、公式リポジトリ [awslabs/aidlc-workflows](https://github.com/awslabs/aidlc-workflows) のルールファイル群が何をやっているかを、自分の理解の整理も兼ねてまとめました。構成・ドキュメント周りの解説は国内にほぼなかったので、「使い方」ではなく「仕組み」にフォーカスしています。

---

## AI-DLC とは

**AI-DLC（AI-Driven Development Life Cycle）** は、AWSがre:Invent 2025で発表したAI駆動のソフトウェア開発手法です。単なるツールではなく「開発プロセスそのものをAIに任せながら、人間がコントロールを手放さない」という方法論として、論文レベルで体系化されています。

実装は [awslabs/aidlc-workflows](https://github.com/awslabs/aidlc-workflows) としてオープンソース公開されており、Amazon Q Developer、Kiro、Cursor、Cline、Claude Code、GitHub Copilot など、多くのAIコーディングエージェントで利用できます。

ひとことで言えば、**マークダウンで書かれたルールファイル群をAIエージェントに読み込ませることで、ソフトウェア開発のライフサイクル全体を構造化されたワークフローとして実行させる仕組み**です。フロー全体は以下の3フェーズで構成されます。

```
[INCEPTION フェーズ] 🔵 ── 何を・なぜ作るかを決める
  └── 要件定義 → ユーザーストーリー → 設計 → Unit分割 → 実行計画

[CONSTRUCTION フェーズ] 🟢 ── どう作るかを決めて実装する
  └── Unit ごとに繰り返し:
      機能設計 → 非機能要件 → インフラ設計 → コード生成 → ビルド・テスト

[OPERATIONS フェーズ] 🟡 ── デプロイ・運用（現在はプレースホルダー）
```

重要なのが**アダプティブ実行**と**Human-in-the-loop**の2点です。リクエストの複雑さに応じてAI自身が実行するステージを判断し、各ステージの完了後には必ずユーザーの明示的な承認を求めます。「AIに開発を任せる」ではなく「AIと一緒に開発プロセスを回す」という思想が、ルールレベルで担保されています。

では、この仕組みが具体的にどのようなルールファイルで実現されているかを見ていきます。

---

## ルールファイルの全体構造

```
aidlc-rules/
├── aws-aidlc-rules/
│   └── core-workflow.md          ← 常時ロードされるエントリポイント
└── aws-aidlc-rule-details/
    ├── common/                   ← 全フェーズ共通ルール
    ├── inception/                ← Inceptionフェーズのステージ別ルール
    ├── construction/             ← Constructionフェーズのステージ別ルール
    ├── operations/               ← Operationsフェーズ（現在プレースホルダー）
    └── extensions/
        └── security/baseline/    ← オプション拡張（セキュリティ等）
```

`core-workflow.md` だけがAIエージェントのルールディレクトリに常時配置されます。詳細ルールは `aws-aidlc-rule-details/` に格納されており、**必要なステージになったタイミングで動的にロード**されます。

これは意図的な設計です。LLMのコンテキストウィンドウは有限なので、全ルールを最初から読み込むとトークンを大量消費してしまいます。「今必要な情報だけを渡す」という原則のもと、ファイルを分割することでコンテキスト効率を最大化しています。

---

## `core-workflow.md` — ワークフロー全体の指揮官

**📄 ファイルリンク**: [aws-aidlc-rules/core-workflow.md](https://github.com/awslabs/aidlc-workflows/blob/main/aidlc-rules/aws-aidlc-rules/core-workflow.md)

このファイルがAI-DLC全体のエントリポイントです。**常時参照される「親ルール」**として機能します。

**ルール詳細ファイルの解決パス定義**

使用しているエージェントに応じて、詳細ルールファイルの参照先ディレクトリを動的に切り替えるよう指示が書かれています。

```
Check these paths in order and use the first one that exists:
  .aidlc-rule-details/         ← Cursor, Cline, Claude Code, GitHub Copilot
  .kiro/aws-aidlc-rule-details/ ← Kiro IDE and CLI
  .amazonq/aws-aidlc-rule-details/ ← Amazon Q Developer
```

これにより同一のルールファイル群を、エージェントを問わず再利用できます。

**ワークフロー開始時に必ずロードするファイルの指示**

```
ALWAYS load common rules at workflow start:
  - common/process-overview.md
  - common/session-continuity.md
  - common/content-validation.md
  - common/question-format-guide.md
```

**ワークフロー設計原則の宣言**

以下の原則がルールとして明文化されています。

- `Adaptive Execution`: 必要なステージだけを実行する
- `Transparent Planning`: 実行前に必ずプランを提示する
- `User Control`: ユーザーがステージの追加・除外を指示できる
- `Complete Audit Trail`: 全インタラクションを `audit.md` にタイムスタンプ付きで記録する。**ユーザーの生の入力を要約・言い換えすることは禁止**

**生成物のディレクトリ構造の定義**

AIが生成するファイルの配置場所も厳密に定義されています。

```
aidlc-docs/              # ドキュメントのみ（コードを置いてはいけない）
├── aidlc-state.md       ← 現在のワークフロー状態
├── audit.md             ← 全インタラクションの監査ログ
├── inception/           ← Inceptionフェーズの成果物
├── construction/        ← Constructionフェーズの成果物
└── operations/
```

**アプリケーションコードはワークスペースルートに置き、`aidlc-docs/` 内には絶対にコードを生成しない**という制約も明記されています。

**Extensions の起動時スキャン**

```
CRITICAL: At workflow start, scan the extensions/ directory recursively
but load ONLY lightweight opt-in files — NOT full rule files.
```

ワークフロー開始時に `*.opt-in.md` だけをロードして拡張の有効化確認を行い、本体ルール（`security-baseline.md` など）はその後必要になってからロードします。これもコンテキスト節約のための設計です。

---

## `common/` — 全フェーズ共通の横断ルール

**📄 ディレクトリリンク**: [aws-aidlc-rule-details/common/](https://github.com/awslabs/aidlc-workflows/tree/main/aidlc-rules/aws-aidlc-rule-details/common)

### `common/process-overview.md` — Human-in-the-loopの思想

AI-DLC全体の哲学を記述したファイルです。「AIは実行者であり、決定者は人間である」という原則を明文化しています。

主なルール：
- 各ステージ完了後に必ず承認プロンプトを出すこと
- AIが自己判断で次のステージへ進まないこと
- 実行計画（`execution-plan.md`）をユーザーが承認するまでConstructionを開始しないこと

### `common/session-continuity.md` — セッションをまたいだ継続性

LLMはセッション間で記憶を持ちません。このファイルは、ワークフローを中断・再開する際の手順を定義しています。

主なルール：
- 再開時はまず `aidlc-docs/aidlc-state.md` を読み込む
- 最後に完了したステージを確認し、そこから再開する
- コンテキストリセット時は `aidlc-state.md` の最初の未完了項目を探し、対応するプランファイルから再開する

```markdown
# 再開時のプロンプト例（WORKING-WITH-AIDLC.md より）
I am resuming a previously stopped conversation.
Here is the context: [paste summary of last output or recent change]
Please continue with [next action or section X].
```

実際に生成される `aidlc-state.md` はこんな形です。

```markdown
## Stage Progress
| Phase       | Stage                | Status      |
|-------------|----------------------|-------------|
| INCEPTION   | Workspace Detection  | COMPLETE    |
| INCEPTION   | Reverse Engineering  | COMPLETE    |
| INCEPTION   | Requirements Analysis| IN_PROGRESS |
| CONSTRUCTION| Functional Design    | PENDING     |
```

なお、公式ドキュメントには**「コンテキストのコンパクション（圧縮）プロンプトが表示されても絶対に承認するな」**という注意書きがあります。コンパクションはコンテキストリセットとは異なり、それまでの重要な指示や成果物を中途半端に失うためです。

### `common/content-validation.md` — 生成物の品質ゲート

ファイルを生成する前に必ず通る検証ルールです。

主なルール：
- ファイルに書き込む前にコンテンツが空でないことを検証すること
- Mermaidダイアグラムの構文が正しいことを確認すること
- 必須セクション（見出し・要件番号など）が揃っていることを確認すること
- 検証に失敗した場合はエラーを記録し、修正してから再試行すること

**AIが生成したコンテンツをそのままファイルに書き出す前に、必ず検証を挟む**という設計は、実運用上でのサイレント品質劣化を防ぐ重要な役割を果たしています。

### `common/question-format-guide.md` — 質問の標準フォーマット

ユーザーへの質問の出し方を統一するルールです。

主なルール：
- 選択肢付きの構造化された質問フォーマットを使うこと
- `[Answer]:` タグを使ってユーザーが回答を書き込める形式にすること
- 質問は一度に複数まとめてファイルに書き出し、インラインで回答を求めること（チャットで一問一答しない）

```markdown
## [質問タイトル]
選択肢:
A) ...
B) ...
C) ...
[Answer]:
```

ユーザーは `[Answer]:` の行に答えを書くだけでよく、AIはそのファイルを読み取って処理を続けます。チャットを往復させず、ファイルベースで非同期に質問・回答を行う設計です。

---

## `inception/` — Inceptionフェーズのステージ別ルール

**📄 ディレクトリリンク**: [aws-aidlc-rule-details/inception/](https://github.com/awslabs/aidlc-workflows/tree/main/aidlc-rules/aws-aidlc-rule-details/inception)

### `inception/workspace-detection.md` — プロジェクトの文脈把握

最初に実行される必須ステージのルールです。

主なルール：
- `aidlc-docs/aidlc-state.md` が存在するかどうかを確認すること
  - **存在しない** → 新規プロジェクト（Greenfield）として Requirements Analysis へ
  - **存在する** → 既存プロジェクト（Brownfield）として Reverse Engineering へ
- `aidlc-docs/` ディレクトリを作成する許可をユーザーに求めること
- `aidlc-state.md`（進捗追跡）と `audit.md`（監査ログ）を初期化すること

ここで生成される `audit.md` には、その後の**全インタラクションがISO 8601タイムスタンプ付きで記録**されていきます。

### `inception/reverse-engineering.md` — 既存コードの分析

Brownfieldプロジェクト専用のステージです。

主なルール：
- 既存のコードベースを解析し、アーキテクチャ・依存関係・技術スタック・データモデル・API構造を文書化すること
- **鮮度チェック**: 前回の解析結果が `aidlc-docs/inception/reverse-engineering/` に残っている場合、それが最新かどうかを確認してからスキップ or 再実行を判断すること
- 「最初からやり直したい」場合は、既存成果物を `.backup` としてアーカイブしてから再実行すること

Reverse Engineeringで得られたコンテキストは、後続のRequirements Analysisで「AIがコードを把握済みの状態」で質問を生成するために使われます。

### `inception/requirements-analysis.md` — 要件定義の9ステップ ← 設計の急所

AI-DLCの中で最も詳細なルールが書かれているファイルのひとつです。

主なルール（9ステップの実行フロー）：

1. **コンテキストのロード**: Brownfieldならリバースエンジニアリング結果を参照する
2. **リクエストの分析**: 明確さ・種別・スコープ・複雑さを評価する
3. **分析深度の決定**: `minimal` / `standard` / `comprehensive` から選択する
4. **既存資料の評価**: ユーザーが提供した要件書・仕様書があれば考慮する
5. **6カテゴリによる網羅性チェック**: 機能・非機能・制約・前提条件・除外事項・成功基準
6. **質問票の生成**: 曖昧な点を `question-format-guide.md` のフォーマットで質問する
7. **要件ドキュメントの構築**: ユーザーの回答を取り込んで `requirements.md` を生成する
8. **検証**: `content-validation.md` に従って成果物を検証する
9. **進捗の更新**: `aidlc-state.md` のステータスを更新する

分析深度（Adaptive Depth）の判断基準：

| 深度 | 適用場面 |
|------|----------|
| Minimal | バグ修正、単純な変更 |
| Standard | 通常の機能追加・プロジェクト |
| Comprehensive | 複雑・高リスク・顧客業務システムなど |

`"Keep this at minimal depth"` や `"Please run at comprehensive depth"` と指示することで明示的に制御することも可能です。

> **📌 注目ポイント — Overconfidence Prevention（過信防止）**
>
> このファイルには、LLMが「わかったふり」をして先に進む傾向を抑制するための明示的なルールが書かれています。「仮定を作らず、不明点は必ず聞く」「要件が不完全な状態でコード生成に進まない」という制約です。他のAIコーディングツールは基本的にこの制約を持たず、曖昧な要件でもとりあえずコードを書き始めます。AI-DLCがルールレベルでこれを禁止しているのは、設計上の重要な差別化点です。

### `inception/user-stories.md` — ユーザーストーリーの生成

ユーザー中心の機能開発時に実行される条件付きステージです。

主なルール：
- ユーザーの目標・行動・期待値をもとにペルソナを定義すること
- 「〜として、〜したい。なぜなら〜」形式のユーザーストーリーを生成すること
- 受け入れ基準（Acceptance Criteria）をGiven-When-Then形式で記述すること
- MermaidのUser Journey図でストーリーを可視化すること

### `inception/application-design.md` — アーキテクチャ設計

複雑なアーキテクチャが必要な場合に実行されます。

主なルール：
- コンポーネント図・サービス境界・データフローをMermaidで可視化すること
- 技術スタックを選定する場合は選定理由を記録すること
- コンポーネント間のインターフェース（API仕様・イベント・データ形式）を定義すること

このフェーズで生成される `components.md` と `services.md` が、Constructionフェーズの設計の基盤となります。

### `inception/units-generation.md` — 実装単位の分割

マルチコンポーネントシステムで実行されます。

主なルール：
- 要件を独立して開発可能な「Unit（作業単位）」に分解すること
- 各Unitに名前・説明・依存関係・技術スタックを付与すること
- Constructionフェーズではこの単位でループ処理を行うこと

実際に生成されるUnit分解のイメージ：

```markdown
| Requirement | backend         | frontend        | infrastructure        |
|-------------|-----------------|------------------|-----------------------|
| FR-001      | POST /api/todos | TodoForm         | Lambda + API Gateway  |
| FR-002      | GET /api/todos  | TodoList         | Lambda + DynamoDB     |
```

### `inception/workflow-planning.md` — 実行計画の策定

Inceptionの最後に実行される必須ステージです。

主なルール：
- どのConstructionステージを実行するかを決定し、`execution-plan.md` を生成すること
- ステージをスキップする場合はその理由を記録すること
- `execution-plan.md` の承認をユーザーに求めること。**承認なしにConstructionへ進んではいけない**

---

## `construction/` — Constructionフェーズのステージ別ルール

**📄 ディレクトリリンク**: [aws-aidlc-rule-details/construction/](https://github.com/awslabs/aidlc-workflows/tree/main/aidlc-rules/aws-aidlc-rule-details/construction)

Constructionフェーズは **各Unitに対してループ実行**されます。

### `construction/functional-design.md` — 機能設計

主なルール：
- **技術スタックを排除した**ビジネスロジック設計を行うこと（Clean Architectureのドメイン層に相当）
- ビジネスルール（入力値の制約・状態遷移・計算ロジックなど）を言語・FW非依存の形式で記述すること
- `business-logic-model.md`・`business-rules.md`・`domain-entities.md` を生成すること

例えば「TODOの作成」というFunctional Designには「タイトル必須・最大100文字、タグは最大5個」といったビジネスルールが定義されます。具体的な実装（ReactやHonoのコード）は後続のCode Generationに委ねます。

### `construction/nfr-requirements.md` / `construction/nfr-design.md` — 非機能要件

`nfr-requirements.md` の主なルール：
- パフォーマンス・可用性・セキュリティ・スケーラビリティ・保守性などの非機能要件を定義すること
- 各要件に測定可能な指標（例：レスポンスタイム < 200ms）を設定すること

`nfr-design.md` の主なルール：
- NFR要件を満たすための設計パターンを選定すること（キャッシュ戦略・サーキットブレーカーなど）
- Mermaidによる論理コンポーネント図を生成すること
- Security Extensionが有効な場合は、セキュリティ設計と要件IDのマッピングを記録すること

### `construction/infrastructure-design.md` — インフラ設計

主なルール：
- クラウドリソース・デプロイアーキテクチャ・ネットワーク構成をドキュメント化すること
- Brownfieldの場合はReverse Engineeringで解析済みの既存インフラを参照すること
- MermaidによるDeployment Architecture図を生成すること
- Security Extensionが有効な場合、最小権限の原則（SECURITY-06）に従ったIAMポリシーの設計を行うこと

### `construction/code-generation.md` — コード生成の2ステップ承認 ← 設計の急所

AI-DLCでもっとも重要な制御が入っているファイルです。コード生成は**2段階の承認**で行われます。

**Step 1: コード計画（Code Plan）の承認**

主なルール：
- 生成するファイルの完全なリストをチェックボックス形式で `code-plan.md` に書き出すこと
- 各ファイルについて「新規作成か既存ファイルの変更か」を明記すること
- Brownfieldでは既存ファイルの変更リストを正確に記述すること（既存ファイルの隣に新しいファイルを作ってはいけない）
- **ユーザーが承認するまでコード生成を開始しない**

```markdown
Code Plan:
- [ ] src/components/TodoForm.tsx （新規）
- [ ] src/hooks/useGameState.ts   （新規）
- [ ] src/routes/todos.ts         （既存ファイルの変更）
```

**Step 2: コード生成の実行**

主なルール：
- 承認後、計画に従ってコードを生成すること
- **アプリケーションコードはワークスペースルートに配置**すること（`aidlc-docs/` 内には絶対に置かない）
- 各ファイル生成が完了したらチェックボックスをリアルタイムで更新すること
- Security Extensionが有効な場合、各セキュリティルールID（SECURITY-03 など）をコメントとして残すこと

> **📌 注目ポイント — なぜ2段階なのか**
>
> 「コードを書く前に何を書くかを承認させる」という設計は、AIが勝手にファイルを増殖させたり、既存ファイルを上書きしたりすることを構造的に防ぐためです。特にBrownfieldでは「既存ファイルの隣に新しいファイルを作ってはいけない」というルールが明示されており、既存コードベースへの破壊的変更を防ぐ意図が読み取れます。

Security Extension有効時に生成されたコードの例：

```typescript
// packages/backend/src/index.ts

// Middleware (SECURITY-03: structured logging)
app.use("*", logger());

// Global error handler (SECURITY-15: fail-safe defaults, SECURITY-09: no stack traces)
app.onError((err, c) => {
  console.error("Unhandled error:", err.message);
  return c.json({ error: "Internal server error" }, 500);
});
```

コメントにルールIDが残ることで、どのセキュリティ要件に基づいて実装したかが追跡できます。

### `construction/build-and-test.md` — ビルドとテスト

全Unitのコード生成完了後に実行される必須ステージです。

主なルール：
- ビルドコマンドを実行し、エラーがあれば修正すること
- テストを実行し、カバレッジと結果をサマリ化すること
- `build-and-test-summary.md` に結果を記録すること
- ビルドやテストの失敗はコード生成フェーズに差し戻してやり直すこと

---

## `extensions/security/baseline/` — セキュリティ拡張のオプトイン

**📄 ディレクトリリンク**: [aws-aidlc-rule-details/extensions/security/baseline/](https://github.com/awslabs/aidlc-workflows/tree/main/aidlc-rules/aws-aidlc-rule-details/extensions/security/baseline)

AI-DLCには**Extension（拡張）システム**が搭載されています。セキュリティ・コンプライアンス・組織固有のルールをコアワークフローの上にレイヤリングできます。

**2ファイル構成**

各Extensionは2つのファイルで構成されます。

- `security-baseline.md` — 拡張のルール本体（15のセキュリティルール、OWASP Top 10 2025マッピング）
- `security-baseline.opt-in.md` — ユーザーへの確認質問

**オプトインの仕組み**

ワークフロー開始時に `extensions/` ディレクトリをスキャンし、`*.opt-in.md` だけをロードして確認質問を行います。有効化の判断は `aidlc-state.md` に記録されます。

```markdown
## Extension Configuration
- Security Extensions:
  - Enabled: true
```

**Extensionが有効になると**

- `Blocking Constraint`（必須要件）と `Non-blocking Guidance`（推奨）の2種類のルールが全ステージに適用される
- Requirements Analysisで追加のセキュリティ質問が挿入される
- NFR DesignでOWASPルールに基づく設計パターンが適用される
- Code GenerationでルールIDコメントが付与される

企業のセキュリティポリシーやAPIガイドラインをExtensionとして追加すれば、全プロジェクトに自動適用できます。公式ドキュメントでは「組織の標準をExtensionとして追加すれば、毎回手動でルールを注入する必要がなくなる」と説明されています。

---

## ルールを読んでわかった、運用上の注意点

### 承認ゲートの多さは仕様である

`process-overview.md` はHuman-in-the-loopを設計思想の核に置いており、「AIが自己判断で次のステージへ進まないこと」がルールとして明文化されています。つまり**承認を何度も求められるのはバグではなく意図した動作**です。Inception〜Constructionを通じて承認が10回前後発生することを事前に想定しておくと、ワークフローを途中で放置するリスクが下がります。

### 生成物の量が多い分、待ち時間がある

`content-validation.md` はファイルへの書き込み前に必ず検証を挟む設計になっています。各ステージで生成ドキュメントの量が多いほど、この検証オーバーヘッドも比例して増えます。特にConstruction・Designフェーズは成果物の量が多く、AIが作業している間は人間が待ちの状態になります。この時間を「チーム内での次ステージのレビュー準備」に充てるようなリズムを作っておくとスムーズです。

### セッション再開時はコンパクションを承認してはいけない

`session-continuity.md` に明示されている注意点です。作業を中断して再開するときに、AIツールがコンテキストのコンパクション（圧縮）を提案することがあります。これを承認すると、それまでのルール指示や成果物の文脈が中途半端に失われます。再開時は必ず `aidlc-state.md` を読み込ませることから始めてください。

---

## まとめ

AI-DLCのルールファイルは、単なる「AIへの命令書」ではなく、ソフトウェア開発プロセス全体を構造化するためのメタプログラムとして設計されています。

| レイヤー | ファイル | 役割 |
|----------|----------|------|
| エントリポイント | `core-workflow.md` | ワークフロー全体の指揮・ファイル解決・コンテキスト節約 |
| 横断関心事 | `common/*.md` | セッション継続・品質検証・質問フォーマット統一 |
| フェーズルール | `inception/*.md` | 何をなぜ作るかの構造化（9ステップ要件定義・Overconfidence Prevention） |
| フェーズルール | `construction/*.md` | どう作るかの制御・2段階承認ゲート・セキュリティIDマッピング |
| 拡張 | `extensions/` | 組織固有ルールのオプトイン・全ステージへの自動適用 |

「AIにコードを書かせる」ではなく「AIと一緒に開発プロセスを回す」——その仕組みがどう実現されているかを理解すると、AI-DLCの使い方だけでなく、なぜこういう設計になっているかも見えてきます。

---

## 参考リンク

- [awslabs/aidlc-workflows - GitHub](https://github.com/awslabs/aidlc-workflows)
- [Building with AI-DLC using Amazon Q Developer - AWS Blog](https://aws.amazon.com/blogs/devops/building-with-ai-dlc-using-amazon-q-developer/)
- [WORKING-WITH-AIDLC.md - GitHub](https://github.com/awslabs/aidlc-workflows/blob/main/docs/WORKING-WITH-AIDLC.md)
- [DeepWiki: awslabs/aidlc-workflows](https://deepwiki.com/awslabs/aidlc-workflows)
- [awslabs/aidlc-workflows で実践するAWS流AI駆動開発 - Fusic Tech Blog](https://tech.fusic.co.jp/posts/awslabs-aidlc-workflows-inception-construction-practice/)
