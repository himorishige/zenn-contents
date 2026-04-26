---
title: "はじめに／本書の歩き方"
free: true
---

:::message alert
本章は **Sprint 1 で執筆予定のスケルトンチャプター** です。内容は確定していません。
:::

## 本書の到達点

AWS Bedrock AgentCore（6 サービス）と Bedrock ネイティブ Nemotron（Super 120B / Nano 9B v2）、LangGraph を組み合わせて、**社内ドキュメント Q&A エージェント**を Production Grade で構築する。ローカル GPU は不要、Mac / Linux / WSL2 + AWS アカウントだけで、月額 500 USD 程度で本番運用品質のエージェント基盤を組み上げる。

## 前作 2 冊との関係

- 前作 1 冊目: [NeMo Agent Toolkit + NIM ハンズオン（DGX Spark 編）](https://zenn.dev/himorishige/books/nemo-agent-toolkit-nim-handson)
- 前作 2 冊目: [NeMo Agent Toolkit 実践運用編 — Guardrails × Langfuse](https://zenn.dev/himorishige/books/nemo-agent-toolkit-production-ops)

本書は前作 2 冊と独立した新作だが、付録 A で「前作の 4 本柱（Orchestration / Guardrails / Observability / Eval Dataset）が AWS マネージドのどこに対応するか」を差分マップで示す。

## 対象読者

- AWS アカウントを持ち Bedrock を多少触ったことがある開発者
- 前作読了者で「OSS スタックを AWS マネージドで組み直したい」層
- SaaS 本番運用に責任を持つ AI エンジニア

## 本書のスコープ外

- AWS の基礎（IAM / VPC / S3 等）の入門
- LangGraph の基礎（前作実践運用編で扱い済み）
- ローカル GPU での推論

## 章依存図（暫定）

```
Ch 0 はじめに
  │
Ch 1 全景
  │
Ch 2 環境セットアップ
  │
  ├─ Ch 3 Bedrock Nemotron 入門 → Ch 4 Service Tiers
  │
  ├─ Ch 5 AgentCore Runtime → Ch 6 Memory → Ch 7 Gateway → Ch 8 Identity → Ch 9 Built-in Tools → Ch 10 Observability
  │
  ├─ Ch 11 Knowledge Bases → Ch 12 Guardrails
  │
  ├─ Ch 13 評価 → Ch 14 マルチエージェント
  │
  └─ Ch 15 IaC → Ch 16 コスト最適化
       │
       └─ 付録 A / B / C
```

<!-- TODO: Sprint 1 で執筆 -->
