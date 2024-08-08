---
title: "Typescript × npmパッケージ で構成されたLambda関数をterraformでデプロイする"
emoji: "🐷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [typescript, npm, terraform, lambda, aws]
published: false
---

Typescriptで、外部のライブラリも利用したLambda関数をTerraformでデプロイできないものかと
考え、下記の記事を参考に環境を構築しました。

https://dev.classmethod.jp/articles/deploy-typescript-lambda-function-with-terraform/

しかし、上記の方法ではコードをs3にアップロードする際にTerraformのリソースの中で
aws cliのコマンドがベタ書きされているため、以下のような問題があると感じました。
- terraform destroy でリソースをうまく削除できない
- aws cli がローカル環境にない場合に動作しない
- Terraformのリソースとaws cliで異なるコンテキストが混ざって可読性と保守性が落ちる

そこで、勝手ながらリポジトリをforkして、大まかな思想を引き継ぎつつもより簡潔にデプロイが出来る構成を検討しました。

https://github.com/sniper-fly/typescript-lambda-function-with-terraform

結論から言うと、`terraform apply`だけでパッケージのインストール、ビルドなどを行うのは難しく、makefileを活用する方針にしました。
terraform の [archive_file](https://registry.terraform.io/providers/hashicorp/archive/latest/docs/data-sources/file)はterraform plan の段階で実行されるため、[terraform_data](https://developer.hashicorp.com/terraform/language/resources/terraform-data) (null_resourceの後継)の前に実行されてしまいます。
これでは、npmでビルドする前のzipイメージがAWSにアップロードされてしまうため、

terraform apply だけで下記の処理が実行できます。
- npm でのパッケージインストール
- パッケージをバンドルし、Typescriptファイルをjsにトランスパイルする
- Lambda関数のzip化
- デプロイメントパッケージをs3にアップロード
- Lambda関数のデプロイ
- そのたIAM roleやポリシーの設定

かつ、2回目以降の変更では、tsファイル、package.json等の変更があった際にのみデプロイ出来ます。

## ディレクトリ構成
```
├── lambda
│   ├── dist
│   │   ├── index.js (後に作成される)
│   │   └── index.js.map (後に作成される)
│   ├── index.ts
│   ├── package.json
│   ├── package-lock.json
├── lambda.tf
├── locals.tf
├── main.tf
├── providers.tf
```

## コードの説明

```tf
resource "null_resource" "lambda_build" {
  triggers = {
    code_diff = join("", [
      for file in fileset(local.src_path, "{*.ts, package*.json}")
      : filebase64("${local.src_path}/${file}")
    ])
  }
  provisioner "local-exec" {
    working_dir = local.src_path
    command     = "npm install"
    on_failure  = fail
  }
  provisioner "local-exec" {
    working_dir = local.src_path
    command     = "npm run build"
    on_failure  = fail
  }
}
```
`null_resource`というものを使っています。
https://registry.terraform.io/providers/hashicorp/null/latest/docs/resources/resource
これ

`triggers` に関しては、よく使われる構成だと思いますが
`local.src_path`(= ./lambda/) に存在するtsファイル、package.json, package-lock.json に変更があった際、このresourceは再作成されます。
