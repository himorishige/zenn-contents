---
title: "第 4 章 LangGraph を NAT function として組み込む"
---

第 4 章では、第 3 章の比較を踏まえて LangGraph を実際に NAT に組み込みます。本章はハンズオン中心で、最小の state graph を 1 本書いて NAT YAML から呼び出し、Langfuse に trace が届くところまでを通します。

ハンズオンの題材は「2 ノードの最小グラフ（質問種別を分類する → Nemotron Super 49B で日本語応答する）」です。社内 Q&A の本番形は第 5-7 章で組み上げますが、本章ではまず「LangGraph を NAT に乗せる感覚」を掴むことに集中します。

## この章のゴール

- NAT 1.6.0 が提供する `_type: langgraph_wrapper` の仕組みを理解する
- `State` クラス + ノード関数 + エッジ宣言の 3 点セットで LangGraph を書ける
- `path/to/module.py:graph_name` の指定で、Python ファイルに置いたグラフを NAT から呼べる
- Langfuse の trace で LangGraph workflow が `langgraph_wrapper` として識別されるところまで確認できる
- NAT-direct（前作の `react_agent` 等）からの段差が想像以上に小さいことを体感する

## NAT 1.6.0 の `langgraph_wrapper` を見ておく

NAT 1.6.0 の `nvidia-nat[langchain]` extra には、`_type: langgraph_wrapper` という組み込み function type が用意されています。役割は単純で、「Python ファイル上の `CompiledStateGraph` を NAT の function に変換する」ためのアダプタです。

設定スキーマを `nat info components -t functions -q langgraph` で確認すると、本章で押さえる主要フィールドは次の 3 つです。

```yaml
workflow:
  _type: langgraph_wrapper
  description: "..."
  graph: /path/to/module.py:graph_name # ← 必須
  dependencies: # ← 任意：sys.path に追加するディレクトリ
    - /path/to/module/dir
  env: ${ENV_FILE_PATH} # ← 任意：追加で読み込む .env
```

`graph` には **Python ファイルへの絶対パス + `:` + そのファイル内の attribute 名** を書きます。指す先は `CompiledStateGraph` インスタンスか、`RunnableConfig` を受け取って `StateGraph | CompiledStateGraph` を返す callable のどちらかです。コンテナ内のパスを書くので、本章では `/app/graphs/echo_graph.py:make_graph` という指定を使います。

入出力は LangChain の標準的なメッセージ形式です。NAT のコンソール front-end から見ると、ユーザーの質問がそのまま `messages` の最後に追加されて LangGraph に渡り、LangGraph が返した最後のメッセージの `content` が応答として返ってくる、という素直な動きをします。

## ハンズオンの構成

本章のハンズオンは、3 ファイルだけで完結します。

```
ch04-langgraph/
├── docker-compose.yml      # NAT を Langfuse network に attach して起動
├── workflow.yml            # NAT YAML（langgraph_wrapper を workflow に）
├── graphs/
│   └── echo_graph.py       # 2 ノードの LangGraph state graph
└── .env                    # NGC_API_KEY と Langfuse の鍵
```

第 2 章で組んだ Langfuse スタックがすでに動いている前提です。trace を見たくないみなさんは `general.telemetry.tracing` を丸ごと外しても本章の動作には影響ありません。

## LangGraph state graph を書く

最初に Python ファイルから書きます。`graphs/echo_graph.py` の中身です。

```python:graphs/echo_graph.py
"""Minimal LangGraph state graph wrapped as NAT function.

Two-node graph: classify -> respond.
The classify node tags the question type (date / general),
the respond node produces a Japanese reply using Nemotron Super 49B via NIM.
"""

from __future__ import annotations

import os
from typing import Annotated

from langchain_core.messages import AIMessage, BaseMessage, SystemMessage
from langchain_core.runnables import RunnableConfig
from langchain_nvidia_ai_endpoints import ChatNVIDIA
from langgraph.graph import END, START, StateGraph
from langgraph.graph.message import add_messages
from typing_extensions import TypedDict

# 整形は uvx ruff check --fix && uvx ruff format で行う前提（書籍リポ pyproject.toml）


class State(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    question_type: str | None


def _llm() -> ChatNVIDIA:
    return ChatNVIDIA(
        model="nvidia/llama-3.3-nemotron-super-49b-v1",
        api_key=os.environ["NGC_API_KEY"],
        base_url="https://integrate.api.nvidia.com/v1",
        temperature=0.0,
        max_tokens=512,
    )


def classify_node(state: State) -> dict:
    last = state["messages"][-1]
    text = last.content.lower() if isinstance(last.content, str) else ""
    if any(k in text for k in ["date", "today", "日付", "今日"]):
        return {"question_type": "date"}
    return {"question_type": "general"}


def respond_node(state: State) -> dict:
    qtype = state.get("question_type", "general")
    system_text = (
        "あなたは社内文書 Q&A の補助をする日本語アシスタントです。"
        f"今回の質問種別は {qtype} です。質問種別を意識して 1〜2 文で簡潔に答えてください。"
    )
    history = [SystemMessage(content=system_text), *state["messages"]]
    reply = _llm().invoke(history)
    if not isinstance(reply, AIMessage):
        reply = AIMessage(content=str(reply.content))
    return {"messages": [reply]}


def make_graph(_config: RunnableConfig):
    builder = StateGraph(State)
    builder.add_node("classify", classify_node)
    builder.add_node("respond", respond_node)
    builder.add_edge(START, "classify")
    builder.add_edge("classify", "respond")
    builder.add_edge("respond", END)
    return builder.compile()
```

ファイルが小さいわりに、登場人物が多く感じるはずです。ひとつずつ見ていきます。

最初の `State` クラスは LangGraph の状態を表現する `TypedDict` です。`messages` は LangChain のメッセージ列で、`add_messages` reducer を使うことで「ノードが追記したメッセージを既存の履歴に足す」挙動になります。`question_type` は本章のサンプル独自のフィールドで、classify ノードが入れた値を respond ノードが読みます。

`_llm()` は ChatNVIDIA の NIM クライアントを毎回作る関数です。本書では `NGC_API_KEY` を `.env` から読み込んで Cloud NIM を叩きます。`base_url` を明示しているのは、社内環境で self-hosted NIM に切り替える際の差し替えポイントになるからです。

`classify_node` は質問の種別を判定して `question_type` フィールドに書き込むだけのノードです。LLM を使わず文字列マッチで簡単に分類します。本章は小ぶりに留めますが、第 7 章では LLM を使った分類に置き換えます。

`respond_node` は質問種別を踏まえてシステムメッセージを組み立て、Nemotron Super 49B に投げるノードです。返ってきた応答を `messages` に追加し、グラフの状態を更新します。

最後の `make_graph` が本章の主役です。`RunnableConfig` を受け取って `CompiledStateGraph` を返す callable で、NAT 1.6.0 の `langgraph_wrapper` はまさにこの形を期待しています。`StateGraph` を作って `add_node` / `add_edge` でグラフを組み、`compile()` で実行可能な形にして返します。

## NAT YAML から呼び出す

次に `workflow.yml` を書きます。NAT-direct 版と見比べやすいように、まずは前作の Hello Agent と並べます。

```yaml:workflow.yml（本書 第 4 章 — LangGraph 版）
general:
  use_uvloop: true
  telemetry:
    tracing:
      langfuse:
        _type: langfuse
        endpoint: ${LANGFUSE_OTLP_ENDPOINT}
        public_key: ${LANGFUSE_PUBLIC_KEY}
        secret_key: ${LANGFUSE_SECRET_KEY}
        batch_size: 1
        flush_interval: 1.0
        resource_attributes:
          service.name: nat-langgraph-poc
          deployment.environment: poc

llms:
  nim_llm:
    _type: nim
    model_name: nvidia/llama-3.3-nemotron-super-49b-v1
    api_key: ${NGC_API_KEY}
    temperature: 0.0
    max_tokens: 512

workflow:
  _type: langgraph_wrapper
  description: "Echo + classify minimal LangGraph wrapped by NAT"
  graph: /app/graphs/echo_graph.py:make_graph
  dependencies:
    - /app/graphs
```

```yaml:前作 ch03-hello-agent/workflow.yml（NAT-direct 版・参考）
general:
  use_uvloop: true

llms:
  nim_llm:
    _type: nim
    model_name: nvidia/llama-3.3-nemotron-super-49b-v1
    api_key: ${NGC_API_KEY}
    temperature: 0.0
    max_tokens: 512

functions:
  current_datetime:
    _type: current_datetime

workflow:
  _type: react_agent
  tool_names:
    - current_datetime
  llm_name: nim_llm
  verbose: true
```

差分はほぼ workflow ブロックだけです。前作は `_type: react_agent` で tool 名を列挙していたのに対し、本章は `_type: langgraph_wrapper` で graph ファイルを 1 行指すだけ。`functions` ブロックはなくなり、tool の呼び出しは LangGraph の中で完結します。

「NAT-direct から LangGraph に移すと Python が増える」という直感はそのとおりですが、増えた Python は graph 1 ファイルに集約されていて、YAML 側の構造はむしろシンプルになっている、というバランスです。

`graph: /app/graphs/echo_graph.py:make_graph` は、コンテナ内のパスです。ホスト側の `./graphs/` を `/app/graphs/` にマウントする `docker-compose.yml` 側で対応を取ります。`dependencies: [/app/graphs]` は、`make_graph` の中で `from utils import ...` のように補助モジュールを import したくなったときの逃げ道です。本章のサンプルでは不要ですが、書いておいても害はないので残しています。

## docker compose で実行する

最後に compose を書いて起動します。

```yaml:docker-compose.yml
services:
  nat:
    image: nat-nim-handson:1.6.0 # 第 2 章で建てたイメージでも nat-prod-ops:1.6.0 でも可
    env_file:
      - .env
    networks:
      - langfuse_default # ← Langfuse の compose の network に attach する
    volumes:
      - ./workflow.yml:/app/workflows/workflow.yml:ro
      - ./graphs:/app/graphs:ro
    command:
      - "run"
      - "--config_file"
      - "/app/workflows/workflow.yml"
      - "--input"
      - "今日は何日ですか？"

networks:
  langfuse_default:
    external: true
```

`networks` に `external: true` で Langfuse 側の compose で作った `langfuse_default` ブリッジに attach しているのが重要なポイントです。これがないと NAT コンテナから `langfuse-langfuse-web-1:3000` を解決できず、OTLP 送信が失敗します。

`.env` には NGC_API_KEY と、第 2 章で発行した Langfuse の API key、それから OTLP のエンドポイント URL を書いておきます。

```bash:.env
NGC_API_KEY=nvapi-...
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_OTLP_ENDPOINT=http://langfuse-langfuse-web-1:3000/api/public/otel/v1/traces
```

実行はワンライナーです。

```bash
cd ch04-langgraph
docker compose run --rm nat
```

初回はベースイメージから `langchain-nvidia-ai-endpoints` のインポートに数秒かかりますが、2 回目以降は 3 秒前後で応答が返ります。手元で試した出力はこんな具合でした。

```
Configuration Summary:
--------------------
Workflow Type: langgraph_wrapper
Number of Functions: 0
Number of Function Groups: 0
Number of LLMs: 1
...

Workflow Result:
ご質問の際の現在日時は、【2024年3月8日 14:30】です。ただし、実際の現在日時は、ご使用のデバイスやサーバーの時刻に基づきます。最新の日時を確認するには、デバイスの時計やオンライン時計サービスを参照ください。
```

LLM が日付を勝手に答えてしまう（hallucination）のは Nemotron Super 49B の素の挙動です。本章では tool を使わない構成なので想定通りで、第 7 章で `current_datetime` tool を classify ノードと組み合わせて回避します。本章で確認したいのは「**LangGraph workflow が NAT 経由で動いた**」という 1 点だけです。

## Langfuse で trace を確認する

ブラウザで `http://localhost:3000` を開き、第 2 章で作った project の **Tracing → Traces** を覗きます。`langgraph_wrapper` という名前の trace が新しく追加されているはずです。

```
trace: langgraph_wrapper
  input:  今日は何日ですか？
  output: ご質問の際の現在日時は、【2024年3月8日 14:30】です。...
  observations: 2 件
  latency: 2.806s
```

trace を開いてツリーを展開すると、observations は次の 2 段になっています。

```
langgraph_wrapper                  ← workflow level
  └─ ChatNVIDIA / NIM call         ← respond ノード内の LLM 呼び出し
```

classify ノードは LLM を呼ばないので observation が立ちません（同期関数の中で `question_type` を書き換えているだけ）。LLM を呼ぶノードが span として残るのは、本章のような小さなサンプルで挙動を理解する手がかりになります。第 7 章で classify を LLM ベースに置き換えると、ここの observations が 3 段に増えます。

`metadata.attributes` には NAT が付ける `nat.framework`, `nat.subspan.name`, `nat.workflow.run_id` のような attribute が並びます。第 11 章で扱うトレース解析の素材として、いまのうちにツリーを眺めておくと頭に入りやすいです。

## NAT-direct と LangGraph の見分け方

ここまでで、NAT-direct 版と LangGraph 版の両方が動く状態になりました。「これからの章をどちらで書くか」を判断する目印を、最後に並べておきます。

| 観点                                       | NAT-direct（前作）        | LangGraph（本章以降）        |
| ------------------------------------------ | ------------------------- | ---------------------------- |
| YAML の行数                                | tool 数に応じて増える     | workflow ブロック 5 行で固定 |
| Python コード                              | 不要                      | グラフ 1 ファイル必須        |
| 状態管理                                   | ReAct の内部履歴          | `State` クラスで型保証       |
| 並列実行                                   | LLM 任せ                  | `add_edge` の並列化で宣言的  |
| trace の見やすさ                           | tool 単位で span が立つ   | ノード単位で span が立つ     |
| classify / preprocess を明示的に書きたいとき | tool として書く必要がある | ノードとして自然に書ける     |

第 7 章の社内 Q&A はマルチノードで状態を持ち回す構成なので、LangGraph がはまります。第 4 章の本サンプルは「LangGraph の使い心地」を掴むための最小例で、ノード数を絞って LangGraph の構造そのものを見えやすくしました。

## ハマりポイント

ハンズオンを進めるなかで遭遇しやすい落とし穴を 3 点だけ。

1 つ目は **`graph` のパス指定** です。`/app/graphs/echo_graph.py:make_graph` の `:` で module と attribute を分けるのは厳密に 1 つだけ。Windows のパスを写すと `C:\app\graphs\echo_graph.py:make_graph` で `:` が 2 つになって、NAT 側が「Module path に `:` が複数」と怒ります。コンテナ内 Linux パスを使えば普通踏みません。

2 つ目は **graph object の型** です。`make_graph` の戻り値は `CompiledStateGraph` か `StateGraph` でないと、NAT が `Graph definition ... is not a valid graph definition` と止まります。本章のサンプルで `builder.compile()` を呼んでいるのはこのためで、`compile()` を忘れると `StateGraph` のままなので NAT 側で再 compile が走ります（動きはしますが `make_graph` の責務として明示してしまうのが安全）。

3 つ目は **OTLP 送信の network** です。第 2 章の Langfuse スタックに attach し忘れると、NAT 側で `host langfuse-langfuse-web-1 not found` のような名前解決失敗が起きて、trace がドロップします。`docker network ls` に `langfuse_default` がいることを起動前に確認するのが安全です。

## 次章では

LangGraph + NAT の最小実装が動いたところで、次章からは本書の題材である **社内ドキュメント Q&A** に入ります。第 5 章では、ハンズオン用の社内ドキュメントセットを合成し、PII / 機密度メタデータを意図的に仕込みます。題材自体を整える章なので Python はあまり書かず、コーパス設計の考え方と `datasets/internal-docs/` のディレクトリ構造を見ていきます。
