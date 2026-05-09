# Step 1: 設定管理 + 構造化ロギング

## 目的

### これは何か

アプリケーションの設定（DB接続先、ポート番号等）を環境ごとに切り替えられるようにし、ログ出力を構造化JSON形式に変更する。さらに、全てのHTTPリクエストに一意のリクエストIDを付与し、ログで追跡できるようにする。

### なぜやるのか

- Step 7までの実装では、DB接続文字列やポート番号がコード内にハードコードされている。本番・ステージング・開発で接続先を変えるには、設定を外部化する必要がある
- テキスト形式のログは人間には読みやすいが、CloudWatch Logs Insights等のツールで検索・集計するには構造化JSON形式が必要
- 障害調査時に「このリクエストに関連するログだけ」を抽出するには、リクエストIDでフィルタできる必要がある

### 何がうれしいのか

- 環境変数を変えるだけで、同じコードが開発・ステージング・本番で動く
- ログが `{"timestamp":"...","level":"info","request_id":"abc-123","event":"..."}` のようなJSON形式になり、ログ検索ツールで `request_id = "abc-123"` と検索するだけで関連ログが全て見つかる
- 「このエラーはどのリクエストで起きた？」「このユーザーの操作履歴は？」といった調査が格段に速くなる

## 完了条件

### 設定管理の確認

以下の観点で動作を確認する:

1. 設定ファイルにDB接続文字列やポート番号が定義されていること
   - Backend: `backend/.env` + `backend/src/config.py` (pydantic-settings)
   - Frontend: `frontend/.env` + `VITE_` プレフィックス付き環境変数
2. 環境変数で設定を上書きできること
   - 例: `DATABASE_URL=postgresql+asyncpg://other-host/db uv run uvicorn src.main:app` で接続先が変わる
   - 例: `PORT=9090 uv run uvicorn src.main:app --port $PORT` でポートが変わる
3. コード内にハードコードされた接続文字列やポート番号が残っていないこと

### 構造化ロギングの確認

1. アプリを起動し、任意のAPIエンドポイント（例: `GET /lots/{id}`）にリクエストを送る
2. コンソールに出力されるログがJSON形式であること
3. ログに以下のフィールドが含まれていること:
   - `timestamp` — いつ
   - `level` — ログレベル（info, warning, error 等）
   - `event` — 何が起きたか
   - `request_id` — どのリクエストか
4. 同じリクエストに対する複数のログ行が、同一の `request_id` を持つこと
5. 異なるリクエストには異なる `request_id` が付与されること

### 確認のコツ

```bash
# 2つのリクエストを連続で送り、ログを見比べる
curl http://localhost:8000/lots/2024-A-001
curl http://localhost:8000/lots/2024-A-002

# ログ出力例（各行がJSON）:
# {"timestamp":"2026-04-22T10:00:00Z","level":"info","request_id":"a1b2c3","event":"request completed","method":"GET","path":"/lots/2024-A-001"}
# {"timestamp":"2026-04-22T10:00:01Z","level":"info","request_id":"d4e5f6","event":"request completed","method":"GET","path":"/lots/2024-A-002"}
# → request_id が異なることを確認
```

---

## 実装ガイド

### Python / FastAPI (Backend)

| 要素 | ライブラリ/機能 |
|---|---|
| 設定ファイル | `.env` + `pydantic-settings` |
| 環境別設定 | `.env.local`, `.env.staging` 等 + `env_file` 指定 |
| 環境変数オーバーライド | OS 環境変数が `.env` を上書き（pydantic-settings 標準） |
| 構造化ログ | `structlog` + `JSONRenderer` |
| リクエストID | `Starlette BaseHTTPMiddleware` + `structlog.contextvars` |

#### 依存パッケージ追加

```bash
cd backend
uv add pydantic-settings structlog
```

#### `backend/src/config.py`

```python
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    database_url: str = "postgresql+asyncpg://app:app@localhost:5432/sales_management"
    port: int = 8000
    log_level: str = "info"
    auth_enabled: bool = False
    jwt_secret_key: str = "dev-secret-min-32-chars-xxxxxxxxxx"
    jwt_audience: str = "sales-api"

    model_config = SettingsConfigDict(env_file=".env", extra="ignore")


settings = Settings()
```

#### `backend/src/logging_config.py`

```python
import logging
import structlog


def configure_logging(log_level: str = "info") -> None:
    logging.basicConfig(level=log_level.upper(), format="%(message)s")
    structlog.configure(
        processors=[
            structlog.stdlib.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.contextvars.merge_contextvars,
            structlog.processors.JSONRenderer(),
        ],
        logger_factory=structlog.PrintLoggerFactory(),
        cache_logger_on_first_use=True,
    )
```

#### `backend/src/middleware/request_id.py`

```python
import uuid
import structlog
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response

logger = structlog.get_logger()


class RequestIdMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next) -> Response:
        request_id = request.headers.get("X-Request-Id", uuid.uuid4().hex)
        structlog.contextvars.clear_contextvars()
        structlog.contextvars.bind_contextvars(request_id=request_id)
        response = await call_next(request)
        response.headers["X-Request-Id"] = request_id
        logger.info("request completed", method=request.method, path=request.url.path)
        return response
```

#### `backend/src/main.py` への追記

```python
from src.config import settings
from src.logging_config import configure_logging
from src.middleware.request_id import RequestIdMiddleware

configure_logging(settings.log_level)

app = FastAPI(title="Sales Management API")
app.add_middleware(RequestIdMiddleware)
```

#### `backend/.env` (`.gitignore` に追加すること)

```ini
DATABASE_URL=postgresql+asyncpg://app:app@localhost:5432/sales_management
PORT=8000
LOG_LEVEL=info
AUTH_ENABLED=false
JWT_SECRET_KEY=dev-secret-min-32-chars-xxxxxxxxxx
JWT_AUDIENCE=sales-api
```

### TypeScript / React (Frontend)

| 要素 | ライブラリ/機能 |
|---|---|
| 設定ファイル | `frontend/.env` (Vite 標準) |
| 環境変数 | `VITE_` プレフィックス必須（ブラウザに公開される） |
| 型安全アクセス | `import.meta.env.VITE_API_BASE_URL` |

#### `frontend/.env`

```ini
VITE_API_BASE_URL=http://localhost:8000
VITE_APP_TITLE=Sales Management
```

#### `frontend/src/config.ts`

```typescript
export const config = {
  apiBaseUrl: import.meta.env.VITE_API_BASE_URL ?? "http://localhost:8000",
  appTitle: import.meta.env.VITE_APP_TITLE ?? "Sales Management",
} as const
```

---

## 次のステップ

Step 1が完了したら [Step 2: エラーハンドリング + バリデーション](./step02.md) へ進む。
