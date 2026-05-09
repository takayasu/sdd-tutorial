# Step 15: 価格査定・販売契約のAPI実装

## 目的

### これは何か

価格査定（通常査定/顧客契約査定）と販売契約の詳細な CRUD API を実装する。Step 14 ではライフサイクル遷移を実装したが、ここでは査定・契約の「中身」（ロット明細単位の査定情報等）に集中する。

### なぜやるのか

- 実際の業務では、価格査定はロットごと・明細ごとに単価や調整率が設定される。この複雑な構造を DB に正しく保存・取得できることを確認する
- 通常査定と顧客契約査定で必要なデータが異なる（顧客契約査定には顧客契約番号と契約調整率が追加）。この違いを型で表現し、混同を防ぐ

### 何がうれしいのか

- 「型で構造を定義 → AI が CRUD を生成 → PBT で検証 → CI で品質担保」というサイクルが、複雑なデータ構造に対しても機能することを確認できる

本ステップは Step 14 で確立した「集約API完全パッケージ」の規約（詳細GET / 一覧GET / version 楽観ロック / problem+json / URL 集約）をそのまま継承する。

## 完了条件

```bash
# 1. 価格査定を作成（type=normal|agreement）
$ curl -sf -X POST http://localhost:8000/sales-cases/1/direct/appraisals \
  -H "Content-Type: application/json" \
  -d '{"type":"normal","appraisal_date":"2024-04-05","delivery_date":"2024-05-01",
       "sales_market":"国内","tax_excluded_estimated_total":500000,
       "lot_appraisals":[{"lot_id":1}],"version":1}'
{"id":1,"type":"normal","version":2,...}

# 2. 販売契約を締結（査定済み案件に対して）
$ curl -sf -X POST http://localhost:8000/sales-cases/1/direct/contracts \
  -H "Content-Type: application/json" \
  -d '{"contract_date":"2024-04-10","person":"担当太郎","buyer":{"customer_number":"C001"},"version":2}'
{"id":1,"status":"contracted","version":3,...}

# 3. 楽観ロック競合 → 409
$ curl -s -o /dev/null -w "%{http_code}" \
  -X PUT http://localhost:8000/sales-cases/1/direct/appraisals \
  -H "Content-Type: application/json" -d '{"appraisal_date":"2024-04-06","version":1}'
409

# 4. 詳細 GET に査定・契約情報が含まれる
$ curl -sf http://localhost:8000/sales-cases/1 \
  | python -c "import sys,json; d=json.load(sys.stdin); assert d['appraisal'] and d['contract']"
```

---

## 価格査定の構造

```
価格査定
├── 通常査定 or 顧客契約査定
├── 査定共通情報（番号、日付、市場、各種適用日、税抜予定総額）
└── List<ロット価格査定>
    ├── ロット ID
    ├── 各種割増・調整率（オプション）
    └── List<ロット明細価格査定>
        ├── 明細 seq
        ├── 基準単価
        ├── 期間調整率
        ├── 取引先調整率
        └── 特殊期間調整率（オプション）
```

---

## マイグレーション追加

### alembic/versions/003_create_appraisal_detail_tables.py

```python
"""create appraisal detail tables"""
from alembic import op
import sqlalchemy as sa

revision = "003"
down_revision = "002"
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.create_table(
        "lot_appraisal",
        sa.Column("id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column("appraisal_id", sa.Integer, sa.ForeignKey("appraisal.id"), nullable=False),
        sa.Column("lot_id", sa.Integer, sa.ForeignKey("lot.id"), nullable=False),
        sa.Column("equipment_cost", sa.Integer, nullable=True),
        sa.Column("order_premium", sa.Numeric, nullable=True),
        sa.Column("selection_premium", sa.Numeric, nullable=True),
        sa.Column("reservation_premium", sa.Numeric, nullable=True),
        sa.Column("adjustment_rate", sa.Numeric, nullable=True),
        sa.Column("quality_adjustment_rate", sa.Numeric, nullable=True),
        sa.Column("manufacturing_cost_unit_price", sa.Integer, nullable=True),
        sa.Column("profit_rate", sa.Numeric, nullable=True),
        sa.UniqueConstraint("appraisal_id", "lot_id", name="uq_lot_appraisal"),
    )

    op.create_table(
        "lot_detail_appraisal",
        sa.Column("id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column("lot_appraisal_id", sa.Integer, sa.ForeignKey("lot_appraisal.id"), nullable=False),
        sa.Column("detail_seq_no", sa.Integer, nullable=False),
        sa.Column("base_unit_price", sa.Integer, nullable=False),
        sa.Column("period_adjustment_rate", sa.Numeric, nullable=False),
        sa.Column("counterparty_adjustment_rate", sa.Numeric, nullable=False),
        sa.Column("special_period_adjustment_rate", sa.Numeric, nullable=True),
        sa.UniqueConstraint("lot_appraisal_id", "detail_seq_no", name="uq_lot_detail_appraisal"),
    )


def downgrade() -> None:
    op.drop_table("lot_detail_appraisal")
    op.drop_table("lot_appraisal")
```

---

## PBT 追加（hypothesis）

```python
# tests/test_appraisal_properties.py
from dataclasses import replace
from hypothesis import given
from hypothesis import strategies as st

@given(appraisal_common_strategy, appraisal_info_strategy)
def test_appraisal_number_unchanged_after_update(common, new_info) -> None:
    """査定更新後も査定番号は変わらない"""
    original = NormalAppraisal(common=common)
    updated = NormalAppraisal(common=replace(common, **new_info))
    assert updated.common.appraisal_number == common.appraisal_number


@given(
    st.text(min_size=1, max_size=50),
    st.floats(min_value=0.01, max_value=2.0),
)
def test_agreement_appraisal_preserves_contract_fields(contract_number: str, rate: float) -> None:
    """顧客契約査定は顧客契約番号と契約調整率を保持する"""
    appraisal = AgreementAppraisal(
        common=make_appraisal_common(),
        customer_contract_number=contract_number,
        contract_adjustment_rate=rate,
    )
    assert appraisal.customer_contract_number == contract_number
    assert appraisal.contract_adjustment_rate == rate
    assert appraisal.kind == "agreement"
```

---

## ci.sh への追加

```bash
echo "=== verify: Appraisal & Contract API ==="
STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
  -X POST "http://localhost:8000/sales-cases/$CASE_ID/direct/appraisals" \
  -H "Content-Type: application/json" \
  -d "{\"type\":\"normal\",\"appraisal_date\":\"2024-04-05\",
       \"delivery_date\":\"2024-05-01\",\"sales_market\":\"domestic\",
       \"tax_excluded_estimated_total\":500000,\"lot_appraisals\":[{\"lot_id\":1}],
       \"version\":1}")
[ "$STATUS" = "201" ] && echo "PASS appraisal-create"

STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
  -X PUT "http://localhost:8000/sales-cases/$CASE_ID/direct/appraisals" \
  -H "Content-Type: application/json" -d '{"appraisal_date":"2024-04-06","version":1}')
[ "$STATUS" = "409" ] && echo "PASS appraisal-version-conflict-409"
```

---

## 次のステップ

Step 15が完了したら [Step 16: 予約・委託販売案件の型定義](./step16.md) へ進む。
Phase 2 はこれで完了。
