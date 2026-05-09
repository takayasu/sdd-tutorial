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
* React / Python + FastAPI スタックでの実践
  * フロントエンド：React + TypeScript（型安全なUI開発）
  * バックエンド：Python + FastAPI（型安全なAPI、OpenAPI自動生成）
  * ビルドの高速化：フィードバックループの回数を増やす

## PoCの詳細

### 前提

- 日本語ドメインDSL（data / behavior）をSSoTとし、AIでコードに変換する
- DSLは人間が書く。AIが生成したコードはCIの全ステップ突破をもってOKとする
- 選定基準：型安全性、開発速度、エコシステム成熟度
- 関数型ドメインモデリングの考え方に基づく仕様駆動開発

### フレームワーク・スタイル方針

| 層 | フレームワーク | DB接続 | マイグレーション | スタイル |
|---|---|---|---|---|
| バックエンド | FastAPI（Python） | SQLAlchemy（async） | Alembic | 関数型スタイル（frozen dataclass + 純粋関数中心、ミューテーションを避けimmutableデフォルト） |
| フロントエンド | React + TypeScript + Vite | — | — | 関数型Reactコンポーネント + Zod バリデーション |

### DB構成

- PostgreSQL（docker-composeで構築）
- Alembicでスキーマをコード管理する
- CIではマイグレーション適用 → テスト実行の順で実行する

```
alembic/
└── versions/
    ├── 001_create_lot_table.py
    ├── 002_create_sales_case_table.py
    └── 003_create_appraisal_table.py
```

### Exit Criteria

1. 環境が構築できること
2. 与えたDSL（domain-model-section1〜3.md）についてAPIを実装し、CIを突破していること
3. 上記達成後、開発サイクル時間等の効率面を評価する

#### 3の評価方法

`./ci.sh` を実行し、体感と実測で比較する。

計測対象：
- 依存インストール時間（`pip install` / `npm install`）
- テスト実行時間（`pytest` / `vitest run`）
- CI全体の実行時間（`time ./ci.sh`）

---

## ステップバイステップ手順

### 完了条件の共通テンプレ (Step 7 / 14 / 15 / 18 で踏襲)

集約 API (Lot / SalesCase / Reservation / Consignment) を実装する step は、**「集約API完全パッケージ」テンプレ** に従う。後付けで漏れがちな項目を最初から含めることで、フロント連携や DAST で発覚する手戻りをゼロに近づける:

1. Mutation (状態遷移)
2. **詳細 GET** (`GET /xxx/{id}`)
3. **一覧 GET** (`GET /xxx?status=&limit=&offset=` — `{ items, total, limit, offset }` 形式)
4. **楽観ロック** (`version INTEGER NOT NULL DEFAULT 1` カラム + `WHERE version = :expected` UPDATE。競合時 **409 + problem+json**)
5. **エラー形式統一** (`application/problem+json` (RFC 9457))
6. **OpenAPI 完全記述** (FastAPI の自動生成 + Pydantic スキーマで `XxxResponse / XxxSummary / XxxListResponse` を定義)
7. **ci.sh verify セクションへ追記** (Step 1 で導入 — false-positive completion を構造的に防ぐ)
8. **URL 集約規約** (販売案件系は `/sales-cases/{id}/{caseType}/...` に集約)

横断ミドルウェア (CORS / nosniff / CORP / problem+json default) は **Step 1 で全部入れる**。後付けにしない。

### Section 1: 在庫ロット実装 + CI段階的構築

| Step | やること | CIスタック | 完了条件 |
|---|---|---|---|
| 0 | 環境構築（Docker, Python, Node.js） | なし | `python --version` / `node --version` が通る |
| 1 | Hello World API（FastAPI）+ React スケルトン + 横断ミドルウェア + ci.sh verify セクション + OpenAPI スケルトン | `pytest` / `vitest` | `/health` 200 + セキュリティヘッダ + 404 が problem+json + verify が緑 |
| 2 | docker-compose構築（PostgreSQL） | `docker compose up -d` | DBに接続できる |
| 3 | マイグレーション導入（Alembic） | + マイグレーション適用 | テーブルが作成される |
| 4 | フォーマッター導入 | + ruff format / Prettier | `--check` がCI上で通る |
| 5 | リンター導入 | + ruff check / ESLint | 警告0で通る |
| 6 | 在庫ロットの型定義（domain-model-section1.mdから生成） | 変更なし | ビルドが通る |
| 7 | 在庫ロット集約API完全パッケージ + DB永続化 (mutation + 詳細GET + 一覧GET + version + problem+json + openapi完全記述) | 変更なし | verify セクションで Lot 系の curl がすべて通る |
| 8 | PBT導入（在庫ロットの状態遷移） | + テスト実行 | PBTが通る |
| 9 | テストカバレッジ | + coverage計測 | カバレッジレポート出力 |
| 10 | gitleaks + SCA | + gitleaks + Trivy | 検出0で通る |
| 11 | SAST（bandit） | + bandit | 高リスク検出0 |

### Section 2: 直接販売案件実装

| Step | やること | CIスタック | 完了条件 |
|---|---|---|---|
| 12 | 直接販売案件 + 価格査定 + 販売契約の型定義（domain-model-section2.md） | 変更なし | ビルドが通る |
| 13 | マイグレーション追加（販売案件・査定・契約テーブル、`version` カラム含む） | 変更なし | マイグレーション適用成功 |
| 14 | 直接販売案件 集約API完全パッケージ + URL 集約規約 (`/sales-cases/{id}/direct/...`) | 変更なし | PBT + verify セクションで SalesCase 系の curl がすべて通る |
| 15 | 価格査定・販売契約のAPI実装 (Step 14 と同じ規約) | 変更なし | PBT + verify (appraisal/contract の version conflict 409 含む) |

### Section 3: 予約・委託実装 + ダッシュボード

| Step | やること | CIスタック | 完了条件 |
|---|---|---|---|
| 16 | 予約・委託販売案件の型定義（domain-model-section3.md） | 変更なし | ビルドが通る |
| 17 | マイグレーション追加（予約・委託テーブル、`version` カラム含む） | 変更なし | マイグレーション適用成功 |
| 18 | 予約・委託・品目変換のAPI実装 | 変更なし | PBT + verify (caseType=reservation/consignment の一覧/詳細/version 含む) |
| 18b | 認証 ON 化 + DevTokenMint CLI + `/auth/config` パブリックエンドポイント | 変更なし | verify セクションが auth=off / auth=on の 2 周どちらも緑 |
| 19 | 品質ダッシュボード構築 | + Grafana | メトリクスが可視化される |
| 20 | DAST | + OWASP ZAP | **`FAIL-NEW: 0` かつ `WARN-NEW: 0`** で `./ci.sh` が exit 0 |

### Section 4: Phase 2 — ハーネスエンジニアリング（RALPHループ構築）

Phase 1 のCIは「人間がCIを読んで修正する」前提で組まれている。Phase 2 では同じパイプラインを「AIエージェントが自走する」ためのハーネスへ昇格させる。

| Step | やること | 追加スタック | 完了条件 |
|---|---|---|---|
| 21 | SARIF統一出力 | + sarif-tools (Python) | merged.sarif が生成される |
| 22 | ミューテーションテスト | + mutmut (Python) | Mutation Score ≥ 75% |
| 23 | アーキテクチャ適合性検査 | + import-linter | レイヤルール pass |
| 24 | APIコントラクトテスト | + schemathesis | スキーマ適合テストが pass |
| 25 | SBOM生成 | + cyclonedx-bom | sbom.cdx.json が生成される |
| 26 | 依存関係自動更新 | + Renovate (npx) | dry-run pass |
| 27 | OpenTelemetryエージェントトレース | + Jaeger + .claude/hooks | Jaeger に span が見える |
| 28 | AGENTS.md自動更新 + `/security-review` skill 統合 | + Stop hook + sarif-to-lessons | AGENTS.md に教訓が追記される |
| 29 | マルチエージェントオーケストレーター | + .harness/master.py + 4 subagents | 4エージェントが協調動作 |
| 30 | 完全自律RALPHループ | + harness/ralph.sh + prd.md | prd.md 全項目が [x] になる |

---

## PBT（プロパティベーステスト）の組み込み方針

### ツール

| 層 | PBTライブラリ |
|---|---|
| Python（バックエンド） | hypothesis |
| TypeScript（フロントエンド） | fast-check |

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
# Python（hypothesis）
pytest -m pbt backend/

# TypeScript（fast-check）
npx vitest run src/**/*.pbt.test.ts
```

PBTはStep 8で導入し、以降の全ステップで新しいbehaviorを追加するたびにPBTも追加する。

---

## 用語集：CIで使うツールの説明

| カテゴリ | ツール名 | 一言で言うと |
|---|---|---|
| フォーマッター | ruff format | コードの見た目（インデント、改行）を自動統一する。Pythonの標準スタイルを強制 |
| フォーマッター | Prettier | TypeScript/JSXの見た目を自動統一する |
| リンター | ruff check | コードの「品質上の問題」を自動検出する。長すぎる関数、未使用変数等 |
| リンター | ESLint | TypeScript/Reactのコード品質問題を検出する |
| PBT | hypothesis | ランダムな入力を大量生成し、「どんな入力でもこの性質を満たす」ことを検証するテスト手法（Python） |
| PBT | fast-check | TypeScript版のPBTライブラリ |
| カバレッジ | pytest-cov | テストで実行されたコードの割合を計測する（Python） |
| カバレッジ | @vitest/coverage-v8 | フロントエンドのカバレッジを計測する（TypeScript） |
| シークレット検出 | gitleaks | コード中にパスワードやAPIキーが含まれていないかをスキャンする |
| SCA | Trivy | 使っているライブラリに既知の脆弱性がないかをスキャンする |
| SAST | bandit | Pythonコードを実行せずに解析し、セキュリティ上の脆弱性を検出する |
| DAST | OWASP ZAP | 実際に動いているAPIに攻撃を模擬し、脆弱性を検出する |
| マイグレーション | Alembic | DBのテーブル構造の変更履歴をPythonファイルで管理し、コマンド1つで適用する |
| ダッシュボード | Grafana | CIの実行結果（カバレッジ等）を時系列グラフで可視化する |
| SARIF統一 | sarif-tools | 各ツールのSARIFを1ファイルにマージする（エージェント可読化） |
| ミューテーションテスト | mutmut | コードを機械的に変異させてテストの厳しさを測る（Python） |
| アーキテクチャテスト | import-linter | レイヤ依存ルールをコードで強制する（Python） |
| コントラクトテスト | schemathesis | OpenAPIスキーマに対してファジングテストを自動実行する |
| SBOM | cyclonedx-bom | 依存ライブラリの部品表を生成する |
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

## ツールチェーン

### バックエンド（Python + FastAPI）

| 観点 | ツール | 備考 |
|---|---|---|
| Webフレームワーク | FastAPI | 非同期対応、型ヒント統合、OpenAPI自動生成 |
| ASGI サーバー | uvicorn | 開発・本番共用 |
| パッケージ管理 | uv + pyproject.toml | Rust製の高速パッケージマネージャ |
| DB接続 | SQLAlchemy (async) + asyncpg | 非同期ORM |
| マイグレーション | Alembic | `alembic upgrade head` でCI実行可 |
| リンター+フォーマッター | ruff | Black + flake8 + isort を統合した高速ツール |
| テスト | pytest + pytest-asyncio | 非同期テスト対応 |
| PBT | hypothesis | `@given` デコレータでPBT |
| カバレッジ | pytest-cov | `--cov` フラグでカバレッジ計測 |
| SAST | bandit | Pythonセキュリティ静的解析 |
| DAST | OWASP ZAP | |
| SCA | Trivy | |
| シークレット検出 | gitleaks | |

### フロントエンド（React + TypeScript + Vite）

| 観点 | ツール | 備考 |
|---|---|---|
| UIフレームワーク | React 19 + TypeScript | |
| ビルドツール | Vite | 高速な開発サーバーとビルド |
| パッケージマネージャ | pnpm | 高速・省スペース |
| リンター | ESLint | TypeScript対応 |
| フォーマッター | Prettier | |
| テスト | vitest + @testing-library/react | |
| PBT | fast-check | TypeScriptのPBTライブラリ |
| カバレッジ | @vitest/coverage-v8 | |
| APIクライアント型生成 | openapi-typescript | OpenAPIスキーマからTypeScript型を自動生成 |
| バリデーション | Zod | ランタイムバリデーション + 型推論 |

---

## CIインフラ構成

### 前提

- 全社共通のCI基盤なし。プロジェクトごとにAWS/Azureアカウントを払い出す
- docker-composeでCIに必要なコンテナリソースを構築
- CI実行コマンドを叩くとCIが走る構成（GitHub Actions等に依存しない）

### 構成方針

```
docker compose up -d     # CIインフラ起動（PostgreSQL, Grafana等）
./ci.sh                  # CI実行（マイグレーション・ビルド・テスト・解析を順次実行）
```

### CI構成

```
docker-compose.yml
├── db               # PostgreSQL（常駐）
└── grafana          # 品質ダッシュボード（常駐）

ci.sh                # ワンショット実行
├── alembic upgrade head              # Alembicマイグレーション適用
├── ruff format --check backend/      # フォーマットチェック（Python）
├── ruff check backend/               # リンター（Python）
├── pytest --cov=backend/src backend/ # テスト + カバレッジ（Python）
├── npx prettier --check frontend/src/  # フォーマットチェック（TypeScript）
├── npx eslint frontend/src/          # リンター（TypeScript）
├── npx vitest run --coverage         # テスト + カバレッジ（TypeScript）
├── bandit -r backend/src/            # SAST（Python）
├── trivy fs --scanners vuln .        # SCA
├── gitleaks detect                   # シークレット検出
└── === verify (smoke) ===            # API起動してcurl検証
```
