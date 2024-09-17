---
title: "チーム全員で品質課題の改善のために取り組んだことを振り返る 〜いつからQAエンジニアがいなければ改善できないと錯覚していた？〜"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["品質向上", "テスト", "スクラム"]
published_at: 2024-12-05 09:00
published: true
publication_name: "loglass"
---

![](https://storage.googleapis.com/zenn-user-upload/24c4f317d6b2-20241202.png)

:::message
この記事は **[株式会社ログラス Productチーム Advent Calendar 2024](https://qiita.com/advent-calendar/2024/loglass)** のシリーズ 2、**5日目** の記事です。
:::

## はじめに


こんにちは、ログラスのエンジニアの粟田（[@wooootack](https://twitter.com/wooootack)）です。時間の流れは早いもので、私がログラスに入社してもうすぐ1年が経過します。

今回は、入社してからの1年を振り返り、私たちのスクラムチームが直面した品質課題と、それを乗り越えるための奮闘を実体験に基づいて紹介します。チームに専属QAがいない環境でも、チーム全体で品質課題の改善に向けてどのような取り組みを行ったのか、その過程で得られた学びや教訓を共有していきます。

エッセイ要素と技術要素が半分ずつくらいになりますので、ぜひ隙間時間などを使って気軽にお読みいただけると幸いです。ここで紹介するプラクティスや事例が、品質改善に取り組む他のエンジニアやチームにとって少しでも参考になる情報があれば幸いです。

また、別のメンバーが異なる視点でチームの成長を振り返った以下の登壇資料も併せてご覧いただけるとより楽しんでいただけるかと思います。

https://speakerdeck.com/hc0208/how-bringing-in-a-scrum-master-to-an-already-self-organized-team-totally-worked-out

:::message
この記事は技術書典17で配付したテックブックにも掲載されています。ぜひチェックしてみてください。そしてなんと無料で読めます！
:::

https://techbookfest.org/product/wCDWvUCj8m07U6vuvv3YSu?productVariantID=cjZ9wf7zmExGysp6bZL6Nw

## 当時のチーム状況

まずは当時の状況を振り返りながら、どのような課題があったのかを見ていきます。私たちのチームで品質課題が顕在化してきたのは、私が入社してから3ヶ月から半年ほど経過した頃でした。当時のチームは、組織拡大によるメンバーの入れ替わりやプロダクトの複雑化、ストレッチなビジネス目標などといったさまざまな要因から、特に品質面での課題を感じていました。それっぽくまとめると、プロダクトの成長にチームの成長が追いついていないという状態だったと言えます。

![](https://storage.googleapis.com/zenn-user-upload/a564f5d7ee46-20240926.png)
*（参考）自分の入社以降のメンバーの入れ替わり*

弊社にはQAエンジニアの方が2名在籍している（2024年9月時点）のですが、私たちのスクラムチームの専属として所属している形ではありませんでした。もちろん相談に乗ってもらうことはいつでも可能な状況ではありつつも、根本的な解決は自分たちで推進していく必要がありました。しかし、私たちのチームは明確な課題感を持ちつつも、どのような改善活動を進めていくことが良いのかという答えを見つけられずにいました。

そんなとき、「LEADING QUALITY」という書籍で紹介されている「品質ナラティブ」という概念を知りました。そしてこのナラティブを使って、私たちのチームがどのような状態にあるのかを言語化し、それに対してどのような改善活動を進めていくべきなのかを整理することができました。

※ 以下引用部分はp19-p27を抜粋しています。

https://www.kadokawa.co.jp/product/302309001510

### 品質ナラティブ

せっかくなので「品質ナラティブ」という概念について少しご紹介させてください。初めて聞いた方に向けて簡単に説明しますので、既に知っている方は飛ばしていただいても構いませんが、復習も兼ねて読んでいただけると幸いです。あくまで概要だけの説明になるので、より詳しく知りたい方は巻末に貼ったリンクからぜひ読んでみてください......というより、品質向上に興味があるかたは絶対に読んでほしい1冊です。

LEADING QUALITYにおいて「品質ナラティブ」は以下のように定義されていて、さらに3つのナラティブに分類されています。

> 品質ナラティブとは、企業で品質について考えたり話したりしている、その「語られ方」だ。ジェイソンが述べたように、その存在を知っていようといまいとナラティブは存在し、企業の品質文化に日々影響している。自組織のナラティブを明晰に理解すればするほど、組織に変革をもたらし、目標を達成するために、どのようにナラティブを調整する必要があるかを楽に考えられるようになる。

この「品質ナラティブ」という概念があることで、空中戦になりがちな品質向上に対して、より具体的な議論を進めやすくなったと感じています。続いてそれぞれのナラティブについての説明と、私たちのチームにおいてどのような状態にあったのかを紹介していきます。

#### 責任ナラティブ（The Ownership Narrative）

このナラティブは以下のように定義されています。

> 誰が品質に責任を持つかが考えられ、語られている

そしてこのナラティブは以下のようになっていることが望ましいとされています。

> テストチームやエンジニアチームだけが品質の責任を持てばいいわけではなく、むしろ、責任ナラティブを広げて、高品質なプロダクトを確実にリリースするために全員が自身の役割を理解できるようにしなくてはならないのだ。

私たちのチームの責任ナラティブは、この望ましいとされている状態にかなり近しい状態にあったと感じています。コードの品質だけでなく、何をつくるべきかという観点からも品質に対する責任を持っているメンバーしかいなかったと断言できます。

#### テストナラティブ（The “How to Test” Narrative）

このナラティブは以下のように定義されています。

> 品質向上につながる正しいテスト技法はどれか・どのツールを使うべきかが考えられ、語られている

そしてこのナラティブは以下のようになっていることが望ましいとされています。

> 最高のテストナラティブは、次の2つのことに焦点を当てている。
>
> まず、テストに関するさまざまなオプションや実現方法を明確に理解することだ。これがわかれば、各オプションの利点や限界、および、プロダクトに関するどんな情報が得られるかを評価できる。これにより、より良い戦略的な意思決定が下せる。
> そして、チーム・プロダクト・組織の成熟度を明確に理解しよう。状況を把握することで選択肢が現在の開発段階に適しているかを確認できる。

私たちのチームのテストナラティブは、残念ながらこの望ましい状態には達していなかったと感じています。そもそもオプションや実現方法についての知識が浅い状態でした。また、成熟度も明確になっていなかったため、どこをどう伸ばしていくべきなのかが分かっていない状態でした。


#### 価値ナラティブ（The Value Narrative）

このナラティブは以下のように定義されています。

> 品質に投資した場合の見返り（ROI: 投資収益率）が考えられ、語られている

そしてこのナラティブは以下のような観点で考えることが望ましいとされています。

> 品質への投資がもたらす価値を議論する場合に重要なのは、3つの主要な分野である収益性・コスト削減・リスク軽減に焦点を当てることだ。

私たちのチームの価値ナラティブは、残念ながらこの望ましい状態には達していなかったと感じています。ただ、全く議論がされていないという訳ではなく、例えば「ここは自動テストを導入することでコストを削減できるのでは？」などといった議論はされていました。しかし、それを収益性やコスト作戦、リスク軽減というようにつなげて議論するレベルにはなかったと感じています。

## なぜ品質課題の解決に取り組んだのか

具体的な改善活動を紹介する前に、そもそもなぜエンジニアが品質課題の改善に取り組んだのか？　という疑問を持っている方がいるかもしれないので、突然ですが少しだけ自分語りをさせてください。私はこれまでの経験から、先頭に立って引っ張っていくよりは、チームの成果を最大化するためのサポートをすることが得意だと認識していました。そしてこれからもそのような形でチームに貢献していきたいと考えています。しかし、チームには私よりも経験豊富で実力もあるメンバーが多かったり、専任のスクラムマスターが所属していたりと、どのようにチームに貢献していくことができるのかが見えていない状況でした。

そんな迷っている私にとって、品質課題の改善はチームに貢献するチャンスでした。品質課題の改善はチームにとって価値があるにも関わらず、品質課題に対しては他のメンバーもそこまで経験や知識がない状態でした。また、普段から社内のQAエンジニアの働き方を見て、少しやってみたいなという気持ちもありました。それ以外にも、チームに貢献できる・やりたいと思える・成長できる、という理由もあり、私は品質課題の改善に取り組むことを決意しました。

ただ、私はこれまで体系的に品質保証について学んだこともなければ、QAエンジニアと一緒に仕事をするのもログラスが初めてで、特段品質について知識が多いわけでもありませんでした。そんな状態でも、チームメンバーやQAエンジニアは常に協力的な姿勢でいてくれましたし、マネージャーも背中を押してくれました。本当に感謝しています。

## 品質課題と改善活動

さて、ここからは私たちのチームが抱えていた具体的な品質課題とそれに対する改善活動を1つずつ紹介していきます。基本的には時系列に沿って紹介していきますが、実際には並行して進めていたり、一部の活動は継続して行っているものもあります。

### 大きくなりすぎたテストとストレッチなスプリントゴール

最初に取り組んだのは、テストを小さくして、各スプリント単位で実行することでした。私たちのチームは、元々以下のような形でスプリントを進めており、いわゆるテストをフェーズとして分けている状況でした。

![](https://storage.googleapis.com/zenn-user-upload/0d6c8986ea0a-20240925.png)
*改善前の開発の進め方*

この状態だと「スプリントの終了ギリギリまで実装を進めてなんとか実装ができたからゴール達成！」といった状況になりがちでした。さらにビジネス目標がストレッチという背景もあり、それを追いかけるためにスプリントゴールもギリギリまで実装を進めることでなんとか達成できるような、ストレッチなものが多かったです。そして、スプリントレビューでフィードバックを得る際に、テストが不足していることでエッジケースで不具合が発生してしまったり、そもそもスプリントレビューで見せられるものが用意できなかったり、「自信を持ってリリースできない状態のものを作ってスプリントゴール達成として良いのか？」という議論がチーム内で起きる状態になっていました。

それ以外にも、テストが後ろ倒しになってしまうことで「まとめてテストする必要があり影響範囲の考慮漏れが発生しやすい」「不具合や欠陥の発見が遅れることで修正コストが高くなる」「リリースまでの見積もりが難しくなる」などの弊害もありました。これらの課題を解決するための1つめの改善活動が「スプリントゴールを緩めて、スプリントごとにリリース可能な完成したリリース物を作る」ことでした。これにより、私たちのチームは以下のような形でスプリントを進めるようになりました。

![](https://storage.googleapis.com/zenn-user-upload/027c80891d19-20240925.png)
*改善後の開発の進め方*

この簡単そうに見えるかもしれない改善活動ですが、実際にはチーム全体での意識変革が必要でした。特にスプリントゴールを緩めることでビジネス目標を達成できなくなるのではないか、という不安がありました。そのため、スプリントゴールを考える際には、いかにやることを少なく抑えつつビジネス目標を達成できるか？　ということを考えるようになりました。ただ、少しでも気を抜くとまたストレッチなスプリントゴールを置きたくなってしまうため、今までの感覚の7割程度というのを目安に考えていました。この改善活動はスクラムマスターのサポートなくしては進めることができなかったと感じています。

これにより、スプリントゴールを達成するためにテストを後回しにすることがなくなり、スプリントごとにリリース可能な完成したリリース物を作ることができるようになりました。また、スプリントレビューでフィードバックを得る際にも、テストが不足していることでエッジケースで不具合が発生してしまったり、そもそもスプリントレビューで見せられるものが用意できないという状況も改善されました。しかし、ここで改善できたのはあくまで進め方の一部であり、影響範囲の考慮漏れという課題は思ったよりも改善されていませんでした。

### 防ぎきれないテストケースの考慮漏れ

テストを小さく進めるようにしたが、影響範囲の考慮漏れを改善できなかった私たちは、新たな改善活動を模索していました。この漠然とした悩みを社内のQAエンジニアに相談してみたところ「テスト分析や設計のプロセスが不足しているんじゃないか」というアドバイスをいただきました。元々のテストは以下のような形で進めており、いわゆるテスト分析と設計は明示的に行ってはいませんでした。つまり、テストナラティブが望ましい状態に達していなかったということです。

![](https://storage.googleapis.com/zenn-user-upload/d15a102f102a-20240925.png)
*PFDでテスト分析と設計と明示的にしていない場合のテストプロセスを表現*

テスト分析や設計を飛ばしてテスト実装の成果物であるテストケースに対してレビューをするため、レビューのコストも高くなりがちですし、考慮漏れも発生しやすい状況でした。また、私たちのチームはテスト分析や設計を行うためのフレームワークやガイドラインがなく、どのように進めていけば良いのかがわからない状況でした。そんなときに、ちょうど社内でテスト分析や設計のプロセスを整備するための取り組みが進められていたこともあり、私たちのチームもそれに乗じて取り組むことにしました。

こちらの取り組みについて詳しく知りたい方は、以下の記事も併せて読んでみてください。

https://zenn.dev/loglass/articles/a71a21d8ce992a

また、個人的な話ではあるのですが、ちょうど私が 「WACATE」 という勉強会で、テストにおける意図の重要さについて学んできたということもあり、ちょうど私もチームに取り入れたいなと考えていました。詳細はこちらの記事「WACATE2024夏に参加しました」にまとめていますので、興味があればご覧ください。

https://wacate.jp/

https://zenn.dev/loglass/articles/c7af93b861f5bb

少し話は脱線しましたが、テスト分析や設計をプロセスに組み込むことで、私たちのチームは以下のような形（次ページ参照）でテストを進めるようになりました。

この改善活動により、テストケースの考慮漏れ自体は改善方向に向かったと感じています。加えて、これまでは属人化しがちだったテストの知識や経験を共有する機会が増え、チーム全体でテストに対する意識が高まったと感じています。この新しいプロセスは一見するとかなり重たいように見えますが、ペアやモブで取り組むことで手戻りやレビューのコストを抑えるといった工夫もしていました。

![](https://storage.googleapis.com/zenn-user-upload/5217a1fba9ef-20240925.png)
*PFDでテスト分析と設計と明示的に行った場合のテストプロセスを表現*

### チームに浸透しきらず、忘れられてしまうテスト活動

次に取り組んだのは、テスト分析や設計をいつどんな時にやるべきなのかを明確にし、チームで認識を揃えることでした。これらが明確になっていないことで、特定の修正に対してテスト分析や設計までやるべきという考えと、やらなくても良いという考えがチーム内に混在してしまい、確認のやり取りが発生してしまうことがありました。この課題を解決するために Definition of Done を定義し、そこにテスト分析や設計のプロセスを明記しました。テスト以外の部分もいくつか追加しているので、関係ないものもありますが、このような形になっています。

![](https://storage.googleapis.com/zenn-user-upload/1bed0ee6a4d8-20240926.png)
**作成したDefinition of Done**

ここで作成した Definition of Done はチェックリスト形式で作成されているため、プルリクエストのテンプレートにも追加しています。これにより、プルリクエストを作成する際には Definition of Done を確認することが自動的に促されるようになりました。

また、作成した Defition of Done には「リリースされるまでに必ず満たしておくべき条件」が明記されているのですが、あえて明示的なチェックポイントは設けませんでした。明示的なチェックポイントを設けることで漏れを減らすことにはつながるかもしれないとは思いますが、チームの意識を変えていくという観点ではあえて実装者とレビュワーに責任を持たせる形にしました。このあたりのルール設計は、スクラムマスターのサポートを受けながら進めていきました。

### 経験と知識不足による不安や、コスト面の懸念

これまでお話してきたように、私たちのチームは新しい開発プロセスを取り入れ改善を進めてきました。しかし、新しいプロセスを試していく中で、「自分たちがうまくやれているのか？」であったり、「コストが大きくなりすぎていないか？」という新たな不安や懸念が生まれてきました。私たちのチームは、これから価値ナラティブをどうしていくべきなのかを話し合う必要がありました。この状況になった背景には、私たちのチームがこれまでの経験や知識が不足していることが大きな要因であると感じています。私はこれに対する特効薬はなく、チームで少しずつ経験や知識を積み重ねていくべきだと考えています。ここでは、私たちのチームがどのようにこれらの課題と向き合い進んでいったのかを紹介していきます。

まず、うまくやれているのかの不安とどう向き合ったのかを紹介します。はじめに自分の考えですが、テスト分析や設計に正解や終わりはないと考えています。そのため、まずは自分たちが自信を持ってリリースできる状態になるまでやり切るということを目標に置きました。とはいえ、私もテストについての知識や経験は浅い状態だったので、テストに関連する本を読んだり社外の勉強会に参加したりとまずは自分自身が知識をつけることを最優先に進めました。そしてこれらの知識を社内のQAエンジニアに協力してもらいながら、チームや組織全体に共有していく場を作っていきました。それに加えて、チームでテスト分析設計をする際は、なるべくモブやペア作業で取り組むようにし、ナレッジシェアやフィードバックを積極的に取り入れるようにしました。短期的な解決策を何も用意できなかったのは悔しいですが、長期的にスキルを磨いていくというのが最も良い解決策だと考えています。

参考までに自分が読んだ本や参加した勉強会を巻末にまとめておきますので、もし興味がある方がいれば参考にしてみてください。

次にこの取り組みに対するコストとどう向き合ったかを紹介します。これに関してはまず「コストは増えて然るべきだ」ということをチームで話し合いました。私たちはこれまで、テスト分析や設計といったフェーズをそもそも省いていた状態であったため、これらのフェーズを取り入れることでコストが増えるのは当然のことだという認識を持ちました。ただし、これを一方的に伝えてもなかなか納得されないと思います。そのため、私はテスト分析や設計にコストを払うようになった分、後から問題が発覚するリスクは下げられておりそれによって後から修正するコストは下げられるはずだという話や、現状では複雑な仕様に対してテストをしようとしているので、このコストは避けては通れないよねといった対話をチームメンバーと重ねていきました。こうしてまずはやってみるという意思決定に至ることができました。

また、コストを低く抑えるという話であれば、テスト分析や設計自体をやめるのではなく、よりコストを低くこれらの取り組みをやっていくためにはどんなやり方があるのか？　ということを考えるようにしました。例えば、先ほど紹介したテスト分析や設計をする際にモブやペアで取り組むことで、個人の知識や経験に依存せずに取り組むことができるようにするのも1つのやり方だと考えています。他にも、そもそもの仕様の複雑さを下げることで、より単純なテスト分析や設計を行うことができるようにしたり、テストに重み付けをすることで、コストをかけるべき部分とそうでない部分を明確にすることも考えられると思います。実際に、これらの取り組みは継続して取り組んでいる最中ではありますが、効果が出てくることを期待しています。

## 結局のところ、何が大事だったのか

私たちのチームで特に意識が変わったなと感じるのは「シフトレフトテストに取り組む」と「テストをフェーズではなくアクティビティとして捉えて継続的に実施する」の2点だと考えています。

シフトレフトテストについて少しだけ解説します。シフトレフトテストとは、ソフトウェア開発プロセスにおいてテストや品質保証活動をプロジェクトの初期段階に前倒しするアプローチのことです。この手法は、開発プロセス全体において品質を向上させることを目的としており、問題を早期に発見し、修正コストとリスクを削減することに貢献します。

伝統的な開発モデルでは、製品の開発がほぼ完成した段階で初めてテストが行われるため、問題の修正に多くの時間とコストが必要とされてきました。しかし、シフトレフトテストでは、要件定義や詳細設計などといった、コードを書くよりも前の段階からテストを組み込むことで、開発の初期段階で問題を検出し対応することが可能です。私は、シフトレフトテストには、以下のような利点があると考えています。

- トータルコストの削減：初期段階のコストは増えるかもしれませんが、総合的に見れば早期に問題を発見することで、修正に必要なコストを大幅に削減できます。
- 開発速度の向上：実装に入る段階での懸念や考慮漏れを減らすことができ、開発速度が向上し、リリースまでの時間を短縮できます。
- 品質の向上：フェーズで区切らずに開発の各段階で品質を確認するため、最終的な製品の品質が向上することが期待できます。

## まとめ

長くなりましたが、ここまで読んでいただきありがとうございます。この記事では、私たちのスクラムチームが直面した品質課題と、それに対する改善活動を紹介してきました。チームに専属のQAエンジニアがいない環境でも、チーム全体で品質課題の改善に取り組むことができることを示すことができたのではないかと考えています。

私がこの経験を通じて最も伝えたいことは、改善活動は特別なスキルがなくても推進できるということです。私自身も品質保証については知識が浅いところからスタートし、チームと一緒に学びながら進めてきました。また、チームに改善を取り入れたい場合には、まずは自分が行動を起こし、小さく始めていくことが大切だと感じています。自分ができることを少しずつ増やしながら、チームにも少しずつ改善を提案していくという動きが今回は特に上手くいったと感じています。もし、課題を感じているけど一歩目が踏み出せないという状況の方がいれば、ぜひこの記事を参考にチャレンジしていただけると幸いです。そして、チャレンジしていく中で困ったことや迷うことがあれば、社内外を問わずに協力を仰ぐという姿勢も大切だと感じています。

もっと詳しく知りたい方や、私たちのチームの取り組みについて質問がある方は、ぜひコメントやDMなどでお気軽にお問い合わせください。最後まで読んでいただき、ありがとうございました。

## 参考書籍・勉強会

本文のなかでご紹介したものも含め、私が参考にした書籍や勉強会をまとめておきます。

https://zenn.dev/loglass/articles/c7af93b861f5bb

https://zenn.dev/loglass/articles/a71a21d8ce992a

https://www.kadokawa.co.jp/product/302309001510

https://www.juse-p.co.jp/products/view/934

https://gihyo.jp/book/2019/978-4-297-10506-8

https://wacate.jp/

https://event.shoeisha.jp/cza/agile-testing

https://speakerdeck.com/hc0208/how-bringing-in-a-scrum-master-to-an-already-self-organized-team-totally-worked-out

https://techbookfest.org/product/wCDWvUCj8m07U6vuvv3YSu?productVariantID=cjZ9wf7zmExGysp6bZL6Nw

## 宣言

最後に少しだけ宣伝ですが、ログラスではエンジニアを大募集していますので、ご興味がある方はぜひ以下もご覧いただければと思います。

https://www.loglass.co.jp/recruit/for-engineers

いきなり申し込むのはちょっとハードル高いなーという場合は、真面目な話も不真面目な話にも対応した私のPiitaにご応募お待ちしています。

https://pitta.me/matches/CMegIkhpGqgD

https://pitta.me/matches/sRRdWLqHjuQD