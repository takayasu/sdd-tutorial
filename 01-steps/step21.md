# Step 21: SARIF統一出力

## 目的

### これは何か

Phase 1 で導入した全CIツール（gitleaks, Trivy, detekt, Roslyn, SonarQube, OWASP ZAP）の出力を、業界標準フォーマット **SARIF v2.1.0**（Static Analysis Results Interchange Format, OASIS標準）に統一し、`ci-results/merged.sarif` という1ファイルに集約する。

### なぜやるのか

- Phase 1 ではツールごとに出力形式が異なる：gitleaks=独自JSON、Trivy=独自JSON、detekt=XML、SonarQube=REST API、ZAP=HTML/MD。AIエージェントが結果を読むにはツールごとにパーサを書く必要があった
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
detekt.sarif  fsharp-build.sarif  gitleaks.sarif  sonar.sarif  trivy.sarif  zap.sarif

$ ls ci-results/merged.sarif
ci-results/merged.sarif

$ jq '.runs | length' ci-results/merged.sarif
6

$ jq '[.runs[].results[] | select(.level == "error")] | length' ci-results/merged.sarif
0
$ echo $?
0
```

---

## 0. ci-results/ の整備（共通）

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
│   ├── trivy.sarif
│   ├── detekt.sarif
│   ├── fsharp-build.sarif
│   ├── sonar.sarif
│   └── zap.sarif
└── merged.sarif        # マージ済み（エージェント参照用）
```

---

## 1. gitleaks（共通）

gitleaks は `--report-format sarif` でネイティブ SARIF 出力可能。

```bash
gitleaks detect --source . \
  --report-format sarif \
  --report-path ci-results/sarif/gitleaks.sarif \
  --exit-code 1
```

---

## 2. Trivy（共通）

Trivy も `--format sarif` でネイティブ対応。

```bash
trivy fs --scanners vuln --severity HIGH,CRITICAL \
  --format sarif \
  --output ci-results/sarif/trivy.sarif .
```

---

## F#（Roslyn ErrorLog + SecurityCodeScan）

F# は SonarQube が使えないため、Roslyn の ErrorLog 機能と SecurityCodeScan で SARIF を出力する。

### 1. SecurityCodeScan の追加

```bash
cd ../sales-management/apps/api-fsharp/src/SalesManagement
dotnet add package SecurityCodeScan.VS2019 --version 5.6.7
```

`SalesManagement.fsproj` に追加される `PackageReference` は以下のように `PrivateAssets` を設定して、配布物に含めない：

```xml
<PackageReference Include="SecurityCodeScan.VS2019" Version="5.6.7">
  <PrivateAssets>all</PrivateAssets>
  <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
</PackageReference>
```

### 2. ビルドコマンドで ErrorLog を有効化

```bash
cd ../sales-management/apps/api-fsharp
dotnet build /p:ErrorLog=../ci-results/sarif/fsharp-build.sarif%2cversion=2.1
```

`%2c` は `,` のURLエンコード。`version=2.1` で SARIF v2.1.0 形式を指定する。

### F# 制約: FSharpLint と Fantomas の SARIF 非対応

FSharpLint と Fantomas はネイティブで SARIF を出力しない。JSON/プレーンテキスト出力を SARIF に変換するヘルパースクリプト `scripts/lint-to-sarif.fsx` を同梱する。

```fsharp
// scripts/lint-to-sarif.fsx（抜粋）
// 使い方: dotnet fsi scripts/lint-to-sarif.fsx <input.json> <output.sarif>
open System.IO
open System.Text.Json

let inputPath = fsi.CommandLineArgs.[1]
let outputPath = fsi.CommandLineArgs.[2]

let lints = JsonDocument.Parse(File.ReadAllText inputPath)
// FSharpLint の JSON 出力を SARIF v2.1.0 の results[] に変換
// (詳細は実装時に省略 - ruleId, level, locations を埋める)
```

このスクリプトは Phase 2 では FSharpLint 結果を `ci-results/sarif/fsharplint.sarif` として書き出すが、本 step では Roslyn 出力で代表させる。

---

## Kotlin（detekt + SonarQube → SARIF）

### 1. detekt の SARIF 出力

`build.gradle.kts` の detekt ブロックを更新：

```kotlin
detekt {
    toolVersion = "1.23.7"
    config.setFrom(files("config/detekt/detekt.yml"))
    buildUponDefaultConfig = true
}

tasks.withType<io.gitlab.arturbosch.detekt.Detekt>().configureEach {
    reports {
        html.required.set(true)
        sarif.required.set(true)
        sarif.outputLocation.set(file("../ci-results/sarif/detekt.sarif"))
    }
}
```

### 2. SonarQube → SARIF 変換スクリプト

SonarQube Community Edition は SARIF エクスポートを直接サポートしないため、`api/issues/search` の JSON を `jq` で SARIF に整形する。

`scripts/sonar-to-sarif.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
SONAR_URL="${SONAR_URL:-http://localhost:9000}"
SONAR_TOKEN="${SONAR_TOKEN:?SONAR_TOKEN required}"
PROJECT="${1:-sales-management-kotlin}"
OUT="${2:-ci-results/sarif/sonar.sarif}"

curl -s -u "$SONAR_TOKEN:" \
  "$SONAR_URL/api/issues/search?componentKeys=$PROJECT&ps=500" \
  | jq -f scripts/sonar-to-sarif.jq > "$OUT"
```

`scripts/sonar-to-sarif.jq`（抜粋）:

```jq
{
  "$schema": "https://schemastore.azurewebsites.net/schemas/json/sarif-2.1.0.json",
  "version": "2.1.0",
  "runs": [{
    "tool": { "driver": { "name": "SonarQube" } },
    "results": [.issues[] | {
      "ruleId": .rule,
      "level": (if .severity == "BLOCKER" or .severity == "CRITICAL" then "error" else "warning" end),
      "message": { "text": .message },
      "locations": [{
        "physicalLocation": {
          "artifactLocation": { "uri": .component },
          "region": { "startLine": (.line // 1) }
        }
      }]
    }]
  }]
}
```

### 3. OWASP ZAP の SARIF 出力

ZAP の reporting add-on を有効化して SARIF テンプレートで出力する。

```bash
docker run --rm --network host \
  -v $(pwd)/openapi.yaml:/zap/openapi.yaml \
  -v $(pwd)/ci-results/sarif:/zap/results \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-api-scan.py \
    -t /zap/openapi.yaml \
    -f openapi \
    -z "addonupdate;addoninstall sarifreport" \
    -J /zap/results/zap.sarif
```

---

## SARIF マージ（共通）

複数 SARIF ファイルを 1 つに統合するため、Microsoft の `Sarif.Multitool` を使う。

```bash
# 初回のみ
dotnet tool install -g Sarif.Multitool --version 4.5.4

# マージ実行
sarif merge ci-results/sarif/*.sarif \
  --output-file-path ci-results/merged.sarif \
  --recurse false
```

マージ後の確認：

```bash
$ jq '.runs[] | { tool: .tool.driver.name, results: (.results | length) }' ci-results/merged.sarif
{ "tool": "gitleaks", "results": 0 }
{ "tool": "trivy", "results": 0 }
{ "tool": "detekt", "results": 0 }
{ "tool": "Microsoft (R) Visual F# Compiler", "results": 0 }
{ "tool": "SonarQube", "results": 0 }
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
trivy fs --scanners vuln --severity HIGH,CRITICAL \
  --format sarif --output ci-results/sarif/trivy.sarif .

# F#:
echo "=== ビルド (SARIF) ==="
dotnet build /p:ErrorLog=ci-results/sarif/fsharp-build.sarif%2cversion=2.1

# Kotlin:
echo "=== リンター (SARIF) ==="
gradle detekt   # build.gradle.kts で sarif 出力先指定済み

echo "=== SonarQube → SARIF ==="
bash scripts/sonar-to-sarif.sh sales-management-kotlin ci-results/sarif/sonar.sarif

echo "=== DAST (SARIF) ==="
docker run --rm --network host \
  -v $(pwd)/openapi.yaml:/zap/openapi.yaml \
  -v $(pwd)/ci-results/sarif:/zap/results \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-api-scan.py -t /zap/openapi.yaml -f openapi \
    -z "addonupdate;addoninstall sarifreport" \
    -J /zap/results/zap.sarif

echo "=== SARIF マージ ==="
sarif merge ci-results/sarif/*.sarif \
  --output-file-path ci-results/merged.sarif --recurse false

echo "=== SARIF サマリ ==="
jq '[.runs[].results[]] | group_by(.level) | map({level: .[0].level, count: length})' \
  ci-results/merged.sarif
```

---

## ci.sh の現時点の構成

```bash
#!/bin/bash
set -e

echo "=== ci-results/ 初期化 ==="
mkdir -p ci-results/sarif

echo "=== マイグレーション ==="
echo "=== ビルド ==="
echo "=== フォーマットチェック ==="
echo "=== リンター ==="
echo "=== テスト + カバレッジ ==="

echo "=== シークレット検出 (SARIF) ==="
gitleaks detect --source . \
  --report-format sarif \
  --report-path ci-results/sarif/gitleaks.sarif \
  --exit-code 1

echo "=== SCA (SARIF) ==="
trivy fs --scanners vuln --severity HIGH,CRITICAL \
  --format sarif --output ci-results/sarif/trivy.sarif .

echo "=== SAST (SonarQube) ==="
gradle sonar
bash scripts/sonar-to-sarif.sh sales-management-kotlin ci-results/sarif/sonar.sarif

echo "=== DAST (OWASP ZAP, SARIF) ==="
# (アプリ起動 → ZAP実行 → アプリ停止)

echo "=== SARIF マージ ==="
sarif merge ci-results/sarif/*.sarif \
  --output-file-path ci-results/merged.sarif --recurse false

echo "=== SARIF サマリ ==="
jq '[.runs[].results[]] | group_by(.level) | map({level: .[0].level, count: length})' \
  ci-results/merged.sarif

echo "=== CI完了 ==="
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
