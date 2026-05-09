# Step 23: アーキテクチャ適合性検査

## 目的

### これは何か

アーキテクチャ適合性検査とは、ソースコードの依存方向・レイヤ・命名規約がアーキテクチャのルール通りであることをテストとして実行可能にする手法。例えば「Domain 層は Infrastructure 層を import してはならない」というルールをコードで書き、CI で強制する。

- Python: `import-linter`（モジュール間の import 方向を設定ファイルで強制）
- TypeScript: `dependency-cruiser`（ファイル間の依存方向をルールで強制）

### なぜやるのか

- AGENTS.md の「ファイル配置規約」は人間向けの自然言語ルール。AI がコードを大量生成するとレビューを通り抜けてレイヤ違反が混入することがある（例: `domain/workflows.py` で SQLAlchemy を直接 import）
- Step 7 で確立した「Domain → Router → Database」の依存方向はドメインモデルの根幹だが、Phase 1 ではインタープリタ的には強制されていなかった
- 設定ファイルでルールを表現し、CI で実行することで、文章ルールでは防げない退行を機械的に止める

### 何がうれしいのか

- AGENTS.md に書いた「Domain は Infrastructure に依存しない」が実行可能なチェックになる
- 新メンバー（人間 or AI）が誤った方向の依存を追加するとCIが落ちる
- レビューの認知負荷が下がる（人間は「ルールを思い出して照合する」必要がなくなる）

## 完了条件

### Python（import-linter）

```bash
$ cd backend && uv run lint-imports
Checking 4 contracts...
✓ domain-must-not-import-routers
✓ domain-must-not-import-database
✓ routers-must-not-import-database-directly
✓ domain-layer-ordering
All contracts kept!
$ echo $?
0
```

### TypeScript（dependency-cruiser）

```bash
$ cd frontend && pnpm depcruise src --config .dependency-cruiser.json
✓ No forbidden dependencies found.
$ echo $?
0
```

---

## 追加パッケージ

```toml
# backend/pyproject.toml
[project.optional-dependencies]
dev = [
    # ... 既存 ...
    "import-linter>=2.1",
]
```

```bash
cd backend && uv pip install import-linter
# TypeScript
cd frontend && pnpm add -D dependency-cruiser
```

---

## Python（import-linter）

### 1. 設定ファイル

`backend/.importlinter`:

```ini
[importlinter]
root_package = src
include_external_packages = True

[importlinter:contract:domain-must-not-import-routers]
name = Domain must not depend on routers
type = forbidden
source_modules =
    src.domain
forbidden_modules =
    src.routers

[importlinter:contract:domain-must-not-import-database]
name = Domain must not depend on database infrastructure
type = forbidden
source_modules =
    src.domain
forbidden_modules =
    src.database
    src.models

[importlinter:contract:routers-must-not-import-database-directly]
name = Routers must use repositories, not direct DB access
type = forbidden
source_modules =
    src.routers
forbidden_modules =
    sqlalchemy
    asyncpg

[importlinter:contract:domain-layer-ordering]
name = Layer ordering: domain → routers → database
type = layers
layers =
    src.routers
    src.domain
```

### 2. 実行

```bash
cd backend
uv run lint-imports
```

### 3. 追加の pytest ベースチェック

`import-linter` でカバーできない「Domain 型がすべて frozen dataclass か」を pytest で検証する：

`tests/test_architecture.py`:

```python
import dataclasses
import importlib
import inspect
import pkgutil
import src.domain as domain_pkg


def test_domain_types_are_frozen_dataclasses():
    """Domain 型はすべて frozen=True の dataclass であること"""
    violations = []
    for _, module_name, _ in pkgutil.walk_packages(
        domain_pkg.__path__, prefix=domain_pkg.__name__ + "."
    ):
        module = importlib.import_module(module_name)
        for name, obj in inspect.getmembers(module, inspect.isclass):
            if obj.__module__ != module_name:
                continue
            if name.startswith("_"):
                continue
            if dataclasses.is_dataclass(obj):
                if not obj.__dataclass_params__.frozen:
                    violations.append(f"{module_name}.{name} is not frozen")
    assert not violations, "\n".join(violations)
```

---

## TypeScript（dependency-cruiser）

### 1. 設定ファイル

`frontend/.dependency-cruiser.json`:

```json
{
  "$schema": "node_modules/dependency-cruiser/schema/configuration.schema.json",
  "forbidden": [
    {
      "name": "domain-no-api",
      "comment": "Domain types must not import from API/component layer",
      "severity": "error",
      "from": { "path": "^src/domain" },
      "to": { "path": "^src/(components|pages|api)" }
    },
    {
      "name": "domain-no-external-http",
      "comment": "Domain must not call external HTTP directly",
      "severity": "error",
      "from": { "path": "^src/domain" },
      "to": { "path": "^(axios|fetch|node-fetch)" }
    },
    {
      "name": "types-no-runtime",
      "comment": "Type definition files must not import runtime code",
      "severity": "error",
      "from": { "path": "\\.d\\.ts$" },
      "to": { "pathNot": "\\.d\\.ts$" }
    }
  ],
  "options": {
    "doNotFollow": { "path": "node_modules" },
    "tsConfig": { "fileName": "tsconfig.json" }
  }
}
```

### 2. 実行

```bash
cd frontend
pnpm depcruise src --config .dependency-cruiser.json

# 依存関係グラフ（SVG）を生成して確認
pnpm depcruise src --output-type dot | dot -T svg > ci-results/dep-graph.svg
```

---

## ルールの追加方針

エージェントがコードを書くたびに違反したくなる典型例をルール化する：

| 違反パターン | 対応するルール |
|---|---|
| `domain/workflows.py` で `sqlalchemy` を直接 import | `domain-must-not-import-database` |
| `routers/` で `session.execute()` を直接呼ぶ | `routers-must-not-import-database-directly` |
| Domain dataclass が `frozen=False` になる | `test_domain_types_are_frozen_dataclasses` |
| TypeScript の domain が API クライアントを呼ぶ | `domain-no-external-http` |

---

## ci.sh への追加

```bash
echo "=== アーキテクチャ適合性 - Python ==="
cd backend && uv run lint-imports && cd ..
cd backend && uv run pytest tests/test_architecture.py -q && cd ..

echo "=== アーキテクチャ適合性 - TypeScript ==="
cd frontend && pnpm depcruise src --config .dependency-cruiser.json && cd ..
```

---

## RALPHループにおける位置づけ

| 層 | 役割 |
|---|---|
| Constraints (制約) | AGENTS.md の文章ルールを実行可能なチェックに昇格 |
| Protocols | Domain / Router / Infrastructure の境界をエージェント間で共有 |

エージェントが大量にコード追加するとき、人間レビューでは見逃される構造ドリフトを機械が止める。Step 28 の AGENTS.md 自動更新と相補的：

- **Step 28** は「過去の失敗を文章で蓄積」する soft な学習
- **Step 23** は「設計ルールを破壊不可能なコードで強制」する hard な制約

両方が揃うと、ハーネスは「学んでも、決して壊さない」状態になる。

---

## 次のステップ

Step 23が完了したら [Step 24: APIコントラクトテスト](./step24.md) へ進む。
