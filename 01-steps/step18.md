# Step 18: 予約・委託・品目変換のAPI実装

## 目的

### これは何か

予約販売案件・委託販売案件・品目変換のライフサイクルを REST API として実装する。これにより、`domain-model-sales-management.md` に定義された全 23 個の behavior が API 化される。

### なぜやるのか

- ドメインモデル全体の実装を完成させる
- 予約と委託はそれぞれ独自のライフサイクルを持つ。直接販売案件とは異なるパターンを体験する

### 何がうれしいのか

- 全 behavior が実装され、PoC の機能面が完成する
- 「DSL → 型 → API → PBT → CI」のサイクルを3回繰り返したことで、パターンが完全に身につく

本ステップも Step 14 で確立した「集約API完全パッケージ」と「URL 集約規約 (`/sales-cases/{id}/{case_type}/...`)」をそのまま継承する。**`/reservation-cases/...` / `/consignment-cases/...` という subtype 別 URL は新設しない**。

## 完了条件

```bash
# 1. 予約販売案件の査定作成
$ curl -sf -X POST http://localhost:8000/sales-cases/2/reservation/appraisals \
  -H "Content-Type: application/json" \
  -d '{"appraisal_date":"2024-04-05","estimated_lot_info":"A商品 10本","estimated_amount":300000,"version":1}'
{"status":"estimate_appraised","version":2}

# 2. 委託販売案件の指定
$ curl -sf -X POST http://localhost:8000/sales-cases/3/consignment/designate \
  -H "Content-Type: application/json" \
  -d '{"consignor_name":"委託業者A","consignor_code":"CN001","designated_date":"2024-04-01","version":1}'
{"status":"consignment_designated"}

# 3. 委託指定解除（往復性）
$ curl -sf -X DELETE http://localhost:8000/sales-cases/3/consignment/designation \
  -H "Content-Type: application/json" -d '{"version":2}'
{"status":"before_consignment"}

# 4. 予約案件への販売契約締結 → URL が存在しないので 404
$ curl -s -o /dev/null -w "%{http_code}" \
  -X POST http://localhost:8000/sales-cases/2/reservation/contracts
404

# ci.sh が緑
$ ./ci.sh
PASS reservation-appraisal-create
PASS consignment-designate
PASS consignment-designation-reverse
PASS reservation-contract-impossible-404
```

---

## ドメインワークフロー

### src/domain/reservation_case_workflows.py

```python
from __future__ import annotations

from datetime import date

from src.domain.reservation_case import (
    BeforeReservationPriceCase,
    EstimateAppraisedCase,
    EstimateDeterminedCase,
    EstimateDeliveredCase,
    ReservationPriceCommon,
    UndeterminedReservationPrice,
)


def create_reservation_price(
    case: BeforeReservationPriceCase,
    info: ReservationPriceCommon,
) -> EstimateAppraisedCase:
    return EstimateAppraisedCase(
        common=case.common,
        appraisal=UndeterminedReservationPrice(common=info),
    )


def determine_estimate(
    case: EstimateAppraisedCase,
    determined_date: date,
    determined_amount: int,
) -> EstimateDeterminedCase:
    return EstimateDeterminedCase(
        common=case.common,
        appraisal=case.appraisal,
        determined_date=determined_date,
    )


def cancel_determination(case: EstimateDeterminedCase) -> EstimateAppraisedCase:
    return EstimateAppraisedCase(common=case.common, appraisal=case.appraisal)


def deliver_estimate(case: EstimateDeterminedCase, delivery_date: date) -> EstimateDeliveredCase:
    return EstimateDeliveredCase(
        common=case.common,
        appraisal=case.appraisal,
        determined_date=case.determined_date,
        delivery_date=delivery_date,
    )
```

### src/domain/consignment_case_workflows.py

```python
from __future__ import annotations

from src.domain.consignment_case import (
    BeforeConsignmentCase,
    ConsignmentDesignatedCase,
    ConsignmentResult,
    ConsignmentResultEnteredCase,
    ConsignorInfo,
)


def designate_consignment(
    case: BeforeConsignmentCase,
    consignor_info: ConsignorInfo,
) -> ConsignmentDesignatedCase:
    return ConsignmentDesignatedCase(common=case.common, consignor_info=consignor_info)


def cancel_designation(case: ConsignmentDesignatedCase) -> BeforeConsignmentCase:
    return BeforeConsignmentCase(common=case.common)


def enter_consignment_result(
    case: ConsignmentDesignatedCase,
    result: ConsignmentResult,
) -> ConsignmentResultEnteredCase:
    return ConsignmentResultEnteredCase(
        common=case.common,
        consignor_info=case.consignor_info,
        result=result,
    )
```

---

## 実装するAPI エンドポイント一覧

### 予約販売案件（`/sales-cases/{id}/reservation/...`）

| メソッド | パス | 動作 |
|---|---|---|
| POST | `/sales-cases/{id}/reservation/appraisals` | 予約価格を作成する |
| POST | `/sales-cases/{id}/reservation/determine` | 予約を確定する |
| DELETE | `/sales-cases/{id}/reservation/determination` | 予約確定を取り消す |
| POST | `/sales-cases/{id}/reservation/delivery` | 納品を指示する |

### 委託販売案件（`/sales-cases/{id}/consignment/...`）

| メソッド | パス | 動作 |
|---|---|---|
| POST | `/sales-cases/{id}/consignment/designate` | 委託販売案件を指定する |
| DELETE | `/sales-cases/{id}/consignment/designation` | 委託販売案件指定を解除する |
| POST | `/sales-cases/{id}/consignment/result` | 委託販売結果を入力する |

### 品目変換

| メソッド | パス | 動作 |
|---|---|---|
| POST | `/lots/{id}/convert-item` | 品目変換を指示する |
| DELETE | `/lots/{id}/convert-item` | 品目変換指示を取り消す |

---

## PBT 追加（hypothesis）

```python
# tests/test_reservation_case_properties.py
from hypothesis import given
from src.domain.reservation_case_workflows import (
    determine_estimate,
    cancel_determination,
    designate_consignment,
    cancel_designation,
)


@given(common_strategy, reservation_price_strategy)
def test_determination_roundtrip(common, appraisal) -> None:
    """予約確定取消後は査定済み状態に戻る（往復性）"""
    from datetime import date

    appraised = EstimateAppraisedCase(common=common, appraisal=appraisal)
    determined = determine_estimate(appraised, date(2024, 5, 1), 100000)
    cancelled = cancel_determination(determined)
    assert cancelled.common == common


@given(common_strategy, consignor_strategy)
def test_designation_roundtrip(common, consignor_info) -> None:
    """委託指定解除後は指定前に戻る（往復性）"""
    before = BeforeConsignmentCase(common=common)
    designated = designate_consignment(before, consignor_info)
    cancelled = cancel_designation(designated)
    assert cancelled.common == common
```

---

## 全 behavior の実装確認チェックリスト

`domain-model-sales-management.md` の全 behavior が実装されていることを確認:

- [x] 製造完了を指示する（Step 7）
- [x] 出荷を指示する（Step 7）
- [x] 出荷完了を指示する（Step 7）
- [x] 販売案件を作成する（Step 14）
- [x] 販売案件を削除する（Step 14）
- [x] 価格査定を作成する（Step 14）
- [x] 価格査定を更新する（Step 15）
- [x] 価格査定を削除する（Step 14）
- [x] 販売契約を締結する（Step 14）
- [x] 販売契約を削除する（Step 14）
- [x] 出庫を指示する（Step 14）
- [x] 出庫完了を指示する（Step 14）
- [x] 出庫指示を取り消す（Step 14）
- [x] 予約価格を作成する（Step 18）
- [x] 予約を確定する（Step 18）
- [x] 予約確定を取り消す（Step 18）
- [x] 納品を指示する（Step 18）
- [x] 委託販売案件を指定する（Step 18）
- [x] 委託販売案件指定を解除する（Step 18）
- [x] 委託販売結果を入力する（Step 18）
- [x] 品目変換を指示する（Step 18）
- [x] 品目変換指示を取り消す（Step 18）

---

## 次のステップ

Step 18が完了したら [Step 19: 品質ダッシュボード構築](./step19.md) へ進む。
