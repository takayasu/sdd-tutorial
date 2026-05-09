# Step 29: マルチエージェントオーケストレーター（Phase 3）

## 目的

### これは何か

マスターエージェント（`master.py`）が `prd.md`（要件定義）を読んでタスクに分解し、Claude Code の subagent 機構で `domain-modeler`、`test-writer`、`refactorer`、`doc-updater` の 4 つの専門エージェントに委譲する。エージェント間の通信は filesystem（`.harness/inbox/<agent>/*.json`、`.harness/outbox/<agent>/*.json`）の JSON メッセージで行う。

これが RALPH ループの **Phase 3: マルチエージェント協調**。

### なぜやるのか

- 単一エージェントは長セッションで「あれもこれも」やろうとしてコンテキストが汚染される。専門化すると各エージェントの責務が明確になり、プロンプトが短く決定論的になる
- 専門エージェントごとに `tools` を制限することで、ドメインモデル生成エージェントが DB を触ったり、テスト生成エージェントが本番コードを書いたりする事故を防ぐ
- Step 27 の OTel と組み合わせると、4 エージェントの並列実行がトレースで可視化できる

### 何がうれしいのか

- コンテキスト窓を効率利用（各エージェントが自分の責務にだけ集中）
- 失敗したエージェントだけを再投入できる（fault isolation）
- 役割分担で再現性が上がる
- Step 30 の完全自律ループの構成要素になる

## 完了条件

```bash
# 1. PRD を作成
$ cat > prd.md <<'EOF'
# PRD

## やるべきこと

- [ ] 直接販売案件: 返品処理 behavior 追加
EOF

# 2. マスターを起動
$ python3 .harness/master.py --prd prd.md
[master] task decomposition: 1 subtasks
[master] dispatch domain-modeler ...
[domain-modeler] wrote backend/src/domain/returns.py
[master] dispatch test-writer ...
[test-writer] wrote backend/tests/test_returns_pbt.py
[master] dispatch refactorer ...
[refactorer] applied ruff format/check and mypy fixes
[master] dispatch doc-updater ...
[doc-updater] updated domain-model-section2.md
[master] all green: ./ci.sh exit 0
$ echo $?
0

# 3. Jaeger で4エージェントのスパンを確認
$ open http://localhost:16686/search?service=claude-agent-harness
# master が親、4エージェントが子としてぶら下がる木構造が見える
```

---

## 1. .harness/ ディレクトリ構造

```
.harness/
├── agents/                # サブエージェント定義
│   ├── domain-modeler.json
│   ├── test-writer.json
│   ├── refactorer.json
│   └── doc-updater.json
├── inbox/                 # 各エージェントへの入力メッセージ
│   ├── domain-modeler/
│   ├── test-writer/
│   ├── refactorer/
│   └── doc-updater/
├── outbox/                # 各エージェントからの出力メッセージ
│   └── ...
├── lessons.md             # エージェント間の共有メモリ
├── master.py              # オーケストレーター
└── README.md              # 構造説明
```

---

## 2. サブエージェント定義（4 件）

### `.harness/agents/domain-modeler.json`

```json
{
  "name": "domain-modeler",
  "description": "DSL から Python frozen dataclass / TypeScript readonly interface を生成する責務のみ",
  "tools": ["Read", "Write", "Edit", "Glob", "Grep"],
  "system_prompt": "あなたは Python と TypeScript の型定義のみを書く専門エージェントです。\n\n責務:\n- domain-model-*.md の DSL を読み、backend/src/domain/*.py と frontend/src/domain/*.ts を生成・更新する\n- Python: @dataclass(frozen=True) + Literal 型判別子による状態機械\n- TypeScript: readonly interface + discriminated union の正しい使い分け\n- 不変性を保つ（Python: frozen=True 必須、TypeScript: readonly 必須）\n\n禁止事項:\n- Workflows / API ハンドラ / DB アクセスへの変更\n- テストコードの記述\n\n出力: 完了時に .harness/outbox/domain-modeler/<msg_id>.json に { \"status\": \"ok\" | \"error\", \"files\": [...] } を書く"
}
```

### `.harness/agents/test-writer.json`

```json
{
  "name": "test-writer",
  "description": "PBT・アーキテクチャ・Pact テストを生成する責務のみ",
  "tools": ["Read", "Write", "Edit", "Glob", "Grep", "Bash"],
  "system_prompt": "あなたはテストのみを書く専門エージェントです。\n\n責務:\n- domain-modeler が生成した型に対する hypothesis (Python PBT) / fast-check (TypeScript PBT) のプロパティテストを書く\n- import-linter のレイヤールールを追加・更新する\n- Pact プロバイダ検証テストを更新する\n\n禁止事項:\n- 本番コード（domain/*.py / workflows/*.py / routers/*.py）の変更\n- DB マイグレーション\n\n出力: 完了時に uv run pytest / pnpm vitest run を実行し、結果を outbox に記録"
}
```

### `.harness/agents/refactorer.json`

```json
{
  "name": "refactorer",
  "description": "Linter / Formatter / SAST の指摘を受けて refactor する責務のみ",
  "tools": ["Read", "Edit", "Bash"],
  "system_prompt": "あなたは Lint/Format/SAST の指摘を解消する専門エージェントです。\n\n責務:\n- ci-results/merged.sarif を読み、warning/error を順に解消\n- ロジック変更は最小限（型を変えてはいけない）\n- ruff format / ruff check / mypy --strict / eslint の警告 0 を目標\n\n禁止事項:\n- 新機能追加\n- 型定義の変更\n- テストの書き換え（テストが落ちたら test-writer に戻す）"
}
```

### `.harness/agents/doc-updater.json`

```json
{
  "name": "doc-updater",
  "description": "domain-model-*.md / README.md / AGENTS.md を整合させる責務のみ",
  "tools": ["Read", "Write", "Edit"],
  "system_prompt": "あなたはドキュメント整合エージェントです。\n\n責務:\n- 実装変更後、domain-model-section*.md を最新化\n- README.md の API 一覧を更新（FastAPI の /openapi.json を参照）\n- AGENTS.md の自動生成セクションには触らない（Stop フックの管轄）\n\n禁止事項:\n- コードの変更\n- AGENTS.md の自動生成セクションの編集"
}
```

---

## 3. マスターオーケストレーター

`.harness/master.py`:

```python
#!/usr/bin/env python3
"""prd.md を読み、4 エージェントにタスクを順次委譲する。"""
import argparse
import json
import os
import pathlib
import re
import subprocess
import sys
import uuid

HARNESS_DIR = pathlib.Path(".harness")
INBOX_DIR = HARNESS_DIR / "inbox"
OUTBOX_DIR = HARNESS_DIR / "outbox"
AGENT_PIPELINE = ["domain-modeler", "test-writer", "refactorer", "doc-updater"]


def decompose(prd_text: str) -> list[dict]:
    """PRD の `- [ ] ...` 行を「タスク」として抽出。"""
    tasks = []
    for line in prd_text.splitlines():
        m = re.match(r"^\s*- \[ \]\s+(.+)$", line)
        if m:
            tasks.append({"description": m.group(1)})
    return tasks


def dispatch(agent: str, payload: dict) -> dict:
    """指定したサブエージェントを起動して outbox の結果を返す。"""
    msg_id = uuid.uuid4().hex
    inbox = INBOX_DIR / agent
    outbox = OUTBOX_DIR / agent
    inbox.mkdir(parents=True, exist_ok=True)
    outbox.mkdir(parents=True, exist_ok=True)

    inbox_msg = inbox / f"{msg_id}.json"
    inbox_msg.write_text(json.dumps(payload, ensure_ascii=False, indent=2))

    print(f"[master] dispatch {agent} (msg={msg_id})")

    # OTel トレース ID を環境変数で渡す（同じ trace に親子スパンを束ねる）
    env = os.environ.copy()
    trace_file = pathlib.Path.home() / ".claude-trace-id"
    if trace_file.exists():
        env["CLAUDE_TRACE_ID"] = trace_file.read_text().splitlines()[0]

    result = subprocess.run(
        [
            "claude", "code",
            "--agent", agent,
            "--input-file", str(inbox_msg),
            "--no-resume",
        ],
        capture_output=True, text=True, env=env, timeout=600
    )

    out_msg = outbox / f"{msg_id}.json"
    if out_msg.exists():
        return json.loads(out_msg.read_text())
    return {"status": "error", "stderr": result.stderr[:500]}


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--prd", default="prd.md")
    args = parser.parse_args()

    prd = pathlib.Path(args.prd)
    if not prd.exists():
        sys.exit(f"PRD not found: {prd}")

    tasks = decompose(prd.read_text())
    print(f"[master] task decomposition: {len(tasks)} tasks")

    for task in tasks:
        # 各タスクに対して 4 エージェントを順に走らせる
        for agent in AGENT_PIPELINE:
            result = dispatch(agent, {"task": task["description"]})
            if result.get("status") != "ok":
                print(f"[master] {agent} failed: {result}", file=sys.stderr)
                sys.exit(2)

    # 最終 CI
    print("[master] running ./ci.sh ...")
    ci = subprocess.run(["./ci.sh"])
    if ci.returncode != 0:
        sys.exit("[master] CI 失敗")
    print(f"[master] all green: ./ci.sh exit 0")


if __name__ == "__main__":
    main()
```

---

## 4. エージェント間共有メモリ

`.harness/lessons.md` は全エージェントが起動時に読む共有メモリ。`AGENTS.md`（Step 28 が更新）と相補：

| ファイル | 更新主体 | 内容 |
|---|---|---|
| `AGENTS.md` | Stop フック（自動） | CI 失敗パターン全般 |
| `.harness/lessons.md` | 各エージェントが自分で追記 | サブエージェント固有の学び |

例：

```markdown
# .harness/lessons.md

## domain-modeler

- 2026-05-08 Python frozen dataclass の field() でデフォルト値が必要な場合は field(default=...) を使う
- 2026-05-08 Literal 型は `from typing import Literal` でインポートが必要（Python 3.8+）

## test-writer

- 2026-05-08 hypothesis の @given で frozen dataclass を生成するには st.builds() を使う
- 2026-05-08 fast-check で readonly な TypeScript 型を生成するときは fc.record() が便利
```

---

## 5. OTel での可視化

マスターは `CLAUDE_TRACE_ID` を環境変数で渡すため、Jaeger 上では以下のような木構造になる：

```
master (root span)
├── tool/dispatch domain-modeler
│   ├── tool/Read (backend/src/domain/returns.py)
│   ├── tool/Edit (backend/src/domain/returns.py)
│   └── tool/Write (.harness/outbox/domain-modeler/xxx.json)
├── tool/dispatch test-writer
│   ├── tool/Read
│   ├── tool/Write (backend/tests/test_returns_pbt.py)
│   └── tool/Bash (uv run pytest)
├── tool/dispatch refactorer
└── tool/dispatch doc-updater
```

UI で 1 トレースを開けば、4 エージェントが何をどの順で実行したかが Gantt チャートで一目見える。

---

## ci.sh への追加

ci.sh 自体は変更しない。マスターが内部から `./ci.sh` を呼ぶ。

---

## ci.sh の現時点の構成

（Step 28 と同じ。マスターは ci.sh を変更しない）

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
| Orchestration | マスターがタスク分解 → 専門エージェントに委譲 |
| Protocols（エージェント間） | filesystem 経由の inbox/outbox メッセージング |
| Skills の特化 | 各エージェントが狭い責務に特化することで品質安定 |

Step 28 の単一エージェント自己改善が「縦方向の学習（時間軸の知識伝搬）」だとすれば、Step 29 は「横方向の分業（責務軸の専門化）」。両者が揃って初めて Step 30 の完全自律ループに進める。

ハーネス論文（arXiv 2604.08224）の主張: マルチエージェント協調はマスターと専門家の双方が「ハーネスから提供されたプロトコル」（ここでは inbox/outbox）でやり取りすることで、個々のエージェントの能力を超えた集合知が成立する。

---

## 次のステップ

Step 29が完了したら [Step 30: 完全自律RALPHループ](./step30.md) へ進む。
