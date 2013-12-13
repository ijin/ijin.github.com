---
layout: post
title: "サバフェス！2013に参加してきた"
date: 2013-12-13 15:59
comments: true
categories: 
- events
- study
---

[tuningathon]: https://www.facebook.com/tuningathon
[isucon]: http://isucon.net 

少し前に[サバフェス！2013 Autumn](http://connpass.com/event/3690/)に参加してきました。
内容を忘れないうちにやった事を書いておきます。

スコアはトップレベルだったものの、運営側がサーバを起動した所、自動でサービスが立ち上がらなかったらしいので残念な事に参考値のみに。（提出前に2回ぐらい再起動確認したのにおかしいなぁ。。）
優勝スコアは90.830 (GET 199,802 : POST 18,106)で、私のは**93.410** (GET 162,923 : POST 29,930)でした。

## お題 ##

「最速インフラを構築せよ！！！

WordPressに一切手を加えずに、どこまで高速化できるのか！？

OSチューニング、サーバチューニング、負荷分散…最適解を探せ！」

--

[IDCFクラウド](http://www.idcf.jp/cloud/service/self.html)上で仮想マシン5台（M8タイプまで）を使ってスコアを競うというもの。
[チューニンガソン][tuningathon]と似てますが、サーバが複数台使えるのが良いですね。

## 構成 ##

表彰式でLTしたけど、LVS (DSR) + php 5.5 + apcu + Varnish + nginx (lua) + memcached

ポイントは

- GET時は1台では帯域の限界に達したのでLVS (DSR)による4台での並列応答
- POST時には必ず各Varnishのキャッシュクリア（ban）する
- POST時にnginx->memcachedへ渡すと高速すぎたのであえてsleepを5msして遅延させる
- nginx(lua)はコード量が多いと通らないのでぎりぎりまで削減
- memcachedからmysqlへの非同期処理（5ms間隔）
- mysqlの更新処理はそんなにいらないので基本チューニングとInnoDBにしただけで、5.1のまま
- サーバはM8までいらないのでM4で

でしょうか。特に意識したのはmemcachedからmysqlへの許容範囲内での同期と複数台あるVarnishのキャッシュクリアですね。他のチームはnginxでTTLを設定して逃げたようですが、実運用時にはPOST時にキャッシュクリアを確実にする必要があるのでVarnish moduleをコンパイルして他のsiblingへ並列でban処理を投げてました。まあ、今回のベンチマークツールはそこまで厳格じゃなかったけど、最終チェックは人間が動作させるので。


LT資料。スコア推移と簡単な構成の紹介
{% slideshare 29171459 %}

## 設定ファイル ##

以下、設定ファイルです。いらない所は削ったけど動くはず。。

backend varnish vcl （各backendの設定は微妙に違う）:
{% gist 7941009  %}

backend nginx.conf:
{% gist 7940999 %}

base nginx.conf:
{% gist 7941214 %}

base syncer.rb:
{% gist 7941042 %}

base supervisord.conf:
```
[program:syncer]
command=/home/mho/.rvm/bin/ruby /home/mho/syncer/sync.rb 5
stdout_logfile_maxbytes=1MB
stderr_logfile_maxbytes=1MB
stdout_logfile=/tmp/%(program_name)s-stdout.log
stderr_logfile=/tmp/%(program_name)s-stderr.log
user=mho
directory=/home/mho/syncer
autostart=true
autorestart=true
```

base my.cnf:
```
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
user=mysql
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
character-set-server = utf8
max_connections = 1000
 
key_buffer_size = 32M
max_allowed_packet = 16M
thread_stack = 192K
thread_cache_size  = 200
 
#slow_query_log=1
#long_query_time=0
query_cache_type = 0
skip-innodb_doublewrite
innodb_buffer_pool_size = 192M
innodb_log_buffer_size = 4M
innodb_flush_log_at_trx_commit = 0
innodb_support_xa = 0
```

sysctl.conf:
```
fs.file-max = 1048576

net.ipv4.ip_local_port_range = 1024 65535

net.core.wmem_max = 16777216
net.core.rmem_max = 16777216

net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216

net.core.somaxconn = 8192

net.core.netdev_max_backlog = 8000

net.ipv4.tcp_max_syn_backlog = 8192

net.ipv4.tcp_synack_retries = 3
net.ipv4.tcp_retries2 = 5

net.ipv4.tcp_keepalive_time = 900

net.ipv4.tcp_keepalive_probes = 3

net.ipv4.tcp_keepalive_intvl = 15

net.nf_conntrack_max = 1000000
```


## 終わりに ##

最初はecho選手権になってたのであんまりやる気がなかったけど、後半はちょっと楽しめました。結果は惜しかったけど、今まで[チューニンガソン][tuningathon]や[ISUCON][isucon]に出場したり、トラブル☆しゅーたーずを主催したり、日々の運用等の経験が生きた感じがします。運営の皆様、ありがとうございました。次があれば楽しみにしています！（レビュレーションの解釈が微妙だったのでそこはブラッシュアップして欲しいですね）

