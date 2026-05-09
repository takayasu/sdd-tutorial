# Step 19: 品質ダッシュボード構築

## 目的

### これは何か

CIの実行結果を時系列で可視化するダッシュボードを構築する。ダッシュボードとは、カバレッジ率・コード複雑度・脆弱性数などのメトリクスをグラフで表示する画面。

Step 2 の docker-compose.yml にはすでに Grafana が含まれている。ここでは CI が出力する JSON を Grafana で読み取り、品質推移グラフを表示できるようにする。

### なぜやるのか

- CIは「通った/落ちた」しかわからない。ダッシュボードがあれば「カバレッジが先週から5%下がった」「複雑度が上がり続けている」といった傾向が見える
- チームの品質状況を一目で把握できる
- 「品質を数値で管理する」文化を作る

### 何がうれしいのか

- 「なんとなく品質が悪い気がする」ではなく、数値で議論できる
- 品質が悪化し始めたタイミングで気づける（手遅れになる前に対処できる）
- ステークホルダーに「品質は管理されている」ことを示せる

## 完了条件

```bash
# Grafanaが起動していること（Step 2から継続）
$ curl -s http://localhost:3000/api/health
{"commit":"...","database":"ok","version":"..."}

# CI実行後、JSON結果が出力されていること
$ ./ci.sh
=== CI完了 ===

$ ls ci-results/
coverage_2024-04-20T10:00:00Z.json  complexity_2024-04-20T10:00:00Z.json  lint_2024-04-20T10:00:00Z.json

# ブラウザで http://localhost:3000 にアクセスし（admin/admin）、
# ダッシュボードにカバレッジ推移等のグラフが表示されること
```

---

## 追加パッケージ

```toml
# backend/pyproject.toml の [project.optional-dependencies] に追加
[project.optional-dependencies]
dev = [
    # ... 既存 ...
    "radon>=6.0",          # 循環的複雑度
    "pytest-json-report",  # pytest JSON出力
]
```

```bash
cd backend && uv pip install radon pytest-json-report
```

---

## CI結果をJSONで出力

### ci.sh の更新（JSON出力追加）

```bash
#!/bin/bash
set -e

RESULTS_DIR="./ci-results"
mkdir -p "$RESULTS_DIR"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "=== マイグレーション ==="
cd backend && alembic upgrade head && cd ..

echo "=== フォーマットチェック (ruff) ==="
cd backend && uv run ruff format --check src/ && cd ..

echo "=== リンター (ruff) ==="
cd backend && uv run ruff check src/ && cd ..

echo "=== テスト + カバレッジ ==="
cd backend
uv run pytest tests/ \
  --cov=src \
  --cov-branch \
  --cov-fail-under=80 \
  --cov-report=json:../ci-results/coverage_raw.json \
  --json-report --json-report-file=../ci-results/pytest_${TIMESTAMP}.json \
  -q
COVERAGE=$(python -c "import json; d=json.load(open('../ci-results/coverage_raw.json')); print(round(d['totals']['percent_covered'], 2))")
echo "{\"timestamp\": \"$TIMESTAMP\", \"coverage\": $COVERAGE}" \
  >> "../$RESULTS_DIR/coverage_${TIMESTAMP}.json"
cd ..

echo "=== 複雑度 (radon) ==="
cd backend
COMPLEXITY=$(uv run python -m radon cc src/ -j)
echo "{\"timestamp\": \"$TIMESTAMP\", \"complexity\": $COMPLEXITY}" \
  > "../$RESULTS_DIR/complexity_${TIMESTAMP}.json"
cd ..

echo "=== 型チェック (mypy) ==="
cd backend && uv run mypy src/ --strict && cd ..

echo "=== フロントエンド型チェック ==="
cd frontend && pnpm tsc --noEmit && cd ..

echo "=== フロントエンドテスト + カバレッジ ==="
cd frontend
pnpm vitest run --coverage.enabled true \
  --reporter=json --outputFile=../ci-results/vitest_${TIMESTAMP}.json
cd ..

echo "=== シークレット検出 ==="
gitleaks detect --source . --exit-code 1

echo "=== パッケージ脆弱性スキャン ==="
cd backend && uv run pip-audit --format json -o ../ci-results/pip_audit_${TIMESTAMP}.json; cd ..
cd frontend && pnpm audit --audit-level=high; cd ..

echo "=== SAST ==="
cd backend && uv run bandit -r src/ -ll -ii -f json \
  -o ../ci-results/bandit_${TIMESTAMP}.json; cd ..

echo "=== 予約・委託・品目変換 API 検証 ==="
# Step 18 完了条件の確認（サービス起動済み前提）
STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
  -X POST http://localhost:8000/sales-cases/2/reservation/appraisals \
  -H "Content-Type: application/json" \
  -d '{"appraisal_date":"2024-04-05","estimated_lot_info":"A商品 10本","estimated_amount":300000,"version":1}')
[ "$STATUS" = "200" ] || [ "$STATUS" = "201" ] && echo "PASS reservation-appraisal-create"

echo "{\"timestamp\": \"$TIMESTAMP\", \"status\": \"success\"}" \
  > "$RESULTS_DIR/lint_${TIMESTAMP}.json"

echo "=== CI完了 ==="
```

---

## Grafana ダッシュボード設定

### docker-compose.yml（Step 2 から継続・確認）

Step 2 で追加済みの Grafana サービスに `ci-results` ボリュームマウントを追加する：

```yaml
services:
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./ci-results:/var/lib/grafana/ci-results  # ← 追加

volumes:
  grafana_data:
```

### ダッシュボード作成手順

1. ブラウザで http://localhost:3000 にアクセス（admin/admin）
2. **Connections** → **Add new data source** → **JSON API** プラグインを追加
3. **Dashboards** → **New dashboard** → **Add visualization**
4. 以下のパネルを追加：

| パネル名 | データソース | 可視化 |
|---|---|---|
| カバレッジ推移 | `ci-results/coverage_*.json` の `coverage` フィールド | 折れ線グラフ |
| 複雑度分布 | `ci-results/complexity_*.json` の `complexity` | 棒グラフ |
| CIステータス | `ci-results/lint_*.json` の `status` | ステートマップ |

---

## 確認するメトリクス

| メトリクス | ツール | 確認内容 |
|---|---|---|
| カバレッジ率 | pytest-cov | PBTでどの程度カバーされているか（80%以上） |
| 循環的複雑度 | radon | 関数の複雑さ（A/B/C/D/E/F評価） |
| SAST指摘数 | bandit | セキュリティ問題の件数 |
| 脆弱性数 | pip-audit / pnpm audit | OSS依存の既知CVE |
| 型エラー数 | mypy / tsc | 型安全性の状態 |

---

## radon 複雑度の見方

```bash
# 循環的複雑度（CC）をファイル別に表示
cd backend && uv run python -m radon cc src/ -s

# 結果例：
# src/domain/sales_case_workflows.py
#     F 14:0 conclude_contract - A (2)
#     F 22:0 instruct_shipping - A (3)
# src/routers/sales_cases.py
#     F 45:0 create_appraisal - B (6)

# グレード基準:
# A: 1-5  （単純、リスク低）
# B: 6-10 （少し複雑）
# C: 11-15（やや複雑、要注意）
# D: 16-20（複雑、リファクタ推奨）
# F: 21+  （非常に複雑、要リファクタ）
```

---

## 次のステップ

Step 19が完了したら [Step 20: DAST](./step20.md) へ進む。
