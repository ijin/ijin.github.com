---
layout: post
title: "AWS SSD (hi1.4xlarge) vs Fusion-IOでのMySQLベンチマーク"
date: 2013-02-22 17:32
comments: true
categories: 
- aws
- mysql
---

（※ [追記](#add)しました - 5/19/13）

巷ではMySQL 5.6 GAが出て騒がしいですが、ちょっと前に5.5系でAWSのSSDインスタンス（hi1.4xlarge）に載せ替える案件があったので、その時に取ったベンチマークを公表します。以前Fusion-IO (ioDrive Duo)でも同じようにやったので、比較になれば。

## 経緯 ##

- あるウェブサービスのDBサイズが巨大でm2.4xlargeでも辛くなってきている
- アクセスパターンによりパーティショニングが効かない
- シャーディングをするにはアプリ改修が大変
- 数週間後に急激なアクセスが予想され、時間的余裕がない！
- データサイズの急激な増加によりbuffer poolから溢れ、ディスクアクセスのさらなる発生が懸念

というわけで、時間がないのでSSDへの移行を検討し、ベンチマークを取りました。

## ベンチマーク ##

buffer poolが徐々に足りなくなった場合のディスクアクセスの発生をシミュレート

- sysbenchのoltpモード
- データサイズは12G（5000万件）
- readonly
- uniform（フルスキャン）

主要my.cnfパラメータ

     sync_binlog = 0
     innodb_buffer_pool_size = XXG
     innodb_flush_method = O_DIRECT
     innodb_flush_log_at_trx_commit = 1
     innodb_file_per_table
     innodb_io_capacity = 2000

コマンド

     time sysbench --test=oltp --oltp-table-size=50000000 --db-driver=mysql --mysql-user=root prepare                                                                                                         
     time sysbench --test=oltp --oltp-table-size=50000000 --db-driver=mysql --mysql-user=root --num-threads=16 --max-requests=0 --max-time=180 --init-rng=on --oltp-read-only=on --oltp-dist-type=uniform 2>&1 run                                                                                                         


## 結果 ##

トランザクション推移

{% img https://docs.google.com/spreadsheet/oimg?key=0Aliw9SoXFJNMdFhhcHJkcDA5MGlackNHTXlPcWt0VWc&oid=2&zx=ii4lryrf8pz8 %}

レスポンスタイム推移

{% img https://docs.google.com/spreadsheet/oimg?key=0Aliw9SoXFJNMdFhhcHJkcDA5MGlackNHTXlPcWt0VWc&oid=3&zx=c2drap7b5561 %}

Fusion-IOと比べて半分ぐらい。ioDrvie Duoの公称IOPSが250,000+ IOPSでhi1.4xlargeが120,000 IOPSなので、まあ合致しますね。

まだ整理されてないベンチマーク結果があるので、後ほど追記しようと思います。

## 追記 <a name="add">&nbsp;</a> ##

先日（5/17/13）の[cloudpack night #6 - Re:Generate -](http://www.zusaar.com/event/668006)でADSJの荒木さんの発表でMySQLのパフォーマンスの話があったので以前[tpcc-mysql](https://code.launchpad.net/~percona-dev/perconatools/tpcc-mysql)でベンチマークを取った資料を思い出し追記しました。

- 500 warehouses (50GBぐらい)
- 24GB Buffer pool
- 16スレッド
- 1時間実行

コマンド（ロードはかなり時間がかかるので注意）

     tpcc_load localhost tpcc root "" 500
     tpcc_start -d tpcc -u root -p "" -w 500 -c 16 -r 300 -l 3600

{% img https://docs.google.com/spreadsheet/oimg?key=0Aliw9SoXFJNMdFhhcHJkcDA5MGlackNHTXlPcWt0VWc&oid=4&zx=y6kwyezb1qth %}

hi1.4xlargeの方が安定するまで少し時間がかかってます。<br />
以下、安定化しだした900sあたりからの数値です。

|Fusion-IO|hi1.4xlarge
-|-|-
total|2288758|1444417
avg|8445.6|5330.0
stddev|245.7|132.8

Fusion-IOに比べて6割ってところでしょうか。
今回はSSDのephemeral disk1本で試したので、RAID0にするともうちょっと違ってくると思います。
