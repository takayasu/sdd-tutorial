# Step 10: gitleaks + SCA（依存脆弱性スキャン）

## 目的

### これは何か

2 種類のセキュリティスキャンを CI に組み込む。

- **gitleaks** — コードに混入した API キーやパスワードなどの秘密情報を検出する
- **SCA（Software Composition Analysis）** — 依存ライブラリの既知脆弱性（CVE）を検出する
  - Python: **pip-audit**
  - Node.js: **pnpm audit**

### なぜやるのか

- AIが生成したコードに `API_KEY = "sk-..."` のような秘密情報がハードコードされることがある
- 古いバージョンのライブラリに CVE が存在しても、手動では気づきにくい
- CI に組み込むことで「マージ前に自動検出」できる

### 何がうれしいのか

- `gitleaks detect` が git 履歴全体をスキャンする（過去のコミットで削除しても検出される）
- `pip-audit` が `pyproject.toml` の依存を自動解析して脆弱性 DB と照合する
- 0 件なら CI が緑、1 件でも赤になる

## 完了条件

```bash
# gitleaks スキャン（秘密情報なし）
$ gitleaks detect --source . --no-git
No leaks found.
$ echo $?
0

# Python SCA
$ cd backend && pip-audit
No known vulnerabilities found
$ echo $?
0

# Node.js SCA
$ cd frontend && pnpm audit
No known vulnerabilities found
$ echo $?
0

# ci.sh が緑
$ ./ci.sh
=== gitleaks ===
No leaks found.
=== SCA (Python) ===
No known vulnerabilities found
=== SCA (Node.js) ===
No known vulnerabilities found
```

---

## gitleaks

### 1. インストール（Step 0 で導入済み）

```bash
# macOS
brew install gitleaks

# Linux / WSL
curl -sSfL https://github.com/gitleaks/gitleaks/releases/latest/download/gitleaks_linux_x64.tar.gz \
  | tar xz -C /usr/local/bin gitleaks
```

### 2. .gitleaks.toml（プロジェクトルートに配置）

```toml
title = "gitleaks config"

[extend]
useDefault = true

[[rules]]
description = "Ignore example .env files"
id = "ignore-env-example"
path = ".env.example"
```

### 3. .gitignore に追加

```
.env
.env.local
*.pem
*.key
```

### 4. 実行

```bash
# git 履歴全体をスキャン
gitleaks detect --source .

# git なしのディレクトリスキャン（CI用）
gitleaks detect --source . --no-git

# レポート出力
gitleaks detect --source . --report-path gitleaks-report.json
```

---

## Python SCA（pip-audit）

### 1. 依存追加（pyproject.toml）

```toml
[project.optional-dependencies]
dev = [
    ...
    "pip-audit>=2.7",
]
```

```bash
cd backend && uv sync
```

### 2. 実行

```bash
cd backend

# 脆弱性チェック
pip-audit

# 修正可能な脆弱性を自動修正
pip-audit --fix
```

---

## Node.js SCA（pnpm audit）

### 1. 実行

```bash
cd frontend

# 脆弱性チェック
pnpm audit

# 重大度 high 以上のみチェック（CI用）
pnpm audit --audit-level=high
```

---

## ci.sh への追加

```bash
echo "=== gitleaks ==="
gitleaks detect --source . --no-git

echo "=== SCA (Python) ==="
cd backend
pip-audit
cd ..

echo "=== SCA (Node.js) ==="
cd frontend
pnpm audit --audit-level=high
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

echo "=== gitleaks ==="
gitleaks detect --source . --no-git

echo "=== SCA (Python) ==="
cd backend && pip-audit && cd ..

echo "=== SCA (Node.js) ==="
cd frontend && pnpm audit --audit-level=high && cd ..

echo "=== verify (smoke) ==="
# Step 1 で導入済み
```

---

## 次のステップ

Step 10が完了したら [Step 11: SAST（静的アプリケーションセキュリティテスト）](./step11.md) へ進む。
