# Step 8: 監査ログ + 分散トレーシング

## 目的

### これは何か

全てのデータ変更操作に「いつ・誰が・何をしたか」を自動記録する監査ログと、リクエストがシステム内をどう流れたかを可視化する分散トレーシングを導入する。トレースの可視化には Jaeger を docker-compose で起動する。

### なぜやるのか

- 業務システムでは「このデータを誰が変更したか」を追跡できることが必須。内部統制や監査対応で求められる
- 分散トレーシングは、1つのリクエストが「API → ドメインロジック → DB → 外部API」とどう流れたかを可視化する。障害調査で「どこで遅くなったか」を特定するのに不可欠

### 何がうれしいのか

- 「このロットの製造完了を指示したのは誰？いつ？」という問い合わせに、DBを1クエリ叩くだけで答えられる
- ブラウザで Jaeger を開くと、リクエストの流れが視覚的に確認できる。「このAPIが遅い原因はDBクエリ」「外部APIの呼び出しで3秒かかっている」が一目でわかる

## 事前準備: Jaeger を docker-compose に追加

### docker-compose.yml に追加

```yaml
  jaeger:
    image: jaegertracing/all-in-one:1.56
    environment:
      COLLECTOR_OTLP_ENABLED: "true"
    ports:
      - "16686:16686"   # Jaeger UI
      - "4317:4317"     # OTLP gRPC
      - "4318:4318"     # OTLP HTTP
```

```bash
docker compose up -d jaeger

# ブラウザで http://localhost:16686 を開く
```

## パート 1: 監査ログ

### やること

全てのテーブルに `created_at`, `created_by`, `updated_at`, `updated_by` カラムを追加し、データ変更時に自動的に記録する。`created_by` / `updated_by` にはJWTトークンから取得したユーザーIDが入る。

### マイグレーション

```sql
-- migrations/V005__add_audit_columns.sql

ALTER TABLE lot ADD COLUMN created_at TIMESTAMPTZ NOT NULL DEFAULT NOW();
ALTER TABLE lot ADD COLUMN created_by TEXT NOT NULL DEFAULT 'system';
ALTER TABLE lot ADD COLUMN updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW();
ALTER TABLE lot ADD COLUMN updated_by TEXT NOT NULL DEFAULT 'system';

-- lot_detail, sales_case 等の他のテーブルにも同様に追加
```

### 「削除」はドメインの状態遷移で表現する

論理削除（`deleted_at` フラグ）は使わない。「削除」が必要なエンティティは、ドメインモデルに「取消済み」等の状態を追加して型で表現する。

```
-- DSL上の「削除」は状態遷移
behavior 販売案件を削除する = 査定前直接販売案件 -> 削除済み OR 削除エラー

-- DB上は status カラムの値で表現
UPDATE sales_case SET status = 'cancelled', updated_by = :user_id WHERE ...
```

### ユーザーIDの伝播

```
HTTPリクエスト
  → JWT認証ミドルウェア（Step 3）
    → JWTの "sub" クレームからユーザーIDを取得（claims["sub"]）
      → ドメインロジック実行
        → DB保存時に created_by / updated_by にユーザーIDを設定
```

### Python / FastAPI での実装

```python
# APIハンドラでユーザーIDを渡す
@router.post("/lots")
async def create_lot(
    body: CreateLotInput,
    claims: dict = Depends(require_auth),
    session: AsyncSession = Depends(get_session),
):
    user_id = claims["sub"]
    return await do_create_lot(body, user_id, session)
```

```python
# DB保存時に監査カラムを設定
from datetime import datetime, timezone

async def do_create_lot(body: CreateLotInput, user_id: str, session: AsyncSession) -> dict:
    now = datetime.now(timezone.utc)
    await session.execute(
        text(
            "INSERT INTO lot (lot_number, status, created_at, created_by, updated_at, updated_by) "
            "VALUES (:lot_number, 'in-production', :now, :user_id, :now, :user_id)"
        ),
        {"lot_number": body.lot_number, "now": now, "user_id": user_id},
    )
    await session.commit()
```

### 完了条件

1. ロットを作成すると、`created_at`, `created_by` が自動的に記録されること:

```bash
TOKEN=$(./scripts/get-token.sh test-operator)
curl -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:8000/lots \
  -d '{"lotNumber":{"year":2024,"location":"A","seq":99}}'

# DB確認
docker compose exec db psql -U app -d sales_management \
  -c "SELECT created_at, created_by, updated_at, updated_by FROM lot WHERE lot_number_seq = 99;"
# → created_by = "a1b2c3d4-..."（JWTの sub クレーム）
```

2. ロットの状態を変更すると、`updated_at`, `updated_by` が更新されること（`created_at`, `created_by` は変わらない）:

```bash
ADMIN_TOKEN=$(./scripts/get-token.sh test-admin)
curl -X POST -H "Authorization: Bearer $ADMIN_TOKEN" \
  http://localhost:8000/lots/2024-A-099/complete-manufacturing \
  -d '{"date":"2026-04-22"}'

# DB確認
docker compose exec db psql -U app -d sales_management \
  -c "SELECT created_by, updated_by FROM lot WHERE lot_number_seq = 99;"
# → created_by = "operator-uuid"（変わらない）
# → updated_by = "admin-uuid"（変わった）
```

---

## パート 2: 分散トレーシング

### やること

OpenTelemetry を導入し、HTTPリクエスト・DBクエリ・外部API呼び出しのトレースを Jaeger に送信する。

### Python / FastAPI での設定

```bash
cd backend
uv add opentelemetry-sdk \
       opentelemetry-instrumentation-fastapi \
       opentelemetry-instrumentation-sqlalchemy \
       opentelemetry-instrumentation-httpx \
       opentelemetry-exporter-otlp-proto-grpc
```

```python
# backend/src/tracing.py
import os
from opentelemetry import trace
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor


def configure_tracing(app) -> None:
    resource = Resource.create({"service.name": "sales-management"})
    provider = TracerProvider(resource=resource)
    exporter = OTLPSpanExporter(
        endpoint=os.environ.get("OTLP_ENDPOINT", "http://localhost:4317"),
        insecure=True,
    )
    provider.add_span_processor(BatchSpanProcessor(exporter))
    trace.set_tracer_provider(provider)

    FastAPIInstrumentor.instrument_app(app)
    SQLAlchemyInstrumentor().instrument()
    HTTPXClientInstrumentor().instrument()
```

```python
# src/main.py への追記
from src.tracing import configure_tracing
configure_tracing(app)
```

```ini
# .env への追記
OTLP_ENDPOINT=http://localhost:4317
```

### structlog との連携

```python
# backend/src/middleware/request_id.py に追記
from opentelemetry import trace

class RequestIdMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        span = trace.get_current_span()
        ctx = span.get_span_context()
        structlog.contextvars.bind_contextvars(
            trace_id=format(ctx.trace_id, "032x") if ctx.is_valid else None
        )
        ...
```

### 完了条件

3. アプリを起動し、APIリクエストを送った後、Jaeger UI でトレースが表示されること:

```bash
curl -H "Authorization: Bearer $TOKEN" http://localhost:8000/lots/2024-A-001

# ブラウザで http://localhost:16686 を開く
# 「Service」ドロップダウンで「sales-management」を選択
# 「Find Traces」をクリック → トレースが表示される
```

4. トレースをクリックすると、以下のスパン（処理の区間）が表示されること:

```
sales-management: GET /lots/2024-A-001  [15ms]
  ├── postgresql: SELECT * FROM lot WHERE ...  [3ms]
  └── postgresql: SELECT * FROM lot_detail WHERE ...  [2ms]
```

5. 外部API呼び出し（Step 6）を含むリクエストでは、外部APIのスパンも表示されること:

```
sales-management: GET /external/price-check  [120ms]
  ├── postgresql: SELECT * FROM lot WHERE ...  [3ms]
  └── HTTP GET http://localhost:8181/api/pricing/...  [100ms]
```

6. structlog のログにも `trace_id` が含まれていること:

```
{"level":"info","trace_id":"abc123def456...","request_id":"xyz","event":"request completed"}
```

### 確認のコツ

- Jaeger UI の「Service」に `sales-management` が表示されない場合、`OTLP_ENDPOINT` の接続先が正しいか確認する
- トレースが表示されるまで数秒かかることがある。リクエスト送信後、少し待ってから「Find Traces」をクリックする
- Jaeger UI でトレースの各スパンをクリックすると、詳細情報（HTTPステータスコード、DBクエリ文等）が表示される

---

## 次のステップ

Step 8が完了したら [Step 9: レート制限 + キャッシュ + グレースフルシャットダウン](./step09.md) へ進む。
