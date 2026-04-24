---
title: "第 7 章 Phoenix で観測する"
---

本書ここまで、ReAct の Thought / Action / Observation はターミナルの verbose ログから読んでいました。小さな例ならそれで十分ですが、ツールが増えたりマルチエージェントになったりすると、どこで時間がかかっているのか、どの LLM 呼び出しが問題を起こしているのかを追うのがしんどくなってきます。

本章では Arize Phoenix という OpenTelemetry ベースの観測ツールを docker compose に足し、ReAct エージェントのトレースをブラウザで眺められるようにします。YAML に 3 行足すだけで、スパンツリー、Input / Output、レイテンシが可視化されます。

## この章のゴール

- `ch07-phoenix/` で Phoenix service と NAT service を同じ compose で立ち上げる
- workflow.yml の `general.telemetry.tracing.phoenix` を読み、どの情報が送られるか理解する
- Phoenix UI のプロジェクト一覧 / スパン一覧 / トレース詳細の 3 画面を行き来できる
- `<workflow>` ルートから個別の tool span まで降りて Input / Output を確認する操作を身につける

## 前章からの引き継ぎ

- 前章で ReAct / ReWOO / Tool Calling を動かしたイメージ `nat-nim-handson:1.6.0` はそのまま使う
- NGC API key が `.env` に書かれている

## この章で追加する compose service

- `phoenix` — OTLP 受信 + UI を兼ねる Arize Phoenix の単一サービス（`arizephoenix/phoenix:14.8.0`）
- 既存の `nat` に telemetry 設定を追加し、phoenix の 6006 ポートにトレースを流す

Phoenix は UI（6006）と gRPC OTLP エンドポイント（4317）の 2 ポートを公開します。NAT からの送信は HTTP OTLP（6006/v1/traces）を使うので、実は UI と同じポートに相乗りしています。

## docker-compose.yml の差分

前章までの compose に `phoenix` service を 1 つ追加するだけです。

```yaml:ch07-phoenix/docker-compose.yml
services:
  phoenix:
    image: arizephoenix/phoenix:14.8.0
    ports:
      - '6006:6006'
      - '4317:4317'

  nat:
    image: nat-nim-handson:1.6.0
    env_file:
      - .env
    volumes:
      - ./workflow.yml:/app/workflows/workflow.yml:ro
    command:
      - 'run'
      - '--config_file'
      - '/app/workflows/workflow.yml'
      - '--input'
      - 'Who is the current CEO of NVIDIA, and what date is it today?'
    depends_on:
      - phoenix
```

`depends_on: phoenix` で NAT の起動を Phoenix のあとに回しています。NAT が走り始めた瞬間に Phoenix がまだ待ち受け前だと、初期化トレースが落とされることがあるためです。

## workflow.yml の差分

本体の差分は 5 行です。

```diff yaml:workflow.yml
 general:
   use_uvloop: true
+  telemetry:
+    tracing:
+      phoenix:
+        _type: phoenix
+        endpoint: http://phoenix:6006/v1/traces
+        project: nat-handson-ch07
```

ポイントは 2 つです。

- `endpoint: http://phoenix:6006/v1/traces` — compose ネットワーク内のサービス名 `phoenix` を使います。同じネットワークに属するコンテナからは DNS 解決が効くので、`localhost` ではなく **サービス名** で指定するのがコツです
- `project: nat-handson-ch07` — Phoenix UI でプロジェクト単位に分けて表示するためのラベル。指定しないと `default` プロジェクトに集約されます。本書では章ごとに `nat-handson-chXX` を付けて、章の切り替えがわかりやすいようにしています

他の部分は第 5 章と同じ `_type: nim` + `react_agent` + `current_datetime` + `wiki_search` の 2 ツール構成です。

## 実行する

`.env` を用意したら、Phoenix を先に起動してから NAT を走らせます。

```bash
cd nemo-agent-toolkit-book/ch07-phoenix
cp ../ch03-hello-agent/.env .env

# Phoenix を先に立ち上げてポートを待ち受け状態にする
docker compose up -d phoenix

# NAT を 1 回走らせてトレースを送信
docker compose run --rm nat
```

NAT のログの末尾近くに、次のような行が出ればトレースの送信が完了しています。

```text
2026-04-24 06:04:53 - INFO     - nat.observability.exporter_manager:275 - Stopped exporter 'phoenix'
```

## Phoenix UI を開く

ブラウザで `http://localhost:6006` を開きます。左サイドバーの **Tracing** 配下に、送ったトレースが入るプロジェクトのリストが並びます。

![Phoenix のプロジェクト一覧画面。「nat-handson-ch07」のカードが最新に表示され、Total Traces 1、Latency P50 9.1s と出ている](/images/nemo-agent-toolkit-nim-handson/ch07-phoenix-01-projects.png =720x)
_Phoenix の Projects 画面。章ごとに project 名を分けておくと、章の切り替えがカードで一目で追える_

`nat-handson-ch07` のカードをクリックすると、スパン一覧画面に遷移します。

![nat-handson-ch07 プロジェクトのスパン一覧。`<workflow>` という name の chain スパンが 1 件並んでいる](/images/nemo-agent-toolkit-nim-handson/ch07-phoenix-02-spans.png =720x)
_プロジェクト配下のスパン一覧。Root Spans タブだと workflow 全体のサマリだけが並ぶ_

1 件だけ `<workflow>` という chain kind のスパンがあります。これは NAT がワークフロー全体を括っているルートスパンです。`<workflow>` をクリックすると、本章の見せ場であるトレース詳細が開きます。

## トレース詳細を読む

トレース詳細は左側に **スパンツリー**、中央に **選択中スパンの Info / Annotations / Attributes / Events**、右側に **Annotation summary** が並ぶ 3 ペイン構成です。

![トレース詳細画面。左に `<workflow>` → `<workflow>` → `current_datetime` と `wikipedia_search` の span ツリー、中央に Input「Who is the current CEO of NVIDIA, and what date is it today?」と Output「The current CEO of NVIDIA is Jensen Huang.」が表示されている](/images/nemo-agent-toolkit-nim-handson/ch07-phoenix-03-trace-detail.png =720x)
_トレース詳細画面。左のスパンツリーで ReAct の流れがそのまま見える_

スパンツリーの各ノードは次のように対応しています。

- 外側の `<workflow>`（chain）— NAT ワークフロー全体。Latency 9.1s はこの span の値
- 内側の `<workflow>`（chain）— LangGraph の ReAct グラフ本体
- `current_datetime`（tool）— current_datetime ツール呼び出し
- `wikipedia_search`（tool）— Wikipedia ツール呼び出し

右ペインが空（`No annotation configurations for this project.`）なのは、ラベル付けの機能をまだ使っていないだけで、本書の範囲ではここは無視して問題ありません。

個別のツールスパンをクリックすると、そのツール呼び出しに渡した引数と戻り値がそのまま読めます。

![wikipedia_search span を選んだときの Info。Input に `{"question": "Current CEO of NVIDIA"}`、Output に Wikipedia 検索結果の本文がそのまま表示されている](/images/nemo-agent-toolkit-nim-handson/ch07-phoenix-04-tool-span.png =720x)
_tool span の Info。引数名が `question` である点や、Wikipedia が期待外のページを返しているケースも一目で確認できる_

ここで `wikipedia_search` の引数が `{"question": ...}` と表示されていることに注目してください。第 5 章で扱ったとおり、NAT 1.6.0 の `wiki_search` Function の引数キーは `question` です。Phoenix を使うと、この種の「引数名の齟齬」を実データから確認できます。

## よく使う操作

Phoenix 画面でエージェント開発中によく使う動線は次の 3 つです。慣れると数秒で目的のスパンにたどり着けます。

- プロジェクト一覧 → プロジェクト名 → Root Spans タブでワークフロー単位を把握
- 個別ワークフローをクリック → スパンツリーで失敗 / 遅延 span を特定
- 該当 span をクリック → Attributes タブで OpenTelemetry 属性を確認

特に本書後半のマルチエージェント章（第 11-12 章）では、Router がどのエージェントに振り分けたか、agent-as-tool の中で tool 呼び出しが何段重なったか、を Phoenix で追うのが近道になります。

## よくある詰まりどころ

**ポート 6006 / 4317 がすでに使われている**

別の Phoenix や OTLP 関連のコンテナが動いている場合、`docker compose up` で `Bind for 0.0.0.0:4317 failed: port is already allocated` のエラーが出ます。対処は 2 通りです。

- 既存コンテナを停止する（たとえば `docker stop phoenix-nat`）
- compose ファイルの `ports:` の左側（ホスト側）を別番号に変更する（たとえば `'16006:6006'`）

後者の場合、ブラウザからのアクセス URL も `http://localhost:16006` に変わります。コンテナ間通信（NAT → phoenix）は 6006 のまま保たれるので、`workflow.yml` は書き換えなくて大丈夫です。

**トレースが Phoenix に出てこない**

endpoint の書き間違いか、NAT の起動時に Phoenix がまだ待ち受けていなかったケースが多いです。`docker compose up -d phoenix` で Phoenix を先行起動し、`curl -s -o /dev/null -w "%{http_code}\n" http://localhost:6006/` が 200 を返すのを確認してから NAT を走らせてみてください。

**project 名が反映されずに default に入る**

`project: nat-handson-ch07` を workflow.yml で指定しているのに default プロジェクトにトレースが入る場合、YAML のインデントミスの可能性があります。`general.telemetry.tracing.phoenix` の階層が崩れていないか `nat validate` で検証すると早いです。

## Phoenix Cloud へ切り替える（任意）

Docker で Phoenix を動かすリソースが辛い、あるいは手元と本番で同じ UI を共有したい場合、Phoenix Cloud（無料 SaaS）へ送信先を切り替えるのも選択肢です。

:::details Phoenix Cloud を使う場合の差分

1. [app.phoenix.arize.com](https://app.phoenix.arize.com) にサインアップ
2. API キーを取得
3. workflow.yml の endpoint と header を次のように書き換え

```yaml:workflow.yml
general:
  telemetry:
    tracing:
      phoenix:
        _type: phoenix
        endpoint: https://app.phoenix.arize.com/v1/traces
        project: nat-handson-ch07
        headers:
          api_key: ${PHOENIX_API_KEY}
```

compose 側の `phoenix` service は不要になります。`.env` に `PHOENIX_API_KEY` を追加すれば、手元 Docker と同じ UI 体験が得られます。

:::

## ここまでで動くもの

- `docker compose up -d phoenix && docker compose run --rm nat` でトレースが Phoenix に流れ込む
- Phoenix UI の Projects → Spans → Trace 詳細 → tool span の 4 段ナビを身につけた
- `<workflow>` 配下のツリー構造で ReAct のツール呼び出し順が目視で追える
- 観測基盤が揃ったので、以降の章の複雑なエージェント構成もビジュアルでデバッグできる

:::message
本章のサンプルコードは [nemo-agent-toolkit-book リポ](https://github.com/himorishige/nemo-agent-toolkit-book) の `ch07-phoenix/` ディレクトリにまとめています。
:::

## 次章では

次章では NAT から MCP（Model Context Protocol）サーバーを立ち上げ、Claude Desktop や他の MCP クライアントから NAT ワークフローを呼び出せるようにします。`nat mcp serve` コマンド 1 発で、これまで作ったエージェントが外の世界へ開かれる感触をつかんでいきます。
