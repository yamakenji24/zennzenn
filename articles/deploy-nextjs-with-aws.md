---
title: "API GatewayとLambdaでNext.jsをAWSにデプロイする with Terraform"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "Terraform", "AWS", "Lambda", "apigateway"]
published: true
---

# はじめに

こんにちは！元フロントエンドエンジニアの@yamakenji です。
最近はフロントエンドだけでなく PHP や AWS 周りといった Web 周りに加えて、Swift や Kotlin といったモバイル系にも少し手を出し始めています。

今回は、Terraform がよくわからない....実際に何か動かしてみようとなったので簡単な Next.js アプリを AWS 上にデプロイしてみます。
Terraform を使用して CloudFront、API Gateway、Lambda Container を管理し、GitHub Actions でデプロイするところまでやっていきたいと思います。

本記事では、具体的な手順を踏みながら、AWS 上でのデプロイ方法を解説していきます。

- Terraform や AWS 周りのサービスについて
- Terraform を使って AWS リソースを作成する
- Next.js アプリケーションのビルドと ECR、S3、Lambda へのプッシュ
- Route53 を使用したカスタムドメインの設定
- GitHub Actions と OIDCを使用して自動デプロイ

![](/images/deploy-nextjs-with-aws/diagram.png)

# Terraform や AWS 周りのサービスについて

## Terraform

AWS 上にリソースを作成するために、Terraform を使用します。  
Terraform は、コードでインフラストラクチャを管理するためのオープンソースツールです。
AWS 上に必要なリソースを定義したコードを Terraform に渡すことで、AWS 上にリソースを作成、更新、削除することができます。
Terraform を使用すると、AWS リソースの作成や変更、削除などを自動化することができます。

まずはじめに、Terraform の初期化を行います。
Terraform から AWS リソースを編集するために、AWS アクセスキーとシークレットアクセスキーが必要なので、
AWS CLI を使用して IAM ユーザーの認証情報のアクセスキーとシークレットアクセスキーを設定します。
設定方法の詳細は以下の[AWS ドキュメント](https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/cli-configure-files.html)を参考にしてください。

```
$ aws configure
AWS Access Key ID [****************GIXI]:
AWS Secret Access Key [****************dMYm]:
Default region name [ap-northeast-1]:
Default output format [None]:
```

Terraform の初期設定を記述するために、main.tf というファイルを作成します。
以下のコードは、AWS プロバイダを初期化するための最小限の設定例です。
この例では、アクセスキーとシークレットアクセスキーを.aws/credentials から読み込むように設定しています。

```hcl: main.tf
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "3.56.0"
    }
  }
}

provider "aws" {
  region  = var.region
  profile = "default" //　defaultあるいは自身のprofile_nameを設定
}
```

main.tf で、`var.region`と呼んでいる箇所がありますが、これは variable.tf の中身をよんでいます。
variable.tf には、Terraform コードで使用する変数の宣言を記述することが一般的です。変数を使用することで、コード内に固定値を直接書く代わりに、外部から値を指定できます。また、異なる環境に対して同じコードを使用する際にも、変数を使用することで簡単に値を変更できます。

```hcl: variable.tf
variable "region" {
  default = "ap-northeast-1"
}

```

作成したら、以下のコマンドを実行して Terraform の初期化、確認、実行を行います

```shell
// 初期化
$ terraform init
// どのようなリソースを作成・変更・削除するかを確認
$ terraform plan
// リソースの作成・変更・削除の実行
$ terraform apply
```

## CloudFront, API Gateway, Lambda Container の簡単な説明

CloudFront は、AWS のグローバルなコンテンツ配信ネットワーク（CDN）で、Web サイトやアプリケーションの静的および動的コンテンツを高速に配信することができます。CloudFront は、オリジンサーバーからコンテンツを取得し、世界中のエッジロケーションにキャッシュされたコンテンツを提供することができます。これにより、ユーザーにより高速でスムーズな Web 体験を提供できます。

[API Gateway](https://docs.aws.amazon.com/ja_jp/apigateway/latest/developerguide/welcome.html)は、API の作成、公開、管理を行うためのフルマネージド型のサービスです。API Gateway を使用することで、RESTful API や WebSocket API などを作成し、バックエンドの AWS サービス、Lambda 関数、HTTP エンドポイントなどに接続することができます。API Gateway は、認証、認可、API キーの管理、アクセス制御、モニタリング、ドキュメント生成などの機能を提供することができます。

[Lambda Container](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/images-create.html)は、AWS Lambda で実行するための Docker コンテナを作成するためのサービスです。Lambda Container を使用することで、AWS Lambda でサポートされていないランタイムやカスタムコンテナイメージを作成し、Lambda 上で実行することができます。
また、コンテナを実行するサービスに ECS もありますが、実行するコンテナサーバーを管理する必要があるかどうかという大きな違いがあります。
Lambda Container は、Lambda と同様にサーバーレスであり、AWS 側でコンテナを実行するためのサーバーを自動的に管理してくれます。
一方、ECS は、コンテナを実行するサービスですが、ユーザー側でコンテナを実行するためのサーバーを管理する必要があります。そのため、ECS を使用する場合は、サーバーのメンテナンスコストがかかることになります。
今回は、なるべく管理を減らしたいので、サーバーレスな構成である Lambda を採用して進めます。

# Terraform を使って AWS リソースを作成する

## CloudFront の作成

さっそく、Cloudfront のリソースを作成していきます。
今回は構成的に cloudFront から API Gateway へ繋げるだけのため、`default_cache_behavior`の`target_origin_id`を API Gateway の ID に指定します(後ほど行います)。
また、`price_class`に`PriceClass_200`を指定しています。CloudFront では、世界中にエッジロケーションがあり、それぞれコストが異なります。
価格帯は`料金クラスAll`、`料金クラス200`、`料金クラス100`に三つが選択でき、それぞれでリクエストできるロケーションが異なります。日本リージョンを利用することができる価格帯が`料金クラス200`であったので、今回はそれを利用します。
[CloudFront ディストリビューションの価格クラスを選択する](https://docs.aws.amazon.com/ja_jp/AmazonCloudFront/latest/DeveloperGuide/PriceClass.html)

```hcl: cloudfront.tf
resource "aws_cloudfront_distribution" "example_distribution" {
  provider = aws.us-east-1

  enabled             = true
  is_ipv6_enabled     = true
  price_class         = "PriceClass_200"

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "TODO"

    forwarded_values {
      query_string = false

      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "allow-all"
    min_ttl                = 0
    default_ttl            = 3600
    max_ttl                = 86400
  }

  viewer_certificate {
    cloudfront_default_certificate = true
    minimum_protocol_version       = "TLSv1.2_2021"
    ssl_support_method             = "sni-only"
  }

  tags = {
    Name        = "Example CloudFront Distribution"
    Environment = "dev"
  }
}
```

## API Gateway と Lambda Container の作成
次に、API GatewayとLambdaコンテナを作成していきます。
まずは、API Gateway作成します。
Lambdaにリクエストを投げられるように、AWS_PROXYを設定していますが、Lambdaは後ほど作成するため、一旦WIPにしています。


```hcl: api-gateway.tf
# REST APIの作成
resource "aws_api_gateway_rest_api" "example_api_gateway" {
  name = "example_api_gateway"

  endpoint_configuration {
    types = ["REGIONAL"]
  }
}

# ANYメソッドでルートに対するリクエストを許可するためのメソッドの作成
# 認証は不要で、全てのユーザーがリクエストを送信可能
resource "aws_api_gateway_method" "example_next_api" {
  rest_api_id   = aws_api_gateway_rest_api.example_api_gateway.id
  resource_id   = aws_api_gateway_rest_api.example_api_gateway.root_resource_id
  http_method   = "ANY"
  authorization = "NONE"
}

# proxy+というワイルドカードを含んでいるため、全てのリソースパスで処理される
resource "aws_api_gateway_resource" "example_resource_paths" {
  rest_api_id = aws_api_gateway_rest_api.example_api_gateway.id
  parent_id   = aws_api_gateway_rest_api.example_api_gateway.root_resource_id
  path_part   = "{proxy+}"
}

resource "aws_api_gateway_method" "aws_api_gateway_resource_paths" {
  rest_api_id   = aws_api_gateway_rest_api.example_api_gateway.id
  resource_id   = aws_api_gateway_resource.example_resource_paths.id
  http_method   = "ANY"
  authorization = "NONE"
}

# Lambdaにリクエストを投げられるようにAWS_PROXYを指定
resource "aws_api_gateway_integration" "example_next_api_root" {
  rest_api_id = aws_api_gateway_rest_api.example_api_gateway.id
  resource_id = aws_api_gateway_rest_api.example_api_gateway.root_resource_id
  http_method = aws_api_gateway_method.example_next_api.http_method

  type                    = "AWS_PROXY"
  uri                     = "TODO: Lambda ARN"
  integration_http_method = "POST"
}

resource "aws_api_gateway_integration" "example_next_api_paths" {
  rest_api_id = aws_api_gateway_rest_api.example_api_gateway.id
  resource_id = aws_api_gateway_resource.example_resource_paths.id
  http_method = aws_api_gateway_method.aws_api_gateway_resource_paths.http_method
  type        = "AWS_PROXY"

  uri                     = "TODO: Lambda ARN"
  integration_http_method = "POST"
}

resource "aws_api_gateway_deployment" "example_next_api" {
  rest_api_id = aws_api_gateway_rest_api.example_api_gateway.id

  triggers = {
    redeployment = filebase64("${path.module}/api-gateway.tf")
  }

  lifecycle {
    create_before_destroy = true
  }

  depends_on = [
    aws_api_gateway_integration.example_next_api_root,
    aws_api_gateway_integration.example_next_api_paths
  ]
}
resource "aws_api_gateway_stage" "example_next_api" {
  deployment_id = aws_api_gateway_deployment.example_next_api.id
  rest_api_id   = aws_api_gateway_rest_api.example_api_gateway.id
  stage_name    = "prod"
}
```

作成するAPI GatewayのOriginとリソースIDをCloudFrontに登録します。

```hcl: cloudfront.tf
resource "aws_cloudfront_distribution" "example_distribution" {
  ...
  // api-gateway
  origin {
    origin_id   = aws_api_gateway_rest_api.example_api_gateway.id
    domain_name = "${aws_api_gateway_rest_api.example_api_gateway.id}.execute-api.ap-northeast-1.amazonaws.com"
    origin_path = "/${aws_api_gateway_stage.example_next_api.stage_name}"

    custom_origin_config {
      http_port  = 80
      https_port = 443

      origin_protocol_policy = "https-only"
      origin_ssl_protocols = [
        "TLSv1",
        "TLSv1.1",
        "TLSv1.2"
      ]
    }
  }

  default_cache_behavior {
    ...
    target_origin_id = aws_api_gateway_rest_api.example_api_gateway.id
    ...
  }
  ...
}
```

Lambda コンテナを使用するための Lambda を作成していきます。
また、この例では、IAM Role に対して AssumeRole を使用して権限を付与しています。
IAM Role とは、AWS リソースへのアクセス権限を管理するための AWS Identity and Access Management（IAM）の機能です。
こちらの記事が参考になると思います。
https://dev.classmethod.jp/articles/iam-role-passrole-assumerole/

```hcl: lambda.tf
resource "aws_ecr_repository" "example_lambda_repo" {
  name                 = "example_lambda_repo"
}

resource "aws_lambda_function" "example_lambda_repo_function" {
  function_name = "exampleLambdaRepo"
  package_type  = "Image"
  image_uri     = "${aws_ecr_repository.example_lambda_repo.repository_url}:latest"
  role          = aws_iam_role.example_lambda_iam.arn
  timeout       = 30

  lifecycle {
    ignore_changes = [
      image_uri
    ]
  }

  depends_on = [
    aws_ecr_repository.example_lambda_repo
  ]
}

resource "aws_iam_role" "example_lambda_iam" {
  name = "example_lambda_iam"

  assume_role_policy = data.aws_iam_policy_document.example_lambda_document.json
}

data "aws_iam_policy_document" "example_lambda_document" {
  statement {
    actions = [
      "sts:AssumeRole",
    ]
    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }
    effect = "Allow"
  }
}
```

作成する Lambda の ARN を API Gateway の uri に挿入していきます。
例えばこんな感じで `aws_lambda_function.example_lambda_repo_function.invoke_arn`
また、API GatewayからLambdaを実行できる権限を付与します
```hcl: api-gateway.tf
resource "aws_lambda_permission" "example_apigw_lambda" {
  statement_id  = "AllowExecutionFromAPIGatewaya"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.example_lambda_repo_function.function_name
  principal     = "apigateway.amazonaws.com"

  source_arn = "${aws_api_gateway_rest_api.example_api_gateway.execution_arn}/*/*/*"
}
```

ここで`terraform apply`すると、以下のようなエラー文が出ると思います。

```
│ Error: creating Lambda Function (exampleLambdaRepo): InvalidParameterValueException: Source image xxxx.dkr.ecr.ap-northeast-1.amazonaws.com/example_lambda_repo:latest does not exist. Provide a valid source image.
│ {
│   RespMetadata: {
│     StatusCode: 400,
│     RequestID: "xxxx"
│   },
│   Message_: "Source image xxxxx.dkr.ecr.ap-northeast-1.amazonaws.com/example_lambda_repo:latest does not exist. Provide a valid source image.",
│   Type: "User"
│ }
```

Lambda を作成して Image を引っ張ってこようとした時に、Image がないから怒られていると思われます。そのため、空の Image を事前にビルドして ECR に Push するか、次の Next.js のビルドを Push することで解消されます。また、エラーは出ていますが Lambda と ECR のリポジトリは作成されているので AWS 上から確認できます。

# Next.js のビルドと ECR、Lambda へのプッシュ
では実際に Next.js をビルドして表示するところまでやっていこうと思います。
Next.js のプロジェクトは、`npx create-next-app .` で作成したものを利用します。
また、`Dockerfile`としては以下のものを利用します。
Dockerfile 内ではすでに package のインストールやビルドされたものを Copy し、イメージとして固めています。
これは、ECRとは別に、Lambdaにもビルドの成果物を push したいからです。

```docker: Dockerfile
FROM amazon/aws-lambda-nodejs:16

# Lambda Web Adapterのインストール
COPY --from=public.ecr.aws/awsguru/aws-lambda-adapter:0.5.0 /lambda-adapter /opt/extensions/lambda-adapter
ENV PORT=3000
ENV NODE_ENV=production

COPY next.config.js ./
COPY public ./public
COPY .next/standalone ./
COPY .next/static ./.next/static

ENTRYPOINT ["node"]
CMD ["server.js"]
```

```
docker build -t your_tag_name . --platform linux/amd64
docker tag your_tag_name:latest ecr-uri(xxx.dkr.ecr.ap-northeast-1.amazonaws.com/example_lambda_repo)
docker push ecr-uri(xxx.dkr.ecr.ap-northeast-1.amazonaws.com/example_lambda_repo)
# denied: Your authorization token has expired. Reauthenticate and try again. と出る場合は、以下を実行
# aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin xxxx.dkr.ecr.ap-northeast-1.amazonaws.com
```

ここで再度`terraform apply`すると、以前エラーが出ていたものが通ると思います。

lambda への push

```
aws lambda update-function-code --function-name exampleLambdaRepo --image-uri xxxx.dkr.ecr.ap-northeast-1.amazonaws.com/example_lambda_repo:latest
```

ここまでできると、CloudFrontのdistributionドメイン名にアクセスすると、Next.jsのランディングページが表示されると思います！

# Route53 を使用したカスタムドメインの設定
ここでは、自身で取得したドメインをRoute53で管理し、利用できるようにしてみます。
この辺りの記事を参考にしました！

https://dev.classmethod.jp/articles/create-subdomain-on-route53/
まずはドメインを持っていない人は取得します。
AWSから取得する方法や、その他色々ありますが、今回はGoogle Domainを参考にしてみます。
[Google Domain](https://domains.google/intl/ja_jp/?gad=1&gclid=CjwKCAjw6IiiBhAOEiwALNqnca1XsfTMKdXDnpuWPTsY06xOTsGbn4dGQqSQzHgli06fzOiz61jTXRoCjVcQAvD_BwE&gclsrc=aw.ds)から取得すると、以下のような画面に行くと思われます。

![](/images/deploy-nextjs-with-aws/googleDomain.png)

まず、ホストゾーンを作成するため、`route53.tf`というファイルで管理していきます。
なお、ホストゾーンは新しく作成するたびに`0.5ドル`発生するので注意が必要です
```hcl:route53.tf
resource "aws_route53_zone" "zone" {
  name = "your domain or sub domain"
}
```
この状態で`terraform apply`すると、AWS ConsoleのRoute53というページのホストゾーンに新しく追加されていると思います。
その中のNSレコードの値を4つ、Google Domainに登録していきます。

また、AWSのACMを利用して証明書を発行します。
```hcl:acm.tf
data "aws_acm_certificate" "your-domain-name" {
  // ACM attaching to cloudfront needs to exist in us-east-1
  provider = aws.us-east-1

  domain      = "your-domain-name, ex: xxxx.xxxx.com"
  types       = ["AMAZON_ISSUED"]
  statuses    = ["ISSUED"]
  most_recent = true
}
```
再度、`terraform apply`すると、AWS ConsoleのAWS Certificate Manager (ACM)ページの証明書一覧に、先ほど登録したドメインの証明書が発行されています。
![](/images/deploy-nextjs-with-aws/acm.png)
この状態ではまだ証明書が発行されただけであり、ドメインに紐づいていないため、紐づけていきます。

```hcl:route53.tf
...
resource "aws_route53_record" "anything is fine" {
  name    = "CNAME名(サブドメインがあるならばここに, ex: .xxxxx.)${aws_route53_zone.zone.name}"
  records = ["CNAME値を入れてください"]
  ttl     = 300
  type    = "CNAME"
  zone_id = aws_route53_zone.zone.id
}

resource "aws_route53_record" "maybe your domain name" {
  name    = "(サブドメインがあればここに, ex: xxxx.)${aws_route53_zone.zone.name}"
  zone_id = aws_route53_zone.zone.id
  type    = "A"

  alias {
    name                   = "ここにはCloudFrontのdistributionドメイン名を入れてください"
    # ここのzone_idは固定値
    # https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-route53-aliastarget-1.html
    zone_id                = "Z2FDTNDATAQYW2" 
    evaluate_target_health = false
  }
}
```
再度`terraform apply`後、少し待つと登録したドメインで表示できるかなと思います。

# GitHub Actions と OIDC で自動デプロイ
最後に、自動でビルド・デプロイするところまでを行います。
今回は、GitHub ActionsとOIDCを利用します。
OIDCを利用することにより、長期間有効なアクセスキーを発行せずとも必要なタイミングでの認証を行うことができるため、トークンの漏洩リスクなどが減ります。

Terraformを用いてIAM Roleを作成し、LambdaとECRに対してGitHub Actionsから更新できるように権限を与えます。
```hcl: iam.tf
resource "aws_iam_role" "deploy_role" {
  name               = "deploy-oidc-role"
  assume_role_policy = data.aws_iam_policy_document.deploy_assume_role_policy.json
}
data "aws_iam_policy_document" "deploy_assume_role_policy" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]
    principals {
      type        = "Federated"
      identifiers = ["arn:aws:iam::${data.aws_caller_identity.self.account_id}:oidc-provider/token.actions.githubusercontent.com"] # ID プロバイダの ARN
    }
    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }
  }
}


# Lambdaのupdateのpolicyの設定
resource "aws_iam_policy" "deploy_to_lambda_policy" {
  name        = "deploy_to_lambda_policy"
  path        = "/"
  description = "policy for deploy_to_lambda"

  policy = data.aws_iam_policy_document.deploy_to_lambda_document.json
}
data "aws_iam_policy_document" "deploy_to_lambda_document" {
  statement {
    actions = [
      "lambda:Update*"
    ]
    effect = "Allow"
    resources = [
      aws_lambda_function.example_next_repo_function.arn
    ]
  }
}
# ここで実際にroleにLambdaのdeploy周りの権限の付与を行なっている
resource "aws_iam_role_policy_attachment" "upload_lambda" {
  role       = aws_iam_role.deploy_role.name
  policy_arn = aws_iam_policy.deploy_to_lambda_policy.arn
}


resource "aws_iam_policy" "deploy_to_ecr_policy" {
  name        = "deploy_to_ecr_policy"
  path        = "/"
  description = "policy for deploy_to_ecr_policy"

  policy = data.aws_iam_policy_document.deploy_to_ecr_document.json
}
data "aws_iam_policy_document" "deploy_to_ecr_document" {
  statement {
    actions = [
      "ecr:BatchGetImage",
      "ecr:BatchCheckLayerAvailability",
      "ecr:CompleteLayerUpload",
      "ecr:GetDownloadUrlForLayer",
      "ecr:InitiateLayerUpload",
      "ecr:PutImage",
      "ecr:UploadLayerPart"
    ]
    effect = "Allow"
    resources = [
      aws_ecr_repository.example_next_repo.arn
    ]
  }

  statement {
    actions = [
      "ecr:GetAuthorizationToken"
    ]
    effect    = "Allow"
    resources = ["*"]
  }
}
resource "aws_iam_role_policy_attachment" "upload_ecr" {
  role       = aws_iam_role.deploy_role.name
  policy_arn = aws_iam_policy.deploy_to_ecr_policy.arn
}
```
この後に`terraform apply`後、GitHub Actions側で挿入するroleのarnを取得しておきます。
AWS ConsoleのIAMページから、ロールの中に今回作成した`deploy-oidc-role`が作成されていると思います。その中のARNをGitHubのSecretsに登録しておきます。

また、GitHub Actionsでの認証には、`aws-actions/configure-aws-credentials`を使用します。
詳細は以下の記事が参考になると思います。
https://dev.classmethod.jp/articles/github-actions-aws-actionsconfigure-aws-credentials-v1deprecated/

```yaml: ./github/workflows/release.yaml
name: release

on:
  push:
    branches:
      - 'main'
  workflow_dispatch:

jobs:
  release-with-nextjs-apigateway-lambda:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v3
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1-node16
        with:
          role-to-assume: ${{ secrets.IAM_ROLE }}
          aws-region: ap-northeast-1
      - uses: actions/setup-node@v3
        with:
          node-version: "16"
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
      - name: set IMAGE_URL
        id: image-url
        run: echo "IMAGE_URL=${{ steps.login-ecr.outputs.registry }}/example_next_repo:${{ github.sha }}" >> $GITHUB_ENV

      - name: npm ci
        run: npm ci
      - name: Build
        run: npm run build
      - name: Build, tag, and push docker image to Amazon ECR
        run: |
          docker build -t ${{ env.IMAGE_URL }} --build-arg build_id=${{ github.sha }} .
          docker push ${{ env.IMAGE_URL }}
      - name: deploy to lambda
        run: aws lambda update-function-code --function-name exampleNextRepo --image-uri ${{ env.IMAGE_URL }}
```

# おわりに
いかだでしたか？
実際に、Terraformを用いてAWSのリソースを管理しながらNext.jsのデプロイ並びに自動化まで行いました。
AWS Console上でリソースを作成するのに比べて、Terraformで管理しておけば環境の作成や破壊が容易に可能になり、遊ぶ環境が作れやすくなるのではないでしょうか。
この記事が、Next.jsをAWSにデプロイする際の一助になれば幸いです。