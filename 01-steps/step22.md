# Step 22: ミューテーションテスト

> **ステータス: SKIP (2026-04-27)** — Stryker.NET は F# プロジェクトを未サポート (3.13.0 / 4.14.1 双方で `Language not supported: Fsharp` または `Mutation testing of F# projects is not ready yet`)。GitHub Issue #1216 (F# Support) は 2020 年から open のまま、Issue #2525 (Move fsharp support out of main codebase) は 2025/01 に "not planned" でクローズ。Faultify などの IL レベルミューテータは研究プロジェクトでメンテ停止のため採用見送り。F# 用 mutation tester が成熟したタイミングで再開する。
>
> 本リポジトリでは本ステップに該当する成果物 (`fsharp/stryker-config.json`, `ci-results/stryker/`, ci.sh の Stryker ステージ) は生成していない。

## 目的

### これは何か

ミューテーションテストとは、本番コードに機械的な小さな変更（ミュータント）を加え、既存のテストがそれを検出できるかを測る手法。例えば `if (x > 0)` を `if (x >= 0)` に変えたバージョンに対してテストが落ちれば「ミュータントを殺せた」、落ちなければ「ミュータントが生き残った」となる。

スコアは `killed / total mutants × 100%` で表され、テストの「厳しさ」の客観指標になる。

### なぜやるのか

- Step 9 で導入したカバレッジは「テストが行を通った」だけを測る。テストが `assert` をしていなくても、あるいは `if` 条件が逆でも通る弱いアサーションでも、カバレッジは100%になりうる
- Step 8 の PBT は入力をランダム化するが、PBT のプロパティ自体が弱いと意味がない。ミューテーションテストはプロパティの厳しさを定量評価する
- AIが生成したテストが「形だけのアサーション」になっていないかを検証できる

### 何がうれしいのか

- Mutation Score 75% 以上を CI ゲートに置けば、テストの実質的な品質が担保される
- 生き残ったミュータントを見ると「ここのテストが弱い」が一目でわかる
- エージェントが「テストだけ通せば良い」抜け道を防ぐハーネス機構になる

## 完了条件

### F#

```bash
$ cd ../sales-management/apps/api-fsharp
$ dotnet stryker
…
Mutation score:        78.50 %
Killed:                157
Survived:              30
Timeout:               5
No coverage:           8
…
$ echo $?
0
```

### Kotlin

```bash
$ cd kotlin
$ gradle pitest
> Task :pitest

Generated 142 mutants
Killed 113 (79%) of 142 mutants
BUILD SUCCESSFUL
$ echo $?
0
```

---

## ミュータントの具体例

ツールは以下のような変異を自動生成する。

| 変異の種類 | 例 |
|---|---|
| Conditional Boundary | `x > 0` → `x >= 0` |
| Negate Conditional | `x > 0` → `!(x > 0)` |
| Math Operator | `a + b` → `a - b`, `a * b` → `a / b` |
| Force Boolean | `if cond then …` → `if true then …` |
| Return Value | `return result` → `return null` / `return 0` |
| Void Method Call | `repository.save(lot)` → 削除 |

---

## F#（Stryker.NET）

### 1. インストール

```bash
cd ../sales-management/apps/api-fsharp
dotnet new tool-manifest 2>/dev/null || true
dotnet tool install dotnet-stryker --version 4.4.1
```

### 2. 設定ファイル

`fsharp/stryker-config.json`:

```json
{
  "stryker-config": {
    "project": "src/SalesManagement/SalesManagement.fsproj",
    "test-projects": [
      "tests/SalesManagement.Tests/SalesManagement.Tests.fsproj"
    ],
    "thresholds": {
      "high": 80,
      "low": 60,
      "break": 60
    },
    "reporters": ["html", "json", "progress"],
    "output-path": "../ci-results/stryker",
    "mutate": [
      "src/SalesManagement/Domain/**/*.fs"
    ]
  }
}
```

`break: 60` は「Mutation Score が 60% を下回ったら CI を失敗させる」という意味。

### 3. 実行

```bash
cd ../sales-management/apps/api-fsharp
dotnet stryker
```

実行後、`ci-results/stryker/reports/mutation-report.html` をブラウザで開くと、生き残ったミュータントを行単位で確認できる。

### 4. 生き残ったミュータントへの対応例

例えば以下のミュータントが生き残ったとする：

```fsharp
// 元
if lot.Common.LotNumber.Year > 2020 then …
// ミュータント (Conditional Boundary)
if lot.Common.LotNumber.Year >= 2020 then …
```

PBT が「Year が 2020 より大きい場合と小さい場合の両方」をテストしていれば殺せる。生き残った場合、PBT に境界値を含むプロパティを追加する：

```fsharp
testProperty "Year=2020 では invalid と扱う" <|
    fun () ->
        let common = { ...; LotNumber = { Year = 2020; ... } }
        // 期待される挙動を検証
```

---

## Kotlin（PITest）

### 1. プラグイン追加

`build.gradle.kts`:

```kotlin
plugins {
    id("info.solidsoft.pitest") version "1.15.0"
}

pitest {
    targetClasses.set(listOf("salesmanagement.domain.*"))
    excludedClasses.set(listOf("salesmanagement.domain.*Types*"))
    mutators.set(listOf("DEFAULTS"))
    threads.set(4)
    outputFormats.set(setOf("HTML", "XML"))
    timestampedReports.set(false)
    reportDir.set(file("../ci-results/pitest"))
    mutationThreshold.set(75)
    coverageThreshold.set(80)
    junit5PluginVersion.set("1.2.1")
}
```

`mutationThreshold = 75` は「Mutation Score が 75% を下回ったら build を失敗させる」。

### 2. 実行

```bash
cd kotlin
gradle pitest
```

レポートは `ci-results/pitest/index.html`。

### 3. PIT のミュータント例

PIT のデフォルトミュテータには以下が含まれる：

- `CONDITIONALS_BOUNDARY` (`<` ⇄ `<=`, `>` ⇄ `>=`)
- `INCREMENTS` (`++` ⇄ `--`)
- `INVERT_NEGS` (`-x` → `x`)
- `MATH` (`+` ⇄ `-`, `*` ⇄ `/`, `%` ⇄ `*` 等)
- `NEGATE_CONDITIONALS` (`==` → `!=`)
- `RETURN_VALS` (`return x` → `return null`)
- `VOID_METHOD_CALLS` (副作用呼び出し削除)

---

## ci.sh への追加

```bash
echo "=== ミューテーションテスト ==="
# F#:
(cd ../sales-management/apps/api-fsharp && dotnet stryker)
# Kotlin:
cd kotlin && gradle pitest && cd ..
```

CI 環境ではミューテーションテストは時間がかかる（数分〜十数分）ため、main ブランチへのマージ前のみ実行する運用も多い。`pre-merge` ジョブとして分離するのも合理的。

---

## ci.sh の現時点の構成

```bash
#!/bin/bash
set -e

echo "=== ci-results/ 初期化 ==="
mkdir -p ci-results/sarif

echo "=== マイグレーション ==="
echo "=== ビルド ==="
echo "=== フォーマットチェック ==="
echo "=== リンター ==="
echo "=== テスト + カバレッジ ==="

echo "=== ミューテーションテスト ==="
(cd ../sales-management/apps/api-fsharp && dotnet stryker)
cd kotlin && gradle pitest && cd ..

echo "=== シークレット検出 (SARIF) ==="
gitleaks detect --source . \
  --report-format sarif --report-path ci-results/sarif/gitleaks.sarif --exit-code 1

echo "=== SCA (SARIF) ==="
trivy fs --scanners vuln --severity HIGH,CRITICAL \
  --format sarif --output ci-results/sarif/trivy.sarif .

echo "=== SAST (SonarQube) ==="
gradle sonar
bash scripts/sonar-to-sarif.sh sales-management-kotlin ci-results/sarif/sonar.sarif

echo "=== DAST (OWASP ZAP, SARIF) ==="
# (アプリ起動 → ZAP実行 → アプリ停止)

echo "=== SARIF マージ ==="
sarif merge ci-results/sarif/*.sarif \
  --output-file-path ci-results/merged.sarif --recurse false

echo "=== CI完了 ==="
```

---

## RALPHループにおける位置づけ

| 層 | 役割 |
|---|---|
| 検証強度 (Verification rigor) | テストの品質そのものを定量評価する |
| Skills のフィードバック源 | エージェントが書いた PBT が形骸化していないか検出 |

エージェントが「`assert true` を並べてカバレッジを稼ぐ」「条件式を雑に書いて PBT が境界値を見逃す」といった抜け道を取れないようにする品質ゲート。Step 8 の PBT を実質的に保証する。

---

## 次のステップ

Step 22が完了したら [Step 23: アーキテクチャ適合性検査](./step23.md) へ進む。
