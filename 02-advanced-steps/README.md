# Advanced Steps: Spring Boot 機能の代替実装

Spring Boot が業務システムに提供する機能（バッチ処理を除く）を、F#（Giraffe / ASP.NET Core）と Kotlin（Ktor）で代替する。01-steps/ を全て完了した前提で、業務システムとしての本番運用に耐えうるかを PoC する。

バッチ処理については [02-batch-steps/](../02-batch-steps/) を参照。

---

## 設計方針

- アプリの核心ロジックはクラウド非依存で自己完結させる
- Spring の「設定ゼロで全部つながる」自動設定は再現しない。各機能を明示的に組み込む
- 関数型スタイルとの相性を重視（暗黙的な AOP/プロキシより明示的な関数合成）

---

## 優先度定義

| 優先度 | 定義 |
|---|---|
| P0 | これがないと業務システムとして成立しない |
| P1 | 運用品質を担保するために強く推奨 |
| P2 | あると便利だが後回し可能 |

---

## 機能対応一覧

### P0（必須）

| 機能 | Spring Boot | F# (Giraffe / ASP.NET Core) | Kotlin (Ktor) | Step |
|---|---|---|---|---|
| 設定管理 | `application.yml` + Profile | `appsettings.json` + 環境変数 | `application.conf` (HOCON) | [Step 1](./step01.md) |
| ロギング | SLF4J + Logback | Serilog（構造化ログ） | SLF4J + Logback / kotlin-logging | [Step 1](./step01.md) |
| エラーハンドリング | `@ControllerAdvice` | Giraffe ErrorHandler + `Results.Problem()` | Ktor StatusPages plugin | [Step 2](./step02.md) |
| 入力バリデーション | Bean Validation | Smart Constructor + `Result` | Arrow `Either` / `Validated` | [Step 2](./step02.md) |
| 認証・認可 | Spring Security | ASP.NET Core Authentication + Authorization | ktor-server-auth + ktor-server-auth-jwt | [Step 3](./step03.md) |
| トランザクション管理 | `@Transactional` | Donald `dbTransaction` | Exposed `transaction { }` | 01-steps/ で実装済み |
| コネクションプール | HikariCP | Npgsql 内蔵プール | HikariCP | 01-steps/ で実装済み |

### P1（運用品質に強く推奨）

| 機能 | Spring Boot | F# | Kotlin | Step |
|---|---|---|---|---|
| ヘルスチェック | Actuator `/health` | ASP.NET Core HealthChecks | 自前エンドポイント | [Step 4](./step04.md) |
| API 仕様 | SpringDoc (OpenAPI) | Swashbuckle | Ktor OpenAPI / Kompendium | [Step 4](./step04.md) |
| テスト支援 | `@SpringBootTest` + MockMvc | `WebApplicationFactory` | Ktor `testApplication { }` | [Step 5](./step05.md) |
| HTTP クライアント | RestClient / WebClient | `HttpClient` + `IHttpClientFactory` | Ktor Client | [Step 6](./step06.md) |
| リトライ・レジリエンス | Spring Retry + Resilience4j | Polly | Resilience4j / Arrow Resilience | [Step 6](./step06.md) |
| 非同期・イベント | `@Async` + `ApplicationEvent` | `Async` CE + `MailboxProcessor` | Coroutines + `SharedFlow` | [Step 7](./step07.md) |
| 監査ログ | Spring Data Auditing | 自前（DB 層で共通化） | 自前（Exposed テーブル定義で共通化） | [Step 8](./step08.md) |
| 分散トレーシング | Micrometer Tracing | OpenTelemetry .NET | OpenTelemetry Java SDK | [Step 8](./step08.md) |
| レート制限 | Bucket4j / Spring Cloud Gateway | ASP.NET Core RateLimiter | Ktor RateLimit plugin / Bucket4j | [Step 9](./step09.md) |
| キャッシュ | `@Cacheable` | `IDistributedCache` / `MemoryCache` | Caffeine / Redis (Lettuce) | [Step 9](./step09.md) |
| グレースフルシャットダウン | `server.shutdown=graceful` | `IHostApplicationLifetime` | `ApplicationEngine.stop(grace, timeout)` | [Step 9](./step09.md) |
| ファイルエクスポート | CsvView / ExcelView | CsvHelper + ストリーミング | Jackson CSV + `respondOutputStream` | [Step 10](./step10.md) |

### P2（あると便利）— ステップに含まれない機能

以下の機能は 02-advanced-steps のステップには含まれていないが、必要に応じて実装できる。

#### DI/IoC

| Spring Boot | F# | Kotlin |
|---|---|---|
| Spring DI Container（`@Component`, `@Autowired`） | ASP.NET Core DI / 関数の部分適用 | Koin |

F# では「関数の引数に依存を渡す」のが基本。DI コンテナの必要性は Spring より大幅に低い。ただしインフラ層のライフサイクル管理には ASP.NET Core DI が有用。Kotlin では Koin（Ktor 公式推奨の軽量 DI）を使う。

#### 国際化（i18n）

| Spring Boot | F# | Kotlin |
|---|---|---|
| `MessageSource` + `LocaleResolver` | `IStringLocalizer` | `ResourceBundle` / JSON ベース |

#### AOP・横断的関心事

| Spring Boot | F# / Kotlin |
|---|---|
| `@Aspect` + `@Around` / `@Before` / `@After` | 高階関数で関数合成 |

Spring の AOP は暗黙的なプロキシで便利だが、デバッグ時にスタックトレースが追いにくい。高階関数は明示的で追跡しやすい。

#### マルチテナント

| Spring Boot | F# | Kotlin |
|---|---|---|
| ライブラリ依存 | `NpgsqlDataSource` テナント切替 | Exposed `Database.connect` テナント切替 |

スキーマ分離方式（テナントごとに PostgreSQL スキーマを切り替え）が最もシンプル。

---

## まとめ

### カバレッジ

| 優先度 | 機能数 | F# カバー率 | Kotlin カバー率 |
|---|---|---|---|
| P0 | 7 | 7/7（100%） | 7/7（100%） |
| P1 | 13 | 13/13（100%） | 13/13（100%） |
| P2 | 4 | 4/4（100%） | 4/4（100%） |

### F# vs Kotlin の差が出るポイント

| 観点 | F# が有利 | Kotlin が有利 |
|---|---|---|
| 認証・認可 | ASP.NET Core の認証基盤が Spring Security 並みに成熟 | — |
| DI | 部分適用で DI コンテナ不要にできる | Koin が軽量で使いやすい |
| バリデーション | Smart Constructor + Result が自然 | Arrow Validated でエラー蓄積が宣言的 |
| メトリクス・トレーシング | OpenTelemetry .NET が統合済み | Micrometer + JVM エコシステムが豊富 |
| テスト | WebApplicationFactory が高速 | Ktor testApplication が非常に軽量 |
| キャッシュ | IDistributedCache が標準抽象 | Caffeine が高性能 |

### Spring との比較: 何を失い、何を得るか

失うもの:
- 「設定ゼロで全部つながる」自動設定（各機能を明示的に組み込む必要がある）
- アノテーション 1 つで横断的関心事を注入する手軽さ
- Spring Security の宣言的メソッドレベル認可（Ktor のみ）

得るもの:
- ビルド速度の大幅な改善（F#: 5-15秒、Ktor: 15-30秒 vs Spring Boot: 30-120秒）
- 暗黙的な AOP/プロキシがないため、コードの追跡性が向上
- 型安全なバリデーション（コンパイル時検出）
- 関数型スタイルによるコードの一貫性（書き方のばらつきが減る）
- テスト起動の高速化（Spring Context のロード不要）
