# Step 1: Hello World API（+ 横断ミドルウェア + verify 機構）

## 目的

### これは何か

Webフレームワーク（Giraffe / Ktor）を使って、最小限のWeb APIプロジェクトを作成する。「Hello World」に相当する最初の一歩。

ただし「動くだけ」では終わらせず、**以降の全ステップが乗る土台**として 3 つを最初から組み込む:

1. **横断ミドルウェア** — CORS / `X-Content-Type-Options: nosniff` / `Cross-Origin-Resource-Policy: same-origin` / problem+json default
2. **`ci.sh` の verify セクション** — `./ci.sh` の最後で API を起動し curl 検証するブロック。各 step の完了条件はここに **永続的に curl 検証を追加する**ことを義務化する
3. **OpenAPI スケルトン** — `openapi.yaml` を最初から置き、`components.responses.{BadRequest,NotFound,Conflict}` を problem+json で定義しておく

### なぜやるのか

- いきなり複雑なドメインモデルを実装する前に、「プロジェクトを作ってビルドして動かす」という基本サイクルを体験する
- **横断ミドルウェアを後付けするとテストが大量に書き直しになる**。最初から入れる方がコストが低い
- **ralph で `[x]` をつけた後にコードが空のまま (false-positive completion) になるバグ**を構造的に防ぐため、完了条件を ci.sh に組み込む規約をここで導入する

### 何がうれしいのか

- 「自分の手でAPIサーバーを起動して、curlで叩いて応答が返ってくる」という成功体験が得られる
- 以降のステップで型定義やDB接続を追加していく土台ができる
- Step 20 (DAST) で発覚しがちなセキュリティヘッダ警告を**最初から踏まない**
- 各 step が「ci.sh に curl を追加した時点で次に進める」ルールになり、エージェント自走時の品質が安定する

## 完了条件

### F#

```bash
# ビルド
$ cd ../sales-management/apps/api-fsharp/src/SalesManagement
$ dotnet build
  SalesManagement -> /path/to/bin/Debug/net8.0/SalesManagement.dll
  Build succeeded.
      0 Warning(s)
      0 Error(s)

# 起動（別ターミナルで）
$ dotnet run
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://localhost:5000

# 1. /health が 200
$ curl -sf http://localhost:5000/health
OK

# 2. レスポンスに必須セキュリティヘッダがついている
$ curl -sI http://localhost:5000/health | grep -iE 'x-content-type-options|cross-origin-resource-policy'
X-Content-Type-Options: nosniff
Cross-Origin-Resource-Policy: same-origin

# 3. CORS preflight が許可オリジンに 204 を返す
$ curl -sI -X OPTIONS http://localhost:5000/health \
    -H "Origin: http://localhost:5173" \
    -H "Access-Control-Request-Method: GET" | head -1
HTTP/1.1 204 No Content

# 4. 存在しないルートが problem+json で 404 を返す
$ curl -s -o /dev/null -w "%{http_code} %{content_type}\n" http://localhost:5000/no-such-route
404 application/problem+json

# 5. ./ci.sh の verify セクションが exit 0
$ ./ci.sh
=== verify (smoke) ===
PASS /health
PASS security-headers
PASS cors-preflight
PASS problem+json-on-404
=== CI完了 ===
```

### Kotlin

```bash
# ビルド
$ cd kotlin
$ gradle build
BUILD SUCCESSFUL in Xs

# 起動（別ターミナルで）
$ gradle run
... Netty started on port(s): 8080

# F# と同じ verify 4 項目が通ること
$ curl -sf http://localhost:8080/health
OK
$ curl -sI http://localhost:8080/health | grep -iE 'x-content-type-options|cross-origin-resource-policy'
$ curl -s -o /dev/null -w "%{http_code} %{content_type}\n" http://localhost:8080/no-such-route
404 application/problem+json
```

---

## 横断ミドルウェアの方針（全ステップ共通の前提）

以降の全ステップは、ここで導入する 4 つのミドルウェアが入っている前提で書かれる。後付けすると Step 20 (DAST) で警告として出てくるが、最初から入れておけば踏まずに済む。

| ミドルウェア | 何のため | 失敗時の症状 |
|---|---|---|
| **CORS** (`AddCors` / Ktor `CORS`) | フロントを別ドメインに置くため | 本番で frontend → backend の XHR がブラウザにブロックされる |
| **`X-Content-Type-Options: nosniff`** | MIME sniffing 攻撃の抑止 | ZAP `[10021]` 警告 |
| **`Cross-Origin-Resource-Policy: same-origin`** | クロスオリジン読み込み制限 | ZAP `[90004]` 警告 |
| **problem+json デフォルト** (`AddProblemDetails` + `UseStatusCodePages`) | エラー形式統一 (RFC 9457) | Step 4 以降で「エラー形式が混在」と言われ書き直しになる |

許可オリジンは `appsettings.json` の `Cors:AllowedOrigins` 配列で設定可能にする (Kotlin は `application.conf`)。

## `ci.sh` の verify セクション規約（全ステップ共通の前提）

`./ci.sh` の末尾に **verify セクション** を設け、API を起動して curl で動作確認する。各 step の完了条件には以下を**必ず含める**:

> このステップで追加した完了条件 (curl 例) を `ci.sh` の verify セクションに `curl -sf ...` として追加し、`./ci.sh` が exit 0 で終わること。

これにより:
- ralph が step を `[x]` にしたあとも「ci.sh が落ちる」形で実装漏れが検出される (false-positive completion 防止)
- DAST (Step 20) と同じく「動いている API に対する自動検証」が早期から積み上がる

`ci.sh` の verify セクション雛形 (Step 1 時点):

```bash
echo "=== verify (smoke) ==="
# 1. API を起動
dotnet run --project src/SalesManagement &  # F#
APP_PID=$!
trap 'kill $APP_PID 2>/dev/null || true' EXIT
# /health が応答するまで待つ
for i in $(seq 1 30); do
  curl -sf http://localhost:5000/health >/dev/null && break
  sleep 1
done

# 2. このステップで導入された検証 (Step 1 では smoke 4 項目)
curl -sf http://localhost:5000/health >/dev/null \
  && echo "PASS /health"

curl -sI http://localhost:5000/health \
  | grep -iq '^x-content-type-options: nosniff' \
  && echo "PASS security-headers"

curl -sI -X OPTIONS http://localhost:5000/health \
  -H "Origin: http://localhost:5173" \
  -H "Access-Control-Request-Method: GET" \
  | head -1 | grep -q '204' \
  && echo "PASS cors-preflight"

curl -s -o /dev/null -w "%{content_type}" http://localhost:5000/no-such-route \
  | grep -q 'application/problem+json' \
  && echo "PASS problem+json-on-404"
```

各 step (Step 7 / 14 / 15 / 18 / 20 等) は、自分が追加した curl をこのブロックに **追記** していく。verify セクションは「実装の真の完成」を継続検証する場所になる。

## F#（Giraffe）

### 1. プロジェクト作成

```bash
mkdir -p fsharp/src/SalesManagement
cd ../sales-management/apps/api-fsharp/src/SalesManagement
dotnet new web -lang F#
dotnet add package Giraffe
```

### 2. Program.fs を編集

```fsharp
open Microsoft.AspNetCore.Builder
open Microsoft.AspNetCore.Cors.Infrastructure
open Microsoft.AspNetCore.Http
open Microsoft.Extensions.DependencyInjection
open Microsoft.Extensions.Hosting
open Giraffe

let webApp =
    choose [
        GET >=> route "/health" >=> text "OK"
        // 未マッチは problem+json で 404
        RequestErrors.notFound (
            negotiateWith
                [ "application/problem+json", json ]
                (Some <| Negotiation.DefaultNegotiator())
                {| ``type`` = "about:blank"
                   title = "Not Found"
                   status = 404 |})
    ]

[<EntryPoint>]
let main args =
    let builder = WebApplication.CreateBuilder(args)
    let allowed =
        builder.Configuration.GetSection("Cors:AllowedOrigins").Get<string[]>()
        |> Option.ofObj
        |> Option.defaultValue [| "http://localhost:5173" |]
    builder.Services.AddCors(fun o ->
        o.AddDefaultPolicy(fun p ->
            p.WithOrigins(allowed).AllowAnyHeader().AllowAnyMethod() |> ignore))
    |> ignore
    builder.Services.AddGiraffe() |> ignore
    builder.Services.AddProblemDetails() |> ignore
    let app = builder.Build()
    app.UseCors() |> ignore
    // 全レスポンスにセキュリティヘッダ
    app.Use(fun (ctx: HttpContext) (next: RequestDelegate) ->
        ctx.Response.OnStarting(fun () ->
            ctx.Response.Headers["X-Content-Type-Options"] <- "nosniff"
            ctx.Response.Headers["Cross-Origin-Resource-Policy"] <- "same-origin"
            System.Threading.Tasks.Task.CompletedTask)
        next.Invoke(ctx))
    |> ignore
    app.UseStatusCodePages() |> ignore
    app.UseGiraffe webApp
    app.Run()
    0
```

### 3. ビルド・実行

```bash
dotnet build
dotnet run
# 別ターミナルで確認
curl http://localhost:5000/health
```

---

## Kotlin（Ktor）

### 1. プロジェクト作成

```bash
mkdir -p kotlin/src/main/kotlin/salesmanagement
mkdir -p kotlin/src/main/resources
```

### 2. build.gradle.kts

```kotlin
plugins {
    kotlin("jvm") version "2.0.0"
    id("io.ktor.plugin") version "2.3.12"
}

group = "com.example"
version = "0.0.1"

application {
    mainClass.set("salesmanagement.ApplicationKt")
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("io.ktor:ktor-server-core-jvm")
    implementation("io.ktor:ktor-server-netty-jvm")
    implementation("ch.qos.logback:logback-classic:1.4.14")
}
```

### 3. settings.gradle.kts

```kotlin
rootProject.name = "sales-management"
```

### 4. Application.kt

```kotlin
package salesmanagement

import io.ktor.http.*
import io.ktor.server.application.*
import io.ktor.server.engine.*
import io.ktor.server.netty.*
import io.ktor.server.plugins.cors.routing.*
import io.ktor.server.plugins.statuspages.*
import io.ktor.server.plugins.defaultheaders.*
import io.ktor.server.response.*
import io.ktor.server.routing.*

fun main() {
    embeddedServer(Netty, port = 8080) {
        install(CORS) {
            allowHost("localhost:5173", schemes = listOf("http", "https"))
            allowMethod(HttpMethod.Options)
            allowHeader(HttpHeaders.ContentType)
            allowHeader(HttpHeaders.Authorization)
        }
        install(DefaultHeaders) {
            header("X-Content-Type-Options", "nosniff")
            header("Cross-Origin-Resource-Policy", "same-origin")
        }
        install(StatusPages) {
            status(HttpStatusCode.NotFound) { call, status ->
                call.respondText(
                    """{"type":"about:blank","title":"Not Found","status":404}""",
                    ContentType.parse("application/problem+json"),
                    status,
                )
            }
        }
        routing {
            get("/health") {
                call.respondText("OK")
            }
        }
    }.start(wait = true)
}
```

### 5. ビルド・実行

```bash
gradle build
gradle run
# 別ターミナルで確認
curl http://localhost:8080/health
```

---

## OpenAPI スケルトン

`openapi.yaml` を最初から置く。Step 7 / 14 / 15 / 18 で paths を追記していく前提。
ここで `components.responses.{BadRequest, NotFound, Conflict}` を **problem+json** で先に定義しておくと、各ステップは `$ref` で参照するだけで済む。

```yaml
openapi: 3.0.3
info:
  title: Sales Management API
  version: 0.1.0
servers:
  - url: http://localhost:5000
paths:
  /health:
    get:
      operationId: health
      responses:
        '200':
          description: OK
          content:
            text/plain:
              schema: { type: string }
components:
  schemas:
    Problem:
      type: object
      required: [type, title, status]
      properties:
        type:    { type: string, format: uri }
        title:   { type: string }
        status:  { type: integer }
        detail:  { type: string }
        instance:{ type: string, format: uri }
  responses:
    BadRequest:
      description: 入力不正
      content:
        application/problem+json:
          schema: { $ref: '#/components/schemas/Problem' }
    NotFound:
      description: 対象が存在しない
      content:
        application/problem+json:
          schema: { $ref: '#/components/schemas/Problem' }
    Conflict:
      description: 楽観ロック競合
      content:
        application/problem+json:
          schema: { $ref: '#/components/schemas/Problem' }
```

## 次のステップ

Step 1が完了したら [Step 2: docker-compose構築](./step02.md) へ進む。
