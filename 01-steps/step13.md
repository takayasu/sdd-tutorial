# Step 13: マイグレーション追加（販売案件・査定・契約テーブル）

## 目的

### これは何か

Step 12で追加した型定義に対応するデータベーステーブル（販売案件・価格査定・販売契約）をマイグレーションで追加する。

### なぜやるのか

- 新しいドメイン概念（販売案件等）のデータを保存するテーブルが必要
- Step 3で導入したマイグレーションの仕組みを使い、既存テーブルに影響を与えずに追加する
- マイグレーションファイルとして残すことで、「いつ何のテーブルを追加したか」が履歴として残る

### 何がうれしいのか

- コマンド1つで新しいテーブルが追加される。手動でSQLを実行する必要がない
- 既存のlotテーブルのデータはそのまま残る（安全な追加）
- チームの他のメンバーも同じコマンドで同じ状態に追いつける

## 完了条件

### F#（DbUp）

```bash
$ cd ../sales-management/apps/api-fsharp
$ dotnet run --project tools/Migrator
Beginning database upgrade
Executing Database Server script '002_create_sales_case_tables.sql'
Upgrade successful
マイグレーション完了

# テーブルが追加されたことを確認
$ docker compose exec db psql -U app -d sales_management -c "\dt"
              List of relations
 Schema |       Name        | Type  | Owner
--------+-------------------+-------+-------
 public | appraisal         | table | app
 public | contract          | table | app
 public | lot               | table | app
 public | lot_detail        | table | app
 public | sales_case        | table | app
 public | sales_case_lot    | table | app
 public | schemaversions    | table | app
(7 rows)
```

### Kotlin（Flyway）

```bash
$ cd kotlin
$ gradle flywayMigrate
> Task :flywayMigrate
Successfully applied 1 migration to schema "public"

BUILD SUCCESSFUL in Xs

# テーブルが追加されたことを確認
$ docker compose exec db psql -U app -d sales_management -c "\dt"
              List of relations
 Schema |         Name          | Type  | Owner
--------+-----------------------+-------+-------
 public | appraisal             | table | app
 public | contract              | table | app
 public | flyway_schema_history | table | app
 public | lot                   | table | app
 public | lot_detail            | table | app
 public | sales_case            | table | app
 public | sales_case_lot        | table | app
(7 rows)
```

---

## マイグレーションファイル

### F#（DbUp）

```sql
-- fsharp/migrations/002_create_sales_case_tables.sql

CREATE TABLE sales_case (
    sales_case_number_year  INTEGER NOT NULL,
    sales_case_number_month INTEGER NOT NULL,
    sales_case_number_seq   INTEGER NOT NULL,
    division_code           INTEGER NOT NULL,
    sales_date              DATE    NOT NULL,
    case_type               TEXT    NOT NULL DEFAULT 'direct',
    status                  TEXT    NOT NULL DEFAULT 'before_appraisal',
    PRIMARY KEY (sales_case_number_year, sales_case_number_month, sales_case_number_seq)
);

-- 販売案件とロットの紐付け
CREATE TABLE sales_case_lot (
    sales_case_number_year  INTEGER NOT NULL,
    sales_case_number_month INTEGER NOT NULL,
    sales_case_number_seq   INTEGER NOT NULL,
    lot_number_year         INTEGER NOT NULL,
    lot_number_location     TEXT    NOT NULL,
    lot_number_seq          INTEGER NOT NULL,
    PRIMARY KEY (sales_case_number_year, sales_case_number_month, sales_case_number_seq,
                 lot_number_year, lot_number_location, lot_number_seq),
    FOREIGN KEY (sales_case_number_year, sales_case_number_month, sales_case_number_seq)
        REFERENCES sales_case (sales_case_number_year, sales_case_number_month, sales_case_number_seq),
    FOREIGN KEY (lot_number_year, lot_number_location, lot_number_seq)
        REFERENCES lot (lot_number_year, lot_number_location, lot_number_seq)
);

CREATE TABLE appraisal (
    appraisal_number_year   INTEGER NOT NULL,
    appraisal_number_month  INTEGER NOT NULL,
    appraisal_number_seq    INTEGER NOT NULL,
    sales_case_number_year  INTEGER NOT NULL,
    sales_case_number_month INTEGER NOT NULL,
    sales_case_number_seq   INTEGER NOT NULL,
    appraisal_type          TEXT    NOT NULL DEFAULT 'normal',
    appraisal_date          DATE    NOT NULL,
    delivery_date           DATE    NOT NULL,
    sales_market            TEXT    NOT NULL,
    base_unit_price_date    TEXT    NOT NULL,
    period_adjustment_rate_date TEXT NOT NULL,
    counterparty_adjustment_rate_date TEXT NOT NULL,
    tax_excluded_estimated_total INTEGER NOT NULL,
    sales_agreement_number  TEXT,
    reservation_premium_rate NUMERIC,
    PRIMARY KEY (appraisal_number_year, appraisal_number_month, appraisal_number_seq),
    FOREIGN KEY (sales_case_number_year, sales_case_number_month, sales_case_number_seq)
        REFERENCES sales_case (sales_case_number_year, sales_case_number_month, sales_case_number_seq)
);

CREATE TABLE contract (
    contract_number_year    INTEGER NOT NULL,
    contract_number_month   INTEGER NOT NULL,
    contract_number_seq     INTEGER NOT NULL,
    appraisal_number_year   INTEGER NOT NULL,
    appraisal_number_month  INTEGER NOT NULL,
    appraisal_number_seq    INTEGER NOT NULL,
    contract_date           DATE    NOT NULL,
    person                  TEXT    NOT NULL,
    customer_number         TEXT    NOT NULL,
    agent_name              TEXT,
    sales_type              INTEGER NOT NULL,
    item                    TEXT    NOT NULL,
    delivery_method         TEXT    NOT NULL,
    sales_method            INTEGER NOT NULL,
    tax_excluded_contract_amount INTEGER NOT NULL,
    consumption_tax         INTEGER NOT NULL,
    tax_excluded_payment_amount INTEGER NOT NULL,
    payment_consumption_tax INTEGER NOT NULL,
    PRIMARY KEY (contract_number_year, contract_number_month, contract_number_seq),
    FOREIGN KEY (appraisal_number_year, appraisal_number_month, appraisal_number_seq)
        REFERENCES appraisal (appraisal_number_year, appraisal_number_month, appraisal_number_seq)
);
```

### Kotlin（Flyway）

```sql
-- kotlin/src/main/resources/db/migration/V002__create_sales_case_tables.sql
-- （内容はF#と同一）
```

---

## 実行

```bash
# F#
cd ../sales-management/apps/api-fsharp
dotnet run --project tools/Migrator

# Kotlin
cd kotlin
gradle flywayMigrate
```

---

## 確認

```bash
# テーブルが作成されたことを確認
docker compose exec db psql -U app -d sales_management -c "\dt"
```

---

## 次のステップ

Step 13が完了したら [Step 14: 直接販売案件のライフサイクルAPI実装](./step14.md) へ進む。
