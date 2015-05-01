---
layout: post
title: "EC2 Auto Recoveryの注意点"
date: 2015-05-01 11:45
comments: true
categories: 
- aws
---

先日、EC2のAuto Recoveryでちょっとハマったのでメモ。

Cloudwatchの`StatusCheckFailed_System`アラームを設定すると、インスタンスを自動的に復旧してくれるEC2 Auto Recoveryという機能があり、使うためには条件がいくつかあります。

* 特定のリージョン
* C3, C4, M3, R3, T2 instances
* VPC
* 共有tenancy
* EBSストレージのみのサーバ

[Recover Your Instance](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-recover.html)

しかし、上記を満たしているのに、特定のAMIをCLI経由で起動するとEC2 Auto Recoveryを設定できなくなります。

（AWSコンソールでラジオボタンが押せない。CLIで設定しても効かない）
{% img https://lh5.googleusercontent.com/-dwWUcfCssA4/VULwrOsgxNI/AAAAAAAABFo/VQy-GnIYQXg/w448-h186-no/Screenshot%2B2015-05-01%2B12.18.16.png %}

原因はAMIにephemeral disk等のblock device mappingが設定されていて、T2やC4等のEBS onlyなinstance typeで起動しているにも関わらず、AWS側がephemeral diskが付与されていると認識してしまう所にあります。なお、AWSコンソールからの起動だとこの現象は発生しません。

<script type="text/javascript" src="https://ajax.googleapis.com/ajax/libs/jquery/1.9.1/jquery.min.js"></script>
<script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/gist-embed/2.1/gist-embed.min.js"></script>
AMIでblock device mappingが埋め込まれている
    aws ec2 describe-images --image-ids ami-e74b60e6 --region ap-northeast-1
<code data-gist-id="55659515d54593c29618" data-gist-highlight-line="21-28"></code>

本来はEBS onlyなinstance typeだとblock device mappingの設定如何に関わらず、付与自体が不可能なので、全く関係ないはずです。

解決方法は現時点（2015/5/1）では3通り

* extraなmappingが設定されていないAMIを使う（Amazon Linux等）
* AWSコンソールから起動する
* 明示的に`--block-device-mappings`パラメータで`NoDevice`と指定
`aws ec2 run-instances --image-id ami-e74b60e6 --instance-type t2.small --subnet-id subnet-xxxxxxxx --block-device-mappings "[{\"DeviceName\": \"/dev/sdb\",  \"NoDevice\": \"\"}, {\"DeviceName\": \"/dev/sdc\",  \"NoDevice\": \"\"}]"`

これは明らかにAWS側のバグなので早く治って欲しいものですね。
