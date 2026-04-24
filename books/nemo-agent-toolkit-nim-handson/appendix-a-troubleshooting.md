---
title: "付録 A トラブルシュート集"
---

本書の実機検証で遭遇したハマりポイントを 1 箇所にまとめました。章順で並んでいるので、読んでいる章で似た症状が出たら該当節を直接たどってください。

## Docker / Colima 周り（第 2 章関連）

### `docker compose version` で v1 が返る

古い `docker-compose` バイナリがシステムに残っているケース。本書は Compose v2（`docker compose`）前提。

```bash
# v2 対応版を入れる（Ubuntu の場合）
sudo apt-get install docker-compose-plugin
docker compose version  # v2.x.x が返ることを確認
```

### Colima 起動時に「Process exited with non-zero status」

macOS 権限設定が原因のケース。

```bash
colima stop && colima delete
colima start --vm-type vz --cpu 4 --memory 8 --disk 30
```

## NAT インストール（第 2 章）

### `langchain-nvidia-ai-endpoints==0.4.0` not found

このバージョンは存在しません。`nvidia-nat[langchain,mcp,eval,phoenix,a2a]==1.6.0` の extras が自動で解決するので、明示インストール不要です。

### `milvus_lite` import error with `ModuleNotFoundError: No module named 'pkg_resources'`

`setuptools 80+` では `pkg_resources` が同梱されていません。本書の `requirements.txt` では `setuptools<80` に pin 済み。

## ReAct / NIM モデルの挙動（第 3-8 章）

### `Missing 'Action Input:' after 'Action:'` で parse 失敗

ReAct の出力フォーマット崩れ。対処順：

1. `parse_agent_response_max_retries: 5` を `workflow:` に追加
2. モデルを `nvidia/llama-3.3-nemotron-super-49b-v1` に切り替える（本書デフォルト）
3. Nemotron Mini 4B や軽量モデルは parse 不安定なので、検証用にとどめる

### `"This model only supports single tool-calls at once!"` 500 エラー

Tool Calling で複数 tool_calls を同時発行した結果。NIM のモデルごとに並列対応が違う。本書は `tool_calling_agent` を `wiki_search` 単独構成で回避しています（第 6 章）。

### `current_datetime` が `unused Field required` で失敗（ReWOO / MCP client）

NAT 1.6.0 の `current_datetime` Function は空引数 `{}` で Pydantic 検証エラーになります。第 6 章 ReWOO と第 8 章 `nat mcp client tool call` で確認済み。`wikipedia_search` など引数ありのツールに絞るのが回避策です。

### `wikipedia_search` の引数キーは `question`

原稿で `{"query": "..."}` と書かれている例を見かけますが、NAT 1.6.0 の `wiki_search` Function のスキーマは `question` です（第 5 章で判明）。

### `nat info components` で `-n <name>` が結果数を減らすだけ

コンポーネント名で絞り込むには `-q <name>`。`-n` は `num_results` です（第 4 章）。

## MCP / A2A（第 8・12 章）

### `nat mcp client tool call` が 60 秒タイムアウト

NAT 1.6.0 の `streamable-http` クライアントが長時間応答のツールで GET stream を切断・再接続するバグっぽい挙動。Claude Desktop / Claude Code 経由のほうが安定します（第 8 章）。

### A2A クライアントが `Shared workflows (such as react_agent) cannot use A2A client function groups directly`

`workflow._type` を `per_user_react_agent` に変更してください。通常の `react_agent` では `a2a_client` function group を `tool_names:` に入れられません（第 12 章）。

### A2A サーバーに `GET /.well-known/agent-card.json` が 404 / タイムアウト

サーバー側の `host` が `localhost` だと他コンテナから叩けません。`0.0.0.0` に直して再起動します（第 12 章）。

### `a2a_client` 呼び出しで `Cannot convert type <str> to <InputArgsSchema>`

NAT 1.6.0 の A2A client が Agent Card から展開したスキルを ReAct から呼ぶときに発生する既知エラー。ReAct 内部で retry が走りますが、引用付きの RAG 応答は返らないケースがあります。本書では第 15 章の「よくある詰まりどころ」で注意喚起しています。安定動作させたいときは第 12 章のように A2A server を単独で curl 叩きし、RAG を直接呼ぶ構成へ分解してください。

### `router_agent` の `branches:` に `function_groups:` の名前を入れると `Function 'xxx' not found`

NAT 1.6.0 の `router_agent` は `functions:` 配下の function 名しか branches に取りません。function_group を展開する workflow が必要なときは `per_user_react_agent` の `tool_names:` を採用します（第 15 章）。

## Router Agent（第 11 章）

### `ChatRequestOrMessage` のスキーマ不一致エラー

`router_agent` の `branches:` に `react_agent` / `rewoo_agent` / `tool_calling_agent` を直接入れると発生します。回避策は 2 つ：

- tool / retriever レベルに branches を限定する（第 11 章）
- `per_user_react_agent` の `tool_names:` に function_group を入れる（第 12 章、第 15 章）

## RAG / Milvus（第 9・10 章）

### `metric type not match: expected=L2, actual=COSINE`

ingest 側の `metric_type` と NAT `milvus_retriever` の検索 metric が食い違っています。NAT 1.6.0 の `milvus_retriever` は L2 デフォルト。ingest も L2 で index を張ります。

### `Milvus Lite の uri` が HttpUrl 検証で拒否される

NAT 1.6.0 の `milvus_retriever.uri` は `HttpUrl` 型。ファイルパスは受け付けないため、Milvus standalone（Docker）を立てる構成にしてください（第 9 章）。

### ingest 後に `fail to connect to milvus:19530`

`docker compose up -d milvus` が終わる前に ingest を走らせた可能性。`docker compose ps` で milvus が `Up` になってから実行。

## nat eval（第 13 章）

### `Input tag 'ragas' found ... does not match any of the expected tags`

`ragas` evaluator は `nvidia-nat[ragas]` extras が必要。本書では `trajectory` + `langsmith` exact_match に絞って回避しています。

### `eval: Extra inputs are not permitted` / `eval: Field required`

YAML スキーマ違反が多いです。次の 3 点を確認：

- `eval.dataset` ではなく `eval.general.dataset` が正解
- `eval.llms` ではなくトップレベル `llms` に judge_llm を置く
- `structure.id_key: id` は NAT 1.6.0 で許容されないので削除

### `OSError: [Errno 16] Device or resource busy: '/app/outputs'`

`eval.general.output.dir` を compose マウントポイント直下にすると rmtree で失敗します。`/app/outputs/run` のように subdir を指定してください。

## nat serve / FastAPI（第 14 章）

### `/v1/chat` は返るが `/generate` が 4xx で parse エラー

実測で確認した挙動差。`/v1/chat` は OpenAI 互換の整形レイヤーで ReAct 崩れを吸収しやすく、`/generate` は生の ReAct を返します。運用は `/v1/chat` に寄せるのが安定です。

### `Dask is not available, async generation endpoints will not be added.`

警告ですが無視可能です。本書で使う同期エンドポイント（`/generate` / `/v1/chat`）は影響しません。

## Phoenix（第 7 章）

### ポート 6006 / 4317 がすでに使われている

本書と別の Phoenix コンテナが動いている可能性。

```bash
docker ps | grep phoenix
docker stop <existing-phoenix>   # 止めて良いか確認してから
```

または `docker-compose.yml` の `ports:` の左側を `16006:6006` のようにずらす。

### NAT からトレースが送信されない

`general.telemetry.tracing.phoenix.endpoint` のサービス名が間違っている可能性。compose 内なら `http://phoenix:6006/v1/traces`（service 名）。

## 一般的な回避策

### NIM API key で 401

`.env` の NGC_API_KEY に改行やスペースが入っているのがよくあります。

```bash
cat .env  # 末尾に余分な文字がないか確認
```

### NIM Rate Limit で 429

build.nvidia.com のデフォルト 40 req/min に抵触。対処：

- `eval.general.max_concurrency: 2` で並列抑制
- 1-2 分待って再試行
- 付録 B の cost 試算を参照

### build 時の pymilvus / faiss エラー

本書 `requirements.txt` の pin を信じて再ビルド：

```bash
docker build --no-cache -t nat-nim-handson:1.6.0 docker/nat/
```
