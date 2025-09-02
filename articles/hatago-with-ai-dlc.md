---
title: 'Hatago MCP Hub を AI-DLC の考え方で開発してみた'
emoji: '🏮'
type: 'idea' # tech: 技術記事 / idea: アイデア
topics: ['mcp', 'ai', 'claudecode', 'codex']
published: false
---

## はじめに

梅雨の終わり頃、チームで共用できるリモート MCP サーバー環境を整えようと仕様を調べていたら、ふと「これ、まとめ役があればもっと楽では？」と思い立ち、軽いノリで作り始めたのが **Hatago MCP Hub** です。  
複数の MCP サーバーをひとまとめにし、Claude Code / Windsurf / Cursor などの AI クライアントから、ひとつの設定で横断利用できるようにする、そんな狙いのツールです。

https://zenn.dev/himorishige/articles/introduce-hatago-mcp-hub

## AI コーディングエージェントと協働する際の課題

これまで様々なアプローチで AI コーディングエージェントを使ってきましたが、How まで仕様ファーストでガチガチに決めて始めると、現時点の生成 AI では「仕様を守ろうとして暴走したり、省略が増えたり」しがちだと感じています。手戻りも多く、完成の 70〜80% で足踏みしてしまうこともしばしばです。

そこで、AI が提示した How の理由を手がかりに、5W（とくに What / Why）で人が舵を取るスタイルが良さそうだと考えました。これは AI-DLC（AI-Driven Development Lifecycle）の考え方にも近く、相性が良いはずです。

今回はこのタイミングに合わせて、**AI-DLC（AI-Driven Development Lifecycle）** に近い開発スタイルを試してみることにしました。

## AI-DLC（AI-Driven Development Lifecycle）

AI に「How（どうやるか）」を任せ、人間が「What（何をするのか）」「Why（なぜやるのか）」で方向を決める。AI と人の役割を分けながら短いサイクルで繰り返します。この流れは、AI-DLC の **Inception → Construction → Operations** という 3 つのフェーズそのものです。

![AI-DLC](/images/hatago-with-ai-dlc/aidlc-image03.png)
*出典：[AI 駆動開発ライフサイクル:ソフトウェアエンジニアリングの再構築/Amazon Web Services ブログ](https://aws.amazon.com/jp/blogs/news/ai-driven-development-life-cycle/)*

以下では、Hatago MCP Hub の開発をこのフレームワークに当てはめて振り返ってみたいと思います。

### Inception（意図の分解）

最初の段階では「そもそもどんな MCP Hub を作るのか」を明確にします。

まず Claude Code（Opus 4.1）に「この機能を作りたい」「こういうユースケースで使う」といった背景や目的を渡し、大まかなプランを出してもらいます。Claude は自由に発想を膨らませ、実装戦略を提示してくれますが、そのままでは機能過多で複雑になりがちです。

そこで MCP 経由で GPT-5 に壁打ちを依頼します。Opus が膨らませた案を GPT-5 が落ち着かせて本質に戻してくれます。GPT-5 の「システムプロンプトに忠実」な特性も相まって、この二段構えはとても有効でした。

たとえば **「ツール名が衝突したらどうする？」** という問題。ここで AI が提案してきたのは以下の 3 戦略でした。

1. **namespace 戦略**：`{serverId}_{toolName}` の形式で一意化
2. **alias 戦略**：通常は元の名前を使い、衝突時のみ namespace 方式にフォールバック
3. **error 戦略**：衝突時には明示的にエラーを返す

AI が How（戦略の選択肢）を提示し、人間が What / Why を吟味することで「デフォルトは namespace 戦略を採用し、状況に応じて切り替えられるようにする」という方向性が固まりました。

```mermaid
flowchart TD
  A[要求・ユースケース入力] --> B[Claude Code が戦略案を生成]
  B --> C[namespace 戦略 / alias 戦略 / error 戦略]
  C --> D[GPT-5 で壁打ち]
  D --> E[人が What / Why で方向性決定]
```

余談ですが、私の環境では Opus を「フリーレン」、GPT-5 を「フェルン」とイメージしていて、師弟のように掛け合いながらプランを練っていく感じです。

### Construction（構築）

次の段階は実際の実装です。

開発環境はあらかじめ整備しておきました。利用する基本パッケージ、テストフレームワーク、Linter や Formatter などを用意し、AI が生成するコードをすぐ検証できる状態にしておきます。

実装の流れは次のようになります。

1. Claude Code にコード生成を任せ、差分をこまめにレビュー
2. Subagent や Codex CLI（MCP 経由）を利用して追加レビューを実施
3. 自分自身でも「この関数は誰が使う？」「何をする？」「いつ呼ばれる？」と 5W を問い直す

ツール名衝突解決の戦略を実装する際にもこのスタイルを活かしました。AI は namespace 戦略のコードを一気に生成してくれますが、そのままではパフォーマンス面や可読性に課題が残ります。そこで人間がレビューを重ね、衝突検出やエラーハンドリングの仕組みを整備しました。

```mermaid
flowchart TD
  A[Claude Code が実装生成] --> B[差分レビュー]
  B --> C[Subagent / Codex CLI で追加レビュー]
  C --> D[人間が 5W で補正]
  D --> E[衝突回避ロジック実装完成]
```

### Operations（運用）

実装が進むと、今度は運用と改善の段階に入ります。

- Claude Code GitHub Actions を活用して、自動レビューを AI に任せる
- Plan Mode を使って修正が過剰になっていないか確認する
- MCP 経由で多段階レビューを回し、得られたフィードバックを次の Inception へ戻す

ツール名衝突解決についても、このフェーズで「実際の利用環境ではどの戦略が使いやすいか」を検証しました。利用者が混乱しないようにエラーメッセージを改善したり、ドキュメントに事例を追記したりするのも Operations での作業です。

```mermaid
flowchart TD
  A[コード完成] --> B[GitHub Actions による AI レビュー]
  B --> C[Plan Mode で修正チェック]
  C --> D[MCP 経由の多段レビュー]
  D --> E[利用状況をフィードバック]
  E --> A
```

## まとめ

Hatago MCP Hub の開発を AI-DLC の観点で振り返ると、AI と人間の協働がごく自然に組み込まれていたことが分かります。

- Inception：AI が How を提案し、人間が What / Why で方向性を決める
- Construction：AI がコードを生成し、人間がレビューで磨く
- Operations：AI がレビューや検証を補助し、人間が次の改善につなげる

ツール名衝突解決の 3 戦略のように、AI が提示した選択肢を人間が吟味して現実的な落とし所を見つける場面は、まさに AI-DLC の実践例だと感じます。

もちろん、丁寧にサイクルを回す分、手間や疲労感は少なくありません。しかし「AI と一緒に開発している感覚」が得られるのはとても魅力的で、OSS 開発における新しい可能性を感じました。

読んでくださったみなさんも、ぜひ自分に合った AI 協働スタイルを探ってみてください。きっと開発の景色が大きく変わるはずです。
