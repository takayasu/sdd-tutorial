# Step 18b: 認証 ON 化 + 開発者向け補助エンドポイント

> Section 3 の Step 18 と Step 19 の間に位置する。Step 1-18 は `AUTH_ENABLED=false` 前提で進めてきたが、本ステップで「ON にしても回せる」状態にする。

## 目的

### これは何か

ここまで全ての API は `AUTH_ENABLED=false` で開放されていた。本ステップで以下を導入し、**`AUTH_ENABLED=true` に切り替えても tutorial の verify が緑のまま回る**状態にする:

| 要素 | 何のため |
|---|---|
| `.env` の `AUTH_ENABLED` 切り替え | dev/staging/prod で auth を on/off 切り替え可能にする |
| `scripts/mint-token.py` CLI | `JWT_SECRET_KEY` (HS256) で署名した開発用 JWT を発行 (Keycloak を立てずに認証 ON で疎通するため) |
| `GET /auth/config` パブリックエンドポイント | フロント (React) が起動時に「認証 ON か / IdP authority」を自動判定できるようにする |
| ci.sh verify セクションへの auth-on 検証追加 | 「auth ON にしてもエンドポイントは生きている」を継続検証 |

**やらないこと**: Keycloak / 本番 OIDC IdP の構築は本ステップ範囲外。あくまで HS256 共有秘密での dev/CI 用認証を扱う。

### なぜやるのか

- 認証 ON の動作確認手段がないと、本番デプロイ直前に「ローカルでは通っていたが本番では 401 連発」という事故を起こす
- フロント (React) は `VITE_AUTH_BYPASS=true` のような環境変数で切り替えるが、サーバ実態と乖離するリスクがある。`/auth/config` をサーバから配信すれば**唯一の事実源**にできる
- integration test でも JWT を都度自前構築すると DRY 違反 + 共通化漏れ。CLI に集約すると「test で使った CLI ロジックを dev でも使う」ようにできる
- Phase 2 (Step 21+) で OIDC IdP に置き換えた時も、`/auth/config` の interface だけ保てばフロントの変更は不要

### 何がうれしいのか

- `python scripts/mint-token.py --role operator --user u1` で JWT がもらえ、`Authorization: Bearer <token>` で API を叩ける
- フロントは `VITE_AUTH_BYPASS` のような二重管理を捨て、起動時に `/auth/config` を 1 回叩くだけで判定できる
- ci.sh が `AUTH_ENABLED=false`/`true` の両方で緑になり、「本番直前で 401」事故を構造的に防げる

## 完了条件

### (a) 動作要件

```bash
# 1. /auth/config が無認証で 200
$ curl -sf http://localhost:8000/auth/config | jq .
{
  "enabled": false,
  "audience": "sales-api"
}

# 2. AUTH_ENABLED=false の時、保護エンドポイントは無認証で 200
$ curl -sf http://localhost:8000/lots | jq -e '.items' >/dev/null

# 3. AUTH_ENABLED=true に切り替えると未認証は 401
$ AUTH_ENABLED=true uv run uvicorn src.main:app --host 0.0.0.0 --port 8000 &
$ curl -s -o /dev/null -w "%{http_code} %{content_type}\n" http://localhost:8000/lots
401 application/problem+json

# 4. /auth/config は ON でも無認証で叩ける
$ curl -sf http://localhost:8000/auth/config | jq -e '.enabled==true and .audience'

# 5. JWT を発行
$ TOKEN=$(python scripts/mint-token.py --role operator --user u1)
$ echo $TOKEN | head -c 4
eyJ

# 6. 発行した JWT で保護エンドポイントが 200
$ curl -sf -H "Authorization: Bearer $TOKEN" http://localhost:8000/lots | jq -e '.items' >/dev/null

# 7. ロール不足は 403
$ TOKEN_VIEWER=$(python scripts/mint-token.py --role viewer --user u2)
$ curl -s -o /dev/null -w "%{http_code}\n" \
    -X POST http://localhost:8000/lots \
    -H "Authorization: Bearer $TOKEN_VIEWER" \
    -H "Content-Type: application/json" \
    -d '{"lotNumber":{"year":2024,"location":"A","seq":99}}'
403

# 8. openapi.json に securitySchemes が記載されている
$ python3 -c "
import urllib.request, json
spec = json.loads(urllib.request.urlopen('http://localhost:8000/openapi.json').read())
assert 'bearerAuth' in spec.get('components', {}).get('securitySchemes', {})
print('OK')
"
OK
```

### (b) `ci.sh` verify セクションへの追記

verify セクションは **2 周回す**: 1 周目は `AUTH_ENABLED=false`、2 周目は `=true`。

```bash
echo "=== verify (auth=off) ==="
AUTH_ENABLED=false uv run uvicorn src.main:app --host 0.0.0.0 --port 8000 &
APP_PID=$!; sleep 2
run_smoke_curls
kill $APP_PID

echo "=== verify (auth=on) ==="
AUTH_ENABLED=true uv run uvicorn src.main:app --host 0.0.0.0 --port 8000 &
APP_PID=$!; sleep 2
TOKEN=$(python scripts/mint-token.py --role operator --user ci)
AUTH_HEADER="Authorization: Bearer $TOKEN" run_smoke_curls
curl -sf http://localhost:8000/auth/config | jq -e '.enabled==true' >/dev/null && echo "PASS auth-config-on"
curl -s -o /dev/null -w "%{http_code}" http://localhost:8000/lots | grep -q 401 && echo "PASS protected-401-without-token"
kill $APP_PID
```

---

## 実装方針

### 依存パッケージ追加

```bash
cd backend && uv add "python-jose[cryptography]"
```

### `scripts/mint-token.py` — 開発用 JWT 発行 CLI

```python
#!/usr/bin/env python3
"""開発用 JWT を発行する CLI。AUTH_ENABLED=true で API を叩くために使う。"""
import argparse
import os
import time
from jose import jwt

SECRET = os.environ.get("JWT_SECRET_KEY", "dev-secret-min-32-chars-xxxxxxxxxx")
ISSUER = os.environ.get("JWT_ISSUER", "dev-issuer")
AUDIENCE = os.environ.get("JWT_AUDIENCE", "sales-api")

parser = argparse.ArgumentParser()
parser.add_argument("--role", choices=["viewer", "operator", "admin"], required=True)
parser.add_argument("--user", required=True)
parser.add_argument("--ttl", type=int, default=3600)
args = parser.parse_args()

now = int(time.time())
claims = {
    "sub": args.user,
    "role": args.role,
    "iat": now,
    "exp": now + args.ttl,
    "iss": ISSUER,
    "aud": AUDIENCE,
}
print(jwt.encode(claims, SECRET, algorithm="HS256"))
```

### `GET /auth/config` パブリックエンドポイント

`backend/src/routers/auth_config.py`:

```python
import os
from fastapi import APIRouter
from pydantic import BaseModel

router = APIRouter()


class AuthConfigResponse(BaseModel):
    enabled: bool
    audience: str
    authority: str | None = None


@router.get("/auth/config", response_model=AuthConfigResponse, openapi_extra={"security": []})
async def get_auth_config() -> AuthConfigResponse:
    return AuthConfigResponse(
        enabled=os.environ.get("AUTH_ENABLED", "false").lower() == "true",
        audience=os.environ.get("JWT_AUDIENCE", "sales-api"),
        authority=os.environ.get("JWT_AUTHORITY") or None,
    )
```

### AUTH_ENABLED 切り替えミドルウェア

`backend/src/middleware/auth.py`:

```python
import os
from fastapi import Depends, HTTPException
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
from jose import JWTError, jwt

SECRET = os.environ.get("JWT_SECRET_KEY", "dev-secret-min-32-chars-xxxxxxxxxx")
AUDIENCE = os.environ.get("JWT_AUDIENCE", "sales-api")
AUTH_ENABLED = os.environ.get("AUTH_ENABLED", "false").lower() == "true"

_bearer = HTTPBearer(auto_error=False)


async def require_auth(
    credentials: HTTPAuthorizationCredentials | None = Depends(_bearer),
) -> dict:
    if not AUTH_ENABLED:
        return {"sub": "anonymous", "role": "operator"}
    if credentials is None:
        raise HTTPException(
            status_code=401,
            detail="Not authenticated",
            headers={"WWW-Authenticate": "Bearer"},
        )
    try:
        return jwt.decode(credentials.credentials, SECRET, algorithms=["HS256"], audience=AUDIENCE)
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")


def require_role(role: str):
    async def check(claims: dict = Depends(require_auth)) -> dict:
        if claims.get("role") not in (role, "admin"):
            raise HTTPException(status_code=403, detail="Insufficient role")
        return claims
    return check
```

`backend/src/main.py` に追記:

```python
from src.routers import auth_config
app.include_router(auth_config.router, tags=["auth"])

from fastapi.openapi.utils import get_openapi

def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema
    schema = get_openapi(title=app.title, version=app.version, routes=app.routes)
    schema.setdefault("components", {}).setdefault("securitySchemes", {})["bearerAuth"] = {
        "type": "http", "scheme": "bearer", "bearerFormat": "JWT",
    }
    app.openapi_schema = schema
    return schema

app.openapi = custom_openapi
```

### `.env` 設定値

```ini
# backend/.env（.gitignore に追加すること）
AUTH_ENABLED=false
JWT_SECRET_KEY=dev-secret-min-32-chars-xxxxxxxxxx
JWT_ISSUER=dev-issuer
JWT_AUDIENCE=sales-api
```

---

## 次のステップ

Step 18b が完了したら [Step 19: 品質ダッシュボード](./step19.md) へ進む。
