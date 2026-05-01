# Step 17: マイグレーション追加（予約・委託テーブル）

## 目的

### これは何か

Step 16で追加した型定義に対応するデータベーステーブル（予約査定・委託業者情報・委託販売結果）をマイグレーションで追加する。

### なぜやるのか

- 予約販売案件と委託販売案件のデータを保存するテーブルが必要
- Step 3, 13と同じパターン（SQLファイルを追加 → コマンド実行）で追加する

### 何がうれしいのか

- マイグレーションの仕組みに慣れてきた頃なので、スムーズに進められる
- 既存テーブル（lot, sales_case, appraisal, contract）に影響を与えずに追加できることを再確認できる

## 完了条件

```bash
# F#
$ cd ../sales-management/apps/api-fsharp
$ dotnet run --project tools/Migrator
Beginning database upgrade
Executing Database Server script '004_create_estimate_consignment_tables.sql'
Upgrade successful
マイグレーション完了

# Kotlin
$ cd kotlin
$ gradle flywayMigrate
> Task :flywayMigrate
Successfully applied 1 migration to schema "public"
BUILD SUCCESSFUL in Xs

# テーブルが追加されたことを確認
$ docker compose exec db psql -U app -d sales_management -c "\dt"
              List of relations
 Schema |         Name              | Type  | Owner
--------+---------------------------+-------+-------
 ...
 public | consignment_info          | table | app
 public | consignment_result        | table | app
 public | reservation_price        | table | app
 ...
```

---

## マイグレーションファイル

### F#（DbUp）

```sql
-- fsharp/migrations/004_create_estimate_consignment_tables.sql

-- 予約価格
CREATE TABLE reservation_price (
    appraisal_number_year   INTEGER NOT NULL,
    appraisal_number_month  INTEGER NOT NULL,
    appraisal_number_seq    INTEGER NOT NULL,
    sales_case_number_year  INTEGER NOT NULL,
    sales_case_number_month INTEGER NOT NULL,
    sales_case_number_seq   INTEGER NOT NULL,
    appraisal_date          DATE    NOT NULL,
    estimated_lot_info      TEXT    NOT NULL,
    estimated_amount        INTEGER NOT NULL,
    status                  TEXT    NOT NULL DEFAULT 'undetermined',
    determined_date         DATE,
    determined_amount       INTEGER,
    PRIMARY KEY (appraisal_number_year, appraisal_number_month, appraisal_number_seq),
    FOREIGN KEY (sales_case_number_year, sales_case_number_month, sales_case_number_seq)
        REFERENCES sales_case (sales_case_number_year, sales_case_number_month, sales_case_number_seq)
);

-- 委託業者情報
CREATE TABLE consignment_info (
    sales_case_number_year  INTEGER NOT NULL,
    sales_case_number_month INTEGER NOT NULL,
    sales_case_number_seq   INTEGER NOT NULL,
    consignor_name          TEXT    NOT NULL,
    consignor_code          TEXT    NOT NULL,
    designated_date         DATE    NOT NULL,
    PRIMARY KEY (sales_case_number_year, sales_case_number_month, sales_case_number_seq),
    FOREIGN KEY (sales_case_number_year, sales_case_number_month, sales_case_number_seq)
        REFERENCES sales_case (sales_case_number_year, sales_case_number_month, sales_case_number_seq)
);

-- 委託販売結果
CREATE TABLE consignment_result (
    sales_case_number_year  INTEGER NOT NULL,
    sales_case_number_month INTEGER NOT NULL,
    sales_case_number_seq   INTEGER NOT NULL,
    result_date             DATE    NOT NULL,
    result_amount           INTEGER NOT NULL,
    PRIMARY KEY (sales_case_number_year, sales_case_number_month, sales_case_number_seq),
    FOREIGN KEY (sales_case_number_year, sales_case_number_month, sales_case_number_seq)
        REFERENCES sales_case (sales_case_number_year, sales_case_number_month, sales_case_number_seq)
);

-- sales_caseテーブルにcase_typeの値として 'reservation', 'consignment' を追加可能にする
-- （既にTEXT型なので制約追加のみ）
ALTER TABLE sales_case ADD CONSTRAINT chk_case_type
    CHECK (case_type IN ('direct', 'reservation', 'consignment'));
```

### Kotlin（Flyway）

```sql
-- kotlin/src/main/resources/db/migration/V004__create_estimate_consignment_tables.sql
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
docker compose exec db psql -U app -d sales_management -c "\dt"
# reservation_price, consignment_info, consignment_result が追加されていること
```

---

## 次のステップ

Step 17が完了したら [Step 18: 予約・委託・品目変換のAPI実装](./step18.md) へ進む。
