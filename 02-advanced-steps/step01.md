# Step 1: 設定管理 + 構造化ロギング

## 目的

### これは何か

アプリケーションの設定（DB接続先、ポート番号等）を環境ごとに切り替えられるようにし、ログ出力を構造化JSON形式に変更する。さらに、全てのHTTPリクエストに一意のリクエストIDを付与し、ログで追跡できるようにする。

### なぜやるのか

- Step 7までの実装では、DB接続文字列やポート番号がコード内にハードコードされている。本番・ステージング・開発で接続先を変えるには、設定を外部化する必要がある
- テキスト形式のログは人間には読みやすいが、CloudWatch Logs Insights等のツールで検索・集計するには構造化JSON形式が必要
- 障害調査時に「このリクエストに関連するログだけ」を抽出するには、リクエストIDでフィルタできる必要がある

### 何がうれしいのか

- 環境変数を変えるだけで、同じコードが開発・ステージング・本番で動く
- ログが `{"timestamp":"...","level":"Information","requestId":"abc-123","message":"..."}` のようなJSON形式になり、ログ検索ツールで `requestId = "abc-123"` と検索するだけで関連ログが全て見つかる
- 「このエラーはどのリクエストで起きた？」「このユーザーの操作履歴は？」といった調査が格段に速くなる

## 完了条件

### 設定管理の確認

以下の観点で動作を確認する:

1. 設定ファイルにDB接続文字列やポート番号が定義されていること
   - F#: `appsettings.json` + `appsettings.Development.json`
   - Kotlin: `application.conf`
2. 環境変数で設定を上書きできること
   - 例: `DATABASE_URL=postgres://other-host/db dotnet run` で接続先が変わる
   - 例: `PORT=9090 gradle run` でポートが変わる
3. コード内にハードコードされた接続文字列やポート番号が残っていないこと

### 構造化ロギングの確認

1. アプリを起動し、任意のAPIエンドポイント（例: `GET /lots/{id}`）にリクエストを送る
2. コンソールに出力されるログがJSON形式であること
3. ログに以下のフィールドが含まれていること:
   - `timestamp` — いつ
   - `level` — ログレベル（Information, Warning, Error等）
   - `message` — 何が起きたか
   - `requestId`（または `traceId`）— どのリクエストか
4. 同じリクエストに対する複数のログ行が、同一の `requestId` を持つこと
5. 異なるリクエストには異なる `requestId` が付与されること

### 確認のコツ

```bash
# 2つのリクエストを連続で送り、ログを見比べる
curl http://localhost:8080/lots/2024-A-001
curl http://localhost:8080/lots/2024-A-002

# ログ出力例（各行がJSON）:
# {"timestamp":"2026-04-22T10:00:00","level":"Information","requestId":"a1b2c3","message":"GET /lots/2024-A-001","elapsed":12}
# {"timestamp":"2026-04-22T10:00:01","level":"Information","requestId":"d4e5f6","message":"GET /lots/2024-A-002","elapsed":8}
# → requestId が異なることを確認
```

---

## 実装ガイド

### F#

| 要素 | ライブラリ/機能 |
|---|---|
| 設定ファイル | `appsettings.json`（ASP.NET Core 標準） |
| 環境別設定 | `appsettings.{ASPNETCORE_ENVIRONMENT}.json` |
| 環境変数オーバーライド | ASP.NET Core 標準（`__` 区切りで階層指定） |
| 構造化ログ | Serilog + `Serilog.Formatting.Compact` |
| リクエストID | `UseSerilogRequestLogging()` + `HttpContext.TraceIdentifier` |

主な作業:
1. `appsettings.json` に `Database.ConnectionString`, `Server.Port` 等を定義
2. NuGet で `Serilog.AspNetCore`, `Serilog.Formatting.Compact` を追加
3. `Program.fs` で `UseSerilog` と `UseSerilogRequestLogging` を設定
4. 既存のハードコードされた接続文字列を `IConfiguration` 経由に変更

### Kotlin

| 要素 | ライブラリ/機能 |
|---|---|
| 設定ファイル | `application.conf`（HOCON 形式、Ktor 標準） |
| 環境変数オーバーライド | HOCON `${?ENV_VAR}` 構文 |
| 構造化ログ | Logback + `logstash-logback-encoder` |
| リクエストID | Ktor `CallId` + `CallLogging` plugin |

主な作業:
1. `application.conf` に `database.url`, `ktor.deployment.port` 等を定義
2. Gradle で `net.logstash.logback:logstash-logback-encoder` を追加
3. `logback.xml` で `LogstashEncoder` を設定
4. `Application.kt` で `CallId` と `CallLogging` plugin をインストール
5. 既存のハードコードされた接続文字列を `environment.config` 経由に変更

---

## 次のステップ

Step 1が完了したら [Step 2: エラーハンドリング + バリデーション](./step02.md) へ進む。
