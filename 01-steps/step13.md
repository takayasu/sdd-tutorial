# Step 13: マイグレーション追加（販売案件・査定・契約テーブル）

## 目的

### これは何か

Step 12 で追加した型定義に対応するデータベーステーブル（販売案件・価格査定・販売契約）を Alembic マイグレーションで追加する。

### なぜやるのか

- 新しいドメイン概念（販売案件等）のデータを保存するテーブルが必要
- Step 3 で導入した Alembic の仕組みを使い、既存テーブルに影響を与えずに追加する
- マイグレーションファイルとして残すことで、「いつ何のテーブルを追加したか」が履歴として残る

### 何がうれしいのか

- `alembic upgrade head` の一発コマンドで新しいテーブルが追加される
- 既存の `lot` テーブルのデータはそのまま残る（安全な追加）
- チームの他のメンバーも同じコマンドで同じ状態に追いつける

## 完了条件

```bash
# マイグレーション適用
$ cd backend && alembic upgrade head
INFO  [alembic.runtime.migration] Running upgrade 001 -> 002, create sales case tables

# テーブルが追加されたことを確認
$ docker compose exec db psql -U app -d sales_management -c "\dt"
              List of relations
 Schema |       Name        | Type  | Owner
--------+-------------------+-------+-------
 public | alembic_version   | table | app
 public | appraisal         | table | app
 public | contract          | table | app
 public | lot               | table | app
 public | sales_case        | table | app
 public | sales_case_lot    | table | app
(6 rows)
```

---

## マイグレーションファイル

### alembic/versions/002_create_sales_case_tables.py

```python
"""create sales case tables"""
from alembic import op
import sqlalchemy as sa

revision = "002"
down_revision = "001"
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.create_table(
        "sales_case",
        sa.Column("id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column("sales_case_number_year", sa.Integer, nullable=False),
        sa.Column("sales_case_number_month", sa.Integer, nullable=False),
        sa.Column("sales_case_number_seq", sa.Integer, nullable=False),
        sa.Column("division_code", sa.Integer, nullable=False),
        sa.Column("sales_date", sa.Date, nullable=False),
        sa.Column("status", sa.Text, nullable=False, server_default="before_appraisal"),
        sa.Column("shipping_instruction_date", sa.Date, nullable=True),
        sa.Column("shipping_completed_date", sa.Date, nullable=True),
        sa.UniqueConstraint(
            "sales_case_number_year",
            "sales_case_number_month",
            "sales_case_number_seq",
            name="uq_sales_case_number",
        ),
    )

    op.create_table(
        "sales_case_lot",
        sa.Column("sales_case_id", sa.Integer, sa.ForeignKey("sales_case.id"), nullable=False),
        sa.Column("lot_id", sa.Integer, sa.ForeignKey("lot.id"), nullable=False),
        sa.PrimaryKeyConstraint("sales_case_id", "lot_id"),
    )

    op.create_table(
        "appraisal",
        sa.Column("id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column("sales_case_id", sa.Integer, sa.ForeignKey("sales_case.id"), nullable=False),
        sa.Column("appraisal_number_year", sa.Integer, nullable=False),
        sa.Column("appraisal_number_month", sa.Integer, nullable=False),
        sa.Column("appraisal_number_seq", sa.Integer, nullable=False),
        sa.Column("appraisal_type", sa.Text, nullable=False, server_default="normal"),
        sa.Column("appraisal_date", sa.Date, nullable=False),
        sa.Column("delivery_date", sa.Date, nullable=False),
        sa.Column("sales_market", sa.Text, nullable=False),
        sa.Column("base_unit_price_date", sa.Text, nullable=False),
        sa.Column("period_adjustment_rate_date", sa.Text, nullable=False),
        sa.Column("counterparty_adjustment_rate_date", sa.Text, nullable=False),
        sa.Column("tax_excluded_estimated_total", sa.Integer, nullable=False),
        sa.Column("customer_contract_number", sa.Text, nullable=True),
        sa.Column("contract_adjustment_rate", sa.Numeric, nullable=True),
        sa.UniqueConstraint(
            "appraisal_number_year",
            "appraisal_number_month",
            "appraisal_number_seq",
            name="uq_appraisal_number",
        ),
    )

    op.create_table(
        "contract",
        sa.Column("id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column("appraisal_id", sa.Integer, sa.ForeignKey("appraisal.id"), nullable=False),
        sa.Column("contract_number_year", sa.Integer, nullable=False),
        sa.Column("contract_number_month", sa.Integer, nullable=False),
        sa.Column("contract_number_seq", sa.Integer, nullable=False),
        sa.Column("contract_date", sa.Date, nullable=False),
        sa.Column("person", sa.Text, nullable=False),
        sa.Column("customer_number", sa.Text, nullable=False),
        sa.Column("agent_name", sa.Text, nullable=True),
        sa.Column("sales_type", sa.Integer, nullable=False),
        sa.Column("item", sa.Text, nullable=False),
        sa.Column("delivery_method", sa.Text, nullable=False),
        sa.Column("sales_method", sa.Integer, nullable=False),
        sa.Column("tax_excluded_contract_amount_taxable", sa.Integer, nullable=False),
        sa.Column("consumption_tax", sa.Integer, nullable=False),
        sa.Column("tax_excluded_payment_amount", sa.Integer, nullable=False),
        sa.Column("payment_consumption_tax", sa.Integer, nullable=False),
        sa.UniqueConstraint(
            "contract_number_year",
            "contract_number_month",
            "contract_number_seq",
            name="uq_contract_number",
        ),
    )


def downgrade() -> None:
    op.drop_table("contract")
    op.drop_table("appraisal")
    op.drop_table("sales_case_lot")
    op.drop_table("sales_case")
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
```

---

## 次のステップ

Step 13が完了したら [Step 14: 直接販売案件の集約API](./step14.md) へ進む。
