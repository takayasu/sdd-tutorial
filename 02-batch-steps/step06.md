# Step 6: 並列処理（パーティショニング）

## 目的

### これは何か

大量データをID範囲で分割（パーティショニング）し、複数パーティションを並列で処理する。Spring Batch の `Partitioner` + `TaskExecutor` に相当する。

### なぜやるのか

- Step 2-5 のチャンク処理はシングルスレッド。10万件を1000件ずつ処理すると100チャンク分の時間がかかる
- パーティショニングにより、例えば10パーティション × 並列度5 で処理すれば、理論上5倍速くなる
- 棚卸・在庫計算のような大量データ集計では、並列処理が実用上必須

### 何がうれしいのか

- 処理時間が大幅に短縮される。10万件のバッチが15分 → 3分になるイメージ
- パーティションごとに独立してリスタートできる（Step 4 の進捗テーブルに `partition_id` を追加済み）
- 並列度を設定で変更できるため、DBの負荷に応じて調整可能

## テスト用データの準備

```bash
# 10万件のテストデータを投入
docker compose exec db psql -U app -d sales_management \
  -c "INSERT INTO lot (lot_number_year, lot_number_location, lot_number_seq,
      division_code, department_code, section_code,
      process_category, inspection_category, manufacturing_category,
      status, manufacturing_completed_date)
      SELECT 2026, 'P', seq, 1, 1, 1, 1, 1, 1, 'manufactured', '2026-04-01'
      FROM generate_series(1, 100000) AS seq ON CONFLICT DO NOTHING;"
```

## 完了条件

### 並列処理の確認

1. 10万件を5パーティションで並列処理し、シングルスレッドより速く完了すること:

```bash
# 並列実行
uv run python -m batch.runner --job=parallel-test --date=2026-04 --partitions=5

# ログでパーティション分割が確認できること
# {"event":"Partitioning","total_count":100000,"partitions":5,"ranges":["1-20000","20001-40000","40001-60000","60001-80000","80001-100000"]}
# {"event":"Partition completed","partition":1,"processed":20000,"elapsed_ms":3200}
# {"event":"Partition completed","partition":3,"processed":20000,"elapsed_ms":3300}
# {"event":"Partition completed","partition":2,"processed":20000,"elapsed_ms":3400}
# ...（完了順序は不定）
# {"event":"All partitions completed","total_processed":100000,"elapsed_ms":3500}
```

2. 処理時間がシングルスレッドより短いこと:

```bash
# シングルスレッド（比較用）
time uv run python -m batch.runner --job=single-test --date=2026-04 --partitions=1
# → real 0m15.000s（例）

# 5パーティション並列
time uv run python -m batch.runner --job=parallel-test --date=2026-04 --partitions=5
# → real 0m4.000s（例、大幅に短縮）
```

### パーティション別リスタートの確認

3. パーティション2が途中で失敗した場合、パーティション2だけ再実行されること:

```bash
# 実行（パーティション2で意図的にエラー）
uv run python -m batch.runner --job=partition-restart-test --date=2026-04 --partitions=5
# → パーティション0,1,3,4は完了、パーティション2はFAILED

# 進捗確認
docker compose exec db psql -U app -d sales_management \
  -c "SELECT partition_id, last_processed_id, processed_count FROM batch_chunk_progress WHERE job_name = 'partition-restart-test';"
# → partition_id=2 のみレコードが残る（他は完了済みで削除）

# 再実行 → パーティション2だけ再開
uv run python -m batch.runner --job=partition-restart-test --date=2026-04 --partitions=5
# → {"event":"Partition already completed, skipping","partition":0}
# → {"event":"Partition already completed, skipping","partition":1}
# → {"event":"Partition restarting","partition":2,"last_processed_id":45000}
# → ...
```

### 確認のコツ

- 最初は2パーティションで動作を確認し、その後5パーティションに増やす
- `batch_chunk_progress` テーブルの `partition_id` カラムでパーティション別の進捗を確認する
- 並列度を上げすぎるとDBのコネクションプールが枯渇する。コネクション数 > 並列度 であることを確認する

---

## 実装ガイド

### パーティション分割

```python
# batch/partitioner.py
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession


async def create_partitions(
    session: AsyncSession, table: str, partition_count: int
) -> list[tuple[int, int, int]]:
    result = await session.execute(text(f"SELECT MIN(id), MAX(id) FROM {table}"))
    row = result.fetchone()
    min_id, max_id = row[0], row[1]

    range_size = (max_id - min_id + 1) // partition_count
    partitions = []
    for i in range(partition_count):
        from_id = min_id + i * range_size
        to_id = max_id if i == partition_count - 1 else from_id + range_size - 1
        partitions.append((i, from_id, to_id))
    return partitions
```

### 並列実行

```python
# batch/parallel.py
import asyncio
from collections.abc import Callable, Awaitable

from batch.partitioner import create_partitions
from batch.database import AsyncSessionLocal


async def process_partitions(
    table: str,
    partition_count: int,
    process_partition: Callable[[int, int, int], Awaitable[int]],
) -> int:
    async with AsyncSessionLocal() as session:
        partitions = await create_partitions(session, table, partition_count)

    results = await asyncio.gather(
        *[process_partition(part_id, from_id, to_id) for part_id, from_id, to_id in partitions]
    )
    return sum(results)
```

```python
# batch/runner.py での使用例
from batch.parallel import process_partitions
from batch.database import AsyncSessionLocal
from batch.jobs.monthly_close import process_partition


async def run_parallel_job(partitions: int) -> None:
    async def run_one(part_id: int, from_id: int, to_id: int) -> int:
        async with AsyncSessionLocal() as session:
            return await process_partition(session, part_id, from_id, to_id)

    total = await process_partitions("lot", partitions, run_one)
    print(f"All partitions completed: {total} records processed")
```

---

## 次のステップ

Step 6が完了したら [Step 7: EventBridge + Step Functions スケジューリング](./step07.md) へ進む。
