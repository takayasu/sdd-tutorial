# Step 18b: 認証 ON 化 + 開発者向け補助エンドポイント

> Section 3 の Step 18 と Step 19 の間に位置する。Step 1-18 は `Authentication.Enabled=false` 前提で進めてきたが、本ステップで「ON にしても回せる」状態にする。

## 目的

### これは何か

ここまで全ての API は `Authentication.Enabled=false` で開放されていた。本ステップで以下を導入し、**`Authentication.Enabled=true` に切り替えても tutorial の verify が緑のまま回る**状態にする:

| 要素 | 何のため |
|---|---|
| `appsettings.json` の `Authentication.Enabled` 切り替え | dev/staging/prod で auth を on/off 切り替え可能にする |
| `tools/DevTokenMint` CLI | `Authentication.SigningKey` (HS256) で署名した開発用 JWT を発行 (Keycloak を立てずに認証 ON で疎通するため) |
| `GET /auth/config` パブリックエンドポイント | フロント (将来導入する SPA) が起動時に「認証 ON か / IdP authority」を自動判定できるようにする |
| ci.sh verify セクションへの auth-on 検証追加 | 「auth ON にしてもエンドポイントは生きている」を継続検証 |

**やらないこと**: Keycloak / 本番 OIDC IdP の構築は本ステップ範囲外。あくまで HS256 共有秘密での dev/CI 用認証を扱う。

### なぜやるのか

- 認証 ON の動作確認手段がないと、本番デプロイ直前に「ローカルでは通っていたが本番では 401 連発」という事故を起こす
- フロント (将来) は `VITE_AUTH_BYPASS=true` のような環境変数で切り替えるが、サーバ実態と乖離するリスクがある。`/auth/config` をサーバから配信すれば**唯一の事実源**にできる
- integration test でも JWT を都度自前構築すると DRY 違反 + 共通化漏れ。CLI に集約すると「test で使った CLI ロジックを dev でも使う」ようにできる
- Phase 2 (Step 21+) で OIDC IdP に置き換えた時も、`/auth/config` の interface だけ保てばフロントの変更は不要

### 何がうれしいのか

- `dotnet run --project tools/DevTokenMint -- --role operator` で JWT がもらえ、`Authorization: Bearer <token>` で API を叩ける
- フロントは `VITE_AUTH_BYPASS` のような二重管理を捨て、起動時に `/auth/config` を 1 回叩くだけで判定できる
- ci.sh が `Authentication.Enabled=false`/`true` の両方で緑になり、「本番直前で 401」事故を構造的に防げる

## 完了条件

### (a) 動作要件

```bash
# 1. /auth/config が無認証で 200
$ curl -sf http://localhost:5000/auth/config | jq .
{
  "enabled": false,
  "audience": "sales-api"
}

# 2. Authentication.Enabled=false の時、保護エンドポイントは無認証で 200 (= Step 7-18 の verify が壊れない)
$ curl -sf http://localhost:5000/lots | jq -e '.items' >/dev/null

# 3. Authentication.Enabled=true に切り替えると未認証は 401 + problem+json
$ AUTH=true ./scripts/restart-with-auth.sh
$ curl -s -o /dev/null -w "%{http_code} %{content_type}\n" http://localhost:5000/lots
401 application/problem+json

# 4. /auth/config は ON でも無認証で叩ける (security: [] が effective)
$ curl -sf http://localhost:5000/auth/config | jq -e '.enabled==true and .audience'

# 5. DevTokenMint CLI で JWT を発行
$ TOKEN=$(dotnet run --project tools/DevTokenMint -- --role operator --user u1)
$ echo $TOKEN | head -c 4
eyJ

# 6. 発行した JWT で保護エンドポイントが 200
$ curl -sf -H "Authorization: Bearer $TOKEN" http://localhost:5000/lots | jq -e '.items' >/dev/null

# 7. ロール不足は 403 + problem+json
$ TOKEN_VIEWER=$(dotnet run --project tools/DevTokenMint -- --role viewer --user u2)
$ curl -s -o /dev/null -w "%{http_code} %{content_type}\n" \
    -X POST http://localhost:5000/lots \
    -H "Authorization: Bearer $TOKEN_VIEWER" \
    -H "Content-Type: application/json" \
    -d '{"lotNumber":{"year":2024,"location":"A","seq":99},...}'
403 application/problem+json

# 8. openapi.yaml に securitySchemes と /auth/config が記載されている
$ python3 -c "
import yaml
y = yaml.safe_load(open('openapi.yaml'))
assert 'bearerAuth' in y['components']['securitySchemes']
assert '/auth/config' in y['paths']
# /auth/config は security: [] (= 無認証で叩ける) であること
assert y['paths']['/auth/config']['get'].get('security') == []
print('OK')
"
OK
```

### (b) `ci.sh` verify セクションへの追記

verify セクションは **2 周回す**ように改修する: 1 周目は `Authentication.Enabled=false`、2 周目は `=true` で同じ curl 群を叩く。

```bash
echo "=== verify (auth=off) ==="
AUTH_ENABLED=false start_app
run_smoke_curls    # Step 1 / 7 / 14 / 15 / 18 で蓄積したもの
stop_app

echo "=== verify (auth=on) ==="
AUTH_ENABLED=true start_app
TOKEN=$(dotnet run --project tools/DevTokenMint -- --role operator --user ci)
AUTH_HEADER="Authorization: Bearer $TOKEN" run_smoke_curls
# 加えて auth 固有の検証
curl -sf http://localhost:5000/auth/config | jq -e '.enabled==true' >/dev/null && echo "PASS auth-config-on"
curl -s -o /dev/null -w "%{http_code}" http://localhost:5000/lots | grep -q 401 && echo "PASS protected-401-without-token"
stop_app
```

期待出力:

```
=== verify (auth=off) ===
PASS /health
... (既存 PASS 群) ...
PASS auth-config-off

=== verify (auth=on) ===
PASS /health
... (Bearer 付きで既存 PASS 群) ...
PASS auth-config-on
PASS protected-401-without-token
PASS protected-403-on-insufficient-role
=== CI完了 ===
```

---

## 実装方針

### `tools/DevTokenMint`

- 単一 .NET プロジェクト (`tools/DevTokenMint/DevTokenMint.fsproj`)
- 引数: `--role {viewer|operator|admin}` `--user <id>` `--ttl <seconds>` (default 3600)
- `appsettings.json` の `Authentication.SigningKey` / `Audience` / `Issuer` を読み込んで HS256 で署名
- 出力は JWT 文字列 1 行のみ (`echo $TOKEN | head -c 4` が `eyJ` で始まる)
- integration test (`tests/IntegrationTests/`) も同じ Mint ロジックを `internal` で参照する (DRY)

### `GET /auth/config`

- ルート定義時に `[<AllowAnonymous>]` 相当 (Giraffe なら `security: []` を `openapi.yaml` で明記、ハンドラを `requireAuth` で囲まない)
- 戻り値スキーマ:

```yaml
AuthConfigResponse:
  type: object
  required: [enabled, audience]
  properties:
    enabled:   { type: boolean }
    audience:  { type: string }
    authority: { type: string, format: uri, description: "OIDC discovery URL (Phase 2 で導入)" }
```

- 認証 OFF 時: `{ "enabled": false, "audience": "sales-api" }`
- 認証 ON 時: `{ "enabled": true, "audience": "sales-api", "authority": "https://idp.example/realms/x" }` (authority は `Authentication.Authority` 設定値)

### Authentication.Enabled 切り替え

`appsettings.json`:

```jsonc
{
  "Authentication": {
    "Enabled": false,
    "Audience": "sales-api",
    "Issuer": "dev-issuer",
    "SigningKey": "dev-secret-min-32-chars-xxxxxxxxxx",
    "Authority": null
  }
}
```

`Program.fs` で `Enabled=true` の時のみ `app.UseAuthentication() |> app.UseAuthorization() |> ignore` を有効化。`/auth/config` は両モードで生かす。

---

## 次のステップ

Step 18b が完了したら [Step 19: 品質ダッシュボード](./step19.md) へ進む。
