# Step 6: 在庫ロットの型定義

## 目的

### これは何か

ドメインモデルの中心となる「在庫ロット（InventoryLot）」を、Python の frozen dataclass と TypeScript の discriminated union で型定義する。

- **状態を型で表現する**: `製造中 | 製造完了 | 出荷指示済み | 出荷済み` を Union 型で表す
- **不正な状態を型レベルで弾く**: `shipped_date` は `ShippedLot` にしか存在しない
- **スマートコンストラクタ**: `Quantity(0)` は `__post_init__` で即座に例外を投げる

### なぜやるのか

- 「出荷済みなのに `shipped_date` が None」のような矛盾した状態をランタイムエラーではなく型エラーにする
- AIが生成したコードが「意味的に正しいか」をmypyとTypeScriptコンパイラが自動検証する
- ドメイン知識がコードに直接埋め込まれ、ドキュメントと乖離しない

### 何がうれしいのか

- `match lot:` で全状態を網羅しないとmypyが警告する
- フロントエンドの `switch (lot.status)` も全分岐を書かないとTypeScriptが怒る
- 新しい状態（例: 返品中）を追加したとき、対応漏れがコンパイル時に発覚する

## 完了条件

```bash
# Python: 型チェック通過
$ cd backend && mypy src/domain/lot.py --strict
Success: no issues found in 1 source file

# Python: リントチェック通過
$ ruff check src/domain/lot.py
All checks passed!

# TypeScript: 型チェック通過
$ cd frontend && npx tsc --noEmit
（エラーなし）

# ci.sh が緑
$ ./ci.sh
（変更なし — この Step は型定義のみ、APIエンドポイントは Step 7）
```

---

## Python 型定義

### 1. 依存追加（pyproject.toml）

```toml
[project.optional-dependencies]
dev = [
    "pytest>=8",
    "pytest-asyncio>=0.23",
    "httpx>=0.27",
    "ruff>=0.6",
    "mypy>=1.10",
    "hypothesis>=6",
]
```

```bash
cd backend && uv sync
```

### 2. src/domain/lot.py

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import date
from typing import Literal


# ---------- 値オブジェクト ----------

@dataclass(frozen=True)
class LotNumber:
    year: int
    location: str
    seq: int

    def __post_init__(self) -> None:
        if self.year < 2000 or self.year > 2099:
            raise ValueError(f"lot_number_year must be 2000–2099, got {self.year}")
        if not self.location:
            raise ValueError("lot_number_location must not be empty")
        if self.seq < 1:
            raise ValueError(f"lot_number_seq must be >= 1, got {self.seq}")

    def __str__(self) -> str:
        return f"{self.year}-{self.location}-{self.seq:04d}"


@dataclass(frozen=True)
class Quantity:
    value: int

    def __post_init__(self) -> None:
        if self.value <= 0:
            raise ValueError(f"Quantity must be > 0, got {self.value}")


@dataclass(frozen=True)
class Count:
    value: int

    def __post_init__(self) -> None:
        if self.value < 0:
            raise ValueError(f"Count must be >= 0, got {self.value}")


@dataclass(frozen=True)
class Amount:
    value: int  # 円単位の整数（小数点以下切り捨て）

    def __post_init__(self) -> None:
        if self.value < 0:
            raise ValueError(f"Amount must be >= 0, got {self.value}")


# ---------- 共通フィールド ----------

@dataclass(frozen=True)
class LotCommon:
    lot_number: LotNumber
    division_code: int
    department_code: int
    section_code: int
    process_category: int
    inspection_category: int
    manufacturing_category: int


# ---------- 状態別 Lot ----------

@dataclass(frozen=True)
class ManufacturingLot:
    """製造中"""
    status: Literal["manufacturing"] = "manufacturing"
    common: LotCommon = ...  # type: ignore[assignment]

    def __post_init__(self) -> None:
        object.__setattr__(self, "status", "manufacturing")


@dataclass(frozen=True)
class ManufacturedLot:
    """製造完了"""
    common: LotCommon
    manufacturing_completed_date: date
    status: Literal["manufactured"] = "manufactured"

    def __post_init__(self) -> None:
        object.__setattr__(self, "status", "manufactured")


@dataclass(frozen=True)
class ShippingInstructedLot:
    """出荷指示済み"""
    common: LotCommon
    manufacturing_completed_date: date
    shipping_deadline_date: date
    status: Literal["shipping_instructed"] = "shipping_instructed"

    def __post_init__(self) -> None:
        object.__setattr__(self, "status", "shipping_instructed")
        if self.shipping_deadline_date < self.manufacturing_completed_date:
            raise ValueError("shipping_deadline_date must be >= manufacturing_completed_date")


@dataclass(frozen=True)
class ShippedLot:
    """出荷済み"""
    common: LotCommon
    manufacturing_completed_date: date
    shipping_deadline_date: date
    shipped_date: date
    status: Literal["shipped"] = "shipped"

    def __post_init__(self) -> None:
        object.__setattr__(self, "status", "shipped")
        if self.shipped_date < self.manufacturing_completed_date:
            raise ValueError("shipped_date must be >= manufacturing_completed_date")


# ---------- Union 型 ----------

InventoryLot = ManufacturingLot | ManufacturedLot | ShippingInstructedLot | ShippedLot
```

### 3. mypy 設定（pyproject.toml に追記）

```toml
[tool.mypy]
python_version = "3.12"
strict = true
exclude = ["alembic/"]
```

### 4. 使用例

```python
from src.domain.lot import (
    LotCommon,
    LotNumber,
    ManufacturingLot,
    ManufacturedLot,
    ShippedLot,
    InventoryLot,
)
from datetime import date

common = LotCommon(
    lot_number=LotNumber(year=2024, location="TK", seq=1),
    division_code=1,
    department_code=10,
    section_code=100,
    process_category=1,
    inspection_category=1,
    manufacturing_category=1,
)

# 製造中 → 製造完了 への遷移
manufacturing = ManufacturingLot(common=common)
manufactured = ManufacturedLot(
    common=common,
    manufacturing_completed_date=date(2024, 3, 1),
)

# match で全状態を網羅
def describe(lot: InventoryLot) -> str:
    match lot:
        case ManufacturingLot():
            return "製造中"
        case ManufacturedLot(manufacturing_completed_date=d):
            return f"製造完了: {d}"
        case ShippingInstructedLot(shipping_deadline_date=d):
            return f"出荷期限: {d}"
        case ShippedLot(shipped_date=d):
            return f"出荷済み: {d}"
```

---

## TypeScript 型定義

### 1. src/types/lot.ts

```typescript
// ---------- 値オブジェクト ----------

export interface LotNumber {
  readonly year: number
  readonly location: string
  readonly seq: number
}

export function formatLotNumber(lot: LotNumber): string {
  return `${lot.year}-${lot.location}-${String(lot.seq).padStart(4, '0')}`
}

// ---------- 共通フィールド ----------

export interface LotCommon {
  readonly lotNumber: LotNumber
  readonly divisionCode: number
  readonly departmentCode: number
  readonly sectionCode: number
  readonly processCategory: number
  readonly inspectionCategory: number
  readonly manufacturingCategory: number
}

// ---------- 状態別 Lot ----------

export interface ManufacturingLot extends LotCommon {
  readonly status: 'manufacturing'
}

export interface ManufacturedLot extends LotCommon {
  readonly status: 'manufactured'
  readonly manufacturingCompletedDate: string  // ISO 8601
}

export interface ShippingInstructedLot extends LotCommon {
  readonly status: 'shipping_instructed'
  readonly manufacturingCompletedDate: string
  readonly shippingDeadlineDate: string
}

export interface ShippedLot extends LotCommon {
  readonly status: 'shipped'
  readonly manufacturingCompletedDate: string
  readonly shippingDeadlineDate: string
  readonly shippedDate: string
}

// ---------- Discriminated Union ----------

export type InventoryLot =
  | ManufacturingLot
  | ManufacturedLot
  | ShippingInstructedLot
  | ShippedLot

// ---------- 型ガード ----------

export function isShipped(lot: InventoryLot): lot is ShippedLot {
  return lot.status === 'shipped'
}
```

### 2. 使用例

```typescript
import type { InventoryLot } from '@/types/lot'

function describeLot(lot: InventoryLot): string {
  switch (lot.status) {
    case 'manufacturing':
      return '製造中'
    case 'manufactured':
      return `製造完了: ${lot.manufacturingCompletedDate}`
    case 'shipping_instructed':
      return `出荷期限: ${lot.shippingDeadlineDate}`
    case 'shipped':
      return `出荷済み: ${lot.shippedDate}`
    // TypeScript が全分岐を網羅しているか検証
    default: {
      const _exhaustive: never = lot
      return _exhaustive
    }
  }
}
```

---

## Pydantic スキーマ（API 層）との分離

ドメイン型（`src/domain/lot.py`）と API スキーマ（`src/schemas/lot.py`）は**別ファイルに分ける**。

```python
# src/schemas/lot.py — FastAPI レスポンス用
from __future__ import annotations

from datetime import date
from typing import Literal

from pydantic import BaseModel


class ManufacturingLotResponse(BaseModel):
    status: Literal["manufacturing"]
    lot_number_year: int
    lot_number_location: str
    lot_number_seq: int
    division_code: int
    department_code: int
    section_code: int
    process_category: int
    inspection_category: int
    manufacturing_category: int


class ManufacturedLotResponse(ManufacturingLotResponse):
    status: Literal["manufactured"]
    manufacturing_completed_date: date


class ShippingInstructedLotResponse(ManufacturedLotResponse):
    status: Literal["shipping_instructed"]
    shipping_deadline_date: date


class ShippedLotResponse(ShippingInstructedLotResponse):
    status: Literal["shipped"]
    shipped_date: date


LotResponse = (
    ManufacturingLotResponse
    | ManufacturedLotResponse
    | ShippingInstructedLotResponse
    | ShippedLotResponse
)
```

### 分離する理由

| 層 | 型 | 役割 |
|---|---|---|
| ドメイン層 | `frozen dataclass` | ビジネスルールの表現・検証 |
| API層 | `Pydantic BaseModel` | シリアライズ・バリデーション・OpenAPI生成 |

ドメイン型は Pydantic を知らず、Pydantic スキーマはドメインロジックを持たない。

---

## 次のステップ

Step 6が完了したら [Step 7: 在庫ロット集約API](./step07.md) へ進む。
