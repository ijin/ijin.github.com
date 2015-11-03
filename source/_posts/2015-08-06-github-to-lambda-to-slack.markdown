---
layout: post
title: "GitHubのeventをLambdaで処理してSlackへ通知"
date: 2015-08-06 01:46
comments: true
categories: 
- aws
- github
---

ちょっと前にGitHubのeventを[AWS Lambda](https://aws.amazon.com/lambda/)で処理して、GitHubやSlackのAPIを叩く仕組みを作ったので、メモ。

材料

* Github
* SNS
* KMS
* Lambda
* Slack

やりたいことはこんな感じ。

{%img https://lh3.googleusercontent.com/-aep_hTkw5Mo/VjkHcm612yI/AAAAAAAABbo/00aae3JXUkc/s548-Ic42/github%25252Blambda%25252Bslack%252520%2525281%252529.png %}


あるGitHubリポジトリのissuesに特定のコメントが書き込まれると、そのユーザはプロジェクトのteamに自動で追加されて、Slackへ通知が流れるというモノです。


## SNS

まずは媒介となるSNSの作成。
```
$ aws sns create-topic --name github --region ap-northeast-1
{
    "TopicArn": "arn:aws:sns:ap-northeast-1:123456789012:github"
}
```

次にSNSに対してpublishできるIAM userを作成

IAM Policy
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "sns:Publish"
      ],
      "Sid": "Stmt0000000000000",
      "Resource": [
        "arn:aws:sns:ap-northeast-1:123456789012:github"
      ],
      "Effect": "Allow"
    }
  ]
}
```

## GitHub

* organization: `my_org`
* repository: `test`
* team: `reader`

### SNS連携 ###

_Webhooks & Services_ からAmazonSNSと連携

{%img https://lh3.googleusercontent.com/u4I9Z_Ing9YzhmVCQhMACwYDVIJxJM7C-aDmyMsNL3o=w328-h190-no %}

AWS KEYには先ほど作成したIAMユーザのを利用。SNS topicのarnにはregionが書かれているにも関わらず、GitHubの方では明示的に指定が必要。

{%img https://lh3.googleusercontent.com/kppZQx_RdhyC11LB7a0cPtppDUiKgfZBm6laYvRG3zA=w382-h289-no %}

### 通知するeventの有効化

さて、GitHubではSNSの場合、[hooksのjson](https://api.github.com/hooks)にある通り、`push`時のeventにしか対応していないので、

```json hooks https://api.github.com/hooks
"name": "amazonsns",
"events": [
  "push"
],
```

GitHubの[Webhook API](https://developer.github.com/v3/orgs/hooks/)に従って`issue_comment`を追加してやります。


先ほどのSNS連携のhook idを取得するには`GET /orgs/:org/hooks`

```bash
export HOOK_ID=$(curl -H 'Uer-Agent: ijin' -X GET -s -H "Authorization: token xxxxxxxxxx" \
https://api.github.com/repos/my_org/test/hooks | jq '.[].id')
```

編集するには`PATCH /orgs/:org/hooks/:id`

```bash
curl -H 'Uer-Agent: ijin' -X PATCH -s -H "Authorization: token xxxxxxxxxx" \
https://api.github.com/repos/my_org/test/hooks/$HOOK_ID -d '{"add_events": ["issue_comment"]}' | jq .
```

Web UIからは分からないので、ついでに`reader` teamのIDも取得しておく

```bash
curl -H 'Uer-Agent: ijin' -X GET -s -H "Authorization: token xxxxxxxxxx" \
https://api.github.com/orgs/my_org/teams | jq '.[] | select(.name=="reader")'
```

（※）User-Agentは[必須](https://developer.github.com/v3/#user-agent-required)

これで、誰かがコメントをした時にもSNSが飛びます。

## KMS

lambdaでは以下の認証情報を使うので、予めKMSで暗号化しておく。

* GitHub token
* Slack webhook 

rubyで暗号化する場合
```ruby
require 'aws-sdk'
require 'base64'
kms = Aws::KMS::Client.new(region: 'us-east-1')
Base64.encode64 kms.encrypt(key_id: "alias/ijin", plaintext: 'my plain text code').ciphertext_blob
```

javascriptの場合
```javascript
var aws = require('aws-sdk');
var kms = new aws.KMS({ region: 'us-east-1' });
var text = 'my plain text code';
kms.encrypt({KeyId: 'alias/ijin', Plaintext: text}, function(err, data) {
  if (err) console.log(err, err.stack);
  else console.log(data.CiphertextBlob.toString('base64'));
});
```

こうしておくと、SCMに入れても安心。

また、KMSキーの実行権限もlambdaのroleに紐づけておく。

{%img https://lh3.googleusercontent.com/oMHQ76XC7RQUCR_fEo9JyaMhPKSbnIdhCzjIWurEJ9c=w542-h358-no %}

## Lambda

Lambda function作成時にはSNSをevent sourceとして指定。

{%img https://lh3.googleusercontent.com/uDXkEVLrXT1BdUSbiMu3v5lnwTqCoXOkv_nu07XVxkk=w555-h235-no %}

(※) KMSは現在us-eastにしかないので、そこ以外のregionでlambdaを実行する場合は、`timeout`は若干長めに指定して置くと良さげ

コードはこんな感じ。

1. GitHubからSNS hookを受け取って
1. コメントした内容が`join`の場合
1. KMSによってGitHubのtokenを復号化し
1. そのユーザをteamへ追加する
1. その後、別lambda functionでslackへ通知する

{% gist ef105e192a030571d83f %}

{% gist f83e33a6ae0acd83902a %}

nodeのlibraryを使うともっとスッキリ書けたけど、1 function 1 fileで纏めたかったのでやや冗長なコードになっちゃいました。。

## 実行

GitHubでコメント

{%img https://lh3.googleusercontent.com/KY7fwR-laXAJBQhJFv9VKzo_ydWFzeKep0G-3kgS0mM=w388-h128-no %}

Lambda発動

```
2015-08-11T15:42:53.108Z	a09e3c80-403f-1e15-bbb5-55b693433c0e	user: ijin2
2015-08-11T15:42:53.108Z	a09e3c80-403f-1e15-bbb5-55b693433c0e	comment: join
2015-08-11T15:42:55.417Z	a09e3c80-403f-1e15-bbb5-55b693433c0e	status code: 200
2015-08-11T15:42:55.418Z	a09e3c80-403f-1e15-bbb5-55b693433c0e	response:
{
    "state": "pending",
    "url": "https://api.github.com/teams/1234567/memberships/ijin2"
}
2015-08-11T15:42:55.418Z	a09e3c80-403f-1e15-bbb5-55b693433c0e	Added to the team
END RequestId: a09e3c80-403f-1e15-bbb5-55b693433c0e
REPORT RequestId: a09e3c80-403f-1e15-bbb5-55b693433c0e	Duration: 5211.48 ms	Billed Duration: 5300 ms Memory Size: 128 MB	Max Memory Used: 14 MB	 
```

```
2015-08-11T15:42:56.058Z	a25c66fc-403f-11e5-b291-25d4ee441689	Received event:
{
    "username": "ijin2",
    "icon_url": "https://avatars.githubusercontent.com/u/12809425?v=3",
    "text": "Added to the team"
}
2015-08-11T15:42:58.310Z	a25c66fc-403f-1e15-b291-25d4ee441689	200
2015-08-11T15:42:58.311Z	a25c66fc-403f-1e15-b291-25d4ee441689	ok
2015-08-11T15:42:58.311Z	a25c66fc-403f-1e15-b291-25d4ee441689	Successfully posted to Slack!
END RequestId: a25c66fc-403f-11e5-b291-25d4ee441689
REPORT RequestId: a25c66fc-403f-1e15-b291-25d4ee441689	Duration: 2268.06 ms	Billed Duration: 2300 ms Memory Size: 128 MB	Max Memory Used: 14 MB	
```

teamへの追加（招待）

{%img https://lh3.googleusercontent.com/lzamiiNraeHYD7FI6rrb8-403hxjzVRrAqGG6k3sBcc=w474-h191-no %}

Slackへ通知

{%img https://lh3.googleusercontent.com/EeEiHydBDc29YhVI6MBUcdj1WUzMXnf_Sis3U-URqhw=w388-h147-no %}

## 結論

簡単にできますね。実はこんな風に作っちゃった2週間後ぐらいにAWSの人も似たような[ブログ](https://aws.amazon.com/blogs/compute/dynamic-github-actions-with-aws-lambda/)を書いていた事を発見しましたが。まあ、こっちはKMSとSlack使ってるので。。

後、KMSのアイコンが欲しい！

