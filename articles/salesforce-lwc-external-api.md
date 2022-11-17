---
title: "【Salesforce】LWCで外部APIを叩く方法"
emoji: "⚡️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Salesforce", "LWC"]
published: true
---

# はじめに

受託開発で広く浅く技術を触ってるからか、最近では自分は何屋さん？てなる場面もありますが、実はマルチクラウドを使ったクラウド間のインテグレーション業務が本分だったのです。

今回は生業らしい題材で Salesforce が提供する Lightning Web Component(以降 LWC)についてです。

LWC を使って Salesforce 上の画面にコンポーネントをカスタマイズできますがコンポーネントを構築するデータ。
これは Salesforce の中だけで完結すると言うことはそんなにないと思います。

基幹システムからデータを取得したいなど、外部サービスからデータを取得してきてコンポーネントを構築したいと言う要件はよくあります。

外部からデータを取得するといえば真っ先に出てくるのは API でしょう。
そんなわけで LWC から API を叩く方法を確認していきます。

それではよろしくお願いします。

## この記事でわかること

:::message

- LWC でコンポーネントを作成する方法
- LWC から Apex コードをコールする方法
- LWC から API を叩く方法

:::

## この記事を読む上での前提条件

:::message alert

- LWC の基礎中の基礎的なところは習得済みの前提です。
  - LWC があまりわからない方はまずは[こちら](https://trailhead.salesforce.com/ja/content/learn/projects/quick-start-lightning-web-components?trail_id=build-lightning-web-components)を実践ください
- 上記リンクを実践いただいたら `Salesforce CLI`がインストールされてるはずなのでこちらは導入済みの前提です
- エディターは [VSCode](https://azure.microsoft.com/ja-jp/products/visual-studio-code/)の前提です

:::

## この記事で取り扱わないこと

:::message alert

- What is LWC 的な概念やアーキテクチャの説明とかはこの記事では扱いません

:::

## 環境情報

- sfdx-cli v7.152.0

# API の準備

ご自身でお好きな API を準備ください。
サーバー持っててコード置いてあるならそれでも良いですし、[OpenWeather](https://openweathermap.org/)とか外部 API サービスを利用するなりで OK です。

割と低コストで API を作れるように以下の記事も作成しましたので併せてご確認ください。

https://zenn.dev/akkie1030/articles/serverless-tutorial-api-gateway-lambda

# 厳密には LWC から API は叩けない

タイトル通りですが、デフォルト設定のままでは LWC から API を叩くことはできないです。
試しに LWC コンポーネントの JavaScript ファイルで API をフェッチしてみたりすると接続エラーになります。

では、どうやってやるのか？
Salesforce には Java ライクな [Apex](https://developer.salesforce.com/docs/atlas.ja-jp.apexcode.meta/apexcode/apex_intro_what_is_apex.htm) という独自のプログラミング言語が存在します。

こちらからは API を実行でき、LWC ではこの Apex のコードを JavaScript の `import`構文を使って取り込み、関数をコールできます。

上記から LWC で Apex を import→Apex から API を叩くという工程が必要ということになります。

# コンポーネントの作成

まずはハードコーディングでアウトプットの画面イメージのコンポーネントを作成します。

API は[こちら](https://zenn.dev/akkie1030/articles/serverless-tutorial-api-gateway-lambda)を使う前提で、完成系のイメージは以下です。

![](https://storage.googleapis.com/zenn-user-upload/349bfd4602ae-20221109.png)

次に、コンポーネントを作成します。

vscode 上で `command⌘` + `shift⇧` + `p`でコマンドパレットを開きます。
パレット内に `sfdx`と入力し、候補で検出された `SFDX: Create Lightning Web Component`を選択します。

![](https://storage.googleapis.com/zenn-user-upload/78e2e9ac95b9-20221108.png)

次に作成するコンポーネントの名前を入力します。今回は `apiSample`とします。

![](https://storage.googleapis.com/zenn-user-upload/ece0024ab8f4-20221108.png)

コンポーネントの作成先を指定します。デフォルトで問題ないので `enter⏎`を押します。

![](https://storage.googleapis.com/zenn-user-upload/69dee2f59b20-20221108.png)

すると `force-app/main/default/lwc`配下に `apiSample`という html,js,xml のファイルで構成されたフォルダが作成されます。

```bash
├── README.md
├── config
│   └── project-scratch-def.json
├── force-app
│   └── main
│       └── default
│           ├── applications
│           ├── aura
│           ├── classes
│           ├── contentassets
│           ├── flexipages
│           ├── layouts
│           ├── lwc
│           │   ├── apiSample
│           │   │   ├── __tests__
│           │   │   │   └── apiSample.test.js
│           │   │   ├── apiSample.html
│           │   │   ├── apiSample.js
│           │   │   └── apiSample.js-meta.xml
│           │   └── jsconfig.json
│           ├── objects
│           ├── permissionsets
│           ├── staticresources
│           ├── tabs
│           └── triggers
├── jest.config.js
├── package.json
├── scripts
│   ├── apex
│   │   └── hello.apex
│   └── soql
│       └── account.soql
└── sfdx-project.json
```

作成された各ファイルに以下の編集を加えます。

```html: force-app/main/default/lwc/apiSample/apiSample.html
<template>
  <lightning-card title="ユーザ一覧" icon-name="standard:service_report">
    <div class="slds-p-around_medium">
      <!--https://developer.salesforce.com/docs/component-library/bundle/lightning-datatable/example-->
      <lightning-datatable
        key-field="id"
        data={tableBody}
        columns={tableHead}
        hide-checkbox-column
      >
      </lightning-datatable>
    </div>
  </lightning-card>
</template>
```

```js: force-app/main/default/lwc/apiSample/apiSample.js
import { LightningElement } from 'lwc';

export default class ApiSample extends LightningElement {

  tableHead = [
    { label: '名前', fieldName: 'name', type: 'text',hideDefaultActions: true },
    { label: '年齢', fieldName: 'age', type: 'number', hideDefaultActions: true },
    { label: 'メールアドレス', fieldName: 'email', type: 'phone', hideDefaultActions: true },
    { label: '性別', fieldName: 'gender', type: 'text', hideDefaultActions: true },
  ];

  tableBody = [
    {
      id: 'abcd',
      name: 'taro',
      age: '20',
      email: 'test@example.com',
      gender: '男',
    }
  ];
}
```

```xml: force-app/main/default/lwc/apiSample/apiSample.js-meta.xml
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata" fqn="apiSample">
  <apiVersion>55.0</apiVersion>
  <isExposed>true</isExposed>
  <targets>
    <target>lightning__AppPage</target>
    <target>lightning__RecordPage</target>
    <target>lightning__HomePage</target>
  </targets>
</LightningComponentBundle>
```

JavaScript に表示するテーブルデータをハードコーディングして HTML で表示させるだけのシンプルなコードです。

こちらを API で取得したデータで表示するように修正していきます。

# Apex で API リクエストの雛形を生成

API を送信するためのベースの処理を記載していきしょう。

まず、`force-app/main/default/classes`のフォルダに `controllers`と `services`というフォルダを作成してください。

ファイルを作成していきます。

vscode 上で `command⌘` + `shift⇧` + `p`でコマンドパレットを開きます。
パレット内に `sfdx`と入力し、候補で検出された `SFDX: Create Apex Class`を選択します。

![](https://storage.googleapis.com/zenn-user-upload/4e4e5aa183c4-20221108.png)

次に作成する Apex Class の名前を入力します。今回は `ApiSampleController`とします。

![](https://storage.googleapis.com/zenn-user-upload/58b17ab787ec-20221108.png)

Apex Class の作成先を指定します。`ディレクトリを選択`を選択します。

![](https://storage.googleapis.com/zenn-user-upload/cf79e4f3242a-20221108.png)

検索できますので `controllers`と入力して、`force-app/main/default/classes/controllers`を選択します。

`force-app/main/default/classes/controllers`配下に `ApiSampleController.cls`と `ApiSampleController.cls-meta.xml`が生成されます。

同じ要領で `force-app/main/default/classes/services`配下に `AwsApiGatewayService`と `HttpRequestService`を作成してください。

実装方法はそれぞれですが、僕は LWC と Apex を繋ぐための Controller クラスを作成し、ロジックは Service クラスで切り分けるようにしてます。

Apex はファイル作成例のようにフォルダ階層を分けることも可能です。
注意としてはフォルダを分けても Salesforce 上のリソースとしては最終的に一つの階層にまとめられます。

ですので、後述します JavaScript 側で Apex を import するときにパスを書きますが、そこでは分けたフォルダ名を記載する必要はございません。

## Controller の実装

CRUD の 5 つのメソッドを用意します。

```java: force-app/main/default/classes/controllers/ApiSampleController.cls
public with sharing class ApiSampleController {

    @AuraEnabled(cacheable=true)
    public static Object createRecord() {
        return new Object();
    }

    @AuraEnabled(cacheable=true)
    public static Object getRecords() {
        return new Object();
    }

    @AuraEnabled(cacheable=true)
    public static Object getRecordById() {
        return new Object();
    }

    @AuraEnabled(cacheable=true)
    public static Object updateRecord() {
        return new Object();
    }

    @AuraEnabled(cacheable=true)
    public static Object deleteRecord() {
        return new Object();
    }
}
```

LWC で Apex を利用するときは必ず `@AuraEnabled()`のデコレータが必要です。
細かいとこは今回割愛しますので一旦おまじないのくくりとして覚えてください。

返り値は `Object`で返しますので、一旦インスタンスを返すだけの設定です。

次に対象の API の [README](https://github.com/akkie-i/serverless-sample) を確認します。

共通で必要な情報はエンドポイントとアクセスシークレットキーですね。ここは他の API を使うとしても基本変わらないはずです。

あとは `Create`と `Update`にはリクエストボディの情報が必要そうですね。

上記を踏まえて各種メソッドに引数を設定しましょう。
以下の編集を加えてください。(関係ない場所は省略してます)

```java: force-app/main/default/classes/controllers/ApiSampleController.cls
public static Object createRecord(String url, String secretKey, String path, String param)

public static Object getRecords(String url, String secretKey, String path)

public static Object getRecordById(String url, String secretKey, String path)

public static Object updateRecord(String url, String secretKey, String path, String param)

public static Object deleteRecord(String url, String secretKey, String path)
```

## Service の実装

ロジックにあたる各種 Service クラスを実装していきます。

まずは汎用的な HTTP リクエスト情報を生成する `HttpRequestService`です。

以下の編集を加えてください。

```java: force-app/main/default/classes/services/HttpRequestService.cls
public with sharing class HttpRequestService {

    private String url;
    private HttpRequest request;

    public HttpRequestService(String url) {
        this.url = url;
        this.request = new HttpRequest();
    }

    public HttpRequest getRequest() {
        return this.request;
    }

    public void setEndpoint(String path) {
        String endpoint = this.url + '/' + path;
        this.request.setEndpoint(endpoint);
    }

    public void setContentType(String type) {
        this.request.setHeader('Content-Type', type);
    }

    public void setMethod(String method) {
        this.request.setMethod(method);
    }

    public void setAuthorizationHeader(String secretKey) {
        this.request.setHeader('Authorization', secretKey);
    }

    public void setBody(String body) {
        this.request.setBody(body);
    }

    public Object sendRequest(HttpRequest request) {
        Http http = new Http();
        HttpResponse response = new HttpResponse();
        response = http.send(request);
        Integer statusCode = response.getStatusCode();
        if (statusCode != 200) {
            System.debug('Error : ' + statusCode + ' => ' + response.getBody());
            throw new AuraHandledException(response.getBody());
        }
        return response.getBody();
    }
}
```

リクエストに必要最低限のエンドポイント、メソッド、ヘッダー(`content-type`, `Authorization`)、リクエストボディの情報をセットするメソッドとリクエスト送信用のメソッドを用意してます。

ドメイン URL は切り替えられるようにプロパティとして設定し、コンストラクタで定義するようにしてます。

ここをベースのクラスとして今回は ApiGateway に対してリクエストをしますので特有の設定はこちらでできるように `AwsApiGatewayService`を実装します。

まず、作成した `HttpRequestService`を `AwsApiGatewayService`で受け取ります。

以下の編集を加えてください。

```java: force-app/main/default/classes/services/AwsApiGatewayService.cls
public with sharing class AwsApiGatewayService {

    private HttpRequestService service;
    private String contentType;

    public AwsApiGatewayService(String url, String contentType) {
       this.service = new HttpRequestService(url);
       this.contentType = contentType;
    }
}
```

`content-type`も変えたい時はあるので一応プロパティで管理するようにしてます。

`HttpRequestService`の `sendRequest()`を実行するメソッドを用意しますので以下の処理を追加ください。

```java: force-app/main/default/classes/services/AwsApiGatewayService.cls
public with sharing class AwsApiGatewayService {

    // 省略

    public Object sendRequest(String secretKey, String path, String param, String method) {
        this.service.setEndpoint(path);
        this.service.setMethod(method);
        this.service.setContentType(this.contentType);
        if (param != '') {
            this.service.setBody(param);
        }
        HttpRequest request = this.service.getRequest();
        request.setHeader('x-api-key', secretKey);
        return this.service.sendRequest(request);
    }
}
```

各リクエスト情報をセットしてます。
リクエストボディはエンドポイントによっては不要なので制御してます。

`HttpRequestService`で Authorization ヘッダー設定用のメソッドは用意したのですが今回の API では `x-api-key`というヘッダーでシークレットキーを送信する仕様なのでこちらはここで設定します。

:::message
一般的には API の認証は `Authorization: Bearer token`で送信することが多いです。
:::

## Controller に Service を適用する

`AwsApiGatewayService`の `sendRequest`の引数に合わせて Controller 側でメソッドをコールするように修正しましょう。

```java: force-app/main/default/classes/controllers/ApiSampleController.cls
public with sharing class ApiSampleController {

    private static String ContentType = 'application/json';

    @AuraEnabled(cacheable=true)
    public static Object createRecord(String url, String secretKey, String path, String param) {
        AwsApiGatewayService service = new AwsApiGatewayService(url, ContentType);
        return service.sendRequest(secretKey, path, param, 'POST');
    }

    @AuraEnabled(cacheable=true)
    public static Object getRecords(String url, String secretKey, String path) {
        AwsApiGatewayService service = new AwsApiGatewayService(url, ContentType);
        return service.sendRequest(secretKey, path, '', 'GET');
    }

    @AuraEnabled(cacheable=true)
    public static Object getRecordById(String url, String secretKey, String path) {
        AwsApiGatewayService service = new AwsApiGatewayService(url, ContentType);
        return service.sendRequest(secretKey, path, '', 'GET');
    }

    @AuraEnabled(cacheable=true)
    public static Object updateRecord(String url, String secretKey, String path, String param) {
        AwsApiGatewayService service = new AwsApiGatewayService(url, ContentType);
        return service.sendRequest(secretKey, path, param, 'PUT');
    }

    @AuraEnabled(cacheable=true)
    public static Object deleteRecord(String url, String secretKey, String path) {
        AwsApiGatewayService service = new AwsApiGatewayService(url, ContentType);
        return service.sendRequest(secretKey, path, '', 'DELETE');
    }
}
```

リクエストの雛形はこれで OK です。

# LWC における秘匿情報の管理

Apex コードを実装してた時に `secretKey`を引数に渡してましたがこの情報はどこから取得するのでしょうか。

もちろんプログラムコードににハードコーディングするのは絶対にダメですね。

LWC では Salesforce の設定で `カスタム設定`というパブリックにしたくない情報を環境変数のようにして管理する手法がありますのでこちらを使って秘匿情報を管理することができます。

やり方を見ていきます。
Salesforce 組織にログインしたら設定画面のクイック検索から `カスタム設定`を検索し、`新規`ボタンをクリック

![](https://storage.googleapis.com/zenn-user-upload/0d9018fd6281-20221108.png)

以下を入力し、`保存`ボタンクリック

- 表示ラベル
  - `AWS APIGatewayの認証情報`
- オブジェクト名
  - `aws_api_gateway`

![](https://storage.googleapis.com/zenn-user-upload/077fa10aefd6-20221108.png)

オブジェクトが生成されますのでカスタム項目の `新規`ボタンクリック

![](https://storage.googleapis.com/zenn-user-upload/2cd5ba662a0a-20221108.png)

通常の Salesforce オブジェクトと同じノリで項目が作れますので `テキスト`を選択し、`次へ`ボタンをクリック

![](https://storage.googleapis.com/zenn-user-upload/a3129914ad45-20221108.png)

以下の項目設定を行い、`次へ`ボタンをクリック

1. 項目の表示ラベル: `url`
2. 文字数: `255`
3. 項目名: `url`
4. 必須項目: checked
5. 値の重複を許可しない: checked
6. 値の重複を許可しない: `「ABC」と「abc」を別の値として扱う(大文字と小文字を区別する)`

![](https://storage.googleapis.com/zenn-user-upload/b1c98f315c32-20221108.png)

`保存`ボタンをクリック。
同じ要領で `secret_key`という項目を作成ください。

![](https://storage.googleapis.com/zenn-user-upload/f71030be60b4-20221108.png)

項目を 2 つ作ったら `Manage`ボタンをクリック

![](https://storage.googleapis.com/zenn-user-upload/b2d747e5f210-20221108.png)

`新規`ボタンをクリック

![](https://storage.googleapis.com/zenn-user-upload/ac53b852bd80-20221108.png)

先ほど作った項目に値が設定できます。
保存場所は虫眼鏡をクリックし `システム管理者`を選択。
ご自身の API の情報に合わせて `保存`ボタンをクリック

![](https://storage.googleapis.com/zenn-user-upload/2a135a1a1982-20221108.png)

こちらで設定完了です。
カスタム設定で作成したオブジェクトは通常の Salesforce オブジェクトのように扱うことができます。

つまり、どういうことかというと SOQL で取得などができるということです。

実際に取得してみましょう。

vscode 上で `command⌘` + `shift⇧` + `p`でコマンドパレットを開きます。
パレット内に `sfdx`と入力し、候補で検出された `SFDX: Execute SOQL Query`を選択します。

![](https://storage.googleapis.com/zenn-user-upload/0988d2538b71-20221108.png)

input 内に `SELECT secret_key__c, url__c FROM aws_api_gateway__c`を入力

![](https://storage.googleapis.com/zenn-user-upload/a57fa7d8271f-20221108.png)

`enter⏎`を押して REST API で実行

![](https://storage.googleapis.com/zenn-user-upload/10a2749b7cdb-20221108.png)

`OUTPUT`タブを選択すると実行した sfdx コマンドの結果として SOQL の値が取得できてることが確認できます。

![](https://storage.googleapis.com/zenn-user-upload/0c3e5431fdd7-20221108.png)

こちらのオブジェクトに対して SOQL を実行する Apex を作成して LWC からコールすれば取得できそうですね。

## Apex からカスタム設定のオブジェクトを取得する

では先ほど作成したオブジェクトを取得するコードを追加しましょう。

`AwsApiGatewayService`に以下の処理を追加ください

```java: force-app/main/default/classes/services/AwsApiGatewayService.cls
public with sharing class AwsApiGatewayService {

    // 省略

    public static aws_api_gateway__c[] getApiInfo() {
        aws_api_gateway__c[] record = new List<aws_api_gateway__c>();
        record = [
            SELECT
                url__c,
                secret_key__c
            FROM
                aws_api_gateway__c
        ];
        return record;
    }
}
```

`ApiSampleController`に以下の処理を追加ください。

```java: force-app/main/default/classes/controllers/ApiSampleController.cls
public with sharing class ApiSampleController {

    // 省略

    @AuraEnabled(cacheable=true)
    public static aws_api_gateway__c[] getApiInfo() {
        return AwsApiGatewayService.getApiInfo();
    }
}
```

# API の URL をリモートサイトに設定する

API を叩くためのサーバーサイドのコードは完成しましたが、もう一つ API を扱う際には行う作業があります。`リモートサイトの設定`です。

Salesforce 組織が知らん URL に対して勝手にリクエスト送らないでということで事前に実行する API の URL は登録しておく必要があります。

設定方法を見ていきましょう。

設定画面のクイック検索ボックスで `リモートサイトの設定`と入力し、`新規リモートサイト`ボタンをクリック。

![](https://storage.googleapis.com/zenn-user-upload/802d3fa9966c-20221108.png)

`リモートサイト名`に任意の名前を入力し、`リモートサイトのURL`に対象の API の URL を入力し、`保存`ボタンをクリックすれば完了です。

![](https://storage.googleapis.com/zenn-user-upload/cb96f9cbb01d-20221108.png)

これで下準備は完了したので LWC から API を実行してみましょう。

# LWC から Apex を実行

一つのエンドポイントを試しに叩いてみましょう。

以下の編集を加えてください。

```js: force-app/main/default/lwc/apiSample/apiSample.js
import { LightningElement } from 'lwc';
import getApiInfo from "@salesforce/apex/ApiSampleController.getApiInfo";
import getRecords from "@salesforce/apex/ApiSampleController.getRecords";

export default class ApiSample extends LightningElement {

  // 省略

  async connectedCallback() {
    try {
      const apiInfo = await getApiInfo();
      if (!apiInfo.length) {
        throw 'aws_api_gateway does not exist';
      }
      const args = {
        url: apiInfo[0].url__c,
        secretKey: apiInfo[0].secret_key__c,
        path: 'staging/users',
      };
      const response = await getRecords(args);
      console.log('response', response);
    } catch(e) {
      console.error(e);
    }
  }
}
```

`connectedCallback()`は LWC のコンポーネントライフサイクルの一つで初回アクセス時にのみ `constructor()`の次に実行される関数です。

実装としては Apex で作成した `getApiInfo()`で API のアクセス情報を取得して `getRecords()`で API を叩いてるだけです。
JavaScript から Apex に渡す引数は `{ Apexで定義した引数1: 値, Apexで定義した引数2: 値 }`のようにオブジェクトにして渡します。

実行結果として API レスポンスが console で確認できることが確認できます。

![](https://storage.googleapis.com/zenn-user-upload/38aaf8d32e53-20221108.png)

テーブルの中身を API で取得した結果を反映するように修正してみます。

以下の修正を加えてください。

```diff js: force-app/main/default/lwc/apiSample/apiSample.js
import { LightningElement } from 'lwc';
import getApiInfo from "@salesforce/apex/ApiSampleController.getApiInfo";
import getRecords from "@salesforce/apex/ApiSampleController.getRecords";

export default class ApiSample extends LightningElement {
  // 省略
+ params = {};
+ allRecords = [];
+ tableBody = [];
- tableBody = [
-   {
-     id: 'abcd',
-     name: 'taro',
-     age: '20',
-     email: 'test@example.com',
-     gender: '男',
-   }
- ];
  // 省略
+ async getUsers() {
+   const params = { ...this.params }
+   params.path = `staging/users`;
+   this.allRecords = JSON.parse(await getRecords(params));
+   this.tableBody = this.allRecords.map((record) => ({...record}));
+ }

  async connectedCallback() {
    try {
-     const apiInfo = await getApiInfo();
-     if (!apiInfo.length) {
-       throw 'aws_api_gateway does not exist';
-     }
-     const args = {
-       url: apiInfo[0].url__c,
-       secretKey: apiInfo[0].secret_key__c,
-       path: 'staging/users',
-     };
-     const response = await getRecords(args);
-     console.log('response', response);


+     this.params.url = apiInfo[0].url__c;
+     this.params.secretKey = apiInfo[0].secret_key__c;
+     this.getUsers();
    } catch(e) {
      console.error(e);
    }
  }
}
```

これで初回アクセス時に常に最新の API の結果を取得してテーブルを表示します。

# 実装

API が無事取得できたので、それぞれの実装をしていきます(全件取得は先ほどやってしまいましたが)

## HTML の雛形を実装

完成形にある機能は新規追加の機能とテーブルデータをコンポボックスから選択して、絞り込み、更新、削除のアクションを選択する機能です。

画面の赤枠の箇所です。

![](https://storage.googleapis.com/zenn-user-upload/36b52a1c2f42-20221109.png)

対象の HTML を以下にまるっと書き換えください。

```html: force-app/main/default/lwc/apiSample/apiSample.html
<template>
  <lightning-card title="ユーザ一覧" icon-name="standard:service_report">
    <div slot="actions">
      <lightning-button
        variant="brand"
        label="新規"
        title="create"
        onclick={handleCreateUser}
      ></lightning-button>
    </div>

    <div class="slds-p-around_medium">
      <div class="slds-grid slds-gutters">
        <div class="slds-col slds-size_5-of-12">
          <div>
            <lightning-combobox
              name="alias"
              label="名前"
              placeholder="名前を選択"
              options={options}
              onchange={getTargetId}
              disabled={isInactive}
              class="select">
            </lightning-combobox>
          </div>
        </div>
        <div class="slds-col slds-size_3-of-12" style="display: flex; align-items: flex-end;">
          <div style="margin-left: auto;">
            <lightning-button
              variant="brand-outline"
              label="1件表示"
              title="search"
              onclick={getUser}>
            </lightning-button>
          </div>
        </div>
        <div class="slds-col slds-size_2-of-12" style="display: flex; align-items: flex-end;">
          <div style="margin-left: auto;">
            <lightning-button
              variant="success"
              label="更新"
              title="update"
              onclick={handleUpdateUser}>
            </lightning-button>
          </div>
        </div>
        <div class="slds-col slds-size_2-of-12" style="display: flex; align-items: flex-end;">
          <div style="margin-left: auto;">
            <lightning-button
              variant="destructive"
              label="削除"
              title="delete"
              onclick={deleteUser}>
            </lightning-button>
          </div>
        </div>
      </div>

      <div class="slds-m-bottom_x-small"></div>
      <lightning-datatable
        key-field="id"
        data={tableBody}
        columns={tableHead}
        hide-checkbox-column
      >
      </lightning-datatable>
    </div>
  </lightning-card>
```

`slds`というプレフィックスがついたクラスの要素は salesforce のデザインに適した bootstrap みたいなのが適用されてると捉えておいてください。

基本は `lightning`とついた要素の中に `onclick`や `onchange`などのイベントハンドラが仕込まれていることに注目してください。

以降からこれらのイベントに対する実装をしていきます。

## JavaScript の雛形のを実装

HTML に仕込んだイベントの空の関数を定義します。

```js: force-app/main/default/lwc/apiSample/apiSample.js
// 省略

export default class ApiSample extends LightningElement {
  // 省略

  // comboBox選択時の処理
  getTargetId() {}

  // 1件表示ボタンクリック時の処理
  async getUser() {}

  // 削除ボタンクリック時の処理
  async deleteUser() {}

  // 新規ボタンクリック時の処理
  handleCreateUser() {}

  // 更新ボタンクリック時の処理
  handleUpdateUser() {}

  // 省略
```

## comboBox 選択時の処理を実装

comboBox 選択時は選択を変えるごとに `onchange`イベントが発火しますので、そちらの値を格納する実装を `getTargetId`に追加します。

```diff js: force-app/main/default/lwc/apiSample/apiSample.js
// 省略

export default class ApiSample extends LightningElement {
  // 省略
+ targetId = '';
  // comboBox選択時の処理
- getTargetId()
+ getTargetId(event) {
+   this.targetId = event.target.value;
+ }
```

`event.target.value`に入ってくる値は HTML に定義している `options={options}`の箇所です。
ここではテーブルの各レコードの ID がほしいのでそれを取得して `options`の初期値に設定する処理を追加します。

```diff js: force-app/main/default/lwc/apiSample/apiSample.js
// 省略

export default class ApiSample extends LightningElement {
+ options = [];

  // 省略
+ setOptions() {
+   this.options = this.tableBody.map(({name, id}) => {
+     return {
+       label: `${name}(${id})`,
+       value: id
+     }
+   });
+ }

  async connectedCallback() {
    try {
      // 省略
+     this.setOptions();
    } catch(e) {
      console.error(e);
    }
  }
  // 省略
```

また、以降の CRUD 形の処理を実行した後に、追加削除すれば comboBox の内容が変わりますし、処理が終わった後に選択しっぱなしは良くないので初期化するようにしておきましょう。

以下の処理を追加してください。

```js: force-app/main/default/lwc/apiSample/apiSample.js
// 省略

export default class ApiSample extends LightningElement {
  // 省略
  initComboBox() {
    this.template.querySelectorAll('.select').forEach(each => {
      each.value = null;
    });
  }

  init() {
    this.setOptions();
    this.initComboBox();
    this.targetId = '';
  }
```

## get の処理を実装

get で選択した ID のレコードを取得する処理を追加します。

以下の実装を追加してください。

```js: force-app/main/default/lwc/apiSample/apiSample.js
import getRecordById from "@salesforce/apex/ApiSampleController.getRecordById";
// 省略

export default class ApiSample extends LightningElement {
  // 省略
  async getUser() {
    if (!this.getTargetId) {
      return;
    }
    const params = { ...this.params }
    params.path = `staging/users/${this.targetId}`;
    const user = JSON.parse(await getRecordById(params));
    this.tableBody = this.tableBody.filter((v) => v.id === user.id);
    this.initComboBox();
    this.targetId = '';
  }
  // 省略
```

comboBox で選択した ID でパスを生成して API を実行しているだけですね。

## delete の処理を実装

次に delete です。

以下の処理を追加してください。

```js: force-app/main/default/lwc/apiSample/apiSample.js
import deleteRecord from "@salesforce/apex/ApiSampleController.deleteRecord";
// 省略

export default class ApiSample extends LightningElement {
  // 省略
  async deleteUser() {
    if (!this.getTargetId) {
      return;
    }
    const params = { ...this.params }
    params.path = `staging/users/${this.targetId}`;
    await deleteRecord(params);
    this.allRecords = this.allRecords.filter(({id}) => id !== this.targetId);
    this.tableBody = this.allRecords.map((record) => ({...record}));
    this.init();
  }
  // 省略
```

get と同じく ID でパスを生成して API を実行。
実行後は対象の ID を前レコード格納してる `allRecords`から削除して `tableBody`を更新してます。

## create / update の処理を実装

create と update は一緒に確認しましょう。

まず作成と更新ではモーダルを表示してフォーム入力するようにしてます。

まずはモダールを作りましょう。以下の HTML を追加してください。

```html: force-app/main/default/lwc/apiSample/apiSample.html
<template>
  <lightning-card title="ユーザ一覧" icon-name="standard:service_report">
    <!--省略-->
  </lightning-card>

  <template if:true={isShowModal}>
    <section
      role="dialog"
      tabindex="-1"
      aria-labelledby="modal-heading-01"
      aria-modal="true"
      aria-describedby="modal-content-id-1"
      class="slds-modal slds-fade-in-open">

      <div class="slds-modal__container">
        <header class="slds-modal__header">
          <button
            class="slds-button
            slds-button_icon
            slds-modal__close
            slds-button_icon-inverse"
            title="Close"
            onclick={handleModal}>
            <lightning-icon
              icon-name="utility:close"
              alternative-text="close"
              variant="inverse"
              size="small" >
            </lightning-icon>
          <span class="slds-assistive-text">Close</span>
        </button>
          <h2 class="slds-text-heading_medium slds-hyphenate">追加 / 更新</h2>
        </header>

        <div class="slds-modal__content slds-p-around_medium">
          <!--ここにフォームの実装-->
        </div>
      </div>
    </section>
    <div class="slds-backdrop slds-backdrop_open"></div>
  </template>
```

`if:true={isShowModal}`というモーダルの ON/OFF を切り替えるフラグとそれを処理する `handleModal`を仕込んでますのでこちらを実行する処理を JavaScript へ追加します。

```js: force-app/main/default/lwc/apiSample/apiSample.js
// 省略

export default class ApiSample extends LightningElement {
  isShowModal = false;
  // 省略
  handleModal() {
    this.isShowModal = !this.isShowModal;
  }
  // 省略
```

フォームの実装には [lightning edit form](https://developer.salesforce.com/docs/component-library/bundle/lightning-record-edit-form/documentation) を使ってみようかと思います。
Salesforce のオブジェクトと紐付けていい感じにフォームを生成してくれるコンポーネントです。

今回はオブジェクトにレコードを登録する訳ではないですが、こっちの方がデザインやスパンメッセージなどの制御がオブジェクト設定と連動してサクッとできるのでこういった用途で使うのもありだと思います。

実装前に API のパラメータ格納用のオブジェクトと同じ項目を用意しておいてください。

![](https://storage.googleapis.com/zenn-user-upload/5e993f0e1850-20221109.png)

![](https://storage.googleapis.com/zenn-user-upload/0e3e3548b900-20221109.png)

フォームの HTML は以下です。こちらを先ほどの HTML の `<!--ここにフォームの実装-->`の部分に差し込んでください。

```html: force-app/main/default/lwc/apiSample/apiSample.html
<lightning-record-edit-form object-api-name="api_users__c">
  <div class="slds-grid slds-wrap">
    <div class="slds-col slds-size_1-of-2 slds-p-left_medium slds-p-right_medium">
      <lightning-input-field
        field-name='id__c'
        value={inputFields.id}
        class="formRequest"
        disabled>
      </lightning-input-field>
    </div>
    <div class="slds-col slds-size_1-of-2 slds-p-left_medium slds-p-right_medium">
      <lightning-input-field
        field-name='name__c'
        value={inputFields.name}
        class="formRequest">
      </lightning-input-field>
    </div>
  </div>
  <div class="slds-grid slds-wrap">
    <div class="slds-col slds-size_1-of-2 slds-p-left_medium slds-p-right_medium">
      <lightning-input-field
      field-name='age__c'
      value={inputFields.age}
      class="formRequest">
    </lightning-input-field>
    </div>
    <div class="slds-col slds-size_1-of-2 slds-p-left_medium slds-p-right_medium">
      <lightning-input-field
      field-name='email__c'
      value={inputFields.email}
      class="formRequest">
    </lightning-input-field>
    </div>
  </div>
  <div class="slds-grid slds-wrap">
    <div class="slds-col slds-size_1-of-2 slds-p-left_medium slds-p-right_medium">
      <lightning-input-field
        field-name='gender__c'
        value={inputFields.gender}
        class="formRequest">
      </lightning-input-field>
    </div>
  </div>
</lightning-record-edit-form>
<lightning-record-form
  object-api-name="api_users__c"
  onsubmit={onSubmit}
  oncancel={handleModal}>
</lightning-record-form>
```

`lightning-record-edit-form`の `object-api-name`に対してオブジェクトの API 参照名を設定することでオブジェクトの項目と連動します。

各項目は `lightning-input-field`の `field-name`に設定し、初期値などは `value`に設定します。
今回は新規の場合は空白で更新の場合現在の値を設定したいので変数にしてます。ここは後ほど JavaScript 側で設定します。

送信するときメソッドは `lightning-record-form`の `onsubmit`で行います。こちらの `object-api-name`にも `lightning-record-edit-form`と同じオブジェクト名を設定します。

こうすることで `onsubmit`で設定した関数のイベントで `lightning-input-field`の値を取得することができます。

それではメソッドの定義をしていきます。
まずは、`新規`と `更新`を押したときの `onclick`イベントに対する処理です。

ここはフォームに設定する値を設定してモーダルをオープンするだけで OK です。

以下の編集を加えてください。

```diff js: force-app/main/default/lwc/apiSample/apiSample.js
import deleteRecord from "@salesforce/apex/ApiSampleController.deleteRecord";
// 省略

export default class ApiSample extends LightningElement {
+ _inputFields = {
+   id: '',
+   name: '',
+   age: null,
+   email: '',
+   gender: '',
+ };
+ inputFields = {};
  // 省略
  handleCreateUser() {
+   const inputFields = { ...this._inputFields };
+   this.inputFields = inputFields;
+   this.handleModal();
  }

  handleUpdateUser() {
+   if (!this.getTargetId) {
+     return;
+   }
+   const inputFields = { ...this._inputFields };
+   const target = this.tableBody.find((v) => v.id === this.targetId);
+   inputFields.id = target.id;
+   inputFields.name = target.name;
+   inputFields.age = target.age;
+   inputFields.email = target.email;
+   inputFields.gender = target.gender;
+   this.inputFields = inputFields;
+   this.handleModal();
  }
  // 省略
```

これで `新規`の時は初期値なしのフォーム、`更新`の時には選択した ID の現在の値が設定されて表示されます。

次に `保存`ボタンを押した時の `onsubmit`のイベントの処理です。

以下の処理を追加してください。

```js: force-app/main/default/lwc/apiSample/apiSample.js
export default class ApiSample extends LightningElement {
  // 省略

  async onSubmit(event) {
    event.preventDefault();
    try {
      const param = {};
      let id;
      this.template.querySelectorAll(".formRequest").forEach((v) => {
        if (v.fieldName !== 'id__c') {
          param[v.fieldName.slice(0, -3)] = v.value;
        } else {
          id = v.value;
        }
      })

      if (Object.keys(param).find((col) => !param[col])) {
        return;
      }

      if (id) {
        await this.updateUser(JSON.stringify(param));
      } else {
        await this.createUser(JSON.stringify(param));
      }
      this.handleModal();
    } catch(e) {
      console.error(e);
    }
  }
  // 省略
```

`querySectorAll`でインプットを取得します。
`id`は API 側で自動採番なのでそれ以外の項目をリクエストボディのパラメータとして取得します。

Salesforce の項目は `__c`が勝手についてきてしまうのでパラメータとして使用するときは取り除いてあげてくださいね。

パラメータを設定したら `id`の有無によって `updateUser`と `createUser`でメソッドをハンドリングします。

こちらのメソッドはまだ未定義なので実装していきます。

以下の処理を追加してください。

```js: force-app/main/default/lwc/apiSample/apiSample.js
export default class ApiSample extends LightningElement {
  import createRecord from "@salesforce/apex/ApiSampleController.createRecord";
  import updateRecord from "@salesforce/apex/ApiSampleController.updateRecord";
  // 省略

  async createUser(param) {
    const params = { ...this.params }
    params.path = `staging/users`;
    params.param = param;
    const response = JSON.parse(await createRecord(params));
    this.allRecords.unshift(response);
    this.tableBody = this.allRecords.map((record) => ({...record}));
    this.init();
  }

  async updateUser(param) {
    const params = { ...this.params }
    params.path = `staging/users/${this.targetId}`;
    params.param = param;
    const response = JSON.parse(await updateRecord(params));
    this.allRecords = this.allRecords.filter(({id}) => id !== this.targetId);
    this.allRecords.unshift(response);
    this.tableBody = this.allRecords.map((record) => ({...record}));
    this.init();
  }
  // 省略
```

新規の場合は `create`の API を叩き、作成されたレコードを `allRecords`の先頭に追加し `tableBody`を更新し各初期化処理を実行

更新の場合 `update`の API を叩き、更新される前のレコードを `allRecords`から取り除いた後に先頭に追加します。
あとは同じで `tableBody`を更新し各初期化処理を実行

これで一通り CRUD の API を実行できることが確認できました。お疲れ様です。

# さいごに

さいごまで読んでいただきありがとうございます。

Salesforce という業務で触る機会がないとわからないニッチな領域かと思いますがいかがでしたでしょうか

とはいえ、Salesforce の中でも LWC は比較的新しいサービスで基本は HTML と JavaScript の Web 標準で作られてるので割ととっつきやすかったのではないでしょうか。

今後 LWC を触る機会ができた方、既に触ってるけど今回の内容を知らなかった方などの参考になれば幸いです。

今回のコードは以下で全体を確認できますので併せてご確認ください。

https://github.com/akkie-i/lwc-samples/tree/main/force-app/main/default/lwc/apiSample

間違いの指摘やリクエストなどありましたら加筆していきたので是非、ご意見をいただけたらと思います。

それではまた次の記事でお会いしましょう。
