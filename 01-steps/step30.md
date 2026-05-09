# Step 30: 完全自律RALPHループ（Phase 4）

## 目的

### これは何か

Geoffrey Huntley が提唱した **"Ralph Wiggum as a Software Engineer"** パターンを実装する。`prd.md`（やるべきこと一覧）と `progress.txt`（達成済み）を filesystem に置き、bash の while ループで以下を自動反復する：

```
1. prd.md から未完了タスクを 1 つ取り出す
2. コンテキストを完全リセットしてエージェント起動
3. CI 実行
4. 緑なら progress 更新 + git commit
5. 失敗ならループの次の周回へ（Stop フックが AGENTS.md に教訓を蓄積する）
6. PRD 全項目が [x] になるまで繰り返す
```

これが RALPH ループの **Phase 4: 完全自律**。Step 21–29 で整えた前提（SARIF / OTel / AGENTS.md自動更新 / マルチエージェント）が全て揃って初めて安全に動く。

### なぜやるのか

- LLM は長セッションで劣化（コンテキスト汚染）するため、ruthless な context reset が品質維持に必須
- Geoffrey Huntley の主張: agent loops が成立するのは LLM の知能ではなく **「ruthless reset + filesystem memory + CI as oracle」** の3点による
- 人間が寝ている間に PRD を処理しきる、という体験を物理的に実装する

### 何がうれしいのか

- PoC の最終目標である「AIに開発を委ねる」が物理的に動く形で実現
- コスト・反復回数のガードレールで安全に動かせる
- Phase 1-3 で築いた全要素が「PRD → 実装 → CI → 学習」の単一ループに統合される

## 完了条件

```bash
# 1. PRD を書く
$ cat > prd.md <<'EOF'
# PRD

## やるべきこと

- [ ] 直接販売案件: 返品処理 behavior 追加
- [ ] 予約販売案件: 予約修正 behavior 追加
- [ ] 委託販売案件: 月次決算ジョブ追加
EOF

# 2. RALPH ループを起動
$ MAX_ITER=20 BUDGET_USD=10 ./harness/ralph.sh prd.md
[ralph] iter=1 task='直接販売案件: 返品処理 behavior 追加' status=ok
[ralph] iter=2 task='予約販売案件: 予約修正 behavior 追加' status=ok
[ralph] iter=3 task='委託販売案件: 月次決算ジョブ追加' status=ok
[ralph] all done. iterations=3 cost=$2.41

# 3. PRD が完了
$ cat prd.md
- [x] 直接販売案件: 返品処理 behavior 追加
- [x] 予約販売案件: 予約修正 behavior 追加
- [x] 委託販売案件: 月次決算ジョブ追加

# 4. git log に各反復のコミットが残る
$ git log --oneline -3
abc123 ralph iter=3: 委託販売案件: 月次決算ジョブ追加
def456 ralph iter=2: 予約販売案件: 予約修正 behavior 追加
789abc ralph iter=1: 直接販売案件: 返品処理 behavior 追加
```

---

## 1. harness/ralph.sh の実装

`harness/ralph.sh`（リポジトリルート直下の `harness/` ディレクトリ）：

```bash
#!/usr/bin/env bash
set -euo pipefail

PRD="${1:-prd.md}"
MAX_ITER="${MAX_ITER:-20}"
BUDGET_USD="${BUDGET_USD:-10}"
ITER=0
COST=0

# 完了判定: prd.md に未チェック項目が残っていなければ true
all_done() { ! grep -q '^- \[ \]' "$PRD"; }

log() { echo "[ralph] $*" >&2; }

while ! all_done; do
  ITER=$((ITER + 1))

  if [ "$ITER" -gt "$MAX_ITER" ]; then
    log "max iterations ($MAX_ITER) reached"
    exit 2
  fi

  if (( $(echo "$COST > $BUDGET_USD" | bc -l) )); then
    log "budget ($BUDGET_USD USD) exceeded"
    exit 3
  fi

  TASK=$(grep -m1 '^- \[ \]' "$PRD" | sed 's/^- \[ \] //')
  log "iter=$ITER task='$TASK'"

  # === Ruthless context reset ===
  # 前回の文脈は AGENTS.md / .harness/lessons.md / prd.md / progress.txt のみ。
  # それ以外は全部捨てる。

  SESSION_OUT=$(mktemp)
  set +e
  claude code \
    --no-resume \
    --system-prompt "$(cat AGENTS.md .harness/lessons.md 2>/dev/null)" \
    --input "次のタスクを単一実行で完遂せよ:
$TASK

実装後、./ci.sh が exit 0 で終わるまで修正を繰り返せ。
完了したら progress.txt に '[x] $TASK' を追記せよ。" \
    --max-cost "$(echo "$BUDGET_USD - $COST" | bc -l)" \
    --json \
    > "$SESSION_OUT" 2>&1
  CLAUDE_EXIT=$?
  set -e

  # コスト累積
  THIS_COST=$(jq -r '.usage.total_cost_usd // 0' < "$SESSION_OUT" 2>/dev/null || echo "0")
  COST=$(echo "$COST + $THIS_COST" | bc -l)

  # CI 実行
  if ./ci.sh > /dev/null 2>&1; then
    sed -i "s|^- \[ \] $(echo "$TASK" | sed 's/[\/&]/\\&/g')$|- [x] $TASK|" "$PRD"
    git add -A
    git commit -m "ralph iter=$ITER: $TASK" || true
    log "iter=$ITER ✓ status=ok cost=\$$THIS_COST"
  else
    log "iter=$ITER ✗ FAILED, retry next loop with updated lessons"
    # Stop フック (Step 28) が既に AGENTS.md に教訓を追記しているため、
    # 次の反復ではその教訓が system_prompt に乗る。
  fi
done

log "all done. iterations=$ITER cost=\$$COST"
```

`chmod +x harness/ralph.sh` で実行可能にする。

---

## 2. ガードレール一覧

| 機構 | 値 / 設定 | 効果 |
|---|---|---|
| `MAX_ITER` | 20 | 無限ループ防止 |
| `BUDGET_USD` | 10 | API 課金上限 |
| `Stop` フック (Step 28) | merged.sarif → AGENTS.md | 同一失敗の繰り返し抑止 |
| `git commit` 毎反復 | 履歴に残す | 破壊的変更からの巻き戻し可能 |
| `--no-resume` | コンテキスト完全リセット | 文脈汚染遮断 |
| ci.sh の SARIF threshold | error level 0 | 品質ゲート |
| `claude --max-cost` | 残予算ベースで動的設定 | セッション単位の浪費防止 |

注意: 実際の `claude` CLI が `--max-cost` フラグを正確にサポートしない場合は、`scripts/track-cost.py` で利用ログから累積コストを推定するラッパーに置き換える。

---

## 3. progress.txt の役割

`progress.txt` は `prd.md` と相補的に動く永続記録：

```text
# progress.txt

[2026-05-08 10:00:00] ralph iter=1 task='直接販売案件: 返品処理' status=ok cost=$0.82
[2026-05-08 10:05:30] ralph iter=2 task='予約販売案件: 予約修正' status=fail reason='uv run pytest failed: AssertionError at test_returns_pbt.py:42'
[2026-05-08 10:08:11] ralph iter=2 task='予約販売案件: 予約修正' status=ok cost=$0.95
[2026-05-08 10:13:00] ralph iter=3 task='委託販売案件: 月次決算ジョブ' status=ok cost=$0.64
```

各 iter の結果を時系列で残すことで、「どこで詰まったか」「どのタスクに時間がかかったか」を後から分析できる。

---

## 4. Geoffrey Huntley の原型への参照

> RALPH の名称は Geoffrey Huntley のブログ記事 *"Ralph Wiggum as a Software Engineer"* に由来する。
> 由来：シンプソンズの Ralph Wiggum は記憶力に問題があるが、毎回新鮮な気持ちで最善を尽くす。
> Huntley の主張: agent loops が動くのは LLM の知能ではなく、以下の 3 点が揃ったとき：
>
> 1. **Ruthless reset** — コンテキストを毎回完全に捨てる
> 2. **Filesystem memory** — 永続化はファイルシステムと git に任せる
> 3. **CI as oracle** — 何が正しいかは CI が判定する（LLM 自身に問わない）
>
> 本ステップはこの 3 原則を PoC に直接実装したもの。

参考: [everything is a ralph loop](https://ghuntley.com/loop/)

---

## 5. 動作確認の進め方

実際に動かすときは以下の段階を踏む。

### 段階 1: 1 タスクで試す

```bash
$ cat > prd.md <<'EOF'
- [ ] テスト用ダミー behavior: ロット番号フォーマット検証関数を追加
EOF

$ MAX_ITER=5 BUDGET_USD=2 ./harness/ralph.sh prd.md
```

このサイズなら 1 反復で完了するはず。失敗時のリトライを観察するため、わざと曖昧な PRD で 2-3 反復を経験するのも学習に良い。

### 段階 2: 3 タスクで連続実行

完了条件のとおり 3 タスクで実行。OTel トレースで「マスター → サブエージェント → ツール」のスパン木が観察できる。

### 段階 3: 自分のドメインで PRD を書く

販売管理ドメインのこれから足したい behavior を PRD に並べて、放置で帰宅する。翌朝には PRD が `[x]` だらけになっているのを目指す。

---

## 6. PoC の最終評価指標

Phase 2 の成功は以下で測る：

| 指標 | 計測方法 |
|---|---|
| Ralph 平均反復回数 | progress.txt の iter 数を PRD タスク数で割る（理想は 1.0、現実は 1.5-2.0） |
| AGENTS.md 教訓蓄積速度 | `git log --oneline AGENTS.md \| wc -l` の増加トレンド |
| 1 タスクあたりコスト | progress.txt のコスト合計 / タスク数 |
| 失敗パターン削減率 | 同じ ruleId が 3 反復連続で出るか（Step 28 が機能しているか） |
| `can-i-deploy` 通過率 | Pact Broker の検証履歴から算出 |

---

## ci.sh への追加

ci.sh 末尾に Phase 2 完成のマーカーを追加：

```bash
echo "=== Phase 2 RALPH ハーネス完成 ==="
```

`harness/ralph.sh` 自体は ci.sh の外側で動き、内部から ci.sh を呼ぶ構造のため ci.sh は変更不要。

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

echo "=== Phase 2 RALPH ハーネス完成 ==="
echo "=== CI完了 ==="
```

---

## RALPHループにおける位置づけ

| 層 | 役割 |
|---|---|
| 完全な harness loop | Phase 1-2 全要素の統合 |
| Phase 4: 完全自律 | PRD 完了まで人間介入なしで反復 |

全前提技術の総合：

- **SARIF (Step 21)** で結果可読化
- **Mutation (Step 22) / アーキテクチャ適合性 (Step 23) / Pact (Step 24)** で多層品質保証
- **SBOM (Step 25) / Renovate (Step 26)** で依存管理
- **OTel (Step 27)** で行動観測
- **AGENTS.md 自動更新 (Step 28)** で自己学習
- **マルチエージェント (Step 29)** で分業
- **Step 30** ですべてを束ね、PRD駆動の green-loop として閉じる

---

## チュートリアルの締め

Step 30 をもって本チュートリアルは終了。あとは：

1. `prd.md` を書いて `./harness/ralph.sh` を回す
2. progress.txt と AGENTS.md の蓄積を観察する
3. 失敗パターンが Step 28 によって学習されているか確認する
4. 平均反復回数とコストをトラッキングする

ハーネスエンジニアリングの本質は **「エージェント自身を改善する仕組みを作ること」** であり、それが Phase 2 の 10 ステップで段階的に確立される。

---

## 次のステップ

このチュートリアル全体の完了です。

実運用での発展課題：

- Step 24 の仮 Consumer Pact を、実際の API 呼び出しから生成されるものへ置換
- Phase 3 として、エージェントが自分で PRD を生成する「Goal-Seeking RALPH」へ拡張
- メトリクス（反復回数・コスト・失敗削減率）を Grafana ダッシュボードに統合し、ハーネスのROIを可視化
