# Step 11: SAST（静的アプリケーションセキュリティテスト）

## 目的

### これは何か

ソースコードを実行せずにセキュリティ脆弱性を検出する静的解析（SAST）を導入する。

- Python: **bandit**（OWASP Top 10 に対応した Python 専用 SAST ツール）
- TypeScript: ESLint の **eslint-plugin-security** ルール群（Step 5 の ESLint に追加）

### なぜやるのか

- AIが生成したコードは `eval()` や `subprocess.shell=True` のような危険なパターンを使うことがある
- `hardcoded_password` / `sql_injection` / `weak_cryptographic_key` を人間のレビューより先に検出できる
- CI に組み込むことで「マージ前にセキュリティ問題を自動ブロック」できる

### 何がうれしいのか

- `bandit -r src/` が OWASP カテゴリ付きで問題箇所を出力する
- 深刻度（HIGH/MEDIUM/LOW）と確信度（HIGH/MEDIUM/LOW）の組み合わせで優先順位をつけられる
- `# nosec B601` でホワイトリスト例外を明示的に記録できる

## 完了条件

```bash
# Python SAST（MEDIUM 重大度・MEDIUM 確信度以上のみ CI でブロック）
$ cd backend && bandit -r src/ -ll -ii
Run started: ...
No issues identified.
$ echo $?
0

# TypeScript SAST（ESLint security rules）
$ cd frontend && npx eslint src/
（エラーなし）

# ci.sh が緑
$ ./ci.sh
=== SAST (Python) ===
No issues identified.
=== SAST (TypeScript) ===
（エラーなし）
```

---

## Python（bandit）

### 1. 依存追加（pyproject.toml）

```toml
[project.optional-dependencies]
dev = [
    ...
    "bandit>=1.7",
]
```

```bash
cd backend && uv sync
```

### 2. bandit 設定（pyproject.toml）

```toml
[tool.bandit]
exclude_dirs = ["tests", "alembic"]
skips = [
    "B101",  # assert文（テストでの使用を許可）
]
```

### 3. 実行

```bash
cd backend

# 全検出（開発時確認用）
bandit -r src/

# MEDIUM重大度・MEDIUM確信度以上のみ（CI用）
bandit -r src/ -ll -ii

# レポート出力
bandit -r src/ -f json -o bandit-report.json
```

### 4. 重大度フラグの意味

| フラグ | 意味 |
|---|---|
| `-l` | LOW 以上を表示（デフォルト） |
| `-ll` | MEDIUM 以上を表示 |
| `-lll` | HIGH のみ表示 |
| `-i` | 確信度 LOW 以上（デフォルト） |
| `-ii` | 確信度 MEDIUM 以上 |
| `-iii` | 確信度 HIGH のみ |

CI では `-ll -ii`（MEDIUM以上・MEDIUM以上）を推奨。初期は `-lll -iii` から始めて徐々に厳しくする。

### 5. よくある検出例

```python
# B602: subprocess の shell=True
import subprocess
subprocess.run("ls " + user_input, shell=True)  # NG: コマンドインジェクション

# B311: 暗号学的に安全でない乱数
import random
token = random.randint(0, 2**32)  # NG

import secrets
token = secrets.token_hex(32)  # OK

# B106: ハードコードされたパスワード
password = "admin123"  # NG

password = os.getenv("DB_PASSWORD")  # OK
```

---

## TypeScript（eslint-plugin-security）

### 1. 依存追加

```bash
cd frontend
pnpm add -D eslint-plugin-security
```

### 2. eslint.config.js に追加

```javascript
import js from '@eslint/js'
import tseslint from 'typescript-eslint'
import reactHooks from 'eslint-plugin-react-hooks'
import reactRefresh from 'eslint-plugin-react-refresh'
import security from 'eslint-plugin-security'

export default tseslint.config(
  { ignores: ['dist', 'coverage'] },
  {
    extends: [js.configs.recommended, ...tseslint.configs.recommended],
    files: ['**/*.{ts,tsx}'],
    plugins: {
      'react-hooks': reactHooks,
      'react-refresh': reactRefresh,
      security,
    },
    rules: {
      ...reactHooks.configs.recommended.rules,
      'react-refresh/only-export-components': ['warn', { allowConstantExport: true }],
      '@typescript-eslint/no-unused-vars': 'error',
      '@typescript-eslint/no-explicit-any': 'warn',
      'security/detect-object-injection': 'warn',
      'security/detect-non-literal-regexp': 'warn',
      'security/detect-unsafe-regex': 'error',
      'security/detect-eval-with-expression': 'error',
    },
  },
)
```

---

## ci.sh への追加

```bash
echo "=== SAST (Python) ==="
cd backend
bandit -r src/ -ll -ii
cd ..
```

（TypeScript は Step 5 の ESLint が security rules も含めて実行するため追加コマンド不要）

---

## 次のステップ

Step 11が完了したら [Step 12: 販売案件ドメイン追加](./step12.md) へ進む。
