# Step 7: 非同期処理・イベント駆動

## 目的

### これは何か

ドメインイベント（「ロットが製造完了した」「契約が締結された」等）を発行し、後続処理を実行する仕組みを導入する。段階的に進める:

1. まずアプリ内の同期イベントバスを作る（シンプル）
2. 次に Outbox パターンでイベントをDBに永続化する（信頼性向上）
3. 最後にバックグラウンドポーリングで非同期処理にする

### なぜやるのか

- 「ロットが製造完了したら倉庫システムに通知する」「契約が締結されたら請求書を生成する」といった後続処理が必要
- 後続処理をAPIハンドラに直接書くと、ドメインロジックと通知ロジックが混ざり、変更が困難になる

### 何がうれしいのか

- ドメインロジックと後続処理が分離される。「製造完了」のロジックを変更しても、通知処理に影響しない
- Outbox パターンにより「DBには保存されたがイベントが発行されなかった」という不整合が起きない
- イベントハンドラを追加するだけで後続処理を増やせる

## フェーズ 1: 同期イベントバス

### やること

ドメインロジックの実行後に、イベントを発行し、登録されたハンドラを呼び出す。まずは同期（APIレスポンスを返す前にハンドラが実行される）で実装する。

### 構造

```
APIリクエスト
  → ドメインロジック実行（製造完了）
  → イベント発行: LotManufacturingCompleted
  → ハンドラ1: ログ出力
  → ハンドラ2: （将来）倉庫通知
  → レスポンス返却
```

### Python での実装イメージ

```python
# backend/src/events/bus.py
from dataclasses import dataclass
from datetime import date
from typing import Callable


@dataclass(frozen=True)
class LotManufacturingCompleted:
    lot_id: str
    date: date


@dataclass(frozen=True)
class ContractSigned:
    contract_id: str


DomainEvent = LotManufacturingCompleted | ContractSigned

_handlers: list[Callable[[DomainEvent], None]] = []


def subscribe(handler: Callable[[DomainEvent], None]) -> None:
    _handlers.append(handler)


def publish(event: DomainEvent) -> None:
    for handler in _handlers:
        handler(event)
```

```python
# アプリ起動時にハンドラ登録（src/main.py）
import structlog
from src.events.bus import subscribe, LotManufacturingCompleted

logger = structlog.get_logger()


def log_event(event: object) -> None:
    if isinstance(event, LotManufacturingCompleted):
        logger.info("lot manufacturing completed", lot_id=event.lot_id, date=str(event.date))


subscribe(log_event)
```

```python
# APIハンドラ内で使用
from src.events.bus import publish, LotManufacturingCompleted

async def complete_manufacturing(lot_id: str, body: CompleteMfgInput, ...):
    lot = await do_complete_manufacturing(lot_id, body, session)
    publish(LotManufacturingCompleted(lot_id=lot_id, date=body.date))
    return lot
```

### フェーズ 1 の完了条件

1. ロットの製造完了APIを叩くと、ログにイベントが記録されること:

```bash
curl -X POST -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/lots/2024-A-001/complete-manufacturing \
  -d '{"date":"2026-04-22"}'
# → 200 OK

# ログ:
# {"level":"info","event":"lot manufacturing completed","lot_id":"2024-A-001","date":"2026-04-22"}
```

2. イベントハンドラが失敗しても、ドメインロジック（DB保存）は成功していること

---

## フェーズ 2: Outbox パターン

### やること

フェーズ 1 のイベント発行を、DBの `outbox_events` テーブルへの INSERT に変更する。業務データの保存とイベントの記録を同一トランザクションで行う。

### なぜ Outbox が必要か

フェーズ 1 の問題点:
- イベントハンドラが失敗すると、イベントが消失する（リトライできない）
- アプリがクラッシュすると、発行予定だったイベントが消失する

Outbox パターンの解決策:
- イベントをDBに保存する → アプリがクラッシュしてもイベントは残る
- 業務データと同一トランザクション → 「データは保存されたがイベントは消えた」が起きない

### Outbox テーブル

```sql
CREATE TABLE outbox_events (
    id           BIGSERIAL PRIMARY KEY,
    event_type   TEXT NOT NULL,
    payload      JSONB NOT NULL,
    status       TEXT NOT NULL DEFAULT 'pending',
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    processed_at TIMESTAMPTZ
);

CREATE INDEX idx_outbox_pending ON outbox_events (status) WHERE status = 'pending';
```

### 構造の変化

```
Before（フェーズ 1）:
  トランザクション: 業務データ保存
  トランザクション外: イベント発行（消失リスクあり）

After（フェーズ 2）:
  同一トランザクション内:
    ├── 業務データ保存（lot テーブル）
    └── イベント保存（outbox_events テーブル）
```

### Python での変更点

```python
# Before: 直接 publish
publish(LotManufacturingCompleted(lot_id=lot_id, date=body.date))

# After: トランザクション内で outbox に INSERT
import json
from sqlalchemy import text

async def complete_manufacturing(lot_id: str, body: CompleteMfgInput, session: AsyncSession):
    async with session.begin():
        lot = await do_complete_manufacturing(lot_id, body, session)
        await session.execute(
            text(
                "INSERT INTO outbox_events (event_type, payload) "
                "VALUES (:type, :payload::jsonb)"
            ),
            {
                "type": "LotManufacturingCompleted",
                "payload": json.dumps({"lot_id": lot_id, "date": str(body.date)}),
            },
        )
    return lot
```

### フェーズ 2 の完了条件

3. 製造完了APIを叩くと、`outbox_events` テーブルにイベントが記録されること:

```bash
curl -X POST -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/lots/2024-A-001/complete-manufacturing \
  -d '{"date":"2026-04-22"}'

# DB確認
docker compose exec db psql -U app -d sales_management \
  -c "SELECT id, event_type, status, created_at FROM outbox_events ORDER BY id DESC LIMIT 1;"
# → id=1, event_type=LotManufacturingCompleted, status=pending, created_at=...
```

4. `lot` テーブルと `outbox_events` テーブルの両方にデータがあること（同一トランザクション）

---

## フェーズ 3: バックグラウンドポーリング

### やること

バックグラウンドで `outbox_events` テーブルの `pending` レコードをポーリングし、イベントハンドラを実行する。処理完了後に `status` を `processed` に更新する。

### 構造

```
バックグラウンドタスク（5秒ごとにポーリング）:
  1. SELECT * FROM outbox_events WHERE status = 'pending' ORDER BY id LIMIT 10
  2. 各イベントに対してハンドラを実行
  3. UPDATE outbox_events SET status = 'processed', processed_at = NOW() WHERE id = ?
  4. ハンドラが失敗した場合: status = 'failed'（手動確認用）
```

### Python での実装イメージ

```python
# backend/src/events/outbox_processor.py
import structlog
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession

logger = structlog.get_logger()


async def process_outbox(session: AsyncSession) -> None:
    result = await session.execute(
        text(
            "SELECT id, event_type, payload FROM outbox_events "
            "WHERE status = 'pending' ORDER BY id LIMIT 10 FOR UPDATE SKIP LOCKED"
        )
    )
    rows = result.fetchall()
    for row in rows:
        event_id, event_type, payload = row.id, row.event_type, row.payload
        try:
            logger.info("processing outbox event", event_type=event_type, event_id=event_id)
            _dispatch(event_type, payload)
            await session.execute(
                text(
                    "UPDATE outbox_events SET status='processed', processed_at=NOW() "
                    "WHERE id=:id"
                ),
                {"id": event_id},
            )
            logger.info("outbox event processed", event_id=event_id)
        except Exception as exc:
            logger.error("outbox event failed", event_id=event_id, error=str(exc))
            await session.execute(
                text("UPDATE outbox_events SET status='failed' WHERE id=:id"),
                {"id": event_id},
            )
    await session.commit()


def _dispatch(event_type: str, payload: dict) -> None:
    if event_type == "LotManufacturingCompleted":
        logger.info("lot manufacturing completed (async)", **payload)
```

```python
# FastAPI lifespan でバックグラウンドタスク起動 (src/main.py)
import asyncio
from contextlib import asynccontextmanager
from src.database import AsyncSessionLocal
from src.events.outbox_processor import process_outbox


async def _outbox_loop() -> None:
    while True:
        async with AsyncSessionLocal() as session:
            await process_outbox(session)
        await asyncio.sleep(5)


@asynccontextmanager
async def lifespan(app):
    task = asyncio.create_task(_outbox_loop())
    yield
    task.cancel()


app = FastAPI(lifespan=lifespan)
```

### フェーズ 3 の完了条件

5. 製造完了APIを叩いた後、数秒待つと `outbox_events` の `status` が `processed` に変わること:

```bash
curl -X POST -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/lots/2024-A-001/complete-manufacturing \
  -d '{"date":"2026-04-22"}'

# 直後: pending
docker compose exec db psql -U app -d sales_management \
  -c "SELECT status FROM outbox_events ORDER BY id DESC LIMIT 1;"

# 5秒後: processed
docker compose exec db psql -U app -d sales_management \
  -c "SELECT status, processed_at FROM outbox_events ORDER BY id DESC LIMIT 1;"
```

6. イベントハンドラの実行ログが出力されていること:

```
{"level":"info","event":"processing outbox event","event_type":"LotManufacturingCompleted","event_id":1}
{"level":"info","event":"outbox event processed","event_id":1}
```

7. APIのレスポンスタイムがフェーズ 1 と変わらないこと（ポーリングはバックグラウンドなので、APIの速度に影響しない）

---

## 次のステップ

Step 7が完了したら [Step 8: 監査ログ + 分散トレーシング](./step08.md) へ進む。
