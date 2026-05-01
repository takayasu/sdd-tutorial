# Step 5: リンター導入

## 目的

### これは何か

リンター（FSharpLint / detekt）を導入する。リンターとは、コードの「品質上の問題」を自動検出するツール。フォーマッターが「見た目」を整えるのに対し、リンターは「書き方の良し悪し」を指摘する。

### なぜやるのか

検出する問題の例：
- 関数が長すぎる（50行以上 → 分割すべき）
- ネストが深すぎる（if文の中にif文の中にif文... → 読みにくい）
- マジックナンバー（`if x > 86400` → `if x > SECONDS_PER_DAY` にすべき）
- 使われていない変数

### 何がうれしいのか

- AIが生成したコードの品質を自動チェックできる。「動くけど読みにくいコード」を防げる
- チーム全体のコード品質が底上げされる
- CIに組み込むことで、品質基準を満たさないコードがマージされることを防げる

## 完了条件

### F#（FSharpLint）

```bash
$ cd ../sales-management/apps/api-fsharp
$ dotnet tool run fsharplint lint src/SalesManagement/SalesManagement.fsproj
========== Linting src/SalesManagement/SalesManagement.fsproj ==========
No lint warnings found.
$ echo $?
0
```

### Kotlin（detekt）

```bash
$ cd kotlin
$ gradle detekt
> Task :detekt
BUILD SUCCESSFUL in Xs

# 警告がある場合
$ gradle detekt
> Task :detekt FAILED
src/main/kotlin/salesmanagement/Application.kt:15:5: MagicNumber - ...
FAILURE: Build failed with an exception.
```

---

## F#（FSharpLint）

### 1. ツールマニフェスト作成

```bash
cd ../sales-management/apps/api-fsharp
dotnet new tool-manifest
dotnet tool install dotnet-fsharplint
```

### 2. 設定ファイル作成

```bash
# fsharp/fsharplint.json
cat > fsharp/fsharplint.json << 'EOF'
{
  "typedChecks": {
    "enabled": true
  },
  "conventions": {
    "recursiveAsyncFunction": { "enabled": true },
    "redundantNewKeyword": { "enabled": true },
    "nestedStatements": {
      "enabled": true,
      "config": { "depth": 5 }
    },
    "numberOfItems": {
      "maxFunctionDefinitionParameters": { "enabled": true, "config": { "maxItems": 5 } },
      "maxTupleSize": { "enabled": true, "config": { "maxItems": 4 } }
    },
    "sourceLength": {
      "maxLinesInFunction": { "enabled": true, "config": { "maxLines": 50 } },
      "maxLinesInModule": { "enabled": true, "config": { "maxLines": 500 } }
    }
  }
}
EOF
```

### 3. 実行

```bash
dotnet tool run fsharplint lint src/SalesManagement/SalesManagement.fsproj
```

### 4. ci.sh への追加

```bash
echo "=== リンター ==="
dotnet tool run fsharplint lint src/SalesManagement/SalesManagement.fsproj
```

---

## Kotlin（detekt）

### 1. Gradle設定追加（build.gradle.kts）

```kotlin
plugins {
    // 既存のpluginsに追加
    id("io.gitlab.arturbosch.detekt") version "1.23.6"
}

detekt {
    config.setFrom("detekt.yml")
    buildUponDefaultConfig = true
}
```

### 2. 設定ファイル作成

```bash
# kotlin/detekt.yml
cat > kotlin/detekt.yml << 'EOF'
complexity:
  LongMethod:
    threshold: 30
  ComplexMethod:
    threshold: 10
  CognitiveComplexMethod:
    threshold: 10
  LargeClass:
    threshold: 300
  TooManyFunctions:
    threshold: 15

style:
  MagicNumber:
    active: true
    ignoreNumbers:
      - '-1'
      - '0'
      - '1'
      - '2'
  VarCouldBeVal:
    active: true
  UnnecessaryLet:
    active: true

potential-bugs:
  CastToNullableType:
    active: true
  UnnecessaryNotNullOperator:
    active: true

# --- 関数型スタイル強制ルール ---
# Kotlinを関数型スタイルで書くための追加制約

style:
  # var禁止（immutableデフォルト）
  VarCouldBeVal:
    active: true
  # mutableコレクション使用を検出
  MutableCollectionMutableState:
    active: true

naming:
  # data classのプロパティはval強制（detektデフォルトで検出）
  InvalidPackageDeclaration:
    active: true

complexity:
  # when式の網羅性を強制（sealed classの全ケース処理）
  # → コンパイラが強制するため追加ルール不要だが、else禁止で明示
  ComplexMethod:
    threshold: 10

potential-bugs:
  # !!演算子（強制アンラップ）禁止 → Eitherで処理すべき
  UnnecessaryNotNullOperator:
    active: true
  # as?キャスト禁止 → sealed classのwhenで処理すべき
  CastToNullableType:
    active: true

# カスタムルール案（detekt custom rule or forbiddenMethodCall で実現）
forbidden:
  ForbiddenMethodCall:
    active: true
    methods:
      # mutableListOf等を禁止
      - 'kotlin.collections.mutableListOf'
      - 'kotlin.collections.mutableMapOf'
      - 'kotlin.collections.mutableSetOf'
      # var宣言を伴うパターンはVarCouldBeValで検出
EOF
```

> **関数型スタイル強制の方針**: `var`禁止・mutableコレクション禁止・`!!`禁止をdetektで静的に検出する。sealed classの網羅的when処理はKotlinコンパイラが`-Werror`で強制する。Arrow（Either/NonEmptyList）の使用はコードレビューとPBTで担保する。

### 3. 実行

```bash
gradle detekt
```

### 4. ci.sh への追加

```bash
echo "=== リンター ==="
gradle detekt
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

echo "=== リンター ==="
# F#: dotnet tool run fsharplint lint src/SalesManagement/SalesManagement.fsproj
# Kotlin: gradle detekt
```

---

## 次のステップ

Step 5が完了したら [Step 6: 在庫ロットの型定義](./step06.md) へ進む。
