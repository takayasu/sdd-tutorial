# Step 15: 価格査定・販売契約のAPI実装

## 目的

### これは何か

価格査定（通常査定/顧客契約査定）と販売契約の詳細なCRUD APIを実装する。Step 14ではライフサイクル遷移を実装したが、ここでは査定・契約の「中身」（ロット明細単位の査定情報等）に集中する。

### なぜやるのか

- 実際の業務では、価格査定はロットごと・明細ごとに単価や調整率が設定される。この複雑な構造をDBに正しく保存・取得できることを確認する
- 通常査定と顧客契約査定で必要なデータが異なる（顧客契約査定には顧客契約番号と契約調整率が追加）。この違いを型で表現し、混同を防ぐ

### 何がうれしいのか

- 「型で構造を定義 → AIがCRUDを生成 → PBTで検証 → CIで品質担保」というサイクルが、複雑なデータ構造に対しても機能することを確認できる
- ロット明細単位の査定情報まで正しく永続化できれば、実業務に近いレベルの実装が完成する

本ステップは Step 14 で確立した「集約API完全パッケージ」の規約 (詳細GET / 一覧GET / version 楽観ロック / problem+json / OpenAPI 完全記述 / URL 集約) をそのまま継承する。新規エンドポイントを足すたびに同じ完了条件を満たすこと。

## 完了条件

### (a) 動作要件

```bash
# 1. 価格査定を作成（URL は /sales-cases/{id}/direct/appraisals。type=normal|agreement）
$ curl -sf -X POST http://localhost:5000/sales-cases/2024-04-001/direct/appraisals \
  -H "Content-Type: application/json" \
  -d '{"type":"normal","appraisalDate":"2024-04-05","deliveryDate":"2024-05-01","salesMarket":"国内","taxExcludedEstimatedTotal":500000,"lotAppraisals":[...],"version":1}' \
  | jq -e '.appraisalNumber and (.type=="normal") and .version' >/dev/null

# 2. 販売契約を締結（査定済み案件に対して）
$ curl -sf -X POST http://localhost:5000/sales-cases/2024-04-001/direct/contracts \
  -H "Content-Type: application/json" \
  -d '{"contractDate":"2024-04-10","person":"担当太郎","buyer":{"customerNumber":"C001"},"version":2}' \
  | jq -e '.contractNumber and (.status=="contracted") and .version' >/dev/null

# 3. 査定 / 契約の version 楽観ロック → 古い version で 409 + problem+json
$ curl -s -o /dev/null -w "%{http_code} %{content_type}\n" \
    -X PUT http://localhost:5000/sales-cases/2024-04-001/direct/appraisals \
    -H "Content-Type: application/json" \
    -d '{"appraisalDate":"2024-04-06","version":1}'
409 application/problem+json

# 4. 査定済みでない案件への契約締結 → 400 + problem+json (`{ "error": ... }` ではない)
$ curl -s -o /dev/null -w "%{http_code} %{content_type}\n" \
    -X POST http://localhost:5000/sales-cases/2024-04-002/direct/contracts \
    -H "Content-Type: application/json" \
    -d '{"contractDate":"2024-04-15","version":1}'
400 application/problem+json

# 5. 詳細 GET に最新の査定/契約情報が含まれている
$ curl -sf http://localhost:5000/sales-cases/2024-04-001 \
  | jq -e '.caseType=="direct" and .appraisal and .contract' >/dev/null

# 6. ロット明細単位の査定情報が DB に保存されていること
$ docker compose exec db psql -U app -d sales_management \
  -c "SELECT COUNT(*) FROM lot_detail_appraisal WHERE appraisal_number_year = 2024"

# 7. PBTが通ること
$ dotnet test --filter "Category=PBT"
$ gradle test --tests "*PropertyTest*"

# 8. OpenAPI 完全記述: AppraisalResponse, ContractResponse, LotDetailAppraisal 等が存在
$ python3 -c "
import yaml
y = yaml.safe_load(open('openapi.yaml'))
required = ['AppraisalResponse','ContractResponse','LotAppraisal','LotDetailAppraisal']
missing = [s for s in required if s not in y['components']['schemas']]
assert not missing, f'missing: {missing}'
print('OK')
"
OK
```

### (b) `ci.sh` verify セクションへ追記

Step 14 までの PASS 群に加え:

```
PASS appraisal-create
PASS contract-create
PASS appraisal-version-conflict-409
PASS contract-precondition-400
PASS sales-case-detail-includes-appraisal-contract
PASS openapi-schemas-complete
```

---

## 実装の焦点

Step 14では販売案件のライフサイクル遷移を実装した。このステップでは、価格査定と販売契約の「中身」の詳細実装に集中する。

---

## 価格査定の構造

```
価格査定
├── 通常査定 or 顧客契約査定
├── 査定共通情報（番号、日付、市場、各種適用日号、税抜予定総額）
└── List<ロット価格査定>
    ├── ロット番号
    ├── 各種割増・調整率（オプション）
    └── List<ロット明細価格査定>
        ├── ロット明細
        ├── 基準単価
        ├── 期間調整率
        ├── 取引先調整率
        └── 特殊期間調整率（オプション）
```

---

## マイグレーション追加

```sql
-- migrations/003_create_appraisal_detail_tables.sql

CREATE TABLE lot_appraisal (
    appraisal_number_year   INTEGER NOT NULL,
    appraisal_number_month  INTEGER NOT NULL,
    appraisal_number_seq    INTEGER NOT NULL,
    lot_number_year         INTEGER NOT NULL,
    lot_number_location     TEXT    NOT NULL,
    lot_number_seq          INTEGER NOT NULL,
    equipment_cost          INTEGER,
    order_premium           NUMERIC,
    selection_premium       NUMERIC,
    reservation_premium     NUMERIC,
    adjustment_rate         NUMERIC,
    quality_adjustment_rate NUMERIC,
    manufacturing_cost_unit_price INTEGER,
    investment_recovery_period INTEGER,
    profit_rate             NUMERIC,
    PRIMARY KEY (appraisal_number_year, appraisal_number_month, appraisal_number_seq,
                 lot_number_year, lot_number_location, lot_number_seq),
    FOREIGN KEY (appraisal_number_year, appraisal_number_month, appraisal_number_seq)
        REFERENCES appraisal (appraisal_number_year, appraisal_number_month, appraisal_number_seq)
);

CREATE TABLE lot_detail_appraisal (
    appraisal_number_year   INTEGER NOT NULL,
    appraisal_number_month  INTEGER NOT NULL,
    appraisal_number_seq    INTEGER NOT NULL,
    lot_number_year         INTEGER NOT NULL,
    lot_number_location     TEXT    NOT NULL,
    lot_number_seq          INTEGER NOT NULL,
    detail_seq_no           INTEGER NOT NULL,
    base_unit_price         INTEGER NOT NULL,
    period_adjustment_rate  NUMERIC NOT NULL,
    counterparty_adjustment_rate  NUMERIC NOT NULL,
    special_period_adjustment_rate NUMERIC,
    PRIMARY KEY (appraisal_number_year, appraisal_number_month, appraisal_number_seq,
                 lot_number_year, lot_number_location, lot_number_seq, detail_seq_no),
    FOREIGN KEY (appraisal_number_year, appraisal_number_month, appraisal_number_seq,
                 lot_number_year, lot_number_location, lot_number_seq)
        REFERENCES lot_appraisal (appraisal_number_year, appraisal_number_month, appraisal_number_seq,
                                  lot_number_year, lot_number_location, lot_number_seq)
);
```

---

## PBT追加

```fsharp
// F#
testProperty "査定更新後も査定番号は変わらない" <|
    fun () ->
        let appraised = createAppraisedCase ()
        let updated = updateAppraisal newAppraisalInfo appraised
        updated.Appraisal.Common.AppraisalNumber = appraised.Appraisal.Common.AppraisalNumber

testProperty "顧客契約査定には顧客契約番号と契約調整率が必須" <|
    fun (agreementNumber: string) (rate: decimal) ->
        rate > 0m ==>
            let appraisal = createAgreementAppraisal agreementNumber rate
            match appraisal with
            | Agreement a -> a.CustomerContractNumber = agreementNumber && a.ContractAdjustmentRate = rate
            | _ -> false
```

```kotlin
// Kotlin
test("査定更新後も査定番号は変わらない") {
    forAll(appraisalCommonArb, appraisalInfoArb) { common, newInfo ->
        val original = PriceAppraisal.Normal(common)
        val updated = updateAppraisal(original, newInfo)
        updated.getOrNull()!!.common.appraisalNumber == common.appraisalNumber
    }
}
```

---

## 次のステップ

Step 15が完了したら [Step 16: 予約・委託販売案件の型定義](./step16.md) へ進む。
Phase 2はこれで完了。
