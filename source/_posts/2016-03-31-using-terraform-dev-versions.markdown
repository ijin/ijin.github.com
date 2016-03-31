---
layout: post
title: "Terraformで特定のブランチを使う"
date: 2016-03-31 16:38
comments: true
categories: 
- terraform
- golang
---

HashiCorpの[Terraform](https://terraform.io/)の開発は非常に活発であり日々進化していますが、リリースまでに待てない実装中や実験的な機能を使いたい場合は、自分で特定のブランチを変更・ビルドして使えます。


### リポジトリ取得

```
$ go get github.com/hashicorp/terraform
$ cd $GOPATH/src/github.com/hashicorp/terraform
$ git checkout new-aws-feature
```


### commit hash付与

どのコミットリビジョンを使っているか簡単に分かるように、versionにgitのcommit hashを指定。

```
sed -i -e "s/dev/dev - `git rev-parse --short HEAD`/" terraform/version.go
```

### ビルド

make devすると全pluginやproviderがコンパイルされ、非常に時間がかかる（特に最近は対応範囲の広がりが著しく、生成されるバイナリの総容量が増加傾向）。
よって、特定のproviderを指定して時間短縮する。以下はaws providerで作業している場合。

```
$ make core-dev plugin-dev PLUGIN=provider-aws
==> Checking that code complies with gofmt requirements...
/Users/ijin/golang/bin/stringer
go generate $(go list ./... | grep -v /vendor/)
go install github.com/hashicorp/terraform
go install github.com/hashicorp/terraform/builtin/bins/provider-aws
mv /Users/ijin/golang/bin/provider-aws /Users/ijin/golang/bin/terraform-provider-aws
```

### 確認

```
$ terraform version
Terraform v0.6.15-dev - 123abcd
```

Happy Terraforming!
