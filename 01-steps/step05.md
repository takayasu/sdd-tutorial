# Step 5: リンター導入

## 目的

### これは何か

コードの「品質上の問題」を自動検出するツールを導入する。フォーマッターが「見た目」を整えるのに対し、リンターは「意味的な問題」を検出する。

- Python: **ruff check**（flake8 + isort + pyupgrade 等を統合した高速リンター）
- TypeScript/React: **ESLint**

### なぜやるのか

- 未使用変数、到達不能なコード、型の誤使用などを自動検出する
- AIが生成したコードに含まれる潜在的なバグをCIで早期発見する
- セキュリティリスクになるコードパターン（例: `eval()` の使用）を検出できる

### 何がうれしいのか

- 人間のコードレビューで指摘するような定型的な問題をツールが代行する
- コードレビューの議論が「実質的な問題」に集中できる

## 完了条件

```bash
# Python: リントチェック
$ cd backend && ruff check src/ tests/
All checks passed!
$ echo $?
0

# TypeScript: リントチェック
$ cd frontend && npx eslint src/
$ echo $?
0

# ci.sh が緑
$ ./ci.sh
=== リンター (Python) ===
All checks passed!
=== リンター (TypeScript) ===
（警告・エラーなし）
```

---

## Python（ruff check）

### 1. ruff の lint 設定（pyproject.toml に追記）

```toml
[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "F",    # pyflakes
    "I",    # isort
    "UP",   # pyupgrade
    "B",    # flake8-bugbear
    "SIM",  # flake8-simplify
    "N",    # pep8-naming
]
ignore = [
    "S101",  # assert 文（テストでの使用を許可）
]

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["B"]
```

### 2. 実行

```bash
cd backend

# チェックのみ（CI用）
ruff check src/ tests/

# 自動修正できる問題を修正（開発時）
ruff check --fix src/ tests/
```

### 3. よくある検出例

```python
# F401: 未使用インポート → 削除される
import os  # 使っていない場合

# B006: ミュータブルなデフォルト引数
def bad(items: list = []) -> list:  # NG
    return items

def good(items: list | None = None) -> list:  # OK
    return items or []

# UP006: 古い型ヒント
from typing import List  # NG
items: List[str]

items: list[str]  # OK（Python 3.9+）
```

---

## TypeScript/React（ESLint）

### 1. eslint.config.js

```javascript
import js from '@eslint/js'
import tseslint from 'typescript-eslint'
import reactHooks from 'eslint-plugin-react-hooks'
import reactRefresh from 'eslint-plugin-react-refresh'

export default tseslint.config(
  { ignores: ['dist', 'coverage'] },
  {
    extends: [js.configs.recommended, ...tseslint.configs.recommended],
    files: ['**/*.{ts,tsx}'],
    plugins: {
      'react-hooks': reactHooks,
      'react-refresh': reactRefresh,
    },
    rules: {
      ...reactHooks.configs.recommended.rules,
      'react-refresh/only-export-components': ['warn', { allowConstantExport: true }],
      '@typescript-eslint/no-unused-vars': 'error',
      '@typescript-eslint/no-explicit-any': 'warn',
    },
  },
)
```

### 2. 実行

```bash
cd frontend

# チェックのみ（CI用）
npx eslint src/

# 自動修正（開発時）
npx eslint --fix src/
```

---

## ci.sh への追加

```bash
echo "=== リンター (Python) ==="
cd backend
ruff check src/ tests/
cd ..

echo "=== リンター (TypeScript) ==="
cd frontend
npx eslint src/
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

echo "=== verify (smoke) ==="
# Step 1 で導入済み
```

---

## 次のステップ

Step 5が完了したら [Step 6: 在庫ロットの型定義](./step06.md) へ進む。
