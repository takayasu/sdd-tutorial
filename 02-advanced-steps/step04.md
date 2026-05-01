# Step 4: ヘルスチェック + OpenAPI

## 目的

### これは何か

アプリケーションとその依存サービス（DB等）の死活監視エンドポイントを実装し、APIの仕様書（OpenAPI / Swagger UI）を自動生成する。

### なぜやるのか

- k8s や ECS のヘルスチェック（liveness / readiness probe）は、アプリが「生きているか」「リクエストを受け付けられるか」を判定するためにHTTPエンドポイントを叩く。これがないとコンテナオーケストレータがアプリの異常を検知できない
- Spring Boot Actuator の `/health` に相当する機能を自前で実装する
- OpenAPI仕様書があると、フロントエンド開発者やAPI利用者が「どんなエンドポイントがあるか」「リクエスト/レスポンスの形式は何か」をブラウザで確認できる

### 何がうれしいのか

- ブラウザで `/health` を開くと、アプリとDBの状態が一目でわかる。DBが落ちていれば `"status": "DOWN"` と表示される
- ブラウザで `/swagger` を開くと、全APIの一覧が表示され、その場でリクエストを試せる。Postmanやcurlでの手動テストが不要になる
- k8sのreadinessProbeに `/health` を設定すれば、DB接続が切れたときに自動的にトラフィックが止まる

## 完了条件

### ヘルスチェックの確認

1. アプリとDBが正常な状態で `/health` を叩くと、全てUPであること:

```
GET /health
→ 200 OK

{
  "status": "UP",
  "checks": {
    "postgresql": "UP",
    "self": "UP"
  }
}
```

2. DBを停止した状態で `/health` を叩くと、ステータスがDOWNになること:

```bash
# DBを停止
docker compose stop db

# ヘルスチェック
GET /health
→ 503 Service Unavailable

{
  "status": "DOWN",
  "checks": {
    "postgresql": "DOWN",
    "self": "UP"
  }
}
```

3. DBを再起動すると、ヘルスチェックがUPに戻ること

```bash
docker compose start db
# 数秒待ってから
GET /health
→ 200 OK
```

### OpenAPIの確認

4. ブラウザで Swagger UI にアクセスできること:
   - F#: `http://localhost:5000/swagger`
   - Kotlin: `http://localhost:8080/swagger`

5. Swagger UI に以下が表示されていること:
   - 全てのAPIエンドポイント（GET /lots, POST /lots, POST /lots/{id}/complete-manufacturing 等）
   - リクエストボディのスキーマ（どんなJSONを送ればいいか）
   - レスポンスのスキーマ（どんなJSONが返ってくるか）
   - 認証が必要なエンドポイントには鍵マークが表示される

6. Swagger UI の「Try it out」ボタンでAPIを実際に叩けること

### 確認のコツ

- ヘルスチェックは認証不要にすること（k8sのprobeは認証トークンを持たない）
- `/health` のレスポンスタイムが遅い場合、DBへのSELECT 1が遅い可能性がある。コネクションプールの設定を確認する
- OpenAPIのJSON仕様は `/swagger/v1/swagger.json`（F#）や `/openapi.json`（Kotlin）で取得できる。これをフロントエンドのコード生成ツールに渡すこともできる

---

## 実装ガイド

### F#

| 要素 | 実装方法 |
|---|---|
| ヘルスチェック | ASP.NET Core `AddHealthChecks().AddNpgSql()` |
| Swagger UI | `Swashbuckle.AspNetCore` |
| OpenAPI生成 | `AddEndpointsApiExplorer` + `AddSwaggerGen` |

主な作業:
1. NuGet で `AspNetCore.HealthChecks.NpgSql`, `Swashbuckle.AspNetCore` を追加
2. `AddHealthChecks().AddNpgSql(connectionString)` でDB死活監視を登録
3. `MapHealthChecks("/health")` でエンドポイントを公開
4. `AddSwaggerGen` + `UseSwagger` + `UseSwaggerUI` を設定
5. ヘルスチェックとSwaggerは認証の外に配置

### Kotlin

| 要素 | 実装方法 |
|---|---|
| ヘルスチェック | 自前エンドポイント（DB接続チェック） |
| Swagger UI | Ktor OpenAPI plugin or Kompendium |
| メトリクス（任意） | `ktor-server-metrics-micrometer` + Prometheus |

主な作業:
1. `/health` エンドポイントを作成し、DB接続チェック（`SELECT 1`）を実行
2. ステータスに応じて200 or 503を返す
3. Ktor OpenAPI plugin を設定し、ルート定義にメタデータを付与
4. Swagger UIを `/swagger` で公開
5. ヘルスチェックとSwaggerは `authenticate` ブロックの外に配置

---

## 次のステップ

Step 4が完了したら [Step 5: 統合テスト + TestContainers](./step05.md) へ進む。
