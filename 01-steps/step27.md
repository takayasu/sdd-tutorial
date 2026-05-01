# Step 27: OpenTelemetryエージェントトレース

## 目的

### これは何か

Claude Code のフック機構（`.claude/settings.json`）を使い、エージェントがツール（Read / Edit / Bash 等）を呼び出すたびに OpenTelemetry の OTLP/HTTP プロトコルで Jaeger にスパンを送信する。エージェントの実行を「分散トレース」として可視化し、Step 21 の SARIF（=結果の観測）と対をなす「プロセスの観測」を確立する。

### なぜやるのか

- Phase 2 ではエージェントが多数のツールを連鎖実行する。失敗時に「どこで詰まったか」を順序付きで追跡できないとデバッグが事実上不可能
- アプリケーションの分散トレーシングと同じメンタルモデルでエージェント実行を扱えるようになる
- Step 29 のマルチエージェント協調では、複数エージェントが同時実行するため、トレース ID で行動を束ねる手段が必須

### 何がうれしいのか

- 1 コマンドの裏で何が走ったかが Jaeger の Gantt チャートで見える
- Step 29 のマスター → サブエージェント実行が親子スパンの木として可視化される
- ハーネス論文（arXiv 2604.08224）が推奨する「外部化された observability」を実装できる

## 完了条件

```bash
# 1. Jaeger 起動
$ docker compose -f docker-compose.harness.yml up -d jaeger
$ curl -fs http://localhost:16686/api/services | jq .
{ "data": [], "total": 0, "limit": 0, "offset": 0, "errors": null }

# 2. Claude Code セッションを実行（任意の操作）
$ claude code "test"
# (フックが発火してスパンが送信される)

# 3. サービス確認
$ curl -fs http://localhost:16686/api/services | jq -r '.data[]'
claude-agent-harness

# 4. トレース取得
$ curl -fs "http://localhost:16686/api/traces?service=claude-agent-harness&limit=1" \
  | jq '.data[0].spans | length'
12

# ブラウザで http://localhost:16686 を開いて Gantt チャートを確認
```

---

## 1. Jaeger を docker-compose に追加

`/docker-compose.harness.yml` に追記：

```yaml
  jaeger:
    image: jaegertracing/all-in-one:1.62
    ports:
      - "16686:16686"   # UI
      - "4318:4318"     # OTLP/HTTP
      - "4317:4317"     # OTLP/gRPC
    environment:
      COLLECTOR_OTLP_ENABLED: "true"
```

起動：

```bash
docker compose -f docker-compose.harness.yml up -d jaeger
```

---

## 2. .claude/settings.json でフック登録

リポジトリルートに `.claude/settings.json`：

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          { "type": "command", "command": "python3 .claude/scripts/start-trace.py" }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": ".*",
        "hooks": [
          { "type": "command", "command": "python3 .claude/scripts/emit-otel.py" }
        ]
      }
    ]
  }
}
```

`SessionStart` で `CLAUDE_TRACE_ID` を生成して `~/.claude-trace-id` に書き、`PostToolUse` がそれを読んで「同一セッションのスパン」を束ねる。

---

## 3. SessionStart フック: トレース ID 生成

`.claude/scripts/start-trace.py`:

```python
#!/usr/bin/env python3
"""SessionStart フック。トレース ID を生成して保存する。"""
import os
import uuid
import pathlib

trace_id = uuid.uuid4().hex
session_id = os.environ.get("CLAUDE_SESSION_ID", "unknown")

trace_file = pathlib.Path.home() / ".claude-trace-id"
trace_file.write_text(f"{trace_id}\n{session_id}\n")

print(f"[OTel] trace_id={trace_id} session={session_id}")
```

---

## 4. PostToolUse フック: スパン送信

`.claude/scripts/emit-otel.py`:

```python
#!/usr/bin/env python3
"""PostToolUse フック。stdin の JSON を OTLP/HTTP で Jaeger に送る。

stdin の形（Anthropic 公式ドキュメント参照）:
{
  "session_id": "...",
  "tool_name": "Read",
  "tool_input": { "file_path": "..." },
  "tool_response": { "..." },
  "duration_ms": 12.3,
  "exit_code": 0
}
"""
import json
import os
import sys
import time
import uuid
import pathlib
import urllib.request

# stdin から フックのペイロードを取得
try:
    payload = json.loads(sys.stdin.read() or "{}")
except json.JSONDecodeError:
    sys.exit(0)

tool_name = payload.get("tool_name", "unknown")
duration_ms = float(payload.get("duration_ms", 0))
exit_code = int(payload.get("exit_code", 0))

# トレース ID を取得（SessionStart で書いた）
trace_file = pathlib.Path.home() / ".claude-trace-id"
trace_id = trace_file.read_text().splitlines()[0] if trace_file.exists() else uuid.uuid4().hex

span_id = uuid.uuid4().hex[:16]
end_ns = int(time.time() * 1e9)
start_ns = end_ns - int(duration_ms * 1e6)

otlp = {
    "resourceSpans": [{
        "resource": {
            "attributes": [
                {"key": "service.name", "value": {"stringValue": "claude-agent-harness"}},
                {"key": "deployment.environment", "value": {"stringValue": os.environ.get("ENV", "local")}},
            ]
        },
        "scopeSpans": [{
            "spans": [{
                "traceId": trace_id,
                "spanId": span_id,
                "name": f"tool/{tool_name}",
                "kind": 1,  # SPAN_KIND_INTERNAL
                "startTimeUnixNano": str(start_ns),
                "endTimeUnixNano": str(end_ns),
                "attributes": [
                    {"key": "tool.name", "value": {"stringValue": tool_name}},
                    {"key": "tool.exit_code", "value": {"intValue": str(exit_code)}},
                    {"key": "tool.duration_ms", "value": {"doubleValue": duration_ms}},
                ],
                "status": {"code": 2 if exit_code != 0 else 1}  # ERROR or OK
            }]
        }]
    }]
}

req = urllib.request.Request(
    "http://localhost:4318/v1/traces",
    data=json.dumps(otlp).encode(),
    headers={"Content-Type": "application/json"},
    method="POST"
)
try:
    urllib.request.urlopen(req, timeout=2)
except Exception as e:
    # Jaeger が落ちててもフックを失敗させない
    print(f"[OTel] failed: {e}", file=sys.stderr)
```

`Content-Type: application/json` を付けるのが OTLP/HTTP では必須。`urllib.request` を使っているのは外部依存（`requests` 等）を増やさないため。

---

## 5. Jaeger UI で確認

ブラウザで `http://localhost:16686` を開く：

1. **Service** ドロップダウンで `claude-agent-harness` を選択
2. **Find Traces** をクリック
3. トレースを選ぶと Gantt チャートで「いつ何を実行したか」が時系列で見える
4. 各スパンをクリックすると `tool.name`, `exit_code`, `duration_ms` などのアトリビュートが見える

---

## 6. ローカル `~/.claude` との関係

`.claude/` がリポジトリにある場合、Claude Code はプロジェクトローカルの設定を**優先**する。グローバル `~/.claude/settings.json` が既にある場合、フックは「両方が実行される」マージ動作になる（公式ドキュメント参照）。

PoC では、`.claude/settings.json` と `.claude/scripts/` をリポジトリにコミットすることを推奨する。これによりリポジトリをクローンした任意の開発者・エージェントが同じ可観測性を享受できる。

---

## ci.sh への追加

```bash
echo "=== Jaeger 起動チェック ==="
curl -fs http://localhost:16686/api/services > /dev/null \
  || (echo "Jaeger 未起動: docker compose -f docker-compose.harness.yml up -d jaeger" && exit 1)
```

CI 自体はトレースを生成しないが、Jaeger が利用可能な状態であることを保証する（Step 29 のマルチエージェント実行で必須）。

---

## ci.sh の現時点の構成

```bash
#!/bin/bash
set -e

echo "=== ci-results/ 初期化 ==="
mkdir -p ci-results/sarif

echo "=== Jaeger 起動チェック ==="
curl -fs http://localhost:16686/api/services > /dev/null \
  || (echo "Jaeger 未起動" && exit 1)

echo "=== マイグレーション ==="
echo "=== ビルド ==="
echo "=== フォーマットチェック ==="
echo "=== リンター ==="
echo "=== テスト + カバレッジ ==="

echo "=== ミューテーションテスト ==="
(cd ../sales-management/apps/api-fsharp && dotnet stryker)
cd kotlin && gradle pitest && cd ..

echo "=== アーキテクチャ適合性 ==="
(cd ../sales-management/apps/api-fsharp && dotnet test --filter "Category=Architecture")
cd kotlin && gradle test --tests "*ArchitectureTest*" && cd ..

echo "=== コントラクトテスト (Pact) ==="
(cd ../sales-management/apps/api-fsharp && dotnet test --filter "Category=Pact")
cd kotlin && gradle pactVerify && cd ..

echo "=== シークレット検出 (SARIF) ==="
gitleaks detect --source . \
  --report-format sarif --report-path ci-results/sarif/gitleaks.sarif --exit-code 1

echo "=== SCA (SARIF) ==="
trivy fs --scanners vuln --severity HIGH,CRITICAL \
  --format sarif --output ci-results/sarif/trivy.sarif .

echo "=== SAST (SonarQube) ==="
gradle sonar
bash scripts/sonar-to-sarif.sh sales-management-kotlin ci-results/sarif/sonar.sarif

echo "=== DAST (OWASP ZAP, SARIF) ==="
# (アプリ起動 → ZAP実行 → アプリ停止)

echo "=== SBOM 生成 ==="
cd ../sales-management/apps/api-fsharp && \
  dotnet CycloneDX src/SalesManagement/SalesManagement.fsproj \
    --json --output ../ci-results --filename sbom-fsharp.cdx.json && \
  cd ..
cd kotlin && gradle cyclonedxBom && cd ..

echo "=== 脆弱性パッケージを Renovate 優先化 ==="
bash scripts/prioritize-from-trivy.sh \
  ci-results/sarif/trivy.sarif renovate.json || true

echo "=== 依存更新チェック (dry-run) ==="
RENOVATE_PLATFORM=local RENOVATE_AUTODISCOVER=false \
  npx --yes renovate --dry-run > ci-results/renovate.log 2>&1 || true

echo "=== SARIF マージ ==="
sarif merge ci-results/sarif/*.sarif \
  --output-file-path ci-results/merged.sarif --recurse false

echo "=== CI完了 ==="
```

---

## RALPHループにおける位置づけ

| 層 | 役割 |
|---|---|
| Observability（プロセス側） | エージェントが何をどの順で実行したかを記録 |
| Step 29 の前提インフラ | マルチエージェントの協調を親子スパンで可視化 |

Step 21 の SARIF が「コードに対して何が起きたか（結果の観測）」を可読化したのに対し、Step 27 は「エージェントが何をしたか（プロセスの観測）」を可読化する。

ハーネスエンジニアリング論文（arXiv 2604.08224）の主張: エージェントの能力は内部知能ではなく、`harness` がどれだけ自身の動作を観測可能にしているかで決まる。Step 27 はその観測層を確立する。

---

## 次のステップ

Step 27が完了したら [Step 28: AGENTS.md自動更新](./step28.md) へ進む。
