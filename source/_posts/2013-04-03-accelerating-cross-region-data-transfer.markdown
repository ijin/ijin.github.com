---
layout: post
title: "リージョン間高速データ転送"
date: 2013-04-03 9:05
comments: true
categories: 
- cdp
- aws
---

[tsunami-udp-link]: http://tsunami-udp.sourceforge.net
[cloudopt-link]: http://www.cloudopt.com

先日の[JAWS DAYS](http://jaws-ug.jp/jawsdays2013/)のセッション[「Behind the scenes of Presidential Campaign」](http://www.publickey1.jp/blog/13/obama_for_america.html)でリージョン間のデータ転送を高速化するツールとして[tsunami udp][tsunami-udp-link]と[cloudopt][cloudopt-link]を使った話が出てたので試してみました。

通常、遠距離のサーバはRTTが大きくなるのでスループットが下がり、巨大なデータ転送には苦労しますが、なんと27TBを9時間で転送したとのこと！実際はマシンを並列にして同時転送したらしいので1台での実験です。

### 前提 ###

- 東京(server1) -> アメリカ西海岸(server2)
- RTT: 120msぐらい
- EC2 instance type: m1.large
- OS: Ubuntu 12.04

### ベーステスト ###

10Gファイルの作成
	server1$ mkdir _tmp; cd _tmp
	server1$ dd if=/dev/zero of=10G count=1 bs=1 seek=10G

転送
	server1$ scp -rp 10G server2:
	10G		100%   10GB  11.1MB/s   15:22


だいたい、90Mbpsぐらい。

### Tsunami UDP ###

インストールは両サーバで

	sudo apt-get install make gcc autoconf
	wget http://downloads.sourceforge.net/project/tsunami-udp/tsunami-udp/tsunami-v1.1-cvsbuild42/tsunami-v1.1-cvsbuild42.tar.gz
	tar xvfz tsunami-v1.1-cvsbuild42.tar.gz
	cd tsunami-udp-v11-b42
	make
	sudo make install

tsunami udpはfetch型の作りなので、送信側のサーバ（server1）で対象ファイルのあるディレクトリに移動してサービス起動

	server1$ cd _tmp
	server1$ tsunamid *

で、クライアント(server2)からファイルをfetch

	server2$ tsunami connect ec2-xx-x-xx-x-x get 10G quit

本当はftp-likeは対話型クライアントだけど、ワンライナーでも可能。	
転送が完了するとクライアント側で情報が出力されます。

	Transfer complete. Flushing to disk and signaling server to stop...
	!!!!
	PC performance figure : 224947 packets dropped (if high this indicates receiving PC overload)
	Transfer duration     : 419.32 seconds
	Total packet data     : 183339.18 Mbit
	Goodput data          : 181958.17 Mbit
	File data             : 81920.00 Mbit
	Throughput            : 437.23 Mbps
	Goodput w/ restarts   : 433.93 Mbps
	Final file rate       : 195.36 Mbps
	Transfer mode         : lossless

おお、確かに速い！スループットもセッションでも言ってた476Mbpsに近いし。

サーバ側では

	Server 1 transferred 10737418241 bytes in 419.33 seconds (195.4 Mbps)


ファイル自身の実際の転送レートはFinal file rateである**195.4Mbps**。
多分、パラメータチューニングやより速いディスクを使うとスループットはさらに向上すると予想。

### cloudopt ###

次は圧縮、TCP最適化、data deduplication等、様々な技術を用いてWANを高速化する[cloudopt][cloudopt-link]の実験。こちらは有料で、ソフトウェアをインストールしてライセンスを購入（15日間のお試しあり）して使う方法とAmazon Marketplaceでセットアップ済みの課金型AMIを起動する方法がとれます。今回はてっとり早く後者で。

AMIはMarketplaceでCloudoptを検索し、各リージョンで1台づつ起動。

{% img https://lh5.googleusercontent.com/-7PWSb_8ClIo/UVuL_bzBeAI/AAAAAAAAAtc/8Lw_bDnaPVE/s665/cloudopt_marketplace_+2013-03-30+at+2.08.07+PM.png %}

Ubuntuベースなのが良いですね。

まずscpから呼べるcloudcopyを使っての転送
	server1$ scp -rp -S cloudcopy 10G server2:
	10G					100%   10GB  41.5MB/s   04:07     
	CloudCopy transferred 17.22 MB, saving 99.8% of bandwidth by sending 9.983 fewer GB 

お、速い。しかしよく見てみると17.22MBしか転送されてません。どうやらファイル自体が全てゼロ埋めなので圧縮とdeduplicationが最大限効いているみたい。

では、今度は実際のDBのバックアップで転送実験（容量14GB）
	server1$ scp -rp -S cloudcopy dbbackup.tar server2:
	dbbackup.tar				100%   14GB  11.2MB/s   20:42   
	CloudCopy transferred 6.041 GB, saving 55.5% of bandwidth by sending 7.528 fewer GB

スループットはほぼ一緒だけど、転送容量が削減できてます。

一度転送したファイルはbyte cacheされるので、次に転送する時は差分だけ送るので高速化されます。
	server2$ rm dbbackup.tar 

	server1$ tar rvf dbbackup.tar file
	server1$ scp -rp -S cloudcopy dbbackup.tar server2:
	dbbackup.tar				100%   14GB  20.0MB/s   11:35  
	CloudCopy transferred 59.57 MB, saving 99.6% of bandwidth by sending 13.511 fewer GB

	
### リージョン間レプリケーション ###

次はMySQLのレプリケーション実験。

各インスタンスでのピア設定
	server1$ sudo cloudconfig peer_add server2 server2_local_ip(10.x.x.x)
	server2$ sudo cloudconfig peer_add server1 server1_local_ip(10.x.x.x)

これでcloudoptを使ったサーバ間のトンネルが確立されます。MySQLのレプリケーションはprivate ipでの設定が必要。
一通りレプリケーションが出来き、スレーブIOを停止した状態でマスターにしばらく書き込んだのちに再開すると、binlogが転送されるのでcloudstatsコマンドで統計が見れます。

	Component - cloudoptimizer
	------------------------------------------------------------
	           Number of connections:                    1
	              Original data size:            111.96 MB
	           Transferred data size:             52.44 MB
	
                Bandwidth Saving:             59.51 MB (53.2%)

**53.2%**の転送容量削減！

### 結論 ###

以上を組み合わせれば、新たなCDP候補である「**リージョン間レプリケーションパターン (Cross-Region Replication Pattern)**」が実現できます。

- 巨大データをリージョン間で転送するにはtsunami udpが有効そう
- リージョン間での差分バックアップやレプリケーション向けにはcloudoptで高速化
- 普通のHTTP通信とかも使えるかも
- s3へのアップロード高速化も対応しているのでいずれ試してみたい

