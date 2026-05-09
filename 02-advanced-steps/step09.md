# Step 9: レート制限 + キャッシュ + グレースフルシャットダウン

## 目的

### これは何か

APIへの過剰なリクエストを制限するレート制限、頻繁に参照されるデータのキャッシュ、そしてアプリ停止時に処理中のリクエストを安全に完了させるグレースフルシャットダウンを導入する。本番運用に必要な最後のピースを揃える。

### なぜやるのか

- レート制限がないと、バグや悪意のあるクライアントが大量のリクエストを送り、システム全体が応答不能になる
- キャッシュがないと、同じデータを何度もDBから読み取り、不要な負荷がかかる。ロットの参照APIは更新より参照の方が圧倒的に多い
- グレースフルシャットダウンがないと、デプロイ時に処理中のリクエストが途中で切断される。k8sのローリングアップデートで「502 Bad Gateway」が発生する原因になる

### 何がうれしいのか

- レート制限により、1クライアントが毎秒100リクエスト送っても、他のクライアントへの影響を防げる。`429 Too Many Requests` が返り、システムは安定したまま
- キャッシュにより、ロット参照APIのレスポンスタイムが大幅に改善する。DBクエリが不要になるため、10ms以下で応答できる
- グレースフルシャットダウンにより、デプロイ時にユーザーがエラーを見ることがなくなる。処理中のリクエストが完了してからアプリが停止する

## 完了条件

### レート制限の確認

1. 短時間に大量のリクエストを送ると、制限を超えた分が `429 Too Many Requests` で返ること:

```bash
# 連続で20回リクエストを送る（制限が10回/分の場合）
for i in $(seq 1 20); do
  echo "Request $i: $(curl -s -o /dev/null -w '%{http_code}' http://localhost:8000/lots/2024-A-001)"
done

# 出力例:
# Request 1: 200
# ...
# Request 10: 200
# Request 11: 429
# ...
# Request 20: 429
```

2. `429` レスポンスに `Retry-After` ヘッダが含まれていること（クライアントがいつリトライすべきかを知るため）

### キャッシュの確認

3. 同じロットを2回連続で取得し、2回目の方が速いこと:

```bash
# 1回目（DBアクセスあり）
time curl -s http://localhost:8000/lots/2024-A-001 > /dev/null
# real 0m0.050s（例）

# 2回目（キャッシュヒット）
time curl -s http://localhost:8000/lots/2024-A-001 > /dev/null
# real 0m0.005s（例、大幅に速い）
```

4. ロットの状態を変更した後、キャッシュが無効化され、最新のデータが返ること:

```bash
curl -X POST http://localhost:8000/lots/2024-A-001/complete-manufacturing -d '{"date":"2026-04-22"}'

curl http://localhost:8000/lots/2024-A-001
# → {"status":"manufactured", ...}（古いキャッシュが返らない）
```

5. ログでキャッシュヒット/ミスを確認できること:

```
{"level":"debug","event":"cache hit","key":"lot:2024-A-001"}
{"level":"debug","event":"cache miss","key":"lot:2024-A-002"}
```

### グレースフルシャットダウンの確認

6. リクエスト処理中にアプリを停止しても、そのリクエストが正常に完了すること:

```bash
# ターミナル1: 遅いリクエストを送信
curl http://localhost:8000/slow-operation &

# ターミナル2: アプリを停止（SIGTERM）
kill -SIGTERM <uvicorn_pid>

# ターミナル1: レスポンスが正常に返ること（接続が切断されない）
# → 200 OK
```

7. アプリ停止後、新しいリクエストは受け付けないこと

### 確認のコツ

- レート制限のテストは `for` ループで連続リクエストを送るのが簡単。`hey` や `ab` を使うとより正確に計測できる
- キャッシュの効果を確認するには、ログでDBクエリの実行有無を見る。キャッシュヒット時はDBクエリのログが出ないはず
- グレースフルシャットダウンのテストは、意図的に遅いエンドポイント（`asyncio.sleep(5)` 等）を作ると確認しやすい
- これら全てが動作したら、Step 0から積み上げてきた全機能が揃ったことになる。`ci.sh` を実行して全てのチェックが通ることを確認しよう

---

## 実装ガイド

### Python / FastAPI (Backend)

| 要素 | 実装方法 |
|---|---|
| レート制限 | `slowapi`（Flask-Limiter の FastAPI 版） |
| キャッシュ | `aiocache`（インメモリ / Redis 対応） |
| キャッシュ無効化 | 状態変更時に `cache.delete(key)` |
| グレースフルシャットダウン | uvicorn `--timeout-graceful-shutdown` + FastAPI lifespan |

#### 依存パッケージ追加

```bash
cd backend
uv add slowapi aiocache
```

#### レート制限 (`backend/src/middleware/rate_limit.py`)

```python
from slowapi import Limiter
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from fastapi import Request
from fastapi.responses import JSONResponse

limiter = Limiter(key_func=get_remote_address, default_limits=["100/minute"])


async def rate_limit_exceeded_handler(request: Request, exc: RateLimitExceeded) -> JSONResponse:
    return JSONResponse(
        status_code=429,
        headers={"Retry-After": "60"},
        content={
            "type": "rate-limit-exceeded",
            "title": "Too many requests",
            "status": 429,
            "detail": str(exc.detail),
        },
    )
```

```python
# src/main.py への追記
from slowapi.errors import RateLimitExceeded
from src.middleware.rate_limit import limiter, rate_limit_exceeded_handler

app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, rate_limit_exceeded_handler)
```

```python
# ルーターでの使用例
from src.middleware.rate_limit import limiter

@router.get("/lots/{lot_id}")
@limiter.limit("60/minute")
async def get_lot(request: Request, lot_id: str, claims: dict = Depends(require_auth)):
    ...
```

#### キャッシュ (`backend/src/cache.py`)

```python
import structlog
from aiocache import Cache

logger = structlog.get_logger()
_cache = Cache(Cache.MEMORY, ttl=300)


async def get_cached(key: str):
    value = await _cache.get(key)
    if value is not None:
        logger.debug("cache hit", key=key)
    else:
        logger.debug("cache miss", key=key)
    return value


async def set_cached(key: str, value: object) -> None:
    await _cache.set(key, value)


async def invalidate(key: str) -> None:
    await _cache.delete(key)
```

```python
# ロット参照APIでの使用例
from src.cache import get_cached, set_cached, invalidate

@router.get("/lots/{lot_id}")
async def get_lot(lot_id: str, ...):
    cache_key = f"lot:{lot_id}"
    cached = await get_cached(cache_key)
    if cached:
        return cached
    lot = await fetch_lot_from_db(lot_id, session)
    await set_cached(cache_key, lot)
    return lot


# 状態変更時にキャッシュを無効化
@router.post("/lots/{lot_id}/complete-manufacturing")
async def complete_manufacturing(lot_id: str, ...):
    lot = await do_complete_manufacturing(lot_id, body, session)
    await invalidate(f"lot:{lot_id}")
    return lot
```

#### グレースフルシャットダウン

uvicorn はデフォルトで SIGTERM を受け取ると、処理中のリクエストが完了するまで待機する:

```bash
# 本番起動コマンド（30秒待機）
uv run uvicorn src.main:app \
  --host 0.0.0.0 \
  --port 8000 \
  --timeout-graceful-shutdown 30
```

FastAPI lifespan でリソースクリーンアップを登録する:

```python
# src/main.py
@asynccontextmanager
async def lifespan(app):
    outbox_task = asyncio.create_task(_outbox_loop())
    yield
    # SIGTERM 受信後、進行中リクエスト完了後に実行
    outbox_task.cancel()
    await engine.dispose()


app = FastAPI(lifespan=lifespan)
```

k8s の `preStop` フックと組み合わせると、さらに安全なシャットダウンが実現できる:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sleep", "5"]  # ロードバランサーからの切り離しを待つ
```

---

## おめでとうございます！

Step 9が完了すると、以下の全機能が揃った業務システムの基盤が完成します:

| カテゴリ | 実装済み機能 | 出典 |
|---|---|---|
| ドメイン | 型安全な状態遷移、PBT、関数型ドメインモデリング | 01-steps/ Step 6-18 |
| CI/CD | フォーマッター、リンター、カバレッジ、SAST、SCA、DAST | 01-steps/ Step 4-11, 19-20 |
| 設定・ログ | 環境別設定（pydantic-settings）、structlog JSON ログ、リクエストID | 02-advanced-steps/ Step 1 |
| API品質 | Problem Details、Pydantic バリデーション | 02-advanced-steps/ Step 2 |
| セキュリティ | JWT認証（python-jose）、RBAC、CORS | 02-advanced-steps/ Step 3 |
| 運用 | ヘルスチェック、FastAPI OpenAPI/Swagger | 02-advanced-steps/ Step 4 |
| テスト | PBT + 統合テスト（pytest + httpx + testcontainers） | 01-steps/ Step 8 + 02-advanced-steps/ Step 5 |
| 外部連携 | httpx + tenacity リトライ + circuitbreaker | 02-advanced-steps/ Step 6 |
| イベント駆動 | ドメインイベント、Outbox パターン（asyncio lifespan） | 02-advanced-steps/ Step 7 |
| 監査・追跡 | 監査ログ（claims["sub"]）、OpenTelemetry + Jaeger | 02-advanced-steps/ Step 8 |
| 本番運用 | slowapi レート制限、aiocache、uvicorn グレースフルシャットダウン | 02-advanced-steps/ Step 9 |

Python/FastAPI + React スタックで、業務システムとして必要な全機能をカバーしました。

次のステップとして:
- [Step 10: CSV/ファイルエクスポート](./step10.md) でデータエクスポート機能を実装する
- `ci.sh` を更新し、新しいテスト（統合テスト）をCIに組み込む
- 本番デプロイに向けて、Dockerfile と k8s マニフェストを整備する
- [02-batch-steps/](../02-batch-steps/) を参考に、バッチ処理基盤を構築する
