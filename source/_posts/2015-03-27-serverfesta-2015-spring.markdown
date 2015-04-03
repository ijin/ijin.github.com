---
layout: post
title: "サバフェス！2015に参加してきた"
date: 2015-03-27 12:56
comments: true
categories: 
- events
- study
---

[サバフェス！2015 Spring](http://connpass.com/event/11571/)に参加してきました。
やった内容を少々。

[前回](/blog/2013/12/13/serverfesta-2013-autumn)と同じくチーム@ijinとして1人で参戦して、順位は「**4位**」。スコアは**44988.867 TpmC**でした。

## お題 ##

Mysql on ioDriveでtpcc-mysqlベンチマークのtransaction throughput競争。

第1陣と第2陣に分かれていて、今回は後者での参戦。

--

競技期間は5日間あったものの、第1陣で[結構な地雷があった](http://netmark.jp/2015/03/svfes-2.html)のと、Fusion-IO（現SanDisk）のioDriveとtpcc-mysqlは2年前に[触った](/blog/2013/02/22/mysql-benchmarks-on-aws-ssd-vs-fusion-io/)ので最初はあんまりやる気が起きなくて困ってました。後は前回と違って施せる施策がかなり限定されるというので、正直5日間は長過ぎるのではないかという印象でした。（結果的に第1陣のトラブルとかを鑑みると長さ的には良かったのかも知れないけど）

という事で初日はログインとベンチで基準値を取るだけして終了。

その後、最近のMySQLやioDrive周りの譲歩収集を軽ーくして数日が経過。。

そして最終日の夜中になって、なんとか本腰を入れてチューニング開始。まあ、やりだしたら楽しいんですけどね。

## マシンスペック ##

- CentOS 6.4
- Intel(R) Xeon(R) CPU E5-2650 v2 @ 2.60GHz x 32
- 32GB RAM
- 40G HDD
- 320GB ioDrive

## 制限 ##

    innodb_doublewrite = 1
    innodb_flush_log_at_trx_commit = 1

その他レギューレーションは[こちら](https://2015spring.serverfesta.info/?page_id=299)

## 方針 ##

今までのtwitterの[タイムライン](https://twitter.com/search?q=%23%E3%82%B5%E3%83%90%E3%83%95%E3%82%A7%E3%82%B9)からして、どうもベンチをウェブ画面から要求しても並列実行の数とキューイング、及び非常に長い実行時間からして1時間に一回実行できるかどうかも怪しかったのでローカルでこまめに回す方向に。

本番データはtpccの1000 warehouse（70GB+）、16GB Buffer Pool、900sの実行時間だったのでそれを250 warehouse, 4GB Buffer Poolと縮小し120sと短めに設定。こうすることによってローカルでのデータコピー時間やベンチ実行時感が短縮され、時間がない中での細かいパラメータのチューニングに専念できる。Buffer Poolはどうせ一番効くと分かりきってるので、あえて後回し。DBは6年前（MySQL 5.0時代）から手に滲んで愛用しているPercona Serverの5.6版を選択。

スコアの推移はこんな感じ。データセットが小さい分、スコアは大きめ。

{% img https://docs.google.com/spreadsheets/d/1zm6-THRYR_EcLoHgacRScxlhSzeQAkbJ0RXiAI4fiRU/pubchart?oid=1067951178&amp;format=image %}

終盤になってやっと本番に近いデータセットや実行時感で細かいチューニング。

{% img https://docs.google.com/spreadsheets/d/1zm6-THRYR_EcLoHgacRScxlhSzeQAkbJ0RXiAI4fiRU/pubchart?oid=2026636320&amp;format=image %}

最終的にローカルでのベストスコアは**53420.332 TpmC**でした。

## 設定ファイル ##

{% gist 341bab7569e372e1addb %}

## 効果があったもの ##

細かいおまじないレベルでのパラメータも他にいくつあったけど、割と効いたのをピックアップ。また、既に設定されていたパラメータは除外（innodb_io_capacity等）

mysql
    datadir=/fioa/mysql
    tmpdir=/fioa/tmp
    sync_binlog=0
    innodb_buffer_pool_size = 28150M
    innodb_buffer_pool_instances=16
    innodb_log_file_size=4G
    innodb_log_files_in_group=3
    innodb_log_group_home_dir=/var/log/mysql
    innodb_log_buffer_size=64M
    innodb_data_file_path=ibdata1:76M;../../var/log/mysql/ibdata2:500M:autoextend
    innodb_checksum_algorithm=0
    innodb_max_dirty_pages_pct=90
    innodb_lru_scan_depth=2000
    numa_interleave=1
    flush_caches
    malloc-lib=/usr/lib64/libjemalloc.so.1

etc
    vm.swappiness=1
    mount option (noatime,nodiratime,  max_batch_time=0,nobarrier,discard)
    cache warmup

基本的にはioDriveにIOをなるべく発生させない、もしくは遅延させる関連のパラメータが効いた感じ。この辺は割と正統なチューニング方法。一つ、若干工夫したのは、doublewrite buffer fileの指定方法。シーケンシャルなIOが発生するログ周りの処理はHDDに逃がした方がioDrive/SSD的には負荷低減になるけど、Percona 5.5までは**innodb_doublewrite_file**というパラメータでいつも指定していたのが、5.6ではなんと未実装！なのでベンチマーク前にコピーされるibdata1の予めのサイズを計っておいて、以降の書き込みをHDD側へ逃がすように**innodb_data_file_path**で調整。

他はNUMAによるメモリの偏り調整とmallocライブラリを使う指定。この辺もPerconaやMariaDB専用オプション。

後は一応初動のスコアをちょっとだけ稼ぐためにmysqlのstartup script内にてテーブルをcount(*)してindexをbuffer poolに乗せたぐらい。

## 効果が微妙だったもの ##

mysql
    innodb_flush_method=ALL_O_DIRECT
    innodb_support_xa=0
    innodb_thread_concurrency=N
    innodb_flush_neighbors=0
    query_cache_size=0
    large-pages
etc
    echo 'noop' > /sys/block/fioa/queue/scheduler
    echo 4096 > /sys/block/fioa/queue/nr_requests
    echo 2 > /sys/block/fioa/queue/rq_affinity
    renice -n19 -p `ps auxf | grep mackerel | grep -v grep | awk ‘{print $2}’`

large-pagesでページサイズを大きく設定すればメモリ効率は向上するはずが、少なくともベンチマークにおいては効果なし。また、スケジューラをnoopやdeadlineと変えたり、nr_requestsやrq_affinityを調整してみたけど、デフォルトのOS bypass設定と比べてあまり変化なし。

## 試したかったもの ##

- ioDrvieのblock sizeのリサイズ
- C0 state（CPUのC stateを制御して変動させずにioDriveの処理向上の期待）
- numa_node_forced_local（IO処理をメモリに一番近接しているCPUで行う）
- IRQ pinning（ioDriveのIRQを固定）
- network tuning

この中で最後やることリストに乗っけていながらやらなかったネットワーク周りのチューニング。多分、これが敗因。（1位のチームzzz([@ttkzwさん](https://twitter.com/ttkzw))はローカルベンチマークで51000ぐらいだったので）もっとリモートからベンチが実行される事に意識を向ければ良かったのかもしれないですね。とはいえ、最後はキュー待ちが8チームだったり、結果が不具合で見れなかったりと、バダバタしてたのでそっちに気を取られたのもまた事実。まあ、半日ちょいのチューニングにしてはそこそこ行った感じでしょうか。

## 終わりに ##

最初は運営側が「目下実装中です！」といいながら**#トラしゅ**をしているような感じでちょっと不安でしたが、最後は（第2陣という事もあり）それなりに楽しめました。ベンチマーク時間の短縮と並列実行数がもうちょっと増やせたらよかったですかね。運営側の皆様、大変お疲れさまでした！

## おまけ ##

賞品として3DマッサージピローとSSDを頂きました！

{% tweet https://twitter.com/ijin/status/581048257171820544 %}

（個人敵には飛び込みLTした人がもらったDroneの方が良さそうだったけど、ピローは家族に好評でした。）


<blockquote class="twitter-tweet" lang="en"><p>3DマッサージピローとSSDをもらった。 <a href="https://twitter.com/hashtag/%E3%82%B5%E3%83%90%E3%83%95%E3%82%A7%E3%82%B9?src=hash">#サバフェス</a> <a href="http://t.co/mKczn9mflZ">pic.twitter.com/mKczn9mflZ</a></p>&mdash; Michael H. Oshita (@ijin) <a href="https://twitter.com/ijin/status/581048257171820544">March 26, 2015</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>
