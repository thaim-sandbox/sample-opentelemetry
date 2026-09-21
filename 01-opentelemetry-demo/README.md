# opentelemetry-demo を公開イメージで起動して観察する

公式デモ [opentelemetry-demo](https://github.com/open-telemetry/opentelemetry-demo) は、複数の言語（Go / Java / .NET / Rust / TypeScript / Python など）で書かれ gRPC / HTTP で通信するマイクロサービス群（Web ストア）に OpenTelemetry の計装（自動計装・計装ライブラリ・手動計装）を施したもので、Locust の負荷生成器、flagd のフィーチャーフラグによる障害シナリオ、Collector と観測バックエンド（Jaeger / Prometheus / OpenSearch / Grafana）までを Docker Compose / Kubernetes で一式起動できる。本トピックではこれを起動し、計装 → OTLP → Collector → バックエンドという全体像を、テレメトリの流れ・トレース・メトリクス・ログ・障害注入の 5 観点で観察する。ソースはビルドせず、リリースタグの公開イメージで起動する。

## 構成

- opentelemetry-demo 3.1.0（2026-09-18 リリース）を使う。fork のブランチ [`01-opentelemetry-demo`](https://github.com/thaim-sandbox/opentelemetry-demo/tree/01-opentelemetry-demo) とタグ 3.1.0 の差分は `.env.override` の `DEMO_VERSION=3.1.0` に限る（`.env` の `DEMO_VERSION=latest` は main から継続 push されるイメージを指すため、設定ファイルとイメージをタグで揃える）
- Minimal 構成（`compose.yaml` + `compose.observability.yaml` + `compose.extras.yaml`）で起動する。Full 構成（`compose.full.yaml` を追加）との差は Kafka / accounting / fraud-detection の有無と、それに伴う checkout（Kafka 接続設定）と otel-collector（設定ファイル）の変更で、観測バックエンドと OpAMP サーバ（OpAMP は Collector をリモート管理するプロトコル）は Minimal にも含まれる
- 実行環境は Linux、4 CPU / 7.9 GB RAM、Docker 29.1.1 / Compose v2.40.3。イメージ 25 個 8.3 GB、コンテナ合計メモリ約 4.1 GB（opensearch と load-generator がそれぞれ 1 GB 超）で 30 時間連続稼働し、再起動・OOM は発生しなかった

## 手順

```bash
git clone -b 01-opentelemetry-demo https://github.com/thaim-sandbox/opentelemetry-demo.git
cd opentelemetry-demo
COMPOSE="docker compose --env-file .env --env-file .env.override -f compose.yaml -f compose.observability.yaml -f compose.extras.yaml"
$COMPOSE pull
$COMPOSE up --no-build --force-recreate --remove-orphans --detach
```

停止:

```bash
$COMPOSE down --remove-orphans --volumes
```

- compose の各サービスは `build` と `image` の両方を持つ。pull 済みなら build は走らないが、ソースからのビルドを避けるため `--no-build` で明示する
- `down` も同じ `-f` を付けて実行する。`compose.yaml` だけだと `compose.observability.yaml` 定義のバックエンドが残る
- otel-collector はホストの `/`（読み取り専用）と `/var/run/docker.sock` を bind mount する（`host_metrics` / `docker_stats` receiver 用）

起動後の URL（Prometheus 以外は Envoy 経由の 8080 番）:

| URL | 内容 |
|-----|------|
| http://localhost:8080/ | Web ストア |
| http://localhost:8080/jaeger/ui/ | Jaeger |
| http://localhost:8080/grafana/ | Grafana（ダッシュボード 10 個をプロビジョニング済み） |
| http://localhost:8080/loadgen/ | Locust 負荷生成器 |
| http://localhost:8080/feature | flagd フィーチャーフラグ UI |
| http://localhost:8080/opamp/ | OpAMP サーバ（Collector の接続状態・有効設定） |
| http://localhost:9090/ | Prometheus |

## 観察結果

### テレメトリの流れ

- 各サービスは `OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317`（gRPC）で Collector（contrib 0.159.0）に送る。この負荷（後述の 30 時間累計で約 107 スパン/秒・36 ログ/秒）での Collector のメモリは約 140 MB で推移した
- Collector の設定は 4 層を `--config` で重ねる（基底 → full → observability → extras）。基底 `otelcol-config.yml` は receivers（otlp のほか docker_stats / host_metrics / nginx / redis / postgresql / prometheus/ad / http_check）、processors（resource_detection、memory_limiter、transform によるスパン名のセマンティック規約への正規化・PII マスク・ログ属性名の正規化、redaction、gen_ai_normalizer、filter によるプロファイルの除外）、exporter `debug`、connector `span_metrics` を定義する。`otelcol-config-observability.yml` は `otlp_grpc/jaeger`、`otlp_http/prometheus`、`opensearch` の各 exporter と `opamp` extension を追加し、pipelines の exporters を置き換える。`otelcol-config-extras.yml` は利用者の追加用で常に最後に読み込む
- 2 層目の `otelcol-config-full.yml` は Full 構成用に `kafkametrics` receiver を追加し、metrics パイプラインの receivers 配列を置き換える（Collector は設定ファイルの配列を結合せず置換するため）。`compose.observability.yaml` は順序を保つため Minimal でもこのファイルを読み込むので、`kafkametrics` は Kafka 不在で scrape に失敗し続け、置換後の配列に `prometheus/ad` が無いため ad の Prometheus 形式メトリクスは取り込まれない
- Prometheus に scrape target は無い。メトリクスは全て Collector から OTLP（`/api/v1/otlp`）で push され、Collector 自身と Jaeger の内部テレメトリも同じ経路で届く。Jaeger v2 は Collector をベースにしているため `otelcol_*` メトリクスを両方が出し、`service_name` で分けないと二重計上になる
- Collector は OpAMP extension で opamp-server に接続し、health と有効設定を報告する

### トレース

Jaeger には 18 サービスが現れる。load-generator の `user_checkout_single`（注文 1 件）のトレースは 72 スパン / 12 サービスで、Python → Envoy → TypeScript → Go → .NET / Go / C++ / Rust / PHP / JavaScript / Ruby / flagd と言語をまたいでコンテキストが伝播している。抜粋:

```
load-generator   internal user_checkout_single                          304.5ms
  load-generator   client   POST                                        244.5ms
    frontend-proxy   server   POST                                      240.1ms
      frontend         server   POST /api/checkout                      237.9ms
        frontend         client   oteldemo.CheckoutService/PlaceOrder   193.3ms
          checkout         server   oteldemo.CheckoutService/PlaceOrder 169.7ms
            checkout         internal prepareOrderItemsAndShippingQuoteFromCart
              checkout         client   oteldemo.CartService/GetCart
                cart             server   POST /oteldemo.CartService/GetCart
                  cart             client   valkey-cart:6379
              checkout         client   oteldemo.ProductCatalogService/GetProduct
                product-catalog  server   oteldemo.ProductCatalogService/GetProduct
                  product-catalog  client   astronomy-db
              checkout         client   oteldemo.CurrencyService/Convert
                currency         server   oteldemo.CurrencyService/Convert
              checkout         client   POST
                shipping         server   POST /get-quote
                  shipping         client   POST
                    quote            server   POST /getquote
            checkout         client   oteldemo.PaymentService/Charge
              payment          server   oteldemo.PaymentService/Charge
                payment          internal charge
            checkout         client   POST
              shipping         server   POST /ship-order
            checkout         client   oteldemo.CartService/EmptyCart
              cart             server   POST /oteldemo.CartService/EmptyCart
                cart             client   flagd.evaluation.v2.Service/ResolveFloat
                  flagd            server   flagd.evaluation.v2.Service/ResolveFloat
            checkout         client   POST
              email            server   POST /send_order_confirmation
                email            internal send_email
```

- SpanKind: サービス境界は client / server の対で表れ、サービス内の処理は internal になる。gRPC は client / server で同じスパン名（`oteldemo.CheckoutService/PlaceOrder`）になる
- 属性: セマンティック規約の属性（`rpc.method`、`http.request.method`、`http.route`、`server.address` など）は計装ライブラリが付与し、業務属性は手動計装が `demo.order.id`、`demo.payment.card_type`、`demo.shipping.cost.total` のように付与する。docs の manual-span-attributes ページは `app.*` 接頭辞だが、3.1.0 の実装は `demo.*` を使う
- 計装スコープ: `otel.scope.name` に計装ライブラリの名前とバージョンが入る（Go: `go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc` 0.71.0、Rust: `opentelemetry-instrumentation-actix-web` 0.24.0、Node.js: `@opentelemetry/instrumentation-pino` 0.68.0）。手動計装のスコープは `payment` のようなアプリ名になる
- リソース属性: `service.name` / `service.namespace` / `service.version` / `telemetry.sdk.*` / `process.*` は SDK が付与する。`host.*` / `os.*` は Collector の resource_detection processor（`detectors: [env, docker, system]`）が付与するため、全サービスに Collector コンテナから見た同じ値が付く（`os.description` に Collector コンテナのホスト名が入る）。`service.version` は `.env` の `IMAGE_VERSION=3.0.0` 由来で、イメージタグ 3.1.0 と一致しない
- スパンイベント: checkout の PlaceOrder に `prepared` / `charged` / `shipped` と `feature_flag.evaluation`、shipping に `shipping.quote.received` が記録される
- 負荷生成器由来のリクエストには下流のスパンまで `user_agent.synthetic.type=test` が付く

### メトリクス

- `span_metrics` connector がトレースから `traces_span_metrics_calls_total{service_name, span_name, status_code}` と duration ヒストグラムを生成する。サービス側のメトリクス計装なしに RED（Rate / Errors / Duration）メトリクスが得られ、Grafana の Spanmetrics Demo Dashboard がこれを使う
- Collector 内部メトリクス（30 時間累計）: スパンは受信 1,152 万 / Jaeger へ送信 1,152 万（`refused` 0、`send_failed` 0）だった。ログは受信 385 万 / OpenSearch へ送信 310 万 / `send_failed` 76 万だった（原因は次節）

### ログ

- OpenSearch の `otel-logs-<日付>` インデックスに約 130K レコード/時、約 1.2 GB/日で入る。件数は frontend-proxy（Envoy アクセスログ）が 4 割超で、product-catalog、Collector 自身のログが続く
- ログレコードに `traceId` / `spanId` が付き、トレースと相関できる。checkout（Go）、cart（.NET）、shipping（Rust）、frontend / payment（Node.js、pino 計装）のいずれも大半のレコードに付いている
- マッピング衝突でログが 22 時間分欠落した。起動直後に cart（.NET）の起動ログ `Overriding HTTP_PORTS '{http}' ...` が `attributes.http = "8080"` を text としてマッピングしたため、その後に届く frontend の `http.request.method` を持つログ（`attributes.http` が object になる）を OpenSearch が全て拒否した。初日のインデックスに frontend の HTTP ログは 0 件、翌日のインデックス（00:00 UTC 作成）では frontend のログが先に届いて object でマッピングされ、以後は正常に入っている。Collector の `transform/sanitize_logs` は `otelcol.signal` の同種の衝突だけを回避している
- `send_failed` 76 万件（受信の約 25%）に対し、インデックスの件数から推定した実際の欠落は約 10% だった。bulk リクエスト内の一部拒否がリクエスト単位で失敗に数えられているとみられる

### 障害注入

`src/flagd/demo.flagd.json` の `paymentFailure.defaultVariant` を `off` → `50%` に書き換える（flagd がファイル変更を検知して即時反映する。http://localhost:8080/feature からも変更できる）。

- 約 2.5 分後から checkout の PlaceOrder にエラーが出始め、span metrics で見た直近 5 分は成功 8.8 件 / エラー 2.5 件だった
- エラートレース（53 スパン）では payment の `charge`（internal）に exception イベント `Payment request failed. Invalid token. demo.user_context.loyalty_level=gold` が記録され、payment のサーバースパン → checkout のクライアントスパン（gRPC UNKNOWN）→ checkout server（INTERNAL）→ frontend → frontend-proxy → load-generator まで status=ERROR で伝播する
- Jaeger の `error=true` 検索は payment ↔ flagd の EventStream（flagd がフラグ変更を通知する gRPC ストリーム。600 秒でタイムアウトする長時間接続）も常時ヒットするため、operation で絞る

## ドキュメントとの対応

docs は https://opentelemetry.io/docs/demo/ 配下を 2026-09-21 に取得したもの。

### 実装との差分

3.1.0 時点で docs と実装が異なる点:

- 手動計装のスパン属性の接頭辞は docs が `app.*`、実装は `demo.*`
- log-coverage は frontend をログ未実装としているが、pino 計装でログを出している
- feature-flags の `loadGeneratorTraffic` / `loadGeneratorVUs` は無く、`aiSlowResponse` / `aiRunawayAgent` / `productCatalogLockContention` / `loadGeneratorFloodHomepage` / `emitRawPii` が追加されている

### 機能の検証状況

docs が紹介する機能と、このトピックでの検証状況。未検証のものは検証先のトピック（ルート README の「予定」）を示す。

| docs ページ | 機能 | 状態 | 検証先 |
|------------|------|------|--------|
| architecture, telemetry-features | 計装 → OTLP → Collector → Jaeger / Prometheus / OpenSearch / Grafana の経路、OpAMP、`span_metrics` connector | 済 | |
| collector-data-flow-dashboard | Grafana の Collector Data Flow ダッシュボード | 未 | メトリクス計装と可視化 |
| self-observability-dashboard | SDK 内部メトリクス `otel.sdk.*`（ad / fraud-detection / kafka） | 未 | メトリクス計装と可視化（ad）、Kafka（fraud-detection / kafka） |
| docker-deployment | Bring your own backend | 未 | Collector 設定 |
| docker-deployment | Agentic / Profiling 構成 | 未 | GenAI 計装、Profiles シグナル |
| trace-coverage, services/* | 言語横断のコンテキスト伝播、SpanKind、セマンティック規約の属性、リソース検出 | 済 | |
| trace-coverage, services/* | 言語ごとの SDK 初期化・自動計装・手動計装の実装 | 未 | 02-instrumentation-by-language |
| trace-coverage, services/* | Baggage、Span links、ブラウザ計装、Envoy tracing の非合成リクエスト限定、Currency の手動伝播 | 未（Envoy は frontend-proxy スパンの存在のみ確認） | トレースの伝播機構（fraud-detection の Span links は Kafka） |
| manual-span-attributes | 手動スパン属性、スパンイベント、例外記録と status の伝播 | 済 | |
| metric-coverage | Collector 内部メトリクス | 済 | |
| metric-coverage, services/* | 手動メトリクス、ランタイムメトリクス、Exemplars、Prometheus bridge、Grafana ダッシュボード 10 個 | 未（Spanmetrics Demo Dashboard のみ確認） | メトリクス計装と可視化 |
| log-coverage | OTLP ログと trace / span ID の相関 | 済（checkout / cart / shipping / frontend / payment） | 全サービスの突き合わせは 02-instrumentation-by-language |
| feature-flags | `paymentFailure` | 済 | |
| feature-flags | 残りのフラグ | 未 | 障害注入（13 件）、Kafka（`kafkaQueueProblems`）、GenAI 計装（`aiSlowResponse` / `aiRunawayAgent`）。`failedReadinessProbe` は検証しない |
| sample-configurations | Tail-based sampling（`service.criticality`） | 未 | Collector 設定 |
| tests | Telemetry tests、Cypress のフロントエンドテスト | 未 | テストスイート |
| kubernetes-deployment | Helm chart での Collector 構成（DaemonSet、`k8s_attributes` など）、ブラウザ計装の `/otlp-http` 経路 | 未 | Kubernetes 上の OpenTelemetry |
