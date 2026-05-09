# Step 9: CSV インポートバッチ

## 目的

### これは何か

CSV ファイルからデータを読み取り、行ごとにバリデーションしてDBに一括登録するバッチジョブを実装する。Step 2-6 で構築した `process_in_chunks` インフラを、ファイルベースの Reader で使う。

### なぜやるのか

- Step 2-8 のバッチは全て DB→DB 処理だった。業務システムでは「外部システムから受け取った CSV をインポートする」パターンが非常に多い
- Spring Batch の `FlatFileItemReader` に相当する機能を、Python 標準の `csv` モジュール + `process_in_chunks` で実現する
- 行ごとのバリデーションエラーを蓄積し、「100行中3行がエラー、97行が正常登録」という結果を返す

### 何がうれしいのか

- `process_in_chunks` の Reader を差し替えるだけで、DB→DB バッチと同じインフラ（リスタート、スキップ、リスナー）がファイルバッチにも使える
- バリデーションエラーの行番号と内容がログに記録されるため、データ提供元にフィードバックできる
- 大量の CSV（数万行）でもチャンク単位でコミットするため、途中で失敗しても処理済み分は確定される

## テスト用 CSV ファイルの準備

```bash
mkdir -p data
```

```csv
# data/import_lots.csv
ロット番号年度,ロット番号保管場所,ロット番号連番,事業部コード,部門コード,担当課コード,工程区分,検査区分,製造区分
2026,C,1,1,1,1,1,1,1
2026,C,2,1,1,1,1,1,1
2026,C,3,1,1,1,1,1,1
2026,C,invalid,1,1,1,1,1,1
2026,C,5,1,1,1,1,1,1
```

4行目の `invalid` はバリデーションエラーを発生させるためのテストデータ。

## 完了条件

### フェーズA: アプリ単体

1. CSV インポートバッチを実行し、正常行がDBに登録されること:

```bash
uv run python -m batch.runner --job=import-lots --file=data/import_lots.csv
```

2. `batch_job_execution` に結果が記録されること:

```bash
docker compose exec db psql -U app -d sales_management \
  -c "SELECT job_name, status, read_count, write_count, skip_count FROM batch_job_execution WHERE job_name = 'import-lots';"
# → import-lots | COMPLETED | 5 | 4 | 1
```

3. 正常行がDBに登録されていること:

```bash
docker compose exec db psql -U app -d sales_management \
  -c "SELECT lot_number_seq, status FROM lot WHERE lot_number_year = 2026 AND lot_number_location = 'C' ORDER BY lot_number_seq;"
# → 1, 2, 3, 5 が manufacturing 状態で登録（4行目の invalid はスキップ）
```

4. スキップされた行がログに記録されていること:

```
{"level":"warning","event":"Row skipped","line":4,"reason":"lot_number_seq must be a positive integer","raw":"2026,C,invalid,1,1,1,1,1,1"}
```

5. エンコーディングが Windows-31J の CSV でも正しく読み取れること:

```bash
# Windows-31J の CSV を作成
nkf -s data/import_lots.csv > data/import_lots_sjis.csv

uv run python -m batch.runner --job=import-lots --file=data/import_lots_sjis.csv --encoding=windows-31j
# → 正常に処理される
```

6. 大量データ（1万行）の CSV でもチャンク処理されること:

```bash
# 1万行の CSV を生成
python3 -c "
print('ロット番号年度,ロット番号保管場所,ロット番号連番,事業部コード,部門コード,担当課コード,工程区分,検査区分,製造区分')
for i in range(1, 10001):
    print(f'2026,D,{i},1,1,1,1,1,1')
" > data/import_lots_large.csv

uv run python -m batch.runner --job=import-lots-large --file=data/import_lots_large.csv
# → チャンクごとにログが出力される
# → batch_job_execution.write_count = 10000
```

---

## 実装ガイド

### 構造

```
process_in_chunks(
  reader:    CSV ファイルから行を読み取り、パース済みオブジェクトを yield するジェネレータ
  processor: 行ごとのバリデーション（Smart Constructor）
  writer:    DB に一括 INSERT
)
```

### Python / SQLAlchemy

| 要素 | 実装方法 |
|---|---|
| CSV 読み取り | Python 標準 `csv` モジュール — ストリーミング読み取り |
| エンコーディング | `open(file, encoding="cp932")` — Windows-31J（CP932） |
| バリデーション | Smart Constructor パターン（Step 2 で作成済み） |
| DB 書き込み | SQLAlchemy `text()` + バッチ INSERT |

```python
# batch/jobs/import_lots.py
import csv
from dataclasses import dataclass

import structlog
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession

logger = structlog.get_logger()


@dataclass
class LotImportRow:
    line_number: int
    lot_number_year: int
    lot_number_location: str
    lot_number_seq: int
    division_code: int
    department_code: int
    section_code: int
    process_category: int
    inspection_category: int
    manufacturing_category: int


def parse_row(line_number: int, raw: dict) -> LotImportRow:
    try:
        return LotImportRow(
            line_number=line_number,
            lot_number_year=int(raw["ロット番号年度"]),
            lot_number_location=raw["ロット番号保管場所"],
            lot_number_seq=int(raw["ロット番号連番"]),
            division_code=int(raw["事業部コード"]),
            department_code=int(raw["部門コード"]),
            section_code=int(raw["担当課コード"]),
            process_category=int(raw["工程区分"]),
            inspection_category=int(raw["検査区分"]),
            manufacturing_category=int(raw["製造区分"]),
        )
    except (ValueError, KeyError) as exc:
        raise ValueError(f"Parse error: {exc}") from exc


async def csv_reader(file_path: str, encoding: str, last_id: int, limit: int):
    with open(file_path, encoding=encoding, newline="") as f:
        reader = csv.DictReader(f)
        line_number = 0
        count = 0
        for raw in reader:
            line_number += 1
            if line_number <= last_id:
                continue
            if count >= limit:
                break
            yield (line_number, raw)
            count += 1


async def writer(session: AsyncSession, rows: list[LotImportRow]) -> None:
    for row in rows:
        await session.execute(
            text(
                "INSERT INTO lot (lot_number_year, lot_number_location, lot_number_seq, "
                "division_code, department_code, section_code, "
                "process_category, inspection_category, manufacturing_category, status) "
                "VALUES (:year, :loc, :seq, :div, :dept, :sec, :proc, :insp, :mfg, 'manufacturing') "
                "ON CONFLICT DO NOTHING"
            ),
            {
                "year": row.lot_number_year, "loc": row.lot_number_location,
                "seq": row.lot_number_seq, "div": row.division_code,
                "dept": row.department_code, "sec": row.section_code,
                "proc": row.process_category, "insp": row.inspection_category,
                "mfg": row.manufacturing_category,
            },
        )


async def run_import_lots(
    session: AsyncSession, file_path: str, encoding: str = "utf-8"
) -> tuple[int, int]:
    from batch.chunk import process_in_chunks
    from batch.chunk_config import ChunkConfig

    def processor(item: tuple[int, dict]) -> LotImportRow | None:
        line_number, raw = item
        try:
            return parse_row(line_number, raw)
        except ValueError as exc:
            logger.warning("row skipped", line=line_number, reason=str(exc), raw=str(raw))
            return None

    cfg = ChunkConfig(max_skips=100, is_skippable=lambda exc: isinstance(exc, ValueError))

    return await process_in_chunks(
        session=session,
        chunk_size=1000,
        reader=lambda last_id, limit: csv_reader(file_path, encoding, last_id, limit),
        processor=processor,
        writer=writer,
        get_id=lambda item: item[0],
        config=cfg,
    )
```

### リスタートとの組み合わせ

CSV インポートのリスタートは「行番号」をオフセットとして使う。`batch_chunk_progress.last_processed_id` に最後に処理した行番号を記録し、再実行時はその行番号以降から読み取る。`csv_reader` の `last_id` 引数がそのオフセットに対応する。

---

## バッチ処理基盤の完成

Step 9 が完了すると、DB→DB バッチ（Step 2-8）に加えて、ファイル→DB バッチも動作する状態になります。`process_in_chunks` の Reader を差し替えるだけで、リスタート・スキップ・リスナー・並列処理の全てが使い回せることが確認できました。
