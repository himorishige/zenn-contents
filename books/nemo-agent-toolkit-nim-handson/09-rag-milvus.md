---
title: "第 9 章 RAG アドオン：Milvus に NAT docs を入れる"
---

本書後半のハイライトである RAG を組み立てます。題材は NVIDIA/NeMo-Agent-Toolkit 公式リポジトリの `docs/source/` 配下（Apache 2.0 ライセンスで配布）から 24 ファイルを抽出し、Milvus ベクトルデータベースに投入、NAT の ReAct エージェントから引ける形にします。

「NAT の docs を NAT 自身で RAG するエージェント」という自己言及的な構成ですが、docs を仕込む部分で NIM Embedding API / Milvus / LangChain splitter を実機で触り、出来上がったエージェントの会話で「retriever が戻したチャンクが Thought にどう効くか」を目視できる、良い題材になっています。

## この章のゴール

- `ch09-rag-milvus/` で etcd + minio + Milvus standalone の docker compose が立ち上がる
- Python の ingest スクリプトで 1,000 件強のチャンクを Milvus に投入できる
- `workflow.yml` の `embedders` / `retrievers` セクションと `_type: nat_retriever` を使って、retriever をツール化する書き方が身につく
- ReAct エージェントが RAG 経由で公式 docs の文面をそのまま参照する動作を目撃する

## 前章からの引き継ぎ

- `nat-nim-handson:1.6.0` イメージに `pymilvus[milvus_lite]` と `setuptools<80` を加えた構成でビルド済み
- NGC API key が `.env` にある
- `../datasets/nat-docs/` に NAT 公式 docs 24 ファイルが配置済み（Apache 2.0、同梱の `NOTICE.md` / `LICENSE` で出典を明示）

本章の `workflow.yml` は埋め込み機能を使うので、ベースイメージに `pymilvus` と `langchain-nvidia-ai-endpoints` が入っている前提です。本書の Dockerfile ではこれらを NAT 1.6 の extras 経由で最初から取り込んでいます。

## この章で追加する compose service

- `etcd` — Milvus のメタデータストア
- `minio` — Milvus のオブジェクトストレージ
- `milvus` — Milvus standalone 本体（2.5.4、ARM64 対応）
- `ingest` — `python /app/scripts/ingest.py` をワンショットで実行する使い捨て service（profile `ingest`）
- `nat` — 既存の NAT ReAct ワークフロー実行

Milvus Lite（ファイル DB モード）の採用も検討しましたが、NAT 1.6.0 の `milvus_retriever` の `uri` フィールドが `HttpUrl` 型で validate される実装のため、実測ではファイルパスを受け付けませんでした。そのため本書では 3 コンテナ構成の standalone 版に揃えます。章 10 ではこの構成を前提に運用面（永続化・メタデータ・top_k チューニング）へ進みます。

所要時間は 30-45 分（初回の Milvus イメージ pull を含めて）。

## データソースと Apache 2.0 の扱い

サンプルリポの `datasets/nat-docs/` には、NVIDIA/NeMo-Agent-Toolkit リポジトリの `docs/source/` 配下から次のカテゴリを抽出しています（2026-04 時点）。

- `get-started/` — インストール、クイックスタート
- `build-workflows/` — YAML 構成、メモリ、オブジェクトストア
- `run-workflows/` — MCP / A2A / UI
- `improve-workflows/` — 評価、profiler、optimizer
- `components/` — 各コンポーネント共有

合計 24 ファイル、約 7,100 行、200KB ほどです。サンプルリポに同梱した `LICENSE`（Apache 2.0 原本）と `NOTICE.md`（出典と取得コマンド）で著作権と再配布条件を明示しています。本書のサンプル全体が Apache 2.0 なので整合が取れています。

## docker-compose.yml を読む

全体像を見ます（冗長になるので抜粋）。

```yaml:ch09-rag-milvus/docker-compose.yml
services:
  etcd:
    image: quay.io/coreos/etcd:v3.5.16
    # メタデータストア。healthcheck で起動完了を待てる
    ...

  minio:
    image: minio/minio:RELEASE.2024-11-07T00-52-20Z
    # Milvus が LOG/index を置くオブジェクトストレージ
    ...

  milvus:
    image: milvusdb/milvus:v2.5.4
    command: ['milvus', 'run', 'standalone']
    environment:
      ETCD_ENDPOINTS: etcd:2379
      MINIO_ADDRESS: minio:9000
    ports:
      - '19530:19530'
    depends_on:
      etcd:
        condition: service_healthy
      minio:
        condition: service_healthy

  ingest:
    <<: *nat-common
    profiles: ['ingest']
    entrypoint: ['python', '/app/scripts/ingest.py']
    depends_on:
      - milvus

  nat:
    <<: *nat-common
    volumes:
      - ./workflow.yml:/app/workflows/workflow.yml:ro
    command: ['run', '--config_file', '/app/workflows/workflow.yml', '--input', '...']
    depends_on:
      - milvus
```

押さえるべき点は 3 つです。

- `milvus` は `depends_on.condition: service_healthy` で etcd/minio の healthy を待つ。これを忘れると milvus が起動時に etcd の未準備を検出して落ちるので要注意
- `ingest` service は compose profile `ingest` を付けてあるので、通常の `docker compose up` では起動しない。投入時だけ `docker compose --profile ingest run --rm ingest` で明示的に動かす
- ボリュームは `etcd-data` / `minio-data` / `milvus-data` の 3 つ。章 10 で永続化の話をするときに `docker compose down -v` の話と絡めて触れる

## ingest.py を読む

Python コードは `scripts/ingest.py` の 1 本だけです。本書で最初に登場する Python ファイルなので、少し丁寧に説明します。

```python:scripts/ingest.py
from langchain_community.document_loaders import TextLoader
from langchain_nvidia_ai_endpoints import NVIDIAEmbeddings
from langchain_text_splitters import RecursiveCharacterTextSplitter
from pymilvus import DataType, MilvusClient

DOCS_DIR = Path("/app/datasets/nat-docs")
MILVUS_URI = os.environ.get("MILVUS_URI", "http://milvus:19530")
COLLECTION = "nat_docs"
EMBED_MODEL = "nvidia/nv-embedqa-e5-v5"
EMBED_DIM = 1024
CHUNK_SIZE = 500
CHUNK_OVERLAP = 100
```

処理は 4 ステップです。

1. `load_documents()` — `datasets/nat-docs/*.md` を `TextLoader` で読み込み、ファイル名を metadata に詰める
2. `split_documents()` — `RecursiveCharacterTextSplitter` で chunk 500 / overlap 100、Markdown の `##` や改行を優先的な区切りに指定
3. `embed_chunks()` — `NVIDIAEmbeddings` で NIM の `nv-embedqa-e5-v5` を呼ぶ（1024 次元）
4. `write_to_milvus()` — スキーマを定義してコレクション作成、`AUTOINDEX` + `L2` で index 張って insert

特に重要なのは **metric type の選択**です。NAT 1.6.0 の `milvus_retriever` は内部で `L2` 距離をデフォルトに検索する実装になっているため、ingest 側も `metric_type="L2"` で index を張る必要があります。ここを `"COSINE"` にすると、あとで検索時に `metric type not match` エラーが出ます。

:::message alert
**metric 不一致のハマり**: Milvus は index 側と検索側の距離関数が一致している必要があります。NAT 1.6.0 の `milvus_retriever` は L2 を既定にしているので、ingest の `metric_type` も `L2` に合わせます。COSINE が好みの場合は workflow.yml の `search_params` を明示的に上書きする必要があります（本書では扱いません）。
:::

## workflow.yml を読む

ここからが NAT native の RAG 構成のキモです。

```yaml:ch09-rag-milvus/workflow.yml
embedders:
  nim_embed:
    _type: nim
    model_name: nvidia/nv-embedqa-e5-v5
    api_key: ${NGC_API_KEY}

retrievers:
  nat_docs_retriever:
    _type: milvus_retriever
    uri: http://milvus:19530
    collection_name: nat_docs
    embedding_model: nim_embed
    content_field: text
    top_k: 3
    description: >
      Search the official NVIDIA NeMo Agent Toolkit documentation.

functions:
  search_nat_docs:
    _type: nat_retriever
    retriever: nat_docs_retriever
    topic: 'NeMo Agent Toolkit 公式ドキュメント'

workflow:
  _type: react_agent
  tool_names:
    - search_nat_docs
  llm_name: nim_llm
  verbose: true
  max_iterations: 6
  parse_agent_response_max_retries: 5
```

4 つのポイントを確認します。

1. **`embedders` セクションが新登場**。本書で初めて埋め込みモデルを宣言しています。`_type: nim` は LLM と同じドライバで、`model_name` に NIM 側の embedding モデル名を指定するだけ
2. **`retrievers` セクション**で Milvus とのつなぎを定義。`embedding_model: nim_embed` で上段の embedder を参照し、検索時のクエリ埋め込みに使います。`top_k: 3` はトレードオフ（多いほど情報量 ↑ / コンテキスト長 ↑ / 雑音 ↑）
3. **`functions.search_nat_docs` の `_type: nat_retriever`** が retriever を ReAct tool に変換する wrapper。`topic` は ReAct の tool description に使われるので、LLM がいつこの tool を選ぶか判断する材料になります
4. **`parse_agent_response_max_retries: 5`** を保険として付けています。本書のデフォルトモデル Nemotron Super 49B は ReAct のフォーマット遵守が安定しており通常は retry 不要ですが、RAG 経由で長い Observation を受けた際にごく稀に崩れることがあるため、NAT2 記事の知見を踏襲して保険値で入れています

## 動かす

準備できたら、3 ステップで RAG が動きます。

```bash
cd nemo-agent-toolkit-book/ch09-rag-milvus
cp ../ch03-hello-agent/.env .env

# Step 1: Milvus スタックを起動（初回は 2-3 分、イメージ pull 含む）
docker compose up -d milvus
docker compose ps

# Step 2: 1,000 件強のチャンクを Milvus に投入
docker compose --profile ingest run --rm ingest
```

ingest の出力例（実測、2026-04-24）:

```text
Loading 24 markdown files from /app/datasets/nat-docs
Split into 1034 chunks (size=500, overlap=100)
Embedded 1034 chunks with nvidia/nv-embedqa-e5-v5
Inserted 1034 records into collection 'nat_docs' at http://milvus:19530
Done.
```

```bash
# Step 3: RAG エージェントを叩く
docker compose run --rm nat
```

出力の核心部分（抜粋）:

```text
[AGENT]
Agent input: What is the NeMo Agent Toolkit, and how do I define a workflow YAML?
Agent's thoughts:
Thought: I need to recall the information about the NeMo Agent Toolkit and its workflow YAML definition.
Action: search_nat_docs
Action Input: {'query': 'NeMo Agent Toolkit workflow YAML definition'}

[AGENT]
Calling tools: search_nat_docs
Tool's response:
{"results": [{"page_content": "# Workflow Configuration\n\nNeMo Agent Toolkit workflows are defined by a YAML configuration file...",
              "metadata": {"source": "workflow-configuration.md", "distance": 0.76}},
             ...]}

[AGENT]
Final Answer: The NeMo Agent Toolkit is a workflow management system that uses a YAML configuration file to define the workflow. The YAML configuration file specifies which entities (functions, LLMs, embedders, etc.) to use in the workflow, along with general configuration settings...
```

ReAct が `search_nat_docs` を 1 回呼び、Milvus からトップ 3 件のチャンクが戻り、`workflow-configuration.md` の実際の文面を根拠に Final Answer が生成されました。これまでの章との違いは、Observation に docs の原文がそのまま乗ってくる点です。

## 違う質問を試す

サンプルリポの docker-compose.yml の `--input` を書き換えるか、`run` コマンドで入力を上書きできます。

```bash
docker compose run --rm nat \
  run --config_file /app/workflows/workflow.yml \
  --input 'What is the difference between retrievers and function groups in NAT?'
```

RAG が効いていれば、NAT の components 章のチャンクが引き出され、正確な違いを答えてくれます。反対に、docs にない話題（たとえば「What is Next.js?」）を聞くと、ReAct が tool を呼んでも低スコアな結果しか戻らず、LLM がその旨に言及する応答を返します。RAG の境界が体感できるケースなので試してみてください。

## よくある詰まりどころ

**`metric type not match: expected=L2, actual=COSINE`**

ingest の `metric_type` と workflow.yml 側の検索設定が食い違っています。本章では `L2` で統一。

**`fail to connect to milvus:19530`**

`docker compose up -d milvus` が終わる前に ingest を走らせた可能性。`docker compose ps` で milvus が `Up` になってから実行します。初回起動は 30 秒ほどかかります。

**ReAct の `Failed to parse agent output`**

本書のデフォルトの Nemotron Super 49B では通常発生しませんが、過負荷の NIM エンドポイントや特殊な長文 Observation で稀に起こります。`parse_agent_response_max_retries: 5` を設定しておけば大半を救えます。軽量な Nemotron Mini 4B / Meta Llama 3.1 8B 等に差し替えた場合は発生頻度が上がるので、差し替え時は注意してください。

**`setuptools` 関連のビルドエラー**

`milvus_lite` パッケージ本体は `pkg_resources` を要求します。`setuptools` の最近のバージョンでは `pkg_resources` が同梱されないため、本書では `setuptools<80` にピン止めしています（`docker/nat/requirements.txt`）。

## Apache 2.0 のライセンス表示

本章のサンプルは Apache 2.0 で配布されている NAT 公式 docs を含んでいます。Apache 2.0 は「NOTICE ファイルがあればそれを引用する」「変更した場合はその旨を示す」などの条件で再配布可能です。本書の `datasets/nat-docs/LICENSE` と `NOTICE.md` を維持したまま clone・利用する限り、読者側で追加の手続きは不要です。自分の業務ドキュメントを RAG に入れたい場合も、同じ感覚でライセンスと変更履歴を `NOTICE.md` に残しておくと、後で振り返りやすくなります。

## ここまでで動くもの

- Milvus standalone（etcd + minio + milvus）を docker compose で立てられる
- Python ingest スクリプトで NAT docs を 1,034 chunks まで分解し、NIM Embedding でベクトル化して Milvus に入れられる
- NAT native の `milvus_retriever` + `nat_retriever` を ReAct から使い、公式 docs を根拠にした応答を得られる
- metric type / parse retry など、実運用で必ず踏む 2 つのハマりどころを回避済み

:::message
本章のサンプルコードは [nemo-agent-toolkit-book リポ](https://github.com/himorishige/nemo-agent-toolkit-book) の `ch09-rag-milvus/` ディレクトリにまとめています。
:::

## 次章では

次章では本章の Milvus を前提に、永続化の運用（`docker compose down -v` と volume の扱い）、メタデータフィルタリング（特定の `source` だけを検索対象にする）、`top_k` のチューニング、そして本書の完成アプリに向けた「NAT docs retriever をサブエージェントに格上げ」する準備を進めます。
