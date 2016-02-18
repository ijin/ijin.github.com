---
layout: post
title: "LambdaでSSHやGitを使ってみよう"
date: 2016-02-18 12:35
comments: true
categories: 
- aws
- lambda
---

AWS Lambda上で`ssh`や`git`を使えたら便利！思ったけど、そもそもバイナリ自体が入ってない上にパッケージのインストールが出来ないので回避方法を悩んでいたところ、幸いそれぞれPython nativeの実装があったので先人の肩に乗っかる事で事無きを得ました。

## SSH

SSHは[Paramiko](http://www.paramiko.org)というライブラリを利用。s3上に[SSE-KMS](http://docs.aws.amazon.com/AmazonS3/latest/dev/UsingKMSEncryption.html)で暗号化されたキーファイルをダウンロードし、それを使ってサーバと認証しログイン。

注意点

- Lambdaの実行環境にはデフォルトでParamikoが入ってないので、deployment packageに含める必要がある
- Mac上ではpythonライブラリの互換性が衝突する為、Linux（EC2等）上でpackageを作成する必要がある
- 最近発表された[VPC対応](https://aws.amazon.com/blogs/aws/new-access-resources-in-a-vpc-from-your-lambda-functions/)を使ってprivate networkで通信をしたい場合、外部への経路はそのままでは不可なので、s3の場合はVPC endpointを作成するかNATが必要になってくる。詳しくはこの[記事](http://qiita.com/ijin/items/94c0bc4b8f6f5e77a591)を。

{% gist 26afca1e9b03ecaf4d8e %}

## Git

Gitは[`dulwich`](https://github.com/jelmer/dulwich)というライブラリを利用。sshプロトコルの場合、デフォルトではシステム上のsshが利用されるのでPython版の`ParamikoSSHVendor`クラスを使えばすんなりいくと思いきや、キーファイル指定が出来なかったのでそこを少し改変。また、Lambda上での特殊な環境の為か`sys.stderr` のencode周りがうまく検出されなかったので、`dulwich.porcelain.push` methodも若干修正。

Paramikoを使っているので、上記同様パッケージ作成はLinux上で。

{% gist 080822d2c7859b528631 %}


それでは、良いLambdaライフを！

