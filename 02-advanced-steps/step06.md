# Step 6: HTTPクライアント + レジリエンス

## 目的

### これは何か

外部APIを呼び出すHTTPクライアントを導入し、リトライ（再試行）とサーキットブレーカー（障害時の遮断）を組み込む。外部APIのモックとして WireMock を docker-compose で起動し、正常応答・遅延・エラーを自由にシミュレートする。

### なぜやるのか

- 業務システムは単独で完結しない。外部の在庫管理システムや決済システムとHTTP通信することが多い
- 外部システムは「たまに遅い」「たまに落ちる」。リトライなしだと、一時的な障害で業務が止まる
- サーキットブレーカーがないと、外部システムが落ちているのに何度もリクエストを送り続け、自分のシステムまで巻き添えで遅くなる（カスケード障害）

### 何がうれしいのか

- WireMock で「1回目は500、2回目は200」といったシナリオを簡単に作れる。実際の外部APIがなくてもレジリエンスの動作を確認できる
- 外部APIが一時的にエラーを返しても、自動的にリトライして成功する
- 外部APIが長時間ダウンしている場合、サーキットブレーカーが即座にエラーを返す。無駄な待ち時間がなくなる

## 事前準備: WireMock を docker-compose に追加

### docker-compose.yml に追加

```yaml
  wiremock:
    image: wiremock/wiremock:3.5.4
    ports:
      - "8181:8080"
    volumes:
      - ./wiremock:/home/wiremock
```

### WireMock のスタブ定義を作成

```bash
mkdir -p wiremock/mappings
```

正常応答のスタブ:

```json
// wiremock/mappings/price-check-ok.json
{
  "request": {
    "method": "GET",
    "urlPathPattern": "/api/pricing/.*"
  },
  "response": {
    "status": 200,
    "headers": { "Content-Type": "application/json" },
    "jsonBody": {
      "basePrice": 10000,
      "adjustmentRate": 1.05,
      "source": "external-pricing-api"
    }
  }
}
```

エラー応答のスタブ（リトライテスト用）:

```json
// wiremock/mappings/price-check-error.json
{
  "request": {
    "method": "GET",
    "urlPathPattern": "/api/pricing/error/.*"
  },
  "response": {
    "status": 500,
    "headers": { "Content-Type": "application/json" },
    "jsonBody": { "error": "Internal Server Error" }
  }
}
```

遅延応答のスタブ（タイムアウトテスト用）:

```json
// wiremock/mappings/price-check-slow.json
{
  "request": {
    "method": "GET",
    "urlPathPattern": "/api/pricing/slow/.*"
  },
  "response": {
    "status": 200,
    "fixedDelayMilliseconds": 5000,
    "headers": { "Content-Type": "application/json" },
    "jsonBody": { "basePrice": 10000 }
  }
}
```

```bash
docker compose up -d wiremock

# 動作確認
curl http://localhost:8181/api/pricing/lot-001
# → {"basePrice":10000,"adjustmentRate":1.05,"source":"external-pricing-api"}

curl http://localhost:8181/api/pricing/error/lot-001
# → 500 Internal Server Error

curl http://localhost:8181/api/pricing/slow/lot-001
# → 5秒後に応答
```

## 完了条件

### HTTPクライアントの確認

1. アプリ経由で WireMock の正常スタブを呼び出し、レスポンスが返ること:

```bash
TOKEN=$(./scripts/get-token.sh test-operator)

curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8080/api/external/price-check?lotId=2024-A-001"
# → 200 OK
# {"basePrice":10000,"adjustmentRate":1.05,"source":"external-pricing-api"}
```

### リトライの確認

2. WireMock の Scenario 機能を使い、「1回目は500、2回目は200」を再現する。WireMock の管理APIでシナリオを動的に設定できる:

```bash
# WireMock のシナリオ設定（1回目→500, 2回目→200）
curl -X POST http://localhost:8181/__admin/mappings -H "Content-Type: application/json" -d '{
  "scenarioName": "retry-test",
  "requiredScenarioState": "Started",
  "newScenarioState": "second-attempt",
  "request": { "method": "GET", "urlPath": "/api/pricing/retry-test" },
  "response": { "status": 500 }
}'

curl -X POST http://localhost:8181/__admin/mappings -H "Content-Type: application/json" -d '{
  "scenarioName": "retry-test",
  "requiredScenarioState": "second-attempt",
  "request": { "method": "GET", "urlPath": "/api/pricing/retry-test" },
  "response": { "status": 200, "jsonBody": { "basePrice": 10000 } }
}'

# アプリ経由で呼び出し → リトライにより成功
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8080/api/external/price-check?lotId=retry-test"
# → 200 OK（リトライにより成功）
```

3. ログにリトライの記録が出力されていること:

```
{"level":"Warning","message":"Retry attempt 1 for external-pricing-api","statusCode":500}
{"level":"Information","message":"external-pricing-api succeeded after 1 retry"}
```

### タイムアウトの確認

4. WireMock の遅延スタブを呼び出し、タイムアウトが発生すること:

```bash
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8080/api/external/price-check?lotId=slow/lot-001"
# → 502 Bad Gateway（タイムアウト）
# {"type":"external-service-error","detail":"Request timed out after 3000ms"}
```

### サーキットブレーカーの確認

5. エラースタブに対して連続でリクエストを送り、サーキットブレーカーが開くこと:

```bash
# 5回連続でエラーを発生させる
for i in $(seq 1 5); do
  curl -s -o /dev/null -w "Request $i: %{http_code}\n" \
    -H "Authorization: Bearer $TOKEN" \
    "http://localhost:8080/api/external/price-check?lotId=error/lot-001"
done

# 6回目以降はサーキットブレーカーが開き、WireMock にリクエストを送らずに即座にエラーを返す
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8080/api/external/price-check?lotId=error/lot-001"
# → 503 Service Unavailable（即座に返る）
# ログ: {"level":"Warning","message":"Circuit breaker OPEN for external-pricing-api"}
```

### 確認のコツ

- WireMock の管理API `http://localhost:8181/__admin/requests` で、WireMock が受け取ったリクエストの履歴を確認できる。サーキットブレーカーが開いた後はリクエスト数が増えないことを確認する
- テスト後は `curl -X POST http://localhost:8181/__admin/scenarios/reset` でシナリオをリセットする

---

## 実装ガイド

### F#

| 要素 | 実装方法 |
|---|---|
| HTTPクライアント | `IHttpClientFactory` + `System.Net.Http.HttpClient` |
| リトライ | Polly `WaitAndRetryAsync`（指数バックオフ） |
| サーキットブレーカー | Polly `CircuitBreakerAsync` |
| タイムアウト | `HttpClient.Timeout` |
| WireMock URL | `appsettings.json` の `ExternalApi.PricingUrl` で設定 |

### Kotlin

| 要素 | 実装方法 |
|---|---|
| HTTPクライアント | Ktor Client (`ktor-client-cio`) |
| リトライ | Ktor `HttpRequestRetry` plugin |
| サーキットブレーカー | Resilience4j `CircuitBreaker` |
| タイムアウト | Ktor `HttpTimeout` plugin |
| WireMock URL | `application.conf` の `externalApi.pricingUrl` で設定 |

---

## 次のステップ

Step 6が完了したら [Step 7: 非同期処理・イベント駆動](./step07.md) へ進む。
