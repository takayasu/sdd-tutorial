# Step 25: SBOM生成（CycloneDX）

## 目的

### これは何か

SBOM（Software Bill of Materials）はプロジェクトが使っている全ライブラリの「部品表」。CycloneDX は OASIS 標準の SBOM フォーマット（JSON / XML）。`ci-results/sbom-backend.cdx.json` と `ci-results/sbom-frontend.cdx.json` として「使っているライブラリ全部」とそのバージョン・ライセンス・PURL を出力する。

### なぜやるのか

- Step 10 の pip-audit/pnpm audit は脆弱性をスキャンするが、SBOM は「いま何を使っているか」のスナップショットそのもの。両者は補完関係
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
ci-results/sbom-backend.cdx.json
ci-results/sbom-frontend.cdx.json

$ jq '.components | length' ci-results/sbom-backend.cdx.json
52

$ jq '.metadata.tools[].name' ci-results/sbom-backend.cdx.json
"cyclonedx-bom"

$ jq '.components[] | { name: .name, version: .version, license: (.licenses[0].license.id // "?") }' \
  ci-results/sbom-backend.cdx.json | head -10
```

---

## Python（cyclonedx-bom）

### 1. インストール

```bash
cd backend
uv pip install cyclonedx-bom
```

### 2. SBOM 生成

```bash
cd backend
uv run cyclonedx-bom \
  environment \
  --of JSON \
  --outfile ../ci-results/sbom-backend.cdx.json
cd ..
```

主要オプション：

| オプション | 効果 |
|---|---|
| `environment` | 現在の仮想環境からスキャン |
| `--of JSON` | JSON 形式で出力（デフォルトは XML） |
| `--schema-version 1.5` | CycloneDX スキーマバージョン指定 |
| `--outfile` | 出力先ファイルパス |

### 3. 検証

```bash
docker run --rm -v $(pwd)/ci-results:/data \
  cyclonedx/cyclonedx-cli:latest \
  validate --input-file /data/sbom-backend.cdx.json
```

---

## TypeScript / Node.js（@cyclonedx/cyclonedx-npm）

### 1. インストール

```bash
cd frontend
pnpm add -D @cyclonedx/cyclonedx-npm
```

### 2. SBOM 生成

```bash
cd frontend
pnpm cyclonedx-npm \
  --output-format JSON \
  --output-file ../ci-results/sbom-frontend.cdx.json \
  --package-lock-only
cd ..
```

`package.json` にスクリプト追加：

```json
{
  "scripts": {
    "sbom": "cyclonedx-npm --output-format JSON --output-file ../ci-results/sbom-frontend.cdx.json"
  }
}
```

### 3. 検証

```bash
docker run --rm -v $(pwd)/ci-results:/data \
  cyclonedx/cyclonedx-cli:latest \
  validate --input-file /data/sbom-frontend.cdx.json
```

---

## Optional: Dependency-Track 連携

SBOM は単独でも価値があるが、Dependency-Track にアップロードすれば「新規 CVE 公開時に自動警告」「ライセンス違反検出」「コンポーネント階層の可視化」ができる。

`docker-compose.harness.yml` に追記：

```yaml
  dt-apiserver:
    image: dependencytrack/apiserver:latest
    ports: ["8083:8080"]
    volumes:
      - dt_data:/data

  dt-frontend:
    image: dependencytrack/frontend:latest
    ports: ["8084:8080"]
    environment:
      API_BASE_URL: "http://localhost:8083"

volumes:
  dt_data:
```

SBOM アップロード：

```bash
DT_KEY="odt_xxxxx"
DT_PROJECT="11111111-2222-3333-4444-555555555555"

curl -s -X POST -H "X-API-Key: $DT_KEY" \
  -F "project=$DT_PROJECT" \
  -F "bom=@ci-results/sbom-backend.cdx.json" \
  http://localhost:8083/api/v1/bom
```

---

## ci.sh への追加

```bash
echo "=== SBOM 生成 ==="
cd backend
uv run cyclonedx-bom environment \
  --of JSON \
  --outfile ../ci-results/sbom-backend.cdx.json
cd ..

cd frontend
pnpm cyclonedx-npm \
  --output-format JSON \
  --output-file ../ci-results/sbom-frontend.cdx.json \
  --package-lock-only
cd ..

echo "=== SBOM サマリ ==="
jq '{ tool: .metadata.tools[0].name, components: (.components | length) }' \
  ci-results/sbom-backend.cdx.json
jq '{ tool: .metadata.tools[0].name, components: (.components | length) }' \
  ci-results/sbom-frontend.cdx.json
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
