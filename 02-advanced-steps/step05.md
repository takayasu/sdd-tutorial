# Step 5: 統合テスト + TestContainers

## 目的

### これは何か

TestContainersを使って、テスト実行時にPostgreSQLコンテナを自動起動し、APIの統合テスト（HTTPリクエスト → ドメインロジック → DB永続化 → HTTPレスポンス）をエンドツーエンドで検証する。

### なぜやるのか

- Step 8のPBTはドメインロジック（純粋関数）のテスト。DBやHTTPレイヤーは検証していない
- 統合テストは「APIを叩いて、DBに正しく保存され、正しいレスポンスが返る」ことを検証する。Spring Boot の `@SpringBootTest` + `MockMvc` に相当する
- TestContainersを使うと、テスト用のDBをDockerコンテナとして自動起動・自動破棄できる。`docker compose up` を手動で実行する必要がない

### 何がうれしいのか

- `dotnet test` / `gradle test` を実行するだけで、DBコンテナが自動起動し、マイグレーションが適用され、テストが実行され、コンテナが自動破棄される。手動のセットアップが一切不要
- CIでも同じコマンドで動く。「CIでDBが起動していない」という問題が起きない
- 「APIを叩いてDBを確認する」という手動テストを自動化できる。Step 7の完了条件で手動curlしていた内容がテストコードになる

## 完了条件

### テスト実行の確認

1. テストコマンドを実行すると、PostgreSQLコンテナが自動起動し、テストが通ること:

```bash
# F#
dotnet test --filter "Category=Integration"
# → Passed! - Failed: 0, Passed: N, Skipped: 0

# Kotlin
gradle test --tests "*IntegrationTest*"
# → BUILD SUCCESSFUL, N tests completed, 0 failed
```

2. テスト完了後、PostgreSQLコンテナが自動的に停止・削除されること（`docker ps` で残っていないこと）

### テストシナリオの確認

最低限、以下のシナリオがテストされていること:

3. ロット作成 → GET で取得できること（正常系エンドツーエンド）
4. ロット作成 → 製造完了 → DBのstatusが `manufactured` に更新されていること
5. 存在しないロットの取得 → 404 Problem Details が返ること
6. 不正な状態遷移 → 400 Problem Details が返ること
7. 認証トークンなし → 401が返ること（Step 3の認証が統合テストでも動くこと）

### 確認のコツ

- テスト実行中に別ターミナルで `docker ps` を叩くと、TestContainersが起動したPostgreSQLコンテナが見える。テスト完了後に消えることを確認する
- テストが失敗した場合、TestContainersのログ（コンテナの起動ログ）を確認する。ポートの競合やDockerデーモンの停止が原因のことが多い
- 各テストは独立して実行できること（テスト間でデータが干渉しない）。テストごとにトランザクションをロールバックするか、テストごとにDBをクリーンアップする

---

## 実装ガイド

### テストの構造

```
テスト起動
  → TestContainers: PostgreSQLコンテナ起動
  → マイグレーション適用（DbUp / Flyway）
  → テスト用APIサーバー起動（インメモリ、JWT検証をテスト用鍵に差し替え）
  → HTTPリクエスト送信 → レスポンス検証
  → テスト完了
  → コンテナ自動破棄
```

### 統合テストでの認証の扱い

Keycloak を TestContainers で起動する方法もあるが、起動に30秒以上かかり統合テストが遅くなる。代わりに、テスト内で自己署名JWTを生成し、アプリのJWT検証をテスト用の公開鍵に差し替える方式を推奨する:

1. テストセットアップで RSA 鍵ペアを生成する
2. アプリの JWT 検証設定を、Keycloak の JWKS ではなくテスト用の公開鍵を使うようにオーバーライドする
3. テスト内でロール付きの JWT を自己署名して `Authorization: Bearer` ヘッダに付与する

これにより、認証・認可のテスト（401/403の確認）が Keycloak なしで高速に実行できる。

### F#

| 要素 | 実装方法 |
|---|---|
| テスト用サーバー | `WebApplicationFactory<Program>` |
| HTTPクライアント | `factory.CreateClient()` |
| TestContainers | `Testcontainers.PostgreSql` (NuGet) |
| マイグレーション | テストセットアップ内で DbUp を実行 |

主な作業:
1. NuGet で `Testcontainers.PostgreSql`, `Microsoft.AspNetCore.Mvc.Testing` を追加
2. テストフィクスチャで PostgreSQLコンテナを起動し、接続文字列を取得
3. `WebApplicationFactory` をカスタマイズし、テスト用の接続文字列を注入
4. `factory.CreateClient()` でHTTPクライアントを取得し、APIを叩く
5. レスポンスのステータスコードとボディを検証

### Kotlin

| 要素 | 実装方法 |
|---|---|
| テスト用サーバー | Ktor `testApplication { }` |
| HTTPクライアント | `testApplication { client }` |
| TestContainers | `org.testcontainers:postgresql` (Gradle) |
| マイグレーション | テストセットアップ内で Flyway を実行 |

主な作業:
1. Gradle で `org.testcontainers:postgresql`, `org.testcontainers:junit-jupiter` を追加
2. テストクラスで `@Testcontainers` + `@Container` でPostgreSQLコンテナを定義
3. `testApplication { }` 内でテスト用の設定（DB接続先）を注入
4. `client.get/post` でAPIを叩き、レスポンスを検証

---

## 次のステップ

Step 5が完了したら [Step 6: HTTPクライアント + レジリエンス](./step06.md) へ進む。
