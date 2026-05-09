# Step 20: DAST（OWASP ZAP）

## 目的

### これは何か

DAST（Dynamic Application Security Testing）を導入する。DASTとは、実際に動いているアプリケーションに対して攻撃を模擬し、脆弱性を検出するツール。ここではOWASP ZAPを使う。

Step 11のSASTが「コードを読んで脆弱性を見つける」のに対し、DASTは「実際にリクエストを送って脆弱性を見つける」。

### なぜやるのか

- SASTでは見つけられない脆弱性がある（例：セキュリティヘッダの欠如、CORS設定ミス、エラーメッセージからの情報漏洩）
- 実際の攻撃者と同じ視点でアプリケーションをテストできる
- FastAPI は `/openapi.json` を自動生成するため、全エンドポイントを ZAP に自動的にスキャンさせやすい

### 何がうれしいのか

- セキュリティの専門知識がなくても、ツールが自動的に脆弱性を発見してくれる
- 「このAPIは外部に公開しても安全か？」に客観的に答えられる
- CIに組み込むことで、新しいエンドポイントを追加するたびに自動スキャンされる
- これでCIパイプラインが完成。全ての品質・セキュリティチェックが自動化された状態になる

## 完了条件

**ZAP は `FAIL-NEW: 0` だけでは不十分。`WARN-NEW: 0` も必須**とし、`./ci.sh` 全体が exit 0 で終わることを完了条件とする。Step 1 で導入したセキュリティヘッダミドルウェアと、各 step で積んだ verify セクションが揃っていれば、以下の典型警告は最初から踏まずに済む。

### よくある ZAP 警告と対処

| Rule | 何の警告 | 対処（どこで） |
|---|---|---|
| `[10021] X-Content-Type-Options Header Missing` | 全レスポンスに `nosniff` がない | **Step 1 のミドルウェアで対応済み**。verify で `curl -sI` チェックするので落ちない |
| `[90004] Cross-Origin-Resource-Policy Header Missing` | 全レスポンスに `CORP` がない | **Step 1 のミドルウェアで対応済み** |
| `[100001] Unexpected Content-Type` | content-type が openapi に未記載 | FastAPI の response_class / response_model で明記 |
| `[10038] Content Security Policy Missing` | CSP ヘッダがない | Step 1 ミドルウェアに追加 |

### 検証

```bash
# 1. アプリケーションを起動
$ cd backend && uv run uvicorn src.main:app --host 0.0.0.0 --port 8000 &
$ APP_PID=$!
$ sleep 3

# 2. FastAPI の OpenAPI 定義が取得できること
$ curl -s http://localhost:8000/openapi.json | python -c "import sys,json; d=json.load(sys.stdin); print(d['info']['title'])"
Sales Management API

# 3. ZAPスキャン実行 — exit code を厳密に確認
$ docker run --rm --network host \
  -v $(pwd)/ci-results:/zap/results \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-api-scan.py \
    -t http://localhost:8000/openapi.json \
    -f openapi \
    -r /zap/results/zap-report.html \
    -z "-config api.disablekey=true"
...
FAIL-NEW: 0   FAIL-INPROG: 0   WARN-NEW: 0   WARN-INPROG: 0   INFO: 0   IGNORE: 0   PASS: 19
$ echo $?
0     # ← 0 でなければ NG。WARN があれば exit 2

# 4. セキュリティヘッダが付いていること
$ curl -sI http://localhost:8000/health | grep -i "x-content-type"
x-content-type-options: nosniff

# 5. CI全体が通ること（DAST含む）
$ ZAP_ENABLED=1 ./ci.sh
=== マイグレーション ===
...
=== DAST (OWASP ZAP) ===
FAIL-NEW: 0   WARN-NEW: 0   PASS: 19
=== CI完了 ===
$ echo $?
0
```

---

## FastAPI セキュリティヘッダミドルウェア（Step 1 の確認）

Step 1 で追加済み。ZAP の警告を防ぐために以下が含まれていることを確認する：

```python
# src/main.py（Step 1 から継続）
from fastapi import FastAPI
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["Cross-Origin-Resource-Policy"] = "same-origin"
        response.headers["Content-Security-Policy"] = "default-src 'none'"
        response.headers["Referrer-Policy"] = "no-referrer"
        return response

app = FastAPI(title="Sales Management API")
app.add_middleware(SecurityHeadersMiddleware)
```

---

## ZAP APIスキャン実行

FastAPI は起動時に `/openapi.json` を自動生成するため、`-t` に直接 URL を指定できる：

```bash
# アプリケーションを起動した状態で実行
cd backend
uv run uvicorn src.main:app --host 0.0.0.0 --port 8000 &
APP_PID=$!
sleep 3

# ZAPスキャン実行（Docker）
docker run --rm --network host \
  -v $(pwd)/../ci-results:/zap/results \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-api-scan.py \
    -t http://localhost:8000/openapi.json \
    -f openapi \
    -r /zap/results/zap-report.html \
    -w /zap/results/zap-report.md \
    -z "-config api.disablekey=true"

ZAP_EXIT=$?

# アプリケーション停止
kill $APP_PID
```

### 結果の確認

ZAPは以下のリスクレベルで報告する：

| レベル | 対応 |
|---|---|
| High | CI失敗。即修正 |
| Medium | 警告。次スプリントで対応 |
| Low | 情報。対応任意 |
| Informational | 無視可 |

---

## ci.sh への追加（最終構成）

```bash
echo "=== DAST (OWASP ZAP) ==="
cd backend
uv run uvicorn src.main:app --host 0.0.0.0 --port 8000 &
APP_PID=$!
sleep 3

docker run --rm --network host \
  -v "$(pwd)/../ci-results:/zap/results" \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-api-scan.py \
    -t http://localhost:8000/openapi.json \
    -f openapi \
    -r /zap/results/zap-report.html \
    -z "-config api.disablekey=true" \
    -l WARN

ZAP_EXIT=$?
kill $APP_PID
cd ..

if [ $ZAP_EXIT -ne 0 ]; then
    echo "DAST: 高リスク脆弱性または警告が検出されました"
    exit 1
fi
echo "PASS zap-warn-new-zero"
```

---

## PoC完了 (Phase 1)

全20ステップが完了。以下が達成されている状態：

1. ✅ 環境構築（Docker, Python/uv, Node.js/pnpm）
2. ✅ FastAPI で REST API 実装 + React フロントエンド
3. ✅ PostgreSQL + Alembic マイグレーション
4. ✅ `domain-model-sales-management.md` の全 behavior（23個）が API 化
5. ✅ hypothesis (Python PBT) + fast-check (TypeScript PBT) で状態遷移の正しさを検証
6. ✅ CI パイプライン完成（マイグレーション → フォーマット → リンター → 型チェック → テスト → SAST → SCA → DAST）
7. ✅ 品質ダッシュボード（Grafana + radon + pytest-cov JSON）

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

- CIビルドサイクル時間の計測
- 各ステップでの学び・気づきのレポート作成
- 本番導入に向けた技術選定の判断

---

## 次のステップ

Step 20が完了したら [Step 21: SARIF統一出力](./step21.md) へ進む。
