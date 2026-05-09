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
```

## 完了条件

### HTTPクライアントの確認

1. アプリ経由で WireMock の正常スタブを呼び出し、レスポンスが返ること:

```bash
TOKEN=$(./scripts/get-token.sh test-operator)

curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/external/price-check?lot_id=2024-A-001"
# → 200 OK
# {"basePrice":10000,"adjustmentRate":1.05,"source":"external-pricing-api"}
```

### リトライの確認

2. WireMock の Scenario 機能を使い、「1回目は500、2回目は200」を再現する:

```bash
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

curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/external/price-check?lot_id=retry-test"
# → 200 OK（リトライにより成功）
```

3. ログにリトライの記録が出力されていること:

```
{"level":"warning","event":"retry attempt","attempt":1,"status_code":500,"service":"external-pricing-api"}
{"level":"info","event":"retry succeeded","attempt":2,"service":"external-pricing-api"}
```

### タイムアウトの確認

4. WireMock の遅延スタブを呼び出し、タイムアウトが発生すること:

```bash
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/external/price-check?lot_id=slow/lot-001"
# → 502 Bad Gateway（タイムアウト）
# {"type":"external-service-error","detail":"Request timed out after 3s"}
```

### サーキットブレーカーの確認

5. エラースタブに対して連続でリクエストを送り、サーキットブレーカーが開くこと:

```bash
for i in $(seq 1 5); do
  curl -s -o /dev/null -w "Request $i: %{http_code}\n" \
    -H "Authorization: Bearer $TOKEN" \
    "http://localhost:8000/external/price-check?lot_id=error/lot-001"
done

# 6回目以降は即座に503が返る
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/external/price-check?lot_id=error/lot-001"
# → 503 Service Unavailable（即座に返る）
```

### 確認のコツ

- WireMock の管理API `http://localhost:8181/__admin/requests` で受け取ったリクエストの履歴を確認できる
- テスト後は `curl -X POST http://localhost:8181/__admin/scenarios/reset` でシナリオをリセットする

---

## 実装ガイド

### Python / FastAPI (Backend)

| 要素 | 実装方法 |
|---|---|
| HTTPクライアント | `httpx.AsyncClient`（非同期） |
| リトライ | `tenacity` `AsyncRetrying`（指数バックオフ） |
| サーキットブレーカー | `circuitbreaker` ライブラリ |
| タイムアウト | `httpx.Timeout` |
| WireMock URL | `.env` の `PRICING_API_URL` で設定 |

#### 依存パッケージ追加

```bash
cd backend
uv add httpx tenacity circuitbreaker
```

#### `backend/src/clients/pricing_client.py`

```python
import os
import structlog
import httpx
from tenacity import AsyncRetrying, stop_after_attempt, wait_exponential, retry_if_exception
from circuitbreaker import circuit

logger = structlog.get_logger()

PRICING_API_URL = os.environ.get("PRICING_API_URL", "http://localhost:8181")
_TIMEOUT = httpx.Timeout(3.0)


def _is_server_error(exc: BaseException) -> bool:
    return isinstance(exc, httpx.HTTPStatusError) and exc.response.status_code >= 500


@circuit(failure_threshold=5, recovery_timeout=30, name="external-pricing-api")
async def _fetch_price_raw(lot_id: str) -> dict:
    async with httpx.AsyncClient(timeout=_TIMEOUT) as client:
        resp = await client.get(f"{PRICING_API_URL}/api/pricing/{lot_id}")
        resp.raise_for_status()
        return resp.json()


async def fetch_price(lot_id: str) -> dict:
    attempt = 0
    async for attempt_ctx in AsyncRetrying(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=0.5, min=0.5, max=4),
        retry=retry_if_exception(_is_server_error),
        reraise=True,
    ):
        with attempt_ctx:
            attempt += 1
            if attempt > 1:
                logger.warning(
                    "retry attempt",
                    attempt=attempt,
                    service="external-pricing-api",
                )
            result = await _fetch_price_raw(lot_id)
    return result
```

#### `backend/src/routers/external.py`

```python
from fastapi import APIRouter, Depends, HTTPException
from circuitbreaker import CircuitBreakerError

from src.clients.pricing_client import fetch_price
from src.middleware.auth import require_auth

router = APIRouter(prefix="/external")


@router.get("/price-check")
async def price_check(lot_id: str, claims: dict = Depends(require_auth)) -> dict:
    try:
        return await fetch_price(lot_id)
    except CircuitBreakerError:
        raise HTTPException(status_code=503, detail="External pricing service unavailable")
    except Exception:
        raise HTTPException(status_code=502, detail="External service error")
```

#### `.env` への追記

```ini
PRICING_API_URL=http://localhost:8181
```

---

## 次のステップ

Step 6が完了したら [Step 7: 非同期処理・イベント駆動](./step07.md) へ進む。
