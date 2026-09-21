# 言語別の計装実装を読む

公式デモの各サービスは言語ごとに異なる方法で OpenTelemetry を組み込んでいる（Java agent のようなゼロコード計装、`opentelemetry-instrument` や `node --require` のような起動時ラッパー、コード内での SDK 初期化）。docs の [Services](https://opentelemetry.io/docs/demo/services/) 配下の各ページと 3.1.0 のソースを読み、言語ごとに SDK 初期化・自動計装・手動計装・メトリクス・ログ・リソース検出の実装を同じ観点で整理し、実際に出力されるテレメトリと突き合わせる。あわせて docs の [trace](https://opentelemetry.io/docs/demo/telemetry-features/trace-coverage/) / [metric](https://opentelemetry.io/docs/demo/telemetry-features/metric-coverage/) / [log](https://opentelemetry.io/docs/demo/telemetry-features/log-coverage/) coverage 表と実測の差分を記録する。

## 構成

- opentelemetry-demo 3.1.0。環境と起動手順は [01-opentelemetry-demo](../01-opentelemetry-demo/) と同じ（Minimal 構成）。ソースは変更しないため fork のブランチは切らず [`01-opentelemetry-demo`](https://github.com/thaim-sandbox/opentelemetry-demo/tree/01-opentelemetry-demo) ブランチを使う。差分が必要になった時点で同名ブランチを切る
- ソースは fork の `src/<service>/`、docs は `https://opentelemetry.io/docs/demo/services/<service>/`
- Python の agent / mcp / chatbot は GenAI 計装として別途検証、react-native-app は compose で起動しないため対象外

## 対象

| 言語 | サービス | 構成 | 状態 |
|------|---------|------|------|
| Go | [checkout](https://opentelemetry.io/docs/demo/services/checkout/), [product-catalog](https://opentelemetry.io/docs/demo/services/product-catalog/) | Minimal | 未 |
| Java | [ad](https://opentelemetry.io/docs/demo/services/ad/) | Minimal | 未 |
| Kotlin | [fraud-detection](https://opentelemetry.io/docs/demo/services/fraud-detection/) | Full | 未 |
| .NET | [cart](https://opentelemetry.io/docs/demo/services/cart/), [accounting](https://opentelemetry.io/docs/demo/services/accounting/) | Minimal / Full | 未 |
| JavaScript | [payment](https://opentelemetry.io/docs/demo/services/payment/) | Minimal | 未 |
| TypeScript | [frontend](https://opentelemetry.io/docs/demo/services/frontend/) | Minimal | 未 |
| Python | [recommendation](https://opentelemetry.io/docs/demo/services/recommendation/), [load-generator](https://opentelemetry.io/docs/demo/services/load-generator/) | Minimal | 未 |
| Rust | [shipping](https://opentelemetry.io/docs/demo/services/shipping/) | Minimal | 未 |
| C++ | [currency](https://opentelemetry.io/docs/demo/services/currency/) | Minimal | 未 |
| Ruby | [email](https://opentelemetry.io/docs/demo/services/email/) | Minimal | 未 |
| PHP | [quote](https://opentelemetry.io/docs/demo/services/quote/) | Minimal | 未 |
| Elixir | [flagd-ui](https://opentelemetry.io/docs/demo/services/flagd-ui/) | Minimal | 未 |
| 設定のみ | [frontend-proxy](https://opentelemetry.io/docs/demo/services/frontend-proxy/)（Envoy）, [image-provider](https://opentelemetry.io/docs/demo/services/image-provider/)（nginx） | Minimal | 未 |

## 記録項目

言語ごとに次の 7 項目を同じ順で記録する。1〜6 はソースと docs から、7 は起動中のデモから確認する。

1. SDK 初期化の方式（ゼロコード agent / 起動時ラッパー / コード内 setup）と、`OTEL_*` 環境変数がどの層で解釈されるか
2. 使用する計装ライブラリと、実測の `otel.scope.name` / version
3. 手動計装の API（スパン生成・属性・イベント・status・例外記録）
4. メトリクス（instrument の種類、ランタイムメトリクスの有無）
5. ログ（ブリッジの仕組み、trace / span ID 相関の方法）
6. リソース検出と propagator の設定
7. 実測との突き合わせ: Jaeger / Prometheus / OpenSearch で当該サービスの出力を確認し、coverage 表との差分を記録する
