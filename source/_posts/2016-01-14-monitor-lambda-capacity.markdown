---
layout: post
title: "Lambdaの容量を監視しよう"
date: 2016-01-14 05:23
comments: true
categories: 
- aws
- lambda
---

`2016/1/14`現在、AWS Lambdaにはなんと**リージョン**毎！にアップロードできるパッケージの合計サイズがたったの**1.5GB**という[悲しい制限](http://docs.aws.amazon.com/lambda/latest/dg/limits.html#limits-list)があります。特にlibraryを同包したり、versioningを使ったりしてCIをガンガン回してると、結構すぐこの上限に達してしまいがちです。そこで、Lambdaの総容量はAWSコンソール上には表示されるものの、トラッキングし辛いので監視する仕組みを作ってみました。

## 仕組み

LambdaのScheduled Eventsを使って、`ListFunctions` APIを叩いて、個別functionの`CodeSize`をサマって、`PutMetricData`でCloudWatchに投げて、Alarm設定してるだけ。

<script src="https://gist-it.appspot.com/https://github.com/ijin/check_lambda_capacity/blob/master/lambda_function.py"></script>


といっても、今後別アカウントでいちいち設定（IAM role&policy、Lambda、SNS、CloudWatch）するのも非常に面倒くさいので、今回は[CloudFormation Designer](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/working-with-templates-cfn-designer.html)を使って、ほぼ一発で環境を再現できるようにしました。

## CloudFormation

ボタン作ってみた。<a href="https://console.aws.amazon.com/cloudformation/home?region=ap-northeast-1#/stacks/new?stackName=check-lambda-capacity&templateURL=https://s3-ap-northeast-1.amazonaws.com/ijin/aws/lambda/check_lambda_capacity/check_lambda_capacity.template">
<img src="https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png">
</a>

### Stack Creation
Designerではこんな感じ。心なしか、jsonの苦痛が多少楽になったような。。後、Propertyの補完機能は良いけどショートカットが`Cmd+Space`なのでSpotlightさんがぁ。

{%img https://lh3.googleusercontent.com/CLHrqYHPDKPxABDkWdrl0FYbEEC9enKmbOaK75TqhqL5tPnQ8oRjA3_f3N3iJoD5cSanPbbvt9lT=w880-h586-no %}

s3からtemplateを指定。
template urlは`https://s3-ap-northeast-1.amazonaws.com/ijin/aws/lambda/check_lambda_capacity/check_lambda_capacity.template`


{%img https://lh3.googleusercontent.com/XeF-yQk1cxy8z8KnHTJS3mP-onExoYmbJ7-OYISmLd3NYxO4fVCQnqAxygAEu6zdMtzEy16641C-=w826-h268-no %}

Parameterとしては以下が指定可能

- アラート閾値（Byte単位）
- SNS topic（空の場合は、自動作成される）

{%img https://lh3.googleusercontent.com/vX0JiEE_me3Wn6__PbNysmSYVkGiOs4jbqivZm7lOoq18kZpyrV9lj01kL9-rDMCX1JAS7BHVmI4=w902-h509-no %}

Stackを作成すると、諸々のリソースが数分で出来上がり。

{%img https://lh3.googleusercontent.com/rQy-Any2cmz9I7YxuGD-Pi8_4VB9dGKFnCbAxWHH4QPjRMF7Y9nKAi2sZPY2HogoL2vpcc8ABy4E=w1050-h260-no %}

### Manual Labor
本当は以上で終了！にしたいところですが、LambdaのScheduled Eventsの設定はAWSコンソールからのみしか出来ないという~~情けない~~[残念な状態](http://docs.aws.amazon.com/lambda/latest/dg/with-scheduled-events.html)（`2016/1/14`現在）なので、ここからポチポチ設定作業。。（API重視の開発姿勢はどこ行ったんだろう）

{%img https://lh3.googleusercontent.com/2pedymzqDYHJ30u7VglVv-7_6IpFhpOnRdnE4-QxjuIdX1fep2FwtoyDNr1Kl5yEqT1rtjoLDLZx=w581-h238-no %}

{%img https://lh3.googleusercontent.com/hcLpNX45gIQZQUQGvsyyIXfOH3s9yDpNLIZJN8TU-RUZDdMGdrsNXAb_NRupjWF9Hf4QrK4y6lXO=w864-h441-no %}

最小頻度が5分毎
{%img https://lh3.googleusercontent.com/lzuWtzvkK1A5pUSPyShVaFzzOQ13cD8PFVfCAz3rxSYkn5sKirWnhq-4PlKyhuesMlCQtrfH57yx=w862-h507-no %}

### Graph

これで、グラフが取れて閾値を超えたらアラートが飛ぶようになる

{%img https://lh3.googleusercontent.com/yzswstqCbaTzpkYfy5MlRS2I9ur5Is3hI8Eii-o27twq-fYA_6o8SY5Rrn0DbyZEn8UJarGhh_9L=w570-h270-no %}

{%img https://lh3.googleusercontent.com/NE43x1KQW76QO7b4TMIjjcEYrpQ07B9SNOb-9L-wEnNKc8oC-69HpgzWiVABnQwBSNGtDz4ayeTR=w286-h221-no %}

## Code

出来上がったCloudFormation templateコードはこちら。`AWS::Lambda::Function`がlambdaのresource担当だけど、templateにfunctionをインラインで埋め込めるのは現時点では[`nodejs`のみ](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-lambda-function-code.html)なので仕方なくzipしたpythonコードをs3にアップして参照するようにしてる。

<script src="https://gist-it.appspot.com/https://github.com/ijin/check_lambda_capacity/blob/master/check_lambda_capacity.template"></script>

元気があれば、そのうちnode版も書こうかな。。

GitHubのリポジトリは[こちらから](https://github.com/ijin/check_lambda_capacity)。
