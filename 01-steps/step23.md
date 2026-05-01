# Step 23: アーキテクチャ適合性検査（ArchUnit）

## 目的

### これは何か

ArchUnit は、ソースコードの依存方向・レイヤ・命名規約・アノテーションがアーキテクチャのルール通りであることをテストとして実行可能にするライブラリ。例えば「Domain 層は Infrastructure 層を import してはならない」というルールをコードで書き、CI で強制する。

### なぜやるのか

- AGENTS.md の「ファイル配置規約」は人間向けの自然言語ルール。AI がコードを大量生成するとレビューを通り抜けてレイヤ違反が混入することがある（例: `Domain/Workflows.fs` で Npgsql を直接 import）
- Step 7 で確立した「Domain → API → Infrastructure」の依存方向はドメインモデルの根幹だが、Phase 1 ではコンパイラ的には強制されていなかった
- ArchUnit はルールをコードで表現し、テストとして CI で実行することで、文章ルールでは防げない退行を機械的に止める

### 何がうれしいのか

- AGENTS.md に書いた「Domain は Infrastructure に依存しない」が実行可能なテストになる
- 新メンバー（人間 or AI）が誤った方向の依存を追加するとビルドが落ちる
- レビューの認知負荷が下がる（人間は「ルールを思い出して照合する」必要がなくなる）

## 完了条件

### F#

```bash
$ cd ../sales-management/apps/api-fsharp
$ dotnet test --filter "Category=Architecture"
  Passed: 4 (
    DomainHasNoInfrastructureDependency,
    DomainHasNoApiDependency,
    InfrastructureMustNotBeReferencedFromApi,
    DomainTypesMustBeRecordsOrDU
  )
$ echo $?
0
```

### Kotlin

```bash
$ cd kotlin
$ gradle test --tests "*ArchitectureTest*"
> Task :test

salesmanagement.architecture.ArchitectureTest
  ✓ domain_does_not_depend_on_infrastructure
  ✓ repositories_live_in_infrastructure
  ✓ api_only_depends_on_domain_abstractions
  ✓ domain_classes_are_immutable_no_var

BUILD SUCCESSFUL
```

---

## F#（ArchUnitNET）

### F# 制約

ArchUnitNET の Fluent API は C# 構文に最適化されており、F# だと記述が読みづらくなる。本 step ではテストコード 1 ファイルだけ C# で書き、F# プロジェクトと同じソリューション内に配置する妥協を許容する。

### 1. テストプロジェクトに ArchUnitNET を追加

```bash
cd ../sales-management/apps/api-fsharp/tests/SalesManagement.Tests
dotnet add package ArchUnitNET.xUnit --version 0.10.6
```

### 2. アーキテクチャテスト

`fsharp/tests/SalesManagement.Tests/ArchitectureTests.cs`:

```csharp
using ArchUnitNET.Domain;
using ArchUnitNET.Loader;
using ArchUnitNET.xUnit;
using Xunit;
using static ArchUnitNET.Fluent.ArchRuleDefinition;

namespace SalesManagement.Tests
{
    public class ArchitectureTests
    {
        private static readonly Architecture Architecture = new ArchLoader()
            .LoadAssemblies(typeof(SalesManagement.Domain.Types.LotNumber).Assembly)
            .Build();

        [Fact, Trait("Category", "Architecture")]
        public void DomainHasNoInfrastructureDependency()
        {
            Classes()
                .That().ResideInNamespace("SalesManagement.Domain", true)
                .Should().NotDependOnAny(
                    Classes().That().ResideInNamespace("SalesManagement.Infrastructure", true))
                .Check(Architecture);
        }

        [Fact, Trait("Category", "Architecture")]
        public void DomainHasNoApiDependency()
        {
            Classes()
                .That().ResideInNamespace("SalesManagement.Domain", true)
                .Should().NotDependOnAny(
                    Classes().That().ResideInNamespace("SalesManagement.Api", true))
                .Check(Architecture);
        }

        [Fact, Trait("Category", "Architecture")]
        public void InfrastructureMustNotBeReferencedFromApi()
        {
            // API は Domain の抽象（interface）経由でのみ Infrastructure を呼ぶ
            Types()
                .That().ResideInNamespace("SalesManagement.Api", true)
                .Should().NotDependOnAny(
                    Types().That().ResideInNamespace("SalesManagement.Infrastructure", true)
                        .And().AreNot(typeof(SalesManagement.Infrastructure.IRepository<>)))
                .Check(Architecture);
        }

        [Fact, Trait("Category", "Architecture")]
        public void DomainTypesMustBeRecordsOrDU()
        {
            // F# の record / discriminated union は IL では sealed class として現れる
            // ミュータブルなプロパティを持つクラスを禁止
            Classes()
                .That().ResideInNamespace("SalesManagement.Domain.Types", true)
                .Should().BeSealed()
                .Check(Architecture);
        }
    }
}
```

### 3. 実行

```bash
cd ../sales-management/apps/api-fsharp
dotnet test --filter "Category=Architecture"
```

---

## Kotlin（ArchUnit）

### 1. 依存関係追加

`build.gradle.kts`:

```kotlin
dependencies {
    testImplementation("com.tngtech.archunit:archunit-junit5:1.3.0")
}
```

### 2. アーキテクチャテスト

`kotlin/src/test/kotlin/salesmanagement/architecture/ArchitectureTest.kt`:

```kotlin
package salesmanagement.architecture

import com.tngtech.archunit.core.importer.ImportOption
import com.tngtech.archunit.junit.AnalyzeClasses
import com.tngtech.archunit.junit.ArchTest
import com.tngtech.archunit.lang.ArchRule
import com.tngtech.archunit.lang.syntax.ArchRuleDefinition.classes
import com.tngtech.archunit.lang.syntax.ArchRuleDefinition.fields
import com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses

@AnalyzeClasses(
    packages = ["salesmanagement"],
    importOptions = [ImportOption.DoNotIncludeTests::class]
)
class ArchitectureTest {

    @ArchTest
    val domain_does_not_depend_on_infrastructure: ArchRule =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAPackage("..infrastructure..")

    @ArchTest
    val domain_does_not_depend_on_api: ArchRule =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAPackage("..api..")

    @ArchTest
    val repositories_live_in_infrastructure: ArchRule =
        classes()
            .that().haveSimpleNameEndingWith("Repository")
            .and().areNotInterfaces()
            .should().resideInAPackage("..infrastructure..")

    @ArchTest
    val api_only_depends_on_domain_abstractions: ArchRule =
        noClasses()
            .that().resideInAPackage("..api..")
            .should().dependOnClassesThat().resideInAPackage("..infrastructure..")

    @ArchTest
    val domain_classes_are_immutable_no_var: ArchRule =
        fields()
            .that().areDeclaredInClassesThat().resideInAPackage("..domain..")
            .should().beFinal()
}
```

`val` フィールドは Kotlin コンパイル時に `final` 修飾子が付くため、`var` を禁止することで domain がイミュータブルであることを強制できる。

### 3. 実行

```bash
cd kotlin
gradle test --tests "*ArchitectureTest*"
```

---

## ルールの追加方針

エージェントがコードを書くたびに違反したくなる典型例をルール化する：

| 違反パターン | 対応するルール |
|---|---|
| Domain で `Npgsql` を直接 open する | `domain_does_not_depend_on_infrastructure` |
| Repository 実装が Domain に紛れ込む | `repositories_live_in_infrastructure` |
| API ハンドラが直接 DB アクセスする | `api_only_depends_on_domain_abstractions` |
| Domain 型に `var` フィールドが入る | `domain_classes_are_immutable_no_var` |
| 状態遷移関数が異なる状態型を返す | （型システムが既に保証）|

---

## ci.sh への追加

```bash
echo "=== アーキテクチャ適合性 ==="
# F#:
(cd ../sales-management/apps/api-fsharp && dotnet test --filter "Category=Architecture")
# Kotlin:
cd kotlin && gradle test --tests "*ArchitectureTest*" && cd ..
```

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

echo "=== アーキテクチャ適合性 ==="
(cd ../sales-management/apps/api-fsharp && dotnet test --filter "Category=Architecture")
cd kotlin && gradle test --tests "*ArchitectureTest*" && cd ..

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
| Constraints (制約) | AGENTS.md の文章ルールを実行可能なテストに昇格 |
| Protocols | Domain / API / Infrastructure の境界をエージェント間で共有 |

エージェントが大量にコード追加するとき、人間レビューでは見逃される構造ドリフトを機械が止める。Step 28 の AGENTS.md 自動更新と相補的：

- **Step 28** は「過去の失敗を文章で蓄積」する soft な学習
- **Step 23** は「設計ルールを破壊不可能なコードで強制」する hard な制約

両方が揃うと、ハーネスは「学んでも、決して壊さない」状態になる。

---

## 次のステップ

Step 23が完了したら [Step 24: APIコントラクトテスト](./step24.md) へ進む。
