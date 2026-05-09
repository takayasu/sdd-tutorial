# Step 16: 予約・委託販売案件の型定義

## 目的

### これは何か

`domain-model-section3.md` の DSL を Python frozen dataclass と TypeScript discriminated union に変換する。予約販売案件・委託販売案件を追加し、販売案件の全種別（直接/予約/委託）を統合した型を定義する。

### なぜやるのか

- 販売案件には3種類（直接/予約/委託）があり、それぞれ全く異なるライフサイクルを持つ
- 「予約販売案件に販売契約を締結する」といった業務上ありえない操作を、型レベルで不可能にする
- 全ステップ完了後、元の `domain-model-sales-management.md` と同一のドメインモデルになる

### 何がうれしいのか

- 3種類の販売案件を1つの `SalesCase` Union 型で統合することで、「どの種類の案件か」を型で判別できる
- `match case:` / `switch (case.status)` で全種別を網羅しないと型エラーになる

## 完了条件

```bash
# Python: 型チェック通過
$ cd backend && mypy src/domain/reservation_case.py src/domain/consignment_case.py --strict
Success: no issues found in 2 source files

# TypeScript: 型チェック通過
$ cd frontend && npx tsc --noEmit
（エラーなし）
```

---

## Python 型定義

### src/domain/reservation_case.py

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import date
from typing import Literal

from src.domain.sales_case import SalesCaseCommon


# ---------- 予約価格 ----------

@dataclass(frozen=True)
class ReservationPriceCommon:
    appraisal_number_year: int
    appraisal_number_month: int
    appraisal_number_seq: int
    appraisal_date: date
    estimated_lot_info: str
    estimated_amount: int


@dataclass(frozen=True)
class UndeterminedReservationPrice:
    common: ReservationPriceCommon
    kind: Literal["undetermined"] = "undetermined"

    def __post_init__(self) -> None:
        object.__setattr__(self, "kind", "undetermined")


@dataclass(frozen=True)
class DeterminedReservationPrice:
    common: ReservationPriceCommon
    determined_date: date
    determined_amount: int
    kind: Literal["determined"] = "determined"

    def __post_init__(self) -> None:
        object.__setattr__(self, "kind", "determined")


EstimatePriceAppraisal = UndeterminedReservationPrice | DeterminedReservationPrice


# ---------- 予約販売案件（4段階の状態） ----------

@dataclass(frozen=True)
class BeforeReservationPriceCase:
    """予約価格査定前"""
    common: SalesCaseCommon
    status: Literal["before_reservation_price"] = "before_reservation_price"

    def __post_init__(self) -> None:
        object.__setattr__(self, "status", "before_reservation_price")


@dataclass(frozen=True)
class EstimateAppraisedCase:
    """予約価格査定済み"""
    common: SalesCaseCommon
    appraisal: EstimatePriceAppraisal
    status: Literal["estimate_appraised"] = "estimate_appraised"

    def __post_init__(self) -> None:
        object.__setattr__(self, "status", "estimate_appraised")


@dataclass(frozen=True)
class EstimateDeterminedCase:
    """予約価格確定済み"""
    common: SalesCaseCommon
    appraisal: EstimatePriceAppraisal
    determined_date: date
    status: Literal["estimate_determined"] = "estimate_determined"

    def __post_init__(self) -> None:
        object.__setattr__(self, "status", "estimate_determined")


@dataclass(frozen=True)
class EstimateDeliveredCase:
    """予約出荷済み"""
    common: SalesCaseCommon
    appraisal: EstimatePriceAppraisal
    determined_date: date
    delivery_date: date
    status: Literal["estimate_delivered"] = "estimate_delivered"

    def __post_init__(self) -> None:
        object.__setattr__(self, "status", "estimate_delivered")


ReservationSalesCase = (
    BeforeReservationPriceCase
    | EstimateAppraisedCase
    | EstimateDeterminedCase
    | EstimateDeliveredCase
)
```

### src/domain/consignment_case.py

```python
from __future__ import annotations

from dataclasses import dataclass
from typing import Literal

from src.domain.sales_case import SalesCaseCommon


@dataclass(frozen=True)
class ConsignorInfo:
    consignor_code: str
    consignor_name: str


@dataclass(frozen=True)
class ConsignmentResult:
    result_amount: int
    result_date: str  # ISO 8601


@dataclass(frozen=True)
class BeforeConsignmentCase:
    """委託指定前"""
    common: SalesCaseCommon
    status: Literal["before_consignment"] = "before_consignment"

    def __post_init__(self) -> None:
        object.__setattr__(self, "status", "before_consignment")


@dataclass(frozen=True)
class ConsignmentDesignatedCase:
    """委託先指定済み"""
    common: SalesCaseCommon
    consignor_info: ConsignorInfo
    status: Literal["consignment_designated"] = "consignment_designated"

    def __post_init__(self) -> None:
        object.__setattr__(self, "status", "consignment_designated")


@dataclass(frozen=True)
class ConsignmentResultEnteredCase:
    """委託結果入力済み"""
    common: SalesCaseCommon
    consignor_info: ConsignorInfo
    result: ConsignmentResult
    status: Literal["consignment_result_entered"] = "consignment_result_entered"

    def __post_init__(self) -> None:
        object.__setattr__(self, "status", "consignment_result_entered")


ConsignmentSalesCase = (
    BeforeConsignmentCase
    | ConsignmentDesignatedCase
    | ConsignmentResultEnteredCase
)
```

### src/domain/all_cases.py（全種別統合）

```python
from __future__ import annotations

from src.domain.sales_case import DirectSalesCase
from src.domain.reservation_case import ReservationSalesCase
from src.domain.consignment_case import ConsignmentSalesCase

SalesCase = DirectSalesCase | ReservationSalesCase | ConsignmentSalesCase
```

---

## TypeScript 型定義

### src/types/reservation_case.ts

```typescript
import type { SalesCaseCommon } from '@/types/sales_case'

export interface ReservationPriceCommon {
  readonly appraisalNumberYear: number
  readonly appraisalNumberMonth: number
  readonly appraisalNumberSeq: number
  readonly appraisalDate: string
  readonly estimatedLotInfo: string
  readonly estimatedAmount: number
}

export interface UndeterminedReservationPrice {
  readonly kind: 'undetermined'
  readonly common: ReservationPriceCommon
}

export interface DeterminedReservationPrice {
  readonly kind: 'determined'
  readonly common: ReservationPriceCommon
  readonly determinedDate: string
  readonly determinedAmount: number
}

export type EstimatePriceAppraisal = UndeterminedReservationPrice | DeterminedReservationPrice

export interface BeforeReservationPriceCase extends SalesCaseCommon {
  readonly status: 'before_reservation_price'
}

export interface EstimateAppraisedCase extends SalesCaseCommon {
  readonly status: 'estimate_appraised'
  readonly appraisal: EstimatePriceAppraisal
}

export interface EstimateDeterminedCase extends SalesCaseCommon {
  readonly status: 'estimate_determined'
  readonly appraisal: EstimatePriceAppraisal
  readonly determinedDate: string
}

export interface EstimateDeliveredCase extends SalesCaseCommon {
  readonly status: 'estimate_delivered'
  readonly appraisal: EstimatePriceAppraisal
  readonly determinedDate: string
  readonly deliveryDate: string
}

export type ReservationSalesCase =
  | BeforeReservationPriceCase
  | EstimateAppraisedCase
  | EstimateDeterminedCase
  | EstimateDeliveredCase
```

### src/types/consignment_case.ts

```typescript
import type { SalesCaseCommon } from '@/types/sales_case'

export interface ConsignorInfo {
  readonly consignorCode: string
  readonly consignorName: string
}

export interface ConsignmentResult {
  readonly resultAmount: number
  readonly resultDate: string
}

export interface BeforeConsignmentCase extends SalesCaseCommon {
  readonly status: 'before_consignment'
}

export interface ConsignmentDesignatedCase extends SalesCaseCommon {
  readonly status: 'consignment_designated'
  readonly consignorInfo: ConsignorInfo
}

export interface ConsignmentResultEnteredCase extends SalesCaseCommon {
  readonly status: 'consignment_result_entered'
  readonly consignorInfo: ConsignorInfo
  readonly result: ConsignmentResult
}

export type ConsignmentSalesCase =
  | BeforeConsignmentCase
  | ConsignmentDesignatedCase
  | ConsignmentResultEnteredCase
```

### src/types/all_cases.ts

```typescript
import type { DirectSalesCase } from '@/types/sales_case'
import type { ReservationSalesCase } from '@/types/reservation_case'
import type { ConsignmentSalesCase } from '@/types/consignment_case'

export type SalesCase = DirectSalesCase | ReservationSalesCase | ConsignmentSalesCase
```

---

## ポイント

- `SalesCase` が全種別を統合するトップレベルの型。「予約販売案件に販売契約を締結する」操作が型レベルで不可能
- 予約価格は確定前/確定後で型が分かれる（`kind` で discriminate）
- `status` フィールドが全種別で一意のため、`switch/match` で全分岐を網羅できる

---

## 次のステップ

Step 16が完了したら [Step 17: マイグレーション追加（予約・委託テーブル）](./step17.md) へ進む。
