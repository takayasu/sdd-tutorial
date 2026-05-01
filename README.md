## 背景

* 生成AIの発展により、「AIに指示を出し、品質を担保する仕組みを設計できる人」と「できない人」の生産性格差が急速に拡大している
  * AIが自走して開発・テストを進められる環境を構築できるエンジニアが、チームに大きな価値を生む時代が来ている
  * 明確なKPIの設定と評価が必要

## 目的

* 開発プロセスの全領域をAIに委ねられる状態を作る
  * 仕様と実装の一体化（= 関数型ドメインモデリング）
  * CI自動化
    * 従来型のコード複雑度評価、脆弱性スキャン
    * PBT（プロパティベーステスト）を含むテスト
* Java + Spring Boot以外の模索
  * ビルドの高速化：フィードバックループの回数を増やす
  * 言語レベルでの書き方の統一：汎用性が低い言語を選択することでハーネスに頼らず品質を安定させる

## PoCの詳細

### 前提

- 日本語ドメインDSL（data / behavior）をSSoTとし、Kiro CLIでコードに変換する
- DSLは人間が書く。AIが生成したコードはCIの全ステップ突破をもってOKとする
- 選定基準：型安全性、ビルド速度、エコシステム成熟度
- 関数型ドメインモデリングの考え方に基づく仕様駆動開発
- 同一のDSLからF#・Kotlin両方のコードを生成し比較する

### フレームワーク・スタイル方針

| 言語 | Webフレームワーク | DB接続 | マイグレーション | スタイル |
|---|---|---|---|---|
| F# | Giraffe（ASP.NET Core上） | Donald + Npgsql | DbUp | 関数型（標準） |
| Kotlin | Ktor | Exposed | Flyway | 関数型スタイル（sealed class + data class + 純粋関数中心、varを避けimmutableデフォルト、Arrow活用） |

### DB構成

- PostgreSQL（docker-composeで構築）
- マイグレーションツールでスキーマをコード管理する
- CIではマイグレーション適用 → テスト実行の順で実行する

```
migrations/
├── V001__create_lot_table.sql
├── V002__create_sales_case_table.sql
└── V003__create_appraisal_table.sql
```

### Exit Criteria

1. 環境が構築できること
2. 与えたDSL（domain-model-section1〜3.md）についてAPIを実装し、CIを突破していること
3. 上記達成後、ビルドサイクル時間等の効率面を評価する

#### 3の評価方法

同一マシン上でF#・Kotlinそれぞれの `./ci.sh` を実行し、体感と実測で比較する。厳密なベンチマークではなく、開発サイクルの快適さを主観込みで判断する。

計測対象：
- クリーンビルド時間（`dotnet build` / `gradle build`）
- インクリメンタルビルド時間（1ファイル変更後の再ビルド）
- CI全体の実行時間（`time ./ci.sh`）

---

## ステップバイステップ手順

### 完了条件の共通テンプレ (Step 7 / 14 / 15 / 18 で踏襲)

集約 API (Lot / SalesCase / Reservation / Consignment) を実装する step は、**「集約API完全パッケージ」テンプレ** に従う。後付けで漏れがちな項目を最初から含めることで、フロント連携や DAST で発覚する手戻りをゼロに近づける:

1. Mutation (状態遷移)
2. **詳細 GET** (`GET /xxx/{id}`)
3. **一覧 GET** (`GET /xxx?status=&limit=&offset=` — `{ items, total, limit, offset }` 形式)
4. **楽観ロック** (`version INTEGER NOT NULL DEFAULT 1` カラム + `WHERE version = @expected` UPDATE。競合時 **409 + problem+json**)
5. **エラー形式統一** (`application/problem+json` (RFC 9457))
6. **OpenAPI 完全記述** (`components.schemas` に `XxxResponse / XxxSummary / XxxListResponse` を全部定義)
7. **ci.sh verify セクションへ追記** (Step 1 で導入 — false-positive completion を構造的に防ぐ)
8. **URL 集約規約** (販売案件系は `/sales-cases/{id}/{caseType}/...` に集約)

横断ミドルウェア (CORS / nosniff / CORP / problem+json default) は **Step 1 で全部入れる**。後付けにしない。

### Section 1: 在庫ロット実装 + CI段階的構築

| Step | やること | CIスタック | 完了条件 |
|---|---|---|---|
| 0 | 環境構築（Docker, .NET SDK, Gradle） | なし | `dotnet --version` / `gradle --version` が通る |
| 1 | Hello World API + 横断ミドルウェア (CORS/nosniff/CORP/problem+json) + ci.sh verify セクション + OpenAPI スケルトン | `dotnet build` / `gradle build` | `/health` 200 + セキュリティヘッダ + 404 が problem+json + verify が緑 |
| 2 | docker-compose構築（PostgreSQL） | `docker-compose up -d` | DBに接続できる |
| 3 | マイグレーション導入（DbUp / Flyway） | + マイグレーション適用 | テーブルが作成される |
| 4 | フォーマッター導入 | + Fantomas / ktfmt | `--check` がCI上で通る |
| 5 | リンター導入 | + FSharpLint / detekt | 警告0で通る |
| 6 | 在庫ロットの型定義（domain-model-section1.mdから生成） | 変更なし | ビルドが通る |
| 7 | 在庫ロット集約API完全パッケージ + DB永続化 (mutation + 詳細GET + 一覧GET + version + problem+json + openapi完全記述) | 変更なし | verify セクションで Lot 系の curl がすべて通る |
| 8 | PBT導入（在庫ロットの状態遷移） | + テスト実行 | PBTが通る |
| 9 | テストカバレッジ | + coverage計測 | カバレッジレポート出力 |
| 10 | gitleaks + SCA | + gitleaks + Trivy | 検出0で通る |
| 11 | SAST（Kotlinのみ） | + SonarQube | 高リスク検出0 |

### Section 2: 直接販売案件実装

| Step | やること | CIスタック | 完了条件 |
|---|---|---|---|
| 12 | 直接販売案件 + 価格査定 + 販売契約の型定義（domain-model-section2.md） | 変更なし | ビルドが通る |
| 13 | マイグレーション追加（販売案件・査定・契約テーブル、`version` カラム含む） | 変更なし | マイグレーション適用成功 |
| 14 | 直接販売案件 集約API完全パッケージ + URL 集約規約 (`/sales-cases/{id}/direct/...`) | 変更なし | PBT + verify セクションで SalesCase 系の curl がすべて通る (caseType ポリモーフィック GET / 一覧 / 409 / 400 / レガシー URL 404) |
| 15 | 価格査定・販売契約のAPI実装 (Step 14 と同じ規約) | 変更なし | PBT + verify (appraisal/contract の version conflict 409 含む) |

### Section 3: 予約・委託実装 + ダッシュボード

| Step | やること | CIスタック | 完了条件 |
|---|---|---|---|
| 16 | 予約・委託販売案件の型定義（domain-model-section3.md） | 変更なし | ビルドが通る |
| 17 | マイグレーション追加（予約・委託テーブル、`version` カラム含む） | 変更なし | マイグレーション適用成功 |
| 18 | 予約・委託・品目変換のAPI実装 (URL は `/sales-cases/{id}/{reservation\|consignment}/...` に集約) | 変更なし | PBT + verify (caseType=reservation/consignment の一覧/詳細/version 含む) |
| 18b | 認証 ON 化 + DevTokenMint CLI + `/auth/config` パブリックエンドポイント | 変更なし | verify セクションが auth=off / auth=on の 2 周どちらも緑 |
| 19 | 品質ダッシュボード構築 | + Grafana / SonarQube | メトリクスが可視化される |
| 20 | DAST | + OWASP ZAP | **`FAIL-NEW: 0` かつ `WARN-NEW: 0`** で `./ci.sh` が exit 0 (CSV content-type 明記 + lotId 入力検証も含む) |

### Section 4: Phase 2 — ハーネスエンジニアリング（RALPHループ構築）

Phase 1 のCIは「人間がCIを読んで修正する」前提で組まれている。Phase 2 では同じパイプラインを「AIエージェントが自走する」ためのハーネスへ昇格させる。

| Step | やること | 追加スタック | 完了条件 |
|---|---|---|---|
| 21 | SARIF統一出力 | + Sarif.Multitool, Roslyn ErrorLog, sarifreport (ZAP) | merged.sarif が生成される |
| 22 | ミューテーションテスト | + Stryker.NET / PITest | Mutation Score ≥ 75% |
| 23 | アーキテクチャ適合性検査 | + ArchUnitNET / ArchUnit | レイヤルール 4件 pass |
| 24 | APIコントラクトテスト | + PactNet / pact-jvm + Pact Broker | can-i-deploy が yes |
| 25 | SBOM生成 | + CycloneDX | sbom.cdx.json が生成される |
| 26 | 依存関係自動更新 | + Renovate (npx) | dry-run pass |
| 27 | OpenTelemetryエージェントトレース | + Jaeger + .claude/hooks | Jaeger に span が見える |
| 28 | AGENTS.md自動更新 + `/security-review` skill 統合（最小RALPH） | + Stop hook + sarif-to-lessons + security-review→SARIF | AGENTS.md に教訓 (静的ツール由来 + レビュー由来) が追記される |
| 29 | マルチエージェントオーケストレーター | + .harness/master.py + 4 subagents | 4エージェントが協調動作 |
| 30 | 完全自律RALPHループ | + harness/ralph.sh + prd.md | prd.md 全項目が [x] になる |

---

## PBT（プロパティベーステスト）の組み込み方針

### ツール

| 言語 | PBTライブラリ |
|---|---|
| F# | FsCheck.Xunit（xUnit + FsCheck 3） |
| Kotlin | Kotest Property Testing |

### テスト対象のプロパティ例

在庫ロット（Section 1）:
- 製造中ロット → 製造完了 → 出荷指示 → 出荷完了 の順序でのみ遷移可能
- 製造中ロットに出荷完了を指示すると必ずエラーになる
- 製造完了を取り消すと製造中ロットに戻る（往復性）

直接販売案件（Section 2）:
- 査定前の案件に契約を締結すると必ずエラーになる
- 査定→契約→出庫の順序でのみ遷移可能
- 契約削除後は査定済み状態に戻る（往復性）

予約・委託（Section 3）:
- 予約販売案件に販売契約を締結する操作は型レベルで不可能
- 委託指定解除後は委託指定前に戻る（往復性）

### CIへの組み込み

```bash
# F#
dotnet test --filter "Category=PBT"

# Kotlin
gradle test --tests "*PropertyTest*"
```

PBTはStep 8で導入し、以降の全ステップで新しいbehaviorを追加するたびにPBTも追加する。

---

## 用語集：CIで使うツールの説明

このPoCで使うツールを、役割ごとに簡潔に説明する。

| カテゴリ | ツール名 | 一言で言うと |
|---|---|---|
| フォーマッター | Fantomas / ktfmt | コードの見た目（インデント、改行）を自動統一する。人による書き方のバラつきをゼロにする |
| リンター | FSharpLint / detekt | コードの「品質上の問題」を自動検出する。長すぎる関数、深すぎるネスト、マジックナンバー等 |
| PBT | FsCheck.Xunit / Kotest Property | ランダムな入力を大量生成し、「どんな入力でもこの性質を満たす」ことを検証するテスト手法 |
| カバレッジ | coverlet / JaCoCo | テストで実行されたコードの割合を計測する。テストが足りない箇所を可視化 |
| シークレット検出 | gitleaks | コード中にパスワードやAPIキーが含まれていないかをスキャンする |
| SCA | Trivy | 使っているライブラリに既知の脆弱性がないかをスキャンする |
| SAST | SonarQube | コードを実行せずに解析し、セキュリティ上の脆弱性を検出する（Kotlinのみ） |
| DAST | OWASP ZAP | 実際に動いているAPIに攻撃を模擬し、脆弱性を検出する |
| マイグレーション | DbUp / Flyway | DBのテーブル構造の変更履歴をSQLファイルで管理し、コマンド1つで適用する |
| ダッシュボード | Grafana / SonarQube | CIの実行結果（カバレッジ、複雑度等）を時系列グラフで可視化する |
| SARIF統一 | Sarif.Multitool | 各ツールのSARIFを1ファイルにマージする（エージェント可読化） |
| ミューテーションテスト | Stryker.NET / PITest | コードを機械的に変異させてテストの厳しさを測る |
| アーキテクチャテスト | ArchUnitNET / ArchUnit | レイヤ依存ルールをコードで強制する |
| コントラクトテスト | PactNet / pact-jvm + Pact Broker | API消費者と提供者の契約を機械検証する |
| SBOM | CycloneDX | 依存ライブラリの部品表を生成する |
| 依存自動更新 | Renovate | 依存ライブラリの最新化PRを自動作成する |
| エージェント観測 | OpenTelemetry + Jaeger | エージェントのツール実行をトレースとして可視化する |
| 自己学習 | Claude Code Stop hook | CI失敗パターンをAGENTS.mdに自動追記する |
| ハーネスループ | harness/ralph.sh | PRD完了までエージェントを自動反復起動する |

---

## Phase 構成

このチュートリアルは 2 つの Phase で構成されている：

- **Phase 1 (Step 0–20)**: 人間レビュー前提の CI 完成。従来型の SDD（仕様駆動開発）。
- **Phase 2 (Step 21–30)**: 同じ CI をエージェント自走可能なハーネスへ昇格。
  - 21: 結果の統一可読化（SARIF）
  - 22–26: 品質ゲート多層化 + 依存管理
  - 27: エージェント挙動の観測（OTel）
  - 28–30: 自己学習 → 協調 → 完全自走（RALPH）

Phase 2 の理論的背景は arXiv 2604.08224 *"Externalization in LLM Agents: A Unified Review of Memory, Skills, Protocols and Harness Engineering"* を参照。

---

## 比較サマリ

| 観点 | F# | Kotlin |
|---|---|---|
| DSLとの構造的一致 | ◎ DSLがほぼ1:1で型に変換される | ○ sealed class + data classで表現可能だがやや冗長 |
| 不正な状態の排除 | ◎ 判別共用体 + 網羅的パターンマッチ + 不変デフォルト | ○ sealed class + when式。Arrow等でResult/Either活用 |
| ビルド速度 | ○ クリーン5〜15秒、インクリメンタル1〜5秒 | ○ Ktor: 15〜30秒（Spring Bootより大幅に速い） |
| SAST（コード脆弱性） | × F#向けSASTなし。型設計で代替 | ○ SonarQube公式対応 |
| 複雑度メトリクス | × 専用ツールなし。scc予約 + FSharpLintで簡易チェック | ◎ detekt + SonarQube |
| 品質ダッシュボード | × 自前構築が必要（CI + Grafana） | ◎ SonarQubeで即利用可能 |
| エコシステム全般 | ○ .NET（ASP.NET Core, NuGet）は成熟 | ◎ JVM（Ktor, Maven/Gradle）は最大級 |
| 日本語情報・採用実績 | △ 少ない | ◎ 多い |

### トレードオフの構造

- **F#**: 型安全性が高い → そもそもバグ・複雑さが生まれにくい → メトリクスの必要性が下がる。ただしツールで品質を「証明」できない
- **Kotlin**: 型安全性はF#より劣る → ツールで品質を計測・可視化して補う → 組織的に品質を管理しやすい。ただし型で防げない不正状態はPBTとツールに依存する

---

## ツールチェーン

### F#

| 観点 | ツール | 備考 |
|---|---|---|
| Webフレームワーク | Giraffe（ASP.NET Core上） | 軽量。ビルドへの影響小 |
| ビルド | `dotnet build` | 追加設定不要 |
| DB接続 | Donald + Npgsql | F#ネイティブ。パイプラインで記述 |
| マイグレーション | DbUp | SQLファイルを順番に適用するだけ |
| リンター | FSharpLint | Ionide統合済み |
| フォーマッター | Fantomas | `dotnet fantomas` でCI実行可 |
| テスト | xUnit + FsCheck.Xunit | PBT統合済み（`[<Property>]` + `[<Trait("Category","PBT")>]`） |
| 複雑度メトリクス | scc | `scc --by-file --format json` |
| SAST | なし（型設計で代替） | |
| DAST | OWASP ZAP | |
| SCA | `dotnet list package --vulnerable` + Trivy | |
| シークレット検出 | gitleaks | |
| 品質ダッシュボード | CI出力（JSON） → Grafana | 自前構築 |

### Kotlin

| 観点 | ツール | 備考 |
|---|---|---|
| Webフレームワーク | Ktor | 関数型スタイルに適合 |
| ビルド | Gradle（Kotlin DSL） | daemon + インクリメンタルビルド |
| DB接続 | Exposed（DSLモード） | Kotlin向け軽量ORM。関数型寄り |
| マイグレーション | Flyway | Gradle plugin対応。`gradle flywayMigrate` |
| リンター | detekt | 複雑度・コードスメル一括チェック |
| フォーマッター | ktfmt | `ktfmt --kotlinlang-style` |
| テスト | Kotest（Property Testing） | PBT内蔵 |
| 複雑度メトリクス | detekt + SonarQube | |
| SAST | SonarQube | |
| DAST | OWASP ZAP | |
| SCA | Trivy + Snyk | |
| シークレット検出 | gitleaks | |
| 品質ダッシュボード | SonarQube | |

---

## CIインフラ構成

### 前提

- 全社共通のCI基盤なし。プロジェクトごとにAWS/Azureアカウントを払い出す
- docker-composeでCIに必要なコンテナリソースを構築
- CI実行コマンドを叩くとCIが走る構成（GitHub Actions等に依存しない）

### 構成方針

```
docker-compose up -d     # CIインフラ起動（PostgreSQL, SonarQube, Grafana等）
./ci.sh                  # CI実行（マイグレーション・ビルド・テスト・解析を順次実行）
```

### F# CI構成

```
docker-compose.yml
├── db               # PostgreSQL（常駐）
├── grafana          # 品質ダッシュボード（常駐）
└── ci.sh            # ワンショット実行
    ├── dotnet run --project tools/Migrator   # DbUpマイグレーション適用
    ├── dotnet build --warnaserror
    ├── dotnet fantomas --check .
    ├── dotnet tool run fsharplint lint src/
    ├── scc --by-file --format json src/
    ├── dotnet test --filter "Category=PBT" --collect:"XPlat Code Coverage"
    ├── trivy fs --scanners vuln .
    └── gitleaks detect
```

### Kotlin CI構成

```
docker-compose.yml
├── db               # PostgreSQL（アプリ + SonarQube共用、常駐）
├── sonarqube        # 品質ダッシュボード + SAST + 複雑度（常駐）
└── ci.sh            # ワンショット実行
    ├── gradle flywayMigrate              # Flywayマイグレーション適用
    ├── gradle build
    ├── gradle detekt
    ├── gradle ktfmtCheck
    ├── gradle test jacocoTestReport
    ├── sonar-scanner（→ SonarQubeへ送信）
    ├── trivy fs --scanners vuln .
    └── gitleaks detect
```
