# Step 22: ミューテーションテスト

## 目的

### これは何か

ミューテーションテストとは、本番コードに機械的な小さな変更（ミュータント）を加え、既存のテストがそれを検出できるかを測る手法。例えば `if x > 0:` を `if x >= 0:` に変えたバージョンに対してテストが落ちれば「ミュータントを殺せた」、落ちなければ「ミュータントが生き残った」となる。

スコアは `killed / total mutants × 100%` で表され、テストの「厳しさ」の客観指標になる。

### なぜやるのか

- Step 9 で導入したカバレッジは「テストが行を通った」だけを測る。テストが `assert` をしていなくても、あるいは条件が逆でも通る弱いアサーションでも、カバレッジは100%になりうる
- Step 8 の PBT は入力をランダム化するが、PBT のプロパティ自体が弱いと意味がない。ミューテーションテストはプロパティの厳しさを定量評価する
- AIが生成したテストが「形だけのアサーション」になっていないかを検証できる

### 何がうれしいのか

- Mutation Score 75% 以上を CI ゲートに置けば、テストの実質的な品質が担保される
- 生き残ったミュータントを見ると「ここのテストが弱い」が一目でわかる
- エージェントが「テストだけ通せば良い」抜け道を防ぐハーネス機構になる

## 完了条件

### Python（mutmut）

```bash
$ cd backend && uv run mutmut run
...
Legend for output:
🎉 Survived mutants: 28
🫥 Killed mutants: 102
⏰ Timeout: 3
🙁 Suspicious: 2

$ uv run mutmut results
Survived mutants (28):
...

$ bash scripts/check-mutmut-score.sh
Mutation score: 78.5% (killed=102, survived=28)
PASS mutation-score >= 75%
```

### TypeScript（Stryker）

```bash
$ cd frontend && pnpm stryker run
...
All files mutated.
Mutation testing is done!

File          | % score | # killed | # timeout | # survived | # no cov | # error |
--------------|---------|----------|-----------|------------|----------|---------|
All files     |   79.17 |      114 |         2 |         27 |        0 |       0 |

$ echo $?
0
```

---

## ミュータントの具体例

ツールは以下のような変異を自動生成する。

| 変異の種類 | 例 |
|---|---|
| Conditional Boundary | `x > 0` → `x >= 0` |
| Negate Conditional | `x > 0` → `not (x > 0)` |
| Math Operator | `a + b` → `a - b`, `a * b` → `a / b` |
| Return Value | `return result` → `return None` / `return 0` |
| Boolean Literal | `if cond:` → `if True:` |
| String | `"foo"` → `""` |

---

## Python（mutmut）

### 1. インストール

```bash
cd backend
uv pip install mutmut
```

### 2. 設定ファイル

`backend/pyproject.toml` に追記：

```toml
[tool.mutmut]
paths_to_mutate = "src/domain/"
backup = false
runner = "python -m pytest tests/ -x -q"
tests_dir = "tests/"
```

`src/domain/` のみをミューテーション対象にする。FastAPI ルーターやDBモデルは対象外にすることで実行時間を短縮できる。

### 3. 実行

```bash
cd backend
# ミューテーション実行（初回は数分かかる）
uv run mutmut run

# 結果確認
uv run mutmut results

# HTML レポート生成
uv run mutmut html
# → mutmut-results/ ディレクトリに index.html が生成される
```

### 4. CI ゲートの設定

`scripts/check-mutmut-score.sh`:

```bash
#!/bin/bash
set -e
cd backend
uv run mutmut run

SURVIVED=$(uv run mutmut results 2>&1 | grep "Survived" | grep -oE '[0-9]+' | head -1 || echo "0")
KILLED=$(uv run mutmut results 2>&1 | grep "Killed" | grep -oE '[0-9]+' | head -1 || echo "0")
TOTAL=$((SURVIVED + KILLED))

if [ "$TOTAL" -eq 0 ]; then
    echo "mutmut: ミュータント生成なし"
    exit 0
fi

SCORE=$(python -c "print(round($KILLED / $TOTAL * 100, 1))")
echo "Mutation score: $SCORE% (killed=$KILLED, survived=$SURVIVED)"

if python -c "exit(0 if $KILLED / $TOTAL >= 0.75 else 1)"; then
    echo "PASS mutation-score >= 75%"
else
    echo "FAIL mutation-score < 75%"
    exit 1
fi
```

### 5. 生き残ったミュータントへの対応例

例えば以下のミュータントが生き残ったとする：

```python
# 元（src/domain/lot.py）
if lot.lot_number.year > 2020:
    ...
# ミュータント（Conditional Boundary）
if lot.lot_number.year >= 2020:
    ...
```

PBT が境界値 `year=2020` をカバーしていないのが原因。hypothesis プロパティに境界値を追加：

```python
@given(st.integers(min_value=2020, max_value=2021))
def test_lot_year_boundary(year: int) -> None:
    """Year=2020 は無効として扱う"""
    with pytest.raises(ValueError):
        LotNumber(year=2020, month=1, seq=1)
```

---

## TypeScript（Stryker）

### 1. インストール

```bash
cd frontend
pnpm add -D @stryker-mutator/core @stryker-mutator/vitest-runner
```

### 2. 設定ファイル

`frontend/stryker.config.json`:

```json
{
  "$schema": "./node_modules/@stryker-mutator/core/schema/stryker-schema.json",
  "testRunner": "vitest",
  "reporters": ["html", "clear-text", "progress"],
  "htmlReporter": { "fileName": "../ci-results/stryker/index.html" },
  "coverageAnalysis": "perTest",
  "thresholds": {
    "high": 80,
    "low": 60,
    "break": 60
  },
  "mutate": [
    "src/domain/**/*.ts",
    "!src/**/*.test.ts",
    "!src/**/*.spec.ts"
  ]
}
```

`break: 60` は「Mutation Score が 60% を下回ったら CI を失敗させる」という意味。

### 3. 実行

```bash
cd frontend
pnpm stryker run
```

レポートは `ci-results/stryker/index.html`。

---

## ci.sh への追加

```bash
echo "=== ミューテーションテスト - Python ==="
mkdir -p ci-results/mutmut
bash scripts/check-mutmut-score.sh

echo "=== ミューテーションテスト - TypeScript ==="
mkdir -p ci-results/stryker
cd frontend && pnpm stryker run && cd ..
```

CI 環境ではミューテーションテストは時間がかかる（数分〜十数分）ため、main ブランチへのマージ前のみ実行する運用も多い。`pre-merge` ジョブとして分離するのも合理的。

---

## RALPHループにおける位置づけ

| 層 | 役割 |
|---|---|
| 検証強度 (Verification rigor) | テストの品質そのものを定量評価する |
| Skills のフィードバック源 | エージェントが書いた PBT が形骸化していないか検出 |

エージェントが「`assert True` を並べてカバレッジを稼ぐ」「条件式を雑に書いて PBT が境界値を見逃す」といった抜け道を取れないようにする品質ゲート。Step 8 の PBT を実質的に保証する。

---

## 次のステップ

Step 22が完了したら [Step 23: アーキテクチャ適合性検査](./step23.md) へ進む。
