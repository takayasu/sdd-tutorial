# Step 2: エラーハンドリング + バリデーション

## 目的

### これは何か

APIのエラーレスポンスを RFC 9457 Problem Details 形式に統一し、入力バリデーションを Smart Constructor パターンで型安全に実装する。

### なぜやるのか

- Step 7までの実装では、エラーレスポンスの形式がエンドポイントごとにバラバラになりがち。`{"error":"..."}` だったり `{"message":"..."}` だったり、ステータスコードも不統一になる
- RFC 9457 Problem Details は「APIエラーレスポンスの標準形式」として広く採用されており、フロントエンドやAPI利用者が一貫した方法でエラーを処理できる
- `@field_validator` の代わりに、型レベルで不正な値を作れない設計にする。これにより「バリデーションを書き忘れる」というバグが構造的に発生しなくなる

### 何がうれしいのか

- 全てのAPIエラーが同じJSON構造で返るため、フロントエンド側のエラーハンドリングが1箇所で済む
- 「金額がマイナス」「数量がゼロ」といった不正な値が、型の生成時点で弾かれる。Pydantic のバリデーションが通った時点で、不正な値がドメインロジックに到達しないことが保証される
- 複数のバリデーションエラーを一度に返せる（「金額が不正」と「数量が不正」を同時に通知）

## 完了条件

### Problem Details の確認

1. 存在しないロットを取得しようとして、404レスポンスが Problem Details 形式で返ること:

```
GET /lots/9999-Z-999
→ 404 Not Found
Content-Type: application/problem+json

{
  "type": "not-found",
  "title": "Resource not found",
  "status": 404,
  "detail": "Lot 9999-Z-999 not found"
}
```

2. 不正な状態遷移を試みて、400レスポンスが Problem Details 形式で返ること:

```
POST /lots/2024-A-001/complete-shipping  (製造完了状態のロットに出荷完了を指示)
→ 400 Bad Request
Content-Type: application/problem+json

{
  "type": "invalid-state-transition",
  "title": "Invalid state transition",
  "status": 400,
  "detail": "Lot is not in shipping-instructed state"
}
```

3. サーバー内部エラーが発生しても、スタックトレースがレスポンスに含まれないこと（セキュリティ上重要）

### バリデーションの確認

1. 不正な値でロットを作成しようとして、バリデーションエラーが返ること:

```
POST /lots
{
  "lotNumber": {"year": -1, "location": "", "seq": 0},
  "quantity": -5
}
→ 400 Bad Request

{
  "type": "validation-error",
  "title": "Validation failed",
  "status": 400,
  "errors": [
    {"field": "lotNumber.year", "message": "Year must be positive"},
    {"field": "lotNumber.location", "message": "Location must not be empty"},
    {"field": "lotNumber.seq", "message": "Seq must be positive"},
    {"field": "quantity", "message": "Quantity must be positive"}
  ]
}
```

2. 複数のバリデーションエラーが一度に返ること（1つ目のエラーで止まらない）

### 確認のコツ

- 正常系だけでなく、意図的に不正なリクエストを送ってエラーレスポンスを確認する
- `Content-Type` ヘッダが `application/problem+json` であることを確認する（`curl -v` でヘッダを表示）
- 全てのエラーレスポンスが同じ構造（`type`, `title`, `status`, `detail`）を持つことを確認する

### 楽観的ロック（Optimistic Locking）の確認

複数ユーザーが同じロットを同時に更新した場合に、後から更新した方がエラーになることを確認する。

3. ロットを取得し、レスポンスに `version` フィールドが含まれること:

```
GET /lots/2024-A-001
→ 200 OK
{"lotNumber":"2024-A-001", "status":"manufactured", "version": 1, ...}
```

4. 正しい `version` を指定して更新すると成功し、`version` がインクリメントされること:

```
POST /lots/2024-A-001/complete-manufacturing
{"date": "2026-04-22", "version": 1}
→ 200 OK
{"status":"manufactured", "version": 2}
```

5. 古い `version` を指定して更新すると `409 Conflict` が返ること（別のユーザーが先に更新した場合）:

```
POST /lots/2024-A-001/instruct-shipping
{"deadline": "2026-05-01", "version": 1}   ← version が古い（現在は2）
→ 409 Conflict

{
  "type": "optimistic-lock-conflict",
  "title": "Resource was modified by another user",
  "status": 409,
  "detail": "Lot 2024-A-001 has been updated. Please reload and try again."
}
```

これは2つのターミナルから同じロットを操作することで確認できる。DB側は `UPDATE ... WHERE version = :expected RETURNING version` で実装し、affected rows が 0 なら競合と判定する。

---

## 実装ガイド

### エラーハンドリングの構造

```
APIリクエスト
  → ルーティング
    → ドメインロジック（raise DomainError）
      → 正常 → 200 + JSONレスポンス
      → DomainError → exception_handler で ProblemDetails に変換
    → Pydantic ValidationError → exception_handler → 400 ProblemDetails
    → 未処理例外 → グローバルエラーハンドラ → 500 ProblemDetails
```

### Python / FastAPI (Backend)

| 要素 | 実装方法 |
|---|---|
| Problem Details | `JSONResponse` + `media_type="application/problem+json"` |
| グローバルエラーハンドラ | `app.add_exception_handler(DomainError, handler)` |
| DomainError → HTTP変換 | ハンドラ内で `error_type` → `status` をマッピング |
| Smart Constructor | Pydantic `Annotated` + `Field(gt=0)` / `min_length=1` |
| エラー蓄積 | Pydantic `ValidationError`（全フィールドを一括検証） |

#### `backend/src/domain/errors.py`

```python
from enum import StrEnum


class DomainErrorType(StrEnum):
    NOT_FOUND = "not-found"
    INVALID_STATE_TRANSITION = "invalid-state-transition"
    VALIDATION_ERROR = "validation-error"
    OPTIMISTIC_LOCK_CONFLICT = "optimistic-lock-conflict"


class DomainError(Exception):
    def __init__(self, error_type: DomainErrorType, detail: str) -> None:
        self.error_type = error_type
        self.detail = detail
        super().__init__(detail)
```

#### `backend/src/middleware/error_handler.py`

```python
from fastapi import Request
from fastapi.responses import JSONResponse
from pydantic import ValidationError

from src.domain.errors import DomainError, DomainErrorType

_STATUS_MAP = {
    DomainErrorType.NOT_FOUND: 404,
    DomainErrorType.INVALID_STATE_TRANSITION: 400,
    DomainErrorType.VALIDATION_ERROR: 400,
    DomainErrorType.OPTIMISTIC_LOCK_CONFLICT: 409,
}

_TITLE_MAP = {
    DomainErrorType.NOT_FOUND: "Resource not found",
    DomainErrorType.INVALID_STATE_TRANSITION: "Invalid state transition",
    DomainErrorType.VALIDATION_ERROR: "Validation failed",
    DomainErrorType.OPTIMISTIC_LOCK_CONFLICT: "Resource was modified by another user",
}


async def domain_error_handler(request: Request, exc: DomainError) -> JSONResponse:
    status = _STATUS_MAP.get(exc.error_type, 400)
    return JSONResponse(
        status_code=status,
        media_type="application/problem+json",
        content={
            "type": exc.error_type,
            "title": _TITLE_MAP.get(exc.error_type, "Error"),
            "status": status,
            "detail": exc.detail,
        },
    )


async def validation_error_handler(request: Request, exc: ValidationError) -> JSONResponse:
    errors = [
        {"field": ".".join(str(loc) for loc in e["loc"]), "message": e["msg"]}
        for e in exc.errors()
    ]
    return JSONResponse(
        status_code=400,
        media_type="application/problem+json",
        content={
            "type": "validation-error",
            "title": "Validation failed",
            "status": 400,
            "errors": errors,
        },
    )


async def unhandled_error_handler(request: Request, exc: Exception) -> JSONResponse:
    return JSONResponse(
        status_code=500,
        media_type="application/problem+json",
        content={
            "type": "internal-server-error",
            "title": "An unexpected error occurred",
            "status": 500,
            "detail": "Please contact support.",
        },
    )
```

#### `backend/src/main.py` への追記

```python
from pydantic import ValidationError
from src.domain.errors import DomainError
from src.middleware.error_handler import (
    domain_error_handler,
    validation_error_handler,
    unhandled_error_handler,
)

app.add_exception_handler(DomainError, domain_error_handler)
app.add_exception_handler(ValidationError, validation_error_handler)
app.add_exception_handler(Exception, unhandled_error_handler)
```

#### Smart Constructor の例 (`backend/src/domain/models.py`)

```python
from typing import Annotated
from pydantic import BaseModel, Field


PositiveInt = Annotated[int, Field(gt=0)]
NonEmptyStr = Annotated[str, Field(min_length=1)]


class LotNumberInput(BaseModel):
    year: PositiveInt
    location: NonEmptyStr
    seq: PositiveInt


class CreateLotInput(BaseModel):
    lot_number: LotNumberInput
    quantity: PositiveInt
```

Pydantic は全フィールドを一括検証するため、`year`, `location`, `seq`, `quantity` が全て不正でも一度に全エラーが `ValidationError` に蓄積される。

#### 楽観的ロック (SQLAlchemy)

```python
from sqlalchemy import update
from src.domain.errors import DomainError, DomainErrorType


async def update_lot_status(
    lot_id: str, new_status: str, expected_version: int, session
) -> None:
    result = await session.execute(
        update(LotTable)
        .where(LotTable.id == lot_id, LotTable.version == expected_version)
        .values(status=new_status, version=LotTable.version + 1)
        .returning(LotTable.version)
    )
    if result.rowcount == 0:
        raise DomainError(
            DomainErrorType.OPTIMISTIC_LOCK_CONFLICT,
            f"Lot {lot_id} has been updated. Please reload and try again.",
        )
```

全テーブルに `version INTEGER NOT NULL DEFAULT 1` カラムを追加する（マイグレーション: `migrations/V004__add_version_column.sql`）。

### TypeScript / React (Frontend)

フロントエンドでは `application/problem+json` を統一的にハンドリングする。

```typescript
// src/lib/api-client.ts
export type ProblemDetail = {
  type: string
  title: string
  status: number
  detail?: string
  errors?: Array<{ field: string; message: string }>
}

export class ApiError extends Error {
  constructor(public readonly problem: ProblemDetail) {
    super(problem.detail ?? problem.title)
  }
}

async function fetchJson<T>(url: string, init?: RequestInit): Promise<T> {
  const res = await fetch(url, init)
  if (!res.ok) {
    const problem: ProblemDetail = await res.json()
    throw new ApiError(problem)
  }
  return res.json() as Promise<T>
}
```

---

## 次のステップ

Step 2が完了したら [Step 3: 認証・認可](./step03.md) へ進む。
