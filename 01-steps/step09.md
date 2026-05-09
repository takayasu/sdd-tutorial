# Step 9: テストカバレッジ

## 目的

### これは何か

テストがコードのどの部分をどれだけ実行しているかを計測し、CI で閾値を強制する。

- Python: **pytest-cov**（`coverage.py` のラッパー）
- TypeScript: **@vitest/coverage-v8**（V8 の組み込みカバレッジエンジン）

### なぜやるのか

- AIが生成したコードに「テストが一行も通っていないパス」が存在することがある
- カバレッジ閾値を CI に組み込むと、テストを書かずにコードだけ追加するプルリクエストがブロックされる
- 80% という数字に意味があるというより、「下がり続けない」ことを自動保証するのが目的

### 何がうれしいのか

- `coverage html` で「赤く塗られた行」が一目でわかる
- ブランチカバレッジで `if` の両方の分岐がテストされているか確認できる
- CIが `Fail — coverage 74%（required: 80%）` と言ってくれる

## 完了条件

```bash
# Python カバレッジ（閾値80%）
$ cd backend && pytest --cov=src --cov-fail-under=80
...
TOTAL   142   8   94%
Required test coverage of 80% reached. Total coverage: 94.00%

# TypeScript カバレッジ
$ cd frontend && npx vitest run --coverage
 % Coverage report from v8
 File        | % Stmts | % Branch | % Funcs | % Lines
 All files   |   88.00 |    85.00 |   90.00 |   88.00

# ci.sh が緑
$ ./ci.sh
=== カバレッジ (Python) ===
Required test coverage of 80% reached.
=== カバレッジ (TypeScript) ===
（閾値超過）
```

---

## Python（pytest-cov）

### 1. 依存追加（pyproject.toml）

```toml
[project.optional-dependencies]
dev = [
    "pytest>=8",
    "pytest-asyncio>=0.23",
    "httpx>=0.27",
    "ruff>=0.6",
    "mypy>=1.10",
    "hypothesis>=6",
    "pytest-cov>=5",
]
```

```bash
cd backend && uv sync
```

### 2. pytest 設定（pyproject.toml）

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
addopts = "--cov=src --cov-branch --cov-report=term-missing --cov-fail-under=80"
```

### 3. カバレッジ設定（pyproject.toml）

```toml
[tool.coverage.run]
branch = true
source = ["src"]
omit = [
    "src/infra/models.py",
]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "if TYPE_CHECKING:",
    "raise NotImplementedError",
]
```

### 4. 実行

```bash
cd backend

# ターミナル出力（CI用）
pytest --cov=src --cov-fail-under=80

# HTML レポート（開発時）
pytest --cov=src --cov-report=html
open htmlcov/index.html
```

---

## TypeScript（@vitest/coverage-v8）

### 1. 依存追加

```bash
cd frontend
pnpm add -D @vitest/coverage-v8
```

### 2. vite.config.ts に追記

```typescript
export default defineConfig({
  // ...
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./tests/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov', 'html'],
      thresholds: {
        lines: 80,
        branches: 80,
        functions: 80,
        statements: 80,
      },
      exclude: [
        'src/main.tsx',
        'src/vite-env.d.ts',
        '**/*.d.ts',
        'tests/**',
      ],
    },
  },
})
```

### 3. 実行

```bash
cd frontend

# ターミナル出力（CI用）
npx vitest run --coverage

# HTML レポート
npx vitest run --coverage --reporter=html
open coverage/index.html
```

### 4. package.json スクリプト更新

```json
{
  "scripts": {
    "test": "vitest",
    "test:coverage": "vitest run --coverage"
  }
}
```

---

## ci.sh への追加

```bash
echo "=== カバレッジ (Python) ==="
cd backend
pytest --cov=src --cov-fail-under=80 -q
cd ..

echo "=== カバレッジ (TypeScript) ==="
cd frontend
npx vitest run --coverage
cd ..
```

## ci.sh の現時点の構成

```bash
#!/bin/bash
set -e

echo "=== DB起動確認 ==="
docker compose up -d db
until docker compose exec db pg_isready -U app -d sales_management >/dev/null 2>&1; do
  sleep 2
done

echo "=== マイグレーション ==="
cd backend && alembic upgrade head && cd ..

echo "=== フォーマットチェック (Python) ==="
cd backend && ruff format --check src/ tests/ && cd ..

echo "=== フォーマットチェック (TypeScript) ==="
cd frontend && npx prettier --check "src/**/*.{ts,tsx}" && cd ..

echo "=== リンター (Python) ==="
cd backend && ruff check src/ tests/ && cd ..

echo "=== リンター (TypeScript) ==="
cd frontend && npx eslint src/ && cd ..

echo "=== カバレッジ (Python) ==="
cd backend && pytest --cov=src --cov-fail-under=80 -q && cd ..

echo "=== カバレッジ (TypeScript) ==="
cd frontend && npx vitest run --coverage && cd ..

echo "=== verify (smoke) ==="
# Step 1 で導入済み
```

---

## 次のステップ

Step 9が完了したら [Step 10: gitleaks + SCA](./step10.md) へ進む。
