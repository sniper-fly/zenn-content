---
title: "Lambda Layerを使ってPlaywrightを動かす環境をTerraformで構築する"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [AWS,Lambda,Playwright,Terraform]
published: true
---
## 結論
[GitHub - sniper-fly/playwright-on-lambda-with-tf: You can ru...](https://github.com/sniper-fly/playwright-on-lambda-with-tf)
上記のリポジトリをクローンして、makeコマンドを叩けばサンプルの関数が構築できます。
あとはAWSコンソールやCLIで適宜テストが可能です。

以下の要件を満たしていれば基本的に問題なく構築できるはずです。
- makeコマンドの実行が出来る
- TerraformでAWSリソースの作成が出来る
- npmコマンドが実行可能

## 経緯
Lambda上でPlaywrightを動かしてみたかったのですが、殆どがDockerイメージを使うものでした。動作させるだけであればこちらでも可ですが、
- ECRでのイメージの管理が個人的にTerraformではやりづらい
- ECRで大量のプライベートイメージを作ると課金額がバカにならない
ということでどうにかLambdaレイヤーだけで動作させられないか、と考えました。

[LambdaでPlaywrightを動かす(Lambdaレイヤー / コンテナ) | 豆蔵デベロッパーサイト](https://developer.mamezou-tech.com/blogs/2024/07/19/lambda-playwright-container-tips/)
上記サイトでLambdaレイヤーを使う方法がとても詳しく解説されています。
個人的にはAWS CDKではなくTerraformで動作させたかったのですが他に参考になる記事がなかったのが執筆動機です。

## 解説

### index.ts
下記が`index.ts`になります。`scrapingTitle`関数を呼び出して、結果をそのまま返しています。
```ts
import { Context, APIGatewayProxyResult, APIGatewayEvent } from "aws-lambda";
import { scrapingTitle } from "./scrapingTitle";

export const handler = async (
  event: APIGatewayEvent,
  context: Context
): Promise<APIGatewayProxyResult> => {
  console.log(`Event: ${JSON.stringify(event, null, 2)}`);
  console.log(`Context: ${JSON.stringify(context, null, 2)}`);
  return {
    statusCode: 200,
    body: JSON.stringify({
      message: await scrapingTitle("https://example.com"),
      // title: Example Domain
    }),
  };
};
```

### scrapingTitle.ts
下記が`scrapingTitle`関数になります。Playwrightの処理が記載されていて、
今回は適当なurlにアクセスしてタイトルを取得するだけの動作になっています。
```ts
import { chromium as playwright } from "playwright-core";
import chromium from "@sparticuz/chromium";
import "dotenv/config";

export async function scrapingTitle(url: string) {
  const browser = await playwright.launch({
    args: chromium.args,
    executablePath: await chromium.executablePath(),
    headless: true,
  });
  const page = await browser.newPage();
  await page.goto(url);
  const pageTitle = await page.title();

  if (process.env.NODE_ENV !== "production") { // ...(1)
    await browser.close();
  }
  return pageTitle;
}
```
(1) の部分に関してですが、これに関して理由はわかりませんが
lambda上でPlaywrightを実行した際、なぜかbrowser.closeを行うとエラーになってしまうため、環境変数で本番環境かどうかを判断しています。

### package.json
次は`package.json`です。注目すべきは`build`のスクリプトです。
`esbuild index.ts --bundle --minify --sourcemap --platform=node --target=es2020 --outfile=dist/index.js --external:@sparticuz/chromium`
```json
{
  "name": "helloworld",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "build": "esbuild index.ts --bundle --minify --sourcemap --platform=node --target=es2020 --outfile=dist/index.js --external:@sparticuz/chromium"
  },
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "@sparticuz/chromium": "^130.0.0",
    "@types/aws-lambda": "^8.10.101",
    "@types/node": "^22.8.1",
    "esbuild": "^0.14.47",
    "ts-node": "^10.9.2",
    "typescript": "^5.6.3"
  },
  "dependencies": {
	"chromium-bidi": "^0.8.1",
    "dotenv": "^16.4.5",
    "playwright-core": "^1.48.2"
  }
}
```
`--external`に`@sparticuz/chromium`が指定されています。
これがないと、ローカルでのコンパイル時に`chromium.executablePath()`の値が決定されてしまいlambda上で上手く動作しないため重要な項目になります。

また`dependencies`に`chromium-bidi`が指定されていますが、これはchromium130.0.0以降で必要なパッケージのようです。
こちらのpackageも先程の`@sparticuz/chromium`のようにexternal指定することでもコンパイル自体は可能です。それ以前のバージョンのchromiumにはこのパッケージは不要である模様です。
（申し訳ないですがこちらについては詳細が調査しきれていません。）

### lambda.tf
`lambda.tf`には一部に以下の記述があります。
```hcl
resource "aws_lambda_function" "helloworld" {
  function_name = "typescript-sample-helloworld"

  s3_bucket = aws_s3_bucket.lambda_assets.bucket
  s3_key    = aws_s3_object.package.key

  source_code_hash = data.archive_file.lambda_package.output_base64sha256
  handler          = "index.handler"

  role        = aws_iam_role.iam_for_lambda.arn
  runtime     = "nodejs20.x"
  timeout     = "60"
  memory_size = "600"

  environment {
    variables = {
      NODE_ENV = "production"
    }
  }

  layers = [aws_lambda_layer_version.chromium_layer.arn]
}
```
`memory_size`ですが、このコードを動かすだけでも大体500MB程利用するようなので、余裕を見て600MBを指定しています。また、`timeout`も`60`に設定されていますが、コードが複雑になって時間がかかるプログラムになってきた場合は様子を見て変更する必要があります。

### その他
また、今回のTerraformの設定では、chromiumのzipレイヤーを各関数毎にS3にアップロードする形式になっていますが、下記のようにlambdaレイヤーを公開している例もあるので、こちらを利用したり、同じlambdaレイヤーを複数の関数で使い回すのも良いかと思います。
[GitHub - shelfio/chrome-aws-lambda-layer: 58 MB Google Chrom...](https://github.com/shelfio/chrome-aws-lambda-layer)

本リポジトリでは、対応しているchromiumのバージョンが違うなどの場合は、レポジトリの`local.tf`、`makefile`の`chromium_version`の記述をよしなに書き換えて利用できるようになっています。

また、`check_update.sh`というファイルがありmakefileで実行されていますが、
これはpakcage.jsonやtypescriptファイルを読み込んでハッシュ値を作成してファイルに保存し、何か修正があった場合にのみ再度ビルドが走るようにするための独自実装です。

何かファイルを編集しても変更が反映されない、などの不具合がありましたらこちらの`check_update.sh`の不具合の可能性もあるので、普通に`lambda`ディレクトリで`npm run build`してみてください。
