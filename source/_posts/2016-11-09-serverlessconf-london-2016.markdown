---
layout: post
title: "ServerlessConf London 2016に参加してきた。"
date: 2016-11-09 01:36
comments: true
categories: 
- events
- serverless
---


先日、[ServerlessConf London 2016](http://london.serverlessconf.io/)に参加したので、そのメモ（例によってtogetter風）。


少し前に話題の「サーバーレス」アーキテクチャに関するイベントである[ServerlessConf Tokyo 2016](http://tokyo.serverlessconf.io/)をお手伝いしたことから、ロンドン版開催のお誘いが来たので行ってみました。

{% img https://lh3.googleusercontent.com/Uk-VZ2mRZ3DL0wef_4E83hYAFqaoyJmf6trHOZKlWqGur9uLBzCvlG3piGup5OlD2jNFehq3TsCg=w320-h425-no %}

開催場所は**etc.venues**という施設。

{% img https://lh3.googleusercontent.com/QuQBUW9pBQxRt6_k0P4blgtq9dzm9sQhdumHPcJgGZWASNszAq4vAzeGNpD1IUB7_rW04y1-UMGE=w433-h319-no %}

## Day 1 ##

### Keynote by Patrick Debois ###

サーバーレスについてのカンファレンスなのに、キーノートでジャブ撃ちまくりが面白かった。

{% tweet https://twitter.com/ijin/status/791559728715268096 %}
{% tweet https://twitter.com/ijin/status/791561074734604289 %}
{% tweet https://twitter.com/ijin/status/791562423224283136 %}
{% tweet https://twitter.com/ijin/status/791563104345731072 %}
{% tweet https://twitter.com/ijin/status/791563738566451200 %}
{% tweet https://twitter.com/ijin/status/791564385244241920 %}
{% tweet https://twitter.com/ijin/status/791565224239169537 %}
{% tweet https://twitter.com/ijin/status/791565471787003904 %}
{% tweet https://twitter.com/ijin/status/791566041629331456 %}


### Sessions ###

iRobotや[Cloud Custodian](https://github.com/capitalone/cloud-custodian)を公開したCapitalOneのセッションも面白かったけど、一番は元Parseのエンジニアである**Charity Majors**のセッション（**Serverlessness, NoOps and the Tooth Fairy**）。サーバーレスを採用しても結局サービスの責任は自分で負わないといけない啓蒙的な話を様々な例を交えつつ、パッションを持って話してくれた。

- **Serverless architecture at iRobot** by Ben Kehoe 
- **Serverless for the Enterprise** by Rafal Gancarz 
- **Real-time Data Processing Using AWS Lambda** by Ken Payne 
- **Cloud Custodian, a serverless rules engine for the cloud** by Kapil Thangavelu 
- **The Serverless Cloud Integration Pattern** by Lars Trieloff
- **Serverlessness, NoOps and the Tooth Fairy** by Charity Majors 
- **Serverless Operations** by Ben Bridts

{% tweet https://twitter.com/ijin/status/791571380621807620 %}
{% tweet https://twitter.com/ijin/status/791573027251257348 %}
{% tweet https://twitter.com/ijin/status/791588644486275072 %}
{% tweet https://twitter.com/ijin/status/791595279661359109 %}
{% tweet https://twitter.com/ijin/status/791624537326649344 %}
{% tweet https://twitter.com/ijin/status/791634525298253826 %}
{% tweet https://twitter.com/ijin/status/791637330742353920 %}
{% tweet https://twitter.com/ijin/status/791638343226392576 %}


### LT ###

LTという割にはミニセッションな感じ。日本のLTみたいな笑い所はあまりなし。最後の火星探査機もどきのやつは少し興味深かったけど。

- **Migrating to Serverless: MindMup 2.0 case study** by Gojko Adzic
- **Serverless with Ansible** by Ryan Brown
- **Cloud Detour - Automation Tool for testing Cloud Applications Resiliency** by Sathiya Shunmugasundaram & Gnani Dathathreya
- **Mission to Mars: Exploring new worlds with AWS IoT** by Jeroen Resoort

{% tweet https://twitter.com/ijin/status/791677468252004352 %}
{% tweet https://twitter.com/ijin/status/791688150833852416 %}
{% tweet https://twitter.com/ijin/status/791693673222340608 %}


### After Party ###

{% tweet https://twitter.com/ijin/status/791695401632010241 %}

## Day 2 ##

### Sessions ###

興味深かったのはnewrelic apm風なlambda専用の解析ツール[IOPipe](https://www.iopipe.com/)を作ってる人による**Ops for NoOps**や[Realm](https://realm.io/)のFounderによる**Serverless in an Offline-First world**。どちらも自社プロダクトを紹介しつつも結構サーバーレスアーキテクチャでの設計やオペレーションを深掘りした話だった。他は[Zappa](https://github.com/Miserlou/Zappa)の作者による**Globally Available Serverless Architectures**がテンポよく進んで見てて飽きなかった。

- **Ops for NoOps: Operational Challenges for Serverless Apps** by Eric Windisch & Pam Selle
- **Serverless in an Offline-First world** by Alexander Stigsen 
- **The future of serverless** by Paul Johnston
- **Getting the most out of the Serverless Framework** by Florian Motlik 
- **Serverless API in the Enterprise** by Scott Patterson & Simon Coward 
- **Globally Available Serverless Architectures** by Rich Jones 
- **Serverless Microservices with Google Cloud Functions** by Bret McGowen

{% tweet https://twitter.com/ijin/status/791916637762957312 %}
{% tweet https://twitter.com/ijin/status/791919435934146560 %}
{% tweet https://twitter.com/ijin/status/791926072568713216 %}
{% tweet https://twitter.com/ijin/status/791928441410981888 %}
{% tweet https://twitter.com/ijin/status/791940529365790720 %}
{% tweet https://twitter.com/ijin/status/791942085750292480 %}
{% tweet https://twitter.com/ijin/status/791958743755788289 %}
{% tweet https://twitter.com/ijin/status/791973348716797952 %}
{% tweet https://twitter.com/ijin/status/791975593407574016 %} 
{% tweet https://twitter.com/ijin/status/791986231051685888 %}
{% tweet https://twitter.com/ijin/status/791988939729100800 %}

### LT ###

MSの人達によるAzureの高速棒読み寸劇が笑えた。

- **Why PubNub moved Serverless Computing into the Network** by Girish Dusane 
- **DevOps for Serverless Applications** by Yochay Kiriaty 
- **Writing Serverless Plugins** by Anna Doubkova 
- **Serverless real-time aggregates** by Christian Blunden
- **Big data processing simplified with Serverless Map Reduce** by Sunil Mallya 

{% tweet https://twitter.com/ijin/status/792021382066479104 %}
{% tweet https://twitter.com/ijin/status/792021542867705857 %}
{% tweet https://twitter.com/ijin/status/792027607088918528 %}
{% tweet https://twitter.com/ijin/status/792033437452472325 %}
{% tweet https://twitter.com/ijin/status/792035672366452736 %}
{% tweet https://twitter.com/ijin/status/792036230204624896 %}
{% tweet https://twitter.com/ijin/status/792036685513101312 %}


### 感想 ###

セッションもまあまあ面白かったりしたけど、[Serverless Framework](https://serverless.com/)等各種ツールの作者と直接話したり、各国のエンジニアとサービス運用について細かく議論できるのが有意義でした。ヨーロッパ圏の人がメインだけど、意外とアメリカからも沢山来てて終始盛り上がりました。日本からはServerlessConf Tokyoの主催者のセクションナイン[吉田さん](https://twitter.com/yoshidashingo/)や[もっぱらさん](https://twitter.com/toricls/)等が来英した他、クラスメソッドのベルリンリージョンにいる吉田さんがわざわざ[メソりに](http://dev.classmethod.jp/series/serverlessconf-london/)来てて、ノルマ大変そうだなぁと思ったり。

後、LTに関しては日本の方が絶対レベルが上だと思う！自社プロダクトの紹介とかやめて。。

### Videos ###

ちなみにセッションの動画は[ここ](https://www.youtube.com/channel/UCqlcVgk8SkUmve4Kw4xSlgw/feed)から見れます。


### おまけ###

カンファレンスの雰囲気。インタビューされちゃいました。

{% youtube bl1IuuqBwo4 %}
