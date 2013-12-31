---
layout: post
title: "i2 instanceでMySQLベンチマーク"
date: 2014-01-01 00:00
comments: true
categories: 
- aws
- mysql
---

新年明けました。おめでとうございます。

すっかり12月の[aws](http://www.zusaar.com/event/1117005)/[mysql](http://www.zusaar.com/event/1847003) advent calendarに乗り遅れたので、AWSのi2 instanceでのMySQLのベンチマークを勝手におまけとして公表します。
以前取ったhi1.4xlargeとの比較になります。


## 構築 ##

SSDディスクはAWSが[推奨する](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/i2-instances.html#i2-instances-diskperf)over-provisioningを有効にする為に10%を非partitionし、それぞれ720Gでパーティション作成。

    sudo mdadm --create /dev/md0 --level 0 --raid-devices 8 /dev/xvdb1 /dev/xvdc1  /dev/xvdd1 /dev/xvde1 /dev/xvdf1 /dev/xvdg1 /dev/xvdh1 /dev/xvdi1
    sudo mkfs.xfs -f -b size=4096 -i size=512 -l size=64m /dev/md0
    sudo mount -t xfs -o noatime,logbufs=8 /dev/md0 /data

OSはkernel versionが3.8以上が[望ましい](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/i2-instances.html#i2-instances-diskperf)のでいつものLTSではなく、Ubuntu Server 13.10のHVMタイプ (ami-b93264d0)

## sysbench ##

やり方やパラメータは[前回](//blog/2013/02/22/mysql-benchmarks-on-aws-ssd-vs-fusion-io/)の計測方法と同じ。i2.8xlaregは32coreですが、今回はhi1.4xlargeの時と比較する為に敢えて16スレッドで計測しました。

- sysbenchのoltpモード
- データサイズは12G（5000万件）
- readonly
- uniform（フルスキャン）

コマンド

	time sysbench --test=oltp --oltp-table-size=50000000 --db-driver=mysql --mysql-user=root --num-threads=16 --max-requests=0 --max-time=180 --init-rng=on --oltp-read-only=on --oltp-dist-type=uniform 2>&1 run

トランザクション推移

{% img https://docs.google.com/spreadsheet/oimg?key=0Aliw9SoXFJNMdFhhcHJkcDA5MGlackNHTXlPcWt0VWc&oid=5&zx=jwv2ytp6xwx3 %}

レスポンスタイム推移

{% img https://docs.google.com/spreadsheet/oimg?key=0Aliw9SoXFJNMdFhhcHJkcDA5MGlackNHTXlPcWt0VWc&oid=6&zx=sfl6ocblw5ob %}

速いですね。

## tpcc-mysql ##

こちらも[前回](//blog/2013/02/22/mysql-benchmarks-on-aws-ssd-vs-fusion-io/))の計測方法と同じ。

- 500 warehouses (50GBぐらい)
- 24GB Buffer pool
- 16スレッド
- 1時間実行

コマンド

     tpcc_load localhost tpcc root "" 500
     tpcc_start -d tpcc -u root -p "" -w 500 -c 16 -r 300 -l 3600

{% img https://docs.google.com/spreadsheet/oimg?key=0Aliw9SoXFJNMdFhhcHJkcDA5MGlackNHTXlPcWt0VWc&oid=7&zx=fc6nz8iel3ez %}

hi1.4xlargeはSSD1台で計測した事を考えると、同価格帯のi2.4xlarge（SSD4台）の半分（SSD2台）のパフォーマンスが出るのは妥当ですね。

## fio ##

ついでにfioでそれぞれRAID0した場合のベンチマークも取ってみたけど、[並河さん](http://d.hatena.ne.jp/rx7/20131224/p1)と結果が違ってwriteがスケールしてます。OSとmkfs.xfsのオプションしか違わないはずだけど。。

{% img https://docs.google.com/spreadsheet/oimg?key=0Aliw9SoXFJNMdFhhcHJkcDA5MGlackNHTXlPcWt0VWc&oid=9&zx=v7nbywda04bp %}

いやー。spot instanceがないので計測だけで結構かかってしまった。
