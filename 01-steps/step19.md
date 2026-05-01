# Step 19: 品質ダッシュボード構築

## 目的

### これは何か

CIの実行結果を時系列で可視化するダッシュボードを構築する。ダッシュボードとは、カバレッジ率・コード複雑度・脆弱性数などのメトリクスをグラフで表示する画面。

- F#: Grafana（汎用ダッシュボードツール。CI出力のJSONを取り込む）
- Kotlin: SonarQube（Step 11で導入済み。コード品質に特化したダッシュボード）

### なぜやるのか

- CIは「通った/落ちた」しかわからない。ダッシュボードがあれば「カバレッジが先週から5%下がった」「複雑度が上がり続けている」といった傾向が見える
- チームの品質状況を一目で把握できる
- 「品質を数値で管理する」文化を作る

### 何がうれしいのか

- 「なんとなく品質が悪い気がする」ではなく、数値で議論できる
- 品質が悪化し始めたタイミングで気づける（手遅れになる前に対処できる）
- ステークホルダーに「品質は管理されている」ことを示せる

## 完了条件

### Kotlin（SonarQube）

```bash
# SonarQubeが起動していること
$ curl -s http://localhost:9000/api/system/status | python3 -c "import sys,json; print(json.load(sys.stdin)['status'])"
UP

# 解析実行後、ダッシュボードにメトリクスが表示されること
$ gradle sonar
BUILD SUCCESSFUL in Xs

# ブラウザで http://localhost:9000 にアクセスし、以下が確認できること：
# - カバレッジ率
# - 循環的複雑度
# - コード重複率
# - 脆弱性数
# - 技術的負債
```

### F#（Grafana）

```bash
# Grafanaが起動していること
$ curl -s http://localhost:3000/api/health
{"commit":"...","database":"ok","version":"..."}

# CI実行後、JSON結果が出力されていること
$ ./ci.sh
=== CI完了 ===

$ ls ci-results/
coverage.json  lint.json  scc_2024-04-20T10:00:00Z.json

# ブラウザで http://localhost:3000 にアクセスし（admin/admin）、
# ダッシュボードにカバレッジ推移等のグラフが表示されること
```

---

## Kotlin（SonarQube ダッシュボード活用）

Step 11でSonarQubeは導入済み。ここでは品質ゲートとダッシュボードの設定に集中する。

### 1. 品質ゲート設定

ブラウザで http://localhost:9000 にアクセスし、以下を設定：

- Quality Gate → 新規作成 or デフォルト編集
  - カバレッジ: 80%以上
  - 重複率: 3%以下
  - 脆弱性: 0
  - バグ: 0
  - コードスメル: A評価

### 2. 確認するメトリクス

| メトリクス | 確認内容 |
|---|---|
| カバレッジ | PBTでどの程度カバーされているか |
| 循環的複雑度 | 関数の複雑さ |
| 認知的複雑度 | 可読性の指標 |
| コード重複率 | DRY原則の遵守 |
| 脆弱性 | SAST検出結果 |
| 技術的負債 | 修正にかかる推定時間 |

---

## F#（Grafana + CI出力JSON）

### 1. docker-compose.yml にGrafanaを追加

```yaml
# fsharp/docker-compose.yml に追加
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
      - ./ci-results:/var/lib/grafana/ci-results

volumes:
  grafana_data:
```

### 2. CI結果をJSONで出力

ci.sh の各ステップでJSON出力を追加：

```bash
#!/bin/bash
set -e

RESULTS_DIR="./ci-results"
mkdir -p "$RESULTS_DIR"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

# カバレッジ結果をJSON化
echo "=== テスト + カバレッジ ==="
dotnet test --collect:"XPlat Code Coverage" --results-directory ./coverage
COVERAGE=$(grep -oP 'line-rate="\K[^"]+' coverage/**/coverage.cobertura.xml | head -1)
echo "{\"timestamp\": \"$TIMESTAMP\", \"coverage\": $COVERAGE}" >> "$RESULTS_DIR/coverage.json"

# scc結果
echo "=== 複雑度 ==="
scc --by-file --format json src/ > "$RESULTS_DIR/scc_$TIMESTAMP.json"

# リンター結果
echo "=== リンター ==="
LINT_WARNINGS=$(dotnet tool run fsharplint lint src/SalesManagement/SalesManagement.fsproj 2>&1 | grep -c "Warning" || echo "0")
echo "{\"timestamp\": \"$TIMESTAMP\", \"lint_warnings\": $LINT_WARNINGS}" >> "$RESULTS_DIR/lint.json"
```

### 3. Grafanaダッシュボード設定

ブラウザで http://localhost:3000 にアクセス（admin/admin）：

1. Data Source → JSON API plugin を追加
2. Dashboard → 新規作成
3. パネル追加：
   - カバレッジ推移（折れ線グラフ）
   - リンター警告数推移
   - コード行数推移（scc）

---

## ci.sh の最終構成（F#）

```bash
#!/bin/bash
set -e

echo "=== マイグレーション ==="
dotnet run --project tools/Migrator

echo "=== ビルド ==="
dotnet build --warnaserror

echo "=== フォーマットチェック ==="
dotnet fantomas --check src/

echo "=== リンター ==="
dotnet tool run fsharplint lint src/SalesManagement/SalesManagement.fsproj

echo "=== テスト + カバレッジ ==="
dotnet test --collect:"XPlat Code Coverage" --results-directory ./coverage

echo "=== 複雑度 ==="
scc --by-file --format json src/

echo "=== シークレット検出 ==="
gitleaks detect --source . --exit-code 1

echo "=== パッケージ脆弱性スキャン ==="
trivy fs --scanners vuln --severity HIGH,CRITICAL .

echo "=== CI完了 ==="
```

## ci.sh の最終構成（Kotlin）

```bash
#!/bin/bash
set -e

echo "=== マイグレーション ==="
gradle flywayMigrate

echo "=== ビルド ==="
gradle build

echo "=== フォーマットチェック ==="
gradle ktfmtCheck

echo "=== リンター ==="
gradle detekt

echo "=== テスト + カバレッジ ==="
gradle test jacocoTestReport

echo "=== SAST (SonarQube) ==="
gradle sonar

echo "=== シークレット検出 ==="
gitleaks detect --source . --exit-code 1

echo "=== パッケージ脆弱性スキャン ==="
trivy fs --scanners vuln --severity HIGH,CRITICAL .

echo "=== CI完了 ==="
```

---

## 次のステップ

Step 19が完了したら [Step 20: DAST](./step20.md) へ進む。
