---
layout: post
title: "TerraformでAPI Gatewway"
date: 2016-04-28 18:01
comments: true
categories: 
- aws
- terraform
---

つい先日、[Terraform](https://terraform.io/)でずっと気になっていたAmazon API Gatewayの`selection_pattern`の[pullrequest](https://github.com/hashicorp/terraform/pull/5893)がmergeされました。

今まではAPI GWをInfrastructure As Codeで構築するにあたって複数のintegration responseパターンを返却できないのがネックだったのが、ようやく解決しました。途中までTerraformで作って、その後に以下のようにawscliで追加するというちょっと煩わしい手順でした。

```
REST_ID=$(aws apigateway get-rest-apis --query 'items[?name==`my_api`].id' --output text)
RESOURCE_ID=$(aws apigateway get-resources --rest-api-id $REST_ID --query 'items[?path==`/my_path`].id' --output text)
aws apigateway put-integration-response --rest-api-id $REST_ID --resource-id $RESOURCE_ID --http-method GET --status-code 400 --response-templates '{"application/json": "$input.path('$').errorMessage"}' --selection-pattern "[^0-9](.|\n)*" 
```

というわけで、早速実験。お題は以前[紹介した](/blog/2015/11/04/elastic-beanstalk-easy-ssh/)Elastic Beanstalk ssh用のAPI GWで。

### Terraform version

まずは、`master`にmergeされた開発版Terraformのビルド。やり方は[こちら](/blog/2016/03/31/using-terraform-dev-versions/)。

```
$ terraform version
Terraform v0.6.16-dev - 5cd27c2
```

### Terraform file

API GWのterraform化はこんな感じで。

```
provider "aws" {
    region = "ap-northeast-1"
    allowed_account_ids = ["${var.names.account_id}"]
}

variable "names" {
    default = {
        account_id = "$ACCOUNT_ID"
    }
}
```

{% gist db027846fefb339187e3f2833fe2d034 %}


### Terraform plan/apply

{% gist ce0147db0a9033837a282011fc46d836 %}


{% gist 06df5cf90c235cf3e4226d5d34bb80d8 %}

できた！

### 考察

- 依存関係

リソースを作成するのに並列処理が出来なかったり、依存関係がまだうまく対応してないので、`depends_on`を駆使する必要があるのが若干まだ面倒。ない場合は、`BadRequestException: Unable to complete operation due to concurrent modification. Please try again later`や`BadRequestException: No integration defined for method status code: 400`等のエラーが発生する。

- Integration/Method Response Headers

CORS等の設定するする際には`Access-Control-Allow-Origin`等のヘッダーをMethodやIntegrationのResponse Headerに設定をする必要があるけど、`.`の扱い問題で未対応（(#2143)[https://github.com/hashicorp/terraform/issues/2143]）。それまではawscliで以下のようにすると事で回避。。

`aws --region ap-northeast-1 apigateway update-integration-response --rest-api-id $rest_id --resource-id $appo_resource_id --http-method OPTIONS --status-code 200 --patch-operations op=add,path="/responseParameters/method.response.header.Access-Control-Allow-Headers",value="\"'Content-Type,X-Amz-Date,Authorization,X-Api-Key, Access-Control-Allow-Origin, x-amz-security-token'\""`

Issueはこの前上げたので（(#6092)[https://github.com/hashicorp/terraform/issues/6092]）、ウォッチしておくと良いと思う

- Infrastructure as Code

API Gatewayはresource, method, integration, method response、integration response等を記述しないといけないので、どうしても長めになってしまう。場合によってはSwaggerでやった方が楽だったりするかも。その場合は、*Infrastructure as YAML*になってしまうけど。ただ、そのままでは使えなく整形する必要があるので、トレードオフがあったり。
Infra as Yaml
