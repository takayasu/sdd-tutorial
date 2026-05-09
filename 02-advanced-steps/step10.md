# Step 10: CSV/ファイルエクスポート

## 目的

### これは何か

ロットや販売案件のデータを CSV ファイルとしてダウンロードするエンドポイントを実装する。日本の業務システムでは Windows-31J（Shift_JIS）エンコーディングの CSV エクスポートが日常的に使われる。

### なぜやるのか

- 経理部門への報告、外部システムへのデータ連携、監査対応など、CSV エクスポートは業務システムの基本機能
- 大量データのエクスポートではメモリに全件載せずにストリーミングで出力する必要がある

### 何がうれしいのか

- ブラウザや curl で `/lots/export?format=csv` を叩くと、CSV ファイルがダウンロードされる
- 日本語が文字化けしない（Windows-31J エンコーディング対応）
- 10万件のデータでもメモリを圧迫しない（`StreamingResponse` + ジェネレータ）

## 完了条件

1. CSV エクスポートエンドポイントが動作すること:

```bash
TOKEN=$(./scripts/get-token.sh test-operator)

curl -H "Authorization: Bearer $TOKEN" \
  -o lots.csv \
  "http://localhost:8000/lots/export?format=csv"

head -3 lots.csv
# → "ロット番号","事業部","状態","製造完了日"
# → "2024-A-001","1","manufactured","2024-04-01"
# → "2024-A-002","1","manufacturing",""
```

2. レスポンスヘッダが正しいこと:

```bash
curl -v -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/lots/export?format=csv" 2>&1 | grep -i content
# → Content-Type: text/csv; charset=windows-31j
# → Content-Disposition: attachment; filename="lots_20260422.csv"
```

3. 日本語が文字化けしないこと（Excel で開いて確認、または `nkf` コマンドで確認）:

```bash
nkf --guess lots.csv
# → Shift_JIS (or CP932/Windows-31J)
```

4. 大量データ（1万件以上）でもタイムアウトせずにダウンロードできること

5. 検索条件を指定してエクスポートできること:

```bash
curl -H "Authorization: Bearer $TOKEN" \
  -o manufactured_lots.csv \
  "http://localhost:8000/lots/export?format=csv&status=manufactured"
# → 製造完了ロットのみが出力される
```

---

## 実装ガイド

### Python / FastAPI (Backend)

| 要素 | 実装方法 |
|---|---|
| CSV 生成 | Python 標準 `csv` モジュール + `io.StringIO` |
| エンコーディング | `.encode("cp932")` — Windows-31J（CP932）でバイト列に変換 |
| ストリーミング | `StreamingResponse` + 非同期ジェネレータ関数 |

Python の `csv` モジュールは標準ライブラリに含まれるため、追加パッケージは不要。

#### `backend/src/routers/export.py`

```python
import csv
import io
from datetime import datetime, timezone
from typing import AsyncGenerator

from fastapi import APIRouter, Depends
from fastapi.responses import StreamingResponse
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession

from src.database import get_session
from src.middleware.auth import require_auth

router = APIRouter()

_HEADERS = ["ロット番号", "事業部", "状態", "製造完了日"]


async def _lot_csv_rows(
    status: str | None,
    session: AsyncSession,
) -> AsyncGenerator[bytes, None]:
    query = "SELECT lot_number, division, status, manufactured_date FROM lot"
    params: dict = {}
    if status:
        query += " WHERE status = :status"
        params["status"] = status
    query += " ORDER BY lot_number"

    result = await session.execute(text(query), params)

    buf = io.StringIO()
    writer = csv.writer(buf, quoting=csv.QUOTE_ALL)
    writer.writerow(_HEADERS)
    yield buf.getvalue().encode("cp932", errors="replace")

    async for row in result:
        buf = io.StringIO()
        writer = csv.writer(buf, quoting=csv.QUOTE_ALL)
        writer.writerow([
            row.lot_number,
            row.division,
            row.status,
            str(row.manufactured_date) if row.manufactured_date else "",
        ])
        yield buf.getvalue().encode("cp932", errors="replace")


@router.get("/lots/export")
async def export_lots(
    format: str = "csv",
    status: str | None = None,
    claims: dict = Depends(require_auth),
    session: AsyncSession = Depends(get_session),
) -> StreamingResponse:
    today = datetime.now(timezone.utc).strftime("%Y%m%d")
    filename = f"lots_{today}.csv"

    return StreamingResponse(
        _lot_csv_rows(status, session),
        media_type="text/csv; charset=windows-31j",
        headers={"Content-Disposition": f'attachment; filename="{filename}"'},
    )
```

#### `backend/src/main.py` への追記

```python
from src.routers import export
app.include_router(export.router, tags=["export"])
```

### 確認のコツ

- Windows-31J で表現できない文字（一部の Unicode 文字）がデータに含まれる場合は `errors="replace"` で `?` に置換するか、事前にデータをサニタイズする
- UTF-8 BOM 付き CSV が必要な場合は、エンコーディングを `utf-8-sig` に変更する（Excel が自動判定できる）:
  ```python
  yield buf.getvalue().encode("utf-8-sig")
  ```
- Excel 向けには CSV よりも TSV（タブ区切り）の方が文字化けしにくい場合がある
- フロントエンドからはリンクに `Authorization` ヘッダを付けられないため、ダウンロード用の署名付き URL またはワンタイムトークン方式を検討する

### TypeScript / React (Frontend)

`fetch` で `Blob` として受け取り、ブラウザのダウンロードダイアログを開く:

```typescript
// src/lib/export.ts
export async function downloadLotsCSV(token: string, status?: string): Promise<void> {
  const params = new URLSearchParams({ format: "csv" })
  if (status) params.set("status", status)

  const res = await fetch(`/lots/export?${params}`, {
    headers: { Authorization: `Bearer ${token}` },
  })
  if (!res.ok) throw new Error("Export failed")

  const blob = await res.blob()
  const url = URL.createObjectURL(blob)
  const a = document.createElement("a")
  a.href = url
  a.download = `lots_${new Date().toISOString().slice(0, 10)}.csv`
  a.click()
  URL.revokeObjectURL(url)
}
```

---

## 02-advanced-steps 完了

Step 10 が完了すると、01-steps/ で構築したドメインモデル + CI 基盤に加え、業務システムとして本番運用に必要な全機能が揃います:

| カテゴリ | 機能 | Step |
|---|---|---|
| 設定・ログ | 環境別設定（pydantic-settings）、structlog JSON ログ、リクエストID | Step 1 |
| API品質 | RFC 9457 Problem Details、Pydantic バリデーション、楽観的ロック | Step 2 |
| セキュリティ | JWT認証（Keycloak + python-jose）、RBAC | Step 3 |
| 運用 | ヘルスチェック、FastAPI OpenAPI / Swagger UI | Step 4 |
| テスト | 統合テスト（pytest + httpx + testcontainers） | Step 5 |
| 外部連携 | httpx + tenacity リトライ + circuitbreaker | Step 6 |
| イベント駆動 | ドメインイベント、Outbox パターン（asyncio lifespan） | Step 7 |
| 監査・追跡 | 監査ログ（claims["sub"]）、OpenTelemetry + Jaeger | Step 8 |
| 本番運用 | slowapi レート制限、aiocache、uvicorn グレースフルシャットダウン | Step 9 |
| データ出力 | CSV エクスポート（Windows-31J / CP932 対応、StreamingResponse） | Step 10 |

次のステップとして:
- [02-batch-steps/](../02-batch-steps/) でバッチ処理基盤を構築する
- `ci.sh` を更新し、統合テストをCIに組み込む
- 本番デプロイに向けて、Dockerfile と k8s マニフェストを整備する
