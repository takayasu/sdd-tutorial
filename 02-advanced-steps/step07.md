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
- Spring の `ApplicationEvent` + `@EventListener` に相当する仕組みを実現する

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

### F# での実装イメージ

```fsharp
// イベント型
type DomainEvent =
    | LotManufacturingCompleted of lotId: string * date: DateOnly
    | ContractSigned of contractId: string

// イベントバス（関数のリストを保持）
type EventBus = {
    mutable Handlers: (DomainEvent -> unit) list
}

let eventBus = { Handlers = [] }

let subscribe (handler: DomainEvent -> unit) =
    eventBus.Handlers <- handler :: eventBus.Handlers

let publish (event: DomainEvent) =
    eventBus.Handlers |> List.iter (fun h -> h event)

// ハンドラ登録（アプリ起動時）
subscribe (fun event ->
    match event with
    | LotManufacturingCompleted (lotId, date) ->
        printfn $"[Event] Lot {lotId} manufacturing completed on {date}"
    | _ -> ())

// APIハンドラ内で使用
let completeManufacturingHandler lotId date =
    let result = completeManufacturing db lotId date
    match result with
    | Ok lot ->
        publish (LotManufacturingCompleted (lotId, date))
        Ok lot
    | Error e -> Error e
```

### Kotlin での実装イメージ

```kotlin
// イベント型
sealed interface DomainEvent {
    data class LotManufacturingCompleted(val lotId: String, val date: LocalDate) : DomainEvent
    data class ContractSigned(val contractId: String) : DomainEvent
}

// イベントバス
object EventBus {
    private val handlers = mutableListOf<(DomainEvent) -> Unit>()
    fun subscribe(handler: (DomainEvent) -> Unit) { handlers.add(handler) }
    fun publish(event: DomainEvent) { handlers.forEach { it(event) } }
}

// ハンドラ登録
EventBus.subscribe { event ->
    when (event) {
        is DomainEvent.LotManufacturingCompleted ->
            println("[Event] Lot ${event.lotId} manufacturing completed on ${event.date}")
        else -> {}
    }
}
```

### フェーズ 1 の完了条件

1. ロットの製造完了APIを叩くと、ログにイベントが記録されること:

```bash
curl -X POST -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/api/lots/2024-A-001/complete-manufacturing \
  -d '{"date":"2026-04-22"}'
# → 200 OK

# ログ:
# {"message":"[Event] Lot 2024-A-001 manufacturing completed on 2026-04-22"}
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

### F# での変更点

```fsharp
// Before: 直接 publish
publish (LotManufacturingCompleted (lotId, date))

// After: トランザクション内で outbox に INSERT
use conn = db.OpenConnection()
use tx = conn.BeginTransaction()
saveLot conn updatedLot
conn |> Db.newCommand
    "INSERT INTO outbox_events (event_type, payload) VALUES (@type, @payload::jsonb)"
|> Db.addParam "type" "LotManufacturingCompleted"
|> Db.addParam "payload" (JsonSerializer.Serialize {| lotId = lotId; date = date |})
|> Db.exec
tx.Commit()
```

### フェーズ 2 の完了条件

3. 製造完了APIを叩くと、`outbox_events` テーブルにイベントが記録されること:

```bash
curl -X POST -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/api/lots/2024-A-001/complete-manufacturing \
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

### F# での実装イメージ

```fsharp
// IHostedService でバックグラウンドタスク
type OutboxProcessor(db: NpgsqlDataSource) =
    interface IHostedService with
        member _.StartAsync(ct) =
            Task.Run(fun () -> async {
                while not ct.IsCancellationRequested do
                    let events = fetchPendingEvents db 10
                    for event in events do
                        try
                            processEvent event
                            markProcessed db event.Id
                        with ex ->
                            markFailed db event.Id ex.Message
                    do! Async.Sleep 5000
            } |> Async.StartAsTask :> Task)
        member _.StopAsync(_) = Task.CompletedTask
```

### Kotlin での実装イメージ

```kotlin
// CoroutineScope でバックグラウンドタスク
fun CoroutineScope.startOutboxProcessor(db: Database) = launch {
    while (isActive) {
        val events = transaction(db) {
            OutboxEvents.selectAll()
                .where { OutboxEvents.status eq "pending" }
                .orderBy(OutboxEvents.id)
                .limit(10)
                .toList()
        }
        for (event in events) {
            try {
                processEvent(event)
                transaction(db) {
                    OutboxEvents.update({ OutboxEvents.id eq event[OutboxEvents.id] }) {
                        it[status] = "processed"
                        it[processedAt] = Clock.System.now()
                    }
                }
            } catch (e: Exception) {
                transaction(db) {
                    OutboxEvents.update({ OutboxEvents.id eq event[OutboxEvents.id] }) {
                        it[status] = "failed"
                    }
                }
            }
        }
        delay(5000)
    }
}
```

### フェーズ 3 の完了条件

5. 製造完了APIを叩いた後、数秒待つと `outbox_events` の `status` が `processed` に変わること:

```bash
# API実行
curl -X POST -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/api/lots/2024-A-001/complete-manufacturing \
  -d '{"date":"2026-04-22"}'

# 直後: pending
docker compose exec db psql -U app -d sales_management \
  -c "SELECT status FROM outbox_events ORDER BY id DESC LIMIT 1;"
# → pending

# 5秒後: processed
docker compose exec db psql -U app -d sales_management \
  -c "SELECT status, processed_at FROM outbox_events ORDER BY id DESC LIMIT 1;"
# → processed, 2026-04-22 10:00:05
```

6. イベントハンドラの実行ログが出力されていること:

```
{"message":"Processing outbox event","eventType":"LotManufacturingCompleted","eventId":1}
{"message":"Outbox event processed","eventId":1}
```

7. APIのレスポンスタイムがフェーズ 1 と変わらないこと（ポーリングはバックグラウンドなので、APIの速度に影響しない）

---

## 次のステップ

Step 7が完了したら [Step 8: 監査ログ + 分散トレーシング](./step08.md) へ進む。
