# Step 5: スキップ/リトライ + リスナー

## 目的

### これは何か

チャンク処理中に一部のアイテムでエラーが発生した場合に、リトライ（再試行）やスキップ（飛ばして続行）する仕組みと、ジョブ/チャンクのライフサイクルにフックを差し込むリスナーを実装する。

### なぜやるのか

- 1万件中1件だけ不正データがあるとき、バッチ全体を止めるのは過剰。「N件まではスキップして続行」という柔軟性が必要
- 外部API呼び出しを含むバッチでは、一時的なネットワークエラーでリトライすれば成功するケースが多い
- リスナーにより、チャンクの開始/完了/エラーをログに記録したり、メトリクスを収集したりできる

### 何がうれしいのか

- 「1万件中3件がスキップされ、9997件が正常処理された」という結果が `batch_job_execution` に記録される
- スキップされたアイテムはログに記録されるため、後から手動で対処できる
- リスナーにより、バッチの実行状況をリアルタイムで監視できる

## 完了条件

### スキップの確認

1. 1万件中5件に不正データを仕込み、スキップ上限を10件に設定してバッチを実行:

```bash
# 不正データを5件作成
docker compose exec db psql -U app -d sales_management \
  -c "UPDATE lot SET status = 'invalid' WHERE lot_number_year = 2026 AND lot_number_location = 'C' AND lot_number_seq IN (100, 500, 2000, 5000, 9000);"

# バッチ実行（max_skips=10）
uv run python -m batch.runner --job=skip-test --date=2026-04
```

2. バッチが正常完了し、スキップ件数が記録されること:

```bash
docker compose exec db psql -U app -d sales_management \
  -c "SELECT status, read_count, write_count, skip_count FROM batch_job_execution WHERE job_name = 'skip-test';"
# → COMPLETED | 10000 | 9995 | 5
```

3. スキップされたアイテムがログに記録されていること:

```
{"level":"warning","event":"Item skipped","lot_id":"2026-C-100","reason":"Invalid status: invalid"}
{"level":"warning","event":"Item skipped","lot_id":"2026-C-500","reason":"Invalid status: invalid"}
...
```

4. スキップ上限を超えるとバッチが失敗すること:

```bash
# 不正データを15件に増やし、max_skips=10 で実行
# → "Skip limit exceeded: 11" で FAILED
```

### リトライの確認

5. 一時的なエラーが発生した場合、リトライにより成功すること。特定のアイテムで初回だけ例外をスローするテスト用ロジックを仕込んで確認する:

```
{"level":"warning","event":"Retry attempt","attempt":1,"item":"2026-C-3000","error":"Transient error (test)"}
{"level":"info","event":"Retry succeeded","item":"2026-C-3000","attempt":2}
```

### リスナーの確認

6. ジョブの開始/完了、チャンクの開始/完了がログに記録されること:

```
{"level":"info","event":"Job started","job":"skip-test","params":"2026-04"}
{"level":"info","event":"Chunk started","chunk":1}
{"level":"info","event":"Chunk completed","chunk":1,"processed":998,"skipped":2,"elapsed_ms":130}
...
{"level":"info","event":"Job completed","job":"skip-test","total_processed":9995,"total_skipped":5}
```

---

## 実装ガイド

### スキップ/リトライの構造

```python
# batch/chunk_config.py
from dataclasses import dataclass, field
from collections.abc import Callable


@dataclass
class ChunkConfig:
    max_skips: int = 10
    max_retries: int = 3
    is_retryable: Callable[[Exception], bool] = field(
        default=lambda exc: "deadlock" in str(exc).lower()
    )
    is_skippable: Callable[[Exception], bool] = field(
        default=lambda exc: isinstance(exc, ValueError)
    )
```

### リスナーの構造

```python
# batch/listeners.py
from dataclasses import dataclass, field
from collections.abc import Callable


@dataclass
class BatchListeners:
    on_job_start: Callable[[str], None] = field(default=lambda job: None)
    on_job_end: Callable[[str], None] = field(default=lambda job: None)
    on_chunk_start: Callable[[int], None] = field(default=lambda idx: None)
    on_chunk_end: Callable[[int, int, int], None] = field(
        default=lambda idx, processed, skipped: None
    )
    on_chunk_error: Callable[[int, Exception], None] = field(
        default=lambda idx, exc: None
    )
    on_item_skipped: Callable[[object, Exception], None] = field(
        default=lambda item, exc: None
    )
```

### process_in_chunks への組み込み

`process_in_chunks` の引数に `ChunkConfig` と `BatchListeners` を追加し、チャンクループ内でリスナーを呼び出す:

```python
# batch/chunk.py（Step 2 からの拡張）
import structlog
from batch.chunk_config import ChunkConfig
from batch.listeners import BatchListeners

logger = structlog.get_logger()


async def process_item_with_retry(item, processor, config: ChunkConfig, listeners: BatchListeners):
    for attempt in range(1, config.max_retries + 1):
        try:
            return processor(item)
        except Exception as exc:
            if attempt < config.max_retries and config.is_retryable(exc):
                logger.warning("retry attempt", attempt=attempt, error=str(exc))
                continue
            if config.is_skippable(exc):
                listeners.on_item_skipped(item, exc)
                return None
            raise
    return None


async def process_in_chunks(
    session,
    chunk_size: int,
    reader,
    processor,
    writer,
    get_id,
    config: ChunkConfig | None = None,
    listeners: BatchListeners | None = None,
) -> tuple[int, int]:
    cfg = config or ChunkConfig()
    lst = listeners or BatchListeners()
    last_id = 0
    chunk_index = 0
    total_processed = 0
    total_skipped = 0

    while True:
        chunk = [row async for row in reader(last_id, chunk_size)]
        if not chunk:
            break

        chunk_index += 1
        lst.on_chunk_start(chunk_index)

        results = []
        chunk_skipped = 0
        for item in chunk:
            result = await process_item_with_retry(item, processor, cfg, lst)
            if result is None:
                chunk_skipped += 1
                total_skipped += 1
                if total_skipped > cfg.max_skips:
                    raise RuntimeError(f"Skip limit exceeded: {total_skipped}")
            else:
                results.append(result)

        async with session.begin():
            await writer(session, results)

        last_id = get_id(chunk[-1])
        total_processed += len(results)
        lst.on_chunk_end(chunk_index, len(results), chunk_skipped)

    return total_processed, total_skipped
```

---

## 次のステップ

Step 5が完了したら [Step 6: 並列処理（パーティショニング）](./step06.md) へ進む。
