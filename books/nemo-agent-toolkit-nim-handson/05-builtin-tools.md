---
title: "第 5 章 組み込みツールを束ねる"
---

ここまでの章で動かした ReAct エージェントは、`current_datetime` というツールを 1 つしか持っていませんでした。使えるツールが 1 つだと、ReAct は常にそれを呼ぶだけで、選択の余地がありません。本章では `wikipedia_search` を追加して 2 ツール構成にし、ReAct が「質問ごとに違うツールを選ぶ」様子を観察していきます。

タイトルに「束ねる」と入れているのは、単にツールを並列で並べるだけでなく、**LLM にツールを取捨選択させる設計**にフォーカスを当てたいからです。この観点は、第 11 章のマルチエージェント（Router + agent-as-tool）の土台にもなります。

## この章のゴール

- `workflow.yml` の `functions:` に 2 種類以上のツールを並べられるようになる
- ReAct ループが質問の内容に応じてツールを選び分ける挙動を目撃する
- `max_iterations` の役割と使いどころを理解する
- 組み込みツールだけで実現できることの範囲を掴む

## 前章からの引き継ぎ

- 第 4 章で YAML の 4 セクションの役割を押さえた
- `ch03-hello-agent/` の挙動は元に戻した（`temperature: 0.0`、`verbose: true`）
- `nat-nim-handson:1.6.0` イメージはビルド済み

## この章で追加する compose service

なし。サンプルリポの `ch05-builtin-tools/` ディレクトリに独立した `workflow.yml` と `docker-compose.yml` が用意されています。ベースイメージは第 3 章と同じものを使い回します。

所要時間は 20-30 分（Wikipedia 検索の体感レイテンシ次第）。

## ディレクトリ構成

```bash
cd nemo-agent-toolkit-book/ch05-builtin-tools
ls
# .env.example  README.md  docker-compose.yml  workflow.yml
```

`.env` の中身は第 3 章と同じ `NGC_API_KEY` だけなので、第 3 章のコピーを流用して構いません。

## workflow.yml を見る

2 ツール構成の YAML はこうなります。

```yaml:ch05-builtin-tools/workflow.yml
general:
  use_uvloop: true

llms:
  nim_llm:
    _type: nim
    model_name: meta/llama-3.1-8b-instruct
    api_key: ${NGC_API_KEY}
    temperature: 0.0
    max_tokens: 512

functions:
  current_datetime:
    _type: current_datetime
  wikipedia_search:
    _type: wiki_search
    max_results: 2

workflow:
  _type: react_agent
  tool_names:
    - current_datetime
    - wikipedia_search
  llm_name: nim_llm
  verbose: true
  max_iterations: 6
```

第 3 章の YAML との差分はわずかです。

```diff yaml:workflow.yml
 functions:
   current_datetime:
     _type: current_datetime
+  wikipedia_search:
+    _type: wiki_search
+    max_results: 2

 workflow:
   _type: react_agent
   tool_names:
     - current_datetime
+    - wikipedia_search
   llm_name: nim_llm
   verbose: true
+  max_iterations: 6
```

見るべきポイントは 3 つです。

1 点目、`functions:` に `wikipedia_search` を足しました。内部的には `_type: wiki_search` の組み込み Function を呼び出しており、`max_results: 2` で Wikipedia から返す検索結果の数を制限しています。NAT 組み込みの Wikipedia ツールは外部 API key を必要としないため、`.env` に追加で書くものはありません。

2 点目、`workflow:` の `tool_names:` にも `wikipedia_search` を追加しました。`functions:` に登録しただけではエージェントはツールを認識しないので、**`tool_names:` で明示的に有効化する**のがお約束です。

3 点目、`max_iterations: 6` を追加しました。ReAct は条件しだいで Thought / Action / Observation のループをいくらでも回せてしまい、うまく答えを出せないまま tool を呼び続けるケースもあります。`max_iterations` を指定すると、ループが上限に達した時点で「Final Answer を出せないまま」でも打ち切ってくれます。本書では 5-8 を目安に、質問の難易度に応じて調整します。

:::details `wiki_search` の他のパラメータ

`max_results` の他に、`lang`（言語コード、デフォルト `en`）や `summary_length`（要約の長さ）なども指定できます。手元で詳細を確認したい場合は次のコマンドが便利です。

```bash
docker compose run --rm nat \
  info components -t function -n wiki_search
```

`-n` にコンポーネント名を渡すと、そのコンポーネントの YAML スキーマが表示されます。Pydantic のフィールドがそのまま出るので、どのキーが必須でどのキーが任意かを確認するときに重宝します。

:::

## docker-compose.yml の違い

compose ファイル側は、既定の入力だけ第 3 章から差し替えています。

```yaml:ch05-builtin-tools/docker-compose.yml
services:
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
```

「NVIDIA の現 CEO は誰？」と「今日の日付は？」の 2 つを同じ質問に詰め込んで、ReAct が両方のツールを呼びに行くかを観察する入力にしました。

## 実行する

`.env` を用意したら 1 コマンドです。

```bash
cd nemo-agent-toolkit-book/ch05-builtin-tools
cp ../ch03-hello-agent/.env .env

docker compose run --rm nat
```

ReAct が 2 ツール使い分ける成功パターンの出力は次のような流れになります。

```text
[AGENT]
Thought: I need to answer two things: the current CEO of NVIDIA, and today's date.
Let me start by searching Wikipedia for NVIDIA's current CEO.
Action: wikipedia_search
Action Input: {"query": "NVIDIA CEO"}

[TOOL]
Observation: 1. Jensen Huang — Co-founder, president and CEO of NVIDIA...
             2. ...

[AGENT]
Thought: Good. Now I need today's date.
Action: current_datetime
Action Input: {}

[TOOL]
Observation: The current time of day is 2026-04-24 10:03:18 +0000

[AGENT]
Thought: I have both pieces of information.
Final Answer: NVIDIA's current CEO is Jensen Huang, and today's date is 2026-04-24.
```

注目してほしいのは、ReAct が Wikipedia 側の Observation を読んだあとに「そういえばもう 1 つ日付の質問もあったな」と `current_datetime` に切り替えている点です。ツールを並列に呼んでいるわけではなく、1 問ずつ順番に解決しています。ReAct はこのように逐次実行が前提のパターンです。

## 違う質問で挙動を揺らす

ReAct の挙動は質問の曖昧さに応じてよく変わります。いくつか試してみましょう。

**知識単体**

```bash
docker compose run --rm nat \
  run --config_file /app/workflows/workflow.yml \
  --input "Give me a short summary of the Apollo 11 mission."
```

Wikipedia だけを 1 回呼んで Final Answer を返す、短い ReAct 軌跡になるはずです。

**どちらのツールも不要**

```bash
docker compose run --rm nat \
  run --config_file /app/workflows/workflow.yml \
  --input "What is 12 plus 34?"
```

算数だけなので ReAct はツールを呼ばず、Thought の中で答えを出すケースが多いです。モデルが Llama 3.1 8B だとたまに `wiki_search` を呼ぼうとして空振りすることもあります。そういうブレも ReAct の個性として観察してみてください。

**ループが長くなる質問**

```bash
docker compose run --rm nat \
  run --config_file /app/workflows/workflow.yml \
  --input "List three famous physicists and their main achievements."
```

Wikipedia を 3 回呼んでから Final Answer を組み立てる、長めの軌跡になる可能性があります。`max_iterations: 6` に届く場合は、Thought / Action / Observation が途中で打ち切られて「It looks like the task has been terminated...」のような出力になります。打ち切りを防ぎたければ `max_iterations` を 10 ほどに上げるか、質問を短く切り直すのが現実解です。

## Python で独自ツールを足す選択肢

本書では扱わないので駆け足で触れますが、NAT では `functions:` に自前の Python 関数を登録するルートも用意されています。大まかな手順は次のとおりです。

1. `@nat.tool` のようなデコレータで関数を登録する Python ファイルを書く
2. `workflow.yml` の `functions:` に `_type: your_module.your_tool` を指定する
3. 引数と戻り値は Pydantic でスキーマ化する

第 11 章のマルチエージェントで 1 箇所だけ利用しますが、それ以外の章は組み込み Function で進めます。「外部 API を叩くツールが自作できる」ことだけ頭の片隅に置いてもらえればこの章としては十分です。

## 組み込みツールでできないこと

組み込み Function は取り回しがよい反面、できることはシンプルな範囲に限られます。ざっくり次のような線引きです。

| 実現したい             | 組み込みの限界                                 | 代替ルート                  |
| ---------------------- | ---------------------------------------------- | --------------------------- |
| 社内ドキュメントの検索 | LangChain retriever 経由にする必要がある       | 第 9-10 章（RAG）で取り組む |
| 外部 SaaS の操作       | 認証情報ごとに個別実装が必要                   | 第 8 章（MCP クライアント） |
| 計算処理・コード実行   | 組み込みで `code_execution` はあるが制約が強い | 独自 Python Function or MCP |

本書では **社内ドキュメント検索 → 第 9-10 章、SaaS 操作 → 第 8 章** の流れで少しずつ広げます。いまは「組み込みだけでもエージェントは動かせる。その先は章を追うごとに広がる」くらいの感覚で構いません。

## ここまでで動くもの

- `ch05-builtin-tools/` で `current_datetime` と `wikipedia_search` を併用する ReAct が回る
- 質問ごとに ReAct がツールを選び分けるのが観察できる
- `max_iterations` の効き方を把握した
- `nat info components -t function -n <name>` で各 Function のパラメータを引けるようになった

:::message
本章のサンプルコードは [nemo-agent-toolkit-book リポ](https://github.com/himorishige/nemo-agent-toolkit-book) の `ch05-builtin-tools/` ディレクトリにまとめています。
:::

## 次章では

次章では `workflow` セクションの `_type` を入れ替えて、ReAct 以外のエージェントパターン（ReWOO / Tool Calling / Router）を試します。同じツール集合でも、パターンを変えると振る舞い・tool 呼び出し回数・レイテンシがどう変わるかを手元で見比べていきます。
