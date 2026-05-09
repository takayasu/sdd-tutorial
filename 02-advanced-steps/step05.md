# Step 5: 統合テスト + TestContainers

## 目的

### これは何か

TestContainersを使って、テスト実行時にPostgreSQLコンテナを自動起動し、APIの統合テスト（HTTPリクエスト → ドメインロジック → DB永続化 → HTTPレスポンス）をエンドツーエンドで検証する。

### なぜやるのか

- Step 8のPBTはドメインロジック（純粋関数）のテスト。DBやHTTPレイヤーは検証していない
- 統合テストは「APIを叩いて、DBに正しく保存され、正しいレスポンスが返る」ことを検証する
- TestContainersを使うと、テスト用のDBをDockerコンテナとして自動起動・自動破棄できる。`docker compose up` を手動で実行する必要がない

### 何がうれしいのか

- `uv run pytest` を実行するだけで、DBコンテナが自動起動し、マイグレーションが適用され、テストが実行され、コンテナが自動破棄される。手動のセットアップが一切不要
- CIでも同じコマンドで動く。「CIでDBが起動していない」という問題が起きない
- Step 7の完了条件で手動curlしていた内容がテストコードになる

## 完了条件

### テスト実行の確認

1. テストコマンドを実行すると、PostgreSQLコンテナが自動起動し、テストが通ること:

```bash
cd backend
uv run pytest tests/integration/ -q
# → passed N, failed 0
```

2. テスト完了後、PostgreSQLコンテナが自動的に停止・削除されること（`docker ps` で残っていないこと）

### テストシナリオの確認

最低限、以下のシナリオがテストされていること:

3. ロット作成 → GET で取得できること（正常系エンドツーエンド）
4. ロット作成 → 製造完了 → DBのstatusが `manufactured` に更新されていること
5. 存在しないロットの取得 → 404 Problem Details が返ること
6. 不正な状態遷移 → 400 Problem Details が返ること
7. 認証トークンなし → 401が返ること（Step 3の認証が統合テストでも動くこと）

### 確認のコツ

- テスト実行中に別ターミナルで `docker ps` を叩くと、TestContainersが起動したPostgreSQLコンテナが見える
- テストが失敗した場合、TestContainersのログ（コンテナの起動ログ）を確認する。ポートの競合やDockerデーモンの停止が原因のことが多い
- 各テストは独立して実行できること（テスト間でデータが干渉しない）。各テストでトランザクションをロールバックするか、テストごとにテーブルを TRUNCATE する

---

## 実装ガイド

### テストの構造

```
テスト起動
  → testcontainers-python: PostgreSQLコンテナ起動
  → alembic upgrade head（マイグレーション適用）
  → httpx.AsyncClient + FastAPI ASGIアプリ（インメモリ）
  → HTTPリクエスト送信 → レスポンス検証
  → テスト完了
  → コンテナ自動破棄
```

### 統合テストでの認証の扱い

Keycloak を TestContainers で起動する方法もあるが、起動に30秒以上かかり統合テストが遅くなる。代わりに、テスト内で HS256 JWTを生成し、アプリの `AUTH_ENABLED=true` + テスト用シークレットで動かす方式を推奨する:

1. テスト用の `JWT_SECRET_KEY` を環境変数で指定する（`scripts/mint-token.py` と同じ仕組み）
2. テスト内でロール付きのJWTを自己署名して `Authorization: Bearer` ヘッダに付与する
3. Keycloak なしで認証・認可テスト（401/403）が高速に実行できる

### Python / pytest (Backend)

| 要素 | 実装方法 |
|---|---|
| テスト用サーバー | `httpx.AsyncClient` + FastAPI `app`（ASGI） |
| TestContainers | `testcontainers-python` (`PostgresContainer`) |
| マイグレーション | テストセットアップ内で `alembic upgrade head` |
| JWT 生成 | `python-jose` で HS256 トークンを生成 |

#### 依存パッケージ追加

```bash
cd backend
uv add --dev testcontainers httpx pytest-asyncio
```

#### `backend/tests/integration/conftest.py`

```python
import asyncio
import os
import pytest
import pytest_asyncio
from httpx import AsyncClient, ASGITransport
from testcontainers.postgres import PostgresContainer
from alembic.config import Config
from alembic import command

from src.main import app


@pytest.fixture(scope="session")
def postgres_container():
    with PostgresContainer("postgres:16") as pg:
        os.environ["DATABASE_URL"] = pg.get_connection_url().replace(
            "postgresql+psycopg2", "postgresql+asyncpg"
        )
        alembic_cfg = Config("alembic.ini")
        command.upgrade(alembic_cfg, "head")
        yield pg


@pytest_asyncio.fixture
async def client(postgres_container):
    async with AsyncClient(
        transport=ASGITransport(app=app), base_url="http://test"
    ) as ac:
        yield ac


def mint_token(role: str = "operator", user: str = "test-user") -> str:
    import time
    from jose import jwt

    secret = os.environ.get("JWT_SECRET_KEY", "dev-secret-min-32-chars-xxxxxxxxxx")
    now = int(time.time())
    return jwt.encode(
        {
            "sub": user,
            "role": role,
            "aud": "sales-api",
            "iss": "dev-issuer",
            "iat": now,
            "exp": now + 3600,
        },
        secret,
        algorithm="HS256",
    )
```

#### `backend/tests/integration/test_lots.py`

```python
import pytest
from httpx import AsyncClient
from tests.integration.conftest import mint_token


@pytest.mark.asyncio
async def test_create_and_get_lot(client: AsyncClient):
    token = mint_token("operator")
    headers = {"Authorization": f"Bearer {token}"}

    create_res = await client.post(
        "/lots",
        json={"lotNumber": {"year": 2024, "location": "A", "seq": 1}},
        headers=headers,
    )
    assert create_res.status_code == 201

    lot_id = create_res.json()["lotNumber"]
    get_res = await client.get(f"/lots/{lot_id}", headers=headers)
    assert get_res.status_code == 200
    assert get_res.json()["status"] == "in-production"


@pytest.mark.asyncio
async def test_complete_manufacturing(client: AsyncClient):
    token = mint_token("operator")
    headers = {"Authorization": f"Bearer {token}"}

    create_res = await client.post(
        "/lots",
        json={"lotNumber": {"year": 2024, "location": "B", "seq": 1}},
        headers=headers,
    )
    lot_id = create_res.json()["lotNumber"]

    res = await client.post(
        f"/lots/{lot_id}/complete-manufacturing",
        json={"date": "2026-04-22", "version": 1},
        headers=headers,
    )
    assert res.status_code == 200
    assert res.json()["status"] == "manufactured"


@pytest.mark.asyncio
async def test_get_nonexistent_lot_returns_404(client: AsyncClient):
    token = mint_token("operator")
    res = await client.get(
        "/lots/9999-Z-999",
        headers={"Authorization": f"Bearer {token}"},
    )
    assert res.status_code == 404
    body = res.json()
    assert body["type"] == "not-found"
    assert res.headers["content-type"] == "application/problem+json"


@pytest.mark.asyncio
async def test_unauthenticated_returns_401(client: AsyncClient):
    res = await client.get("/lots/2024-A-001")
    assert res.status_code == 401


@pytest.mark.asyncio
async def test_viewer_cannot_transition_state(client: AsyncClient):
    operator_token = mint_token("operator")
    viewer_token = mint_token("viewer")

    create_res = await client.post(
        "/lots",
        json={"lotNumber": {"year": 2024, "location": "C", "seq": 1}},
        headers={"Authorization": f"Bearer {operator_token}"},
    )
    lot_id = create_res.json()["lotNumber"]

    res = await client.post(
        f"/lots/{lot_id}/complete-manufacturing",
        json={"date": "2026-04-22", "version": 1},
        headers={"Authorization": f"Bearer {viewer_token}"},
    )
    assert res.status_code == 403
```

#### `backend/pytest.ini` への追記

```ini
[pytest]
asyncio_mode = auto
```

#### CI での実行

```bash
cd backend
AUTH_ENABLED=true uv run pytest tests/integration/ -q
```

---

## 次のステップ

Step 5が完了したら [Step 6: HTTPクライアント + レジリエンス](./step06.md) へ進む。
