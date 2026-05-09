# Advanced Steps: Spring Boot 機能の Python/FastAPI + React 実装

Spring Boot が業務システムに提供する機能（バッチ処理を除く）を、Python/FastAPI（バックエンド）と React/TypeScript（フロントエンド）で代替する。01-steps/ を全て完了した前提で、業務システムとしての本番運用に耐えうるかを PoC する。

バッチ処理については [02-batch-steps/](../02-batch-steps/) を参照。

---

## 設計方針

- アプリの核心ロジックはクラウド非依存で自己完結させる
- Spring の「設定ゼロで全部つながる」自動設定は再現しない。各機能を明示的に組み込む
- Python の型ヒント + Pydantic による型安全性を最大限活用する
- フロントエンドは React + TypeScript で、バックエンドの OpenAPI スキーマからコード生成する

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

| 機能 | Spring Boot | Python/FastAPI | Step |
|---|---|---|---|
| 設定管理 | `application.yml` + Profile | `pydantic-settings` + `.env` | [Step 1](./step01.md) |
| ロギング | SLF4J + Logback | `structlog`（構造化 JSON ログ） | [Step 1](./step01.md) |
| エラーハンドリング | `@ControllerAdvice` | `app.add_exception_handler` + `JSONResponse` | [Step 2](./step02.md) |
| 入力バリデーション | Bean Validation | Pydantic `Annotated` + `Field` / Smart Constructor | [Step 2](./step02.md) |
| 認証・認可 | Spring Security | `python-jose` JWT + `Depends(require_auth)` | [Step 3](./step03.md) |
| トランザクション管理 | `@Transactional` | SQLAlchemy `session.begin()` + `async with` | 01-steps/ で実装済み |
| コネクションプール | HikariCP | SQLAlchemy `AsyncEngine`（asyncpg 内蔵プール） | 01-steps/ で実装済み |

### P1（運用品質に強く推奨）

| 機能 | Spring Boot | Python/FastAPI | Step |
|---|---|---|---|
| ヘルスチェック | Actuator `/health` | 自前エンドポイント（`SELECT 1`） | [Step 4](./step04.md) |
| API 仕様 | SpringDoc (OpenAPI) | FastAPI 標準（`/docs`, `/openapi.json`） | [Step 4](./step04.md) |
| テスト支援 | `@SpringBootTest` + MockMvc | `pytest` + `httpx.AsyncClient` + ASGI | [Step 5](./step05.md) |
| HTTP クライアント | RestClient / WebClient | `httpx.AsyncClient` | [Step 6](./step06.md) |
| リトライ・レジリエンス | Spring Retry + Resilience4j | `tenacity` + `circuitbreaker` | [Step 6](./step06.md) |
| 非同期・イベント | `@Async` + `ApplicationEvent` | asyncio + `asynccontextmanager` lifespan | [Step 7](./step07.md) |
| 監査ログ | Spring Data Auditing | `claims["sub"]` → DB カラム（自前） | [Step 8](./step08.md) |
| 分散トレーシング | Micrometer Tracing | `opentelemetry-sdk` + `FastAPIInstrumentor` | [Step 8](./step08.md) |
| レート制限 | Bucket4j / Spring Cloud Gateway | `slowapi` | [Step 9](./step09.md) |
| キャッシュ | `@Cacheable` | `aiocache`（インメモリ / Redis） | [Step 9](./step09.md) |
| グレースフルシャットダウン | `server.shutdown=graceful` | uvicorn `--timeout-graceful-shutdown` | [Step 9](./step09.md) |
| ファイルエクスポート | CsvView / ExcelView | Python `csv` + `StreamingResponse`（CP932） | [Step 10](./step10.md) |

### P2（あると便利）— ステップに含まれない機能

以下の機能は 02-advanced-steps のステップには含まれていないが、必要に応じて実装できる。

#### DI/IoC

| Spring Boot | Python/FastAPI |
|---|---|
| Spring DI Container（`@Component`, `@Autowired`） | FastAPI `Depends()` による関数ベース DI |

FastAPI では「関数の引数に依存を渡す」`Depends()` が基本。DI コンテナの必要性は Spring より大幅に低い。

#### 国際化（i18n）

| Spring Boot | Python/FastAPI |
|---|---|
| `MessageSource` + `LocaleResolver` | `Accept-Language` ヘッダ + JSON メッセージカタログ |

#### AOP・横断的関心事

| Spring Boot | Python/FastAPI |
|---|---|
| `@Aspect` + `@Around` / `@Before` / `@After` | ミドルウェア（`BaseHTTPMiddleware`）または高階関数で関数合成 |

Spring の AOP は暗黙的なプロキシで便利だが、デバッグ時にスタックトレースが追いにくい。ミドルウェアは明示的で追跡しやすい。

#### マルチテナント

| Spring Boot | Python/FastAPI |
|---|---|
| ライブラリ依存 | リクエストごとに `AsyncEngine` の接続先を切替 |

スキーマ分離方式（テナントごとに PostgreSQL スキーマを切り替え）が最もシンプル。

---

## まとめ

### カバレッジ

| 優先度 | 機能数 | Python/FastAPI カバー率 |
|---|---|---|
| P0 | 7 | 7/7（100%） |
| P1 | 12 | 12/12（100%） |
| P2 | 4 | 4/4（100%） |

### Spring との比較: 何を失い、何を得るか

失うもの:
- 「設定ゼロで全部つながる」自動設定（各機能を明示的に組み込む必要がある）
- アノテーション 1 つで横断的関心事を注入する手軽さ
- Spring Security の宣言的メソッドレベル認可

得るもの:
- 起動時間の大幅な改善（uvicorn: 1-2 秒 vs Spring Boot: 30-120 秒）
- Python エコシステムの豊富なデータ処理・ML ライブラリとの統合
- Pydantic による型安全なバリデーション（コンパイル相当の検出）
- FastAPI の自動 OpenAPI 生成（コードと仕様が常に一致）
- pytest の高速テスト実行（Spring Context ロード不要）
- asyncio による高い並行性（async/await ファースト設計）
