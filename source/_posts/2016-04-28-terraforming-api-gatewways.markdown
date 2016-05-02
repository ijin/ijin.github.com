---
layout: post
title: "TerraformでAPI Gatewway"
date: 2016-04-28 18:01
comments: true
categories: 
- aws
- terraform
---

つい先日、[Terraform](https://terraform.io/)でずっと気になっていたAmazon API Gatewayの`selection_pattern`の[pull request](https://github.com/hashicorp/terraform/pull/5893)がmergeされました。

今まではAPI GWをInfrastructure As Codeで構築するにあたって複数のintegration responseパターンを返却できないのがネックだったのが、これでようやく解決。途中までTerraformで作って、その後に以下のようにawscliで追加するというちょっと煩わしい手順でした。

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

{% gist db027846fefb339187e3f2833fe2d034 %}

(*) permissionはlambda作成後に許可

### Terraform plan/apply

```
$ terraform plan
Refreshing Terraform state prior to plan...


The Terraform execution plan has been generated and is shown below.
Resources are shown in alphabetical order for quick scanning. Green resources
will be created (or destroyed and then created if an existing resource
exists), yellow resources are being changed in-place, and red resources
will be destroyed.

Note: You didn't specify an "-out" parameter to save this plan, so when
"apply" is called, Terraform can't guarantee this is what will execute.

+ aws_api_gateway_deployment.eb_deployment
    rest_api_id: "" => "${aws_api_gateway_rest_api.EB.id}"
    stage_name:  "" => "prod"

+ aws_api_gateway_integration.ip_get
    http_method:                        "" => "GET"
    integration_http_method:            "" => "POST"
    request_templates.#:                "" => "1"
    request_templates.application/json: "" => "{ \"env_name\": \"$input.params('env_name')\" }"
    resource_id:                        "" => "${aws_api_gateway_resource.ip.id}"
    rest_api_id:                        "" => "${aws_api_gateway_rest_api.EB.id}"
    type:                               "" => "AWS"
    uri:                                "" => "arn:aws:apigateway:ap-northeast-1:lambda:path/2015-03-31/functions/arn:aws:lambda:ap-northeast-1:123456789012:function:eb_ip/invocations"

+ aws_api_gateway_integration.server_num_get
    http_method:                        "" => "GET"
    integration_http_method:            "" => "POST"
    request_templates.#:                "" => "1"
    request_templates.application/json: "" => "{\n \"env_name\": \"$input.params('env_name')\",\n \"server_num\": \"$input.params('server_num')\" \n}"
    resource_id:                        "" => "${aws_api_gateway_resource.server_num.id}"
    rest_api_id:                        "" => "${aws_api_gateway_rest_api.EB.id}"
    type:                               "" => "AWS"
    uri:                                "" => "arn:aws:apigateway:ap-northeast-1:lambda:path/2015-03-31/functions/arn:aws:lambda:ap-northeast-1:123456789012:function:eb_ip/invocations"

+ aws_api_gateway_integration_response.ip_get_200
    http_method:                         "" => "GET"
    resource_id:                         "" => "${aws_api_gateway_resource.ip.id}"
    response_templates.#:                "" => "1"
    response_templates.application/json: "" => "$input.path('$')"
    rest_api_id:                         "" => "${aws_api_gateway_rest_api.EB.id}"
    status_code:                         "" => "200"

+ aws_api_gateway_integration_response.ip_get_400
    http_method:                         "" => "GET"
    resource_id:                         "" => "${aws_api_gateway_resource.ip.id}"
    response_templates.#:                "" => "1"
    response_templates.application/json: "" => "$input.path('$').errorMessage"
    rest_api_id:                         "" => "${aws_api_gateway_rest_api.EB.id}"
    selection_pattern:                   "" => "[^0-9](.|\n)*"
    status_code:                         "" => "400"

+ aws_api_gateway_integration_response.server_num_get_200
    http_method:                         "" => "GET"
    resource_id:                         "" => "${aws_api_gateway_resource.server_num.id}"
    response_templates.#:                "" => "1"
    response_templates.application/json: "" => "$input.path('$')"
    rest_api_id:                         "" => "${aws_api_gateway_rest_api.EB.id}"
    status_code:                         "" => "200"

+ aws_api_gateway_integration_response.server_num_get_400
    http_method:                         "" => "GET"
    resource_id:                         "" => "${aws_api_gateway_resource.server_num.id}"
    response_templates.#:                "" => "1"
    response_templates.application/json: "" => "$input.path('$').errorMessage"
    rest_api_id:                         "" => "${aws_api_gateway_rest_api.EB.id}"
    selection_pattern:                   "" => "[^0-9](.|\n)*"
    status_code:                         "" => "400"

+ aws_api_gateway_method.ip_get
    api_key_required: "" => "0"
    authorization:    "" => "NONE"
    http_method:      "" => "GET"
    resource_id:      "" => "${aws_api_gateway_resource.ip.id}"
    rest_api_id:      "" => "${aws_api_gateway_rest_api.EB.id}"

+ aws_api_gateway_method.server_num_get
    api_key_required: "" => "0"
    authorization:    "" => "NONE"
    http_method:      "" => "GET"
    api_key_required: "" => "0"
    authorization:    "" => "NONE"
    http_method:      "" => "GET"
    resource_id:      "" => "${aws_api_gateway_resource.server_num.id}"
    rest_api_id:      "" => "${aws_api_gateway_rest_api.EB.id}"

+ aws_api_gateway_method_response.ip_200
    http_method: "" => "GET"
    resource_id: "" => "${aws_api_gateway_resource.ip.id}"
    rest_api_id: "" => "${aws_api_gateway_rest_api.EB.id}"
    status_code: "" => "200"

+ aws_api_gateway_method_response.ip_400
    http_method: "" => "GET"
    resource_id: "" => "${aws_api_gateway_resource.ip.id}"
    rest_api_id: "" => "${aws_api_gateway_rest_api.EB.id}"
    status_code: "" => "400"

+ aws_api_gateway_method_response.server_num_200
    http_method: "" => "GET"
    resource_id: "" => "${aws_api_gateway_resource.server_num.id}"
    rest_api_id: "" => "${aws_api_gateway_rest_api.EB.id}"
    status_code: "" => "200"

+ aws_api_gateway_method_response.server_num_400
    http_method: "" => "GET"
    resource_id: "" => "${aws_api_gateway_resource.server_num.id}"
    rest_api_id: "" => "${aws_api_gateway_rest_api.EB.id}"
    status_code: "" => "400"

+ aws_api_gateway_resource.eb
    parent_id:   "" => "${aws_api_gateway_rest_api.EB.root_resource_id}"
    path:        "" => "<computed>"
    path_part:   "" => "eb"
    rest_api_id: "" => "${aws_api_gateway_rest_api.EB.id}"

+ aws_api_gateway_resource.env_name
    parent_id:   "" => "${aws_api_gateway_resource.eb.id}"
    path:        "" => "<computed>"
    path_part:   "" => "{env_name}"
    rest_api_id: "" => "${aws_api_gateway_rest_api.EB.id}"

+ aws_api_gateway_resource.ip
    parent_id:   "" => "${aws_api_gateway_resource.env_name.id}"
    path:        "" => "<computed>"
    path_part:   "" => "ip"
    rest_api_id: "" => "${aws_api_gateway_rest_api.EB.id}"

+ aws_api_gateway_resource.server_num
    parent_id:   "" => "${aws_api_gateway_resource.ip.id}"
    path:        "" => "<computed>"
    path_part:   "" => "{server_num}"
    rest_api_id: "" => "${aws_api_gateway_rest_api.EB.id}"

+ aws_api_gateway_rest_api.EB
    description:      "" => "get EB info"
    name:             "" => "EB"
    root_resource_id: "" => "<computed>"


Plan: 18 to add, 0 to change, 0 to destroy.
```

```
$ terraform apply
aws_api_gateway_rest_api.EB: Creating...
  description:      "" => "get EB info"
  name:             "" => "EB"
  root_resource_id: "" => "<computed>"
aws_api_gateway_rest_api.EB: Creation complete
aws_api_gateway_resource.eb: Creating...
  parent_id:   "" => "k9x3d7qlhd"
  path:        "" => "<computed>"
  path_part:   "" => "eb"
  rest_api_id: "" => "mdsyn3w42a"
aws_api_gateway_resource.eb: Creation complete
aws_api_gateway_resource.env_name: Creating...
  parent_id:   "" => "nr2lkm"
  path:        "" => "<computed>"
  path_part:   "" => "{env_name}"
  rest_api_id: "" => "mdsyn3w42a"
aws_api_gateway_resource.env_name: Creation complete
aws_api_gateway_resource.ip: Creating...
  parent_id:   "" => "g29h7n"
  path:        "" => "<computed>"
  path_part:   "" => "ip"
  rest_api_id: "" => "mdsyn3w42a"
aws_api_gateway_resource.ip: Creation complete
aws_api_gateway_resource.server_num: Creating...
  parent_id:   "" => "sthj28"
  path:        "" => "<computed>"
  path_part:   "" => "{server_num}"
  rest_api_id: "" => "mdsyn3w42a"
aws_api_gateway_method.ip_get: Creating...
  api_key_required: "" => "0"
  authorization:    "" => "NONE"
  http_method:      "" => "GET"
  resource_id:      "" => "sthj28"
  rest_api_id:      "" => "mdsyn3w42a"
aws_api_gateway_method.ip_get: Creation complete
aws_api_gateway_method_response.ip_200: Creating...
  http_method: "" => "GET"
  resource_id: "" => "sthj28"
  rest_api_id: "" => "mdsyn3w42a"
  status_code: "" => "200"
aws_api_gateway_integration.ip_get: Creating...
  http_method:                        "" => "GET"
  integration_http_method:            "" => "POST"
  request_templates.#:                "" => "1"
  request_templates.application/json: "" => "{ \"env_name\": \"$input.params('env_name')\" }"
  resource_id:                        "" => "sthj28"
  rest_api_id:                        "" => "mdsyn3w42a"
  type:                               "" => "AWS"
  uri:                                "" => "arn:aws:apigateway:ap-northeast-1:lambda:path/2015-03-31/functions/arn:aws:lambda:ap-northeast-1:123456789012:function:eb_ip/invocations"
aws_api_gateway_resource.server_num: Creation complete
aws_api_gateway_method.server_num_get: Creating...
  api_key_required: "" => "0"
  authorization:    "" => "NONE"
  http_method:      "" => "GET"
  resource_id:      "" => "9w68fs"
  rest_api_id:      "" => "mdsyn3w42a"
aws_api_gateway_method_response.ip_200: Creation complete
aws_api_gateway_method_response.ip_400: Creating...
  http_method: "" => "GET"
  resource_id: "" => "sthj28"
  rest_api_id: "" => "mdsyn3w42a"
  status_code: "" => "400"
aws_api_gateway_integration_response.ip_get_200: Creating...
  http_method:                         "" => "GET"
  resource_id:                         "" => "sthj28"
  response_templates.#:                "" => "1"
  response_templates.application/json: "" => "$input.path('$')"
  rest_api_id:                         "" => "mdsyn3w42a"
  status_code:                         "" => "200"
aws_api_gateway_integration.ip_get: Creation complete
aws_api_gateway_method.server_num_get: Creation complete
aws_api_gateway_method_response.server_num_200: Creating...
  http_method: "" => "GET"
  resource_id: "" => "9w68fs"
  rest_api_id: "" => "mdsyn3w42a"
  status_code: "" => "200"
aws_api_gateway_integration.server_num_get: Creating...
  http_method:                        "" => "GET"
  integration_http_method:            "" => "POST"
  request_templates.#:                "" => "1"
  request_templates.application/json: "" => "{\n \"env_name\": \"$input.params('env_name')\",\n \"server_num\": \"$input.params('server_num')\" \n}"
  resource_id:                        "" => "9w68fs"
  rest_api_id:                        "" => "mdsyn3w42a"
  type:                               "" => "AWS"
  uri:                                "" => "arn:aws:apigateway:ap-northeast-1:lambda:path/2015-03-31/functions/arn:aws:lambda:ap-northeast-1:123456789012:function:eb_ip/invocations"
aws_api_gateway_integration_response.ip_get_200: Creation complete
aws_api_gateway_method_response.ip_400: Creation complete
aws_api_gateway_integration_response.ip_get_400: Creating...
  http_method:                         "" => "GET"
  resource_id:                         "" => "sthj28"
  response_templates.#:                "" => "1"
  response_templates.application/json: "" => "$input.path('$').errorMessage"
  rest_api_id:                         "" => "mdsyn3w42a"
  selection_pattern:                   "" => "[^0-9](.|\n)*"
  status_code:                         "" => "400"
aws_api_gateway_integration.server_num_get: Creation complete
aws_api_gateway_method_response.server_num_200: Creation complete
aws_api_gateway_method_response.server_num_400: Creating...
  http_method: "" => "GET"
  resource_id: "" => "9w68fs"
  rest_api_id: "" => "mdsyn3w42a"
  status_code: "" => "400"
aws_api_gateway_method_response.server_num_400: Creation complete
aws_api_gateway_integration_response.ip_get_400: Creation complete
aws_api_gateway_integration_response.server_num_get_200: Creating...
  http_method:                         "" => "GET"
  resource_id:                         "" => "9w68fs"
  response_templates.#:                "" => "1"
  response_templates.application/json: "" => "$input.path('$')"
  rest_api_id:                         "" => "mdsyn3w42a"
  status_code:                         "" => "200"
aws_api_gateway_integration_response.server_num_get_200: Creation complete
aws_api_gateway_integration_response.server_num_get_400: Creating...
  http_method:                         "" => "GET"
  resource_id:                         "" => "9w68fs"
  response_templates.#:                "" => "1"
  response_templates.application/json: "" => "$input.path('$').errorMessage"
  rest_api_id:                         "" => "mdsyn3w42a"
  selection_pattern:                   "" => "[^0-9](.|\n)*"
  status_code:                         "" => "400"
aws_api_gateway_integration_response.server_num_get_400: Creation complete
aws_api_gateway_deployment.eb_deployment: Creating...
  rest_api_id: "" => "mdsyn3w42a"
  stage_name:  "" => "prod"
aws_api_gateway_deployment.eb_deployment: Creation complete

Apply complete! Resources: 18 added, 0 changed, 0 destroyed.

The state of your infrastructure has been saved to the path
below. This state is required to modify and destroy your
infrastructure, so keep it safe. To inspect the complete state
use the `terraform show` command.

State path: terraform.tfstate
```

できた！

### 考察

- 依存関係

リソースを作成するのに並列処理が出来なかったり、依存関係がまだうまく対応できてないので、`depends_on`を駆使する必要があるのが若干まだ面倒。ない場合は、`BadRequestException: Unable to complete operation due to concurrent modification. Please try again later`や`BadRequestException: No integration defined for method status code: 400`等のエラーが発生する。

- Integration/Method Response Headers

CORS等の設定するする際には`Access-Control-Allow-Origin`等のヘッダーをMethodやIntegrationのResponse Headerに設定をする必要があるけど、`.`の扱い問題で未対応（[#2143](https://github.com/hashicorp/terraform/issues/2143)）。それまではawscliで以下のようにすると事で回避。

```
aws apigateway update-integration-response --rest-api-id $rest_id --resource-id $appo_resource_id --http-method OPTIONS --status-code 200 --patch-operations op=add,path="/responseParameters/method.response.header.Access-Control-Allow-Headers",value="\"'Content-Type,X-Amz-Date,Authorization,X-Api-Key, Access-Control-Allow-Origin, x-amz-security-token'\""
```

Issueはこの前上げたので（[#6092](https://github.com/hashicorp/terraform/issues/6092)）、ウォッチしておくと良い。

- Infrastructure as Code

API Gatewayは`resource`, `method`, `integration`, `method response`、`integration response`等を記述しないといけないので、どうしてもコードが多くになってしまう事からSwaggerでやった方が楽だったりするかも。ただ、その場合は**Infrastructure as YAML**になってしまうけど。。また、YAMLは整形してからimportする必要があったりするので、その辺は諸々トレードオフかなぁ。

