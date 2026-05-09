# Step 21: SARIF統一出力

## 目的

### これは何か

Phase 1 で導入した全CIツール（gitleaks, pip-audit, bandit, eslint, OWASP ZAP）の出力を、業界標準フォーマット **SARIF v2.1.0**（Static Analysis Results Interchange Format, OASIS標準）に統一し、`ci-results/merged.sarif` という1ファイルに集約する。

### なぜやるのか

- Phase 1 ではツールごとに出力形式が異なる：gitleaks=独自JSON、pip-audit=独自JSON、bandit=JSON/TEXT、ZAP=HTML/MD。AIエージェントが結果を読むにはツールごとにパーサを書く必要があった
- SARIF は `level`, `ruleId`, `locations[].physicalLocation.artifactLocation.uri`, `message.text` という共通スキーマを持ち、`jq` 1コマンドで全失敗を抽出できる
- 結果が機械可読になることで、Step 28 の AGENTS.md 自己学習ループの入力源になる

### 何がうれしいのか

- AIエージェントが `jq '.runs[].results[]' ci-results/merged.sarif` で全CI失敗を一括取得できる
- VS Code の SARIF Viewer や GitHub Code Scanning などUIが流用できる
- Phase 1 の「人間がCIを読む」前提から「エージェントがCIを読む」前提へ移行する第一歩

## 完了条件

```bash
$ ./ci.sh
=== CI完了 ===

$ ls ci-results/sarif/
bandit.sarif  eslint.sarif  gitleaks.sarif  pip_audit.sarif  zap.sarif

$ ls ci-results/merged.sarif
ci-results/merged.sarif

$ jq '.runs | length' ci-results/merged.sarif
5

$ jq '[.runs[].results[] | select(.level == "error")] | length' ci-results/merged.sarif
0
$ echo $?
0
```

---

## 0. ci-results/ の整備

Phase 1 ではツール出力先がバラバラだったが、Phase 2 では `ci-results/` に集約する。

```bash
# リポジトリルートで実行
mkdir -p ci-results/sarif
echo "ci-results/" >> .gitignore
git add .gitignore
```

ディレクトリ構成：

```
ci-results/
├── sarif/              # ツール別 SARIF
│   ├── gitleaks.sarif
│   ├── pip_audit.sarif
│   ├── bandit.sarif
│   ├── eslint.sarif
│   └── zap.sarif
└── merged.sarif        # マージ済み（エージェント参照用）
```

---

## 追加パッケージ

```toml
# backend/pyproject.toml
[project.optional-dependencies]
dev = [
    # ... 既存 ...
    "sarif-tools>=3.0",   # SARIF マージ・分析
]
```

```bash
cd backend && uv pip install sarif-tools
# フロントエンド
cd frontend && pnpm add -D @microsoft/eslint-formatter-sarif
```

---

## 1. gitleaks（ネイティブ SARIF）

gitleaks は `--report-format sarif` でネイティブ SARIF 出力可能。

```bash
gitleaks detect --source . \
  --report-format sarif \
  --report-path ci-results/sarif/gitleaks.sarif \
  --exit-code 1
```

---

## 2. pip-audit → SARIF 変換

pip-audit はネイティブ SARIF 非対応のため、JSON 出力を Python で変換する。

```bash
# JSON 出力を取得
cd backend
uv run pip-audit --format json -o ../ci-results/pip_audit_raw.json || true
cd ..

# SARIF に変換
python scripts/pip-audit-to-sarif.py \
  ci-results/pip_audit_raw.json \
  ci-results/sarif/pip_audit.sarif
```

`scripts/pip-audit-to-sarif.py`:

```python
#!/usr/bin/env python3
import json
import sys

input_path, output_path = sys.argv[1], sys.argv[2]

with open(input_path) as f:
    data = json.load(f)

results = []
for dep in data.get("dependencies", []):
    for vuln in dep.get("vulns", []):
        results.append({
            "ruleId": vuln["id"],
            "level": "error",
            "message": {"text": vuln["description"]},
            "locations": [{
                "physicalLocation": {
                    "artifactLocation": {"uri": "backend/pyproject.toml"}
                }
            }],
        })

sarif = {
    "$schema": "https://schemastore.azurewebsites.net/schemas/json/sarif-2.1.0.json",
    "version": "2.1.0",
    "runs": [{
        "tool": {"driver": {"name": "pip-audit", "version": "1.0"}},
        "results": results,
    }],
}

with open(output_path, "w") as f:
    json.dump(sarif, f, indent=2)
```

---

## 3. bandit（ネイティブ SARIF）

bandit は `-f sarif` でネイティブ SARIF 出力可能。

```bash
cd backend
uv run bandit -r src/ -ll -ii -f sarif \
  -o ../ci-results/sarif/bandit.sarif || true
cd ..
```

---

## 4. eslint（SARIF フォーマッター）

`@microsoft/eslint-formatter-sarif` を使って SARIF 出力する。

```bash
cd frontend
pnpm eslint src/ \
  --format @microsoft/eslint-formatter-sarif \
  --output-file ../ci-results/sarif/eslint.sarif || true
cd ..
```

---

## 5. OWASP ZAP の SARIF 出力

ZAP の reporting add-on を有効化して SARIF テンプレートで出力する。

```bash
cd backend
uv run uvicorn src.main:app --host 0.0.0.0 --port 8000 &
APP_PID=$!
sleep 3

docker run --rm --network host \
  -v "$(pwd)/../ci-results/sarif:/zap/results" \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-api-scan.py \
    -t http://localhost:8000/openapi.json \
    -f openapi \
    -z "addonupdate;addoninstall sarifreport" \
    -J /zap/results/zap.sarif

kill $APP_PID
cd ..
```

---

## SARIF マージ（sarif-tools）

Python の `sarif-tools` で複数 SARIF ファイルを 1 つに統合する。

```bash
cd backend
uv run python -m sarif merge \
  ../ci-results/sarif/*.sarif \
  -o ../ci-results/merged.sarif
cd ..
```

マージ後の確認：

```bash
$ jq '.runs[] | { tool: .tool.driver.name, results: (.results | length) }' ci-results/merged.sarif
{ "tool": "gitleaks", "results": 0 }
{ "tool": "pip-audit", "results": 0 }
{ "tool": "bandit", "results": 0 }
{ "tool": "ESLint", "results": 0 }
{ "tool": "OWASP ZAP", "results": 0 }
```

---

## ci.sh への追加

```bash
echo "=== ci-results/ 初期化 ==="
mkdir -p ci-results/sarif

echo "=== シークレット検出 (SARIF) ==="
gitleaks detect --source . \
  --report-format sarif \
  --report-path ci-results/sarif/gitleaks.sarif \
  --exit-code 1

echo "=== SCA (SARIF) ==="
cd backend
uv run pip-audit --format json -o ../ci-results/pip_audit_raw.json || true
cd ..
python scripts/pip-audit-to-sarif.py \
  ci-results/pip_audit_raw.json \
  ci-results/sarif/pip_audit.sarif

echo "=== SAST - Python (SARIF) ==="
cd backend
uv run bandit -r src/ -ll -ii -f sarif \
  -o ../ci-results/sarif/bandit.sarif || true
cd ..

echo "=== SAST - TypeScript (SARIF) ==="
cd frontend
pnpm eslint src/ \
  --format @microsoft/eslint-formatter-sarif \
  --output-file ../ci-results/sarif/eslint.sarif || true
cd ..

echo "=== DAST (SARIF) ==="
cd backend
uv run uvicorn src.main:app --host 0.0.0.0 --port 8000 &
APP_PID=$!
sleep 3
docker run --rm --network host \
  -v "$(pwd)/../ci-results/sarif:/zap/results" \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-api-scan.py \
    -t http://localhost:8000/openapi.json \
    -f openapi \
    -z "addonupdate;addoninstall sarifreport" \
    -J /zap/results/zap.sarif
kill $APP_PID
cd ..

echo "=== SARIF マージ ==="
cd backend
uv run python -m sarif merge \
  ../ci-results/sarif/*.sarif \
  -o ../ci-results/merged.sarif
cd ..

echo "=== SARIF サマリ ==="
jq '[.runs[].results[]] | group_by(.level) | map({level: .[0].level, count: length})' \
  ci-results/merged.sarif

echo "=== エラー件数確認 ==="
ERROR_COUNT=$(jq '[.runs[].results[] | select(.level == "error")] | length' ci-results/merged.sarif)
if [ "$ERROR_COUNT" -gt 0 ]; then
  echo "SARIF error count: $ERROR_COUNT — CI失敗"
  exit 1
fi
echo "PASS sarif-error-count-zero"
```

---

## RALPHループにおける位置づけ

| 層 | 役割 |
|---|---|
| 観測 (Observability) | CIの結果を機械可読な形でエージェントに渡す |
| フィードバック信号 | Step 28 の自己学習ループの主要入力源 |

ハーネスエンジニアリング（arXiv 2604.08224）における「外部化されたフィードバック信号」の最小実装。**この step がないと Phase 2 の他要素（特に Step 28 の AGENTS.md 自動更新）が成立しない**。

---

## 次のステップ

Step 21が完了したら [Step 22: ミューテーションテスト](./step22.md) へ進む。
