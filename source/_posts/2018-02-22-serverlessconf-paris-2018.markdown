---
layout: post
title: "ServerlessConf Paris 2018で発表してきた"
date: 2018-02-22 02:29
comments: true
categories: 
- events
- serverless
---

既に世界数カ所の都市で行われている[ServerlessConf](http://serverlessconf.io)がフランスのパリで開催されるという事でCFPに応募したら通ってしまったので[発表](http://paris.serverlessconf.io/speakers.html)してきました。75個の応募の中から選んで頂いて光栄ですね。

タイトルは**Accelerating DevOps with Serverless**で、DevOpsを手助けする為にServerlessを使って解決したストーリーや仕組み。

{% speakerdeck b0bc56c91fa54b1eadde7e7d2b5c169f %}

---

内容としてはシンプルでgithubのbranchに応じて、テスト環境用のECSクラスタ内にrandom portでALBにtarget group, listenerを割り当ててservice作ってコンテナを配置するだけです。branch delete時にはtaskを停止し、逆順に削除していきます。通知はSNSでslack等へ連携可能。

少し特色があるとすれば、serviceを作成したりtaskを停止するのにかなり時間を要する場合があるので、単一のlambdaではなく[step functions](https://aws.amazon.com/step-functions/)で制御した事ですかね。今回、[serverless framework](https://serverless.com)の[step functions plugin](https://github.com/horike37/serverless-step-functions)のおかげで[state machine data](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-state-machine-data.html)をjsonで書かなくて済んだので作者の[horike](https://twitter.com/horike37)さんには感謝です。

{% img https://lh3.googleusercontent.com/FdRvgy-OwOnIN0ouQWsNV-JeiBURfOujH49arMUpqkCZZTBnYRcpUTuO1xwc4fvHQqCcgzDpyWIe-yUkMh55_GUGMJaotRqhjdj7QfVi7dBv4UyJOF9A4WCggIlouYDDtCxHujluaw=w600-h346-no %}

発表した内容のモノは現在、複数プロジェクトで使えるようになるべく汎用的な作りにして**ECS deity**として公開しています。参考にでもなれば。

[https://github.com/ijin/ecs-deity](https://github.com/ijin/ecs-deity)

尚、GitHub eventによる自動デプロイだと毎回煩わしい場合もある（特定のbranch prefixが対象であるにせよ）ので、slackによるchatops手動デプロイも実装中です。

と、まあやってる事はそんなに難しくないので、海外カンファレンスのCFPがあれば誰でも気軽に応募してみると良いと思います！

### Afterthoughts ###

やはり、serverlessだとログ周りが様々なCloudWatch logsのlog streamsに分散（api gw, lambda, step functions）しているので、トレーシングがしんどかったりしたけど今回のConfで何度も取り上げられたトピックが**Obervability**だったので皆同様に苦労しているんだなぁと感じました。ブースにもその解決に取り込もうとしている企業がいくつかいたので、今後に期待ですね。

この辺は一緒に行った[yoshidashingo](https://twitter.com/yoshidashingo)さんが[ブログ](http://yoshidashingo.hatenablog.com/entry/serverlessconfparis18)でよく考察してくれています。

後、Keynote Speakerの[Yan Cui](https://twitter.com/theburningmonk)さんのトークが重厚で実に素晴らしかったのでYouTubeに上がったら是非観てください。内容もテンポもスライドも勉強になります。

### 最後に ###

- 海外カンファレンスだと世界中からのエンジニアと話せるので非常に良い刺激になりますよ（可能なら発表を！！）
- 2月のパリは寒い！行くなら観光客が少なく暖かい6月をオススメされた
- ご飯はどこ行っても美味しい！

{% img https://lh3.googleusercontent.com/vcaTvpcnns1YSEmeRfwgBG3Mtsgfv_dJC0UgksXLE7usl7y4lInNrkJKiRkrsPUwdkkby8zrxs-wIZhSngHL7kQlg3c6tSd1HRG849GzQdnXFlYB1FHtu4_Pb9wVYANNR-NffEBshw=w675-h900-no %}
{% img https://lh3.googleusercontent.com/HFNnHhADFLYTxW1V5qyUqRtr762bCGbeDsCBXbt6Lov_W1tSBAWrlp96IEWFt2hj6WTFa70R6nIVlGiGUq-RDVSOuRBpH3u0zkEQ5RtrVVYjmAAoJdbC9qQOCCjJg4cH2oZ2GK1TZw=w675-h900-no %}

{% youtube qcKKwuVJQPk %}
