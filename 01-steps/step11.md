# Step 11: SAST（Kotlinのみ）

## 目的

### これは何か

SAST（Static Application Security Testing）をCIに組み込む。SASTとは、コードを実行せずに解析し、セキュリティ上の脆弱性（SQLインジェクション、クロスサイトスクリプティング等）を検出するツール。ここではSonarQubeを使う。

### なぜやるのか

- リンター（Step 5）が「コードの品質」を見るのに対し、SASTは「セキュリティの穴」を見る
- 例：ユーザー入力をそのままSQLに埋め込んでいる箇所を検出する（SQLインジェクション）
- F#には対応するSASTツールがないため、Kotlinのみ。F#は型設計で代替する

### 何がうれしいのか

- セキュリティの専門知識がなくても、ツールが自動的に危険なコードを指摘してくれる
- AIが生成したコードにセキュリティ上の問題があっても、CIで検出できる
- SonarQubeのダッシュボードで脆弱性の一覧と修正方法が確認できる

## 完了条件

```bash
# SonarQubeが起動していることを確認
$ curl -s http://localhost:9000/api/system/status
{"id":"...","version":"10.x","status":"UP"}

# Sonar解析を実行
$ cd kotlin
$ gradle sonar
> Task :sonar
BUILD SUCCESSFUL in Xs

# 品質ゲートがパスしていることを確認
$ curl -s "http://localhost:9000/api/qualitygates/project_status?projectKey=sales-management-kotlin" | python3 -m json.tool
{
    "projectStatus": {
        "status": "OK",
        ...
    }
}

# ブラウザで http://localhost:9000 にアクセスすると
# プロジェクトのダッシュボードが表示される
```

---

## Kotlin（SonarQube）

### 1. docker-compose.yml にSonarQubeを追加

```yaml
# kotlin/docker-compose.yml に追加
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: sales_management
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  sonarqube:
    image: sonarqube:community
    depends_on:
      - sonarqube-db
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://sonarqube-db:5432/sonarqube
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: sonar
    ports:
      - "9000:9000"
    volumes:
      - sonarqube_data:/opt/sonarqube/data

  sonarqube-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonar
      POSTGRES_DB: sonarqube
    volumes:
      - sonarqube_pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
  sonarqube_data:
  sonarqube_pgdata:
```

### 2. SonarQube初期設定

```bash
docker compose up -d sonarqube sonarqube-db

# 起動を待つ（初回は1〜2分かかる）
echo "SonarQubeの起動を待機中..."
until curl -s http://localhost:9000/api/system/status | grep -q '"status":"UP"'; do
    sleep 5
done

# 初期パスワード変更（admin/admin → admin/任意のパスワード）
# ブラウザで http://localhost:9000 にアクセスして設定
```

### 3. Gradle設定追加（build.gradle.kts）

```kotlin
plugins {
    // 既存に追加
    id("org.sonarqube") version "5.0.0.4638"
}

sonar {
    properties {
        property("sonar.projectKey", "sales-management-kotlin")
        property("sonar.projectName", "Sales Management (Kotlin)")
        property("sonar.host.url", "http://localhost:9000")
        property("sonar.token", System.getenv("SONAR_TOKEN") ?: "")
        property("sonar.coverage.jacoco.xmlReportPaths", "build/reports/jacoco/test/jacocoTestReport.xml")
    }
}
```

### 4. トークン生成・実行

```bash
# SonarQubeのUIからトークンを生成し、環境変数に設定
export SONAR_TOKEN="squ_xxxxx"

# 解析実行
gradle sonar
```

### 5. ci.sh への追加

```bash
echo "=== SAST (SonarQube) ==="
gradle sonar
# 品質ゲートの確認
curl -s "http://localhost:9000/api/qualitygates/project_status?projectKey=sales-management-kotlin" \
  | grep -q '"status":"OK"' || (echo "品質ゲート失敗" && exit 1)
```

---

## F# の代替方針

F#にはSASTツールがないため、以下で代替する：

- 型設計で不正な入力を防ぐ（生文字列をSQL/HTMLに直接渡せない設計）
- Donald のパラメータ化クエリで SQLインジェクションを防止
- Step 20のDAST（OWASP ZAP）で実行時の脆弱性を検出

---

## ci.sh の現時点の構成（Kotlin）

```bash
#!/bin/bash
set -e

echo "=== マイグレーション ==="
gradle flywayMigrate

echo "=== ビルド ==="
gradle build

echo "=== フォーマットチェック ==="
gradle ktfmtCheck

echo "=== リンター ==="
gradle detekt

echo "=== テスト + カバレッジ ==="
gradle test jacocoTestReport

echo "=== SAST (SonarQube) ==="
gradle sonar

echo "=== シークレット検出 ==="
gitleaks detect --source . --exit-code 1

echo "=== パッケージ脆弱性スキャン ==="
trivy fs --scanners vuln --severity HIGH,CRITICAL .
```

---

## 次のステップ

Step 11が完了したら [Step 12: 直接販売案件の型定義](./step12.md) へ進む。
Phase 1のCI構築はこれで完成。以降はドメインモデルの拡張に集中する。
