# sample-opentelemetry

sample project for OpenTelemetry

OpenTelemetry の機能を 1 つずつ検証する。トピックごとに `<NN>-<topic>/` を切り、各ディレクトリの `README.md` に検証記録を書く。opentelemetry-demo を使う検証は fork [thaim-sandbox/opentelemetry-demo](https://github.com/thaim-sandbox/opentelemetry-demo) にディレクトリと同名のブランチを切って差分を置き、検証記録からそのブランチを参照する。自作のサンプルはディレクトリに直接置く。

## トピック

| ディレクトリ | 内容 |
|-------------|------|
| [01-opentelemetry-demo](01-opentelemetry-demo/) | 公式デモ（複数言語のマイクロサービス + 負荷生成器 + フィーチャーフラグ + Collector と観測バックエンド一式）を公開イメージで起動し、テレメトリの流れ・トレース・メトリクス・ログ・障害注入の 5 観点で観察する |
| [02-instrumentation-by-language](02-instrumentation-by-language/) | 公式デモの各サービスについて、言語ごとの SDK 初期化・自動計装・手動計装・メトリクス・ログ・リソース検出の実装をソースと docs から整理し、実際の出力と coverage 表に突き合わせる |

## 予定

公式デモの docs から抽出した検証項目を、環境を変えずに進められる順に並べる。

| トピック | 内容 | 構成 |
|---------|------|------|
| トレースの伝播機構 | Baggage（`synthetic_request` / `session.id`）、Span links、ブラウザ計装（WebTracerProvider / SessionIdProcessor / CORS）、Envoy tracing の非合成リクエスト限定、Currency（C++）の手動伝播 | Minimal + 実ブラウザ |
| メトリクス計装と可視化 | 手動メトリクス、ランタイムメトリクス、Exemplars、Prometheus bridge（ad）、Grafana ダッシュボード 10 個 | Minimal |
| Collector 設定 | Tail-based sampling（`service.criticality`）、Bring your own backend（`otelcol-config-extras.yml`） | Minimal |
| 障害注入 | `paymentFailure` 以外のフラグ 13 件がトレース・メトリクス・ログにどう現れるかを docs の説明と突き合わせる | Minimal |

環境変更が必要、または独立したテーマとして別途検証するもの:

| テーマ | 内容 | 構成 |
|-------|------|------|
| テストスイート | Telemetry tests（`make run-telemetry-tests-minimal`。pytest が Jaeger / Prometheus / OpenSearch に各サービスの期待シグナルを問い合わせる）、Cypress のフロントエンドテスト（`make run-frontend-tests`）の操作で発生するブラウザ計装スパンと Envoy トレースの観察 | Minimal |
| Kafka | checkout の producer 計装、fraud-detection の consumer と Span links、`kafkaQueueProblems`、Self-Observability の fraud-detection / kafka、`kafkametrics` receiver | Full |
| Kubernetes 上の OpenTelemetry | DaemonSet Collector とノードローカル送信、`k8s_attributes`、kubeletstats / k8s_cluster / hostmetrics、annotation discovery、preset ごとの RBAC、ブラウザ計装の `/otlp-http` 経路、Helm values での Collector 設定マージ | Kubernetes クラスタ（Helm chart） |
| Profiles シグナル | eBPF プロファイラ → Collector → Firepit、`filter/sanitize_profiles`、トレースとの相関 | Profiling |
| GenAI 計装 | agent / mcp / chatbot の LangGraph 計装、`gen_ai_normalizer` と GenAI セマンティック規約、`aiSlowResponse` / `aiRunawayAgent` | Agentic |

OpenTelemetry に関わらないため検証しないもの: Helm のインストール・アップグレード運用、port-forward / Ingress / Service type、`failedReadinessProbe`、OOMKilled の観察、compose 構成の切替そのもの、バックエンド（Jaeger / Prometheus / OpenSearch）自体の運用差。
