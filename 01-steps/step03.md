# Step 3: マイグレーション導入（Alembic）

## 目的

### これは何か

データベースのテーブル構造（スキーマ）の変更履歴を Python ファイルで管理し、コマンド1つで適用できる仕組みを作る。ツールは **Alembic**（SQLAlchemy 公式のマイグレーションツール）を使う。

### なぜやるのか

- 「開発者Aがローカルで手動でテーブルを作った」「開発者BはそのDDLを知らない」という状況を防ぐ
- CIでも「マイグレーション適用 → テスト実行」を自動化するため
- Step 7以降のDB永続化ステップで、テーブルが存在することを前提にする

### 何がうれしいのか

- DBのスキーマ変更がコードと同じリポジトリで管理される
- `alembic upgrade head` の一発コマンドで最新スキーマが適用される
- `alembic downgrade -1` で1つ前に戻せる

## 完了条件

```bash
# マイグレーション適用
$ cd backend && alembic upgrade head
INFO  [alembic.runtime.migration] Context impl PostgreSQLImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> 001, create lot table

# テーブルが作成されていることを確認
$ docker compose exec db psql -U app -d sales_management -c "\dt"
          List of relations
 Schema |       Name      | Type  | Owner
--------+-----------------+-------+-------
 public | alembic_version | table | app
 public | lot             | table | app
(2 rows)
```

---

## Alembic セットアップ

### 1. 依存追加（pyproject.toml）

```toml
[project]
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.30",
    "pydantic>=2.8",
    "sqlalchemy[asyncio]>=2.0",
    "asyncpg>=0.29",
    "alembic>=1.13",
]
```

### 2. 初期化

```bash
cd backend
uv sync
alembic init alembic
```

### 3. alembic.ini の修正

```ini
[alembic]
script_location = alembic
# sqlalchemy.url は env.py の os.getenv で設定するため空欄
sqlalchemy.url =
```

### 4. alembic/env.py の修正

```python
from __future__ import annotations

import asyncio
import os
from logging.config import fileConfig

from alembic import context
from sqlalchemy import pool
from sqlalchemy.ext.asyncio import create_async_engine

config = context.config

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

from src.infra.models import Base  # noqa: E402

target_metadata = Base.metadata

DATABASE_URL = os.getenv(
    "DATABASE_URL",
    "postgresql+asyncpg://app:app@localhost:5432/sales_management",
)


def run_migrations_offline() -> None:
    context.configure(
        url=DATABASE_URL,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )
    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection) -> None:
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


async def run_async_migrations() -> None:
    connectable = create_async_engine(DATABASE_URL, poolclass=pool.NullPool)
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()


def run_migrations_online() -> None:
    asyncio.run(run_async_migrations())


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

### 5. SQLAlchemy モデルベース作成

```python
# src/infra/models.py
from __future__ import annotations

from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass
```

### 6. 最初のマイグレーション作成

```bash
alembic revision -m "create lot table"
```

生成されたファイルを編集する:

```python
# alembic/versions/001_create_lot_table.py
"""create lot table"""
from alembic import op
import sqlalchemy as sa

revision = "001"
down_revision = None
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.create_table(
        "lot",
        sa.Column("id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column("lot_number_year", sa.Integer, nullable=False),
        sa.Column("lot_number_location", sa.Text, nullable=False),
        sa.Column("lot_number_seq", sa.Integer, nullable=False),
        sa.Column("division_code", sa.Integer, nullable=False),
        sa.Column("department_code", sa.Integer, nullable=False),
        sa.Column("section_code", sa.Integer, nullable=False),
        sa.Column("process_category", sa.Integer, nullable=False),
        sa.Column("inspection_category", sa.Integer, nullable=False),
        sa.Column("manufacturing_category", sa.Integer, nullable=False),
        sa.Column("status", sa.Text, nullable=False),
        sa.Column("manufacturing_completed_date", sa.Date, nullable=True),
        sa.Column("shipping_deadline_date", sa.Date, nullable=True),
        sa.Column("shipped_date", sa.Date, nullable=True),
        sa.Column("version", sa.Integer, nullable=False, server_default="1"),
        sa.UniqueConstraint(
            "lot_number_year",
            "lot_number_location",
            "lot_number_seq",
            name="uq_lot_number",
        ),
    )


def downgrade() -> None:
    op.drop_table("lot")
```

### 7. 適用

```bash
alembic upgrade head
```

---

## マイグレーションの命名規約

```
alembic/versions/
├── 001_create_lot_table.py
├── 002_create_sales_case_table.py    # Step 13 で追加
├── 003_create_appraisal_table.py     # Step 13 で追加
├── 004_create_reservation_table.py   # Step 17 で追加
└── 005_create_consignment_table.py   # Step 17 で追加
```

---

## ci.sh への追加

```bash
echo "=== マイグレーション ==="
cd backend
alembic upgrade head
echo "マイグレーションOK"
cd ..
```

---

## 次のステップ

Step 3が完了したら [Step 4: フォーマッター導入](./step04.md) へ進む。
