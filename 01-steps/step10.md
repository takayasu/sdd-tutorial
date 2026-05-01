# Step 10: gitleaks + SCA

## 目的

### これは何か

2つのセキュリティツールをCIに組み込む：

- **gitleaks**：コードの中にパスワードやAPIキーなどの秘密情報（シークレット）がうっかり含まれていないかを検出するツール
- **Trivy（SCA）**：プロジェクトが使っているライブラリ（依存パッケージ）に既知の脆弱性がないかをスキャンするツール

### なぜやるのか

- パスワードやAPIキーをGitにコミットしてしまうと、リポジトリにアクセスできる全員に漏洩する。一度コミットすると履歴に残るため、削除が非常に困難
- 使っているライブラリに脆弱性が見つかることは日常的に起きる。放置すると攻撃の入口になる

### 何がうれしいのか

- AIがコードを生成する際にうっかりシークレットを含めてしまっても、CIで検出できる
- ライブラリの脆弱性を自動検出し、アップデートすべきタイミングがわかる
- セキュリティ事故を「仕組み」で防げる

## 完了条件

```bash
# gitleaks: シークレットが検出されないこと
$ gitleaks detect --source . --exit-code 1 --verbose
    ○
    │╲
    │ ○
    ○ ░
    ░    gitleaks

No leaks found.
$ echo $?
0

# Trivy: HIGH/CRITICAL脆弱性が検出されないこと
$ trivy fs --scanners vuln --severity HIGH,CRITICAL .
2024-04-01T10:00:00.000+0900  INFO  Vulnerability scanning is enabled
...
Total: 0 (HIGH: 0, CRITICAL: 0)
$ echo $?
0

# もしシークレットが検出された場合
$ gitleaks detect --source . --exit-code 1
Finding:     password = "hardcoded_secret"
Secret:      hardcoded_secret
File:        src/config.fs
$ echo $?
1
```

---

## gitleaks（共通）

### 1. 実行

```bash
# リポジトリ全体をスキャン
gitleaks detect --source . --verbose

# CIでは非ゼロ終了コードで失敗させる
gitleaks detect --source . --exit-code 1
```

### 2. 設定ファイル（オプション）

```bash
# .gitleaks.toml（許可リスト等のカスタマイズ）
cat > .gitleaks.toml << 'EOF'
[allowlist]
description = "許可リスト"
paths = [
    '''\.env\.example''',
    '''docker-compose\.yml'''
]
EOF
```

### 3. pre-commit hook（オプション）

```bash
# .git/hooks/pre-commit
#!/bin/bash
gitleaks protect --staged --exit-code 1
```

---

## Trivy（SCA: パッケージ脆弱性スキャン）

### F#

```bash
cd ../sales-management/apps/api-fsharp

# .NETの脆弱性チェック（標準コマンド）
dotnet list package --vulnerable --include-transitive

# Trivyでファイルシステムスキャン
trivy fs --scanners vuln --severity HIGH,CRITICAL .
```

### Kotlin

```bash
cd kotlin

# Trivyでファイルシステムスキャン（build.gradle.ktsの依存関係を解析）
trivy fs --scanners vuln --severity HIGH,CRITICAL .
```

---

## ci.sh への追加

```bash
echo "=== シークレット検出 ==="
gitleaks detect --source . --exit-code 1

echo "=== パッケージ脆弱性スキャン ==="
# F#:
dotnet list package --vulnerable --include-transitive
trivy fs --scanners vuln --severity HIGH,CRITICAL .

# Kotlin:
trivy fs --scanners vuln --severity HIGH,CRITICAL .
```

---

## ci.sh の現時点の構成

```bash
#!/bin/bash
set -e

echo "=== マイグレーション ==="
echo "=== ビルド ==="
echo "=== フォーマットチェック ==="
echo "=== リンター ==="
echo "=== テスト + カバレッジ ==="
echo "=== シークレット検出 ==="
gitleaks detect --source . --exit-code 1

echo "=== パッケージ脆弱性スキャン ==="
trivy fs --scanners vuln --severity HIGH,CRITICAL .
```

---

## 次のステップ

Step 10が完了したら [Step 11: SAST](./step11.md) へ進む。
