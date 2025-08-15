---
title: 'Windsurf 波乗りガイド: Wave 12'
emoji: '🌊'
type: 'idea'
topics: ['windsurf', 'devin', 'ポエム']
published: false
---

https://windsurf.com/

:::message
この記事は、2025 年 8 月時点、Windsurf Wave 12 までの内容を基にしたものです。
ここに示された意見はわたし個人のものであり、所属する組織を代表するものではありません。
:::

## はじめに

2025 年 8 月 14 日、Windsurf 開発チームから待望の大型アップデート「Wave 12」がリリースされました。Cognition による買収後、2 つ目のメジャーリリースとなる今回は、単なる機能追加に留まりません。ある意味 SWE エージェント「Devin」の魂（？）が、Windsurf に吹き込まれた最初の波とも言えるかもしれません。

今回も、DeepWiki の統合から全面的な UI 刷新まで、Wave 12 で導入された革新的な機能を、ポエム要素を多めに含めてご紹介します。Windsurf を使った日常的な開発体験がどう進化するのか、ざっくりと見ていきましょう。

## Wave 12 で追加された新機能一覧

https://windsurf.com/blog/windsurf-wave-12

Wave 12 では、開発者がより開発に集中できるための多数の革新的機能が導入されました。

- **DeepWiki Integration**
- **Vibe and Replace**
- **Cascade Agent の改善**
- **Tab Autocomplete の進化**
- **全面的な UI 刷新**
- **Dev Containers サポート**
- **100 以上のバグ修正**

## DeepWiki Integration

Wave 12 でもっとも印象的な機能の 1 つが、[DeepWiki](https://docs.devin.ai/work-with-devin/deepwiki) の統合です。ソースコードとの向き合い方が大きく変わりそうです。

コンテキストエンジニアリングという考え方が一般的になってきた今、ある意味最強のコンテキストを得られる仕組みが登場したのではないでしょうか。

### 従来のホバーカードを超えた体験

これまでは、関数にマウスをホバーしても型情報やコメントが表示されるだけでした。しかし DeepWiki との統合により、そのシンボルが「なぜ存在するのか」「どう使われるのか」といった詳細なドキュメントまで参照できます。

![DeepWikiのホバー表示のスクリーンショット](/images/windsurf-guide-wave12/deepwiki-hover.png)

### 2 つの利用パターン

ホバーで情報を得るだけでなく、`cmd/ctrl+shift+click`（または Read More をクリック）でパネルに詳細を表示し、その情報を`cmd/ctrl + l`で Cascade のコンテキストに追加できます。

![DeepWikiのパネル表示のスクリーンショット](/images/windsurf-guide-wave12/deepwiki-panel.png)

## Vibe and Replace

Find and Replace は開発の基本ツールですが、Vibe and Replace はこの概念を知的作業へと昇華されています。

### 単純な置換から

Vibe and Replace は、見つけた文字列それぞれに AI プロンプトを適用し、文脈に応じたインテリジェントな変換をします。

たとえば、関数定義を見つけて関数の内容に応じたドキュメントを自動生成したり、`data`のような曖昧な変数名を文脈に合わせて`userProfile`や`apiResponse`へとリネームしたり。フィーチャーフラグの削除のような複雑なリファクタリングも、コードの意図を汲み取って安全に実行してくれるようです。

まだ試せていないのですが、単純作業から解放され、より創造的な問題解決に集中できそうですね。

## Cascade のアップデート

Wave 12 の Cascade は、より賢く、より自律的に動くエージェントに進化しました。

### Planning Mode の完全自動化

複雑なタスクを依頼すると、Cascade が自動で計画の必要性を判断し、ToDo リストを生成します。開発者は「このタスクは複雑だから…」と考えて Planning Mode を有効にする必要すらありません。（個人的に Planning Mode の ON/OFF がわかりづらく悩んでいたので嬉しいところ）

![Cascadeが自動で生成したToDoリストのスクリーンショット](/images/windsurf-guide-wave12/cascade-todo.png)

## Tab Autocomplete

新しい Tab Autocomplete は、「より頻繁で賢い提案を、より控えめに」という哲学に基づいて提案の頻度が上がり、入力位置と重なっていた UI が改修され UI の圧迫感は軽減されています。

![Tab Autocompleteの控えめなUIとdiff表示のスクリーンショット](/images/windsurf-guide-wave12/tab-autocomplete.png)

## 全面的な UI 刷新

刷新された UI は、AI エージェントとの協働を可視化し、より効果的にするために再設計されています。機能の拡充によってボタンやアイコンがかなり多くなってきていましたが、新しい UI ではより簡潔なデザインになっています。

![新しいChatパネルとCascadeパネルのスクリーンショット](/images/windsurf-guide-wave12/window-new.png)
_新しい UI、シンプルなインターフェース_
![古いChatパネルとCascadeパネルのスクリーンショット](/images/windsurf-guide-wave12/window-old.png)
_旧 UI、要素がそれなりに多い_

## Dev Containers

VS Code をフォークした IDE では Dev Containers のサポートがイマイチな部分ありましたが、Wave 12 へのアップデートで大幅な改善が入りました。

Windsurf エディターから直接コンテナーで作業できるため、チームで統一された開発環境を簡単に構築できます。リモート SSH 経由でも利用でき、開発からデプロイまで一貫した環境で作業できるのは、品質向上に大きく貢献するはずです。

## 安定性への徹底的な取り組み

Wave 12 では 100 以上のバグが修正され、信頼性が向上しました。詳細は不明ですが、より安定した開発環境が利用できるようになるはずです。

## まとめ

Wave 10、11 も大きな機能追加が続いていましたが、大人の事情もあったのかどこか少し物足りなさもありました。Cognition との統合による Wave 12 は一気にその進化を加速させたのではないかと感じています。とくに DeepWiki との統合は、新しいコードベースに飛び込む際の心理的な壁を大きく下げてくれる、非常に画期的な機能だと感じています。

何より AI に任せて開発することだけでなく、開発者自身が手を動かして開発することの楽しさをサポートしてくれる UX を提供してくれているのではないかと思います。

AI との協働を前提とした新しいワークフローを学び、実践していく。そんな能動的な姿勢が、これからの開発者には求められるでしょう。

この新しい波に乗って、より効率的で創造的な開発体験を手に入れましょう。

Surf's up 🏄‍♂️🤙

---

## 参考

[1]Windsurf Team. "Windsurf Wave 12: Devin features in Windsurf." Windsurf Blog, August 14, 2025. https://windsurf.com/blog/windsurf-wave-12

[2]Windsurf Team. "Windsurf Editor Changelog." Windsurf, August 14, 2025. https://windsurf.com/changelog
