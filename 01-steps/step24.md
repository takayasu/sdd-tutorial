# Step 24: APIコントラクトテスト（Pact）

## 目的

### これは何か

Pact は API の **コンシューマー（呼ぶ側）** と **プロバイダー（実装側）** の間の「契約」を JSON で記述し、双方が契約に従っていることを機械的に検証する仕組み。コンシューマーが「私はこのリクエストを送り、このレスポンスを期待する」を Pact ファイルとして書き、プロバイダー側で再生して実装が満たすかをテストする。

Pact Broker は契約と検証結果を集中管理するサーバー。`can-i-deploy` というコマンドで「このバージョンを本番に出して大丈夫か」を機械判定できる。

### なぜやるのか

- Phase 1 では F# 版と Kotlin 版が同じドメインを実装している。両者の API 仕様が「だいたい同じ」状態でも、フィールド名や Enum 値が微妙に違っていてもテストは通ってしまう
- 将来 React フロントエンドを追加するとき、フロントが期待する API シグネチャと両言語の実装が一致していることを継続的に保証する必要がある
- マルチエージェント開発では、片方の言語実装を変更したエージェントが、もう片方を壊していても気づけない。コントラクトテストはそれを止める唯一の機械的手段

### 何がうれしいのか

- F# / Kotlin の両プロバイダーが同一の Consumer Pact を満たすことを CI で検証できる
- `can-i-deploy` でデプロイ可否がスクリプト判定でき、エージェントの自律デプロイ判断に組み込める
- 将来 React フロントを足すときも同じ仕組みを流用できる

## 完了条件

```bash
# 1. Pact Broker 起動
$ docker compose -f docker-compose.harness.yml up -d pact-broker
$ curl -s http://localhost:9292/diagnostic/status/heartbeat
{"ok":true}

# 2. F# Provider 検証
$ cd ../sales-management/apps/api-fsharp
$ dotnet test --filter "Category=Pact"
  Passed: 5
$ echo $?
0

# 3. Kotlin Provider 検証
$ cd kotlin
$ gradle pactVerify
BUILD SUCCESSFUL
$ echo $?
0

# 4. can-i-deploy
$ docker run --rm pactfoundation/pact-cli:latest \
    pact-broker can-i-deploy \
    --broker-base-url http://host.docker.internal:9292 \
    --pacticipant sales-management-fsharp \
    --version $(git rev-parse HEAD) \
    --to-environment production
Computer says yes
```

---

## 0. ハーネス共有 docker-compose の導入

Step 21-23 までは言語別の `docker-compose.yml` を使ってきたが、Step 24 で導入する Pact Broker と Step 27 の Jaeger は言語横断のインフラなので、リポジトリルートに新規 `docker-compose.harness.yml` を導入する。

`/docker-compose.harness.yml`:

```yaml
version: "3.9"
services:
  pact-broker:
    image: pactfoundation/pact-broker:latest
    ports: ["9292:9292"]
    environment:
      PACT_BROKER_DATABASE_URL: postgres://pact:pact@pact-broker-db/pact
      PACT_BROKER_BASIC_AUTH_USERNAME: admin
      PACT_BROKER_BASIC_AUTH_PASSWORD: admin
    depends_on: [pact-broker-db]

  pact-broker-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: pact
      POSTGRES_PASSWORD: pact
      POSTGRES_DB: pact
    volumes:
      - pact_pgdata:/var/lib/postgresql/data

volumes:
  pact_pgdata:
```

---

## 1. Consumer Pact の準備（仮 React フロント）

実フロントエンドが未実装のため、手書きの仮 Consumer Pact を同梱する。

`pacts/frontend-sales-management.json`:

```json
{
  "consumer": { "name": "frontend-app" },
  "provider": { "name": "sales-management" },
  "interactions": [
    {
      "description": "ロット取得",
      "request": {
        "method": "GET",
        "path": "/lots/2024-A-001"
      },
      "response": {
        "status": 200,
        "headers": { "Content-Type": "application/json" },
        "body": {
          "lotNumber": "2024-A-001",
          "status": "manufacturing"
        },
        "matchingRules": {
          "$.body.lotNumber": { "match": "regex", "regex": "\\d{4}-[A-Z]-\\d{3}" },
          "$.body.status": {
            "match": "regex",
            "regex": "manufacturing|manufactured|shipping_instructed|shipped"
          }
        }
      }
    },
    {
      "description": "製造完了指示",
      "request": {
        "method": "POST",
        "path": "/lots/2024-A-001/complete-manufacturing",
        "body": { "date": "2024-04-01" }
      },
      "response": {
        "status": 200,
        "body": {
          "status": "manufactured",
          "manufacturingCompletedDate": "2024-04-01"
        }
      }
    }
  ],
  "metadata": {
    "pactSpecification": { "version": "3.0.0" }
  }
}
```

このファイルを Pact Broker に publish する：

```bash
docker run --rm -v $(pwd)/pacts:/pacts \
  pactfoundation/pact-cli:latest \
  pact-broker publish /pacts \
    --broker-base-url http://host.docker.internal:9292 \
    --broker-username admin --broker-password admin \
    --consumer-app-version 1.0.0 \
    --branch main
```

実フロントエンド（React + `@pact-foundation/pact`）が登場した時、このファイルは Pact 消費者テストから自動生成されるものに置き換える。

---

## F#（PactNet）

### 1. パッケージ追加

```bash
cd ../sales-management/apps/api-fsharp/tests/SalesManagement.Tests
dotnet add package PactNet --version 5.0.0
```

### 2. Provider 検証テスト

`fsharp/tests/SalesManagement.Tests/PactProviderTests.fs`:

```fsharp
module SalesManagement.Tests.PactProviderTests

open System
open Xunit
open PactNet
open PactNet.Verifier
open SalesManagement.Tests.TestServer

[<Trait("Category", "Pact")>]
type PactProviderTests() =

    [<Fact>]
    member _.``F# provider satisfies frontend-app pact``() =
        use server = startTestServer ()  // テスト用 Giraffe サーバ起動
        use verifier = new PactVerifier(PactVerifierConfig())

        verifier
            .ServiceProvider(
                "sales-management-fsharp",
                Uri server.BaseAddress)
            .WithPactBrokerSource(
                Uri "http://localhost:9292",
                fun o ->
                    o.BasicAuthentication("admin", "admin")
                     .PublishVerificationResults(
                        ProviderVersion = (Environment.GetEnvironmentVariable "GIT_SHA"),
                        BranchName = "main")
                |> ignore)
            .WithProviderStateUrl(Uri (server.BaseAddress.ToString() + "provider-states"))
            .Verify()
```

### 3. State setup フック

`fsharp/tests/SalesManagement.Tests/StateHandlers.fs` で「ロット 2024-A-001 が製造中で存在する」のような前提状態をセットアップ：

```fsharp
module SalesManagement.Tests.StateHandlers

let setupLotInManufacturing (db: NpgsqlConnection) =
    db |> Db.newCommand """
        INSERT INTO lot (lot_number_year, lot_number_location, lot_number_seq, status)
        VALUES (2024, 'A', 1, 'manufacturing')
        ON CONFLICT DO NOTHING"""
       |> Db.exec
```

API サーバーは `/provider-states` エンドポイントを公開し、Pact 検証時に状態名を受け取って該当ハンドラを呼ぶ。

---

## Kotlin（pact-jvm）

### 1. プラグイン追加

`build.gradle.kts`:

```kotlin
plugins {
    id("au.com.dius.pact") version "4.6.10"
}

pact {
    serviceProviders {
        create("sales-management-kotlin") {
            protocol = "http"
            host = "localhost"
            port = 8080

            hasPactsFromPactBroker(
                "http://localhost:9292",
                authentication = listOf("Basic", "admin", "admin")
            )
        }
    }

    publish {
        pactBrokerUrl = "http://localhost:9292"
        pactBrokerAuthenticationCredentials = listOf("admin", "admin")
    }
}

dependencies {
    testImplementation("au.com.dius.pact.provider:junit5:4.6.10")
    testImplementation("au.com.dius.pact.provider:junit5spring:4.6.10")
}
```

### 2. Provider 検証テスト

`kotlin/src/test/kotlin/salesmanagement/pact/ProviderPactTest.kt`:

```kotlin
package salesmanagement.pact

import au.com.dius.pact.provider.junit5.HttpTestTarget
import au.com.dius.pact.provider.junit5.PactVerificationContext
import au.com.dius.pact.provider.junitsupport.Provider
import au.com.dius.pact.provider.junitsupport.State
import au.com.dius.pact.provider.junitsupport.loader.PactBroker
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.TestTemplate
import org.junit.jupiter.api.extension.ExtendWith
import au.com.dius.pact.provider.junit5.PactVerificationInvocationContextProvider

@Provider("sales-management-kotlin")
@PactBroker(
    url = "http://localhost:9292",
    authentication = au.com.dius.pact.core.support.auth.Auth.BasicAuthentication("admin", "admin")
)
@ExtendWith(PactVerificationInvocationContextProvider::class)
class ProviderPactTest {

    @BeforeEach
    fun setUp(context: PactVerificationContext) {
        context.target = HttpTestTarget("localhost", 8080, "/")
    }

    @TestTemplate
    fun pactVerificationTestTemplate(context: PactVerificationContext) {
        context.verifyInteraction()
    }

    @State("ロット 2024-A-001 が製造中で存在する")
    fun lotInManufacturing() {
        // テスト DB に該当データを INSERT
    }
}
```

### 3. 実行

```bash
# Kotlin アプリケーションを起動した状態で
cd kotlin
gradle pactVerify
```

---

## can-i-deploy デモ

特定のバージョンが本番デプロイ可能かを Pact Broker に問い合わせる：

```bash
# F# サービスを「コミット abc123 で本番に出して大丈夫か」
docker run --rm pactfoundation/pact-cli:latest \
  pact-broker can-i-deploy \
  --broker-base-url http://host.docker.internal:9292 \
  --broker-username admin --broker-password admin \
  --pacticipant sales-management-fsharp \
  --version $(git rev-parse HEAD) \
  --to-environment production

# 出力例:
# Computer says yes \o/
#
#  CONSUMER     | C.VERSION | PROVIDER             | P.VERSION | SUCCESS?
# --------------|-----------|----------------------|-----------|---------
#  frontend-app | 1.0.0     | sales-management-fs  | abc123    | true
```

エージェントが自律デプロイする場合、`can-i-deploy` の終了コードを判定材料に使える。

---

## ci.sh への追加

```bash
echo "=== Pact Broker 起動チェック ==="
curl -fs http://localhost:9292/diagnostic/status/heartbeat > /dev/null \
  || (echo "Pact Broker 未起動: docker compose -f docker-compose.harness.yml up -d" && exit 1)

echo "=== Consumer Pact publish ==="
docker run --rm -v $(pwd)/pacts:/pacts \
  pactfoundation/pact-cli:latest \
  pact-broker publish /pacts \
    --broker-base-url http://host.docker.internal:9292 \
    --broker-username admin --broker-password admin \
    --consumer-app-version $(git rev-parse HEAD) \
    --branch $(git branch --show-current)

echo "=== Provider 検証 ==="
# F#:
(cd ../sales-management/apps/api-fsharp && dotnet test --filter "Category=Pact")
# Kotlin:
cd kotlin && gradle pactVerify && cd ..

echo "=== can-i-deploy ==="
docker run --rm pactfoundation/pact-cli:latest \
  pact-broker can-i-deploy \
    --broker-base-url http://host.docker.internal:9292 \
    --broker-username admin --broker-password admin \
    --pacticipant sales-management-fsharp \
    --version $(git rev-parse HEAD) \
    --to-environment production
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

echo "=== SARIF マージ ==="
sarif merge ci-results/sarif/*.sarif \
  --output-file-path ci-results/merged.sarif --recurse false

echo "=== CI完了 ==="
```

---

## RALPHループにおける位置づけ

| 層 | 役割 |
|---|---|
| Protocols | コンシューマーとプロバイダーの間の取り決めを機械化 |
| 言語横断 SSoT | F# 版と Kotlin 版が同一仕様を満たすことの単一の真実源 |

**マルチエージェント開発における核心的な意義**: エージェント A が F# を変更し、エージェント B が Kotlin を別々に変更しても、両方が同じ Consumer Pact に対して検証されるため、「片方だけ辻褄合わせする」逃げ道が塞がれる。`can-i-deploy` はエージェントの自律デプロイ判断（Step 30 の RALPH ループ）における主要な品質ゲートになる。

---

## 次のステップ

Step 24が完了したら [Step 25: SBOM生成](./step25.md) へ進む。
