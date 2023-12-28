---
title: 'イベントストーミング'
---

# イベントストーミングとは

イベントストーミングは、**複雑なビジネスプロセスやドメインの知識を共有、可視化、そして理解するための協同作業ベースのモデリング手法**です。
伝統的なモデリングや要件定義の手法は、多くの場合、ビジネスステークホルダーとソフトウェア開発者の間にコミュニケーションのギャップが生じる可能性がありました。イベントストーミングは、このギャップを埋めるために生み出されました。この手法は、関係者が同じ空間に集まり、ホワイトボードや `Miro` などのオンラインコラボレーションツールを用いて直感的でインタラクティブな方法でドメインの知識を共有し、問題点やビジネスルールを特定することを目的としています。

## イベントストーミングを通じて作成する図

本章でイベントストーミングを通じて作成された図は以下の通りです。この図では、オンライン書店のドメインにおける主要なドメインイベントと、それらを実行するためのプロセスが示されています。また、コンテキストの境界も定義されています。今回は省略しますが、これらのコンテキスト間の統合のためにコンテキスト間の関連性を追加し、コンテキストマップを作成することもできます。

@[figma](https://www.figma.com/file/g04nAogGCGgM62IKXHUSLT/Online-bookstore?type=whiteboard&node-id=843-1791&t=hvtQJVFBGW90SwuN-0)

それではどのような過程で作成されるか見ていきましょう。

## イベントストーミングのルール

イベントストーミングにはいくつかの要素とルールが存在します。
まずは登場する主要要素を 7 つ紹介します。

1. **ドメインイベント (Domain event)**
   これは「何が起こったか」という出来事を表します。イベントは過去に起きるものなので過去形の動詞で表現されます (例:「注文が承認された」「商品が出荷された」など) 。ドメインイベントはビジネス上の重要な出来事や変更を示すもので、ドメインの中心的な要素として捉えられます。
2. **ポリシー (Policy)**
   ドメインイベントが発生したときに何をすべきかを定義するビジネスルールやロジックを表します。ポリシーはあるドメインイベントに応じて実行すべきコマンドを決定するためのガイドラインとなります。
3. **コマンド (Command)**
   システムに何かを**行うように**という指示や命令を表します。その結果としてドメインイベントが発生します。 (例:「注文を承認する」「商品を出荷する」など)
4. **アクター (Actor)**
   コマンドを発行する人やシステムのことです。アクターは外部の利用者やシステムであり、特定のコマンドを実行することでドメインイベントを発生させます。
5. **ビュー/リードモデル (View/Read model)**
   システムの状態や情報を参照するための表現です。これは通常、ユーザーインターフェイスで提供されるデータの表現を指します。
6. **集約 (Aggregates)**
   集約とは、関連するデータとそのデータを操作するビジネスルールや制約を一つのグループにまとめたものです。これにより、データの不整合を防ぐための「一貫性の境界」が設けられ、この境界内でのみデータの状態変更が可能です。集約は複数の値オブジェクトやエンティティをまとめ、それらが共に働くことでビジネスの要件を満たすよう設計された、データとロジックのカプセル化された集合体です。
   :::message
   値オブジェクト (chapter08) 、エンティティ (chapter09)、集約 (chapter10)の詳細や実装はそれぞれの章で説明します。
   イベントストーミングの段階で完璧な集約を定義することは難しいです。
   プロジェクトが進行するにつれてその定義を洗練させていくのが良いでしょう。
   :::
7. **外部システム**
   その名の通り外部のシステムです。たとえばメール送信に`SendGrid`、決済に`Stripe Payments` などを利用する場合、それらは外部システムとなります。

そしてこれらの要素には次の図のような関係性があります。
@[figma](https://www.figma.com/file/g04nAogGCGgM62IKXHUSLT/Online-bookstore?type=whiteboard&node-id=10-1551&t=ppbGdPusmMcJCkrs-0)
ドメインイベントを中心に捉え、イベント同士に繋がりがある場合に必ず以下の 2 パターンのどちらかを経由し、繋げる必要があります。

- ドメインイベント -> ビュー/リードモデル -> アクター -> コマンド -> 外部システム/集約 -> ドメインイベント
- ドメインイベント -> ポリシー -> コマンド -> 外部システム/集約 -> ドメインイベント

複数の関係者やシステムがどう絡み合っているのかを理解するため、参加者全員で情報を出し合い、誰がどんな行動を取り、それによってどんな出来事が起こるのかを一緒に書き出していきます。このやり取りを繰り返すことで、全体の流れが見えやすくなり、より明確にドメインを理解できるようになります。

## イベントストーミングの進め方

フローを細分化して説明していきます。

1. **参加者の選定**

- ドメインエキスパート
- ソフトウェア開発者
- プロダクトオーナーやプロジェクトマネージャー
- UX/UI デザイナーなどの関連するステークホルダー
  :::message
  構築したいサービスに UI が必要となる場合、UI/UX デザイナーも同席することをオススメします。理由は 3 点です。

  - **直感的なインターフェイスの議論**
    システムの動作やプロセスを議論する際、それをサポートする UI の要件や挙動に関するディスカッションも必要となることがあります。デザイナーが同席することで、これらの議論がより具体的かつ効果的に進行します。

  - **プロトタイピングの迅速なフィードバック**
    イベントストーミングの内容をもとにプロトタイプやモックアップを作成し、関係者にフィードバックをもらうことは非常に重要です。デザイナーが同席することでその場で迅速かつ効果的なフィードバックを得ることができます。

  - **UI/UX の向上**
    システムやサービスの成功は、技術的な実装だけでなく、最終的な` UI/UX` にも大きく依存します。デザイナーが初期段階から関与することで、最終的な製品の品質を向上させることができます。
    :::

2. **ツールの選定**

- ホワイトボード (オフライン)
- `Miro`、`FigJam` などのオンラインコラボレーションツール

3. **ドメインイベントの洗い出し**
   主要なドメインで起きるドメインイベントを参加者全員で洗い出します。恐れず、思いつく限り出しましょう。この時に何となくの時系列で配置しておくと 4 の手間が軽減します。
4. **ドメインイベント同士を時系列で繋げる**
   ドメインイベント同士に繋がりがある場合、ドメインイベントと同士を時系列の流れに合わせて矢印で繋ぎます。
5. **ドメインイベント同士のギャップを埋める**
   前述したルールに則ってドメインイベント間のギャップを埋めていきます。
   a. **疑問、懸念事項、不確実要素をメモする**
   イベントストーミングを通して発生した疑問、懸念事項、不確実要素などをホットスポットとしてログを残します。この作業はどのタイミングで行っても大丈夫です。
6. **集約を特定する**
   コマンドの目的語に当たるものが集約である可能性が高いです。たとえば「注文を承認する」がコマンドの場合「注文」が集約になります。
7. **境界づけられたコンテキストの定義、コアドメイン、サブドメインを特定する**
   6 で集約を特定した後に同一名称の集約が複数存在する場合があります。その場合集約同士を比較し、言葉が持つ意味や関連情報が異なる場合、コンテキストの境界になる可能性があります。適当に線を引き区分けしましょう。その後、それぞれのコンテキストで、それがコアドメインなのかサブドメインなのかを特定します。

:::message
上記の順番を完璧に守る必要はありません。フローが進むにつれて新しいドメインイベントの発見や、考慮漏れが必ず発生するのでその都度書き加えるなど柔軟に対応しましょう。
:::

# オンライン書店を例に実践

これまでドメインの一例として扱ってきたオンライン書店で実際にイベントストーミングを行なってみましょう。

:::message alert
ここでの内容はオンライン書店のビジネスモデルや要件に応じて異なります。シンプルにするため、簡略化した内容となっています。あくまでイベントストーミングの手法を理解するための例としてご覧ください。

また、著者はオンライン書店のドメインエキスパートではございません。
書店や `EC` 領域の解釈に間違いがある可能性がございます。あらかじめご了承ください。
:::

1. **参加者の選定**
   今回は著者一名で行います。
2. **ツールの選定**
   今回は[FigJam](https://www.figma.com/ja/figjam/)を利用していきます。
3. **ドメインイベントの洗い出し**
   @[figma](https://www.figma.com/file/g04nAogGCGgM62IKXHUSLT/Online-bookstore?type=whiteboard&node-id=1-1547&t=ppbGdPusmMcJCkrs-0)
4. **ドメインイベント同士を時系列で繋げる**
   @[figma](https://www.figma.com/file/g04nAogGCGgM62IKXHUSLT/Online-bookstore?type=whiteboard&node-id=332-553&t=P5SgzhzwPW6aWrpT-0)
5. **ドメインイベント同士のギャップを埋める**
   ここではプロセス中に考慮漏れに気づき、ドメインイベントを追加しています。
   @[figma](https://www.figma.com/file/g04nAogGCGgM62IKXHUSLT/Online-bookstore?type=whiteboard&node-id=486-862&t=3OOKVvzzGCmuQPXD-0)
   a. **疑問、懸念事項、不確実要素をメモする**
   @[figma](https://www.figma.com/file/g04nAogGCGgM62IKXHUSLT/Online-bookstore?type=whiteboard&node-id=729-1163&t=vWwcw54QwFOojvjC-0)
6. **集約を特定する**
   @[figma](https://www.figma.com/file/g04nAogGCGgM62IKXHUSLT/Online-bookstore?type=whiteboard&node-id=729-1434&t=vWwcw54QwFOojvjC-0)
7. **境界づけられたコンテキストの定義、コアドメイン、サブドメインを特定する**
   @[figma](https://www.figma.com/file/g04nAogGCGgM62IKXHUSLT/Online-bookstore?type=whiteboard&node-id=843-1791&t=hvtQJVFBGW90SwuN-0)

以上でイベントストーミングは完了です。

複数のコンテキストが存在することがわかりました。今回は省略していますが、実際にはこれら以外にも「決済コンテキスト」や「認証コンテキスト」などのコンテキストが存在するでしょう。

イベントストーミングを通じて作成された図は、ドメインにとって重要なワークフローを視覚的に表現するものです。このプロセスは、関係者がビジネスの要件、ルールを理解し、システムの振る舞いや相互作用を明確に把握するのに非常に役立ちます。しかしながら、ビジネスや市場のニーズは時間とともに変化します。したがって、イベントストーミングが一度完了したとしても、その成果物は静的なものではなく、進化し続けるドメインの動きを反映するために定期的なレビューと更新が必要となります。

# まとめ

- イベントストーミングは、複雑なビジネスプロセスやドメインの知識を共有、可視化、そして理解するための協同作業ベースのモデリング手法
- 変化に対応するため定期的更新が必要

本章では「コアドメイン、サブドメインの特定」「境界づけられたコンテキストの定義」「エンティティの特定 (ビジネス要件の確認、エンティティの識別) 」「ビジネスルールの把握」を行いました。
以降「在庫管理コンテキスト (サブドメイン) 」を開発対象として扱い、ドメイン駆動設計 を探究していきます。
次章では戦術的設計に向けて「エンティティの特定(属性の識別、エンティティ間の関連の定義)」を、よりわかりやすい形で可視化していきます。
ツールには`PlantUML`を利用し、コードとして管理する流れを紹介します。

## 参考文献

https://www.eventstorming.com/
https://www.youtube.com/watch?v=jC9lE4YqgyY