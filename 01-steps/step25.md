# Step 25: SBOM生成（CycloneDX）

## 目的

### これは何か

SBOM（Software Bill of Materials）はプロジェクトが使っている全ライブラリの「部品表」。CycloneDX は OASIS 標準の SBOM フォーマット（JSON / XML）。`ci-results/sbom.cdx.json` として 1 ファイルに「使っているライブラリ全部」とそのバージョン・ライセンス・PURL を出力する。

### なぜやるのか

- Step 10 の Trivy は脆弱性をスキャンするが、SBOM は「いま何を使っているか」のスナップショットそのもの。両者は補完関係
- 米国 EO 14028 以降、政府調達で SBOM 提出が事実上必須になりつつある。コンプライアンス対応として
- 将来新しい脆弱性が公開されたとき、過去のビルドの SBOM を遡って「影響を受けたバージョンはどれか」を即座に特定できる
- エージェントが新しい依存をうっかり追加したとき、SBOM の差分で検出できる

### 何がうれしいのか

- 「いまこの依存を使っている」が機械可読な形で永続化される
- Dependency-Track などの SBOM 監視サービスにアップロードすれば、新規 CVE 公開時に自動で警告
- Step 26 の Renovate と組み合わせれば「どの依存が古いか」が常に見える

## 完了条件

```bash
$ ./ci.sh
=== CI完了 ===

$ ls ci-results/sbom*.cdx.json
ci-results/sbom-fsharp.cdx.json
ci-results/sbom-kotlin.cdx.json

$ jq '.components | length' ci-results/sbom-fsharp.cdx.json
38

$ jq '.metadata.tools[].name' ci-results/sbom-fsharp.cdx.json
"CycloneDX module for .NET"

$ jq '.components[] | { name: .name, version: .version, license: (.licenses[0].license.id // "?") }' \
  ci-results/sbom-fsharp.cdx.json | head -10
```

---

## F#（CycloneDX-dotnet）

### 1. インストール

```bash
dotnet tool install -g CycloneDX --version 4.0.2
```

### 2. SBOM 生成

```bash
cd ../sales-management/apps/api-fsharp
dotnet CycloneDX src/SalesManagement/SalesManagement.fsproj \
  --json \
  --output ../ci-results \
  --filename sbom-fsharp.cdx.json
```

主要オプション：

| オプション | 効果 |
|---|---|
| `--json` | JSON 形式で出力（デフォルトは XML） |
| `--include-dev` | 開発依存（テスト用パッケージ）も含める |
| `--exclude-test-projects` | テストプロジェクト自体を除外 |
| `--scan-projects` | ソリューション全体をスキャン |

### 3. 検証

```bash
# CycloneDX のスキーマで検証
docker run --rm -v $(pwd)/ci-results:/data \
  cyclonedx/cyclonedx-cli:latest \
  validate --input-file /data/sbom-fsharp.cdx.json
```

---

## Kotlin（cyclonedx-gradle-plugin）

### 1. プラグイン追加

`build.gradle.kts`:

```kotlin
plugins {
    id("org.cyclonedx.bom") version "1.10.0"
}

tasks.cyclonedxBom {
    setIncludeConfigs(listOf("runtimeClasspath"))
    setSkipConfigs(listOf("compileClasspath", "testCompileClasspath"))
    outputFormat.set("json")
    outputName.set("sbom-kotlin.cdx")
    destination.set(file("../ci-results"))
    schemaVersion.set("1.5")
    includeMetadataResolution.set(true)
    includeLicenseText.set(false)
}
```

### 2. SBOM 生成

```bash
cd kotlin
gradle cyclonedxBom
```

出力先は `ci-results/sbom-kotlin.cdx.json`。

### 3. 検証

```bash
docker run --rm -v $(pwd)/ci-results:/data \
  cyclonedx/cyclonedx-cli:latest \
  validate --input-file /data/sbom-kotlin.cdx.json
```

---

## Optional: Dependency-Track 連携

SBOM は単独でも価値があるが、Dependency-Track にアップロードすれば「新規 CVE 公開時に自動警告」「ライセンス違反検出」「コンポーネント階層の可視化」ができる。

### docker-compose に追加

`/docker-compose.harness.yml` に追記：

```yaml
  dt-apiserver:
    image: dependencytrack/apiserver:latest
    ports: ["8081:8080"]
    volumes:
      - dt_data:/data

  dt-frontend:
    image: dependencytrack/frontend:latest
    ports: ["8082:8080"]
    environment:
      API_BASE_URL: "http://localhost:8081"

volumes:
  dt_data:
```

### SBOM アップロード

```bash
# Dependency-Track の API トークンを取得後
DT_KEY="odt_xxxxx"
DT_PROJECT="11111111-2222-3333-4444-555555555555"  # 事前作成

curl -s -X POST -H "X-API-Key: $DT_KEY" \
  -F "project=$DT_PROJECT" \
  -F "bom=@ci-results/sbom-fsharp.cdx.json" \
  http://localhost:8081/api/v1/bom

# ブラウザで http://localhost:8082 を開いて閲覧
```

---

## ci.sh への追加

```bash
echo "=== SBOM 生成 ==="
# F#:
cd ../sales-management/apps/api-fsharp && \
  dotnet CycloneDX src/SalesManagement/SalesManagement.fsproj \
    --json --output ../ci-results --filename sbom-fsharp.cdx.json && \
  cd ..

# Kotlin:
cd kotlin && gradle cyclonedxBom && cd ..

echo "=== SBOM サマリ ==="
jq '{ tool: .metadata.tools[0].name, components: (.components | length) }' \
  ci-results/sbom-fsharp.cdx.json
jq '{ tool: .metadata.tools[0].name, components: (.components | length) }' \
  ci-results/sbom-kotlin.cdx.json
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

echo "=== ミューテーションテスト ==="
(cd ../sales-management/apps/api-fsharp && dotnet stryker)
cd kotlin && gradle pitest && cd ..

echo "=== アーキテクチャ適合性 ==="
(cd ../sales-management/apps/api-fsharp && dotnet test --filter "Category=Architecture")
cd kotlin && gradle test --tests "*ArchitectureTest*" && cd ..

echo "=== コントラクトテスト (Pact) ==="
curl -fs http://localhost:9292/diagnostic/status/heartbeat > /dev/null \
  || (echo "Pact Broker 未起動" && exit 1)
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

echo "=== SARIF マージ ==="
sarif merge ci-results/sarif/*.sarif \
  --output-file-path ci-results/merged.sarif --recurse false

echo "=== CI完了 ==="
```

---

## RALPHループにおける位置づけ

| 層 | 役割 |
|---|---|
| Memory | ある時点の依存スナップショット（時系列で追える） |
| Compliance | 規制・セキュリティ要件への対応物証 |

エージェントが過去 SBOM と現在を比較すれば、「自分が追加した依存」を構造的に説明できる。Step 26 の Renovate と組み合わせて「依存進化のログ」になる：

- **Step 25 (SBOM)**: 「いま何を使っているか」のスナップショット
- **Step 26 (Renovate)**: 「どれを更新すべきか」の提案

両者が揃って、エージェントの依存管理が完結する。

---

## 次のステップ

Step 25が完了したら [Step 26: 依存関係自動更新](./step26.md) へ進む。
