# Step 2: docker-compose構築（PostgreSQL）

## 目的

### これは何か

PostgreSQL データベースを Docker コンテナで起動する。開発環境とCI環境で同じDBを使えるようにする。

### なぜやるのか

- アプリからDBに接続する準備を整える（Step 3のマイグレーション、Step 7のDB永続化で必要）
- 「自分のPCに直接PostgreSQLをインストール」しなくてよいため、環境が汚れない
- docker-compose で構成を宣言的に管理し、`docker compose up -d` の一発起動を実現する

### 何がうれしいのか

- チーム全員が同じDB設定で開発できる
- CIサーバーでも同じコマンドでDBを起動できる
- コンテナを捨てて作り直すことで、クリーンな状態に戻せる

## 完了条件

```bash
# コンテナ起動
$ docker compose up -d

# DBに接続できる
$ docker compose exec db psql -U app -d sales_management -c "SELECT 1"
 ?column?
----------
        1
(1 row)

# バックエンドからの接続確認
$ cd backend && python -c "
import asyncio, asyncpg

async def main():
    conn = await asyncpg.connect('postgresql://app:app@localhost:5432/sales_management')
    result = await conn.fetchval('SELECT 1')
    print('DB接続OK:', result)
    await conn.close()

asyncio.run(main())
"
DB接続OK: 1
```

---

## docker-compose.yml

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: sales_management
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d sales_management"]
      interval: 5s
      timeout: 5s
      retries: 5

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin

volumes:
  pgdata:
  grafana_data:
```

---

## バックエンドへの DB 接続設定追加

### pyproject.toml に依存追加

```toml
[project]
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.30",
    "pydantic>=2.8",
    "sqlalchemy[asyncio]>=2.0",
    "asyncpg>=0.29",
]
```

### src/database.py

```python
from __future__ import annotations

import os

from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

DATABASE_URL = os.getenv(
    "DATABASE_URL",
    "postgresql+asyncpg://app:app@localhost:5432/sales_management",
)

engine = create_async_engine(DATABASE_URL, echo=False)
async_session_factory = async_sessionmaker(engine, expire_on_commit=False)


async def get_session() -> AsyncSession:
    async with async_session_factory() as session:
        yield session
```

---

## .env ファイル（ローカル開発用）

```bash
# .env（.gitignore に追加すること）
DATABASE_URL=postgresql+asyncpg://app:app@localhost:5432/sales_management
CORS_ORIGINS=http://localhost:5173
```

```bash
# .env.example（コミット用のサンプル）
DATABASE_URL=postgresql+asyncpg://USER:PASSWORD@localhost:5432/DBNAME
CORS_ORIGINS=http://localhost:5173
```

---

## ci.sh への追加

```bash
echo "=== DB起動確認 ==="
docker compose up -d db
until docker compose exec db pg_isready -U app -d sales_management >/dev/null 2>&1; do
  sleep 2
done
echo "DB起動OK"
```

---

## 次のステップ

Step 2が完了したら [Step 3: マイグレーション導入](./step03.md) へ進む。
