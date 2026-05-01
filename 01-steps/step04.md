# Step 4: フォーマッター導入

## 目的

### これは何か

コードフォーマッター（Fantomas / ktfmt）を導入する。フォーマッターとは、コードのインデント（字下げ）、改行位置、スペースの入れ方などを自動的に統一するツール。

### なぜやるのか

- 人によってコードの書き方（見た目）がバラバラだと、差分（diff）が読みにくくなる
- 「タブかスペースか」「括弧の位置」といった議論に時間を使わなくて済む
- AIが生成したコードも自動的に統一されたスタイルになる

### 何がうれしいのか

- コードレビューで「見た目」の指摘がゼロになる。ロジックの議論に集中できる
- CIで `--check` モードを使えば、フォーマットされていないコードがマージされることを防げる
- `dotnet fantomas src/` / `gradle ktfmtFormat` を実行するだけで全コードが整形される

## 完了条件

### F#（Fantomas）

```bash
# フォーマット適用
$ cd ../sales-management/apps/api-fsharp
$ dotnet fantomas src/
Formatted: src/SalesManagement/Program.fs

# チェックモード（CI用）— フォーマット済みなら成功
$ dotnet fantomas --check src/
All files are correctly formatted.
$ echo $?
0

# もしフォーマット違反があった場合
$ dotnet fantomas --check src/
src/SalesManagement/Program.fs was not formatted correctly.
$ echo $?
1
```

### Kotlin（ktfmt）

```bash
# フォーマット適用
$ cd kotlin
$ gradle ktfmtFormat
BUILD SUCCESSFUL in Xs

# チェックモード（CI用）— フォーマット済みなら成功
$ gradle ktfmtCheck
BUILD SUCCESSFUL in Xs

# もしフォーマット違反があった場合
$ gradle ktfmtCheck
> Task :ktfmtCheck FAILED
src/main/kotlin/salesmanagement/Application.kt: Failed to format
FAILURE: Build failed with an exception.
```

---

## F#（Fantomas）

### 1. 設定ファイル作成

```bash
# fsharp/.editorconfig
cat > fsharp/.editorconfig << 'EOF'
root = true

[*.fs]
indent_size = 4
max_line_length = 120
fsharp_multiline_bracket_style = cramped
fsharp_newline_before_multiline_computation_expression = false
EOF
```

### 2. フォーマット実行

```bash
cd ../sales-management/apps/api-fsharp
# フォーマット適用
dotnet fantomas src/

# チェックのみ（CI用）
dotnet fantomas --check src/
```

### 3. ci.sh への追加

```bash
echo "=== フォーマットチェック ==="
dotnet fantomas --check src/
if [ $? -ne 0 ]; then
    echo "フォーマット違反があります。dotnet fantomas src/ を実行してください。"
    exit 1
fi
```

---

## Kotlin（ktfmt）

### 1. Gradle設定追加（build.gradle.kts）

```kotlin
plugins {
    // 既存のpluginsに追加
    id("com.ncorti.ktfmt.gradle") version "0.18.0"
}

ktfmt {
    kotlinLangStyle()
}
```

### 2. フォーマット実行

```bash
cd kotlin
# フォーマット適用
gradle ktfmtFormat

# チェックのみ（CI用）
gradle ktfmtCheck
```

### 3. ci.sh への追加

```bash
echo "=== フォーマットチェック ==="
gradle ktfmtCheck
if [ $? -ne 0 ]; then
    echo "フォーマット違反があります。gradle ktfmtFormat を実行してください。"
    exit 1
fi
```

---

## ci.sh の現時点の構成

```bash
#!/bin/bash
set -e

echo "=== ビルド ==="
# F#: dotnet build --warnaserror
# Kotlin: gradle build

echo "=== フォーマットチェック ==="
# F#: dotnet fantomas --check src/
# Kotlin: gradle ktfmtCheck
```

---

## 次のステップ

Step 4が完了したら [Step 5: リンター導入](./step05.md) へ進む。
