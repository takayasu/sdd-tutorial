# Step 4: フォーマッター導入

## 目的

### これは何か

コードの書き方（インデント、改行、クォートスタイルなど）を自動統一するツールを導入する。

- Python: **ruff format**（Black互換の高速フォーマッター）
- TypeScript/React: **Prettier**

### なぜやるのか

- 「タブかスペースか」「引用符はシングルかダブルか」といった議論をゼロにする
- コードレビューで「スタイルの違い」が混在しなくなり、本質的な変更だけを見られる
- AIが生成したコードも自動整形されるため、一貫したスタイルを保てる

### 何がうれしいのか

- `--check` モードでCIに組み込み、スタイル違反があるとCIが落ちる
- `--fix` モードで自動修正される
- チーム全員が同じ設定を使うため「自分の環境では通るのに」がなくなる

## 完了条件

```bash
# Python: フォーマットチェック
$ cd backend && ruff format --check src/ tests/
All checks passed!
$ echo $?
0

# Python: 自動修正（開発時）
$ ruff format src/ tests/
2 files reformatted

# TypeScript: フォーマットチェック
$ cd frontend && npx prettier --check "src/**/*.{ts,tsx}"
Checking formatting...
All matched files use Prettier code style!
$ echo $?
0

# ci.sh が緑
$ ./ci.sh
=== フォーマットチェック (Python) ===
All checks passed!
=== フォーマットチェック (TypeScript) ===
All matched files use Prettier code style!
```

---

## Python（ruff format）

### 1. ruff の設定（pyproject.toml に追記）

```toml
[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
```

### 2. 実行

```bash
cd backend

# チェックのみ（CI用）
ruff format --check src/ tests/

# 自動修正（開発時）
ruff format src/ tests/
```

### 3. VS Code 設定（.vscode/settings.json）

```json
{
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true
  }
}
```

---

## TypeScript/React（Prettier）

### 1. .prettierrc

```json
{
  "semi": false,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "all",
  "printWidth": 100
}
```

### 2. .prettierignore

```
node_modules
dist
coverage
```

### 3. 実行

```bash
cd frontend

# チェックのみ（CI用）
npx prettier --check "src/**/*.{ts,tsx}"

# 自動修正（開発時）
npx prettier --write "src/**/*.{ts,tsx}"
```

### 4. VS Code 設定

```json
{
  "[typescript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode",
    "editor.formatOnSave": true
  },
  "[typescriptreact]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode",
    "editor.formatOnSave": true
  }
}
```

---

## package.json にスクリプト追加

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "format": "prettier --write \"src/**/*.{ts,tsx}\"",
    "format:check": "prettier --check \"src/**/*.{ts,tsx}\"",
    "lint": "eslint src/",
    "test": "vitest",
    "test:coverage": "vitest run --coverage"
  }
}
```

---

## ci.sh への追加

```bash
echo "=== フォーマットチェック (Python) ==="
cd backend
ruff format --check src/ tests/
cd ..

echo "=== フォーマットチェック (TypeScript) ==="
cd frontend
npx prettier --check "src/**/*.{ts,tsx}"
cd ..
```

---

## 次のステップ

Step 4が完了したら [Step 5: リンター導入](./step05.md) へ進む。
