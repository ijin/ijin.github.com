---
layout: post
title: "Elastic Beanstalkへの簡単ssh"
date: 2015-11-04 08:43
comments: true
categories: 
- aws
---

AWS Elastic Beanstalk上で管理されているinstanceにsshするには`eb ssh`コマンドを使えば割りと簡単に接続できますが、アクセスキーの設定が必要で、開発者が多い場合にそれを発行してばら撒いて管理するのはかなり面倒です（IAM user/roleの割当にしても）。特に極稀にしか接続する必要がない場合。

そこで、API GatewayとLambdaでinstanceのpublic ipを返却するエンドポイントを作り、SSH configを設定すれば誰でもアクセスできるようにしてみました。

{% img https://lh3.googleusercontent.com/vqW8J9ASHnvZSjAZnwS-_4BMxUo37m6FJ3snn1c2j1dgbX1SVybDaz9TN0mG-fVLqKRUGDtUBO4uDbL2L_QILPXYNbKFCaCmG5Au4Ar8mNacZATEX3FjT2L08T-K4-JWwIXRvsd5SM1iwPu6az7ABb28N2ezTM5wp4-B4rB7tIHFzx8ZWCq9EvZQJGFnqVZpZxXEbdGc2edk6J4dd-QgrZRAmzx2XHXSK4oWnWCkbTB1VS-LGIk0ShMbqn1wjj8AugSoHozG1hsufsuEyeNxBN8-6lQowH_O8JWRdJc28ffNtEaQeGr6Xrj66DjjxYZmO_NdS9hnlgFzUAP4t9iWt1cRAVzDHHVXfqeROmtdXc1ogYLwQY3EOMpd42l-YDNzKbp_bi4H4YJ-Mr-pmF91szERTw2avQmc2MGYcUcsOATYxnyz47mNW1f6alfbzeUf1sFwlUZ1FqS3if6CWiaKnqNjRPvHHs8ccV7magouBQ0Qm3_DOgSKsuosLYXZzNYEuPZl1WYv-m2Hnz0pMglMUA42V_7zrl6lY-cleeY=w605-h431-no %}

## 前提条件 ##

- instanceにはログインユーザーのssh keyが登録積み（.ebextensions等で）
- ssh localhost可能である
- Elastic Beanstalkのenvironment名は数値で終わらない
- Auto ScalingのTermination Policyが`NewestInstance`

## Lambda ##

やってる事は単純で自動的に付与される`elasticbeanstalk:environment-name`タグのついたinstanceのipを（パラメータに応じて）返却するだけ。複数ある場合は、起動した順で。

せっかくなので新しくサポートされたPythonで記述。

{% gist bf735efbec42e1f96d4b %}

当然、LambdaのIAM roleで`ec2:Describe*`のAction許可も必要。

## Swagger ##

API Gatewayのresourceをコンソールでポチポチ作るのがイヤだったのでAPIの状態を[Swagger](http://swagger.io/)で定義してみた。awscliは[生REST APIが辛い](http://dev.classmethod.jp/cloud/aws/getting-started-with-api-gateway-lambda-integration/)。

編集しながらドキュメントも見れる[online editor](http://editor.swagger.io/#/)が結構便利。

{% gist a41059039937ffc8dada %}

## API Gateway ##

Swagger上でYAMLで作成した定義書はjsonとしてexportし、[aws-apigateway-importer](https://github.com/awslabs/aws-apigateway-importer)でAPIのresourceやmethodやらを生成する。

### import処理 ###

```
$ ./aws-api-import.sh -c swagger.json
2015-11-04 02:23:18,507 INFO - Attempting to create API from Swagger definition. Swagger file: swagger.json
reading from swagger.json
2015-11-04 02:23:18,649 INFO - Parsed Swagger with 2 paths
2015-11-04 02:23:18,655 INFO - Creating API with name EB API
2015-11-04 02:23:19,417 INFO - Removing default model Error
2015-11-04 02:23:19,634 INFO - Removing default model Empty
2015-11-04 02:23:19,872 INFO - Creating model for api id nx3r6d6yhh with name IP
2015-11-04 02:23:20,114 INFO - Generated json-schema for model IP: {"type":"object","properties":{"ip":{"type":"string","description":"IP address."}},"definitions":{}}
2015-11-04 02:23:20,329 INFO - Creating model for api id nx3r6d6yhh with name Error
2015-11-04 02:23:20,338 INFO - Generated json-schema for model Error: {"type":"object","properties":{"code":{"type":"integer","format":"int32"},"message":{"type":"string"},"fields":{"type":"string"}},"definitions":{}}
2015-11-04 02:23:20,770 INFO - Creating resource 'eb' on bd5pqsq37h
2015-11-04 02:23:21,420 INFO - Creating resource '{env_name}' on 8rjsbt
2015-11-04 02:23:22,079 INFO - Creating resource 'ip' on 4cg26s
2015-11-04 02:23:22,931 INFO - Creating method response for api nx3r6d6yhh and method GET and status 200
2015-11-04 02:23:23,144 INFO - Creating new model referenced from response: IPaddresses
2015-11-04 02:23:23,144 INFO - Creating model for api id nx3r6d6yhh with name IPaddresses
2015-11-04 02:23:23,153 INFO - Generated json-schema for model IPaddresses: {"type":"string","definitions":{}}
2015-11-04 02:23:23,770 WARN - Default response not supported, skipping
2015-11-04 02:23:23,772 INFO - Creating method parameter for api nx3r6d6yhh and method GET with name method.request.path.env_name
2015-11-04 02:23:23,998 INFO - Creating method for api id nx3r6d6yhh and resource id md6qcm with method get
2015-11-04 02:23:24,822 INFO - Creating resource '{server_num}' on qmd6mc
2015-11-04 02:23:25,689 INFO - Creating method response for api nx3r6d6yhh and method GET and status 200
2015-11-04 02:23:25,896 INFO - Found reference to existing model IPaddresses
2015-11-04 02:23:26,306 WARN - Default response not supported, skipping
2015-11-04 02:23:26,306 INFO - Creating method parameter for api nx3r6d6yhh and method GET with name method.request.path.env_name
2015-11-04 02:23:26,525 INFO - Creating method parameter for api nx3r6d6yhh and method GET with name method.request.path.server_number
2015-11-04 02:23:26,764 INFO - Creating method for api id nx3r6d6yhh and resource id 5udxeo with method get
```

これでAPI Gateway上に階層が出来上がり。

{% img https://lh3.googleusercontent.com/cM7s04iaMWI73VwJcH3C9lsnFjrrFpw0qqnUCM4hkmKWZiB3Pp0idjace8rxQYPCVwYzpucRKJep1j3CN9rDQ4uG0xJa4iXM_ua_lLo6LjxCus_lXV9aPCanWS17Ab6eVfgzH3wMDS1YvUhBq_6aLl3JmGmQVlZPFAUrOjcT-Xwra2o4VRKgaTbE5LQCvoBkzkylYhe6JECBu-lqBW0T9CnyhEsbJ1KTUwnIvLLTYtvTPtG6xEqoBMiCo26QLrASGg_k5fXW66jB-SH3fK-W-2TlYHwGdvlJunbuJp__AN2RUAhU64IPTQmZS-PfBTknMF273wF0DyqoL4UsS4HCmAMytc4LBx4s3iJdxFlPkMHPQ5eM9Q6REK92yP1a9FUqXKGF15FD9M6njOGvXFQYH8BsDruCa-HvxY7l6J0oj7PjyWwgq3_Jpd13F27rQkAcK13ni-sSB17wTRi-la8Tgu5ZMvnpUYBK-ARup3vHz791VQW-LVcdecok2Sao2wG4sUiUi5pMSOUabX02OMORVCu5OSxCdw-GxGSzikM=w350-h280-no %}

後は各`GET method`でLambdaへの関連付けをし、`Integration Request`と`Integration Response`の`mapping template`でパラメータやレスポンスを設定。（多分、この手順もswaggerで定義できるとは思うけど）


### integration request ###

{% img https://lh3.googleusercontent.com/JtD50KDCwP5ggylHzSuD7WiM-fr1QvVkZFpGFKD0efPbe_wvJPR0HbaTihe8wSlBNJzFYHAAwN9ZvKS1-R1WVGqbWT-t5PCdlptIVgJkk8xjDF0HJTRXeFEjF6F5Z2sp5wfY3BMgKZnyAM0CL6Ax5WtLNxCENl1SskcqajLDAapVQP8bj0UMlxUhsATwg9wh6zbS-jKp6tjf8YlkDD3PDFV_jhE2oVnzytNTlKBb5sU_kwMRFOGfQiUIpwgsrPq11af4ETcdFiU99lxa2ey1fgwxIaoEddLNOgMyObSd7LV0ISNY_kbZj0aRi2PFFmUfnQmYfc0UTajjAejDRms08vn6oy0-6kwGRjcKMq8zDSLFMCxgaSU4_2m2zh4T1zo54DJCHRm8ilXg_r3Z2u4IdBBHf4xYdvvs2J0p4ZT6GOQy6tMT0cNAFN5XUQiC6zDVutj2Ia1OQSlhUFRvnXwtivqGdMvpOIyUqoyuJ5RHLhO3hWP90SsW_DuILy6eXAHkSMHHpDPu5uemVBD29KfKsf67GS1bmJPwu5_9gX4=w836-h208-no %}

### integration response ###

{% img https://lh3.googleusercontent.com/-MuJyoH_nLN5jbjrcTsDW9qfsFUhDt6nHq_gNlYwMhVNZJqYL3C5qrgPwXLrPRXL0oRICG8xGpBVJpsLKIC3KhhFL4eHyo5LPZRVMbCXQ20UfT-tMV-Pt72-Rr7AKkhXDSzLcBANpRyJgioD5VlLb6YfqcnDLpczt1gnCOvnLHeWuuIf8NTa6q8hoGZXKyBo79Xapag_xvzbqco8AzSqu_yUxR6MXArEpp_we0JGhkJJHrve41LXNDX27QfC-Fj84feKgXY9CTgB1ytHWWDT_71gU05NwMdMisuQnAZPg9kRnoze55OBXu6c8TtzN5Idvdgmc9dbeKKxFG0f4bn_pt4lN6ZzOJ0ptOteVl-EpMDPk6M9_kwKerOc1GQbH47MEEYmQTXmoMeUp9ECJwhYdxdqu_LEB6ZxrjOC3VBPP6rE8sxBZJG8od_adtClp4UYU5-LBPm9_DYsde5hThh2su9PVBph8fbk42WE8I6o3BoOSciA1G9eIwfaa99OQh-4NZhJRVIGs7M4uMHVyUnDQk0RqtjFjf7TlCkeRC4=w811-h210-no %}

参考：

- [Amazon API GatewayでAWS Lambda関数にクエリ文字列をパラメータとして渡す](http://qiita.com/yuyakato/items/89fcef9746afbf48977a)
- [Amazon API Gatewayで文字列をクォーテーションなしで返却する](http://qiita.com/ijin/items/6edafc1f351a9de49b54)

### endpoint ###

Deployすれば、以下のAPIでipが取得可能。

`/$API_ENDPOINT/prod/eb/$EB_ENV_NAME/ip/[n]`

instance ip一覧

```
$ curl -s https://altvsxa2kj.execute-api.ap-northeast-1.amazonaws.com/prod/eb/my-ebenv-production/ip
54.189.149.5
54.189.168.83
```

特定のinstance ip

```
$ curl -s https://altvsxa2kj.execute-api.ap-northeast-1.amazonaws.com/prod/eb/my-ebenv-production/ip/1
54.189.149.5
```

ipの部分だけ、PATHじゃなくquery paramterにした方がAPI Gatewayの設定がシンプルだったけど、少しでもRESTfulにしたかったもので。（まあ、そもそもjson返してないけど）


## SSH Config ##

さて、ここからがキモです。

~/.ssh/config

```
Host my-ebenv-production*
  User ec2-user
  StrictHostKeyChecking no
  ProxyCommand ssh -A localhost -W `N=$(grep -o '[0-9]\+$' <<< %h || echo 1); E=$(perl -pe 's/\d+$//' <<< %h); curl -s https://altvsxa2kj.execute-api.ap-northeast-1.amazonaws.com/prod/eb/$E/ip/$N`:%p
```

上記の設定で`ssh $EB_ENV_NAME[n]`で指定したサーバに接続できるようになります。

仕組みとしては、ホスト名をparseしAPI Gatewayに適切なリクエストを行い、返却されるpublic ipにローカル端末自身を踏み台としてsshする`ProxyCommand`の設定です。


例えば、2番目（に起動した）のサーバに接続する場合は

```
$ ssh my-ebenv-production2
Last login: Wed Nov  4 01:19:14 2015 from ndx6-ppp413.tokyo.sannet.ne.jp
 _____ _           _   _      ____                       _        _ _
| ____| | __ _ ___| |_(_) ___| __ )  ___  __ _ _ __  ___| |_ __ _| | | __
|  _| | |/ _` / __| __| |/ __|  _ \ / _ \/ _` | '_ \/ __| __/ _` | | |/ /
| |___| | (_| \__ \ |_| | (__| |_) |  __/ (_| | | | \__ \ || (_| | |   <
|_____|_|\__,_|___/\__|_|\___|____/ \___|\__,_|_| |_|___/\__\__,_|_|_|\_\
                                       Amazon Linux AMI

This EC2 instance is managed by AWS Elastic Beanstalk. Changes made via SSH 
WILL BE LOST if the instance is replaced by auto-scaling. For more information 
on customizing your Elastic Beanstalk environment, see our documentation here: 
http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/customize-containers-ec2.html
```

1番目の場合は明示的に指定する必要はありません。

```
$ ssh my-ebenv-production
```

また、Auto Scalingでinstanceが増減せずに1台のみの場合はもっとシンプルに記述できます。

```
  ProxyCommand ssh -A localhost -W `curl -s https://altvsxa2kj.execute-api.ap-northeast-1.amazonaws.com/prod/eb/$E/ip`:%p
```

簡単！

でも、よくよく考えたら普通のAuto Scalingの場合でも利用できる事に気付きました。

## まとめ ##

- Swaggerなかなか面白い
- API GatewayとLambda便利
- Lambda pythonも良い。けど、rubyサポートも早う。。
