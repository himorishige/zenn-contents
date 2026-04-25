---
title: "第 8 章 MCP サーバー連携"
---

MCP（Model Context Protocol）は Anthropic が 2024 年 11 月に公開した、LLM と外部ツールをつなぐためのオープンな仕様です。Claude Desktop / Claude Code / Cursor / Windsurf などのクライアントが標準対応しており、ローカル・リモートを問わずツールを差し替えやすいことが特徴です。

NAT には MCP 両方向のサポートがあります。`nat mcp serve` でワークフローを MCP サーバーとして公開することも、`_type: mcp_tool` を使って外部の MCP サーバーを NAT 内部の tool として組み込むこともできます。本章では前者（サーバー公開）を実機で試しながら、後者の使い方も手短に紹介します。

## この章のゴール

- `nat mcp serve` で NAT ワークフローを MCP サーバーとして立てる
- `nat mcp client ping` / `tool list` でサーバーが健康か、何が expose されているかを確認する
- Claude Desktop や Claude Code から本書のサーバーにつなぐ設定例を把握する
- NAT を MCP クライアントとして使う場合の `_type: mcp_tool` の形を知る

## 前章からの引き継ぎ

- `nat-nim-handson:1.6.0` イメージがビルド済み
- NGC API key が `.env` にある

## この章で追加する compose service

サンプルリポの `ch08-mcp/` には 2 つの service を置いています。

- `nat-server` — `nat mcp serve` で MCP サーバーを 9901 番ポートに立てる（long-running）
- `nat-client` — `nat mcp client` 系コマンドを都度実行するための使い捨て service（compose profile `client` で有効化）

`workflow.yml` は第 5 章と同じ `current_datetime` + `wikipedia_search` の 2 ツール ReAct です。MCP 公開するワークフロー側の中身は特に変える必要はありません。

所要時間は 20-30 分。

## docker-compose.yml を読む

```yaml:ch08-mcp/docker-compose.yml
services:
  nat-server:
    image: nat-nim-handson:1.6.0
    env_file:
      - .env
    ports:
      - '9901:9901'
    volumes:
      - ./workflow.yml:/app/workflows/workflow.yml:ro
    command:
      - 'mcp'
      - 'serve'
      - '--config_file'
      - '/app/workflows/workflow.yml'
      - '--host'
      - '0.0.0.0'
      - '--port'
      - '9901'

  nat-client:
    image: nat-nim-handson:1.6.0
    env_file:
      - .env
    volumes:
      - ./workflow.yml:/app/workflows/workflow.yml:ro
    profiles: ['client']
    entrypoint: ['nat', 'mcp', 'client']
    depends_on:
      - nat-server
```

押さえるべきポイントは次のとおりです。

- `nat-server` の command は `nat mcp serve --host 0.0.0.0 --port 9901`。`0.0.0.0` にしておかないとコンテナ外（別コンテナ / ホスト）から叩けません
- `nat-client` には `profiles: ['client']` を付けて通常の `docker compose up` では起動しないようにしています。実行時は `docker compose --profile client run --rm nat-client ...` と明示
- `nat-client` の `entrypoint` は `nat mcp client` を指定。サブコマンド（`ping` / `tool list` / `tool call`）は CLI 引数で渡す

## MCP サーバーを起動する

```bash
cd nemo-agent-toolkit-book/ch08-mcp
cp ../ch03-hello-agent/.env .env

docker compose up -d nat-server
docker compose logs nat-server | tail
```

起動ログの末尾近くで、次の 2 行を確認できれば準備 OK です。

```text
MCP server URL: http://0.0.0.0:9901/mcp
Uvicorn running on http://0.0.0.0:9901 (Press CTRL+C to quit)
```

NAT は `workflow.yml` に並んでいる Function と、`workflow` 自体を 1 つのツール（`react_agent`）として自動で expose します。上の例だと、MCP クライアントからは 3 つの tool が見えるはずです。

```text
react_agent       # workflow 本体（ReAct エージェント全体）
current_datetime  # 組み込みの現在時刻ツール
wikipedia_search  # 組み込みの Wikipedia 検索ツール
```

## ping でサーバーの健康を確認する

NAT 付属の MCP クライアントで ping を打ってみます。

```bash
docker compose --profile client run --rm nat-client \
  ping --url http://nat-server:9901/mcp
```

次のような応答が返れば成功です。

```text
Server at http://nat-server:9901/mcp is healthy (response time: 4.34ms)
```

サーバーが立ち上がっていない / URL のパスが `/mcp` になっていない場合は、`Connection refused` や `404` が返ります。

## tool list で expose されているツールを確認する

```bash
docker compose --profile client run --rm nat-client \
  tool list --url http://nat-server:9901/mcp
```

ツール名が 1 行ずつ返ります。

```text
react_agent
current_datetime
wikipedia_search
```

`--detail` / `--tool <name>` を付けると、各ツールの入力スキーマも取れます。`current_datetime` のように引数を持たないツールも、NAT のスキーマ定義では `unused` フィールドが必須扱いになっている点は押さえておくと後で引っかかりません。

## tool call で実際に呼ぶ

引数は `--json-args` で JSON 文字列を渡します。

```bash
docker compose --profile client run --rm nat-client \
  tool call wikipedia_search \
  --json-args '{"question":"Who founded NVIDIA?"}' \
  --url http://nat-server:9901/mcp
```

実機で試したところ、NAT 1.6.0 の `streamable-http` クライアントは長時間レスポンスのツールで GET stream が切断・再接続を繰り返すことがあり、タイムアウト（デフォルト 60 秒）に達してしまうケースがありました。本章の wikipedia 検索もヒット状況により 20-60 秒かかる場合があります。**そうした挙動は後段の Claude Desktop / Claude Code 経由の方が安定する**ことが多いので、本書では CLI での `tool call` は疎通確認程度にとどめ、本格的なツール利用はクライアント側（Claude Desktop）に任せる方針にしています。

## Claude Desktop / Claude Code から接続する

macOS や Windows の Claude Desktop から、手元のマシン（Mac / Linux / Windows）で動かしている本章の MCP サーバーに接続できます。Claude Desktop の設定ファイル（`~/Library/Application Support/Claude/claude_desktop_config.json`）に次のような block を追加します。

```json:claude_desktop_config.json
{
  "mcpServers": {
    "nat-handson": {
      "url": "http://NAT_HOST:9901/mcp"
    }
  }
}
```

`NAT_HOST` は NAT を動かしているマシンのホスト名・IP です。Claude Desktop と同じマシンで NAT を動かしているなら `localhost` で OK。別マシン（LAN 内の Linux サーバーや別の PC）なら LAN IP、拠点をまたぐ場合は Tailscale / Cloudflare Tunnel などで振った IP / FQDN を指定します。設定後 Claude Desktop を再起動すると、tool 一覧に `wikipedia_search` / `current_datetime` / `react_agent` が加わります。

Claude Code（CLI）から接続する場合は次のコマンドで登録できます。

```bash
claude mcp add nat-handson -t http http://NAT_HOST:9901/mcp
```

Claude 側は MCP の `streamable-http` transport を標準で喋れるので、サーバー側は追加設定なしで繋がります。

:::message
NAT が発行するツールのうち、`react_agent` は workflow 全体を 1 回のツール呼び出しとして扱うためのエントリポイントです。引数はユーザー質問 1 件（`"What is today's date?"` など）で、裏で本書第 3 章と同じ ReAct ループが走ります。Claude の会話から「NAT 側の判断を丸投げ」したいときに便利です。
:::

## NAT を MCP クライアントとして使う（逆方向の連携）

ここまでとは逆に、外部の MCP サーバー（Claude の Filesystem サーバー、独自 MCP サーバーなど）を NAT ワークフロー内の tool として呼びたい場合は、`_type: mcp_tool` を使います。概念を示すだけの YAML 抜粋ですが、第 11 章以降の構成候補になるので形だけ見ておきます。

```yaml
functions:
  fs_read:
    _type: mcp_tool
    url: http://some-mcp-server:8080/mcp
    tool_name: read_file
```

`url` は接続先 MCP サーバー、`tool_name` はそのサーバーが expose しているツールの 1 つです。NAT は起動時に MCP サーバーから schema を取得し、`fs_read` として内部の Function に組み込みます。ReAct の tool リストに混ぜて使えるので、組み込みツールと自作ツールと外部 MCP ツールを同列に扱える設計になっています。

本書では第 11 章のマルチエージェントで必要になれば登場予定ですが、基本発想は「外側に既製の MCP サーバーがあるなら、`_type: mcp_tool` で 1 行足すだけで tool 化できる」です。

## サーバーの後片付け

検証が終わったら MCP サーバーを停止してクレジット消費とポートを解放しておきます。

```bash
docker compose down
```

## よくある詰まりどころ

**`--host localhost` で外部から繋がらない**

サーバー起動時のホストを `localhost` のままにすると、コンテナ内部でしか listen しません。他コンテナや他ホストから叩きたい場合は `--host 0.0.0.0` を指定します。

**`http://.../` に GET を打っても 406 が返る**

MCP の `streamable-http` は独自プロトコルなので、普通の `curl` では 406 が返ります。これは正常です。疎通確認は `nat mcp client ping` を使います。

**`tool call` がタイムアウトする**

NAT 1.6.0 の streamable-http クライアントは、長時間応答するツール（Wikipedia API が遅い、LLM が絡む tool）で GET stream の切断と再接続を繰り返すことがあり、デフォルト 60 秒のタイムアウトに引っかかる場合があります。Claude Desktop / Claude Code 経由ではこの挙動は見られないので、本格的なツール呼び出しは MCP クライアント側に任せるのが現実解です。

**Port 9901 がすでに使われている**

別の MCP サーバーが動いている場合は `docker compose` の `ports:` の左側を別番号（`'19901:9901'` など）に振り替えます。クライアント側の `--url` もそのポートに合わせます。

## ここまでで動くもの

- `ch08-mcp/` で `nat mcp serve` が MCP サーバーを 9901 番ポートに立ち上げる
- `nat mcp client ping` / `tool list` で疎通と expose 状況を確認できる
- Claude Desktop / Claude Code の設定に `http://NAT_HOST:9901/mcp` を 1 行足せば、本書のエージェントが外から叩けるようになる
- 逆方向（NAT から他 MCP サーバーを呼ぶ）は `_type: mcp_tool` で 1 行追加するだけで済むと把握した

:::message
本章のサンプルコードは [nemo-agent-toolkit-book リポ](https://github.com/himorishige/nemo-agent-toolkit-book) の `ch08-mcp/` ディレクトリにまとめています。
:::

## 次章では

次章から 2 章にわたって RAG を扱います。第 9 章では Milvus standalone（etcd + minio + milvus）を Docker で立て、NAT 公式 docs 24 ファイル（Apache 2.0）を 1,034 チャンクに分割して投入し、NIM の Embedding API と `milvus_retriever` を通じて「NAT docs から回答を組み立てる RAG エージェント」を組みます。第 10 章ではそれを運用面（カテゴリ別フィルタ・top_k チューニング）へ発展させます。
