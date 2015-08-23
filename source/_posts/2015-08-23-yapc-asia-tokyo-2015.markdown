---
layout: post
title: "YAPC::Asia Tokyo 2015に参加してきた"
date: 2015-08-23 15:27
comments: true
categories: 
- study
- event
---

例年この時期は海外にいるので、なかなか行けなかったYAPC::Asia Tokyo 2015にやっと参加してきた。

スケジュール上、今年は参加できるぞって意気込んでいたら、まさかのSold Out...

それをtwitter上で嘆いていたら、[@kenjiskywalker](https://twitter.com/kenjiskywalker)さんから救いの手が！

{% tweet https://twitter.com/kenjiskywalker/status/619079064880463872 %}

感謝！


さて、一口コメント

**Consulと自作OSSを活用した100台規模のWebサービス運用**] - [@fujiwara](https://twitter.com/fujiwara)

[mamiya](https://github.com/sorah/mamiya)や[AWS CodeDeploy](http://aws.amazon.com/codedeploy/)にインスパイアされた[stretcher](https://github.com/fujiwara/stretcher)を使ったイベントベースのdeploy周りが面白かった。s3からの取得失敗時の処理は考えてないといけない。

**Conway's Law of Distributed Work** - Casey West

最近はリモートワークが殆どなので、なかなか共感できる部分が多かった。コミュニケーション大事。

**大規模でも小中規模サービスでも捗る microservices な Web サービスのつくりかた** - [@Yappo](https://twitter.com/yappo)

どのタイミングでコンポーネントの分割をするか。ちゃんと設計しようねって話

**ISUCONの勝ち方** - [@kazeburo](https://twitter.com/kazeburo)

ポイントがよく纏められていて、うちらのチームでも多くを実践していた。まだ勝ててないけど。
CPUやB+ Treeの気持ちになって考える事が大事。

**我々はどのように冗長化を失敗したのか** - [@kenjiskywalker](https://twitter.com/kenjiskywalker)

Networkが不安定だと、様々な冗長化に仕組みがうまく作動しない。検証はProductionと同じ環境でやりましょう。

**Profiling & Optimizing in Go** - Brad Fitzpatrick

Bradさんのライブコーディング。奥が深かった。


**LT**

どれもこれもハイレベル！

特に素晴らしかったのが、テンポの良い[@yoku0825](https://twitter.com/yoku0825)さんの**MySQL 5.7の罠があなたを狙っている** と超早業を披露した[CONBU](http://conbu.net)さんの**CONBUの道具箱**


<iframe allowfullscreen="" frameborder="0" height="355" marginheight="0" marginwidth="0" scrolling="no" src="//www.slideshare.net/slideshow/embed_code/key/dJ6c2p0tc7QM3B" style="border-width: 1px; border: 1px solid #CCC; margin-bottom: 5px; max-width: 100%;" width="425"> </iframe> <br />

{% tweet https://twitter.com/spacepro_be/status/635004222291902464 %}


いやー、非常に楽しめました。運営の皆様、ありがとうございました！！

さて、これから出国ー。
