---
title: "第 13 章 nat eval で採点する"
---

ここまでの章では、エージェントが「動くかどうか」を手作業で確認してきました。実運用では「動くだけ」では足りません。**質問に対する回答が期待どおりか、ReAct の軌跡がきれいか、モデルを差し替えたらスコアがどう動くか**を定量的に測れないと、継続的にエージェントを改善していけません。本章では NAT の `nat eval` コマンドを使い、第 9-10 章で組んだ NAT docs RAG エージェントを **20 問のデータセットで採点**します。

本書の題材は「軽量モデル（Nemotron Mini 4B）で ReAct RAG を走らせ、強いモデル（Nemotron Super 49B）で採点する」構成です。評価軸は 2 つに絞ります。`trajectory`（ReAct の軌跡を LLM が採点）と `exact_match`（期待答えとの文字列一致）。これだけでも「弱いモデルはどこでつまずくか」が数字で見えてきます。

## この章のゴール

- `nat eval` の YAML スキーマ（`eval.general.dataset` / `eval.evaluators`）を読めるようになる
- 20 問のファクト Q&A データセットを JSONL 形式で用意する
- `trajectory`（LLM-as-Judge）と `exact_match` の 2 evaluator を走らせる
- 軽量モデル（Mini 4B）被験者 + 強 judge（Super 49B）構成で、スコア差を定量的に見る

## 前章からの引き継ぎ

- 第 9 章の Milvus 構成（etcd + minio + milvus + nat_docs コレクション）
- `nat-nim-handson:1.6.0` イメージ（`nvidia-nat[langchain,mcp,eval,phoenix,a2a]==1.6.0`）
- NGC API key が `.env` にある

## この章で追加する compose service

第 9 章の Milvus スタックに `eval` service を追加します。`nat eval` を実行する使い捨てコンテナで、ingest 専用と同じパターンです。

- `etcd` / `minio` / `milvus` / `ingest` — Milvus スタック（既存）
- `eval` — `nat eval --config_file /app/workflows/workflow.yml` を実行

所要時間は 30-60 分（20 問の ReAct 実行 + 採点、NIM のレイテンシに依存）。

## データセット設計（20 問 × 5 カテゴリ）

`dataset/nat-qa-20.jsonl` にカテゴリ別 4 問ずつの Q&A を置きます。各問題は NAT 公式 docs から正解が引けるファクト系です。

```json:抜粋
{"id": "g01", "question": "Which Python versions does NeMo Agent Toolkit officially support?", "answer": "Python 3.11, 3.12, or 3.13.", "category": "get-started"}
{"id": "b01", "question": "What are the four main top-level sections of a NeMo Agent Toolkit workflow YAML configuration file?", "answer": "general, llms, functions, and workflow.", "category": "build-workflows"}
{"id": "r02", "question": "What default TCP port does the NAT MCP server bind to?", "answer": "9901", "category": "run-workflows"}
{"id": "i02", "question": "Which built-in NAT evaluator type scores the full agent trajectory (thoughts, actions, observations)?", "answer": "trajectory", "category": "improve-workflows"}
{"id": "c01", "question": "Which two retriever types are built into the core nvidia-nat package?", "answer": "milvus_retriever and nemo_retriever.", "category": "components"}
```

カテゴリは第 10 章の Milvus に投入したデータの区分（get-started / build-workflows / run-workflows / improve-workflows / components）と揃えています。章 10 で retriever のカテゴリ絞り込みを練習した流れを引き継いで、本章では「カテゴリごとのスコア差」を後でまとめる切り口にも使えます。

`answer` は短めの正解表現で、`exact_match` で拾える形に書いています。ReAct を経由すると応答が冗長になるので完全一致はほぼ出ませんが、**exact_match が低く出ることで「trajectory で LLM に採点させる評価がどれだけ効いているか」の対比が見えます**。

## workflow.yml の構造（2 モデル同居）

本章の workflow.yml は「被評価 LLM」と「judge LLM」の 2 つを `llms:` トップに並べるのが特徴です。

```yaml:ch13-nat-eval/workflow.yml
llms:
  # 被評価モデル: 軽量な Nemotron Mini 4B（4B、ReAct parse が崩れやすい）
  workflow_llm:
    _type: nim
    model_name: nvidia/nemotron-mini-4b-instruct
    api_key: ${NGC_API_KEY}
    temperature: 0.0
    max_tokens: 512

  # 採点役: 強い Nemotron Super 49B
  judge_llm:
    _type: nim
    model_name: nvidia/llama-3.3-nemotron-super-49b-v1
    api_key: ${NGC_API_KEY}
    temperature: 0.0
    max_tokens: 1024

# ... retrievers / functions は章 9 と同じ ...

workflow:
  _type: react_agent
  llm_name: workflow_llm
  tool_names: [search_nat_docs]
  max_iterations: 6
  parse_agent_response_max_retries: 5

eval:
  general:
    output:
      # NAT は eval 起動時に output.dir を rmtree で掃除する.
      # compose マウントポイント直下だと OSError になるため subdir を指定
      dir: /app/outputs/run
    dataset:
      _type: jsonl
      file_path: /app/dataset/nat-qa-20.jsonl
      structure:
        question_key: question
        answer_key: answer

  evaluators:
    trajectory_eval:
      _type: trajectory
      llm_name: judge_llm

    exact_match:
      _type: langsmith
      evaluator: exact_match
```

抑えておきたい 4 つのポイントです。

1. **`llms` はトップレベル**（`eval.llms` ではない）。workflow 側と judge 側の両方を 1 つの dict に並べ、`workflow.llm_name` と `eval.evaluators.*.llm_name` で参照する
2. **`eval.general.dataset.structure`** で JSONL の `question_key` / `answer_key` を指定。本書は key が `question` / `answer` のストレート形式
3. **`eval.general.output.dir` は rmtree の対象**。compose の volume マウントポイントを直接指定すると Device busy エラーになるので、必ず subdir を指定する
4. **evaluator 2 種類**。`_type: trajectory` は LLM-as-Judge、`_type: langsmith` + `evaluator: exact_match` は LLM 不要の文字列一致

:::message
**YAML スキーマの罠**: `eval` セクションは NAT 1.6.0 でかなり厳格にチェックされます。執筆中の実機検証で次のエラーに順番に遭遇しました（本書は対処済み）。<br/><br/>・`eval.dataset` と書く → `eval.general.dataset` が正解<br/>・`eval.llms` と書く → トップレベル `llms` に移す<br/>・`structure.id_key: id` → NAT 1.6.0 では許容されないので削除<br/>・`output_dir: /app/outputs` → `output.dir: /app/outputs/run` のように subdir を指定<br/><br/>`nat validate` で事前チェックすると、クレジット消費前に発見できます。
:::

## 実機で走らせる

Milvus とデータ投入は第 9 章と同じ手順です。

```bash
cd nemo-agent-toolkit-book/ch13-nat-eval
cp ../ch03-hello-agent/.env .env

# Milvus スタック起動 + NAT docs を Milvus に投入
docker compose up -d milvus
docker compose --profile ingest run --rm ingest

# nat eval 実行（20 問 × 2 evaluator、推定 10-15 分）
docker compose run --rm eval
```

`nat eval` はまず workflow を 20 問に対して順次実行し、続いて各 evaluator で採点します。`--max_concurrency 2` 相当で並列抑制していますが、NIM 無料枠の Rate Limit（40 req/min）に収まる設計です。

実行中のログには次のような流れが出ます。

```text
Running workflow for 20 inputs with max concurrency 2...
[q=g01] Final Answer: Python 3.11, 3.12, or 3.13.
[q=g02] Final Answer: pip install nvidia-nat
[q=b01] ReActAgentParsingFailedError: Missing 'Action Input:' after 'Action:'
...
Running evaluator trajectory_eval...
Running evaluator exact_match...
Writing results to /app/outputs/run/
```

Mini 4B だと parse 失敗が数問は発生するのが通常の挙動で、そのまま採点に進みます（失敗問題は空応答として採点されるので trajectory スコアが下がります）。

## 実測結果（DGX Spark、Nemotron Mini 4B × Super 49B 構成）

20 問 × 2 evaluator を実行した結果のスコアサマリです。読者の手元では NIM のレスポンスや Mini 4B の確率的挙動で多少前後しますが、大きな傾向は同じ方向に出るはずです。

```text
=== EVALUATION SUMMARY ===
Workflow Status: INTERRUPTED (workflow_output.json)
Total Runtime: 7.94s

Per evaluator results:
| Evaluator       |   Avg Score | Output File                 |
|-----------------|-------------|-----------------------------|
| exact_match     |         0   | exact_match_output.json     |
| trajectory_eval |         0.2 | trajectory_eval_output.json |
```

### 衝撃の結果: tool 呼び出しが 0 回

`workflow_output.json` の `intermediate_steps` を数えると、衝撃の事実が見えます。

```bash
jq '[.[] | select(.intermediate_steps | length > 0)] | length' \
  outputs/run/workflow_output.json
# => 0
```

**20 問すべてで tool（`search_nat_docs`）呼び出しが 0 回**。Nemotron Mini 4B は RAG retriever を見ないまま、LLM 自身の知識でいきなり Final Answer を返しています。典型的な失敗例：

```json
{
  "id": "g01",
  "question": "Which Python versions does NeMo Agent Toolkit officially support?",
  "answer": "Python 3.11, 3.12, or 3.13.",
  "generated_answer": "NeMo Agent Toolkit officially supports Python 3.6 and above.",
  "intermediate_steps": []
}
```

正解は 3.11 / 3.12 / 3.13 なのに、Mini 4B は「3.6 and above」と自信を持って答えています（ハルシネーション）。retriever を呼べば docs から `3.11, 3.12, or 3.13` を引けたはずですが、Mini 4B の推論能力では ReAct の `Thought → Action → Observation` ループを構築できていません。

Total Runtime が 7.94 秒と異様に短いのも tool 呼び出しがないからです。20 問 × Final Answer 直出しで平均 0.4 秒/問。

### カテゴリ別スコア

`workflow_output.json` の `category` と `trajectory_eval_output.json` の `score` を join して集計した結果：

| category            | avg trajectory score | 備考                                                |
| ------------------- | -------------------- | --------------------------------------------------- |
| `run-workflows`     | **0.562**            | MCP / A2A は一般知識で近い回答を作れる              |
| `components`        | 0.250                | retriever / embedder の用語を拾える                 |
| `get-started`       | 0.125                | Python バージョン等の細かい数値で外す               |
| `build-workflows`   | 0.062                | YAML の構造問題で総崩れ                             |
| `improve-workflows` | **0.000**            | eval / profiler / optimizer など NAT 固有概念で全滅 |

- **NAT 固有の知識領域（improve-workflows）は 0 点**。Mini 4B は `nat eval` や `tunable_rag_evaluator` といった NAT の独自用語を学習していない
- **一般的な用語が近いほど無理やり当たる傾向**。run-workflows（MCP / A2A）は業界一般用語で高めに出る
- **exact_match は 20 問すべて 0 点**。ReAct Final Answer が常に冗長で、データセットの短い `answer` と一致しない

### 得られた教訓

3 つまとめます。

1. **軽量モデルは tool 呼び出しの判断もできない**。Mini 4B は「docs を引こう」という Thought すら組み立てず、LLM 知識で直答してしまう。RAG エージェントとしては実用にならない
2. **exact_match は ReAct では使いづらい**。応答の冗長さで全滅するので、期待値を「生成文の部分一致」「キーワード包含」などに書き換える必要がある。本書では「ReAct の適性を見るなら trajectory 系が向く」という対照にあえて残している
3. **カテゴリ別分析で弱点が特定できる**。improve-workflows の 0 点は、RAG を強制する `router_agent` や tool description の工夫が必要、という改善の方向性を示す

## `outputs/run/` の中身を読む

## `outputs/run/` の中身を読む

`nat eval` が吐き出すファイルを覗いてみます。

```bash
ls ch13-nat-eval/outputs/run/
# all_requests_profiler_traces.json
# eval_output.json
# workflow_output.json
# trajectory_eval_output.json
# exact_match_output.json
```

特に重要なのは次の 2 つです。

- `workflow_output.json` — 20 問の入力・出力・中間 Thought / Action のログ
- `<evaluator_name>_output.json` — 各 evaluator のスコア詳細（問題 ID ごとに 0.0〜1.0 のスコア + reasoning）

`jq` でざっと眺めるコマンドの例：

```bash
# 各問題のスコアを取り出す
jq '.results[] | {id, score: .eval_output_items[0].score}' \
  ch13-nat-eval/outputs/run/trajectory_eval_output.json

# スコアが 0.5 未満の「弱い問題」を抽出
jq '.results[] | select(.eval_output_items[0].score < 0.5) | .id' \
  ch13-nat-eval/outputs/run/trajectory_eval_output.json
```

自動化するなら、上記 JSON を pandas に読ませて `score_<0.5` の問題を Issue に落とすような CI パイプラインを組めます。

## カテゴリ別にスコアを眺めると

データセットには `category` フィールドを持たせているので、カテゴリごとの集計が取りやすいです。evaluator 出力には `category` は載りませんが、データセット側と `id` で join することで「このカテゴリは弱いな」が見えてきます。

```python
# 擬似コード
import json
import pandas as pd

ds = pd.read_json("dataset/nat-qa-20.jsonl", lines=True)
traj = json.load(open("outputs/run/trajectory_eval_output.json"))
scores = pd.DataFrame(
    [{"id": r["id"], "score": r["eval_output_items"][0]["score"]} for r in traj["results"]]
)
merged = ds.merge(scores, on="id")
print(merged.groupby("category")["score"].mean().sort_values())
```

前掲の実測結果（run-workflows 0.562 / components 0.250 / get-started 0.125 / build-workflows 0.062 / improve-workflows 0.000）のように、Mini 4B では NAT 固有の知識領域ほどスコアが下がります。改善するには workflow_llm を強いモデルに差し替えるか、system_prompt で「必ず retriever を呼んでから答えて」と指示するのが効きます。

## モデル差し替え実験（次の一手）

章 4 の「モデルを差し替える」実験を思い出してください。本章の workflow.yml で `workflow_llm.model_name` を **`nvidia/llama-3.3-nemotron-super-49b-v1`**（本書デフォルト）に差し替え、同じ 20 問を走らせてみると、trajectory スコアがぐっと上がるはずです。

```diff yaml:workflow.yml
 llms:
   workflow_llm:
     _type: nim
-    model_name: nvidia/nemotron-mini-4b-instruct
+    model_name: nvidia/llama-3.3-nemotron-super-49b-v1
```

「モデルを強くすればスコアは上がる」は直感的にも当然ですが、`nat eval` で**どれだけ上がるかを定量化できる**のが評価駆動開発の醍醐味です。NAT2 記事では workflow を Nemotron 9B-JP から Nemotron Super 49B に差し替えて、parse 失敗率が 22% → 0% に改善した事例を紹介しました。本書でも同じ構図を実測で追えます。

:::message
**NIM クレジット消費の目安**: 20 問 × workflow 呼び出し（Mini 4B、4 LLM call / 問）+ trajectory 採点（Super 49B、1 LLM call / 問）= 約 **100 リクエスト**。無料枠 5,000 req/月に対しても軽微です。ただし workflow を Super 49B にすると 80 LLM call 追加で消費が 2 倍近くなるので、反復実験時は `max_iterations` の制限で暴走を防ぐのが無難です。
:::

## よくある詰まりどころ

**`Input tag 'ragas' found ... does not match any of the expected tags`**

NAT 1.6.0 の `ragas` evaluator は `nvidia-nat[ragas]` extras 追加インストールが必要です。本書では `trajectory` と `langsmith` exact_match に絞って回避しています。ragas を使いたい場合はベースイメージに extras を追加して再ビルドしてください。

**`OSError: [Errno 16] Device or resource busy: '/app/outputs'`**

`output.dir` を compose マウントポイント直下にすると、NAT の `rmtree` でマウントポイント自身を削除しようとして失敗します。必ず subdir（`/app/outputs/run` など）を指定します。

**`tunable_rag_evaluator` の `judge_llm_prompt` missing**

この evaluator は `judge_llm_prompt` と `default_score_weights` が必須です。本書ではシンプル運用のため使わず、`trajectory` と `exact_match` に絞っています。

**Mini 4B の parse 失敗が多すぎる**

`parse_agent_response_max_retries: 5` を入れても救いきれない場合は、workflow_llm を Super 49B に差し替えて再実行してください。CI で毎回走らせる評価なら Mini 4B、開発中の手動確認なら Super 49B、と使い分けるのが現実的です。

## ここまでで動くもの

- `ch13-nat-eval/` で 20 問 × 2 evaluator の `nat eval` が完走する
- `trajectory_eval_output.json` と `exact_match_output.json` でスコアを読める
- 軽量モデル Mini 4B の「parse が崩れやすい」「冗長応答で exact_match は全滅」という特性が数字で見える
- workflow_llm を強いモデルに差し替えれば trajectory スコアが上がる、という改善サイクルの土台ができた

:::message
本章のサンプルコードは [nemo-agent-toolkit-book リポ](https://github.com/himorishige/nemo-agent-toolkit-book) の `ch13-nat-eval/` ディレクトリにまとめています。
:::

## 次章では

次章では `nat serve` を使い、ここまでに作ったワークフローを **FastAPI として HTTP 公開**します。curl / httpx クライアントから叩ける形にして、既存の Web アプリや業務ツールからエージェントを呼べるようにするのが第 14 章のゴールです。
