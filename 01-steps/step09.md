# Step 9: テストカバレッジ

## 目的

### これは何か

テストカバレッジ（コードのうち、テストで実行された割合）を計測する。例えば「カバレッジ85%」は、コード全体の85%がテスト実行時に通過したことを意味する。

### なぜやるのか

- テストを書いたつもりでも、実は通っていないコードパスがあるかもしれない
- カバレッジを可視化することで「テストが足りていない箇所」が一目でわかる
- 品質の客観的な指標として使える

### 何がうれしいのか

- 「テストは書いたけど本当に十分か？」という不安に数値で答えられる
- HTMLレポートで「どの行が通っていないか」を視覚的に確認できる
- CIで計測し続けることで、カバレッジが下がったら気づける

## 完了条件

### F#

```bash
$ cd ../sales-management/apps/api-fsharp
$ dotnet test --collect:"XPlat Code Coverage" --results-directory ./coverage
  Passed!  - Failed:     0, Passed:     3, Skipped:     0, Total:     3

# カバレッジレポートが生成されていることを確認
$ ls coverage/*/coverage.cobertura.xml
coverage/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/coverage.cobertura.xml

# カバレッジ率を確認
$ grep -oP 'line-rate="\K[^"]+' coverage/*/coverage.cobertura.xml
0.85
```

### Kotlin

```bash
$ cd kotlin
$ gradle test jacocoTestReport
BUILD SUCCESSFUL in Xs

# レポートが生成されていることを確認
$ ls build/reports/jacoco/test/html/index.html
build/reports/jacoco/test/html/index.html

# ブラウザで確認可能
# open build/reports/jacoco/test/html/index.html
```

---

## F#（coverlet）

### 1. パッケージ追加

```bash
cd ../sales-management/apps/api-fsharp/tests/SalesManagement.Tests
dotnet add package coverlet.collector
```

### 2. 実行

```bash
cd ../sales-management/apps/api-fsharp
dotnet test --collect:"XPlat Code Coverage" --results-directory ./coverage
```

### 3. レポート生成（オプション）

```bash
# ReportGeneratorでHTML化
dotnet tool install -g dotnet-reportgenerator-globaltool
reportgenerator -reports:coverage/**/coverage.cobertura.xml -targetdir:coverage/report -reporttypes:Html
```

### 4. ci.sh への追加

```bash
echo "=== テスト + カバレッジ ==="
dotnet test --collect:"XPlat Code Coverage" --results-directory ./coverage
```

---

## Kotlin（JaCoCo）

### 1. Gradle設定追加（build.gradle.kts）

```kotlin
plugins {
    // 既存に追加
    jacoco
}

tasks.jacocoTestReport {
    dependsOn(tasks.test)
    reports {
        xml.required.set(true)
        html.required.set(true)
    }
}
```

### 2. 実行

```bash
gradle test jacocoTestReport
# レポートは build/reports/jacoco/test/html/index.html に出力
```

### 3. ci.sh への追加

```bash
echo "=== テスト + カバレッジ ==="
gradle test jacocoTestReport
```

---

## ci.sh の現時点の構成

```bash
#!/bin/bash
set -e

echo "=== マイグレーション ==="
# F#: dotnet run --project tools/Migrator
# Kotlin: gradle flywayMigrate

echo "=== ビルド ==="
# F#: dotnet build --warnaserror
# Kotlin: gradle build

echo "=== フォーマットチェック ==="
# F#: dotnet fantomas --check src/
# Kotlin: gradle ktfmtCheck

echo "=== リンター ==="
# F#: dotnet tool run fsharplint lint src/SalesManagement/SalesManagement.fsproj
# Kotlin: gradle detekt

echo "=== テスト + カバレッジ ==="
# F#: dotnet test --collect:"XPlat Code Coverage" --results-directory ./coverage
# Kotlin: gradle test jacocoTestReport
```

---

## 次のステップ

Step 9が完了したら [Step 10: gitleaks + SCA](./step10.md) へ進む。
