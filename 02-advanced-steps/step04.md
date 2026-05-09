# Step 4: ヘルスチェック + OpenAPI

## 目的

### これは何か

アプリケーションとその依存サービス（DB等）の死活監視エンドポイントを実装し、APIの仕様書（OpenAPI / Swagger UI）を自動生成する。

### なぜやるのか

- k8s や ECS のヘルスチェック（liveness / readiness probe）は、アプリが「生きているか」「リクエストを受け付けられるか」を判定するためにHTTPエンドポイントを叩く。これがないとコンテナオーケストレータがアプリの異常を検知できない
- OpenAPI仕様書があると、フロントエンド開発者やAPI利用者が「どんなエンドポイントがあるか」「リクエスト/レスポンスの形式は何か」をブラウザで確認できる

### 何がうれしいのか

- ブラウザで `/health` を開くと、アプリとDBの状態が一目でわかる。DBが落ちていれば `"status": "DOWN"` と表示される
- FastAPI は `/docs`（Swagger UI）と `/redoc` を標準で提供する。Pydantic モデルから自動生成されるため、仕様書とコードが常に一致する
- k8sのreadinessProbeに `/health` を設定すれば、DB接続が切れたときに自動的にトラフィックが止まる

## 完了条件

### ヘルスチェックの確認

1. アプリとDBが正常な状態で `/health` を叩くと、全てUPであること:

```
GET /health
→ 200 OK

{
  "status": "UP",
  "checks": {
    "postgresql": "UP",
    "self": "UP"
  }
}
```

2. DBを停止した状態で `/health` を叩くと、ステータスがDOWNになること:

```bash
# DBを停止
docker compose stop db

# ヘルスチェック
GET /health
→ 503 Service Unavailable

{
  "status": "DOWN",
  "checks": {
    "postgresql": "DOWN",
    "self": "UP"
  }
}
```

3. DBを再起動すると、ヘルスチェックがUPに戻ること

```bash
docker compose start db
# 数秒待ってから
GET /health
→ 200 OK
```

### OpenAPIの確認

4. ブラウザで Swagger UI にアクセスできること:
   - `http://localhost:8000/docs`

5. Swagger UI に以下が表示されていること:
   - 全てのAPIエンドポイント（GET /lots, POST /lots, POST /lots/{id}/complete-manufacturing 等）
   - リクエストボディのスキーマ（どんなJSONを送ればいいか）
   - レスポンスのスキーマ（どんなJSONが返ってくるか）
   - 認証が必要なエンドポイントには鍵マークが表示される（`bearerAuth` スキーム）

6. Swagger UI の「Try it out」ボタンでAPIを実際に叩けること

### 確認のコツ

- ヘルスチェックは認証不要にすること（k8sのprobeは認証トークンを持たない）
- `/health` のレスポンスタイムが遅い場合、DBへの `SELECT 1` が遅い可能性がある。コネクションプールの設定を確認する
- OpenAPIのJSON仕様は `http://localhost:8000/openapi.json` で取得できる。フロントエンドのコード生成ツール（`openapi-typescript` 等）に渡すこともできる

---

## 実装ガイド

### Python / FastAPI (Backend)

| 要素 | 実装方法 |
|---|---|
| ヘルスチェック | 自前エンドポイント（SQLAlchemy `SELECT 1`） |
| Swagger UI | FastAPI 標準（`/docs`） |
| ReDoc | FastAPI 標準（`/redoc`） |
| OpenAPI JSON | FastAPI 標準（`/openapi.json`） |
| 認証スキーム | `bearerAuth` を `custom_openapi` で追加済み（Step 18b 参照） |

FastAPI は Pydantic モデルから OpenAPI スキーマを自動生成するため、別途アノテーションは不要。

#### `backend/src/routers/health.py`

```python
from fastapi import APIRouter, Depends
from fastapi.responses import JSONResponse
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession

from src.database import get_session

router = APIRouter()


@router.get("/health", include_in_schema=False)
async def health(session: AsyncSession = Depends(get_session)) -> JSONResponse:
    checks: dict[str, str] = {"self": "UP"}
    try:
        await session.execute(text("SELECT 1"))
        checks["postgresql"] = "UP"
    except Exception:
        checks["postgresql"] = "DOWN"

    overall = "UP" if all(v == "UP" for v in checks.values()) else "DOWN"
    status_code = 200 if overall == "UP" else 503
    return JSONResponse(
        status_code=status_code,
        content={"status": overall, "checks": checks},
    )
```

#### `backend/src/main.py` への追記

```python
from src.routers import health

app.include_router(health.router)
```

ヘルスチェックは `include_in_schema=False` で Swagger UI から非表示にし、認証ミドルウェアの依存なしで登録する。

#### Swagger UI の認証スキーム（Step 18b の再掲）

```python
# backend/src/main.py
from fastapi.openapi.utils import get_openapi

def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema
    schema = get_openapi(title=app.title, version=app.version, routes=app.routes)
    schema.setdefault("components", {}).setdefault("securitySchemes", {})["bearerAuth"] = {
        "type": "http",
        "scheme": "bearer",
        "bearerFormat": "JWT",
    }
    app.openapi_schema = schema
    return schema

app.openapi = custom_openapi
```

#### 動作確認

```bash
# Swagger UI
open http://localhost:8000/docs

# OpenAPI JSON（コード生成ツールに渡す）
curl http://localhost:8000/openapi.json | jq .info

# ヘルスチェック
curl http://localhost:8000/health
```

### TypeScript / React (Frontend) — openapi-typescript によるコード生成

OpenAPI JSON からフロントエンドの型定義を自動生成できる。

```bash
cd frontend
pnpm add -D openapi-typescript
pnpm openapi-typescript http://localhost:8000/openapi.json -o src/types/api.d.ts
```

生成された型を使ってリクエストを型安全に組み立てる:

```typescript
import type { paths } from "./types/api"

type GetLotResponse =
  paths["/lots/{lot_id}"]["get"]["responses"]["200"]["content"]["application/json"]
```

---

## 次のステップ

Step 4が完了したら [Step 5: 統合テスト + TestContainers](./step05.md) へ進む。
