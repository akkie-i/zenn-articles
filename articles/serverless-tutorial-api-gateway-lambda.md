---
title: "【serverless入門】APIGateway × Lambda で作る Rest API"
emoji: "🦈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS","serverless","APIGateway","Lambda","GitHub Actions"]
published: true
---

## はじめに

自己学習で何か簡単なアプリケーションを作成するときに使える API を簡易的に(できれば無料で)サーバに公開できないものかな。
外部公開されている API の無料枠とかでもいいけど、何か味気ないし、登録処理系がちょっとめんどくさいし、返す JSON は自由にしたい。

そんなこと考えて色々サービス探してたのですが、いい感じのが見つからず。
お！と思っても既にサービス終了してたりするのを見るとプロバイダー側の都合にきっと振り回されるんだろうなぁ。などなど

そう思い、探すのやめて、開発及び公開と金銭的なコストなるべく低めで自分で作ろうと考えた末 serverless 使おうという結論に至りました。

コスト低めとは言え serverless も経験浅いからここもいい勉強にもなりそうだったのでなおよしでした。

本当は自分だけじゃなく、後輩とか他のどなたとかフロント勉強したい人に向けてこの API 好きに叩いていいよ。そんな優しい世界にしたかったけど。すまんな Lambda は従量課金だからそれはできねぇ

コード自体は GitHub に置いておくから、クローンして自分の aws 環境にデプロイすれば API 完成までにはしておくからそれで妥協させてください。

API サクッと欲しい人は上記で、serverless もちょっと興味ありって人は続きを是非読んでください。

それではよろしくお願いします。

## この記事でわかること

:::message

- serverlessで APIGateway、Lambda を使用した REST API 作成の流れ
- serverless.yml の細かい設定方法
- GitHub Actions を使用した CI 構築の流れ

:::

## この記事を読む上での前提条件

:::message alert

以下のツールはあらかじめ導入されていること

- [AWSアカウント](https://aws.amazon.com/jp/)
- [npm](https://www.npmjs.com/)
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

:::

## この記事で取り扱わないこと

:::message alert

serverless 入門として主に設定周りを中心に取り扱います。
Lambda の関数などはテンプレート使用しますので、関数に使われている JavaScript のコードの補足などはありません

:::

## 環境情報

- npm: v8.9.0
- serverless
  - Framework Core: 3.24.1 (local)
  - Plugin: 6.2.2
  - SDK: 4.3.2
- aws-cli v2.7.2

# serverless とは

言葉が被ってややこしいですが、まず「サーバーレス」とはサーバーの構築や保守などの管理の必要がなくサーバー上でプログラムを実行できる仕組みです。

「[serverless](https://www.serverless.com/)」はこのサーバーレス開発を便利にしてくれるフレームワークになります。
割と AWS との組み合わせで説明してる記事が多いので AWS 専用と思う方もいるかもしれませんが、Lambda のようないわゆる FaaS サービスを提供している GCP や Azure などでも使えます。

# serverless の使い所

この記事でも扱います API 設定はほぼ 100%使った方がいいです。
興味ある方は serverless を使わずに AWS マネジメントコンソールで API 設定とかしてみると良いです。めんどいです。

他にも SQS や DynamoDB などのリソースの作成、リソースに対するトリガー、トリガーで実行される関数をソースコードで管理もできます。

これらを GUI 操作で色々設定するのはいいのですが、どうしてもブラックボックス化してしまうのでどこになんの設定があるのか初見で理解するのが大変です。

その点 serverless のコードベースで管理しておくとコード内に集約されるのでこの辺も serverless を使うメリットだと思います。

結論管理できるものは serverless にした方が良いです。

以下、serverless での管理対象 AWS Resource の一例です。

- Lambda
- APIGateway
- CloudWatch
- S3
- SQS
- DynamoDB

# serverless のサンプルを使って開発コストを下げる

serverless では公式で豊富なサンプルを用意してくれています。

https://www.serverless.com/examples/

各クラウド、各言語ごとでの実装例が複数ありますので自分のやりたいことに沿った内容を参考にすると開発コストを下げれます。

ちなみに今回の記事では[こちら](https://www.serverless.com/examples/aws-node-http-api-dynamodb)をベースにコードを書きました。
`serverless.yml`の設定をいじってるだけで関数はほぼサンプルのものの使い回しです。

# serverless のパッケージの導入

以下のコマンドで serverless を導入します。

```bash
$ npm install serverless -g
$ serverless --version

# 以下の出力結果が出ればOK
Framework Core: 3.24.1 (local)
Plugin: 6.2.2
SDK: 4.3.2
```

# IAM の作成

今回唯一の AWS マネジメントコンソールでの作業です。

serverless から AWS 環境にコードをデプロイするためには認証をする必要があります。
認証にはアクセスキーとアクセスシークレットキーを認証情報として設定するのですが、このキーを発行するための IAM ユーザを作成する工程になります。

検索バーに `iam`を入力し対象のサービスをクリック

![](https://storage.googleapis.com/zenn-user-upload/7c4d32526ef8-20221107.png)

右のメニューから `ユーザー`を選択し `ユーザーを追加`をクリック

![](https://storage.googleapis.com/zenn-user-upload/eac96e9f19b6-20221107.png)

1. `ユーザー名*`に任意の名前を入力。今回は `serverless-full-access`とします。
2. `AWS 認証情報タイプを選択*`の `アクセスキー-プログラムによるアクセス`にチェック
3. `次のステップ: アクセス権限`ボタンをクリック

![](https://storage.googleapis.com/zenn-user-upload/1fefea210b84-20221107.png)

1. `既存のポリシーを直接アタッチ`を選択
2. `AdministratorAccess`のにチェック
3. `次のステップ: タグ`ボタンをクリック

![](https://storage.googleapis.com/zenn-user-upload/677c00ebd4db-20221107.png)

:::message alert
AdministratorAccess はなんでもできる一番強力な権限でおいそれと誰にでも付与するような権限ではないです。
実務では適切な権限設定でユーザーを作成する必要がありますのでその点は覚えておいてください。
自分の環境ならひとまずいいですが、例えお試しだとしても会社のアカウントなどを触る場合などあったら特に慎重に。
:::

とは言え IAM の権限設定は難しいですし僕も詳しいとは言えません。
以下はこの記事のコードをデプロイするときに権限系でエラーが出てしまったコラムです。
読まなくても進行上問題ないので興味ある方は読んでください。

:::details デプロイ時に発生する権限エラーの例

最初に IAM を作成していて Lambda の権限だし、Lambda のフルアクセス権とりあえず付与をしました。(これでもだいぶ緩い権限設定だと思う)

![](https://storage.googleapis.com/zenn-user-upload/d9bb34260f73-20221107.png)

結果として以下のエラーになりました。

```bash
Error:
CREATE_FAILED: ApiGatewayRestApi (AWS::ApiGateway::RestApi)
User: arn:aws:iam::*****:user/serverless-full-access-staging is not authorized to perform: apigateway:POST on resource: arn:aws:apigateway:ap-northeast-1::/restapis because no identity-based
policy allows the apigateway:POST action (Service: AmazonApiGateway; Status Code: 403; Error Code: AccessDeniedException; Request ID: *****; Proxy: null)
```

APIGateway に対する権限エラーです。
追加で APIGateway のフルアクセスのポリシーをアタッチするも次は S3, cloudFormation と次々を権限エラーが発生しました。

そんなことから、しんどくなりそうだったのでひとまず本記事では `AdministratorAccess`にすることにしました。

権限設定は難しくて、とても大事な知識なのでどこかのタイミングで本腰入れて調べようと思います。
:::

続けます。タグの追加は必要ないので `次のステップ: 確認`ボタンをクリック

![](https://storage.googleapis.com/zenn-user-upload/8c348b2967c3-20221107.png)

内容が問題ないことを確認し、`ユーザーの作成`ボタンをクリック

![](https://storage.googleapis.com/zenn-user-upload/1be5e980595e-20221107.png)

`.csvのダウンロード`ボタンをクリックするとアクセスキー情報が取得できます。ここでしか取得できないのでダウンロードのし忘れに気をつけてください。

![](https://storage.googleapis.com/zenn-user-upload/93832d1f5b16-20221107.png)

ダウンロードしてきた csv の形式は以下です。この中の `Access key ID`と `Secret access key`を使用します。

```csv
User name,Password,Access key ID,Secret access key,Console login link
serverless-full-access,,********,*****************,https://********.signin.aws.amazon.com/console
```

# ローカル環境に AWS への認証情報を登録する

先ほどダウンロードした情報をローカル環境に登録します。
「serverless のパッケージの導入」で`serverless`コマンドが使えますのでそちらで登録します(`sls`と省略も可能)

適宜、説明を読んで該当箇所を変更して以下のコマンドを実行ください。

```bash
$ sls config credentials \
--provider aws \
--key ***** \
--secret ***** \
--profile aws1-serverless-full-access-staging

# 以下が出力されればOK
✔ Profile "aws1-serverless-full-access-staging" has been configured
```

コマンドの内容の説明です。

- `--provider aws`
  - serverless をデプロイするクラウド先の指定
- `--key *****`
  - AWS のアクセスキー、`*****`は csv で取得した `Access key ID`の値を設定
- `--secret *****`
  - AWS のシークレットキー、`*****`は csv で取得した `Secret access key`の値を設定
- `--profile aws1-serverless-full-access-staging`
  - 認証情報のセットに名前をつける。
  - 名前は任意でよく今回は `aws1-serverless-full-access-staging`とする
  - serverless のデプロイ先が複数ある時などはこの名前でデプロイ先をハンドルする

コマンドが成功すると `~/.aws/credentials`で登録した内容を確認できます。

以下のコマンドで確認できます。

```bash
$ cat ~/.aws/credentials

# 出力結果
[--profileで設定した値]
aws_access_key_id=--keyで設定した値
aws_secret_access_key=--secretで設定した値
```

# プログラムの作成

まずは以下の構造でフォルダ、ファイルを作成してください。

```json
.
├── README.md
├── config
│   └── serverless
│       ├── apikeys.yml
│       ├── environment.yml
│       ├── functions.yml
│       └── resources.yml
├── .gitignore
├── package.json
├── serverless.yml
└── users
    ├── create.js
    ├── delete.js
    ├── get.js
    ├── list.js
    └── update.js
```

[こちら](https://www.serverless.com/examples/aws-node-http-api-dynamodb)を参考にしてますが、`todos`の関数の JavaScript ファイルは名前とか変えてます。
とは言えコードは的にはほとんど変わりないので説明とかは割愛します。コードの中身は [GitHub](https://github.com/akkie-i/serverless-sample) を参照ください。

それでは各ファイル確認してみます。

## package.json の内容

[こちら](https://www.serverless.com/examples/aws-node-http-api-dynamodb)とほぼ同様です。

```json
{
  "name": "serverless-http-api-dynamodb",
  "version": "1.0.0",
  "description": "Serverless CRUD service exposing a REST HTTP interface",
  "author": "",
  "license": "ISC",
  "dependencies": {
    "uuid": "^9.0.0"
  }
}
```

## .gitignore の内容

以下の修正を加えてください。

```
node_modules
.serverless
.vscode
config/serverless/apikeys.yml
config/serverless/environment.yml
```

`config/serverless/`の一部ファイルを追加してますが、理由については後述の「GitHub Actions で CI を構築」で説明します。

## serverless.yml の内容

今回はここを重点的に取り扱います。
serverless にデプロイする設定をこのファイルに定義していきます。

`serverless.yml`全体を確認します。

```yml
# https://www.serverless.com/framework/docs/providers/aws/guide/serverless.yml
# https://github.com/serverless/examples/blob/v3/aws-node-http-api-dynamodb/todos/create.js

service: aws1-serverless-full-access

provider:
  name: aws
  runtime: nodejs16.x
  stage: ${opt:stage, "staging"}
  profile: ${self:custom.profiles.${sls:stage}}
  environment: ${file(./config/serverless/environment.yml)}
  apiGateway:
    apiKeys: ${file(./config/serverless/apikeys.yml)}
    usagePlan:
      quota:
        limit: 100
        offset: 0
        period: MONTH
      throttle:
        rateLimit: 2
        burstLimit: 3
  httpApi:
    cors: true
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:Query
            - dynamodb:Scan
            - dynamodb:GetItem
            - dynamodb:PutItem
            - dynamodb:UpdateItem
            - dynamodb:DeleteItem
          Resource: "arn:aws:dynamodb:${aws:region}:*:table/${self:provider.environment.DYNAMO_USERS_TABLE}"
custom:
  profiles:
    staging: aws1-serverless-full-access-staging
functions: ${file(./config/serverless/functions.yml)}
resources: ${file(./config/serverless/resources.yml)}
```

設定の細かいところは[こちら](https://www.serverless.com/framework/docs/providers/aws/guide/serverless.yml)で確認してもらい今回は主要な部分をピックアップして説明します。

`${}`でかこまれた箇所は変数や関数を使用できます。

それぞれ確認します。

- `opt:stage`
  - デプロイコマンドの `stage`オプションで指定した値を取得します。
  - 第 2 引数はデフォルト設定値になります。
  - `sls:stage`も同義です。
- `self`
  - 自分自身、つまり `serverless.yml` を指します。
  - yml はインテントで階層構造になってますが、階層に沿って `.`で繋ぎアクセスできます
  - `${self:custom.profiles.${sls:stage}}`を例に分解してみます
    - yml 内の `custom`→ `profiles`にアクセスします
    - `sls:stage`はコマンドオプションで指定した値を取得。(`staging`とする)
    - 上記から `custom.profiles.staging`になるので `aws1-serverless-full-access-staging`が取得される
      - 参考ドキュメントは[こちら](https://www.serverless.com/framework/docs/providers/aws/guide/credentials/)
    - この値を `profile`に設定することでデプロイ時に参照するプロフィール情報の指定する
      - プロフィールは`~/.aws/credentials`の値を指す
- `file`
  - 引数にファイルパスを指定することで読み込みが可能
  - 全てを `serverless.yml`に集約させると長くなるので部分的に別ファイルに切り出すと読みやすくなる

## config/serverless のファイルを確認する

`serverless.yml`で別ファイルとして切り出した yml ファイルです。

以下は API キーの情報です。こちらは秘匿情報として管理するために切り分けました。
詳しくは以降の「S3 に秘匿情報をおいておく」を参照ください。

`value:`の値はハッシュ値などを生成して任意の値を設定ください。

```yml:config/serverless/apikeys.yml
- name: ${opt:stage, self:provider.stage}-free-key
  value: *****
```

以下は環境変数系の情報です。今回はあまりないですが結構量も増えるし、秘匿情報も書いたりするので同じくファイルを切り分けてます。

```yml:config/serverless/environment.yml
DYNAMO_USERS_TABLE: ${self:service}-${opt:stage, self:provider.stage}-users
```

以下は関数の定義です。
イベントに対して実行する関数をここに定義します。
これもサービスが大きくなるほど量が多くなるので切り分けておいた方が良いと思います。

今回は `http`イベントなので API のエンドポイントのイベントと紐づく関数を指定してます。
`private: true`を付与することで `apikeys.yml`で設定した値を認証ヘッダーとして付与する設定になります。

```yml:config/serverless/environment.yml
create:
  handler: users/create.create
  events:
    - http:
        path: /users
        method: post
        private: true

list:
  handler: users/list.list
  events:
    - http:
        path: /users
        method: get
        private: true

get:
  handler: users/get.get
  events:
    - http:
        path: /users/{id}
        method: get
        private: true

update:
  handler: users/update.update
  events:
    - http:
        path: /users/{id}
        method: put
        private: true

delete:
  handler: users/delete.delete
  events:
    - http:
        path: /users/{id}
        method: delete
        private: true
```

以下は AWS に生成するリソースの定義です。
ここに定義したリソースがデプロイすると生成されるので AWS マネジメントコンソールで作成する手間などが省けます。

今回は DynamoDB を生成する記述のみですね。

```yml:config/serverless/resources.yml
Resources:
  TodosDynamoDbTable:
    Type: "AWS::DynamoDB::Table"
    DeletionPolicy: Retain
    Properties:
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH
      BillingMode: PAY_PER_REQUEST
      TableName: ${self:provider.environment.DYNAMO_USERS_TABLE}
```

# デプロイしてみる

実行する関数と `serverless.yml`の記述が終わればもうデプロイ可能です。
最終コードは[こちら](https://github.com/akkie-i/serverless-sample)を参考に一回デプロイしてみます。

以下のコマンドで依存関係をインストール

```bash
$ npm install
```

以下の `sls deploy`コマンドでデプロイします。

```bash
$ sls deploy \
--verbose \
--stage staging \
--aws-profile aws1-serverless-full-access-staging \
--region ap-northeast-1
```

`serverless.yml`に記載しているオプション情報は省略していいのですが、説明のため明示的に指定してます。
以下オプションの解説です。

- `--verbose`
  - デプロイ時の詳細のログを出力
- `--stage`
  - デプロイ時の識別子(タグ)
- `--aws-profile`
  - `~/.aws/credentials`のプロフィールの指定
- `--region`
  - デプロイするリージョンの指定

デプロイの結果として `✔ Service deployed to stack aws1-serverless-full-access-staging (96s)`と表示されれば完了です。

デプロイの結果として `.serverless`フォルダにデプロイした資材情報が格納されます。

また、`sls info`コマンドでデプロイした結果の情報が確認できます。

```bash
$ sls info

# 出力結果
service: aws1-serverless-full-access
stage: staging
region: ap-northeast-1
stack: aws1-serverless-full-access-staging
api keys:
  staging-free-key: *****
endpoints:
  POST - https://*****.execute-api.ap-northeast-1.amazonaws.com/staging/users
  GET - https://*****.execute-api.ap-northeast-1.amazonaws.com/staging/users
  GET - https://*****.execute-api.ap-northeast-1.amazonaws.com/staging/users/{id}
  PUT - https://*****.execute-api.ap-northeast-1.amazonaws.com/staging/users/{id}
  DELETE - https://*****.execute-api.ap-northeast-1.amazonaws.com/staging/users/{id}
functions:
  create: aws1-serverless-full-access-staging-create
  list: aws1-serverless-full-access-staging-list
  get: aws1-serverless-full-access-staging-get
  update: aws1-serverless-full-access-staging-update
  delete: aws1-serverless-full-access-staging-delete
```

## serverless のデプロイは cloudFormation で管理される

serverless のデプロイは `serverless.yml`の内容に沿って cloudFormation で管理されます。

試しに AWS マネジメントコンソールで `cloudFormation`を検索してみてください。

![](https://storage.googleapis.com/zenn-user-upload/bb995e6c94f3-20221107.png)

なにやらデプロイしたサービスのスタックらしきものが確認できます。
試しに中を見て `リソース`タブを選択してください。

![](https://storage.googleapis.com/zenn-user-upload/398c75d15bd9-20221107.png)

色々なリソースが生成されていることが確認できます。

`serverless.yml`で明示的に指定したリソースをいくつか確認していきます。

まずは `APIGateway`の以下のあたりの箇所です。

```yml
apiGateway:
  apiKeys: ${file(./config/serverless/apikeys.yml)}
  usagePlan:
    # 省略
```

`functions:`に設定したエンドポイントが生成されています。

![](https://storage.googleapis.com/zenn-user-upload/193c58985835-20221107.png)

対応する関数も存在します。

![](https://storage.googleapis.com/zenn-user-upload/f47abb7d7f03-20221107.png)

以下で DynamoDB のロールも作成しました。

```yml
iam:
  role:
    statements:
      - Effect: Allow
        Action:
          - dynamodb:Query
          - dynamodb:Scan
          - dynamodb:GetItem
          - dynamodb:PutItem
          - dynamodb:UpdateItem
          - dynamodb:DeleteItem
        Resource: "arn:aws:dynamodb:${aws:region}:*:table/${self:provider.environment.DYNAMO_USERS_TABLE}"
```

IAM ロール周りを探してみると発見できました。

![](https://storage.googleapis.com/zenn-user-upload/009fd8ad925e-20221107.png)

DynamoDB も作られてます。

![](https://storage.googleapis.com/zenn-user-upload/96420c508bf2-20221107.png)

その他 yml に書いてないけど自動で生成されるリソースも結構あります。

代表的なのは CloudWatch です。

![](https://storage.googleapis.com/zenn-user-upload/e1f2beaefb36-20221107.png)

こんな感じで関数ごとにロググループが生成されます。
関数にコンソールなどを仕込むとここに出力されますのでデバッグなどのときに大活躍します。

ただ、cloudFormation も良いことばかりではなく、S3 に実行ログのようなファイルを生成するなどなくても良いかな。という無駄なリソースを作ってしまうなどの側面もあります。

# GitHub Actions で CI を構築

デプロイのコマンドを GitHub Actions を使用して CI で構築する方法を確認してみます。

CI を構築することで GitHub にマージされたら自動でデプロイコマンドが実行されるようになります。

これで便利で楽にもなりますし、仮に複数人で開発していた場合みんながみんなローカルからデプロイコマンドを実行してデグレが発生するリクスなども無くなりますのでできる限り導入することをおすすめします。

## アクセスキーを GitHub の環境変数に設定する

ローカルから直接デプロイするまでの内容を振り返ると serverless のコードをデプロイするには `~/.aws/credentials`にアクセスキー情報が登録されている必要があるのが確認できます。

GitHub ではそういった環境情報を保持していないので登録する必要がありますのでやっていきましょう。※あらかじめ空のリポジトリを作成しておいてください。

対象のリポジトリに移動して `settings`タブを選択 → `Actions`を選択 → `New repository secret`ボタンをクリックします。

![](https://storage.googleapis.com/zenn-user-upload/c04ef83f06df-20221107.png)

`Name *`に任意の変数名を入力します。今回は `STG_AWS_ACCESS_KEY_ID`とします。
`Secret *`に `~/.aws/credentials`に記載された対象プロフィールの `aws_access_key_id`の値を入力し `Add secret`ボタンをクリック

![](https://storage.googleapis.com/zenn-user-upload/0c422e102d44-20221107.png)

同じ要領で `~/.aws/credentials`に記載された対象プロフィールの `aws_secret_access_key`の値をもとに `STG_AWS_SECRET_ACCESS_KEY`という変数を作成ください。

作成した結果として 2 つの変数が追加されたことを確認できます。

![](https://storage.googleapis.com/zenn-user-upload/2a754b92104e-20221107.png)

## S3 に秘匿情報をおいておく

ローカル環境からデプロイする時はどんなファイルを置いておいても自分にしか見えないので良いのですが、GitHub に置くとなると見られたくない情報は隠しておきたいですよね。

リポジトリを private にするもいいのですが、今回みたいに公開用に public で作ったりするならなおのことです。

今回は API キーの情報を設定しているのでここの値は知られたくないです。何かしら API を使ったことあるならシークレットキーはコードにベタ書きしないで環境情報として管理するのと同じ要領でイメージできるかと思います。

このように `serverless.yml`に記載はするけど見せたくない情報はどうやって管理しようかという時の一アプローチとして見せたくない情報は別の yml ファイルとして切り出しておいて S3 に置いておく。

GitHub Action 内で `aws s3`コマンドを実行して、秘匿情報をダウンロードして取得するという方法にしてみました。(これがベストかはわかりませんが)

`.gitignore`に `config/serverless/apikeys.yml`と `config/serverless/environment.yml`を設定していたのはこのためです。(今回 `environment.yml`は見えても問題ない情報ですが、比較的秘匿情報が集まりやすそうな場所だと思ったのでついでに入れておきました)

では、S3 にバケットを作ってファイルを配置してみましょう。

`バケットを作成`ボタンをクリック。

![](https://storage.googleapis.com/zenn-user-upload/5fa67d87d92b-20221107.png)

バケット名に任意の名前を入力。今回は `serverless-sample-staging`とします。
バケット名を設定したら `バケットを作成`ボタンをクリック。

![](https://storage.googleapis.com/zenn-user-upload/e9120e48359b-20221107.png)

バケットが作成されたのでリンクをクリックし中に入る。

![](https://storage.googleapis.com/zenn-user-upload/ec4129c1fa72-20221107.png)

ローカルのフォルダ構造と同じで管理したいのでまずフォルダを作成します。
`フォルダの作成`ボタンをクリック。

![](https://storage.googleapis.com/zenn-user-upload/a539323b0d3b-20221107.png)

フォルダ名に任意の名前を入力。今回は `config`とします。
入力したら `フォルダの作成`ボタンをクリック。

![](https://storage.googleapis.com/zenn-user-upload/9b0e74262d9f-20221107.png)

作成されたフォルダのリンクをクリックして中に入る

![](https://storage.googleapis.com/zenn-user-upload/5a8c712cb45d-20221107.png)

`serverless-sample-staging/config/`内でさらに `フォルダの作成`ボタンをクリック。

![](https://storage.googleapis.com/zenn-user-upload/198478ffce98-20221107.png)

フォルダ名に任意の名前を入力。今回は `serverless`とします
入力したら `フォルダの作成`ボタンをクリック。

![](https://storage.googleapis.com/zenn-user-upload/94cdf08523c3-20221107.png)

作成されたフォルダのリンクをクリックして中に入る

![](https://storage.googleapis.com/zenn-user-upload/5f43d3440f6c-20221107.png)

`serverless-sample-staging/config/serverless`内でさらに `アップロード`ボタンをクリック。

![](https://storage.googleapis.com/zenn-user-upload/7176ff25aee8-20221107.png)

1. `ファイルを追加`ボタンをクリック
2. 1.で `apikeys.yml`と `environment.yml`を選択
3. `アップロード`ボタンをクリック

![](https://storage.googleapis.com/zenn-user-upload/3f8f90c62ce9-20221107.png)

## workflow を作成

下準備が完了したので workflow で action の内容を定義していきます。

プロジェクトフォルダのルートディレクトリに `.github/workflows/deploy.yml`ファイルを作成してください。

作成したファイルに以下の編集をします。

```yml
name: デプロイ ステージング

on:
  push:
    branches:
      - main
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v1
      - uses: actions/setup-node@v2
        with:
          node-version: "18"
      - name: set aws configure
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.STG_AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.STG_AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-northeast-1
      - name: get serverless env from s3
        run: |
          aws s3 cp s3://serverless-sample-staging/config/serverless/apikeys.yml ./config/serverless
          aws s3 cp s3://serverless-sample-staging/config/serverless/environment.yml ./config/serverless
      - name: install packages
        run: |
          npm i
      - name: deploy serverless
        uses: serverless/github-action@v3.1
        with:
          args: deploy --stage staging
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.STG_AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.STG_AWS_SECRET_ACCESS_KEY }}
```

内容の説明は以下です。細かいところは[公式](https://docs.github.com/ja/actions)にお任せするとして主要な部分を扱っていきます。

- `on`
  - 何のアクションが発生したらというトリガーを設定します
  - 今回は `main`ブランチにコードがプッシュされたらになります
- `jobs`
  - トリガーが検知された時の実際のジョブを定義していきます
- `name: set aws configure`
  - ここでローカル環境で言うとこの `~/.aws/credentials`にあたる設定を行います
  - [aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials)に沿って値を設定
    - `aws-access-key-id: ${{ secrets.STG_AWS_ACCESS_KEY_ID }}`
      - `~/.aws/credentials`の `AWS_ACCESS_KEY_ID`にあたる値を設定
      - `${{}}`で `secrets.STG_AWS_ACCESS_KEY_ID`を設定。
      - 上記の値は「アクセスキーを GitHub の環境変数に設定する」で設定済み
      - GitHub の変数を使用するにはこのように `secrets.変数名`でアクセスできます
    - 同じ要領で `aws-secret-access-key`を設定
    - `aws-region`はマストで設定する必要があるっぽいです
- `name: get serverless env from s3`
  - ここで「S3 に秘匿情報をおいておく」で置いたファイルを取得
  - [aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials)で指定したアクセスキー情報に対して `aws`コマンドが実行可能になる
  - `aws s3 cp`を実行してファイルを取得
    - cp は `from to`の順番で from のファイルを to へ保存となります。
      - from: `s3://serverless-sample-staging/config/serverless/apikeys.yml`で S3 のファイルを
      - to: `./config/serverless`に保存
- `name: deploy serverless`
  - デプロイを実行します。[serverless/github-action](https://github.com/serverless/github-action)の手順に準拠
  - `env:`のアクセスキー情報に対して `with: args:`のコマンドを実行
  - `sls deploy --stage staging`を実行している

## serverless.yml の修正

「デプロイしてみる」で実行した以下のコマンドを振り返ってみます。

```yml
$ sls deploy \
--verbose \
--stage staging \
--aws-profile aws1-serverless-full-access-staging \
--region ap-northeast-1
```

こちらと `deploy.yml`を見比べてみるとこちらには `--aws-profile`にあたる情報を定義していません。[aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials)にも `profile`を指定するようなやり方はなさそうです。

なので `serverless.yml`から `profile:`を指定してた箇所を削除しましょう。
実際に現状のまま CI を実行すると `Error: AWS profile "aws1-serverless-full-access-staging" doesn't seem to be configured`となるのが確認できます。

修正内容は以下です。

```diff yml
# https://www.serverless.com/framework/docs/providers/aws/guide/serverless.yml
# https://github.com/serverless/examples/blob/v3/aws-node-http-api-dynamodb/todos/create.js

service: aws1-serverless-full-access

provider:
  name: aws
  region: ap-northeast-1
  runtime: nodejs16.x
  stage: ${opt:stage, "staging"}
- profile: ${self:custom.profiles.${sls:stage}}
  environment: ${file(./config/serverless/environment.yml)}
  apiGateway:
    apiKeys: ${file(./config/serverless/apikeys.yml)}
    usagePlan:
      quota:
        limit: 100
        offset: 0
        period: MONTH
      throttle:
        rateLimit: 2
        burstLimit: 3
  httpApi:
    cors: true
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:Query
            - dynamodb:Scan
            - dynamodb:GetItem
            - dynamodb:PutItem
            - dynamodb:UpdateItem
            - dynamodb:DeleteItem
          Resource: "arn:aws:dynamodb:${aws:region}:*:table/${self:provider.environment.DYNAMO_USERS_TABLE}"
-custom:
- profiles:
-   staging: aws1-serverless-full-access-staging
functions: ${file(./config/serverless/functions.yml)}
resources: ${file(./config/serverless/resources.yml)}
```

:::message
credentials 情報生成時に `--aws-profile`でプロフィールを指定しなかった場合 `default`と言うプロフィール名になります。
sls コマンド実行時に明示的にプロフィールを指定する、または `serverless.yml`内に `profile:`の情報がない場合は `default`に対して実行するという仕様があります。
以上から[aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials)では `default`で設定しているのと同義になりますので `profile:`を無くせば設定したアクセスキー情報に対してデプロイを実行できるようになります。
:::

## CI でデプロイしてみる

あとは GitHub の `main`ブランチに対してプッシュをすれば `deploy.yml`の内容で CI が走ります。

```bash
$ git push -u origin main
```

プッシュできたら `Actions`タブで詳細を確認できます。

![](https://storage.googleapis.com/zenn-user-upload/56879901bdc0-20221107.png)

# 動作確認

最後にデプロイした API がうまいこと動くのか確認してみます。

まだ何もデータないのでまずは `create`してみます。
以下のコマンドを実行してください。

```
$ curl -X POST https://*****.execute-api.ap-northeast-1.amazonaws.com/staging/users \
-H "x-api-key: *****" \
--data '{ "name": "taro", "age": 20, "email": "text@example.com", "gender": "男"}'
```

`https://*****`のと `-H "x-api-key: *****"`の値は以下のコマンドで確認できます。

```bash
sls info --aws-profile aws1-serverless-full-access-staging
```

実行結果として DynamoDB にデータが格納されたことを確認できます。

![](https://storage.googleapis.com/zenn-user-upload/4c8c32cc5602-20221107.png)

ちゃんと API キーの認証もかかってるか確認するために `-H "x-api-key: *****"`をなしで実行してください。

`{"message":"Forbidden"}`とレスポンスが変えることが確認できます。

次に `list`で取得してみます。
以下のコマンドを実行してください。

```bash
$ curl https://*****.execute-api.ap-northeast-1.amazonaws.com/staging/users \
-H "x-api-key: *****"
```

以下のレスポンスが取得できます。

```json
[
  {
    "updatedAt": 1667793979478,
    "createdAt": 1667793979478,
    "id": "888edf60-5e51-11ed-994c-e7cef1591f54",
    "email": "text@example.com",
    "name": "taro",
    "age": 20,
    "gender": "男"
  }
]
```

次に `get`で取得してみます。
以下のコマンドを実行してください。 `{id}`は `list`で取得した `id`に差し替えてください。

```bash
$ curl https://*****.execute-api.ap-northeast-1.amazonaws.com/staging/users/{id} \
-H "x-api-key: *****"
```

以下のレスポンスが取得できます。

```json
{
  "updatedAt": 1667793979478,
  "createdAt": 1667793979478,
  "id": "888edf60-5e51-11ed-994c-e7cef1591f54",
  "email": "text@example.com",
  "name": "taro",
  "age": 20,
  "gender": "男"
}
```

次に `update`で年齢を更新してみましょう。
以下のコマンドを実行してください。`{id}`は `list`で取得した `id`に差し替えてください。

```bash
$ curl -X PUT https://*****.execute-api.ap-northeast-1.amazonaws.com/staging/users/{id} \
-H "x-api-key: *****" \
--data '{ "name": "taro", "age": 999, "email": "text@example.com", "gender": "男"}'
```

DynamoDB の値が更新されたことが確認できます。

最後に `delete`を実行してみましょう
以下のコマンドを実行してください。`{id}`は `list`で取得した `id`に差し替えてください。

```bash
$ curl -X DELETE https://*****.execute-api.ap-northeast-1.amazonaws.com/staging/users/{id} \
-H "x-api-key: *****" \
```

DynamoDB からレコードが削除されたことを確認できます。

以上でデプロイした API が問題なく動くことが確認できました。お疲れ様です。

# さいごに

さいごまで読んでいただきありがとうございます。
serverless 入門として抑えとくべき箇所はカバーできるようにと記事を作成してみましたがいかがでしたでしょうか。

是非、自分で作るのをトライして API のバリエーションを増やす、ちゃんとバリデーション設計する。TypeScript にしてみる。

などなど色々カスタマイズしながら遊んで学習の材料にしてもらえたら幸いです。

間違いの指摘やリクエストなどありましたら加筆していきたので是非、ご意見をいただけたらと思います。

それではまた次の記事でお会いしましょう。
