---
title: "第 10 章 RAG 運用：永続化・メタデータフィルタ・top_k チューニング"
---

前章で動いた RAG エージェントは「Milvus 全体検索・top_k=3」という最小構成でした。実運用に乗せようとすると、「永続化したつもりが `docker compose down -v` で全部消えた」「特定カテゴリだけ検索したい」「top_k が少なすぎて文脈が足りない」といった課題が出てきます。本章ではこの 3 つを実機で試し、運用に必要な型を身につけます。

## この章のゴール

- `docker compose down` と `docker compose down -v` の違いを押さえ、Milvus データを残すコマンドを使い分けられる
- 3 種類の retriever を同時に並べて、ReAct が用途に応じてツールを選び分ける挙動を観察する
- `search_params.filter` で `category == 'get-started'` のようなメタデータ絞り込みをかける
- `top_k` を増やした retriever を用意し、レイテンシとコンテキスト量のトレードオフを確かめる

## 前章からの引き継ぎ

- Milvus standalone + etcd + minio の docker compose 構成
- `nat-docs` コレクションに 1,034 chunks（章 9 で投入済み、あるいは本章で再投入する）
- `nat-nim-handson:1.6.0` イメージ
- Apache 2.0 の NAT docs（`../datasets/nat-docs/`）

## この章で追加する compose service

なし。第 9 章の docker-compose.yml とほぼ同じ構成を `ch10-rag-operations/` に置いて、ingest スクリプトと workflow だけ差し替えます。project 名が別になる（デフォルトはディレクトリ名）ので、第 9 章の volume とは独立したデータが作られます。同じスタックを別コピーで動かすことになりますが、章ごとに検証を切り分けられる利点があります。

所要時間は 30-45 分。

## 永続化の落とし穴

Milvus の永続データは `milvus-data` / `etcd-data` / `minio-data` の 3 volume に分かれて保存されます。docker compose の 2 種類の停止コマンドを押さえておきます。

- `docker compose down` — コンテナとネットワークを削除し、volume は保持する
- `docker compose down -v` — コンテナ / ネットワークに加えて **volume も削除**する

前章の `docker compose down` の後でも、`docker compose up -d milvus` を再実行すれば前回の ingest 結果がそのまま使えます。一方で `-v` を付けた場合は Milvus の中身がまっさらになるので、ingest からやり直しになります。本書を複数日にわたって進める場合、間違えて `-v` を打つと数分の ingest 時間を毎回失うことになるため、コマンド履歴の先頭に貼り付けない運用にするのが現実的です。

:::message alert
`docker compose down -v` は「本章で作ったデータすべてを消す」意味になります。別章と volume 名が衝突している場合は他章のデータも巻き添えになりうるので、本書の各章はディレクトリを分けて compose project を独立させています。
:::

## カテゴリ metadata を付けて再 ingest する

第 9 章の `ingest.py` は `source`（ファイル名）しか metadata に入れていませんでした。本章ではカテゴリ（`get-started` / `build-workflows` / `run-workflows` / `improve-workflows` / `components`）を追加して、あとで filter クエリを書けるようにします。

```python:scripts/ingest_with_category.py
CATEGORY_MAP = {
    "installation": "get-started",
    "quick-start": "get-started",
    "workflow-configuration": "build-workflows",
    "mcp-server": "run-workflows",
    "evaluate": "improve-workflows",
    "retrievers": "components",
    # ... 全 24 ファイル分のマッピング
}

schema.add_field("category", DataType.VARCHAR, max_length=64)
```

Milvus のスキーマに `category` フィールドを追加し、ingest 時にファイル名からカテゴリを引いて詰める形です。読者が自分のドキュメントに置き換えるときも、フォルダ構造や YAML の front matter からカテゴリを推定する発想が流用できます。

実機で再 ingest すると次のようなカテゴリ別カウントが返ります。

```text
Loading 24 markdown files from /app/datasets/nat-docs
Split into 1034 chunks (size=500, overlap=100)
Embedded 1034 chunks with nvidia/nv-embedqa-e5-v5
Inserted 1034 records into 'nat_docs' at http://milvus:19530
Per-category chunk counts:
  build-workflows: 89
  components: 76
  get-started: 116
  improve-workflows: 515
  run-workflows: 238
```

カテゴリ別の偏り（`improve-workflows` が約半分を占める）を見るだけでも、コレクションの中身を把握できます。

## 3 つの retriever を同時に並べる

本章の `workflow.yml` では、同じ `nat_docs` コレクションを 3 種類の retriever として読ませます。

```yaml:ch10-rag-operations/workflow.yml
retrievers:
  # 1. 全体検索（章 9 と同じ top_k=3）
  nat_docs_retriever:
    _type: milvus_retriever
    uri: http://milvus:19530
    collection_name: nat_docs
    embedding_model: nim_embed
    content_field: text
    top_k: 3

  # 2. get-started カテゴリだけに絞った retriever
  nat_getstarted_retriever:
    _type: milvus_retriever
    uri: http://milvus:19530
    collection_name: nat_docs
    embedding_model: nim_embed
    content_field: text
    top_k: 3
    search_params:
      filter: "category == 'get-started'"

  # 3. top_k を増やした retriever（情報量重視）
  nat_docs_wide_retriever:
    _type: milvus_retriever
    uri: http://milvus:19530
    collection_name: nat_docs
    embedding_model: nim_embed
    content_field: text
    top_k: 8

functions:
  search_nat_docs:
    _type: nat_retriever
    retriever: nat_docs_retriever
    topic: 'NeMo Agent Toolkit 公式ドキュメント（全体）'

  search_getstarted:
    _type: nat_retriever
    retriever: nat_getstarted_retriever
    topic: 'NAT の入門ドキュメント（get-started カテゴリのみ）'

  search_wide:
    _type: nat_retriever
    retriever: nat_docs_wide_retriever
    topic: 'NAT 公式ドキュメント（top_k=8、調査用）'

workflow:
  _type: react_agent
  tool_names:
    - search_nat_docs
    - search_getstarted
    - search_wide
  llm_name: nim_llm
  verbose: true
  max_iterations: 8
  parse_agent_response_max_retries: 5
```

抑えておくポイントは 4 つです。

1. 同じ collection を別の retriever として宣言できる。`uri` / `collection_name` / `embedding_model` は共通、差分だけ違う形に書ける
2. `search_params.filter` に Milvus 式の expression を書くと、index 検索のあとにフィルタが効く。`==` / `in` / `>=` / `and` / `or` 等が使える
3. `top_k` はレスポンスサイズに直結する。多いほど情報量は増えるが、LLM に渡すコンテキストが長くなりレイテンシが悪化する
4. ReAct の `tool_names` に 3 つ並べれば、ReAct は「質問に応じてどの retriever を呼ぶか」を LLM が判断する

## ReAct がカテゴリフィルタを選ぶ様子

本章の既定の入力は「How do I install the NeMo Agent Toolkit for the first time?」。ReAct が `search_getstarted`（get-started カテゴリフィルタ）を選ぶことを狙った質問です。実機で走らせると次のような軌跡になります。

```text
[AGENT]
Agent input: How do I install the NeMo Agent Toolkit for the first time?
Agent's thoughts:
Thought: I need to know how to install the NeMo Agent Toolkit for the first time
Action: search_getstarted
Action Input: {'query': 'install NeMo Agent Toolkit for the first time'}

[AGENT]
Calling tools: search_getstarted
Tool's response:
{"results": [
  {"page_content": "## 🚀 Installation\n\nBefore you begin using NeMo Agent Toolkit, ...",
   "metadata": {"source": "nat-readme.md", "category": "get-started", "distance": 0.806}},
   ...
]}

[AGENT]
Thought: I now know how to install the NeMo Agent Toolkit for the first time
Final Answer: To install the NeMo Agent Toolkit for the first time, you need to have
Python 3.11, 3.12, or 3.13 installed on your system. You can install the latest stable
version from PyPI with `uv sync`. ...
```

注目してほしいのは、返ってきたチャンクの `metadata.category` がすべて `"get-started"` になっている点です。全体検索（`search_nat_docs`）なら `improve-workflows` の長い実験系 docs が紛れ込む可能性があるところ、フィルタで入門向けの文書だけに絞れました。

質問を別のものに変えると、ReAct が別の tool を選びます。

- 「How do I evaluate my workflow?」→ `search_nat_docs`（全体から拾う）
- 「Give me a deep overview of how profiler and optimizer work together.」→ `search_wide`（top_k=8 で情報量を稼ぐ）

実際には LLM の気まぐれで期待と違うツールを選ぶこともあります。tool description（本書では `topic:` で設定）を細かく書き分けるほど選択精度が上がる、という地味な積み重ねが RAG の仕上げ工程の中心になります。

## top_k を変えるとどう変わるか

`top_k` は「いくつチャンクを返すか」のパラメータです。3 つの retriever を同じ質問で走らせると、だいたいこんな傾向が見えます。

| retriever           | top_k | 応答トークン数の目安 | 体感レイテンシ   | 使い分け                           |
| ------------------- | ----- | -------------------- | ---------------- | ---------------------------------- |
| `search_nat_docs`   | 3     | ~600-900 tokens      | 速い（7-10 秒）  | 通常の質問応答、初期値             |
| `search_getstarted` | 3     | ~600-900 tokens      | 速い（7-10 秒）  | 入門系・カテゴリが明確な質問       |
| `search_wide`       | 8     | ~1,800-2,500 tokens  | 遅い（12-18 秒） | 調査系・複数情報源を組合せたいとき |

`top_k=8` まで増やすと、ReAct の Observation に 8 個のチャンクが並び、LLM が読む入力が長くなります。max_tokens の枠を圧迫しない範囲で、かつ関連性の低いチャンクで LLM を惑わさない範囲での絶妙な値を探すことになります。本書のモデル（Llama 3.1 8B、max_tokens 512）なら **基本は 3、調査用に 8**、という構え方が無難です。

## ドキュメントを差し替えて再 ingest する

本書では NAT docs を 24 ファイル固定で扱っていますが、実運用では「docs が週次で更新されるので embedding を取り直したい」といったケースが発生します。本書の ingest スクリプトは **既存コレクションを drop してから作り直す**設計なので、毎回まっさらな状態から embedding を引き直します。

```python:scripts/ingest_with_category.py（再掲）
if client.has_collection(COLLECTION):
    client.drop_collection(COLLECTION)
```

本格運用では「差分だけ追加する」「バージョン番号を metadata に入れて古い版を残す」といった工夫が必要ですが、本書では**シンプルに毎回上書き**する運用で進めます。embedding 呼び出しは 1,000 チャンクで 10-15 秒、NIM 無料枠では 1-2 リクエスト分なので、更新頻度が日次くらいまでならこの運用で十分です。

## よくある詰まりどころ

**フィルタを追加したら結果が 0 件になる**

`filter` の文字列の書き方は Milvus 式です。`category == 'get-started'` のようにシングルクォートで値を括る必要があります。ダブルクォートは YAML との相性が悪く、ハマりやすい罠です。

**ReAct が期待した tool を選んでくれない**

`topic:` の文言が不明瞭なことが多いです。「〜のみ」「〜専用」のような排他的な言い回しや、「入門」「評価」「運用」のような役割を示す単語を入れると、LLM の選択精度が上がります。実機でトライアンドエラーする前提で書き直していきます。

**`docker compose down` したら再 ingest が必要になった**

`-v` を付けて volume を消したか、別 project（別ディレクトリ）で compose を起動し直した可能性があります。本書の章 9 と章 10 はディレクトリが分かれているので、**同じデータを 2 回 ingest する必要がある**構成になっています。自分のアプリでは 1 つの compose project に集約するのが自然です。

**volume が意図せず溜まっていく**

`docker volume ls` で昔の検証の volume が残っていないか確認します。ディスクが逼迫してきたら `docker volume prune` で未使用 volume を掃除できます（実行前にダイアログで確認が入ります）。

## ここまでで動くもの

- 同じ Milvus コレクションを 3 種類の retriever として使い、ReAct が用途別に選べる
- `search_params.filter` で metadata 絞り込み検索が動く
- `top_k` の違いがレイテンシとコンテキスト量にどう効くか体感できた
- `docker compose down` と `down -v` の違い、再 ingest のコストを把握した

:::message
本章のサンプルコードは [nemo-agent-toolkit-book リポ](https://github.com/himorishige/nemo-agent-toolkit-book) の `ch10-rag-operations/` ディレクトリにまとめています。
:::

## 次章では

第 11 章ではマルチエージェントに踏み込みます。本章までに作った「NAT docs RAG」と、第 5 章の `wikipedia_search` を別エージェントとして分離し、Router に質問を振り分けさせます。`_type: agent_tool_wrapper` で既存のエージェントを別エージェントの tool として包む、NAT ならではのマルチエージェント構成を組み上げる章です。
