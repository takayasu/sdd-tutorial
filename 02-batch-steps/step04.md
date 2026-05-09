# Step 4: チャンクリスタート

## 目的

### これは何か

バッチ処理が途中で失敗した場合に、処理済みのチャンクをスキップして失敗箇所から再開する仕組みを実装する。Spring Batch の `ExecutionContext`（オフセット永続化）に相当する。

### なぜやるのか

- Step 3 では失敗時に `FAILED` を記録するが、再実行すると最初からやり直しになる。10万件中9万件目で失敗したら、また1件目から処理し直すのは非効率
- `batch_chunk_progress` テーブルに「どこまで処理したか」を記録し、再実行時にその位置から再開する
- 業務データの書き込みと進捗記録を同一トランザクションで行うことで、「書いたのに進捗が記録されない」不整合を防ぐ

### 何がうれしいのか

- 10万件中5万件目で失敗しても、再実行時は5万1件目から再開される。処理済みデータの二重処理が起きない
- 「夜間バッチが途中で落ちた → 朝に再実行 → 残りだけ処理して完了」という運用が可能になる
- Spring Batch のリスタート機能と同等の信頼性を、1テーブル + 数行のコードで実現できる

## 完了条件

### リスタートの確認

1. テストデータを投入し、途中で意図的に失敗させる:

```bash
# 1万件のテストデータを投入
docker compose exec db psql -U app -d sales_management \
  -c "INSERT INTO lot (lot_number_year, lot_number_location, lot_number_seq,
      division_code, department_code, section_code,
      process_category, inspection_category, manufacturing_category,
      status, manufacturing_completed_date)
      SELECT 2026, 'B', seq, 1, 1, 1, 1, 1, 1, 'manufactured', '2026-04-01'
      FROM generate_series(1, 10000) AS seq ON CONFLICT DO NOTHING;"

# 5000件目で失敗するように仕込む
docker compose exec db psql -U app -d sales_management \
  -c "UPDATE lot SET status = 'invalid_status' WHERE lot_number_year = 2026 AND lot_number_location = 'B' AND lot_number_seq = 5000;"
```

2. バッチを実行すると、5000件目付近で失敗すること:

```bash
uv run python -m batch.runner --job=restart-test --date=2026-04
# → エラー発生、status = FAILED
```

3. `batch_chunk_progress` に進捗が記録されていること:

```bash
docker compose exec db psql -U app -d sales_management \
  -c "SELECT * FROM batch_chunk_progress WHERE job_name = 'restart-test';"
# → last_processed_id = 4000（4000件目まで処理済み）, processed_count = 4000
```

4. 不正データを修正して再実行すると、4001件目から再開されること:

```bash
# 不正データを修正
docker compose exec db psql -U app -d sales_management \
  -c "UPDATE lot SET status = 'manufactured' WHERE lot_number_year = 2026 AND lot_number_location = 'B' AND lot_number_seq = 5000;"

# 再実行
uv run python -m batch.runner --job=restart-test --date=2026-04
```

5. ログで再開位置が確認できること:

```
{"event":"Restarting from last_processed_id","last_processed_id":4000}
{"event":"Chunk 1 completed","processed":5000,"started_from":4001}
...
{"event":"Job completed","total_processed":10000}
```

6. 完了後、`batch_chunk_progress` のレコードが削除されていること（完了したジョブの進捗は不要）:

```bash
docker compose exec db psql -U app -d sales_management \
  -c "SELECT * FROM batch_chunk_progress WHERE job_name = 'restart-test';"
# → 0 rows
```

### 同一トランザクションの確認

7. 業務データの更新と進捗記録が同一トランザクションであることを確認する。チャンク処理中にアプリを強制終了（`kill -9`）しても、進捗と業務データが一致していること:

```bash
# 処理中に別ターミナルから kill -9 <pid>
# → batch_chunk_progress.last_processed_id と実際に更新されたロット数が一致する
```

### 確認のコツ

- チャンクサイズを小さく（100件）して、リスタートの動作を細かく観察する
- `batch_chunk_progress` テーブルを処理中に SELECT して、チャンクごとに `last_processed_id` が更新されることを確認する
- Reader の SQL は `WHERE id > :last_id ORDER BY id LIMIT :chunk_size` の形にする。これにより、`last_id` を渡すだけでリスタート位置から読み取れる

---

## 実装ガイド

### process_in_chunks の変更点（Step 2 からの差分）

```
Before（Step 2）:
  チャンクループ:
    Read → Process → Write → COMMIT

After（Step 4）:
  起動時: get_last_processed_id() → offset 取得
  チャンクループ:
    Read(offset以降) → Process → Write → upsert_progress() → COMMIT（同一TX）
  完了時: delete_progress()
```

### Python / SQLAlchemy での実装

```python
# batch/chunk_progress.py
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession


async def get_last_processed_id(
    session: AsyncSession, job_name: str, job_params: str, partition_id: int = 0
) -> int:
    result = await session.execute(
        text(
            "SELECT last_processed_id FROM batch_chunk_progress "
            "WHERE job_name = :name AND job_params = :params AND partition_id = :pid"
        ),
        {"name": job_name, "params": job_params, "pid": partition_id},
    )
    row = result.fetchone()
    return row.last_processed_id if row else 0


async def upsert_progress(
    session: AsyncSession,
    job_name: str,
    job_params: str,
    partition_id: int,
    last_processed_id: int,
    processed_count: int,
) -> None:
    await session.execute(
        text(
            "INSERT INTO batch_chunk_progress "
            "(job_name, job_params, partition_id, last_processed_id, processed_count) "
            "VALUES (:name, :params, :pid, :last_id, :count) "
            "ON CONFLICT (job_name, job_params, partition_id) DO UPDATE "
            "SET last_processed_id = :last_id, processed_count = :count, updated_at = NOW()"
        ),
        {
            "name": job_name, "params": job_params, "pid": partition_id,
            "last_id": last_processed_id, "count": processed_count,
        },
    )


async def delete_progress(
    session: AsyncSession, job_name: str, job_params: str
) -> None:
    await session.execute(
        text("DELETE FROM batch_chunk_progress WHERE job_name = :name AND job_params = :params"),
        {"name": job_name, "params": job_params},
    )
    await session.commit()
```

### upsert_progress の呼び出しタイミング

業務データの Write と同一トランザクション内で `batch_chunk_progress` を UPSERT する。`ON CONFLICT ... DO UPDATE` で冪等に。`process_in_chunks` の `writer` コールバックと同じトランザクション内で呼び出すよう `process_in_chunks` を拡張する。

---

## 次のステップ

Step 4が完了したら [Step 5: スキップ/リトライ + リスナー](./step05.md) へ進む。
