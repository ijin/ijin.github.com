---
layout: post
title: "Auto Scalingによる自動復旧（AWS Lambda編）"
date: 2014-12-05 23:14
comments: true
categories: 
- aws
---

ちょうど1年程前に「[非ELBなAutoscalingによる自動復旧](/blog/2013/02/08/self-healing-with-non-elb-autoscaling/)」の[再検証](/blog/2013/12/14/self-healing-with-non-elb-autoscaling2/)をしました。前回も復旧までのタイムラグが20分だったので、この1年で変わったかまた検証してみました。

(*) このエントリは[AWS Advent Calendar 2014](http://qiita.com/advent-calendar/2014/aws)の5日目分です。

### 設定 ###

[前回](/blog/2013/12/14/self-healing-with-non-elb-autoscaling2/)とほぼ一種ですが、今回はついでにEC2 Status AlarmをCloudwatch経由でSNSでアラートを飛ばします。

SNS作成 & Subscribe（送られてくる確認メールは手動で承認）
```
$ aws sns create-topic --name instance-alert
{
    "TopicArn": "arn:aws:sns:us-west-2:123456789012:instance-alert"
}

$ aws sns subscribe --topic-arn arn:aws:sns:us-west-2:123456789012:instance-alert --protocol email --notification-endpoint user@example.com                               
{
    "SubscriptionArn": "pending confirmation"
}

```

EC2 Status Alarm登録
```
export INSTANCE=i-xxxxxxxx
aws cloudwatch put-metric-alarm --alarm-name StatusCheckFailed-Alarm-for-$INSTANCE \
--alarm-description "Instance $INSTANCE has failed" --metric-name StatusCheckFailed \
--namespace AWS/EC2 --statistic Maximum --dimensions Name=InstanceId,Value=$INSTANCE \
--period 60 --unit Count --evaluation-periods 2 --threshold 1 --comparison-operator \
  GreaterThanOrEqualToThreshold --alarm-actions arn:aws:sns:us-west-2:123456789012:instance-alert
```

### 自動復旧 ###

通信を遮断し、Status Check Failを発動させる
```
ubuntu@ip-172-31-19-185:~$ date; sudo ifdown eth0
Fri Dec  5 13:02:39 UTC 2014
```

EC2のStatus Check。前回同様、3分ぐらいでfail検知
```
Fri Dec  5 22:05:22 JST 2014
ok
DETAILS reachability    passed

Fri Dec  5 22:05:28 JST 2014
ok
DETAILS reachability    passed

Fri Dec  5 22:05:34 JST 2014
impaired
DETAILS 2014-12-05T13:05:00.000Z        reachability    failed

Fri Dec  5 22:05:40 JST 2014
impaired
DETAILS 2014-12-05T13:05:00.000Z        reachability    failed
```


SNS通知。飛ぶまで2分弱
```
Alarm Details:
- Name:                       StatusCheckFailed-Alarm-for-i-xxxxxxxx
- Description:                Instance i-xxxxxxxx has failed
- State Change:               OK -> ALARM
- Reason for State Change:    Threshold Crossed: 2 datapoints were greater than or equal to the threshold (1.0). The most recent datapoints: [1.0, 1.0].
- Timestamp:                  Friday 05 December, 2014 13:07:19 UTC


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


AutoscalingのHealth Status
```
Fri Dec  5 22:11:08 JST 2014
aws-advent2014-sg       us-west-2a      HEALTHY i-xxxxxxxx      aws-advent2014-lc       InService

Fri Dec  5 22:11:14 JST 2014
aws-advent2014-sg       us-west-2a      HEALTHY i-xxxxxxxx      aws-advent2014-lc       InService

Fri Dec  5 22:11:20 JST 2014
aws-advent2014-sg       us-west-2a      UNHEALTHY       i-xxxxxxxx      aws-advent2014-lc       InService

Fri Dec  5 22:11:26 JST 2014
aws-advent2014-sg       us-west-2a      UNHEALTHY       i-xxxxxxxx      aws-advent2014-lc       InService

Fri Dec  5 22:11:32 JST 2014
aws-advent2014-sg       us-west-2a      UNHEALTHY       i-xxxxxxxx      aws-advent2014-lc       Terminating


```
お、4分ぐらいでAuto ScalingがUnhealthyと認識。

何回か繰り返したところ、トータルで障害から復旧までの時間が10分を切りました！

なんと1年前と比べて時間が半分に短縮されてますねぇ。

## Lambda ##

さて。改善されたものの、Auto ScalingがEC2 status checkの状態を検知するまでタイムラグがまだあるので、もうちょっと短縮したいですよね。できればSNSが発行されたタイミングで。

そこで、AWSの新サービス「[Lambda](http://aws.amazon.com/lambda/)」を使ってイベント通知できたら良いかも！！。。と思ったものの、残念ながらLambdaはまだSNSには対応してません。

なので、ひとまずSQSをSNSにsubscribeして、messageが届いたらLambda functionへ渡してinvokeへしてくれるsqs-to-lambdaを使ってAuto ScalingのHealthStatusを直接API経由でLambdaが叩く仕組みを試しました。
図にするとこんな感じですね。

{% img https://lh4.googleusercontent.com/-IxSeVgkwfQU/VIIapiet4tI/AAAAAAAABBw/ukic0BIBIT0/w529-h393-no/aws-advent-2014.png %}

ELB付けた方が楽な気もするけど、まあ集約できるのと検証も兼ねて。。

### 設定 ###

SQSの作成
```
$ aws sqs create-queue --queue-name instance-failed
{
    "QueueUrl": "https://us-west-2.queue.amazonaws.com/123456789012/instance-failed"
}
```

SQSをSNSへsubscribe
```
$ aws sqs add-permission --queue-url https://us-west-2.queue.amazonaws.com/123456789012/instance-failed \
 --label SQSDefaultPolicy --aws-account-ids  --actions SendMessage
$ aws sqs set-queue-attributes --queue-url https://us-west-2.queue.amazonaws.com/123456789012/instance-failed  \
 --attributes '{"Policy": "{\"Version\":\"2008-10-17\",\"Id\":\"arn:aws:sqs:us-west-2:123456789012:instance-failed/SQSDefaultPolicy\",\"Statement\":[{\"Sid\":\"Sid1417796380309\",\"Effect\":\"Allow\",\"Principal\":{\"AWS\":\"*\"},\"Action\":\"SQS:SendMessage\",\"Resource\":\"arn:aws:sqs:us-west-2:123456789012:instance-failed\",\"Condition\":{\"ArnEquals\":{\"aws:SourceArn\":\"arn:aws:sns:us-west-2:123456789012:instance-alert\"}}}]}"}'
```

Lambda function

{% gist 7bcba3354814d3ca704d %}


node.js製のsqs-to-lambda。long pollingしつつ、messageを取得後にLambda functionをinvokeしてくれる。12/5現在ではEscape characterがAWS/JDKやCLIから送信できないという[大きな問題](https://forums.aws.amazon.com/thread.jspa?threadID=166893&tstart=0)がある為、[upstream](https://github.com/robinjmurphy/sqs-to-lambda)を少し[改修](https://github.com/ijin/sqs-to-lambda/commit/080f1dcbf5f8bb3f7f4a6e0abdde72dce7ce5553)
```
sudo apt-get install nodejs npm
sudo ln -s /usr/bin/nodejs /usr/local/bin/node
git@github.com:ijin/sqs-to-lambda.git
cd sqs-to-lambda
npm install
./index.js --queueUrl https://sqs.us-west-2.amazonaws.com/123456789012/instance-failed --functionName myFunction --region us-west-2  
```

### Lambdaによる復旧 ###

通信を遮断し、Status Check Failを発動させる
```
ubuntu@ip-172-31-21-180:~$ date; sudo ifdown eth0
Fri Dec  5 19:37:32 UTC 2014
```

EC2のStatus Check。約3分ぐらいでfail検知
```
Sat Dec  6 04:40:09 JST 2014
ok
DETAILS reachability    passed

Sat Dec  6 04:40:16 JST 2014
impaired
DETAILS 2014-12-05T19:40:00.000Z        reachability    failed

Sat Dec  6 04:40:22 JST 2014
impaired
DETAILS 2014-12-05T19:40:00.000Z        reachability    failed
```

SNS通知。飛ぶまで2分弱
```
Alarm Details:
- Name:                       StatusCheckFailed-Alarm-for-i-yyyyyyyy
- Description:                Instance i-yyyyyyyy has failed
- State Change:               OK -> ALARM
- Reason for State Change:    Threshold Crossed: 2 datapoints were greater than or equal to the threshold (1.0). The most recent datapoints: [1.0, 1.0].
- Timestamp:                  Friday 05 December, 2014 19:42:33 UTC
- AWS Account:                123456789012 
```

そしてLambda発動！
```
START RequestId: e64849be-7cb6-12e4-9951-11b87edac46e
2014-12-05T19:42:36.728Z        e64849be-7cb6-12e4-9951-11b87edac46e    Received event:
2014-12-05T19:42:36.728Z        e64849be-7cb6-12e4-9951-11b87edac46e    { Type: 'Notification', MessageId: '2d85a5d8-c596-5f83-94a9-e924c97fd676', TopicArn: 'arn:aws:sns:us-west-2:123456789012:instance-alert', Subject: 'Status Check Alarm: !!StatusCheckFailed-Alarm-for-i-yyyyyyyy!! in US-West-2', Message: '{!!AlarmName!!:!!StatusCheckFailed-Alarm-for-i-yyyyyyyy!!,!!AlarmDescription!!:!!Instance i-yyyyyyyy has failed!!,!!AWSAccountId!!:!!123456789012!!,!!NewStateValue!!:!!ALARM!!,!!NewStateReason!!:!!Threshold Crossed: 2 datapoints were greater than or equal to the threshold (1.0). The most recent datapoints: [1.0, 1.0].!!,!!StateChangeTime!!:!!2014-12-05T19:42:33.757+0000!!,!!Region!!:!!US-West-2!!,!!OldStateValue!!:!!OK!!,!!Trigger!!:{!!MetricName!!:!!StatusCheckFailed!!,!!Namespace!!:!!AWS/EC2!!,!!Statistic!!:!!MAXIMUM!!,!!Unit!!:!!Count!!,!!Dimensions!!:[{!!name!!:!!InstanceId!!,!!value!!:!!i-yyyyyyyy!!}],!!Period!!:60,!!EvaluationPeriods!!:2,!!ComparisonOperator!!:!!GreaterThanOrEqualToThreshold!!,!!Threshold!!:1.0}}', Timestamp: '2014-12-05T19:42:33.841Z', SignatureVersion: '1', Signature: '1IYhSVfZmNxWWzoc539jBDN2HCo0Y5k/dUhWbaAEZSt/tkISjFkNTb9VsVwNAfZDOaLneO/sE2PwfUc/3aU9eedlAassxHOXAB6h844NVKxJzR5Xwg4dUx0mIb+fk9pMy/elcwk13GbDxLJ1cCTef7Bu7zyJU3TAF626YfAVhI9QdEo4o44g/y2osEXb+CuvFc5ICYpIWAad7gM5YPYxCU6tJ/CEtWGzaPz+O5Vk4NLm2/AizZ6LKA8/zqhQkqwnUwhzQDwuDGbJ2DXtTJwAO2r4M+zU8RwOxwPgEdgxA270xrmB6AlWV0mhsQIqqJVxo5Xm2v7y3iNUjKfov5DCZm==', SigningCertURL: 'https://sns.us-west-2.amazonaws.com/SimpleNotificationService-ad6697a11189d5c6f9eccf214ff9e123.pem', UnsubscribeURL: 'https://sns.us-west-2.amazonaws.com/?Action=Unsubscribe&SubscriptionArn=arn:aws:sns:us-west-2:123456789012:instance-alert:2d85a5d8-faea-5f83-baa1-9fecd7a5e71b' }
2014-12-05T19:42:36.728Z        e64849be-7cb6-12e4-9951-11b87edac46e    Changing instance health for: i-yyyyyyyy
014-12-05T19:42:36.831Z e64849be-7cb6-12e4-9951-11b87edac46e    { ResponseMetadata: { RequestId: 'dd605f39-7cb6-12e4-a2c2-d57010899d82' } }
END RequestId: e64849be-7cb6-12e4-9951-11b87edac46e
REPORT RequestId: e64849be-7cb6-12e4-9951-11b87edac46e  Duration: 175.86 ms     Billed Duration: 200 ms Memory Size: 128 MB     Max Memory Used: 18 MB
```

AutoscalingのHealth Status
```
Sat Dec  6 04:42:18 JST 2014
aws-advent2014-sg       us-west-2a      HEALTHY i-yyyyyyyy      aws-advent2014-lc       InService

Sat Dec  6 04:42:24 JST 2014
aws-advent2014-sg       us-west-2a      HEALTHY i-yyyyyyyy      aws-advent2014-lc       InService

Sat Dec  6 04:42:30 JST 2014
aws-advent2014-sg       us-west-2a      UNHEALTHY       i-yyyyyyyy      aws-advent2014-lc       InService

Sat Dec  6 04:42:35 JST 2014
aws-advent2014-sg       us-west-2a      UNHEALTHY       i-yyyyyyyy      aws-advent2014-lc       InService

Sat Dec  6 04:42:42 JST 2014
aws-advent2014-sg       us-west-2a      UNHEALTHY       i-yyyyyyyy      aws-advent2014-lc       InService

jSat Dec  6 04:42:48 JST 2014
aws-advent2014-sg       us-west-2a      UNHEALTHY       i-yyyyyyyy      aws-advent2014-lc       Terminating
```

おお。さらに短く5分ぐらいに短縮！！

### 終わりに ###

Lambdaと組み合わせる事によってSNS通知によるELBを使わないレスポンシブなAuto Scalingの自動復旧が実現できました。

折角のサーバいらずのLambdaの良さが全く活かされてないけど。。早くSNSにも対応して欲しいものです。
