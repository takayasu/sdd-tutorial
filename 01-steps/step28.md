# Step 28: AGENTS.md自動更新 + security-review 統合（Phase 2: 最小RALPH）

## 目的

### これは何か

Claude Code の `Stop` フック（セッション終了時に発火）を使い、`ci-results/merged.sarif`（Step 21）から繰り返し出る失敗パターン（同じ ruleId が 3 回以上）を自動抽出し、自然言語の「教訓」として `AGENTS.md` の `## 失敗から学んだこと（自動生成）` セクションに追記する。次回エージェントを起動すると `AGENTS.md` がコンテキストとして読まれ、同じミスを繰り返さない。

加えて、**`/security-review` skill を ci.sh に組み込み**、自動ツール (gitleaks/pip-audit/bandit/ZAP) では拾えない種類の問題 — 例えば以下のような H/H/M/M レベルのレビュー観点 — も SARIF 化して同じ「教訓化」ループに乗せる:

| 例 | 何のレビューが要るか |
|---|---|
| 開発時のみ存在すべき public エンドポイント (`/test/slow` 等) を本番でも露出させていないか | デプロイ環境とエンドポイント可視性 |
| Outbox / queue の claim が non-atomic で、複数ワーカー時に二重 publish しないか | 並行処理パターン |
| `except Exception as e: return JSONResponse({"detail": str(e)})` でスタックトレースを漏らしていないか | エラー応答の情報漏洩 |
| ID の表記ゆれ (`2024-A-1` vs `2024-A-001`) がキャッシュキーやルックアップで吸収されているか | データモデル正規化 |

これが **RALPH ループの最小プロトタイプ**（Phase 2.1: 単一エージェント自己改善）。

### なぜやるのか

- 人間の retrospective（振り返り）は手動で時間がかかる。エージェントが学べる形で自動化する
- 「コンテキストを毎回リセットする」RALPH の原則に従いつつ、外部メモリ（AGENTS.md）に学習を蓄積することで、リセットしても知識が消えない
- Step 21 (SARIF) と Step 27 (OTel) の観測基盤の上に初めて成立する自己改善ループ
- **静的ツールで拾えない種類のバグはレビュー眼でしか取れない**。`/security-review` skill の出力も SARIF として merged.sarif に流し込めば、ツール検出と同じ経路で AGENTS.md へ蓄積でき、次のラウンドで予防的に避けられる

### 何がうれしいのか

- 「前回 AWS キーをハードコードした」「bandit の B105 を頻発した」が次回コンテキストに自動的に乗る
- 人手の retrospective が不要になり、エージェントが自分で学ぶ
- レビューで発見した「DoS 可能なテストエンドポイント露出」「outbox 二重発行」「エラー情報漏洩」「ID 正規化漏れ」のような種類の発見も、次のラウンドで「コードを書く前に避ける」ように学習される
- Step 29 のマルチエージェント、Step 30 の完全自律ループの土台になる

## 完了条件

```bash
# 1. 故意に失敗を作る
$ cat >> backend/src/demo.py <<'EOF'
# demo: intentional secret (for RALPH loop test)
AWS_KEY_1 = "AKIAIOSFODNN7EXAMPLE"
AWS_KEY_2 = "AKIAJ7Q3HOY77LHEXMPL"
AWS_KEY_3 = "AKIA3X4Y5Z6789EXAMPL"
AWS_KEY_4 = "AKIAQQRRZZXXTTYYUUEX"
EOF
$ git add backend/src/demo.py && git commit -m "demo: intentional secret"

# 2. CI 実行（gitleaks が拾う）
$ ./ci.sh
…
Finding: hardcoded AWS access key
$ echo $?
1

# 3. Claude Code セッションを終わらせる → Stop フック発火
# (Claude Code が exit したタイミングで発火する)

# 4. AGENTS.md を確認
$ tail -20 AGENTS.md
…
## 失敗から学んだこと (自動生成)
- 2026-05-08 gitleaks.aws-access-token: 4回検出。AWS Access Token detected
- 2026-05-08 bandit.B105: 3回検出。Possible hardcoded password...

# 5. git diff で追記内容を確認
$ git diff AGENTS.md
+ ## 失敗から学んだこと (自動生成)
+ - 2026-05-08 gitleaks.aws-access-token: ...
…

# 6. security-review skill が ci.sh から呼ばれて SARIF を出力
$ SECURITY_REVIEW_ENABLED=1 ./ci.sh
…
=== security-review ===
ci-results/security-review.sarif: 2 results
=== sarif merge ===
ci-results/merged.sarif: 14 total findings (gitleaks: 0, pip_audit: 1, bandit: 8, zap: 3, security-review: 2)

# 7. security-review 由来の findings も AGENTS.md に教訓化される
$ tail -30 AGENTS.md | grep -i 'security-review'
- 2026-05-08 security-review.test-endpoint-public: 3回検出。/test/slow が ENV!=production でも include_router されている (DoS 経路)
- 2026-05-08 security-review.outbox-double-publish: 4回検出。fetchPending/markProcessed が claim 非アトミック
```

---

## 1. .claude/settings.json に Stop フックを追加

Step 27 で作成した `.claude/settings.json` に Stop フックを追記：

```json
{
  "hooks": {
    "SessionStart": [
      { "matcher": "", "hooks": [
        { "type": "command", "command": "python3 .claude/scripts/start-trace.py" }
      ]}
    ],
    "PostToolUse": [
      { "matcher": ".*", "hooks": [
        { "type": "command", "command": "python3 .claude/scripts/emit-otel.py" }
      ]}
    ],
    "Stop": [
      { "matcher": "", "hooks": [
        { "type": "command", "command": "python3 .claude/scripts/sarif-to-lessons.py" }
      ]}
    ]
  }
}
```

---

## 2. SARIF → 教訓変換スクリプト

`.claude/scripts/sarif-to-lessons.py`:

```python
#!/usr/bin/env python3
"""Stop フック。merged.sarif の失敗パターンを AGENTS.md に追記する。

ロジック:
  1. ci-results/merged.sarif を読む
  2. ruleId 別にカウント
  3. 3 回以上検出されたルールを「教訓」として追記
  4. 既に同日に追記済みなら重複しない
"""
import collections
import datetime
import json
import pathlib
import sys

SARIF_PATH = pathlib.Path("ci-results/merged.sarif")
AGENTS_MD = pathlib.Path("AGENTS.md")  # リポジトリルート
THRESHOLD = 3
SECTION_HEADER = "## 失敗から学んだこと (自動生成)"

if not SARIF_PATH.exists():
    sys.exit(0)
if not AGENTS_MD.exists():
    print(f"[lessons] {AGENTS_MD} not found", file=sys.stderr)
    sys.exit(0)

# 1. SARIF パース
try:
    sarif = json.loads(SARIF_PATH.read_text())
except json.JSONDecodeError:
    sys.exit(0)

# 2. ruleId 集計
counter = collections.Counter()
samples = {}
for run in sarif.get("runs", []):
    tool_name = run.get("tool", {}).get("driver", {}).get("name", "unknown")
    for r in run.get("results", []):
        rid = r.get("ruleId", "unknown")
        key = f"{tool_name}.{rid}"
        counter[key] += 1
        samples.setdefault(key, r.get("message", {}).get("text", "")[:120])

# 3. 閾値以上のルールを教訓化
today = datetime.date.today().isoformat()
new_lessons = [
    f"- {today} {key}: {n}回検出。{samples[key]}"
    for key, n in counter.most_common()
    if n >= THRESHOLD
]
if not new_lessons:
    sys.exit(0)

# 4. AGENTS.md に追記（重複回避）
text = AGENTS_MD.read_text()
if SECTION_HEADER not in text:
    text += f"\n\n{SECTION_HEADER}\n\n"

existing_lessons = set(text.splitlines())
fresh_lessons = [l for l in new_lessons if l not in existing_lessons]

if not fresh_lessons:
    print("[lessons] 追加すべき新しい教訓なし")
    sys.exit(0)

text = text.rstrip() + "\n" + "\n".join(fresh_lessons) + "\n"
AGENTS_MD.write_text(text)
print(f"[lessons] AGENTS.md に {len(fresh_lessons)} 件の教訓を追記")
```

---

## 3. AGENTS.md の対象

このスクリプトが書き込むのは **リポジトリルートの `/AGENTS.md`**（Claude Code がセッション開始時に読むファイル）。

ルート `AGENTS.md` 末尾に以下のプレースホルダーを置いておく（初回のみ手動）：

```markdown
<!-- 以下は Stop フックが自動追記する領域 -->

## 失敗から学んだこと (自動生成)

```

---

## 4. デモシナリオ

実際に学習ループが回ることを観察する手順：

### ステップ A: 同じ失敗を 3 回以上起こす

```bash
# Python のソースに故意にハードコードされたシークレットを書く
cat >> backend/src/demo.py <<'EOF'
# demo: intentional secrets for RALPH loop test
AWS_KEY_1 = "AKIAIOSFODNN7EXAMPLE"
AWS_KEY_2 = "AKIAJ7Q3HOY77LHEXMPL"
AWS_KEY_3 = "AKIA3X4Y5Z6789EXAMPL"
AWS_KEY_4 = "AKIAQQRRZZXXTTYYUUEX"
EOF

./ci.sh  # 4 件の AWS キー検出（gitleaks が拾う）
```

### ステップ B: Claude Code セッションを終了

```bash
exit
# Stop フックが発火 → AGENTS.md に教訓追記
```

### ステップ C: 追記内容を確認

```bash
$ tail -10 AGENTS.md
## 失敗から学んだこと (自動生成)

- 2026-05-08 gitleaks.aws-access-token: 4回検出。AWS Access Token detected
```

### ステップ D: 次のセッションを起動

```bash
$ claude "AWS連携の機能を追加して"
```

エージェントが起動するとき AGENTS.md が読まれ、「AWS access token をハードコードしてはいけない」という教訓が自動的にコンテキストに乗る。同じミスを繰り返さなくなる。

### ステップ E: Jaeger でフック実行を確認

Step 27 の OTel と組み合わせて、Stop フックが実際に発火したことを Jaeger で見られる。`http://localhost:16686` でサービス `claude-agent-harness` のトレースを開くと、最後のスパンとして `Stop` 系のイベントが現れる。

---

## RALPH 段階表

このステップは RALPH ループの**最小実装**である。

| Phase | 範囲 | 実装 step |
|---|---|---|
| **2.1 単一エージェント自己改善** | 1セッション → 次セッションへ教訓を伝搬 | **Step 28（このステップ）** |
| **3 マルチエージェント協調** | スペシャリスト間で役割分担 | Step 29 |
| **4 完全自律ループ** | PRD 完了まで人間介入なしで反復 | Step 30 |

Phase 2.1 だけでも実用価値は十分にある。「人間がCI失敗を見て、エージェントに次回伝える」というステップが消える。

---

## ci.sh への追加

ci.sh 自体は変更しない（フックは Claude Code セッション側で動く）。ただし `AGENTS.md` の差分が CI 内で発生した場合に検知するため、以下を末尾に追加：

```bash
echo "=== AGENTS.md 自動更新差分 ==="
git diff --stat AGENTS.md || true
```

加えて、**`/security-review` skill の出力を SARIF に変換して merged.sarif に流し込む**:

```bash
echo "=== security-review ==="
if [[ "${SECURITY_REVIEW_ENABLED:-0}" == "1" ]]; then
  claude --skill security-review --no-interactive --output-format sarif \
    > ci-results/sarif/security-review.sarif || true
else
  echo "(skip: SECURITY_REVIEW_ENABLED!=1)"
fi
```

`SECURITY_REVIEW_ENABLED=1 ./ci.sh` で有効化される。Phase 1 (Step 1-20) では呼ばないので開発体験を阻害しない。

---

## security-review が拾うべきレビュー観点 (チェックリスト)

`/security-review` の出力 SARIF にこれらの ruleId が現れたら、`sarif-to-lessons.py` のしきい値 (3 回以上) で AGENTS.md に教訓化される。skill 側でも明示的にチェックするよう prompt を構築しておく:

| ruleId 例 | 何を検出したいか |
|---|---|
| `test-endpoint-public` | `/test/slow` 等の開発専用エンドポイントが `ENV != "production"` ガードなしで `app.include_router` されていないか |
| `outbox-double-publish` | Outbox / queue / job-claim パターンで `SELECT ... FOR UPDATE SKIP LOCKED` 相当のアトミック claim を使っているか |
| `error-message-leak` | `except Exception as e: return JSONResponse({"detail": str(e)})` でスタックトレースを 4xx/5xx のレスポンス body に直書きしていないか（汎用文言を使う） |
| `id-normalization` | URL から取った文字列 ID をキャッシュキーに使う前に、ドメイン側の正規化関数（`LotNumber.parse` → `LotNumber.format`）を通しているか |
| `unbounded-input` | 数値クエリ (`ms`, `count`, `limit`) に Pydantic の `ge=0, le=MAX` バリデーションがあるか（例: `?ms=-1` で無限待機にならないか） |

これらは **静的解析 (gitleaks/pip-audit/bandit) では検出できない**設計レベルのレビュー観点で、`/security-review` skill を ci.sh に組み込むことで初めて自動回収できる。

---

## ci.sh の現時点の構成

```bash
#!/bin/bash
set -e

echo "=== ci-results/ 初期化 ==="
mkdir -p ci-results/sarif

echo "=== Jaeger 起動チェック ==="
curl -fs http://localhost:16686/api/services > /dev/null \
  || (echo "Jaeger 未起動: docker compose -f docker-compose.harness.yml up -d jaeger" && exit 1)

echo "=== マイグレーション ==="
cd backend && alembic upgrade head && cd ..

echo "=== フォーマットチェック ==="
cd backend && uv run ruff format --check src/ && cd ..

echo "=== リンター ==="
cd backend && uv run ruff check src/ && cd ..

echo "=== 型チェック ==="
cd backend && uv run mypy src/ --strict && cd ..
cd frontend && pnpm tsc --noEmit && cd ..

echo "=== テスト + カバレッジ ==="
cd backend && uv run pytest tests/ --cov=src --cov-branch --cov-fail-under=80 -q && cd ..
cd frontend && pnpm vitest run --coverage.enabled true && cd ..

echo "=== ミューテーションテスト ==="
bash scripts/check-mutmut-score.sh
cd frontend && pnpm stryker run && cd ..

echo "=== アーキテクチャ適合性 ==="
cd backend && uv run lint-imports && cd ..
cd frontend && pnpm depcruise src --config .dependency-cruiser.json && cd ..

echo "=== コントラクトテスト (Pact) ==="
curl -fs http://localhost:9292/diagnostic/status/heartbeat > /dev/null \
  || (echo "Pact Broker 未起動" && exit 1)
cd frontend && pnpm test:pact && cd ..
cd backend && PACT_PROVIDER_STATES=true uv run pytest tests/pact/ -q && cd ..

echo "=== シークレット検出 (SARIF) ==="
gitleaks detect --source . \
  --report-format sarif --report-path ci-results/sarif/gitleaks.sarif --exit-code 1

echo "=== SCA (SARIF) ==="
cd backend
uv run pip-audit --format json -o ../ci-results/pip_audit_raw.json || true
python ../scripts/pip-audit-to-sarif.py ../ci-results/pip_audit_raw.json ../ci-results/sarif/pip_audit.sarif
cd ..

echo "=== SAST (SARIF) ==="
cd backend && uv run bandit -r src/ -ll -ii -f sarif -o ../ci-results/sarif/bandit.sarif || true && cd ..
cd frontend && pnpm eslint src/ --format @microsoft/eslint-formatter-sarif --output-file ../ci-results/sarif/eslint.sarif || true && cd ..

echo "=== DAST (OWASP ZAP, SARIF) ==="
cd backend
uv run uvicorn src.main:app --host 0.0.0.0 --port 8000 &
APP_PID=$!
sleep 3
docker run --rm --network host \
  -v "$(pwd)/../ci-results/sarif:/zap/results" \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-api-scan.py \
    -t http://localhost:8000/openapi.json -f openapi \
    -z "addonupdate;addoninstall sarifreport" -J /zap/results/zap.sarif
kill $APP_PID
cd ..

echo "=== SBOM 生成 ==="
cd backend && uv run cyclonedx-bom environment --of JSON --outfile ../ci-results/sbom-backend.cdx.json && cd ..
cd frontend && pnpm cyclonedx-npm --output-format JSON --output-file ../ci-results/sbom-frontend.cdx.json && cd ..

echo "=== 脆弱性パッケージを Renovate 優先化 ==="
bash scripts/prioritize-from-sarif.sh ci-results/sarif/pip_audit.sarif renovate.json || true

echo "=== 依存更新チェック (dry-run) ==="
mkdir -p renovate-out
RENOVATE_PLATFORM=local RENOVATE_AUTODISCOVER=false \
  npx --yes renovate --dry-run > ci-results/renovate.log 2>&1 || true

echo "=== security-review ==="
if [[ "${SECURITY_REVIEW_ENABLED:-0}" == "1" ]]; then
  claude --skill security-review --no-interactive --output-format sarif \
    > ci-results/sarif/security-review.sarif || true
else
  echo "(skip: SECURITY_REVIEW_ENABLED!=1)"
fi

echo "=== SARIF マージ ==="
cd backend && uv run python -m sarif merge ../ci-results/sarif/*.sarif -o ../ci-results/merged.sarif && cd ..

echo "=== AGENTS.md 自動更新差分 ==="
git diff --stat AGENTS.md || true

echo "=== CI完了 ==="
```

---

## RALPHループにおける位置づけ

| 層 | 役割 |
|---|---|
| Memory（long-term, lessons-as-text） | 永続外部メモリ（AGENTS.md）の最小実装 |
| Phase 2.1: 単一エージェント自己改善 | 1セッションが学んだ事を次セッションへ伝搬 |

ハーネス論文（arXiv 2604.08224）の主張する **「外部化された記憶」** の最も基本的な実装。

このステップが完了して初めて、エージェントは「過去の失敗から学べる」状態になる。

ここまでで Phase 2 の **基盤** は完成し、Step 29 で複数エージェントの協調へ、Step 30 で完全自律ループへと拡張する。

---

## 次のステップ

Step 28が完了したら [Step 29: マルチエージェントオーケストレーター](./step29.md) へ進む。
