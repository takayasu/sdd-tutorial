# Step 12: 直接販売案件 + 価格査定 + 販売契約の型定義

## 目的

### これは何か

`domain-model-section2.md` の DSL を Python frozen dataclass と TypeScript discriminated union に変換する。Step 6 では在庫ロットだけだったが、ここでは「直接販売案件」「価格査定」「販売契約」という、より複雑なビジネス概念を型で表現する。

### なぜやるのか

- 直接販売案件は5段階の状態（査定前→査定済み→契約済み→出荷指示→出荷完了）を持ち、各段階で保持するデータが累積的に増える
- 「査定前の案件に契約を締結する」といった業務上ありえない操作を、型レベルで防ぐ
- Step 6 で学んだ「状態を型で表現する」パターンを、より複雑なドメインに適用する

### 何がうれしいのか

- ビジネスルールの複雑さが増しても、型が正しさを保証してくれる
- 既存の CI（フォーマッター、リンター、PBT）がそのまま動くことで、「新しいコードを追加しても既存が壊れていない」ことが自動確認される

## 完了条件

```bash
# Python: 型チェック通過
$ cd backend && mypy src/domain/sales_case.py --strict
Success: no issues found in 1 source file

# Python: リントチェック通過
$ ruff check src/domain/sales_case.py
All checks passed!

# TypeScript: 型チェック通過
$ cd frontend && npx tsc --noEmit
（エラーなし）
```

---

## Python 型定義

### src/domain/sales_case.py

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import date
from typing import Literal

from src.domain.lot import Amount, InventoryLot, LotCommon


# ---------- 識別番号 ----------

@dataclass(frozen=True)
class SalesCaseNumber:
    year: int
    month: int
    seq: int

    def __post_init__(self) -> None:
        if not (2000 <= self.year <= 2099):
            raise ValueError(f"year must be 2000–2099, got {self.year}")
        if not (1 <= self.month <= 12):
            raise ValueError(f"month must be 1–12, got {self.month}")
        if self.seq < 1:
            raise ValueError(f"seq must be >= 1, got {self.seq}")


@dataclass(frozen=True)
class AppraisalNumber:
    year: int
    month: int
    seq: int


@dataclass(frozen=True)
class ContractNumber:
    year: int
    month: int
    seq: int


# ---------- 価格査定 ----------

@dataclass(frozen=True)
class AppraisalCommon:
    appraisal_number: AppraisalNumber
    appraisal_date: date
    delivery_date: date
    sales_market: str
    base_unit_price_date: str
    period_adjustment_rate_date: str
    counterparty_adjustment_rate_date: str
    tax_excluded_estimated_total: Amount
    lot_appraisals: tuple[LotCommon, ...]  # 1件以上

    def __post_init__(self) -> None:
        if len(self.lot_appraisals) < 1:
            raise ValueError("lot_appraisals must have at least 1 item")


@dataclass(frozen=True)
class NormalAppraisal:
    common: AppraisalCommon
    kind: Literal["normal"] = "normal"

    def __post_init__(self) -> None:
        object.__setattr__(self, "kind", "normal")


@dataclass(frozen=True)
class AgreementAppraisal:
    common: AppraisalCommon
    customer_contract_number: str
    contract_adjustment_rate: float
    kind: Literal["agreement"] = "agreement"

    def __post_init__(self) -> None:
        object.__setattr__(self, "kind", "agreement")


PriceAppraisal = NormalAppraisal | AgreementAppraisal


# ---------- 販売契約 ----------

@dataclass(frozen=True)
class Buyer:
    customer_number: str
    agent_name: str | None = None


@dataclass(frozen=True)
class SalesInfo:
    sales_type: int
    item: str
    delivery_method: str
    sales_method: int
    payment_deferral_condition: str | None = None
    reason: str | None = None
    usage: str | None = None
    payment_deferral_amount: Amount | None = None


@dataclass(frozen=True)
class SalesPriceInfo:
    tax_excluded_contract_amount_taxable: Amount
    consumption_tax: Amount
    tax_excluded_payment_amount: Amount
    payment_consumption_tax: Amount


@dataclass(frozen=True)
class SalesContract:
    contract_number: ContractNumber
    contract_date: date
    person: str
    buyer: Buyer
    sales_info: SalesInfo
    sales_price_info: SalesPriceInfo
    appraisal_number: AppraisalNumber


# ---------- 直接販売案件（5段階の状態） ----------

@dataclass(frozen=True)
class SalesCaseCommon:
    sales_case_number: SalesCaseNumber
    division_code: int
    sales_date: date
    lots: tuple[InventoryLot, ...]  # 1件以上

    def __post_init__(self) -> None:
        if len(self.lots) < 1:
            raise ValueError("lots must have at least 1 item")


@dataclass(frozen=True)
class BeforeAppraisalCase:
    """査定前"""
    common: SalesCaseCommon
    status: Literal["before_appraisal"] = "before_appraisal"

    def __post_init__(self) -> None:
        object.__setattr__(self, "status", "before_appraisal")


@dataclass(frozen=True)
class AppraisedCase:
    """査定済み"""
    common: SalesCaseCommon
    appraisal: PriceAppraisal
    status: Literal["appraised"] = "appraised"

    def __post_init__(self) -> None:
        object.__setattr__(self, "status", "appraised")


@dataclass(frozen=True)
class ContractedCase:
    """契約済み"""
    common: SalesCaseCommon
    appraisal: PriceAppraisal
    contract: SalesContract
    status: Literal["contracted"] = "contracted"

    def __post_init__(self) -> None:
        object.__setattr__(self, "status", "contracted")


@dataclass(frozen=True)
class ShippingInstructedCase:
    """出荷指示済み"""
    common: SalesCaseCommon
    appraisal: PriceAppraisal
    contract: SalesContract
    shipping_instruction_date: date
    status: Literal["shipping_instructed"] = "shipping_instructed"

    def __post_init__(self) -> None:
        object.__setattr__(self, "status", "shipping_instructed")


@dataclass(frozen=True)
class ShippingCompletedCase:
    """出荷完了"""
    common: SalesCaseCommon
    appraisal: PriceAppraisal
    contract: SalesContract
    shipping_instruction_date: date
    shipping_completed_date: date
    status: Literal["shipping_completed"] = "shipping_completed"

    def __post_init__(self) -> None:
        object.__setattr__(self, "status", "shipping_completed")


DirectSalesCase = (
    BeforeAppraisalCase
    | AppraisedCase
    | ContractedCase
    | ShippingInstructedCase
    | ShippingCompletedCase
)
```

---

## TypeScript 型定義

### src/types/sales_case.ts

```typescript
import type { InventoryLot } from '@/types/lot'

// ---------- 識別番号 ----------

export interface SalesCaseNumber {
  readonly year: number
  readonly month: number
  readonly seq: number
}

export interface AppraisalNumber {
  readonly year: number
  readonly month: number
  readonly seq: number
}

export interface ContractNumber {
  readonly year: number
  readonly month: number
  readonly seq: number
}

// ---------- 価格査定 ----------

export interface AppraisalCommon {
  readonly appraisalNumber: AppraisalNumber
  readonly appraisalDate: string
  readonly deliveryDate: string
  readonly salesMarket: string
  readonly baseUnitPriceDate: string
  readonly periodAdjustmentRateDate: string
  readonly counterpartyAdjustmentRateDate: string
  readonly taxExcludedEstimatedTotal: number
  readonly lotAppraisals: readonly InventoryLot[]
}

export interface NormalAppraisal {
  readonly kind: 'normal'
  readonly common: AppraisalCommon
}

export interface AgreementAppraisal {
  readonly kind: 'agreement'
  readonly common: AppraisalCommon
  readonly customerContractNumber: string
  readonly contractAdjustmentRate: number
}

export type PriceAppraisal = NormalAppraisal | AgreementAppraisal

// ---------- 販売契約 ----------

export interface Buyer {
  readonly customerNumber: string
  readonly agentName?: string
}

export interface SalesInfo {
  readonly salesType: number
  readonly item: string
  readonly deliveryMethod: string
  readonly salesMethod: number
  readonly paymentDeferralCondition?: string
  readonly reason?: string
  readonly usage?: string
  readonly paymentDeferralAmount?: number
}

export interface SalesPriceInfo {
  readonly taxExcludedContractAmountTaxable: number
  readonly consumptionTax: number
  readonly taxExcludedPaymentAmount: number
  readonly paymentConsumptionTax: number
}

export interface SalesContract {
  readonly contractNumber: ContractNumber
  readonly contractDate: string
  readonly person: string
  readonly buyer: Buyer
  readonly salesInfo: SalesInfo
  readonly salesPriceInfo: SalesPriceInfo
  readonly appraisalNumber: AppraisalNumber
}

// ---------- 直接販売案件（5段階の状態） ----------

export interface SalesCaseCommon {
  readonly salesCaseNumber: SalesCaseNumber
  readonly divisionCode: number
  readonly salesDate: string
  readonly lots: readonly InventoryLot[]
}

export interface BeforeAppraisalCase extends SalesCaseCommon {
  readonly status: 'before_appraisal'
}

export interface AppraisedCase extends SalesCaseCommon {
  readonly status: 'appraised'
  readonly appraisal: PriceAppraisal
}

export interface ContractedCase extends SalesCaseCommon {
  readonly status: 'contracted'
  readonly appraisal: PriceAppraisal
  readonly contract: SalesContract
}

export interface ShippingInstructedCase extends SalesCaseCommon {
  readonly status: 'shipping_instructed'
  readonly appraisal: PriceAppraisal
  readonly contract: SalesContract
  readonly shippingInstructionDate: string
}

export interface ShippingCompletedCase extends SalesCaseCommon {
  readonly status: 'shipping_completed'
  readonly appraisal: PriceAppraisal
  readonly contract: SalesContract
  readonly shippingInstructionDate: string
  readonly shippingCompletedDate: string
}

export type DirectSalesCase =
  | BeforeAppraisalCase
  | AppraisedCase
  | ContractedCase
  | ShippingInstructedCase
  | ShippingCompletedCase
```

---

## ポイント

- 直接販売案件は5段階の状態を持ち、各段階で保持するデータが累積的に増える
- 価格査定は通常/顧客契約の2種類（`kind` で discriminate）で、顧客契約には追加フィールドがある
- 販売契約は `appraisal_number` を持つことで、どの査定に基づくかを追跡できる
- `lots` は 1件以上必須 → `__post_init__` で検証、TypeScript では `readonly` 配列

---

## 次のステップ

Step 12が完了したら [Step 13: マイグレーション追加](./step13.md) へ進む。
