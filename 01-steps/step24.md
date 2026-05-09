# Step 24: APIコントラクトテスト（Pact）

## 目的

### これは何か

Pact は API の **コンシューマー（呼ぶ側）** と **プロバイダー（実装側）** の間の「契約」を JSON で記述し、双方が契約に従っていることを機械的に検証する仕組み。

- **Consumer（React フロントエンド）**: 「私はこのリクエストを送り、このレスポンスを期待する」を Pact ファイルとして書く
- **Provider（FastAPI バックエンド）**: Consumer が作成した Pact に対して実装が満たすかを検証する

Pact Broker は契約と検証結果を集中管理するサーバー。`can-i-deploy` というコマンドで「このバージョンを本番に出して大丈夫か」を機械判定できる。

### なぜやるのか

- React と FastAPI を別々に開発・デプロイするとき、フロントが期待する API シグネチャとバックエンドの実装が一致していることを継続的に保証する必要がある
- マルチエージェント開発では、バックエンドを変更したエージェントが、フロントを壊していても気づけない。コントラクトテストはそれを止める唯一の機械的手段
- `can-i-deploy` でデプロイ可否がスクリプト判定でき、エージェントの自律デプロイ判断に組み込める

### 何がうれしいのか

- React (Consumer) と FastAPI (Provider) が同一の Pact を満たすことを CI で検証できる
- バックエンドのレスポンス型を変えた瞬間に Consumer テストが落ちる
- `can-i-deploy` でデプロイ可否を自動判定できる

## 完了条件

```bash
# 1. Pact Broker 起動
$ docker compose -f docker-compose.harness.yml up -d pact-broker
$ curl -s http://localhost:9292/diagnostic/status/heartbeat
{"ok":true}

# 2. Consumer Pact 生成（React）
$ cd frontend && pnpm test:pact
PASS src/pact/lots.pact.test.ts
  Lot API
    ✓ GET /lots/{id} - ロット取得
    ✓ POST /lots/{id}/complete-manufacturing - 製造完了指示

# 3. FastAPI Provider 検証
$ cd backend && uv run pytest tests/pact/ -q
PASSED tests/pact/test_provider.py::test_provider_satisfies_consumer_pact
1 passed in 8.23s

# 4. can-i-deploy
$ docker run --rm pactfoundation/pact-cli:latest \
    pact-broker can-i-deploy \
    --broker-base-url http://host.docker.internal:9292 \
    --pacticipant sales-management-api \
    --version $(git rev-parse HEAD) \
    --to-environment production
Computer says yes
```

---

## 0. ハーネス共有 docker-compose の導入

Step 27 の Jaeger と共有するため、リポジトリルートに `docker-compose.harness.yml` を新設する。

`docker-compose.harness.yml`:

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

## 1. Consumer Pact（React / TypeScript）

### パッケージ追加

```bash
cd frontend
pnpm add -D @pact-foundation/pact
```

### Consumer テスト

`frontend/src/pact/lots.pact.test.ts`:

```typescript
import { PactV3, MatchersV3 } from '@pact-foundation/pact'
import path from 'path'
import { getLot, completeManufacturing } from '@/api/lots'

const { like, regex } = MatchersV3

const provider = new PactV3({
  consumer: 'frontend-app',
  provider: 'sales-management-api',
  dir: path.resolve(process.cwd(), '../pacts'),
  port: 8081,
})

describe('Lot API', () => {
  test('GET /lots/{id} - ロット取得', () => {
    return provider
      .addInteraction({
        states: [{ description: 'ロット 2024-A-001 が製造中で存在する' }],
        uponReceiving: 'ロット取得リクエスト',
        withRequest: { method: 'GET', path: '/lots/2024-A-001' },
        willRespondWith: {
          status: 200,
          headers: { 'Content-Type': 'application/json' },
          body: {
            lotNumber: regex('\\d{4}-[A-Z]-\\d{3}', '2024-A-001'),
            status: regex(
              'manufacturing|manufactured|shipping_instructed|shipped',
              'manufacturing'
            ),
          },
        },
      })
      .executeTest(async (mockServer) => {
        const lot = await getLot(mockServer.url, '2024-A-001')
        expect(lot.status).toBe('manufacturing')
      })
  })

  test('POST /lots/{id}/complete-manufacturing - 製造完了指示', () => {
    return provider
      .addInteraction({
        states: [{ description: 'ロット 2024-A-001 が製造中で存在する' }],
        uponReceiving: '製造完了指示リクエスト',
        withRequest: {
          method: 'POST',
          path: '/lots/2024-A-001/complete-manufacturing',
          body: { date: like('2024-04-01') },
        },
        willRespondWith: {
          status: 200,
          body: {
            status: 'manufactured',
            manufacturingCompletedDate: like('2024-04-01'),
          },
        },
      })
      .executeTest(async (mockServer) => {
        const result = await completeManufacturing(mockServer.url, '2024-A-001', '2024-04-01')
        expect(result.status).toBe('manufactured')
      })
  })
})
```

### `package.json` にスクリプト追加

```json
{
  "scripts": {
    "test:pact": "vitest run src/pact/"
  }
}
```

### Pact Broker に publish

```bash
docker run --rm -v $(pwd)/pacts:/pacts \
  pactfoundation/pact-cli:latest \
  pact-broker publish /pacts \
    --broker-base-url http://host.docker.internal:9292 \
    --broker-username admin --broker-password admin \
    --consumer-app-version $(git rev-parse HEAD) \
    --branch $(git branch --show-current)
```

---

## 2. Provider 検証（FastAPI / Python）

### パッケージ追加

```toml
# backend/pyproject.toml
[project.optional-dependencies]
dev = [
    # ... 既存 ...
    "pact-python>=2.2",
]
```

```bash
cd backend && uv pip install pact-python
```

### Provider 検証テスト

`backend/tests/pact/test_provider.py`:

```python
import subprocess
import time
import pytest
from pact import Verifier

PROVIDER_URL = "http://localhost:8001"
BROKER_URL = "http://localhost:9292"


@pytest.fixture(scope="module", autouse=True)
def start_provider():
    proc = subprocess.Popen(
        ["uv", "run", "uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8001"],
        env={**__import__("os").environ, "PACT_PROVIDER_STATES": "true"},
    )
    time.sleep(3)
    yield
    proc.terminate()


def test_provider_satisfies_consumer_pact():
    verifier = Verifier(
        provider="sales-management-api",
        provider_base_url=PROVIDER_URL,
    )
    output, _ = verifier.verify_with_broker(
        broker_url=BROKER_URL,
        broker_username="admin",
        broker_password="admin",
        provider_states_setup_url=f"{PROVIDER_URL}/provider-states",
        publish_verification_results=True,
        provider_version="1.0.0",
    )
    assert output == 0, "Provider verification failed"
```

### Provider State エンドポイント

`src/routers/provider_states.py`:

```python
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from src.database import get_session

router = APIRouter()

@router.post("/provider-states")
async def setup_provider_state(
    body: dict,
    session: AsyncSession = Depends(get_session),
):
    state = body.get("state", "")
    if state == "ロット 2024-A-001 が製造中で存在する":
        await session.execute(
            """
            INSERT INTO lot (lot_number_year, lot_number_location, lot_number_seq, status)
            VALUES (2024, 'A', 1, 'manufacturing')
            ON CONFLICT DO NOTHING
            """
        )
        await session.commit()
    return {"result": "success"}
```

`src/main.py` に環境変数ガードで追加：

```python
import os
if os.getenv("PACT_PROVIDER_STATES") == "true":
    from src.routers import provider_states
    app.include_router(provider_states.router)
```

---

## can-i-deploy デモ

```bash
docker run --rm pactfoundation/pact-cli:latest \
  pact-broker can-i-deploy \
  --broker-base-url http://host.docker.internal:9292 \
  --broker-username admin --broker-password admin \
  --pacticipant sales-management-api \
  --version $(git rev-parse HEAD) \
  --to-environment production

# 出力例:
# Computer says yes \o/
#
#  CONSUMER      | C.VERSION | PROVIDER              | P.VERSION | SUCCESS?
# ---------------|-----------|------------------------|-----------|----------
#  frontend-app  | abc123    | sales-management-api   | def456    | true
```

---

## ci.sh への追加

```bash
echo "=== Pact Broker 起動チェック ==="
curl -fs http://localhost:9292/diagnostic/status/heartbeat > /dev/null \
  || (echo "Pact Broker 未起動: docker compose -f docker-compose.harness.yml up -d" && exit 1)

echo "=== Consumer Pact 生成 & publish ==="
cd frontend && pnpm test:pact && cd ..
docker run --rm -v $(pwd)/pacts:/pacts \
  pactfoundation/pact-cli:latest \
  pact-broker publish /pacts \
    --broker-base-url http://host.docker.internal:9292 \
    --broker-username admin --broker-password admin \
    --consumer-app-version $(git rev-parse HEAD) \
    --branch $(git branch --show-current)

echo "=== Provider 検証 (FastAPI) ==="
cd backend
PACT_PROVIDER_STATES=true uv run pytest tests/pact/ -q
cd ..

echo "=== can-i-deploy ==="
docker run --rm pactfoundation/pact-cli:latest \
  pact-broker can-i-deploy \
    --broker-base-url http://host.docker.internal:9292 \
    --broker-username admin --broker-password admin \
    --pacticipant sales-management-api \
    --version $(git rev-parse HEAD) \
    --to-environment production
echo "PASS can-i-deploy"
```

---

## RALPHループにおける位置づけ

| 層 | 役割 |
|---|---|
| Protocols | Consumer と Provider の間の取り決めを機械化 |
| フロント/バック SSoT | React と FastAPI が同一仕様を満たすことの単一の真実源 |

**マルチエージェント開発における核心的な意義**: エージェント A がバックエンドを変更し、エージェント B がフロントエンドを別々に変更しても、両方が同じ Consumer Pact に対して検証されるため、「片方だけ辻褄合わせする」逃げ道が塞がれる。`can-i-deploy` はエージェントの自律デプロイ判断（Step 30 の RALPH ループ）における主要な品質ゲートになる。

---

## 次のステップ

Step 24が完了したら [Step 25: SBOM生成](./step25.md) へ進む。
