# Step 1: Hello World API（+ 横断ミドルウェア + verify 機構）

## 目的

### これは何か

FastAPI を使って最小限のWeb APIプロジェクトを作成し、React のフロントエンドスケルトンを用意する。

ただし「動くだけ」では終わらせず、**以降の全ステップが乗る土台**として 3 つを最初から組み込む:

1. **横断ミドルウェア** — CORS / `X-Content-Type-Options: nosniff` / `Cross-Origin-Resource-Policy: same-origin` / problem+json default
2. **`ci.sh` の verify セクション** — `./ci.sh` の最後で API を起動し curl 検証するブロック。各 step の完了条件はここに **永続的に curl 検証を追加する**ことを義務化する
3. **OpenAPI スケルトン** — FastAPI が `/openapi.json` を自動生成する。エラースキーマを最初から定義しておく

### なぜやるのか

- いきなり複雑なドメインモデルを実装する前に、「プロジェクトを作ってビルドして動かす」という基本サイクルを体験する
- **横断ミドルウェアを後付けするとテストが大量に書き直しになる**。最初から入れる方がコストが低い
- **ralph で `[x]` をつけた後にコードが空のまま（false-positive completion）になるバグ**を構造的に防ぐため、完了条件を ci.sh に組み込む規約をここで導入する

### 何がうれしいのか

- 「自分の手でAPIサーバーを起動して、curlで叩いて応答が返ってくる」という成功体験が得られる
- Step 20 (DAST) で発覚しがちなセキュリティヘッダ警告を**最初から踏まない**
- FastAPI の自動生成 OpenAPI を使うことで、フロントエンドは `openapi-typescript` で型を自動取得できる

## 完了条件

```bash
# バックエンド起動（別ターミナルで）
$ cd backend && uvicorn src.main:app --reload

# 1. /health が 200
$ curl -sf http://localhost:8000/health
{"status":"ok"}

# 2. セキュリティヘッダ確認
$ curl -sI http://localhost:8000/health | grep -iE 'x-content-type-options|cross-origin-resource-policy'
x-content-type-options: nosniff
cross-origin-resource-policy: same-origin

# 3. CORS preflight
$ curl -sI -X OPTIONS http://localhost:8000/health \
    -H "Origin: http://localhost:5173" \
    -H "Access-Control-Request-Method: GET" | head -1
HTTP/1.1 200 OK

# 4. 存在しないルートが problem+json で 404
$ curl -s -o /dev/null -w "%{http_code} %{content_type}\n" http://localhost:8000/no-such-route
404 application/problem+json

# 5. OpenAPI JSON が返る
$ curl -sf http://localhost:8000/openapi.json | python -c "import sys,json; d=json.load(sys.stdin); print(d['info']['title'])"
Sales Management API

# 6. ./ci.sh の verify セクションが exit 0
$ ./ci.sh
=== verify (smoke) ===
PASS /health
PASS security-headers
PASS cors-preflight
PASS problem+json-on-404
PASS openapi-json
=== CI完了 ===
$ echo $?
0
```

---

## 横断ミドルウェアの方針（全ステップ共通の前提）

以降の全ステップは、ここで導入する 4 つのミドルウェアが入っている前提で書かれる。

| ミドルウェア | 何のため | 失敗時の症状 |
|---|---|---|
| **CORS** | フロントを別ドメインに置くため | 本番で frontend → backend の fetch がブラウザにブロックされる |
| **`X-Content-Type-Options: nosniff`** | MIME sniffing 攻撃の抑止 | ZAP `[10021]` 警告 |
| **`Cross-Origin-Resource-Policy: same-origin`** | クロスオリジン読み込み制限 | ZAP `[90004]` 警告 |
| **problem+json デフォルト** | エラー形式統一 (RFC 9457) | Step 4 以降で「エラー形式が混在」と言われ書き直しになる |

許可オリジンは環境変数 `CORS_ORIGINS` で設定可能にする（デフォルト: `http://localhost:5173`）。

---

## バックエンド（FastAPI）実装

### 1. プロジェクト作成

```bash
mkdir -p backend/src backend/tests
cd backend
```

### 2. pyproject.toml

```toml
[project]
name = "sales-management"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.30",
    "pydantic>=2.8",
]

[project.optional-dependencies]
dev = [
    "pytest>=8",
    "pytest-asyncio>=0.23",
    "httpx>=0.27",
    "ruff>=0.6",
    "hypothesis>=6",
]

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

### 3. src/\_\_init\_\_.py（空ファイル）

```bash
touch backend/src/__init__.py
```

### 4. src/main.py

```python
from __future__ import annotations

import os
from typing import Any

from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
from starlette.middleware.base import BaseHTTPMiddleware

app = FastAPI(title="Sales Management API", version="0.1.0")

# CORS
_allowed_origins = os.getenv("CORS_ORIGINS", "http://localhost:5173").split(",")
app.add_middleware(
    CORSMiddleware,
    allow_origins=_allowed_origins,
    allow_methods=["*"],
    allow_headers=["*"],
)

# セキュリティヘッダ
class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next: Any) -> Any:
        response = await call_next(request)
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["Cross-Origin-Resource-Policy"] = "same-origin"
        return response

app.add_middleware(SecurityHeadersMiddleware)

# 404 を problem+json で返す
@app.exception_handler(404)
async def not_found_handler(request: Request, exc: Any) -> JSONResponse:
    return JSONResponse(
        status_code=404,
        content={"type": "about:blank", "title": "Not Found", "status": 404},
        media_type="application/problem+json",
    )

# ヘルスチェック
@app.get("/health")
async def health() -> dict[str, str]:
    return {"status": "ok"}
```

### 5. 依存インストール・起動

```bash
cd backend
uv sync
uvicorn src.main:app --reload
# → http://localhost:8000 で起動
# → http://localhost:8000/docs で Swagger UI
# → http://localhost:8000/openapi.json で OpenAPI JSON
```

---

## フロントエンド（React + TypeScript + Vite）スケルトン

### 1. プロジェクト作成

```bash
pnpm create vite frontend --template react-ts
cd frontend
pnpm install
pnpm add zod
pnpm add -D prettier eslint @eslint/js typescript-eslint \
  vitest @vitest/coverage-v8 @testing-library/react @testing-library/jest-dom \
  fast-check openapi-typescript
```

### 2. vite.config.ts

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        rewrite: (path) => path.replace(/^\/api/, ''),
      },
    },
  },
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./tests/setup.ts'],
    coverage: { provider: 'v8' },
  },
})
```

### 3. 動作確認

```bash
cd frontend
pnpm dev
# → http://localhost:5173 で起動
```

---

## `ci.sh` の verify セクション規約（全ステップ共通の前提）

`./ci.sh` の末尾に **verify セクション** を設け、API を起動して curl で動作確認する。各 step の完了条件には以下を**必ず含める**:

> このステップで追加した完了条件（curl 例）を `ci.sh` の verify セクションに `curl -sf ...` として追加し、`./ci.sh` が exit 0 で終わること。

`ci.sh` の verify セクション雛形（Step 1 時点）:

```bash
#!/bin/bash
set -e

echo "=== verify (smoke) ==="
cd backend
uvicorn src.main:app --port 8000 &
APP_PID=$!
trap 'kill $APP_PID 2>/dev/null || true' EXIT

for i in $(seq 1 30); do
  curl -sf http://localhost:8000/health >/dev/null 2>&1 && break
  sleep 1
done

curl -sf http://localhost:8000/health >/dev/null \
  && echo "PASS /health"

curl -sI http://localhost:8000/health \
  | grep -iq '^x-content-type-options: nosniff' \
  && echo "PASS security-headers"

curl -sI -X OPTIONS http://localhost:8000/health \
  -H "Origin: http://localhost:5173" \
  -H "Access-Control-Request-Method: GET" \
  | head -1 | grep -q '200' \
  && echo "PASS cors-preflight"

curl -s -o /dev/null -w "%{content_type}" http://localhost:8000/no-such-route \
  | grep -q 'application/problem+json' \
  && echo "PASS problem+json-on-404"

curl -sf http://localhost:8000/openapi.json >/dev/null \
  && echo "PASS openapi-json"

echo "=== CI完了 ==="
```

---

## 次のステップ

Step 1が完了したら [Step 2: docker-compose構築](./step02.md) へ進む。
