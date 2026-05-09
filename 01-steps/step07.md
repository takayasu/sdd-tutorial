# Step 7: 在庫ロット集約API完全パッケージ + DB永続化

## 目的

### これは何か

Step 6 で定義したドメイン型を使い、CRUD API と DB 永続化を一気に完成させる。

- **SQLAlchemy ORM モデル** — `lot` テーブルへのマッピング
- **リポジトリ層** — DB アクセスをドメインロジックから隔離
- **FastAPI ルーター** — `POST /lots`, `GET /lots/{id}`, `GET /lots`, `PATCH /lots/{id}/status`
- **統合テスト** — `pytest` + `httpx.AsyncClient` でエンドポイントを直接叩く

### なぜやるのか

- ドメイン型が「実際に動く API」に繋がることで、型定義の意味が具体化する
- リポジトリパターンで DB アクセスを抽象化すると、テストでモックが不要になる（実DB を使う）
- `PATCH /lots/{id}/status` が「不正な状態遷移」を 422 で弾くことを統合テストで確認する

### 何がうれしいのか

- `InventoryLot` の `match` 文が実際の API レスポンスに反映される
- フロントエンドは Step 6 の TypeScript 型を使って型安全に API を呼び出せる
- 新しい状態遷移ルールを追加するとき、テストが先に失敗して実装漏れを防ぐ

## 完了条件

```bash
# 統合テスト通過
$ cd backend && pytest tests/test_lots.py -v
PASSED tests/test_lots.py::test_create_lot
PASSED tests/test_lots.py::test_get_lot
PASSED tests/test_lots.py::test_list_lots
PASSED tests/test_lots.py::test_invalid_status_transition

# ci.sh の verify セクション
$ ./ci.sh
PASS POST /lots
PASS GET /lots/{id}
PASS PATCH /lots/{id}/status invalid transition → 422
```

---

## 実装

### 1. SQLAlchemy ORM モデル（src/infra/models.py）

```python
from __future__ import annotations

from datetime import date

import sqlalchemy as sa
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class LotRow(Base):
    __tablename__ = "lot"

    id: Mapped[int] = mapped_column(sa.Integer, primary_key=True, autoincrement=True)
    lot_number_year: Mapped[int] = mapped_column(sa.Integer, nullable=False)
    lot_number_location: Mapped[str] = mapped_column(sa.Text, nullable=False)
    lot_number_seq: Mapped[int] = mapped_column(sa.Integer, nullable=False)
    division_code: Mapped[int] = mapped_column(sa.Integer, nullable=False)
    department_code: Mapped[int] = mapped_column(sa.Integer, nullable=False)
    section_code: Mapped[int] = mapped_column(sa.Integer, nullable=False)
    process_category: Mapped[int] = mapped_column(sa.Integer, nullable=False)
    inspection_category: Mapped[int] = mapped_column(sa.Integer, nullable=False)
    manufacturing_category: Mapped[int] = mapped_column(sa.Integer, nullable=False)
    status: Mapped[str] = mapped_column(sa.Text, nullable=False)
    manufacturing_completed_date: Mapped[date | None] = mapped_column(sa.Date, nullable=True)
    shipping_deadline_date: Mapped[date | None] = mapped_column(sa.Date, nullable=True)
    shipped_date: Mapped[date | None] = mapped_column(sa.Date, nullable=True)
    version: Mapped[int] = mapped_column(sa.Integer, nullable=False, server_default="1")

    __table_args__ = (
        sa.UniqueConstraint(
            "lot_number_year",
            "lot_number_location",
            "lot_number_seq",
            name="uq_lot_number",
        ),
    )
```

### 2. Pydantic スキーマ（src/schemas/lot.py）

```python
from __future__ import annotations

from datetime import date
from typing import Literal

from pydantic import BaseModel


class LotCreateRequest(BaseModel):
    lot_number_year: int
    lot_number_location: str
    lot_number_seq: int
    division_code: int
    department_code: int
    section_code: int
    process_category: int
    inspection_category: int
    manufacturing_category: int


class StatusTransitionRequest(BaseModel):
    status: Literal["manufactured", "shipping_instructed", "shipped"]
    manufacturing_completed_date: date | None = None
    shipping_deadline_date: date | None = None
    shipped_date: date | None = None


class LotResponse(BaseModel):
    id: int
    lot_number_year: int
    lot_number_location: str
    lot_number_seq: int
    division_code: int
    department_code: int
    section_code: int
    process_category: int
    inspection_category: int
    manufacturing_category: int
    status: str
    manufacturing_completed_date: date | None
    shipping_deadline_date: date | None
    shipped_date: date | None
    version: int

    model_config = {"from_attributes": True}
```

### 3. リポジトリ（src/infra/lot_repository.py）

```python
from __future__ import annotations

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from src.infra.models import LotRow
from src.schemas.lot import LotCreateRequest, LotResponse, StatusTransitionRequest


class LotRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def create(self, req: LotCreateRequest) -> LotResponse:
        row = LotRow(
            lot_number_year=req.lot_number_year,
            lot_number_location=req.lot_number_location,
            lot_number_seq=req.lot_number_seq,
            division_code=req.division_code,
            department_code=req.department_code,
            section_code=req.section_code,
            process_category=req.process_category,
            inspection_category=req.inspection_category,
            manufacturing_category=req.manufacturing_category,
            status="manufacturing",
        )
        self._session.add(row)
        await self._session.flush()
        await self._session.refresh(row)
        return LotResponse.model_validate(row)

    async def get_by_id(self, lot_id: int) -> LotRow | None:
        return await self._session.get(LotRow, lot_id)

    async def list_all(self) -> list[LotResponse]:
        result = await self._session.execute(select(LotRow))
        return [LotResponse.model_validate(r) for r in result.scalars().all()]

    async def transition_status(
        self, lot_id: int, req: StatusTransitionRequest
    ) -> LotResponse:
        row = await self._session.get(LotRow, lot_id)
        if row is None:
            raise ValueError(f"Lot {lot_id} not found")

        _validate_transition(row.status, req)

        row.status = req.status
        if req.manufacturing_completed_date is not None:
            row.manufacturing_completed_date = req.manufacturing_completed_date
        if req.shipping_deadline_date is not None:
            row.shipping_deadline_date = req.shipping_deadline_date
        if req.shipped_date is not None:
            row.shipped_date = req.shipped_date
        row.version += 1

        await self._session.flush()
        await self._session.refresh(row)
        return LotResponse.model_validate(row)


_VALID_TRANSITIONS: dict[str, set[str]] = {
    "manufacturing": {"manufactured"},
    "manufactured": {"shipping_instructed"},
    "shipping_instructed": {"shipped"},
    "shipped": set(),
}


def _validate_transition(current: str, req: StatusTransitionRequest) -> None:
    allowed = _VALID_TRANSITIONS.get(current, set())
    if req.status not in allowed:
        raise ValueError(
            f"Cannot transition from '{current}' to '{req.status}'. "
            f"Allowed: {allowed or 'none'}"
        )
    if req.status == "manufactured" and req.manufacturing_completed_date is None:
        raise ValueError("manufacturing_completed_date is required for 'manufactured'")
    if req.status == "shipping_instructed" and req.shipping_deadline_date is None:
        raise ValueError("shipping_deadline_date is required for 'shipping_instructed'")
    if req.status == "shipped" and req.shipped_date is None:
        raise ValueError("shipped_date is required for 'shipped'")
```

### 4. ルーター（src/routers/lots.py）

```python
from __future__ import annotations

from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession

from src.database import get_session
from src.infra.lot_repository import LotRepository
from src.schemas.lot import LotCreateRequest, LotResponse, StatusTransitionRequest

router = APIRouter(prefix="/lots", tags=["lots"])


@router.post("", response_model=LotResponse, status_code=201)
async def create_lot(
    body: LotCreateRequest,
    session: AsyncSession = Depends(get_session),
) -> LotResponse:
    repo = LotRepository(session)
    result = await repo.create(body)
    await session.commit()
    return result


@router.get("", response_model=list[LotResponse])
async def list_lots(
    session: AsyncSession = Depends(get_session),
) -> list[LotResponse]:
    repo = LotRepository(session)
    return await repo.list_all()


@router.get("/{lot_id}", response_model=LotResponse)
async def get_lot(
    lot_id: int,
    session: AsyncSession = Depends(get_session),
) -> LotResponse:
    repo = LotRepository(session)
    row = await repo.get_by_id(lot_id)
    if row is None:
        raise HTTPException(status_code=404, detail="Lot not found")
    return LotResponse.model_validate(row)


@router.patch("/{lot_id}/status", response_model=LotResponse)
async def transition_lot_status(
    lot_id: int,
    body: StatusTransitionRequest,
    session: AsyncSession = Depends(get_session),
) -> LotResponse:
    repo = LotRepository(session)
    try:
        result = await repo.transition_status(lot_id, body)
        await session.commit()
        return result
    except ValueError as e:
        raise HTTPException(status_code=422, detail=str(e)) from e
```

### 5. main.py にルーター登録

```python
# src/main.py に追加
from src.routers.lots import router as lots_router

app.include_router(lots_router)
```

---

## 統合テスト

### tests/test_lots.py

```python
from __future__ import annotations

import pytest
from httpx import AsyncClient, ASGITransport

from src.main import app


@pytest.fixture
async def client():
    async with AsyncClient(
        transport=ASGITransport(app=app), base_url="http://test"
    ) as c:
        yield c


LOT_PAYLOAD = {
    "lot_number_year": 2024,
    "lot_number_location": "TK",
    "lot_number_seq": 1,
    "division_code": 1,
    "department_code": 10,
    "section_code": 100,
    "process_category": 1,
    "inspection_category": 1,
    "manufacturing_category": 1,
}


async def test_create_lot(client: AsyncClient) -> None:
    res = await client.post("/lots", json=LOT_PAYLOAD)
    assert res.status_code == 201
    body = res.json()
    assert body["status"] == "manufacturing"
    assert body["id"] is not None


async def test_get_lot(client: AsyncClient) -> None:
    create_res = await client.post("/lots", json=LOT_PAYLOAD)
    lot_id = create_res.json()["id"]

    res = await client.get(f"/lots/{lot_id}")
    assert res.status_code == 200
    assert res.json()["id"] == lot_id


async def test_list_lots(client: AsyncClient) -> None:
    await client.post("/lots", json=LOT_PAYLOAD)
    res = await client.get("/lots")
    assert res.status_code == 200
    assert isinstance(res.json(), list)
    assert len(res.json()) >= 1


async def test_valid_status_transition(client: AsyncClient) -> None:
    from datetime import date

    create_res = await client.post("/lots", json=LOT_PAYLOAD)
    lot_id = create_res.json()["id"]

    res = await client.patch(
        f"/lots/{lot_id}/status",
        json={
            "status": "manufactured",
            "manufacturing_completed_date": str(date.today()),
        },
    )
    assert res.status_code == 200
    assert res.json()["status"] == "manufactured"


async def test_invalid_status_transition(client: AsyncClient) -> None:
    create_res = await client.post("/lots", json=LOT_PAYLOAD)
    lot_id = create_res.json()["id"]

    # manufacturing → shipped は不正（中間状態をスキップ）
    res = await client.patch(
        f"/lots/{lot_id}/status",
        json={"status": "shipped", "shipped_date": "2024-03-01"},
    )
    assert res.status_code == 422
```

---

## ci.sh への追加

```bash
echo "=== verify: Lots API ==="
LOT_ID=$(curl -sf -X POST http://localhost:8000/lots \
  -H "Content-Type: application/json" \
  -d '{"lot_number_year":2024,"lot_number_location":"TK","lot_number_seq":1,
       "division_code":1,"department_code":10,"section_code":100,
       "process_category":1,"inspection_category":1,"manufacturing_category":1}' \
  | python -c "import sys,json; print(json.load(sys.stdin)['id'])")
echo "PASS POST /lots (id=$LOT_ID)"

curl -sf "http://localhost:8000/lots/$LOT_ID" >/dev/null
echo "PASS GET /lots/$LOT_ID"

STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
  -X PATCH "http://localhost:8000/lots/$LOT_ID/status" \
  -H "Content-Type: application/json" \
  -d '{"status":"shipped","shipped_date":"2024-03-01"}')
[ "$STATUS" = "422" ] && echo "PASS PATCH /lots/$LOT_ID/status invalid transition → 422"
```

---

## 次のステップ

Step 7が完了したら [Step 8: PBT導入（hypothesis + fast-check）](./step08.md) へ進む。
