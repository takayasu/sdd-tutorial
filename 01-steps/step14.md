# Step 14: 直接販売案件の集約API完全パッケージ + URL 集約規約

## 目的

### これは何か

直接販売案件のライフサイクル（査定前→査定済み→契約済み→出荷指示→出荷完了）を **Step 7 の「集約API完全パッケージ」テンプレに従って** REST API 化する。すなわち以下を**1 step で揃える**:

| 要素 | 内容 |
|---|---|
| Mutation | 案件作成 / 査定作成 / 契約締結 / 出荷指示 / 出荷完了 / 取消系 |
| **詳細 GET** | `GET /sales-cases/{id}` — `case_type` を含むポリモーフィック |
| **一覧 GET** | `GET /sales-cases?case_type=&status=&limit=&offset=` |
| **楽観ロック** | `version` フィールドで競合を 409 で弾く |
| **エラー形式** | 全レスポンス `application/problem+json` |
| **URL 集約規約** | mutation も含め全エンドポイントを `/sales-cases/{id}/...` 配下に集約 |

### URL 集約規約（Step 14 / 15 / 18 共通）

| 避ける | 採用 |
|---|---|
| `POST /reservation-cases/{id}/appraisals` | `POST /sales-cases/{id}/reservation/appraisals` |
| `GET /reservation-cases/{id}` | `GET /sales-cases/{id}` （`case_type` でポリモーフィック） |

これにより「ID は分かっているが種類が分からない時にどの URL を叩くか」問題が消え、フロントの SWR キーも `/sales-cases/${id}` プレフィックスで統一できる。

### なぜやるのか

- DSL の `behavior`（販売案件を作成する、価格査定を作成する、販売契約を締結する等）を API エンドポイントとして実現する
- 「査定前の案件に契約を締結する」操作が 422 で弾かれることを統合テストで確認する
- Step 18 で予約・委託を足すときも URL 規約が共通なので増分が小さい

## 完了条件

```bash
# 1. 販売案件作成
$ curl -sf -X POST http://localhost:8000/sales-cases \
  -H "Content-Type: application/json" \
  -d '{"case_type":"direct","lot_ids":[1],"division_code":1,"sales_date":"2024-04-01"}'
{"id":1,"status":"before_appraisal","version":1,...}

# 2. 詳細 GET
$ curl -sf http://localhost:8000/sales-cases/1
{"id":1,"case_type":"direct","status":"before_appraisal","version":1}

# 3. 一覧 GET（ページング）
$ curl -sf "http://localhost:8000/sales-cases?case_type=direct&limit=10&offset=0"
{"items":[...],"total":1,"limit":10,"offset":0}

# 4. 査定前案件への契約締結 → 422
$ curl -s -o /dev/null -w "%{http_code}" \
  -X POST http://localhost:8000/sales-cases/1/direct/contracts \
  -H "Content-Type: application/json" -d '{"contract_date":"2024-04-15","version":1}'
422

# 5. 楽観ロック競合 → 409
$ curl -s -o /dev/null -w "%{http_code}" \
  -X POST http://localhost:8000/sales-cases/1/direct/appraisals \
  -H "Content-Type: application/json" -d '{"appraisal_date":"2024-04-05","version":99}'
409

# ci.sh が緑
$ ./ci.sh
PASS POST /sales-cases
PASS GET /sales-cases/1
PASS GET /sales-cases (list)
PASS invalid-transition → 422
PASS version-conflict → 409
```

---

## ドメインワークフロー

### src/domain/sales_case_workflows.py

```python
from __future__ import annotations

from datetime import date

from src.domain.sales_case import (
    AppraisedCase,
    BeforeAppraisalCase,
    ContractedCase,
    PriceAppraisal,
    SalesCaseCommon,
    SalesContract,
    ShippingCompletedCase,
    ShippingInstructedCase,
)


def create_sales_case(common: SalesCaseCommon) -> BeforeAppraisalCase:
    return BeforeAppraisalCase(common=common)


def create_appraisal(case: BeforeAppraisalCase, appraisal: PriceAppraisal) -> AppraisedCase:
    return AppraisedCase(common=case.common, appraisal=appraisal)


def delete_appraisal(case: AppraisedCase) -> BeforeAppraisalCase:
    return BeforeAppraisalCase(common=case.common)


def conclude_contract(case: AppraisedCase, contract: SalesContract) -> ContractedCase:
    return ContractedCase(common=case.common, appraisal=case.appraisal, contract=contract)


def delete_contract(case: ContractedCase) -> AppraisedCase:
    return AppraisedCase(common=case.common, appraisal=case.appraisal)


def instruct_shipping(case: ContractedCase, shipping_instruction_date: date) -> ShippingInstructedCase:
    return ShippingInstructedCase(
        common=case.common,
        appraisal=case.appraisal,
        contract=case.contract,
        shipping_instruction_date=shipping_instruction_date,
    )


def cancel_shipping_instruction(case: ShippingInstructedCase) -> ContractedCase:
    return ContractedCase(
        common=case.common,
        appraisal=case.appraisal,
        contract=case.contract,
    )


def complete_shipping(case: ShippingInstructedCase, shipping_completed_date: date) -> ShippingCompletedCase:
    return ShippingCompletedCase(
        common=case.common,
        appraisal=case.appraisal,
        contract=case.contract,
        shipping_instruction_date=case.shipping_instruction_date,
        shipping_completed_date=shipping_completed_date,
    )
```

---

## APIルーター（抜粋）

### src/routers/sales_cases.py

```python
from __future__ import annotations

from fastapi import APIRouter, Depends, HTTPException, Query
from sqlalchemy.ext.asyncio import AsyncSession

from src.database import get_session

router = APIRouter(prefix="/sales-cases", tags=["sales-cases"])


@router.post("", status_code=201)
async def create_sales_case(body: SalesCaseCreateRequest, session: AsyncSession = Depends(get_session)):
    repo = SalesCaseRepository(session)
    result = await repo.create(body)
    await session.commit()
    return result


@router.get("")
async def list_sales_cases(
    case_type: str | None = Query(None),
    status: str | None = Query(None),
    limit: int = Query(20, ge=1, le=100),
    offset: int = Query(0, ge=0),
    session: AsyncSession = Depends(get_session),
):
    repo = SalesCaseRepository(session)
    return await repo.list_all(case_type=case_type, status=status, limit=limit, offset=offset)


@router.get("/{sales_case_id}")
async def get_sales_case(sales_case_id: int, session: AsyncSession = Depends(get_session)):
    repo = SalesCaseRepository(session)
    row = await repo.get_by_id(sales_case_id)
    if row is None:
        raise HTTPException(status_code=404, detail="SalesCase not found")
    return row


@router.post("/{sales_case_id}/direct/appraisals")
async def create_appraisal(
    sales_case_id: int,
    body: AppraisalCreateRequest,
    session: AsyncSession = Depends(get_session),
):
    repo = SalesCaseRepository(session)
    try:
        result = await repo.create_appraisal(sales_case_id, body)
        await session.commit()
        return result
    except VersionConflictError as e:
        raise HTTPException(status_code=409, detail=str(e)) from e
    except InvalidTransitionError as e:
        raise HTTPException(status_code=422, detail=str(e)) from e


@router.post("/{sales_case_id}/direct/contracts")
async def conclude_contract(
    sales_case_id: int,
    body: ContractCreateRequest,
    session: AsyncSession = Depends(get_session),
):
    repo = SalesCaseRepository(session)
    try:
        result = await repo.conclude_contract(sales_case_id, body)
        await session.commit()
        return result
    except VersionConflictError as e:
        raise HTTPException(status_code=409, detail=str(e)) from e
    except InvalidTransitionError as e:
        raise HTTPException(status_code=422, detail=str(e)) from e


@router.post("/{sales_case_id}/direct/shipping-instruction")
async def instruct_shipping(
    sales_case_id: int,
    body: ShippingInstructionRequest,
    session: AsyncSession = Depends(get_session),
):
    repo = SalesCaseRepository(session)
    try:
        result = await repo.instruct_shipping(sales_case_id, body)
        await session.commit()
        return result
    except VersionConflictError as e:
        raise HTTPException(status_code=409, detail=str(e)) from e
    except InvalidTransitionError as e:
        raise HTTPException(status_code=422, detail=str(e)) from e
```

---

## エンドポイント一覧

| メソッド | パス | 動作 |
|---|---|---|
| POST | `/sales-cases` | 販売案件を作成する |
| GET | `/sales-cases` | 一覧取得（`case_type`, `status`, `limit`, `offset`） |
| GET | `/sales-cases/{id}` | 詳細取得（`case_type` でポリモーフィック） |
| DELETE | `/sales-cases/{id}` | 販売案件を削除する |
| POST | `/sales-cases/{id}/direct/appraisals` | 価格査定を作成する |
| PUT | `/sales-cases/{id}/direct/appraisals` | 価格査定を更新する |
| DELETE | `/sales-cases/{id}/direct/appraisals` | 価格査定を削除する |
| POST | `/sales-cases/{id}/direct/contracts` | 販売契約を締結する |
| DELETE | `/sales-cases/{id}/direct/contracts` | 販売契約を削除する |
| POST | `/sales-cases/{id}/direct/shipping-instruction` | 出荷を指示する |
| POST | `/sales-cases/{id}/direct/shipping-completion` | 出荷完了を記録する |
| DELETE | `/sales-cases/{id}/direct/shipping-instruction` | 出荷指示を取り消す |

すべての mutation は body に **`version: int`** を必須化する。サーバは `WHERE version = {expected}` の UPDATE を実行し、行数 0 で 409 Conflict + problem+json を返す。

---

## PBT 追加（hypothesis）

```python
# tests/test_sales_case_properties.py
from hypothesis import given
from src.domain.sales_case_workflows import create_appraisal, delete_appraisal, conclude_contract, delete_contract

@given(common_strategy, appraisal_strategy)
def test_appraisal_roundtrip(common, appraisal) -> None:
    """査定作成→削除で査定前に戻る（往復性）"""
    before = BeforeAppraisalCase(common=common)
    appraised = create_appraisal(before, appraisal)
    restored = delete_appraisal(appraised)
    assert restored.common == common


@given(common_strategy, appraisal_strategy, contract_strategy)
def test_contract_roundtrip(common, appraisal, contract) -> None:
    """契約締結→削除で査定済みに戻る（往復性）"""
    appraised = AppraisedCase(common=common, appraisal=appraisal)
    contracted = conclude_contract(appraised, contract)
    restored = delete_contract(contracted)
    assert restored.common == common
    assert restored.appraisal == appraisal
```

---

## ci.sh への追加

```bash
echo "=== verify: Sales Cases API ==="
CASE_ID=$(curl -sf -X POST http://localhost:8000/sales-cases \
  -H "Content-Type: application/json" \
  -d '{"case_type":"direct","lot_ids":[1],"division_code":1,"sales_date":"2024-04-01"}' \
  | python -c "import sys,json; print(json.load(sys.stdin)['id'])")
echo "PASS POST /sales-cases (id=$CASE_ID)"

curl -sf "http://localhost:8000/sales-cases/$CASE_ID" >/dev/null
echo "PASS GET /sales-cases/$CASE_ID"

curl -sf "http://localhost:8000/sales-cases?limit=10&offset=0" >/dev/null
echo "PASS GET /sales-cases (list)"

STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
  -X POST "http://localhost:8000/sales-cases/$CASE_ID/direct/contracts" \
  -H "Content-Type: application/json" -d '{"contract_date":"2024-04-15","version":1}')
[ "$STATUS" = "422" ] && echo "PASS invalid-transition → 422"

STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
  -X POST "http://localhost:8000/sales-cases/$CASE_ID/direct/appraisals" \
  -H "Content-Type: application/json" -d '{"appraisal_date":"2024-04-05","version":99}')
[ "$STATUS" = "409" ] && echo "PASS version-conflict → 409"
```

---

## 次のステップ

Step 14が完了したら [Step 15: 価格査定・販売契約のAPI実装](./step15.md) へ進む。
