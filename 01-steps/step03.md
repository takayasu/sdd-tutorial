# Step 3: マイグレーション導入（DbUp / Flyway）

## 目的

### これは何か

マイグレーションツール（DbUp / Flyway）を導入する。マイグレーションとは、データベースのテーブル構造（スキーマ）の変更履歴をSQLファイルとして管理し、コマンド1つで適用する仕組み。

### なぜやるのか

- DBのテーブル構造を「いつ・誰が・何を変えたか」をGitで追跡できるようにする
- 手動でSQLを実行する運用だと、「本番にこのALTER TABLE当てた？」「開発環境と本番でテーブルが違う」といった事故が起きる
- CIでテスト用DBを毎回クリーンに構築するために必要

### 何がうれしいのか

- コマンド1つ（`dotnet run --project tools/Migrator` / `gradle flywayMigrate`）でDBが最新状態になる
- 新しいメンバーが参加しても、同じコマンドで同じDB構造が手に入る
- 2回目以降は「適用済み」としてスキップされるので、何度実行しても安全

## 完了条件

### F#（DbUp）

```bash
# マイグレーション実行（1回目）
$ cd ../sales-management/apps/api-fsharp
$ dotnet run --project tools/Migrator
Beginning database upgrade
Checking whether journal table exists..
Journal table does not exist
Executing Database Server script '001_create_lot_table.sql'
Upgrade successful
マイグレーション完了

# テーブルが作成されたことを確認
$ docker compose exec db psql -U app -d sales_management -c "\dt"
              List of relations
 Schema |    Name     | Type  | Owner
--------+-------------+-------+-------
 public | lot         | table | app
 public | lot_detail  | table | app
 public | schemaversions | table | app
(3 rows)

# マイグレーション実行（2回目 → スキップされる）
$ dotnet run --project tools/Migrator
Beginning database upgrade
No new scripts need to be executed - completing.
マイグレーション完了
```

### Kotlin（Flyway）

```bash
# マイグレーション実行（1回目）
$ cd kotlin
$ gradle flywayMigrate
> Task :flywayMigrate
Successfully applied 1 migration to schema "public" (execution time 00:00.042s)

BUILD SUCCESSFUL in Xs

# テーブルが作成されたことを確認
$ docker compose exec db psql -U app -d sales_management -c "\dt"
              List of relations
 Schema |         Name          | Type  | Owner
--------+-----------------------+-------+-------
 public | flyway_schema_history | table | app
 public | lot                   | table | app
 public | lot_detail            | table | app
(3 rows)

# マイグレーション実行（2回目 → スキップされる）
$ gradle flywayMigrate
> Task :flywayMigrate
Schema "public" is up to date. No migration necessary.

BUILD SUCCESSFUL in Xs
```

---

## F#（DbUp）

### 1. マイグレーション用プロジェクト作成

```bash
mkdir -p fsharp/tools/Migrator
cd ../sales-management/apps/api-fsharp/tools/Migrator
dotnet new console -lang F#
dotnet add package DbUp.PostgreSQL
```

### 2. Program.fs

```fsharp
open DbUp
open System

[<EntryPoint>]
let main args =
    let connectionString =
        match Environment.GetEnvironmentVariable("DATABASE_URL") with
        | null -> "Host=localhost;Port=5432;Database=sales_management;Username=app;Password=app"
        | url -> url

    let upgrader =
        DeployChanges.To
            .PostgresqlDatabase(connectionString)
            .WithScriptsFromFileSystem("migrations")
            .LogToConsole()
            .Build()

    let result = upgrader.PerformUpgrade()

    if result.Successful then
        printfn "マイグレーション完了"
        0
    else
        eprintfn "マイグレーション失敗: %s" (result.Error.ToString())
        1
```

### 3. 最初のマイグレーションファイル

```bash
mkdir -p fsharp/migrations
```

```sql
-- fsharp/migrations/001_create_lot_table.sql

CREATE TABLE lot (
    lot_number_year     INTEGER NOT NULL,
    lot_number_location TEXT    NOT NULL,
    lot_number_seq      INTEGER NOT NULL,
    division_code       INTEGER NOT NULL,
    department_code     INTEGER NOT NULL,
    section_code        INTEGER NOT NULL,
    process_category    INTEGER NOT NULL,
    inspection_category INTEGER NOT NULL,
    manufacturing_category INTEGER NOT NULL,
    status              TEXT    NOT NULL DEFAULT 'manufacturing',
    manufacturing_completed_date DATE,
    shipping_deadline_date       DATE,
    shipped_date                 DATE,
    PRIMARY KEY (lot_number_year, lot_number_location, lot_number_seq)
);

CREATE TABLE lot_detail (
    lot_number_year     INTEGER NOT NULL,
    lot_number_location TEXT    NOT NULL,
    lot_number_seq      INTEGER NOT NULL,
    seq_no              INTEGER NOT NULL,
    item_category       TEXT    NOT NULL,
    premium_category    TEXT,
    product_category_code TEXT NOT NULL,
    length_spec_lower   NUMERIC NOT NULL,
    thickness_spec_lower NUMERIC NOT NULL,
    thickness_spec_upper NUMERIC NOT NULL,
    quality_grade       TEXT    NOT NULL,
    quantity_count      INTEGER NOT NULL,
    quantity_amount     NUMERIC NOT NULL,
    pass_fail_category  TEXT,
    PRIMARY KEY (lot_number_year, lot_number_location, lot_number_seq, seq_no),
    FOREIGN KEY (lot_number_year, lot_number_location, lot_number_seq)
        REFERENCES lot (lot_number_year, lot_number_location, lot_number_seq)
);
```

### 4. 実行

```bash
cd ../sales-management/apps/api-fsharp
dotnet run --project tools/Migrator
```

---

## Kotlin（Flyway）

### 1. Gradle設定追加（build.gradle.kts）

```kotlin
plugins {
    // 既存のpluginsに追加
    id("org.flywaydb.flyway") version "10.15.0"
}

flyway {
    url = "jdbc:postgresql://localhost:5432/sales_management"
    user = "app"
    password = "app"
    locations = arrayOf("filesystem:src/main/resources/db/migration")
}
```

### 2. 最初のマイグレーションファイル

```bash
mkdir -p kotlin/src/main/resources/db/migration
```

```sql
-- kotlin/src/main/resources/db/migration/V001__create_lot_table.sql

CREATE TABLE lot (
    lot_number_year     INTEGER NOT NULL,
    lot_number_location TEXT    NOT NULL,
    lot_number_seq      INTEGER NOT NULL,
    division_code       INTEGER NOT NULL,
    department_code     INTEGER NOT NULL,
    section_code        INTEGER NOT NULL,
    process_category    INTEGER NOT NULL,
    inspection_category INTEGER NOT NULL,
    manufacturing_category INTEGER NOT NULL,
    status              TEXT    NOT NULL DEFAULT 'manufacturing',
    manufacturing_completed_date DATE,
    shipping_deadline_date       DATE,
    shipped_date                 DATE,
    PRIMARY KEY (lot_number_year, lot_number_location, lot_number_seq)
);

CREATE TABLE lot_detail (
    lot_number_year     INTEGER NOT NULL,
    lot_number_location TEXT    NOT NULL,
    lot_number_seq      INTEGER NOT NULL,
    seq_no              INTEGER NOT NULL,
    item_category       TEXT    NOT NULL,
    premium_category    TEXT,
    product_category_code TEXT NOT NULL,
    length_spec_lower   NUMERIC NOT NULL,
    thickness_spec_lower NUMERIC NOT NULL,
    thickness_spec_upper NUMERIC NOT NULL,
    quality_grade       TEXT    NOT NULL,
    quantity_count      INTEGER NOT NULL,
    quantity_amount     NUMERIC NOT NULL,
    pass_fail_category  TEXT,
    PRIMARY KEY (lot_number_year, lot_number_location, lot_number_seq, seq_no),
    FOREIGN KEY (lot_number_year, lot_number_location, lot_number_seq)
        REFERENCES lot (lot_number_year, lot_number_location, lot_number_seq)
);
```

### 3. 実行

```bash
cd kotlin
gradle flywayMigrate
```

---

## マイグレーションの命名規則

| 言語 | 形式 | 例 |
|---|---|---|
| F#（DbUp） | `NNN_説明.sql` | `001_create_lot_table.sql` |
| Kotlin（Flyway） | `VNNN__説明.sql` | `V001__create_lot_table.sql` |

DbUpはファイル名のアルファベット順で適用。Flywayはバージョン番号順で適用。

---

## 次のステップ

Step 3が完了したら [Step 4: フォーマッター導入](./step04.md) へ進む。
