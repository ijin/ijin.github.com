---
layout: post
title: "LambdaのログをSlackで見よう"
date: 2015-12-10 00:56
comments: true
categories: 
- aws
- lambda
---

[今年もやるよ！AWS Lambda縛り Advent Calendar 2015](http://qiita.com/advent-calendar/2015/lambda)の10日分です。

## 背景
AWS Lambdaで開発してるとちょこちょこ実行ログを見たりします。cliであれば、[@sgwr_dts](https://twitter.com/sgwr_dts)さんの[lambchop](https://github.com/winebarrel/lambchop)が`tail`的に使えて素敵なんだけど、後で見返したり、検索したりするので、最近ではログをSlackに通知するようにしているのでその紹介を。

イメージはこんな感じ。

{%img https://lh3.googleusercontent.com/A1fsUgw4m52Db4UeOqR3xaQHIGF940MsbA1tchFkF8Pul-ixO6E=w563-h445-no %}

ログはCloudWatch logsに溜まるのでsubscriptionさえ出来れば、別にソースはLambdaじゃなくても良いんですけどね。

## Lambda

{%gist 917193e443ff41cdf98b %}

CloudWatch logsのイベントをparseして、日付の色付けやタイムゾーン変換等ちょっこっと加工してメッセージと共に指定の`#channel`に飛ばすようにし、別のSlack通知用lambda functionをinvokeしているだけですね。
（Slack用のlambdaは[以前のエントリ](/blog/2015/08/06/github-to-lambda-to-slack/)を参照）

何故Slackの部分を別functionにしてるかというと、最小単位の機能の切り出しによる**portability**とcross-account間のinvokeが可能となる**reusability**からです。

## 紐付け

### cloudwatch logsにlambda呼び出しの権限設定

```
aws --region ap-northeast-1 lambda add-permission --function-name "cloudwatch_logs" \
  --statement-id "logs-my_lambda" --principal "logs.ap-northeast-1.amazonaws.com" \
  --action "lambda:InvokeFunction" --source-account "123456789012" --source-arn \
  "arn:aws:logs:ap-northeast-1:123456789012:log-group:/aws/lambda/my_lambda:*"
```

### subscription filterの作成

```
aws --region ap-northeast-1 logs put-subscription-filter \
  --log-group-name "/aws/lambda/my_lambda" --filter-name logs-my_lambda \
  --filter-pattern ""
  --destination-arn arn:aws:lambda:ap-northeast-1:123456789012:function:cloudwatch_logs`
```

## 実行

これでlambda functionが実行されると、Slackにログが通知されます。

{%img https://lh3.googleusercontent.com/-PgkSA7jGbVTf3w1Eq5CZuvUAN4_bkKpR811Vcij5MlLam22C10=w706-h396-no %}
{%img https://lh3.googleusercontent.com/JKU31Qsd6cOSWe011m8UJaBYljDrXwz08ym0ff79DngGMxDL_y0=w705-h234-no %}

うん、見やすい。

## 問題点

これでログが飛ぶようになったのは良いですが、Lambdaが実行されてからSlackへの通知まで10数秒とやや長めの時間がかかってしまうのが現在の難点です。（Immediate Feedbackはないものの履歴や検索用途には十分だけど）

調べてみるとLambdaからCloudWatch logsへの書き出しが一番時間がかかっているのが判明。Logsへ書き出されたら、そこからの処理時間は1〜2秒とちょっと前に[発表された](https://aws.amazon.com/about-aws/whats-new/2015/09/near-real-time-processing-of-amazon-cloudwatch-logs-with-aws-lambda/)CloudWatch logsの*near realtime processing*の通りなので、早く`Lambda -> CloudWatch logs`も同じような処理時間を実現して欲しいものです。

まあ、別にCloudWatch logsにさえ入れば早いので、lambda logに限らずいろいろ応用できそうですが。

では、Happy Lambda Life!


参考

- <http://docs.aws.amazon.com/AmazonCloudWatch/latest/DeveloperGuide/Subscriptions.html#LambdaFunctionExample>
- <http://inokara.hateblo.jp/entry/2015/10/18/201212>
