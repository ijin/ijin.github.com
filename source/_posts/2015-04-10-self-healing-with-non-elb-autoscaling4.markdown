---
layout: post
title: "Auto Scalingによる自動復旧（AWS Lambda+SNS編）"
date: 2015-04-17 14:01
comments: true
categories: 
- aws
---

先週の[AWS Summit San Fransisco](http://aws.amazon.com/summits/san-francisco/)にて、ついにLambdaがSNSに[対応](http://docs.aws.amazon.com/sns/latest/dg/sns-lambda.html)しました。
様々なサービスが発表された中、個人的にはこれが一番のヒットです！というのも、この機能によってAWS間のサービスがより連携しやすくなり、新しいリアクティブなアーキテクチャをどんどん実現できそうだからです。

というわけで、少し試してみました。

お題は去年12月に試したAutoScaling + Lambda。（当時はLambdaはまだこの機能がなかったのでSNS→SQSにてイベントをプロセスする仕組みを[作りました](/blog/2014/12/05/self-healing-with-non-elb-autoscaling3/)。）

SNS連携によって前回のこれが

{% img https://lh4.googleusercontent.com/-IxSeVgkwfQU/VIIapiet4tI/AAAAAAAABBw/ukic0BIBIT0/w529-h393-no/aws-advent-2014.png %}

こうなります。（Lambdaのアイコンが出たので差し替えてます）

{% img https://lh3.googleusercontent.com/-ejPyB1qrZyQ/VTCEjbhe2cI/AAAAAAAABFI/VCYVIo5hFao/w404-h393-no/as-sns-lambda2.png %}

うーん、シンプル！

### 設定 ###

前回とほぼ同様。

SNS作成
```
$ aws sns create-topic --name instance-alert --region ap-northeast-1
{
    "TopicArn": "arn:aws:sns:ap-northeast-1:123456789012:instance-alert"
}
```

LambdaとSNS連携できるようにポリシーを付与
```
$ aws lambda add-permission --function-name makeASUnhealty --statement-id sns-instance-alert \
--action "lambda:invokeFunction" --principal sns.amazonaws.com \
--source-arn arn:aws:sns:ap-northeast-1:123456789012:instance-alert --region us-west-2                                                                                                                                                                                                  
{
    "Statement": "{\"Condition\":{\"ArnLike\":{\"AWS:SourceArn\":\"arn:aws:sns:ap-northeast-1:123456789012:instance-alert\"}},\"Resource\":\"arn:aws:lambda:us-west-2:123456789012:function:makeASUnhealty\",\"Action\":[\"lambda:invokeFunction\"],\"Principal\":{\"Service\":\"sns.amazonaws.com\"},\"Sid\":\"sns-instance-alert\",\"Effect\":\"Allow\"}"
}
```

Subscribe
```
$ aws sns subscribe --topic-arn arn:aws:sns:ap-northeast-1:123456789012:instance- --protocol lambda \
--notification-endpoint arn:aws:lambda:us-west-2:123456789012:function:makeASUnhealty --region ap-northeast-1
{
    "SubscriptionArn": "arn:aws:sns:ap-northeast-1:123456789012:as-test:4b22eec6-aeb5-4421-7a2f-99ca33a4b8ab"
}
```
aws cliはのヘルプにはまだSNSをLambdaへsubscribeするやり方は書いてませんが、上記のようにやればできます。 [Thanks Eric!](http://alestic.com/2015/04/aws-cli-sns-lambda)

EC2 Status Check
```
$ export INSTANCE=i-xxxxxxxx
$ aws cloudwatch put-metric-alarm --alarm-name StatusCheckFailed-Alarm-for-$INSTANCE \
--alarm-description "Instance $INSTANCE has failed" --metric-name StatusCheckFailed \
--namespace AWS/EC2 --statistic Maximum --dimensions Name=InstanceId,Value=$INSTANCE \
--period 60 --unit Count --evaluation-periods 2 --threshold 1 --comparison-operator \
  GreaterThanOrEqualToThreshold --alarm-actions arn:aws:sns:ap-northeast-1:560336700862:instance-alert \
--region ap-northeast-1
```

Lambda Function
{% gist 52033eb3b9b02c1fe975 %}

### 自動復旧 ###

通信を遮断し、Status Check Failを発動させる
```
$ date; sudo ifdown eth0
Fri Apr 17 03:08:39 UTC 2015
```

EC2 Status Check。2分でfail検知
```
Fri Apr 17 12:10:25 JST 2015
ok
DETAILS reachability    passed

Fri Apr 17 12:10:31 JST 2015
impaired
DETAILS 2015-04-17T03:10:00.000Z        reachability    failed
```

SNS通知。さらに2分ちょい。
```
Alarm Details:
- Name:                       StatusCheckFailed-Alarm-for-i-xxxxxxxx
- Description:                Instance i-xxxxxxxx has failed
- State Change:               OK -> ALARM
- Reason for State Change:    Threshold Crossed: 2 datapoints were greater than or equal to the threshold (1.0). The most recent datapoints: [1.0, 1.0].
- Timestamp:                  Friday 17 April, 2015 03:13:09 UTC
- AWS Account:                123456789012

Threshold:
- The alarm is in the ALARM state when the metric is GreaterThanOrEqualToThreshold 1.0 for 60 seconds.

Monitored Metric:
- MetricNamespace:            AWS/EC2
- MetricName:                 StatusCheckFailed
- Dimensions:                 [InstanceId = i-xxxxxxxx]
- Period:                     60 seconds
- Statistic:                  Maximum
- Unit:                       Count
```

Lambdaログ
```
2015-04-17T03:13:14.504Z  ac9ed52f-e4af-1e14-b826-ee9008a99db9	Received event:..
2015-04-17T03:13:14.504Z  ace9d52f-e4af-1e14-b826-ee9008a99db9	Changing instance health for: i-xxxxxxxx
2015-04-17T03:13:15.682Z  ace9d25f-e4af-1e14-b826-ee9008a99db9	{ ResponseMetadata: { RequestId: 'b0194dfb-e4af-1e14-895f-abdf96b0b593' } }
2015-04-17T03:13:15.684Z  ace9d25f-e4af-1e14-b826-ee9008a99db9	result: ""
```
タイムスタンプによるとSNS発砲されてたからLambda発火まで5秒！

Auto ScalingのHealth Status
```
Fri Apr 17 12:13:10 JST 2015
as-sg   ap-northeast-1a HEALTHY i-ce02563d      as-lc   InService

Fri Apr 17 12:13:15 JST 2015
as-sg   ap-northeast-1a UNHEALTHY       i-ce02563d      as-lc   InService

Fri Apr 17 12:13:20 JST 2015
as-sg   ap-northeast-1a UNHEALTHY       i-ce02563d      as-lc   InService

Fri Apr 17 12:13:26 JST 2015
as-sg   ap-northeast-1a UNHEALTHY       i-ce02563d      as-lc   InService

Fri Apr 17 12:13:31 JST 2015
as-sg   ap-northeast-1a UNHEALTHY       i-ce02563d      as-lc   InService

Fri Apr 17 12:13:37 JST 2015
as-sg   ap-northeast-1a UNHEALTHY       i-ce02563d      as-lc   Terminating

Fri Apr 17 12:13:43 JST 2015
as-sg   ap-northeast-1a UNHEALTHY       i-ce02563d      as-lc   Terminating

Fri Apr 17 12:13:49 JST 2015
as-sg   ap-northeast-1a UNHEALTHY       i-ce02563d      as-lc   Terminating

Fri Apr 17 12:13:54 JST 2015
as-sg   ap-northeast-1a UNHEALTHY       i-ce02563d      as-lc   Terminating

Fri Apr 17 12:13:59 JST 2015

Fri Apr 17 12:14:05 JST 2015
as-sg   ap-northeast-1a HEALTHY i-525601a1      as-lc   Pending

Fri Apr 17 12:14:10 JST 2015
as-sg   ap-northeast-1a HEALTHY i-525601a1      as-lc   Pending

Fri Apr 17 12:14:16 JST 2015
as-sg   ap-northeast-1a HEALTHY i-525601a1      as-lc   Pending

Fri Apr 17 12:14:21 JST 2015
as-sg   ap-northeast-1a HEALTHY i-525601a1      as-lc   Pending

Fri Apr 17 12:14:26 JST 2015
as-sg   ap-northeast-1a HEALTHY i-525601a1      as-lc   Pending

Fri Apr 17 12:14:32 JST 2015
as-sg   ap-northeast-1a HEALTHY i-525601a1      as-lc   Pending

Fri Apr 17 12:14:37 JST 2015
as-sg   ap-northeast-1a HEALTHY i-525601a1      as-lc   InService
```

ちゃんとTerminateされてリプレースされてますね。

AutoScalingの通知
```
Service: AWS Auto Scaling
Time: 2015-04-17T03:13:59.367Z
RequestId: efa97137-fa15-4aa4-9c8c-5241961a2d0e
Event: autoscaling:EC2_INSTANCE_TERMINATE
AccountId: 123456789012
AutoScalingGroupName: as-sg
AutoScalingGroupARN: arn:aws:autoscaling:ap-northeast-1:123456789012:autoScalingGroup:c395c157-3a7e-4d56-287b-5ad9b26eb464:autoScalingGroupName/as-sg
ActivityId: efa97137-fa15-4aa4-9c8c-5241961a2d0e
Description: Terminating EC2 instance: i-xxxxxxxx
Cause: At 2015-04-17T03:13:36Z an instance was taken out of service in response to a user health-check.
StartTime: 2015-04-17T03:13:36.342Z
EndTime: 2015-04-17T03:13:59.367Z
StatusCode: InProgress
StatusMessage:
Progress: 50
EC2InstanceId: i-xxxxxxxx
Details: {"Availability Zone":"ap-northeast-1a","Subnet ID":"subnet-bbbbbbbb"}
```

通常は`Cause`が`an instance was taken out of service in response to a EC2 health check indicating it has been terminated or stopped.`となるのが`an instance was taken out of service in response to a user health-check.`となっているのでAutoScalingのEC2 Health Checkより前にアクションが起こされた事が分かります。

障害発生からInstanceがリプレースされて`InService`になるAuto Healingのトータル時間は6分ちょいになりました。
[EC2 Auto Recovery](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-recover.html)を使えば済む場合もありますが、あちらはAWS側の障害に起因する`StatusCheckFailed_System`のみで`StatusCheckFailed_Instance`はトリガー対象じゃないのと、特定のインスタンスタイプやVPC等若干制限があります。

### 終わりに ###

ちなみに今回はinstanceやSNSは東京リージョン（ap-northeast-1）、Lambdaはオレゴンリージョン（us-west-2）というリージョンを跨いだ連携も可能という事が分かりました。まだ東京に来てないけど、既にproduction readyなのでもう普通に使っていけます。

いやー。SNS連携によって夢は広がりますねぇ。

