---
layout: post
title: "非ELBなAutoscalingによる自動復旧"
date: 2013-02-08 09:29
comments: true
categories: 
- aws
---


バッチ処理等、サーバの冗長化が難しく仕方なく1台で動かさざるを得ない場合があります。でも可用性は確保したい。また、Pacemakerやkeepalived等できなくはないが、お金もあんまりかけられない場合もあります。
そんな時にAWS上でよく使うのがAutoscalingによる1台構成です。最低台数・最大台数共に「1」に設定しておけばEC2インスタンスが壊れても自動的に新しいのにリプレースされて復旧されます。

しかし、Autoscalingのhealth-check-typeを「EC2」にした場合、インスタンスの起動状態（running, stopped）しか見てくれないので、今までこの構成を実現するのにELBによる死活監視が必要でした。インスタンスがHTTPサーバじゃない場合、ちょっとムダです。

ところが、ちょっと前にAutoscalingがインスタンスの健康状態をチェックするEC2 status checkに[対応](http://aws.amazon.com/about-aws/whats-new/2012/12/14/auto-scaling-now-uses-amazon-ec2-status-checks/)し、ELBが不要になったはずなので試してみました。

今回は各種AWSサービスに対応した統合されたPython版の[AWS CLI](http://aws.amazon.com/cli/)ツールを使います。

### セットアップ  ###
まずは、ツールのインストール

	sudo apt-get install python-pip
	sudo pip install awscli
	complete -C aws_completer was

### Autoscaling設定  ###

Launch Configの設定

	aws autoscaling create-launch-configuration --image-id ami-4a12aa4b \
	--launch-configuration-name test-lc --instance-type t1.micro --key-name ijin-tokyo \
	--security-groups test --iam-instance-profile test_iam
	
	{
	    "ResponseMetadata": {
	        "RequestId": "c0e66974-7103-11e2-9780-a53199bac60e"
	    }
	}

Scaling Groupの設定

	aws autoscaling create-auto-scaling-group --auto-scaling-group-name test-sg \
	--launch-configuration-name test-lc --min-size 1 --max-size 1 \
	--health-check-grace-period 180 --tags '{"key":"Name", "value":"as-test"}' \
	'{"key":"Use Case", "value":"test"}' --availability-zones ap-northeast-1a --health-check-type "EC2"
	
	{
	    "ResponseMetadata": {
	        "RequestId": "e3808ef3-7103-11e2-9780-a53199bac60e"
	    }
	}

通知

	aws autoscaling put-notification-configuration --auto-scaling-group-name test-sg \
	--topic-arn arn:aws:sns:ap-northeast-1:521026608000:test \
	--notification-types autoscaling:EC2_INSTANCE_LAUNCH autoscaling:EC2_INSTANCE_TERMINATE \
	autoscaling:EC2_INSTANCE_LAUNCH_ERROR autoscaling:EC2_INSTANCE_TERMINATE_ERROR
	
	{
	    "ResponseMetadata": {
	        "RequestId": "f68359be-7103-11e2-9a1a-5f77b12b596e"
	    }
	}


レスポンスがjsonなのが良いですね。
また、[Auto Scaling Command Line Tool](http://aws.amazon.com/developertools/2535)と違って、tagでスペースが使えるようになったのが素晴らしい！

以上の設定でAutoscalingによってインスタンスが1台立ち上がります。

### 自動復旧 ###

最後にインスタンス不調（status check failure）をシミュレートする為にインスタンス内からネットワークを落とします。

	ubuntu@ip-10-128-25-25:~$ sudo ifdown eth0

これでstatus checkがfailし、Autoscalingが自動的に新しいインスタンスと取り替えてくれるはず！

{% img https://lh6.googleusercontent.com/-9FpM6ywjg84/URN6q29FD-I/AAAAAAAAAsw/paP8HBnisPE/s161/aws_status_check_2013-02-07+18.06.20.png %}

{% img https://lh5.googleusercontent.com/-5avxLvp5RH8/URN6n-ihwiI/AAAAAAAAAso/zOHxli4vM8o/s702/aws_instance_check_2013-02-07+18.09.19.png %}

と、期待して待っていたらなかなかアラートメールが来ません。。


設定間違ったかなーっていろいろ見直していたら20分経った頃にやっと到着。

```
Service: AWS Auto Scaling
Time: 2013-02-07T09:17:42.304Z
RequestId: f395660b-4847-4415-ad33-f8cc5091bdb3
Event: autoscaling:EC2_INSTANCE_TERMINATE
AccountId: 521026608000
AutoScalingGroupName: test-sg
AutoScalingGroupARN: arn:aws:autoscaling:ap-northeast-1:521026608000:autoScalingGroup:a036877b-dab7-4e5d-a6e1-1d3424b20d14:autoScalingGroupName/test-sg
ActivityId: f395660b-4837-4415-ad33-f8cc5071bdb3
Description: Terminating EC2 instance: i-15fadd17
Cause: At 2013-02-07T09:16:57Z an instance was taken out of service in response to a system health-check.
StartTime: 2013-02-07T09:16:57.389Z
EndTime: 2013-02-07T09:17:42.304Z
StatusCode: InProgress
StatusMessage:
Progress: 50
EC2InstanceId: i-15fadd17
Details: {}
```

うーん。動く事は動いたけど、ちょっと時間がかかるなぁ。

この後何回か試してみたけど、Autoscaling発動までどれも20分ぐらいかかりました。


### 結論 ###

20分程サーバダウンが許容できるようなゆるめの条件に限定した場合には非ELBでも使えるかな。まあ、それでも適用する場面は多々あるとは思いますが。（早める方法あるのかなー。）


