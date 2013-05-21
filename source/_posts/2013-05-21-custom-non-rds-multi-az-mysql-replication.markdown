---
layout: post
title: "Non-RDSなオレオレMulti-AZ MySQL Replication"
date: 2013-05-21 18:03
comments: true
categories: 
- aws
- mysql
- events
---


先日（5/17/13）[cloudpack night #6 - Re:Generate -](http://www.zusaar.com/event/668006)に参加してきました。

当日の様子はcloudpack[吉田](https://twitter.com/yoshidashingo)さんの以下のレポートで。<br />
[cloudpack Night #6 - re:Generate - を開催しました]( http://d.hatena.ne.jp/yoshidashingo/20130518/1368853720)

私はDJを少々とLTで参加させて頂いたのでその内容になります。

## オレオレMulti-AZのススメ ##

AWS上でMySQLを使う場合、RDSはてっとり早くて良いんですが、たまにもうちょっと柔軟性が欲しい時があります。
例えば別のストーレジエンジンやディストリビューションを使ったり（Percona Server, MariaDB, TokuDB, Mronnga等）、RDSでは使えないインスタンスファミリー（hi1, m3, c1系）を使ったり、OSレベルでのチューニングができたり、スレーブのバッファプールを予め温めておいたり。等々。

しかし、RDSにはAvailability Zone (AZ)をまたいでフェールオーバーするMulti-AZ機能があり、AWSで設計するにはAZ障害を考慮した方が推奨されます。

そこで、MHAとVPCを組み合わせて柔軟性をもったMulti-AZ環境を実現します。
（ちなみに私の場合はhi1.4xlargeでPercona Serverを冗長化をする必要があったから）

### MHA ###

[MHA](https://code.google.com/p/mysql-master-ha/)とはFacebookの[松信](http://yoshinorimatsunobu.blogspot.jp/)さんがDeNA時代に作ったMySQLの自動フェールオーバーしてくれうナイスなツールです。Master障害時にbinlog同期とSlaveの昇格を全自動でやってくれます。昇格時にはカタログデータベースに更新をかけて新DB構成のIPの情報を更新するか、Virtual IPを切り替えるかをする必要（この担当部分はmaster_ip_failover_scriptで対応）があるけど、今回は後者的なアプローチになります。

### VPC ###

ENIを使えばVirtual IP的な使い方はできるけどAZは超えられないので、source/dest. checkを無効化した上でrouting tableによって擬似Virtual IPを実現する[Routing-Based HAパターン](http://aws.clouddesignpattern.org/index.php/CDP:Routing-Based_HA%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3)（CDP 2.0候補）を使います。ADSJ[片山](https://twitter.com/c9katayama)さんの[エントリ](http://d.hatena.ne.jp/c9katayama/20111225/1324837509)が発端ですね。

## Demo ##

以下、LT時に見せたデモ動画

{% youtube tovb29K6ddc %}

擬似Virtual IPに対してそれぞれread/writeを行いつつMasterのプロセスをkillすると、通信できなくエラーが出続けるが20秒以内に自動フェールオーバーが完了しread/write共に再開します。pingも平均0.5msから2.0msに変わった事からAZが移った事が確認できます。

RDSのMulti-AZの場合、フェールオーバーには3-6分かかるので、かなり短時間で復旧が可能です。
また、RDSはCNAME切替によるDNSベースに対して、MHA+VPC構成の場合はIP指定することができます。そうするとアプリからのresolveが不要になり、去年起こった内部DNSが引けない障害が起きても問題ありません。

## 注意点 ##

- RDSな自動バックアップがない
  - Xtrabackupとbinlogの定期s3保存で対応
- Point in Timeリカバリー
  - Chef等で自動化しましょう
- Read Replicaの作成
  - Chefで頑張りましょう
- 学習曲線
  - 勉強しましょう
- API backplaneがSPoF
  - AWSに祈りましょう

特に一番の懸念点は最後のAPI backplane。AWSの今までの大規模障害状況を見ていると、皆が同時に復旧をしようとしAPIへのリクエストが大量に集中してそこがボトルネックになり、リソースの操作不能に陥るという悲惨な事象が何回かありました。まあ、その場合はRDSでも同じような気はしますが、ここは当時の障害を経験にキャパシティが増加されている事を信じておくしかありませんね。。

## おわりに ##

とまあ、こんなLTをしたわけですが、この後に続いたCookpadの[菅原](https://twitter.com/sgwr_dts)さんの[LT](http://www.slideshare.net/winebarrel/ec2keepalivedlvsdsr)の方が盛り上がって自分のは余興に終わってしまいました。

あ、ついでにその日はcpniteの資料作成やDJの選曲であんまり寝てなかったにもかかわらず、無事AWSソリューションアーキテクト（Associate）の認定試験に受かりました！

{% img https://lh5.googleusercontent.com/-d_HVsb6DgBc/UZs39XB1KGI/AAAAAAAAAuw/ntgPp33hcOI/w294-h120-no/Solutions-Architect-Associate.png %}

## LTスライド ##

{% slideshare 21341276 %}

