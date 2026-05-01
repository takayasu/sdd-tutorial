# Step 20: DAST（OWASP ZAP）

## 目的

### これは何か

DAST（Dynamic Application Security Testing）を導入する。DASTとは、実際に動いているアプリケーションに対して攻撃を模擬し、脆弱性を検出するツール。ここではOWASP ZAPを使う。

Step 11のSASTが「コードを読んで脆弱性を見つける」のに対し、DASTは「実際にリクエストを送って脆弱性を見つける」。

### なぜやるのか

- SASTでは見つけられない脆弱性がある（例：セキュリティヘッダの欠如、CORS設定ミス、エラーメッセージからの情報漏洩）
- 実際の攻撃者と同じ視点でアプリケーションをテストできる
- OpenAPI定義を渡すことで、全エンドポイントを自動的にスキャンしてくれる

### 何がうれしいのか

- セキュリティの専門知識がなくても、ツールが自動的に脆弱性を発見してくれる
- 「このAPIは外部に公開しても安全か？」に客観的に答えられる
- CIに組み込むことで、新しいエンドポイントを追加するたびに自動スキャンされる
- これでCIパイプラインが完成。全ての品質・セキュリティチェックが自動化された状態になる

## 完了条件

**ZAP は `FAIL-NEW: 0` だけでは不十分。`WARN-NEW: 0` も必須**とし、`./ci.sh` 全体が exit 0 で終わることを完了条件とする。Step 1 で導入したセキュリティヘッダミドルウェアと、各 step で積んだ verify セクションが揃っていれば、以下 4 件の典型警告は最初から踏まずに済むはずである。

### よくある ZAP 警告と対処

| Rule | 何の警告 | 対処（どこで） |
|---|---|---|
| `[10021] X-Content-Type-Options Header Missing` | 全レスポンスに `nosniff` がない | **Step 1 のミドルウェアで対応済み**。verify で `curl -sI` チェックするので落ちない |
| `[90004] Cross-Origin-Resource-Policy Header Missing` | 全レスポンスに `CORP` がない | **Step 1 のミドルウェアで対応済み** |
| `[100001] Unexpected Content-Type: text/csv` | `/lots/export` の content-type が openapi に未記載 | openapi.yaml で `text/csv; charset=windows-31j` を明記 |
| `[100000] 502 Server Error` | `/api/external/price-check?lotId=...` で上流の不正値を素通し | ハンドラ先頭で lotId のフォーマット検証して 400 を先に返す (上流に invalid を流さない) |

### 検証

```bash
# 1. アプリケーションを起動
$ dotnet run --project src/SalesManagement &   # F#の場合
$ gradle run &                                  # Kotlinの場合

# 2. ZAPスキャン実行 — exit code を厳密に確認
$ docker run --rm --network host \
  -v $(pwd)/openapi.yaml:/zap/openapi.yaml \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-api-scan.py \
    -t /zap/openapi.yaml \
    -f openapi \
    -z "-config api.disablekey=true"
...
FAIL-NEW: 0   FAIL-INPROG: 0   WARN-NEW: 0   WARN-INPROG: 0   INFO: 0   IGNORE: 0   PASS: 19
$ echo $?
0     # ← 0 でなければ NG。WARN があれば exit 2

# 3. /lots/export の Content-Type が openapi の宣言と一致
$ curl -s -o /dev/null -w "%{content_type}\n" http://localhost:5000/lots/export
text/csv; charset=windows-31j

# 4. 外部 API の lotId フォーマット検証 (上流に到達する前に 400)
$ curl -s -o /dev/null -w "%{http_code} %{content_type}\n" \
    "http://localhost:5000/api/external/price-check?lotId=invalid"
400 application/problem+json

# 5. CI全体が通ること（DAST含む）
$ ZAP_ENABLED=1 ./ci.sh
=== マイグレーション ===
...
=== DAST (OWASP ZAP) ===
FAIL-NEW: 0   WARN-NEW: 0   PASS: 19
=== verify (smoke) ===
PASS /health
... (Step 1 / 7 / 14 / 15 / 18 の PASS 群すべて) ...
PASS lots-export-content-type
PASS external-pricecheck-validates-lotid
=== CI完了 ===
$ echo $?
0
```

### `ci.sh` verify セクションへ追記

```
PASS lots-export-content-type
PASS external-pricecheck-validates-lotid
PASS zap-warn-new-zero
```

---

## OWASP ZAP（共通）

### 1. OpenAPI定義の準備

DASTはAPIのエンドポイントを知る必要がある。OpenAPI（Swagger）定義を用意する。

```yaml
# openapi.yaml（例：在庫ロットAPI部分）
openapi: 3.0.3
info:
  title: Sales Management API
  version: 1.0.0
paths:
  /health:
    get:
      responses:
        '200':
          description: OK
  /lots/{id}:
    get:
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: ロット取得
  /lots/{id}/complete-manufacturing:
    post:
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                date:
                  type: string
                  format: date
      responses:
        '200':
          description: 製造完了
        '400':
          description: エラー
```

### 2. ZAP APIスキャン実行

```bash
# アプリケーションを起動した状態で実行
# F#
cd ../sales-management/apps/api-fsharp
dotnet run --project src/SalesManagement &
APP_PID=$!
sleep 5

# Kotlin
cd kotlin
gradle run &
APP_PID=$!
sleep 10

# ZAPスキャン実行（Docker）
docker run --rm --network host \
  -v $(pwd)/openapi.yaml:/zap/openapi.yaml \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-api-scan.py \
    -t /zap/openapi.yaml \
    -f openapi \
    -r /zap/report.html \
    -w /zap/report.md \
    -z "-config api.disablekey=true"

# アプリケーション停止
kill $APP_PID
```

### 3. 結果の確認

ZAPは以下のリスクレベルで報告する：

| レベル | 対応 |
|---|---|
| High | CI失敗。即修正 |
| Medium | 警告。次スプリントで対応 |
| Low | 情報。対応任意 |
| Informational | 無視可 |

### 4. ci.sh への追加

```bash
echo "=== DAST (OWASP ZAP) ==="
# アプリ起動
dotnet run --project src/SalesManagement &  # or: gradle run &
APP_PID=$!
sleep 5

# ZAPスキャン
docker run --rm --network host \
  -v $(pwd)/openapi.yaml:/zap/openapi.yaml \
  -v $(pwd)/ci-results:/zap/results \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-api-scan.py \
    -t /zap/openapi.yaml \
    -f openapi \
    -r /zap/results/zap-report.html \
    -z "-config api.disablekey=true" \
    -l WARN

ZAP_EXIT=$?
kill $APP_PID

if [ $ZAP_EXIT -ne 0 ]; then
    echo "DAST: 高リスク脆弱性が検出されました"
    exit 1
fi
```

---

## よく検出される脆弱性と対策

| 脆弱性 | 対策 |
|---|---|
| Missing Security Headers | `X-Content-Type-Options`, `X-Frame-Options` 等をレスポンスヘッダに追加 |
| CORS Misconfiguration | 許可するオリジンを明示的に設定 |
| Information Disclosure | エラーレスポンスにスタックトレースを含めない |
| Cookie Without Secure Flag | 本番ではSecure/HttpOnly/SameSite設定 |

### F# でのヘッダ追加例

```fsharp
let securityHeaders : HttpHandler =
    setHttpHeader "X-Content-Type-Options" "nosniff"
    >=> setHttpHeader "X-Frame-Options" "DENY"
    >=> setHttpHeader "X-XSS-Protection" "1; mode=block"
```

### Kotlin でのヘッダ追加例

```kotlin
install(DefaultHeaders) {
    header("X-Content-Type-Options", "nosniff")
    header("X-Frame-Options", "DENY")
    header("X-XSS-Protection", "1; mode=block")
}
```

---

## PoC完了 (Phase 1)

全20ステップが完了。以下が達成されている状態：

1. ✅ 環境構築（Docker, .NET, Gradle）
2. ✅ F#（Giraffe）/ Kotlin（Ktor）でREST API実装
3. ✅ PostgreSQL + マイグレーション（DbUp / Flyway）
4. ✅ domain-model-sales-management.md の全behaviorがAPI化
5. ✅ PBTで状態遷移の正しさを検証
6. ✅ CIパイプライン完成（ビルド→フォーマット→リンター→テスト→SAST→SCA→DAST）
7. ✅ 品質ダッシュボード（Grafana / SonarQube）

Phase 1 のCIは **「人間がCIを読み、人間が修正する」** 前提で組まれている。

## Phase 2 への橋渡し

次の Phase 2 では同じパイプラインを **「AIエージェントが自走する」ためのハーネス** へ昇格させる：

- 全ツールの結果を **SARIF** に統一しエージェント可読にする（Step 21）
- **ミューテーションテスト / ArchUnit / Pact** で品質ゲートを多層化する（Step 22-24）
- **SBOM / Renovate** で依存を継続管理する（Step 25-26）
- **OpenTelemetry** でエージェント自身の動作を観測する（Step 27）
- **AGENTS.md自動更新 → マルチエージェント → 完全自律RALPHループ** と段階的に自走化する（Step 28-30）

### Phase 1 の振り返り課題（任意）

Phase 2 へ進む前にやっておくと有益：

- ビルドサイクル時間の計測・比較
- F# vs Kotlin の開発体験の比較レポート作成
- 本番導入に向けた技術選定の判断

---

## 次のステップ

Step 20が完了したら [Step 21: SARIF統一出力](./step21.md) へ進む。
