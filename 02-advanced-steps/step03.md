# Step 3: 認証・認可

## 目的

### これは何か

APIにJWT（JSON Web Token）ベースの認証と、ロールベースのアクセス制御（RBAC）を導入する。IDプロバイダとして Keycloak を docker-compose で起動し、ユーザー作成からトークン取得、APIでの検証までを一気通貫で実装する。

### なぜやるのか

- Step 7までのAPIは誰でもアクセスできる状態。業務システムでは「誰がこの操作を行ったか」を特定し、権限のない操作を防ぐ必要がある
- Spring Security が提供していた認証・認可の仕組みを、Giraffe / Ktor で実現する
- Keycloak は OSS のIDプロバイダで、本番では AWS Cognito や Auth0 に差し替え可能

### 何がうれしいのか

- 「認証トークンなしでAPIを叩くと401が返る」「権限のないAPIを叩くと403が返る」という、業務システムとして当たり前のセキュリティが実現する
- Keycloak の管理画面でユーザーやロールを視覚的に管理できる
- JWTのペイロードからユーザーIDやロールを取得できるため、「誰が操作したか」をログや監査証跡に記録できる（Step 8で活用）

## 事前準備: Keycloak を docker-compose に追加（自動初期化）

### 1. レルム設定ファイルを作成

レルム・クライアント・ロール・ユーザーを全て含むJSONファイルを用意する。`docker compose up` するだけで全て自動構築される。

```bash
mkdir -p keycloak
```

```json
// keycloak/sales-management-realm.json
{
  "realm": "sales-management",
  "enabled": true,
  "roles": {
    "realm": [
      { "name": "viewer", "description": "Read-only access" },
      { "name": "operator", "description": "Can perform state transitions" },
      { "name": "admin", "description": "Full access" }
    ]
  },
  "clients": [
    {
      "clientId": "sales-api",
      "enabled": true,
      "publicClient": true,
      "directAccessGrantsEnabled": true,
      "redirectUris": ["http://localhost:*"],
      "webOrigins": ["http://localhost:*"],
      "protocolMappers": [
        {
          "name": "audience-mapper",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-audience-mapper",
          "config": {
            "included.client.audience": "sales-api",
            "id.token.claim": "false",
            "access.token.claim": "true"
          }
        }
      ]
    }
  ],
  "users": [
    {
      "username": "test-viewer",
      "enabled": true,
      "credentials": [{ "type": "password", "value": "password", "temporary": false }],
      "realmRoles": ["viewer"]
    },
    {
      "username": "test-operator",
      "enabled": true,
      "credentials": [{ "type": "password", "value": "password", "temporary": false }],
      "realmRoles": ["operator"]
    },
    {
      "username": "test-admin",
      "enabled": true,
      "credentials": [{ "type": "password", "value": "password", "temporary": false }],
      "realmRoles": ["admin"]
    }
  ]
}
```

### 2. docker-compose.yml に追加

```yaml
  keycloak:
    image: quay.io/keycloak/keycloak:24.0
    command: start-dev --import-realm
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
    ports:
      - "8180:8080"
    volumes:
      - ./keycloak/sales-management-realm.json:/opt/keycloak/data/import/sales-management-realm.json:ro
```

ポイント: `--import-realm` と `volumes` のマウントにより、コンテナ起動時にレルム設定が自動インポートされる。手動でのブラウザ操作は不要。

### 3. 起動と確認

```bash
docker compose up -d keycloak
# 起動まで30秒ほど待つ

# レルムが作成されたことを確認
curl -s http://localhost:8180/realms/sales-management | jq .realm
# → "sales-management"

# （任意）ブラウザで http://localhost:8180 を開き、admin/admin でログインすると
# sales-management レルムにユーザーとロールが作成済みであることを確認できる
```

### 4. トークンを取得する

```bash
# operator ユーザーのトークンを取得
TOKEN=$(curl -s -X POST "http://localhost:8180/realms/sales-management/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password" \
  -d "client_id=sales-api" \
  -d "username=test-operator" \
  -d "password=password" | jq -r '.access_token')

echo $TOKEN
# → eyJhbGciOiJSUzI1NiIs... （長いJWT文字列）

# トークンの中身を確認（jwt.io に貼り付けるか、以下のコマンド）
echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | jq .
# → {"sub":"user-uuid","realm_access":{"roles":["operator"]},...}
```

この `TOKEN` を以降のAPI呼び出しで使う。

## 完了条件

### 認証の確認

1. トークンなしでAPIを叩くと `401 Unauthorized` が返ること:

```bash
curl -v http://localhost:8080/api/lots/2024-A-001
# → 401 Unauthorized
# レスポンスヘッダに WWW-Authenticate: Bearer が含まれる
```

2. 有効なトークンを付けてAPIを叩くと正常にレスポンスが返ること:

```bash
curl -H "Authorization: Bearer $TOKEN" http://localhost:8080/api/lots/2024-A-001
# → 200 OK
```

3. 期限切れトークンで叩くと `401 Unauthorized` が返ること（Keycloakのトークン有効期限はデフォルト5分。5分以上待ってから試す）

### 認可（RBAC）の確認

4. `operator` ロールのユーザーがロットの状態遷移APIを叩けること:

```bash
# operator トークンで製造完了を指示
curl -X POST -H "Authorization: Bearer $OPERATOR_TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:8080/api/lots/2024-A-001/complete-manufacturing \
  -d '{"date":"2026-04-22"}'
# → 200 OK
```

5. `viewer` ロールのユーザーが状態遷移APIを叩くと `403 Forbidden` が返ること:

```bash
# viewer トークンを取得
VIEWER_TOKEN=$(curl -s -X POST "http://localhost:8180/realms/sales-management/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password&client_id=sales-api&username=test-viewer&password=password" | jq -r '.access_token')

# viewer トークンで製造完了を指示 → 拒否される
curl -X POST -H "Authorization: Bearer $VIEWER_TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:8080/api/lots/2024-A-001/complete-manufacturing \
  -d '{"date":"2026-04-22"}'
# → 403 Forbidden
```

6. `viewer` ロールでもGET（参照）APIは叩けること:

```bash
curl -H "Authorization: Bearer $VIEWER_TOKEN" http://localhost:8080/api/lots/2024-A-001
# → 200 OK
```

### ヘルスチェックは認証不要であること

7. `/health` はトークンなしでもアクセスできること（k8sのprobeは認証トークンを持たない）:

```bash
curl http://localhost:8080/health
# → 200 OK（認証不要）
```

---

## 実装ガイド

### ロール設計

| ロール | 参照（GET） | 状態遷移（POST） | 管理操作 |
|---|---|---|---|
| `viewer` | ○ | × | × |
| `operator` | ○ | ○ | × |
| `admin` | ○ | ○ | ○ |

### Keycloak の JWT ペイロード構造

Keycloak が発行するJWTの `realm_access.roles` にロールが入る:

```json
{
  "sub": "a1b2c3d4-...",
  "preferred_username": "test-operator",
  "realm_access": {
    "roles": ["operator", "default-roles-sales-management"]
  },
  "exp": 1750000000
}
```

アプリ側では `realm_access.roles` からロールを取得する。

### F#

| 要素 | 実装方法 |
|---|---|
| JWT認証 | `AddAuthentication().AddJwtBearer()` — Authority に Keycloak のレルムURLを指定 |
| ロール認可 | `AddAuthorization` + Policy（`realm_access.roles` からロール取得） |
| Giraffe統合 | `requiresAuthentication` + `authorizeByPolicyName` |
| CORS | `AddCors` + `UseCors` |

```fsharp
// JWT設定のポイント
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(fun opts ->
        opts.Authority <- "http://localhost:8180/realms/sales-management"
        opts.Audience <- "sales-api"
        opts.RequireHttpsMetadata <- false  // 開発環境のみ
    ) |> ignore
```

### Kotlin

| 要素 | 実装方法 |
|---|---|
| JWT認証 | Ktor `ktor-server-auth-jwt` — Keycloak の JWKS エンドポイントで公開鍵を取得 |
| ロール認可 | `validate` ブロック内で `realm_access.roles` をチェック |
| CORS | Ktor `CORS` plugin |

```kotlin
// JWT設定のポイント
install(Authentication) {
    jwt("auth-jwt") {
        realm = "sales-management"
        verifier(
            jwkProvider,  // Keycloak の JWKS URL: http://localhost:8180/realms/sales-management/protocol/openid-connect/certs
            "http://localhost:8180/realms/sales-management"
        )
        validate { credential ->
            if (credential.payload.audience?.contains("sales-api") == true)
                JWTPrincipal(credential.payload) else null
        }
    }
}
```

### トークン取得を楽にするスクリプト（任意）

開発中に毎回curlでトークンを取得するのは面倒なので、ヘルパースクリプトを作っておくと便利:

```bash
#!/bin/bash
# scripts/get-token.sh
USER=${1:-test-operator}
PASS=${2:-password}

curl -s -X POST "http://localhost:8180/realms/sales-management/protocol/openid-connect/token" \
  -d "grant_type=password&client_id=sales-api&username=$USER&password=$PASS" | jq -r '.access_token'
```

```bash
# 使い方
TOKEN=$(./scripts/get-token.sh test-operator)
curl -H "Authorization: Bearer $TOKEN" http://localhost:8080/api/lots/2024-A-001
```

---

## 次のステップ

Step 3が完了したら [Step 4: ヘルスチェック + OpenAPI](./step04.md) へ進む。
