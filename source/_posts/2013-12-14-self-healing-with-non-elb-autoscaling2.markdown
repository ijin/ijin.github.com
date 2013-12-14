---
layout: post
title: "Autoscalingによる自動復旧(Immutable Infrastucture)"
date: 2013-12-14 23:40
comments: true
categories: 
- aws
---

以前、「[非ELBなAutoscalingによる自動復旧](/blog/2013/02/08/self-healing-with-non-elb-autoscaling/)」でインスタンスの自動復旧の挙動をテストしました。
障害が発生したサーバをterminateし、新サーバをstartしてリプレースする仕組みはまさに最近話題のImmutable Infrastructureですね。
CDP的には「[Server Swappingパターン](http://aws.clouddesignpattern.org/index.php/CDP:Server_Swapping%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3)」が一番近いですが、今後はImmutable分類もあっても良いような気がします。

前回はAuto Scalingがインスタンス障害を検知してリプレースするまでのタイムラグが約20分だと分かりました。
本日、インスタンスの状態をチェックするEC2 Status Checkが1分間隔になった（以前は5分間隔）と[発表](https://forums.aws.amazon.com/ann.jspa?annID=2266)されたので、
これによってタイムラグが短縮されたかを検証してみます。

### 設定 ###

手順は前回と一緒なので省略

### 自動復旧 ###

通信を遮断し、Status Check Failを発動させる
```
ubuntu@ip-10-123-32-180:~$ date; sudo ifdown eth0
Sat Dec 14 14:19:19 UTC 2013
Write failed: Broken pipe
```

EC2のStatus Checkを流す
    while true; do date; aws ec2 describe-instance-status --instance-ids i-b03788b5 --query 'InstanceStatuses[*].InstanceStatus' --output text ; echo ; sleep 10; done

```
Sat Dec 14 23:22:05 JST 2013
ok
DETAILS reachability    passed

Sat Dec 14 23:22:16 JST 2013
ok
DETAILS reachability    passed

Sat Dec 14 23:22:27 JST 2013
impaired
DETAILS 2013-12-14T14:22:00.000Z        reachability    failed

Sat Dec 14 23:22:37 JST 2013
impaired
DETAILS 2013-12-14T14:22:00.000Z        reachability    failed
```

約3分でStatus異常が検知されました。

{% img https://lh5.googleusercontent.com/-pabfvBU5fW0/Uqx4SEmBYLI/AAAAAAAAA2E/abCLUwoESds/w734-h154-no/Instance_status_check_2013-12-14+at+11.34.27+PM.png %}

Auto ScalingのHealthStatusを流す
    while true; do date; aws autoscaling describe-auto-scaling-instances --query 'AutoScalingInstances[*].HealthStatus' --output text; echo; sleep 10; done

```
Sat Dec 14 23:38:06 JST 2013
[
    {
        "State": "InService", 
        "Health": "HEALTHY", 
        "ID": "i-b03788b5"
    }
]

Sat Dec 14 23:38:17 JST 2013
[
    {
        "State": "InService", 
        "Health": "UNHEALTHY", 
        "ID": "i-b03788b5"
    }
]

Sat Dec 14 23:38:28 JST 2013
[
    {
        "State": "InService", 
        "Health": "UNHEALTHY", 
        "ID": "i-b03788b5"
    }
]

Sat Dec 14 23:38:38 JST 2013
[
    {
        "State": "Terminating", 
        "Health": "UNHEALTHY", 
        "ID": "i-b03788b5"
    }
]


Sat Dec 14 23:38:49 JST 2013
[
    {
        "State": "Terminating", 
        "Health": "UNHEALTHY", 
        "ID": "i-b03788b5"
    }
]

Sat Dec 14 23:38:59 JST 2013
[
    {
        "State": "Terminating", 
        "Health": "UNHEALTHY", 
        "ID": "i-b03788b5"
    }
]

Sat Dec 14 23:39:10 JST 2013
[
    {
        "State": "Pending", 
        "Health": "HEALTHY", 
        "ID": "i-4cc7a849"
    }, 
    {
        "State": "Terminating", 
        "Health": "UNHEALTHY", 
        "ID": "i-b03788b5"
    }
]

```
やっとAuto Scalingの方でも異常検知。

{% img https://lh4.googleusercontent.com/-8Dj_s0mm9I8/Uqx4R1-Wl9I/AAAAAAAAA2I/laUZmwhUw8o/w873-h151-no/Autoscaling_health_2013-12-14+at+11.57.42+PM.png %}

SNS通知

```
Service: AWS Auto Scaling
Time: 2013-12-14T14:39:41.271Z
RequestId: f622c6d2-8c77-4fef-8b38-ece463574712
Event: autoscaling:EC2_INSTANCE_TERMINATE
AccountId: 111155559999
AutoScalingGroupName: test-sg
AutoScalingGroupARN: arn:aws:autoscaling:ap-northeast-1:11115559999:autoScalingGroup:0e771015-f979-4afe-b065-595abafdbf9b:autoScalingGroupName/test-sg
ActivityId: f622c6d2-8c77-4fef-8b38-ece463574712
Description: Terminating EC2 instance: i-b03788b5
Cause: At 2013-12-14T14:38:38Z an instance was taken out of service in response to a system health-check.
StartTime: 2013-12-14T14:38:38.257Z
EndTime: 2013-12-14T14:39:41.271Z
StatusCode: InProgress
StatusMessage:
Progress: 50
EC2InstanceId: i-b03788b5
```

やはり20分のタイムラグ変わらずですね。。

### 結論 ###

というわけで、EC2 Status Checkが1分間隔になっても、EC2のみ（ELBを使わい場合）のAuto Scalingによる不調インスタンスの自動復旧時間は変わらずでした。

ちなみにAWS ConsoleでAuto Scalingの設定ができるようになったけど、まだscaling groupにtagが付けられないのがちょっと微妙ですね。。GUIで状態を見る分には楽だけど。
