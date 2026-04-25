---
title: "付録 B NGC API key / 無料枠 / cost 試算"
---

本書が前提としている **build.nvidia.com の無料枠で全章を通しで回せる** という試算の根拠を置いておきます。クレジット消費が気になるみなさんは、走らせる前に本章の見積もりを参考にしてください。

## NGC API key の取得手順

1. ブラウザで [build.nvidia.com](https://build.nvidia.com/) を開く
2. 「Sign In」から NVIDIA Developer アカウントでサインイン（初回は無料登録）
3. ナビゲーションのアバターアイコン → API Keys
4. 「Generate API Key」で `nvapi-` から始まる 64 文字程度の key を発行
5. 生成直後の画面でコピーし、**安全な場所に保管**（再表示できない）

NGC API key は本書のすべてのサンプルで共通です。`.env` に 1 行書いて使い回します。

:::message alert
NGC API key が漏れると誰かに無料クレジットを使われるリスクがあります。公開リポにコミットしない、`.gitignore` の `.env` 除外を確認する、の 2 点は必ず守ってください。万が一公開してしまった場合は NGC UI から即 Revoke して再発行できます。
:::

## 無料枠の現状（2026-04 時点）

build.nvidia.com の Developer プログラムには次のような枠があります。金額ベースではなく**リクエスト数ベース**が基本で、本書の範囲では全章を回しても余裕があります（以下は執筆時点の把握、公式ドキュメントで最新を必ず確認）。

| 項目           | 目安                                              |
| -------------- | ------------------------------------------------- |
| 月間リクエスト | 5,000 req 以上（モデルによる）                    |
| レート制限     | 40 req/min 前後（モデルによる）                   |
| モデル一覧     | Llama 3.x / Nemotron 2 Super / Embedding など多数 |
| 有効期限       | アカウント単位で持続、月次リセット                |

## 本書の章別リクエスト見積もり

各章を通しで 1 回ずつ回したときの NIM リクエスト数の見積もりです。workflow LLM が Nemotron Super 49B、judge LLM も Super 49B、embedding が nv-embedqa-e5-v5 の前提です。

| 章       | 内容                                           | LLM 呼び出し数    | Embedding 呼び出し数  | 備考                                  |
| -------- | ---------------------------------------------- | ----------------- | --------------------- | ------------------------------------- |
| 3        | Hello Agent（1 質問、1 tool）                  | 2-3 回            | 0                     | シンプル                              |
| 4        | YAML 実験（温度・モデル差し替え 3 回）         | 10 回前後         | 0                     | 差し替え実験込み                      |
| 5        | 2 ツール ReAct（4 質問）                       | 12-16 回          | 0                     |                                       |
| 6        | 3 パターン比較（ReAct / ReWOO / Tool Calling） | 15-20 回          | 0                     | ReWOO は 2 回固定                     |
| 7        | Phoenix 観測（2-3 クエリ）                     | 6-9 回            | 0                     | 観測検証用                            |
| 8        | MCP serve / client（ping + tool list）         | 2-3 回            | 0                     | tool call は軽量                      |
| 9        | RAG 初回（1 質問）                             | 3-5 回            | **1,034 回**          | **ingest が重い**                     |
| 10       | RAG 3 retriever + カテゴリ実験（4 質問）       | 12-16 回          | 1,034 回（再 ingest） | 同じ docs を再投入                    |
| 11       | Router（3 質問）                               | 3-6 回            | 1,034 回（再 ingest） |                                       |
| 12       | A2A（3 質問）                                  | 6-12 回           | 1,034 回（再 ingest） | agent-main + agent-nat-docs の両方    |
| 13       | nat eval（20 問 × 2 evaluator）                | 60 回             | 1,034 回（再 ingest） | workflow 20 + trajectory 20 + 0 exact |
| 14       | nat serve（curl 3-5 回）                       | 6-15 回           | 0                     |                                       |
| 15       | 完成アプリ（3 質問）                           | 6-10 回           | 1,034 回（再 ingest） | Router → A2A → RAG の連鎖             |
| **合計** |                                                | **約 140-180 回** | **約 6,200 回**       | 2 ヶ月分の作業想定                    |

Embedding の呼び出し回数が多く見えますが、**1,034 chunks を一度に `embed_documents()` で投げる**ため、実際の HTTP リクエスト数は 1 回（バッチ送信）になります。build.nvidia.com 側でも embedding は消費が軽めにカウントされるケースが多いので、全章を通しでも無料枠の数 % で済むはずです。

## 軽量化のコツ

### 反復検証時のポイント

本書の執筆時は各章を何度も回して実測値を取ります。反復時にクレジット消費を抑える工夫：

- **章 9 の ingest は 1 回だけ実行**。Milvus の volume が残っていれば章 10/11/12/13/15 で再 ingest 不要
- **章 13 nat eval は 20 問を maxi 1 回/日にする**。retry で繰り返すと 60 → 120 → 180 と跳ね上がる
- **章 15 完成アプリは curl 数回で止める**。「動いた」確認で切り上げて、本格運用は `/v1/chat/stream` で 1 リクエストずつ
- **Nemotron Mini 4B など軽量モデルに一時差し替え**。検証完了したら Super 49B に戻す

### Milvus volume の永続化運用

`docker compose down` を使えば volume は残るので再 ingest 不要。`docker compose down -v` は絶対に付けないこと（第 10 章）。

```bash
# 安全な停止（再ingest 不要）
docker compose down

# データまで消す（再 ingest が必要になる）
docker compose down -v
```

## もし有料枠に切り替えたいとき

build.nvidia.com は無料枠を超えた場合の有料プラン案内があります。本書のスコープでは不要ですが、業務で使う場合は NVIDIA のエンタープライズ契約（AI Enterprise）を検討する流れが一般的です。

自社 GPU サーバーに NIM を自前デプロイする選択肢もあります（本書では扱いません）。その場合は：

- DGX Spark や H100 などの NVIDIA GPU
- NIM コンテナイメージ（NGC Catalog からダウンロード）
- Kubernetes / Docker でのセルフホスト

レイテンシと cost のコントロールが必要な本番で活きる選択肢です。

## 本書執筆時の実測参考値

著者が DGX Spark 上で本書全章を通し実行した際の累計 NIM 呼び出し数です（本章の「本書の章別リクエスト見積もり」とほぼ一致）。

- workflow LLM（Nemotron Super 49B）: 約 **180 回**
- judge LLM（Nemotron Super 49B）: 約 **40 回**（13 章 + A2A 経由）
- embedding（nv-embedqa-e5-v5）: 約 **5,200 vectors**（6 回ぶん ingest、実 HTTP は 6 回）
- 合計： build.nvidia.com Developer プランの無料枠の **約 5-10 %** 消費

実際の課金ページで見ると、反復検証込みで月額想定で 10-20 % くらいに収まります。余裕を持って本書を消化できる範囲です。

## まとめ

本書は **無料枠のみで全章を完走できる**設計です。反復検証が増えるほど消費は伸びますが、Milvus volume を消さない、nat eval を日 1 回までに絞る、軽量モデルを試し切りに使う、の 3 点を守れば無料枠を使い切ることはまずありません。

それでも心配な場合は、各章の冒頭に書いた「この章のゴール」を基準に「ここまで確認できたら次章へ」という進め方にすれば、消費は最小限で済みます。
