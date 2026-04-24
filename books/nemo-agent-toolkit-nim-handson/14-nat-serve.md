---
title: "第 14 章 nat serve で Web API 化"
---

ここまでの章は、`nat run` で CLI からワンショット実行するか、`nat mcp serve` / `nat a2a serve` で MCP や A2A のプロトコル越しに呼ぶ形でした。本章では一番素直な HTTP API として公開する `nat serve` を扱います。出力は FastAPI の OpenAPI 3.1 アプリなので、**curl からでも OpenAI 互換クライアントからでも、同じエンドポイントを叩ける**のが魅力です。

## この章のゴール

- `nat serve` でワークフローを 8000 番ポートの FastAPI として立ち上げる
- `/health` / `/docs` / `/generate` / `/v1/chat` など主要エンドポイントの役割を把握する
- curl から 2 系統（生の `/generate` と OpenAI 互換 `/v1/chat`）で叩き分ける
- Swagger UI から直接エンドポイントを試す

## 前章からの引き継ぎ

- `nat-nim-handson:1.6.0` イメージ
- NGC API key が `.env` にある
- workflow.yml は第 5 章と同じ `current_datetime` + `wikipedia_search` の 2 ツール ReAct を使う（本書シリーズで馴染みが深いため）

## この章で追加する compose service

- `nat-api` — `nat serve` で FastAPI を 8000 番ポートに立てる

他に Milvus や A2A は要りません。本章は HTTP API の作法と OpenAI 互換レイヤーの体験が主目的なので、単純な 2 ツール ReAct に戻して挙動を読みやすくします。

所要時間は 20-30 分。

## docker-compose.yml を読む

```yaml:ch14-nat-serve/docker-compose.yml
services:
  nat-api:
    image: nat-nim-handson:1.6.0
    env_file:
      - .env
    ports:
      - '8000:8000'
    volumes:
      - ./workflow.yml:/app/workflows/workflow.yml:ro
    command:
      - 'serve'
      - '--config_file'
      - '/app/workflows/workflow.yml'
      - '--host'
      - '0.0.0.0'
      - '--port'
      - '8000'
```

`nat a2a serve` / `nat mcp serve` と同じノリで、`nat serve` の引数をそのまま compose の `command:` に展開するだけです。`--host 0.0.0.0` が大事で、これを抜くとホストマシンから叩けません。

## 起動して /health を確認する

```bash
cd nemo-agent-toolkit-book/ch14-nat-serve
cp ../ch03-hello-agent/.env .env

docker compose up -d
docker compose logs nat-api | tail -15
```

起動ログの末尾に次の 2 行が出れば準備 OK です。

```text
Application startup complete.
Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

ヘルスチェックを打ちます。

```bash
curl -s http://localhost:8000/health
# => {"status":"healthy"}
```

200 / `healthy` が返れば、NAT の FastAPI レイヤーはワークフローの構築まで終わっている証拠です。

## Swagger UI でエンドポイント一覧を眺める

ブラウザで `http://localhost:8000/docs` を開くと、FastAPI 標準の Swagger UI が出てきます。

![FastAPI の Swagger UI。/auth/redirect / /executions / /v1/workflow / /generate / /v1/chat などの POST エンドポイントが並んでいる](/images/nemo-agent-toolkit-nim-handson/ch14-swagger-ui.png =720x)
_NAT が workflow.yml から自動生成した OpenAPI 3.1 のスキーマ。エンドポイントは 20 本以上_

タイトルが `FastAPI 0.1.0` と素っ気ないのは、NAT がデフォルトで設定していないだけです。`workflow.yml` の `general.front_end` で `title` や `version` を上書きできます（本書では割愛）。

この画面から直接「Try it out」ボタンでリクエストを発射できるので、curl を使わなくても手軽に試せます。

## 主要エンドポイントの使い分け

NAT の FastAPI front end が公開するエンドポイントは 20 本近くあります。実運用で使う頻度が高いのは以下の 4 種類です。

| path               | 役割                                                                               |
| ------------------ | ---------------------------------------------------------------------------------- |
| `/generate`        | **単発実行（raw）**。ワークフローを 1 回回し、Final Answer を 1 オブジェクトで返す |
| `/generate/stream` | 同上の SSE ストリーミング版。ReAct の Thought をトークンごとに吐く                 |
| `/v1/chat`         | **OpenAI Chat Completions 互換**。既存の OpenAI SDK / LangChain からそのまま叩ける |
| `/v1/chat/stream`  | 同上のストリーミング版                                                             |

`/generate` は ReAct の生の出口で、API レスポンスがワークフローの出力そのままなのでデバッグに便利。`/v1/chat` は**形を `choices[0].message.content` に整形してくれる**ので、既存のアプリに組み込みやすいです。

## curl で叩く

### `/v1/chat`（推奨、既存アプリから使う想定）

```bash
curl -s -X POST http://localhost:8000/v1/chat \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"What is today date?"}]}' | jq .
```

実機のレスポンス（本書執筆時、Nemotron Super 49B）:

```json
{
  "id": "514be7c9-3df2-4fca-b559-d87c9147020b",
  "object": "chat.completion",
  "model": "unknown-model",
  "created": 1777018084,
  "choices": [
    {
      "finish_reason": "stop",
      "index": 0,
      "message": {
        "content": "Today's date is **2026-04-24**.",
        "role": "assistant"
      }
    }
  ],
  "usage": {
    "prompt_tokens": 5,
    "completion_tokens": 4,
    "total_tokens": 9
  }
}
```

OpenAI Chat Completions 形式そのままです。既存コードが `openai-python` を使っているなら、`base_url="http://localhost:8000/v1"` に差し替えるだけで動きます。

### `/generate`（生の実行、詳細が欲しいとき）

```bash
curl -s -X POST http://localhost:8000/generate \
  -H "Content-Type: application/json" \
  -d '{"input_message":"What is today date?"}' | jq .
```

このエンドポイントは ReAct の Final Answer をそのまま返します。parse が失敗したときは 200 ではなく 4xx 系で `ReActAgentParsingFailedError` の詳細が戻ってくるので、原因切り分けに向いています。

### /v1/chat と /generate のクセの違い

本書の実機検証で、同じ Nemotron Super 49B / 同じ workflow.yml でも挙動が違うパターンに遭遇しました。

- `/v1/chat` は「What is today date?」を 200 で回答し、`Today's date is **2026-04-24**.` が返る
- `/generate` は同じ質問で `ReActAgentParsingFailedError` を 4xx で返すケースあり

推測ですが、`/v1/chat` はメッセージ配列を「文脈付きの直接応答」として扱い ReAct ループを省略した経路に乗るケースがあるのに対し、`/generate` は必ずフル ReAct を回す設計だからだと思われます。本番は **OpenAI 互換の `/v1/chat` に寄せておくのが挙動が安定**という結論でよさそうです。

## ストリーミング応答を受ける

SSE（Server-Sent Events）ストリーミングで、ReAct の Thought が生成される様子を見られます。

```bash
curl -N -X POST http://localhost:8000/v1/chat/stream \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"Briefly list three types of NVIDIA GPUs."}]}'
```

`-N`（`--no-buffer`）を付けないと curl がバッファリングして遅延するので注意。出力は `data: {...}` 形式の行が連続して流れてきます。Web UI なら EventSource API、Python なら `httpx` の `stream=True` で取り込めます。

## よくある詰まりどころ

**`/docs` が 404 / ポートが開かない**

サーバー側の `--host` が `localhost` のまま。`0.0.0.0` に直して再起動すると compose 外から叩けます。

**`/v1/chat` は返るのに `/generate` が parse エラーで返す**

本章の実測で確認した差です。ReAct のフル実行が不安定なケースでは、OpenAI 互換の `/v1/chat` のほうが整形レイヤーが効いて安定します。`/generate` で困ったら `/v1/chat` への差し替えを試してください。

**`Dask is not available, async generation endpoints will not be added.`**

起動ログに出る警告です。NAT の非同期ジョブ機能（`/async` 系）は `dask` 依存が必要で、本書のベースイメージには入れていません。本章で使う同期エンドポイント（`/generate`、`/v1/chat`）は問題なく動くので無視して構いません。

**`uvicorn` がポート競合で起動しない**

別の FastAPI / MCP サーバーが 8000 番を使っている可能性があります。`docker compose` の `ports:` の左側を `18000` などにずらすのが早いです。

## クライアント側の選択肢

本書スコープ外なので深入りしませんが、NAT の API を叩くクライアント 3 パターンを覚えておくと便利です。

- OpenAI SDK `openai-python` — `base_url` を書き換えて使う形。OpenAI Chat Completions にすでに慣れているチームに最適
- LangChain — `ChatOpenAI` の `base_url` を書き換える、または NAT の Python 側を直接叩く。`/v1/chat` 互換なので既存 chain をそのまま移植できる
- httpx / requests — 生の REST クライアント。`/generate` と `/v1/chat` のどちらもフラットに扱える

NAT を「Chat Completions を喋る LLM サービス」としてフロントエンドから使うなら、`/v1/chat` 一択で十分です。

## ここまでで動くもの

- `docker compose up -d` 一発で NAT ワークフローが HTTP で叩ける
- Swagger UI（`http://localhost:8000/docs`）でエンドポイント一覧を見られる
- `/generate` と `/v1/chat` の使い分けが理解できた（安定運用は後者）
- SSE ストリーミングで ReAct の Thought を受け取れる経路を把握した

:::message
本章のサンプルコードは [nemo-agent-toolkit-book リポ](https://github.com/himorishige/nemo-agent-toolkit-book) の `ch14-nat-serve/` ディレクトリにまとめています。
:::

## 次章では

次章は本書の最終章、題材アプリの総仕上げです。第 9-10 章の RAG、第 11 章の Router、第 12 章の A2A、第 13 章の評価、本章の FastAPI 公開を**1 つの docker compose に集約**して、「NVIDIA NAT docs & DGX Spark Q&A エージェント」の完成形を組み立てます。
