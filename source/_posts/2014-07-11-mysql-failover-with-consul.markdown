---
layout: post
title: "ConsulによるMySQLフェールオーバー"
date: 2014-07-11 10:12
comments: true
categories: 
- consul
- mysql
- events
---


先日(6/22/14)、6月なのにどういう分けか早めに開催された[July Tech Festa 2014](http://2014.techfesta.jp/)でConsulについて[発表](http://2014.techfesta.jp/p/program.html?m=1#c43)してきた。そのユースケースの一つとしてMySQL failoverをちょっとだけ紹介したので、ここに詳しく書いておく。


## MHA ##

MySQLレプリケーションの障害時にフェールオーバーしたい場合、MHAを使うの結構ポピュラー（日本では）だと思います。MHAは最新binlogの適用、Slaveの昇格とレプリケーションの張替えまではやってくれますが、実際のフェールオーバーの部分はユーザに委ねられていて、master_ip_failover_scriptのテンプレートをカスタマイズするか独自実装する必要があり、一般的な実現方法としてはカタログデータベースの更新かVirtual IPの切替等があります。

Virtual IPだと居残りセッションの問題や切替の保証難しかったり、そもそも環境によっては使えなかったりするので、私はあんまり好きじゃありません。また以前、後者的アプローチとしてAWSのVPC内であればrouting tableを変更する事によってこの挙動に似た実現方法を[紹介](/blog/2013/05/21/custom-non-rds-multi-az-mysql-replication/)した事がありますが、一番の問題点はAPI backplaneがSPoFになってて、ここが落ちたらそもそも動かない；また、APIのrate limitに達して呼び出しさえ出来ないという結構痛い目に会ったりします。

そこで前者的なアプローチとしてmasterの情報を管理するカタログデータベースの更新部分にConsulを使ってみました。

## CONSUL ##


ConsulとはHTTP APIとDNSで操作ができる分散型クラスタで、VagrantやSerf等を開発しているHashicorpの新プロダクトです。Key featureは以下のとおり。

* Service Discovery
* Failure Detection
* Multi Datacenter
* Key/Value Store

### ConsulのConsistencyについて ###

ConsulはCAP定理でいうCPという特性をもっており、Consistency（一貫性）に重きをおいてあります。Paxosを由来とするRaftをベースにしたconsensus protocolで実現していて、ピアセット内の各サーバノードでlog entryの書き込みがquorumで決定された後にcommitされたと見なされ、FSMに書き込まれます。よって、書き込みに関してはStrongly Consistentな処理となります。読み込みに関しては、パフォーマンス要求に応じてConsistencyレベルをクエリータイプによって調整可能（まるでCassandraのように！）で、Usually Consistent, Strongly Consistent, Staleから選択可能です。DNSベースのクエリーはデフォルトで単一のリーダーノードが返答するのでStrongly Consistentな処理となっています。（Staleにする事も可能）

## ConsulとMHAの連携概要 ##
* Consulを内部DNSとして使い、clientはDNSベースでmasterに接続
* MySQL masterはAPIで予めサービス名（alias DNS）を登録しておく
* mysql_failover_scriptで旧情報の削除と新情報の登録をやる
  * 旧master IPを無効化する部分でConsulのCatalog endpointを使ってderegister (/v1/catalog/deregister)
  * 新master ipを登録する部分でConsulのCatalog endpointを使ってregister (/v1/catalog/register)
  * 成功しない限り進まない（exit code check）
* 削除、及び登録はconsulのconsistency modelによって一貫性は保証される

## DEMO ###

{% youtube rA4hyJ-pccU %}


## Notes ##

DNSは最新のv0.3.0になってからTTLを設定できるようになったので、Amazon RDSっぽい感じのフェールオーバーも可能ですが、v0.2系に比べて格段にパフォーマンスが向上（スライド参照）したので、デフォルトのTTL 0でも問題ない範囲になってる感じです。また、もうちょっと詳しい内容は今度の[#hbstudy](http://connpass.com/event/7322/)で話す予定です。

## スライド ##

以下、July Tech Festa 2014で発表した時のスライドです。

{% speakerdeck e07317d0dc040131a0982229a5c1e016  %}
