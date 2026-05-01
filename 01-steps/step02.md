# Step 2: docker-compose構築（PostgreSQL）

## 目的

### これは何か

Docker Compose（複数のコンテナをまとめて管理するツール）を使って、PostgreSQL（データベース）を起動する。

### なぜやるのか

- アプリケーションのデータを永続化するためにデータベースが必要
- Docker Composeを使うことで、`docker compose up -d` の1コマンドでDBが起動する。手動でインストール・設定する手間がなくなる
- チーム全員が同じDB環境を再現できる

### 何がうれしいのか

- 「環境構築に半日かかる」がなくなる。コマンド1つで同じ環境が手に入る
- PCを壊しても、`docker compose up -d` で元通り
- CIでも同じコマンドでテスト用DBを立てられる

## 完了条件

```bash
# PostgreSQL起動
$ docker compose up -d
[+] Running 2/2
 ✔ Network fsharp_default  Created
 ✔ Container fsharp-db-1   Started

# コンテナが起動していることを確認
$ docker compose ps
NAME           SERVICE   STATUS    PORTS
fsharp-db-1    db        running   0.0.0.0:5432->5432/tcp

# DBに接続できることを確認
$ docker compose exec db psql -U app -d sales_management -c "SELECT 1"
 ?column?
----------
        1
(1 row)
```

---

## F#

### 1. docker-compose.yml

```yaml
# fsharp/docker-compose.yml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: sales_management
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

### 2. NuGetパッケージ追加

```bash
cd ../sales-management/apps/api-fsharp/src/SalesManagement
dotnet add package Npgsql
dotnet add package Donald
```

### 3. 接続確認コード（Program.fsに一時追加）

```fsharp
open Npgsql
open Donald

let connectionString = "Host=localhost;Port=5432;Database=sales_management;Username=app;Password=app"

let testConnection () =
    use conn = new NpgsqlConnection(connectionString)
    conn.Open()
    conn
    |> Db.newCommand "SELECT 1"
    |> Db.scalar (fun rd -> rd.GetInt32(0))
    |> printfn "DB接続確認: %d"
```

### 4. 起動・確認

```bash
cd ../sales-management/apps/api-fsharp
docker compose up -d
# 接続確認
docker compose exec db psql -U app -d sales_management -c "SELECT 1"
```

---

## Kotlin

### 1. docker-compose.yml

```yaml
# kotlin/docker-compose.yml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: sales_management
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

### 2. 依存関係追加（build.gradle.kts）

```kotlin
dependencies {
    // 既存の依存関係に追加
    implementation("org.jetbrains.exposed:exposed-core:0.52.0")
    implementation("org.jetbrains.exposed:exposed-dao:0.52.0")
    implementation("org.jetbrains.exposed:exposed-jdbc:0.52.0")
    implementation("org.postgresql:postgresql:42.7.3")
}
```

### 3. 接続確認

```kotlin
import org.jetbrains.exposed.sql.*
import org.jetbrains.exposed.sql.transactions.transaction

fun testConnection() {
    Database.connect(
        url = "jdbc:postgresql://localhost:5432/sales_management",
        driver = "org.postgresql.Driver",
        user = "app",
        password = "app"
    )
    transaction {
        exec("SELECT 1") { rs ->
            rs.next()
            println("DB接続確認: ${rs.getInt(1)}")
        }
    }
}
```

### 4. 起動・確認

```bash
cd kotlin
docker compose up -d
docker compose exec db psql -U app -d sales_management -c "SELECT 1"
```

---

## 次のステップ

Step 2が完了したら [Step 3: マイグレーション導入](./step03.md) へ進む。
