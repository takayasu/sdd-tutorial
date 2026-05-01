# Step 9: レート制限 + キャッシュ + グレースフルシャットダウン

## 目的

### これは何か

APIへの過剰なリクエストを制限するレート制限、頻繁に参照されるデータのキャッシュ、そしてアプリ停止時に処理中のリクエストを安全に完了させるグレースフルシャットダウンを導入する。本番運用に必要な最後のピースを揃える。

### なぜやるのか

- レート制限がないと、バグや悪意のあるクライアントが大量のリクエストを送り、システム全体が応答不能になる
- キャッシュがないと、同じデータを何度もDBから読み取り、不要な負荷がかかる。ロットの参照APIは更新より参照の方が圧倒的に多い
- グレースフルシャットダウンがないと、デプロイ時に処理中のリクエストが途中で切断される。k8sのローリングアップデートで「502 Bad Gateway」が発生する原因になる

### 何がうれしいのか

- レート制限により、1クライアントが毎秒100リクエスト送っても、他のクライアントへの影響を防げる。`429 Too Many Requests` が返り、システムは安定したまま
- キャッシュにより、ロット参照APIのレスポンスタイムが大幅に改善する。DBクエリが不要になるため、10ms以下で応答できる
- グレースフルシャットダウンにより、デプロイ時にユーザーがエラーを見ることがなくなる。処理中のリクエストが完了してからアプリが停止する
- これら3つが揃うと、「本番運用に耐えるAPIサーバー」が完成する。Step 0から始めたPoCが、業務システムとしての品質に到達する

## 完了条件

### レート制限の確認

1. 短時間に大量のリクエストを送ると、制限を超えた分が `429 Too Many Requests` で返ること:

```bash
# 連続で20回リクエストを送る（制限が10回/分の場合）
for i in $(seq 1 20); do
  echo "Request $i: $(curl -s -o /dev/null -w '%{http_code}' http://localhost:8080/api/lots/2024-A-001)"
done

# 出力例:
# Request 1: 200
# ...
# Request 10: 200
# Request 11: 429
# ...
# Request 20: 429
```

2. `429` レスポンスに `Retry-After` ヘッダが含まれていること（クライアントがいつリトライすべきかを知るため）

### キャッシュの確認

3. 同じロットを2回連続で取得し、2回目の方が速いこと:

```bash
# 1回目（DBアクセスあり）
time curl -s http://localhost:8080/api/lots/2024-A-001 > /dev/null
# real 0m0.050s（例）

# 2回目（キャッシュヒット）
time curl -s http://localhost:8080/api/lots/2024-A-001 > /dev/null
# real 0m0.005s（例、大幅に速い）
```

4. ロットの状態を変更した後、キャッシュが無効化され、最新のデータが返ること:

```bash
# 製造完了を指示
curl -X POST http://localhost:8080/api/lots/2024-A-001/complete-manufacturing -d '{"date":"2026-04-22"}'

# 取得 → 最新の状態（manufactured）が返ること（古いキャッシュが返らない）
curl http://localhost:8080/api/lots/2024-A-001
# → {"status":"manufactured", ...}
```

5. ログでキャッシュヒット/ミスを確認できること:

```
# {"level":"Debug","message":"Cache HIT","key":"lot:2024-A-001"}
# {"level":"Debug","message":"Cache MISS","key":"lot:2024-A-002"}
```

### グレースフルシャットダウンの確認

6. リクエスト処理中にアプリを停止しても、そのリクエストが正常に完了すること:

```bash
# ターミナル1: 遅いリクエストを送信（処理に数秒かかるエンドポイントを用意）
curl http://localhost:8080/api/slow-operation &

# ターミナル2: アプリを停止
kill -SIGTERM <pid>
# または: docker compose stop app

# ターミナル1: レスポンスが正常に返ること（接続が切断されない）
# → 200 OK
```

7. アプリ停止後、新しいリクエストは受け付けないこと

### 確認のコツ

- レート制限のテストは `for` ループで連続リクエストを送るのが簡単。`ab`（Apache Bench）や `hey` を使うとより正確に計測できる
- キャッシュの効果を確認するには、ログでDBクエリの実行有無を見る。キャッシュヒット時はDBクエリのログが出ないはず
- グレースフルシャットダウンのテストは、意図的に遅いエンドポイント（`Thread.Sleep` / `delay`）を作ると確認しやすい
- これら全てが動作したら、Step 0から積み上げてきた全機能が揃ったことになる。`ci.sh` を実行して全てのチェックが通ることを確認しよう

---

## 実装ガイド

### F#

| 要素 | 実装方法 |
|---|---|
| レート制限 | ASP.NET Core `AddRateLimiter`（.NET 7+ 標準） |
| キャッシュ | `IMemoryCache`（インメモリ）or `IDistributedCache`（Redis） |
| キャッシュ無効化 | 状態変更時に `cache.Remove(key)` |
| グレースフルシャットダウン | `HostOptions.ShutdownTimeout` + `IHostApplicationLifetime` |

主な作業:
1. `AddRateLimiter` で Fixed Window（例: 100リクエスト/分）を設定
2. `IMemoryCache` でロット参照結果をキャッシュ（TTL: 5分）
3. 状態変更APIでキャッシュを無効化（Step 7 の Outbox イベントハンドラ内で `cache.Remove` するとドメインロジックとの分離を保てる）
4. `ShutdownTimeout` を30秒に設定
5. `ApplicationStopping` イベントでDBコネクション等のクリーンアップを登録

### Kotlin

| 要素 | 実装方法 |
|---|---|
| レート制限 | Ktor `RateLimit` plugin or Bucket4j |
| キャッシュ | Caffeine（インメモリ、高性能） |
| キャッシュ無効化 | 状態変更時に `cache.invalidate(key)` |
| グレースフルシャットダウン | Ktor `ApplicationEngine.stop(gracePeriod, timeout)` |

主な作業:
1. Ktor `RateLimit` plugin を設定（例: 100リクエスト/分）
2. Caffeine キャッシュを設定（`maximumSize`, `expireAfterWrite`）
3. 状態変更APIでキャッシュを無効化（Step 7 の Outbox イベントハンドラ内で `cache.invalidate` するとドメインロジックとの分離を保てる）
4. `environment.monitor.subscribe(ApplicationStopping)` でクリーンアップを登録

---

## 🎉 おめでとうございます！

Step 9が完了すると、以下の全機能が揃った業務システムの基盤が完成します:

| カテゴリ | 実装済み機能 | 出典 |
|---|---|---|
| ドメイン | 型安全な状態遷移、PBT、関数型ドメインモデリング | 01-steps/ Step 6-18 |
| CI/CD | フォーマッター、リンター、カバレッジ、SAST、SCA、DAST | 01-steps/ Step 4-11, 19-20 |
| 設定・ログ | 環境別設定、構造化JSON ログ、リクエストID | 02-advanced-steps/ Step 1 |
| API品質 | Problem Details、Smart Constructor バリデーション | 02-advanced-steps/ Step 2 |
| セキュリティ | JWT認証、RBAC、CORS | 02-advanced-steps/ Step 3 |
| 運用 | ヘルスチェック、OpenAPI/Swagger | 02-advanced-steps/ Step 4 |
| テスト | PBT + 統合テスト + TestContainers | 01-steps/ Step 8 + 02-advanced-steps/ Step 5 |
| 外部連携 | HTTPクライアント、リトライ、サーキットブレーカー | 02-advanced-steps/ Step 6 |
| イベント駆動 | ドメインイベント、Outbox パターン | 02-advanced-steps/ Step 7 |
| 監査・追跡 | 監査ログ、分散トレーシング | 02-advanced-steps/ Step 8 |
| 本番運用 | レート制限、キャッシュ、グレースフルシャットダウン | 02-advanced-steps/ Step 9 |

Spring Boot が提供していた業務システム向け機能を、F# / Kotlin の軽量スタックで全てカバーしました。ビルド速度の改善と型安全性の向上を維持しながら、業務システムとしての品質を確保できています。

次のステップとして:
- [Step 10: CSV/ファイルエクスポート](./step10.md) でデータエクスポート機能を実装する
- `ci.sh` を更新し、新しいテスト（統合テスト）をCIに組み込む
- 本番デプロイに向けて、Dockerfile と k8s マニフェストを整備する
- [02-batch-steps/](../02-batch-steps/) を参考に、バッチ処理基盤を構築する
