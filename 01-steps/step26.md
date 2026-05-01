# Step 26: 依存関係自動更新（Renovate）

## 目的

### これは何か

Renovate は依存ライブラリのバージョン更新を自動で提案するツール。GitHub App / GitLab App / セルフホスト / ローカル CLI として動く。`build.gradle.kts`、`*.csproj`、`docker-compose.yml`、`renovate.json` などをスキャンし、新しいバージョンが出ていれば「アップデート PR」を自動生成する。

### なぜやるのか

- Step 10 の Trivy が脆弱性を検出しても、人間が手で対応すると遅い。Renovate を組み合わせれば「脆弱性検出 → 即更新 PR」が自動化される
- 依存更新を後回しにすると指数関数的に痛くなる（一度に大量のメジャーアップデートを処理することになる）
- AIエージェントの skill（特定ライブラリ API の知識）は古い API を覚えていることがある。常に最新化しておけばエージェントの skill ドリフトを抑えられる

### 何がうれしいのか

- パッチ・マイナーアップデートが自動マージされ、メンテナンス工数が激減
- メジャーアップデートはラベル付きで提案され、人間が判断
- Trivy の SARIF 出力（Step 21）と組み合わせて、脆弱性のある依存を優先的に更新できる

## 完了条件

```bash
$ npx renovate --dry-run --platform=local --autodiscover=false
DEBUG: 12 dependencies found
INFO: 3 updates available:
  - Giraffe 7.0.2 → 7.0.4 (minor)
  - kotlinx.serialization 1.6.0 → 1.7.0 (minor)
  - postgres:16-alpine → 17-alpine (major)
$ echo $?
0

$ ls renovate-out/
upgrades.json  log.txt
```

---

## 1. renovate.json の作成

リポジトリルートに `renovate.json` を配置：

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:recommended",
    ":dependencyDashboard"
  ],
  "schedule": ["before 6am on monday"],
  "prHourlyLimit": 5,
  "prConcurrentLimit": 10,
  "packageRules": [
    {
      "matchUpdateTypes": ["patch", "minor"],
      "automerge": true,
      "automergeType": "branch"
    },
    {
      "matchPackagePatterns": ["*"],
      "matchUpdateTypes": ["major"],
      "automerge": false,
      "addLabels": ["major-upgrade"]
    },
    {
      "matchManagers": ["docker-compose", "dockerfile"],
      "addLabels": ["docker"]
    },
    {
      "matchManagers": ["nuget"],
      "addLabels": ["nuget"]
    },
    {
      "matchManagers": ["gradle"],
      "addLabels": ["gradle"]
    }
  ],
  "vulnerabilityAlerts": {
    "enabled": true,
    "labels": ["security"],
    "automerge": true,
    "schedule": ["at any time"]
  }
}
```

### 主要設定の意味

| 設定 | 意味 |
|---|---|
| `extends: config:recommended` | コミュニティ推奨のデフォルト設定を継承 |
| `:dependencyDashboard` | "Dependency Dashboard" issue を作って全更新を一覧化 |
| `schedule: before 6am on monday` | 月曜 6am 前にだけ PR を作る（ノイズ抑制） |
| `prHourlyLimit: 5` | 1時間あたりの PR 数上限 |
| `vulnerabilityAlerts.automerge: true` | 脆弱性絡みは即時自動マージ |

---

## 2. ローカル dry-run

GitHub App を使わずに、ローカルで「何が更新できるか」だけを確認する。

```bash
mkdir -p renovate-out
npm install -g renovate

LOG_LEVEL=debug \
RENOVATE_PLATFORM=local \
RENOVATE_AUTODISCOVER=false \
  renovate --dry-run > renovate-out/log.txt 2>&1

# 検出された更新を構造化して取得
grep -oP 'depName: \K[^,]+' renovate-out/log.txt | sort -u > renovate-out/dependencies.txt
```

---

## 3. Trivy 連携: 脆弱性のある依存を優先更新

Step 21 で Trivy の SARIF 出力が `ci-results/sarif/trivy.sarif` にある。ここから CVE の影響を受けるパッケージ名を抽出し、`renovate.json` の優先度を動的に上げる。

`scripts/prioritize-from-trivy.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
SARIF="${1:-ci-results/sarif/trivy.sarif}"
RENOVATE_CONFIG="${2:-renovate.json}"

# Trivy SARIF から脆弱なパッケージ名を抽出
VULN_PKGS=$(jq -r '
  .runs[].results[]
  | .properties["package_name"] // empty
' "$SARIF" | sort -u)

if [ -z "$VULN_PKGS" ]; then
  echo "脆弱性のあるパッケージなし"
  exit 0
fi

# packageRules に優先度ルールを追加
TMP=$(mktemp)
jq --argjson pkgs "$(echo "$VULN_PKGS" | jq -R . | jq -s .)" '
  .packageRules += [{
    "matchPackageNames": $pkgs,
    "automerge": true,
    "schedule": ["at any time"],
    "addLabels": ["security-critical"],
    "prPriority": 10
  }]
' "$RENOVATE_CONFIG" > "$TMP"
mv "$TMP" "$RENOVATE_CONFIG"

echo "Trivy の脆弱パッケージ ${VULN_PKGS} を Renovate の優先更新リストに追加"
```

このスクリプトをCI で Trivy 実行直後に呼び出すと、脆弱性検出 → 自動的に優先更新ルール追加 → 次の Renovate 実行で即更新 PR、というループが組める。

---

## 4. ホスト型 Renovate（GitHub App）への移行（任意）

PoC では GitHub Actions を使わない方針だが、本番運用するなら GitHub App として動かすのが楽：

1. https://github.com/apps/renovate からインストール
2. `renovate.json` を main にコミット
3. Renovate が自動で onboarding PR を作成
4. マージしたら定期スケジュールで動き出す

セルフホストする場合は Mend Renovate Community Edition を使う（Docker イメージあり）。

---

## ci.sh への追加

```bash
echo "=== 脆弱性パッケージを Renovate 優先化 ==="
bash scripts/prioritize-from-trivy.sh \
  ci-results/sarif/trivy.sarif renovate.json || true

echo "=== 依存更新チェック (dry-run) ==="
mkdir -p renovate-out
RENOVATE_PLATFORM=local \
RENOVATE_AUTODISCOVER=false \
  npx --yes renovate --dry-run > ci-results/renovate.log 2>&1 || true

# 更新候補数をサマリ
UPDATE_COUNT=$(grep -c "Update available:" ci-results/renovate.log || true)
echo "更新候補: $UPDATE_COUNT 件"
```

`|| true` を付けているのは「更新候補が見つかっただけで CI を失敗させない」ため。CI を落とすか落とさないかは運用判断。

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

echo "=== ミューテーションテスト ==="
(cd ../sales-management/apps/api-fsharp && dotnet stryker)
cd kotlin && gradle pitest && cd ..

echo "=== アーキテクチャ適合性 ==="
(cd ../sales-management/apps/api-fsharp && dotnet test --filter "Category=Architecture")
cd kotlin && gradle test --tests "*ArchitectureTest*" && cd ..

echo "=== コントラクトテスト (Pact) ==="
(cd ../sales-management/apps/api-fsharp && dotnet test --filter "Category=Pact")
cd kotlin && gradle pactVerify && cd ..

echo "=== シークレット検出 (SARIF) ==="
gitleaks detect --source . \
  --report-format sarif --report-path ci-results/sarif/gitleaks.sarif --exit-code 1

echo "=== SCA (SARIF) ==="
trivy fs --scanners vuln --severity HIGH,CRITICAL \
  --format sarif --output ci-results/sarif/trivy.sarif .

echo "=== SAST (SonarQube) ==="
gradle sonar
bash scripts/sonar-to-sarif.sh sales-management-kotlin ci-results/sarif/sonar.sarif

echo "=== DAST (OWASP ZAP, SARIF) ==="
# (アプリ起動 → ZAP実行 → アプリ停止)

echo "=== SBOM 生成 ==="
cd ../sales-management/apps/api-fsharp && \
  dotnet CycloneDX src/SalesManagement/SalesManagement.fsproj \
    --json --output ../ci-results --filename sbom-fsharp.cdx.json && \
  cd ..
cd kotlin && gradle cyclonedxBom && cd ..

echo "=== 脆弱性パッケージを Renovate 優先化 ==="
bash scripts/prioritize-from-trivy.sh \
  ci-results/sarif/trivy.sarif renovate.json || true

echo "=== 依存更新チェック (dry-run) ==="
RENOVATE_PLATFORM=local RENOVATE_AUTODISCOVER=false \
  npx --yes renovate --dry-run > ci-results/renovate.log 2>&1 || true

echo "=== SARIF マージ ==="
sarif merge ci-results/sarif/*.sarif \
  --output-file-path ci-results/merged.sarif --recurse false

echo "=== CI完了 ==="
```

---

## RALPHループにおける位置づけ

| 層 | 役割 |
|---|---|
| メンテナンス自動化 | 依存ライブラリの陳腐化を防ぐ |
| Skills のリフレッシュ | エージェントが古い API を覚えていることを抑制 |

Trivy（Step 10/21）の SARIF を入力に、Renovate の `renovate.json` を出力として動的に変更するループが Step 26 の核心。これにより「脆弱性検出 → 自動修正 PR」が完全に閉じる。Step 30 の RALPH ループから見ると、依存メンテナンスをエージェントが意識しなくて済む、つまり PRD タスクに集中できる効果がある。

---

## 次のステップ

Step 26が完了したら [Step 27: OpenTelemetryエージェントトレース](./step27.md) へ進む。
