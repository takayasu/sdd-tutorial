# Step 2: チャンク処理の基本

## 目的

### これは何か

大量データを一定件数（チャンク）ずつ読み取り→加工→書き込みする `process_in_chunks` 関数を実装する。「月次締め処理」を題材に、在庫ロットの一括状態遷移をバッチで実行する。

### なぜやるのか

- 10万件のロットを一度にメモリに載せると OutOfMemory になる。チャンク単位で処理すればメモリ使用量を一定に保てる
- チャンク単位でコミットすることで、途中で失敗しても処理済みのデータは確定される
- Spring Batch の `Reader → Processor → Writer` パイプラインに相当する仕組みを、関数3つの合成で実現する

### 何がうれしいのか

- 「10万件のロットを1000件ずつ処理して、全件の状態を更新する」バッチが動く
- 処理中にログで進捗が確認できる（「チャンク 5/100 完了」等）
- Reader / Processor / Writer を差し替えるだけで、別のバッチ処理にも使い回せる

## テスト用データの準備

バッチ処理を試すために、大量のテストデータを投入する:

```sql
-- 1万件のテストロットを作成（製造完了状態）
INSERT INTO lot (lot_number_year, lot_number_location, lot_number_seq,
                 division_code, department_code, section_code,
                 process_category, inspection_category, manufacturing_category,
                 status, manufacturing_completed_date)
SELECT 2026, 'A', seq,
       1, 1, 1, 1, 1, 1,
       'manufactured', '2026-04-01'
FROM generate_series(1, 10000) AS seq
ON CONFLICT DO NOTHING;
```

## 完了条件

1. バッチを実行すると、1万件のロットが1000件ずつチャンク処理されること:

```bash
uv run python -m batch.runner --job=monthly-close --date=2026-04
```

2. ログにチャンクごとの進捗が出力されること:

```
{"event":"Chunk 1/10 completed","processed":1000,"elapsed_ms":120}
{"event":"Chunk 2/10 completed","processed":2000,"elapsed_ms":115}
...
{"event":"Chunk 10/10 completed","processed":10000,"elapsed_ms":110}
{"event":"Job completed","job":"monthly-close","total_processed":10000}
```

3. 全ロットの状態が更新されていること:

```bash
docker compose exec db psql -U app -d sales_management \
  -c "SELECT status, count(*) FROM lot WHERE lot_number_year = 2026 GROUP BY status;"
# → shipping_instructed | 10000（または期待する遷移先の状態）
```

4. メモリ使用量がデータ件数に比例して増加しないこと（チャンクサイズ分だけ使用）

### 確認のコツ

- 最初はチャンクサイズを小さく（100件）して動作を確認し、その後1000件に増やす
- 処理時間を計測して、チャンクサイズによる性能差を体感する
- `process_in_chunks` 関数は汎用的に作る。月次締め固有のロジックは Reader / Processor / Writer に閉じ込める

---

## 実装ガイド

### 構造

```
BatchRunner（エントリポイント: batch/runner.py）
  └── process_in_chunks(reader, processor, writer, chunk_size)
        ├── reader: DB から製造完了ロットを読み取る（WHERE id > last_id ORDER BY id LIMIT chunk_size）
        ├── processor: ドメインロジック適用（状態遷移）
        └── writer: DB に一括更新
```

### Python / SQLAlchemy

```python
# batch/chunk.py
from collections.abc import AsyncGenerator, Callable
from typing import TypeVar

import structlog
from sqlalchemy.ext.asyncio import AsyncSession

T = TypeVar("T")
R = TypeVar("R")

logger = structlog.get_logger()


async def process_in_chunks(
    session: AsyncSession,
    chunk_size: int,
    reader: Callable[[int, int], AsyncGenerator[T, None]],
    processor: Callable[[T], R | None],
    writer: Callable[[AsyncSession, list[R]], None],
    get_id: Callable[[T], int],
) -> int:
    last_id = 0
    chunk_index = 0
    total_processed = 0

    while True:
        chunk = [row async for row in reader(last_id, chunk_size)]
        if not chunk:
            break

        chunk_index += 1
        results = [r for item in chunk if (r := processor(item)) is not None]

        async with session.begin():
            await writer(session, results)

        last_id = get_id(chunk[-1])
        total_processed += len(results)
        logger.info("chunk completed", chunk=chunk_index, processed=total_processed)

    return total_processed
```

```python
# batch/jobs/monthly_close.py
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession
from batch.chunk import process_in_chunks


async def reader(session: AsyncSession, last_id: int, limit: int):
    result = await session.execute(
        text(
            "SELECT id, lot_number, status FROM lot "
            "WHERE status = 'manufactured' AND id > :last_id "
            "ORDER BY id LIMIT :limit"
        ),
        {"last_id": last_id, "limit": limit},
    )
    for row in result:
        yield row


async def writer(session: AsyncSession, lots: list) -> None:
    for lot in lots:
        await session.execute(
            text("UPDATE lot SET status = 'shipping_instructed' WHERE id = :id"),
            {"id": lot.id},
        )


async def run_monthly_close(session: AsyncSession, date_param: str) -> int:
    return await process_in_chunks(
        session=session,
        chunk_size=1000,
        reader=lambda last_id, limit: reader(session, last_id, limit),
        processor=lambda row: row,
        writer=writer,
        get_id=lambda row: row.id,
    )
```

### バッチ用エントリポイント

API サーバーとは別に、コマンドライン引数でジョブを指定して実行するエントリポイントを作成する:

```python
# batch/runner.py
import argparse
import asyncio
import sys

from batch.database import AsyncSessionLocal
from batch.jobs.monthly_close import run_monthly_close


async def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("--job", required=True)
    parser.add_argument("--date", required=True)
    args = parser.parse_args()

    async with AsyncSessionLocal() as session:
        if args.job == "monthly-close":
            count = await run_monthly_close(session, args.date)
            print(f"Completed: {count} records processed")
            return 0
        else:
            print(f"Unknown job: {args.job}", file=sys.stderr)
            return 1


if __name__ == "__main__":
    sys.exit(asyncio.run(main()))
```

---

## 02-advanced-steps を併用する場合の注意

02-advanced-steps と 02-batch-steps の両方を実施する場合、以下の点に注意する:

- **楽観的ロック（02-advanced-steps Step 2）**: バッチは大量のレコードを一括更新するため、API 経由の更新と version が競合する可能性がある。バッチ処理では楽観的ロックをスキップするか、バッチ専用の更新関数（version チェックなし）を用意する
- **Outbox（02-advanced-steps Step 7）**: バッチで1万件の状態遷移を行うと、1万件の outbox_events が INSERT される。バッチ処理ではイベント発行を抑制するか、バッチ完了後にサマリイベント1件だけ発行する設計を検討する
- **監査ログ（02-advanced-steps Step 8）**: バッチ実行時の `updated_by` には `batch:<job_name>` 等のシステムユーザー識別子を設定する（JWT からのユーザー ID は存在しない）

---

## 次のステップ

Step 2が完了したら [Step 3: ジョブ実行管理 + 二重実行防止](./step03.md) へ進む。
