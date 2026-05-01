# Step 2: エラーハンドリング + バリデーション

## 目的

### これは何か

APIのエラーレスポンスを RFC 9457 Problem Details 形式に統一し、入力バリデーションを Smart Constructor パターンで型安全に実装する。

### なぜやるのか

- Step 7までの実装では、エラーレスポンスの形式がエンドポイントごとにバラバラになりがち。`{"error":"..."}` だったり `{"message":"..."}` だったり、ステータスコードも不統一になる
- RFC 9457 Problem Details は「APIエラーレスポンスの標準形式」として広く採用されており、フロントエンドやAPI利用者が一貫した方法でエラーを処理できる
- Bean Validation（`@NotNull`, `@Size` 等のアノテーション）の代わりに、型レベルで不正な値を作れない設計にする。これにより「バリデーションを書き忘れる」というバグが構造的に発生しなくなる

### 何がうれしいのか

- 全てのAPIエラーが同じJSON構造で返るため、フロントエンド側のエラーハンドリングが1箇所で済む
- 「金額がマイナス」「数量がゼロ」といった不正な値が、型の生成時点で弾かれる。コンパイルが通った時点で、不正な値がドメインロジックに到達しないことが保証される
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

これは2つのターミナルから同じロットを操作することで確認できる。DB側は `UPDATE ... WHERE version = @expected RETURNING version` で実装し、affected rows が 0 なら競合と判定する。

---

## 実装ガイド

### エラーハンドリングの構造

```
APIリクエスト
  → ルーティング
    → ドメインロジック（Result / Either を返す）
      → Ok → 200 + JSONレスポンス
      → Error → DomainError を ProblemDetails に変換
    → 未処理例外 → グローバルエラーハンドラ → 500 ProblemDetails
```

### F#

| 要素 | 実装方法 |
|---|---|
| Problem Details | ASP.NET Core `Results.Problem()` or 自前の `ProblemDetails` レコード |
| グローバルエラーハンドラ | Giraffe `ErrorHandler` |
| Result → HTTP変換 | 共通の `toApiResponse` 関数 |
| Smart Constructor | `type PositiveAmount = private PositiveAmount of int` + `create` 関数 |
| エラー蓄積 | `FsToolkit.ErrorHandling` の `validation { }` CE（Applicative 合成） |

主な作業:
1. `DomainError` 型を定義（`NotFound`, `ValidationFailed`, `InvalidStateTransition`, `OptimisticLockConflict` 等）
2. `DomainError → ProblemDetails` の変換関数を作成（`OptimisticLockConflict` → `409 Conflict`）
3. 全APIハンドラで `Result` を返し、共通の変換関数でレスポンスに変換
4. `PositiveAmount`, `NonEmptyString` 等の Smart Constructor を作成
5. ロット作成APIの入力バリデーションを Smart Constructor で実装
6. 全テーブルに `version INT NOT NULL DEFAULT 1` カラムを追加（マイグレーション: F# `migrations/V004__add_version_column.sql` / Kotlin `V004__add_version_column.sql`）
7. UPDATE 文に `WHERE version = @expected` を追加し、affected rows = 0 なら `OptimisticLockConflict` を返す

### Kotlin

| 要素 | 実装方法 |
|---|---|
| Problem Details | `data class ProblemDetail(...)` + Ktor ContentNegotiation |
| グローバルエラーハンドラ | Ktor `StatusPages` plugin |
| Either → HTTP変換 | `ApplicationCall.respondEither()` 拡張関数 |
| Smart Constructor | `@JvmInline value class PositiveAmount private constructor(...)` + `create` |
| エラー蓄積 | Arrow `zipOrAccumulate` / `ValidatedNel` |

主な作業:
1. `DomainError` sealed interface を定義（`OptimisticLockConflict` を含む）
2. `StatusPages` plugin で例外 → ProblemDetails 変換を設定（`OptimisticLockConflict` → `409`）
3. `respondEither()` 拡張関数を作成し、全APIハンドラで使用
4. Arrow の `zipOrAccumulate` で複数バリデーションエラーを蓄積
5. ロット作成APIの入力バリデーションを Smart Constructor で実装
6. 全テーブルに `version INT NOT NULL DEFAULT 1` カラムを追加（マイグレーションファイルは上記 F# と同様）
7. Exposed の `update` で `where { version eq expected }` を追加し、更新件数 0 なら `OptimisticLockConflict`

---

## 次のステップ

Step 2が完了したら [Step 3: 認証・認可](./step03.md) へ進む。
