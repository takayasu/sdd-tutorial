# Step 17: マイグレーション追加（予約・委託テーブル）

## 目的

### これは何か

Step 16 で追加した型定義に対応するデータベーステーブル（予約査定・委託業者情報・委託販売結果）を Alembic マイグレーションで追加する。

### なぜやるのか

- 予約販売案件と委託販売案件のデータを保存するテーブルが必要
- Step 3, 13 と同じパターン（マイグレーションファイルを追加 → `alembic upgrade head`）で追加する

### 何がうれしいのか

- 既存テーブル（lot, sales_case, appraisal, contract）に影響を与えずに追加できることを再確認できる

## 完了条件

```bash
# マイグレーション適用
$ cd backend && alembic upgrade head
INFO  [alembic.runtime.migration] Running upgrade 003 -> 004, create reservation consignment tables

# テーブルが追加されたことを確認
$ docker compose exec db psql -U app -d sales_management -c "\dt"
              List of relations
 Schema |         Name              | Type  | Owner
--------+---------------------------+-------+-------
 ...
 public | consignment_info          | table | app
 public | consignment_result        | table | app
 public | reservation_price         | table | app
 ...
```

---

## マイグレーションファイル

### alembic/versions/004_create_reservation_consignment_tables.py

```python
"""create reservation consignment tables"""
from alembic import op
import sqlalchemy as sa

revision = "004"
down_revision = "003"
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.create_table(
        "reservation_price",
        sa.Column("id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column("sales_case_id", sa.Integer, sa.ForeignKey("sales_case.id"), nullable=False),
        sa.Column("appraisal_number_year", sa.Integer, nullable=False),
        sa.Column("appraisal_number_month", sa.Integer, nullable=False),
        sa.Column("appraisal_number_seq", sa.Integer, nullable=False),
        sa.Column("appraisal_date", sa.Date, nullable=False),
        sa.Column("estimated_lot_info", sa.Text, nullable=False),
        sa.Column("estimated_amount", sa.Integer, nullable=False),
        sa.Column("status", sa.Text, nullable=False, server_default="undetermined"),
        sa.Column("determined_date", sa.Date, nullable=True),
        sa.Column("determined_amount", sa.Integer, nullable=True),
        sa.UniqueConstraint(
            "appraisal_number_year",
            "appraisal_number_month",
            "appraisal_number_seq",
            name="uq_reservation_price_number",
        ),
    )

    op.create_table(
        "consignment_info",
        sa.Column("id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column(
            "sales_case_id",
            sa.Integer,
            sa.ForeignKey("sales_case.id"),
            nullable=False,
            unique=True,
        ),
        sa.Column("consignor_name", sa.Text, nullable=False),
        sa.Column("consignor_code", sa.Text, nullable=False),
        sa.Column("designated_date", sa.Date, nullable=False),
    )

    op.create_table(
        "consignment_result",
        sa.Column("id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column(
            "sales_case_id",
            sa.Integer,
            sa.ForeignKey("sales_case.id"),
            nullable=False,
            unique=True,
        ),
        sa.Column("result_date", sa.Date, nullable=False),
        sa.Column("result_amount", sa.Integer, nullable=False),
    )


def downgrade() -> None:
    op.drop_table("consignment_result")
    op.drop_table("consignment_info")
    op.drop_table("reservation_price")
```

---

## 実行

```bash
cd backend
alembic upgrade head
```

---

## 確認

```bash
docker compose exec db psql -U app -d sales_management -c "\dt"
# reservation_price, consignment_info, consignment_result が追加されていること
```

---

## 次のステップ

Step 17が完了したら [Step 18: 予約・委託・品目変換のAPI実装](./step18.md) へ進む。
