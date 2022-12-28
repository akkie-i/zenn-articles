---
title: "【Salesforce】LWCでnpmライブラリを利用する方法"
emoji: "⚡️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Salesforce", "LWC", "npm"]
published: true
---

# はじめに

Lightning Web Components(以降 LWC)で開発している時に npm のライブラリが使えないのか気になったので調べていました。

結論から言うと、通常の node.js で開発してる時のように `package.json` に追加して対象のファイルで `import` して、みたいな使い方はできません。

と言うのも npm ライブラリを利用するためには `package.json` の内容を `npm install` して、その結果としてルートディレクトリに `node_modules` が生成されるから利用できるのですが、LWC では以下のディレクトリ構造の内、`force-app/`フォルダ配下のファイルを salesforce 組織にデプロイするだけですので、ローカルで `node_modules` を生成したからといって salesforce のサーバー上には `node_modules` を保持していないためです。

```bash
.
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
│           ├── objects
│           ├── permissionsets
│           ├── staticresources
│           ├── tabs
│           └── triggers
├── jest.config.js
├── package.json
└── sfdx-project.json
```

上記の理由から LWC で npm ライブラリを使えるようにするには一工夫が必要ですので今回はその方法を記事にしたいと思います。

## この記事でわかること

:::message

- LWC で npm ライブラリを使う方法 2 パターン
  - 静的リソースとして利用
  - npm ライブラリを LWC 化して利用

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
- npm v.8.9.0
- node v18.2.0

# 静的リソースとしての利用

こちらは公式のドキュメントなどでも記載されている salesforce からアナウンスのある方法です。

https://help.salesforce.com/s/articleView?id=sf.pages_static_resources.htm&type=5

使うための手順は以下です。

- npm ライブラリの js ファイルを静的リソースファイルとして配置する
- ライブラリ公開用の XML を配置する
- LWC コンポーネントで公開した静的リソースを読み込む

順番に確認してみます。なお例として [chart.js](https://www.npmjs.com/package/chart.js?activeTab=readme) を LWC で使うとします。

## npm ライブラリの js ファイルを静的リソースファイルとして配置する

[こちら](https://cdn.jsdelivr.net/npm/chart.js@4.1.0/dist/chart.umd.min.js)のリンクを右クリックして、「リンク先を別名で保存」をクリック。
名前を `ChartJs.js` で保存します。（ここは任意の名前で問題ないです。）

保存したファイルを `./force-app/main/default/staticresources` に配置してください。

ちなみにリンクにした npm ライブラリの js ファイルの探し方は[こちら](https://zenn.dev/akkie1030/articles/javascript-dayjs)で取り扱ってますので最新版がリリースされた時などはご自身で探してください。

## ライブラリ公開用の XML を配置する

js ファイルを配置したら同じく `./force-app/main/default/staticresources` のフォルダ配下に `{配置したjsのファイル名}.resource-meta.xml` の名前で配置します。

ここは**必ずこの命名規則に則ってください**
今回で言うと `ChartJs.resource-meta.js` です。

作成したファイルに以下の編集を加えてください。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<StaticResource xmlns="http://soap.sforce.com/2006/04/metadata">
<contentType>text/javascript</contentType>
<description>パッケージの説明</description>
</StaticResource>
```

大事なのは `<contentType>text/javascript</contentType>` の部分です。
今回は js なので `text/javascript` ですが、テキストなら `text/plain`、画像なら `image/jpeg` など適切な [MINE](https://developer.mozilla.org/ja/docs/Web/HTTP/Basics_of_HTTP/MIME_types/Common_types) タイプを指定してあげます。

このように LWC の静的リソースには `アップロードしたいファイル` + `対象のXML` で 1 セットのファイルを準備する必要があります。
※静的リソースに関わらず、この xml をセットにするのは LWC を扱う上で基本必須の作業と覚えておくと良いです。

ちなみにこの作業は GUI でもできますのでお好みで(個人的にはソースコード管理にした方が良い派です)

![](https://storage.googleapis.com/zenn-user-upload/9a41929f25cc-20221220.png)

## LWC コンポーネントで公開した静的リソースを読み込む

vscode 上で `command⌘` + `shift⇧` + `p`でコマンドパレットを開きます。
パレット内に `sfdx`と入力し、候補で検出された `SFDX: Create Lightning Web Component`を選択します。

![](https://storage.googleapis.com/zenn-user-upload/78e2e9ac95b9-20221108.png)

次に作成するコンポーネントの名前を入力します。今回は `barGraphSample`とします。

作成された `html` `js` `xml` 各ファイルに以下の編集を加えます。

```html: force-app/main/default/lwc/barGraphSample/barGraphSample.html
<template>
  <lightning-card title="月次売上" icon-name="utility:chart">
    <div class="slds-grid slds-wrap slds-grid--pull-padded">
      <div
        if:true={isChartJsInitialized}
        class="slds-col--padded slds-size--1-of-1"
      >
        <canvas class="chart" lwc:dom="manual"></canvas>
      </div>
      <div
        if:false={isChartJsInitialized}
        class="slds-col--padded slds-size--1-of-1"
      >
        ChartJs Not loaded yet
      </div>
    </div>
  </lightning-card>
</template>

```

```js: force-app/main/default/lwc/barGraphSample/barGraphSample.js
import { LightningElement, track, api } from "lwc";
import chartjs from "@salesforce/resourceUrl/ChartJs";
import { loadScript } from "lightning/platformResourceLoader";

export default class BarGraphSample extends LightningElement {
  @track isChartJsInitialized = false;
  chart;

  async connectedCallback() {
    await Promise.all([
      loadScript(this, chartjs),
    ])
      .then(() => {
        this.isChartJsInitialized = true;
      })
      .catch((e) => console.error(e));

    if (!this.isChartJsInitialized) return;

    const config = {
      type: "bar",
      data: {
        labels: ["1月", "2月", "3月", "4月", "5月", "6月"],
        datasets: [
          {
            label: "売上",
            data: [120, 80, 97, 105, 94, 110],
            backgroundColor: "#E1BEE7"
          }
        ]
      },
      options: {
        plugins: {
          datalabels: {
            font: {
              size: 13
            },
            formatter: function (value, context) {
              return value.toString() + "万円";
            }
          }
        }
      }
    };

    const ctx = this.template.querySelector("canvas.chart").getContext("2d");
    this.chart = new window.Chart(ctx, config);
    // サイズ設定
    this.chart.canvas.parentNode.style.height = "100%";
    this.chart.canvas.parentNode.style.width = "100%";
  }
}
```

```xml: force-app/main/default/lwc/barGraphSample/barGraphSample.js-meta.xml
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
  <apiVersion>55.0</apiVersion>
  <isExposed>true</isExposed>
  <targets>
    <target>lightning__AppPage</target>
    <target>lightning__RecordPage</target>
    <target>lightning__HomePage</target>
  </targets>
</LightningComponentBundle>
```

デプロイしたら棒グラフが表示されていることが確認できます。

![](https://storage.googleapis.com/zenn-user-upload/3666e203bd1c-20221220.png)

## ライブラリ毎に静的リソースをアップロードする必要がある

少し余談ですが、今やった静的リソースでライブラリをアップロードする方法はライブラリを追加したい時に毎度行わなければなりません。

試しに先ほど追加した `chart.js` に対して、プラグインである [chartjs-plugin-datalabels](https://www.npmjs.com/package/chartjs-plugin-datalabels) を追加して適応させてみましょう。

[こちら](https://cdn.jsdelivr.net/npm/chartjs-plugin-datalabels@2.2.0/dist/chartjs-plugin-datalabels.min.js) のリンクを右クリックして、「リンク先を別名で保存」をクリック。
名前を `ChartJsPluginDatalabels.js` で保存します。（ここは任意の名前で問題ないです。）

「ライブラリ公開用の XML を配置する」でやったのと同じように xml ファイルを作成し、js ファイルに以下の編集を加えます。

```diff js: force-app/main/default/lwc/barGraphSample/barGraphSample.js
import { LightningElement, track, api } from "lwc";
import chartjs from "@salesforce/resourceUrl/ChartJs";
+ import chartJsPluginDatalabels from "@salesforce/resourceUrl/ChartJsPluginDatalabels";
import { loadScript } from "lightning/platformResourceLoader";

export default class BarGraphSample extends LightningElement {
  @track isChartJsInitialized = false;
  chart;

  async connectedCallback() {
    await Promise.all([
      loadScript(this, chartjs),
+     loadScript(this, chartJsPluginDatalabels)
    ])
      .then(() => {
        this.isChartJsInitialized = true;
      })
      .catch((e) => console.error(e));

    if (!this.isChartJsInitialized) return;

    const config = {
      type: "bar",
      data: {
        labels: ["1月", "2月", "3月", "4月", "5月", "6月"],
        datasets: [
          {
            label: "売上",
            data: [120, 80, 97, 105, 94, 110],
            backgroundColor: "#E1BEE7"
          }
        ]
      },
      options: {
        plugins: {
+         tooltip: {
+         enabled: false
+         },
          datalabels: {
            font: {
              size: 13
            },
            formatter: function (value, context) {
              return value.toString() + "万円";
            }
          }
        }
      }
+     plugins: [ChartDataLabels]
    };

    const ctx = this.template.querySelector("canvas.chart").getContext("2d");
    this.chart = new window.Chart(ctx, config);
    // サイズ設定
    this.chart.canvas.parentNode.style.height = "100%";
    this.chart.canvas.parentNode.style.width = "100%";
  }
}
```

デプロイするとプラグインが適用されて表示が変わることが確認できます。

![](https://storage.googleapis.com/zenn-user-upload/e1bea6737ada-20221220.png)

プラグイン追加を例に静的リソースを追加する手順を確認しましたが、数個のライブラリならまだしももっと増えてくるちょっと大変ですよね。

また、ライブラリを以下のように非同期で読み込む必要がありますので状態を管理したり、読み込むパッケージ数が増えるごとにパフォーマンス面なども気になるところです。

```js
await Promise.all([
  loadScript(this, chartjs),
  loadScript(this, chartJsPluginDatalabels),
]);
```

以上からもう少し追加を通常の `node_modules` のように手軽に追加できないか、非同期読み込みではなくライブラリはあらかじめ読み込みが終わってる状態にできないか。
この 2 点を解決する方法を考えてみます。

# npm ライブラリを LWC 化して利用

ここでは通常の node.js 開発のように `package.json` でライブラリを管理し、追加したライブラリを lwc のコンポーネントとして利用する方法を試してみます。

これを実現する上で以下の 2 つの課題があります

:::message alert

- LWC は [web 標準](https://html.spec.whatwg.org/multipage/custom-elements.html) に準拠して開発されているため、基本は web 画面上で動く `mjs` で開発されていることが期待されますが、対して node.js だとライブラリが `cjs` で作られている場合があるのでそのままでは使えない場合があります
- LWC は 1 ファイルが 131,072 文字で収まっている必要があるのでサイズの大きいライブラリを上限の文字数で収める必要がある
  :::details ↑ 変更が入ったかもしれない(2022 年 12 月)
  以前、注意書きの記載通り 131,072 文字のファイルをデプロイすると `Error: Value too long for field: Source maximum length is:131072` のエラーが発生することを確認した記憶があるのですが、改めて擬似的にやってみたところ問題なくデプロイが出来ました。

  そもそもこのエラーは公式で見つけられなかったので、しれっとどこかのバージョンアップで改修したのかもしれないですね。

  ちなみに、検証コードは [mino0123](https://github.com/mino0123/lwc-max-size/blob/master/make-lwc.js) のスクリプトがそのまま使えそうだったので使用させていただきました。
  :::

上記を意識しながらやっていきます。

## cjs を mjs にバンドルする

依存関係のある npm ライブラリも含め web でも利用可能な形式に変換して 1 ファイルにまとめるツールとしては webpack が馴染み深いかと思います。

今回は同様のサービスですが、ES Modules 形式をサポートしている[rollup.js](https://rollupjs.org/guide/en/) を使ってみます。

まずは以下のコマンドで必要なライブラリをインストールします。

```bash
$ npm install rollup rollup-plugin-node-globals

$ npm install \
@rollup/plugin-commonjs \
@rollup/plugin-node-resolve \
rollup-plugin-node-builtins \
@rollup/plugin-terser --save-dev
```

次に使いたいライブラリをインストールします。[dayjs](https://www.npmjs.com/package/dayjs) でも使ってみましょうか。

以下のコマンドでインストールします。

```bash
$ npm install dayjs
```

次にインストールしたライブラリのバンドルファイルを格納するためのフォルダとして `./node_modules.lwc` を作成し、作成したフォルダに以下のファイルを作成します。

```js: ./node_modules.lwc/dayjs.js
import dayjs from "dayjs";
import isSameOrBefore from "dayjs/plugin/isSameOrBefore";

export {
  dayjs,
  isSameOrBefore,
};
```

ライブラリをインポートしてそれをエクスポートするだけのファイルです。せっかくなのでプラグインも検証用として一緒にエクスポートしてます。

次にライブラリをモジュール形式で使うことを `package.json` に定義します。以下の編集を加えてください。

```diff json:./package.json
{
  "name": "salesforce-app",
+ "type": "module",
  省略
  "devDependencies": {
    省略
    "@rollup/plugin-commonjs": "^23.0.4",
    "@rollup/plugin-node-resolve": "^15.0.1",
    "@rollup/plugin-terser": "^0.2.0",
    "rollup-plugin-node-builtins": "^2.1.2"
  },
  "dependencies": {
    省略
    "dayjs": "^1.11.7",
    "rollup": "^3.7.4",
    "rollup-plugin-node-globals": "^1.4.0"
  }
}

```

次にルートディレクトリに `rollup.config.js` を作成して、以下の編集を加えます。

```js:./rollup.config.js

import commonjs from "@rollup/plugin-commonjs";
import nodeResolve from "@rollup/plugin-node-resolve";
import nodeBuiltins from "rollup-plugin-node-builtins";
import nodeGlobals from "rollup-plugin-node-globals";
import terser from "@rollup/plugin-terser";

const plugins = [
  commonjs(),
  nodeBuiltins(),
  nodeGlobals({ buffer: false }),
  nodeResolve(),
  terser()
];

export default [
  {
    input: [
      "./node_modules.lwc/dayjs.js",
    ],
    output: {
      dir: "./force-app/main/default/lwc/dayjs",
      format: "es"
    },
    plugins: plugins,
  }
]
```

これで先ほど作成した `./node_modules.lwc/dayjs.js` をバンドルして `output.dir` に指定したフォルダに出力してくれます。
`format` に `es` と指定することで ES Module 形式に変換してくれます。

作成した `rollup.config.js` を以下のコマンドで実行します。

```bash
$ npx rollup -c --bundleConfigAsCjs

# 実行ログ
./node_modules.lwc/dayjs.js → ./force-app/main/default/lwc/dayjs...
created ./force-app/main/default/lwc/dayjs in 391ms
```

実行結果として `./force-app/main/default/lwc/dayjs/dayjs.js` が生成されていることが確認できます。

### バンドルしたファイルを LWC で使ってみる

作成された `./force-app/main/default/lwc/dayjs/dayjs.js` をデプロイするための xml を作成します。

```xml:./force-app/main/default/lwc/dayjs/dayjs.js-meta.xml
<?xml version="1.0" encoding="utf-8"?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>55.0</apiVersion>
    <isExposed>true</isExposed>
</LightningComponentBundle>
```

ライブラリを使うための LWC を作成します。

vscode 上で `command⌘` + `shift⇧` + `p`でコマンドパレットを開きます。
パレット内に `sfdx`と入力し、候補で検出された `SFDX: Create Lightning Web Component`を選択します。

![](https://storage.googleapis.com/zenn-user-upload/78e2e9ac95b9-20221108.png)

作成するコンポーネントの名として `usePackages` を指定してコンポーネントを作成します。

生成された js ファイルに以下の編集を加えてデプロイしてください。

```js:./force-app/main/default/lwc/usePackages/usePackages.js
import { LightningElement } from "lwc";
import { dayjs } from "c/dayjs";
import { isSameOrBefore } from "c/dayjs";

export default class UsePackages extends LightningElement {
  connectedCallback() {
    dayjs.extend(isSameOrBefore);
    console.log("dayjs", dayjs().format());
    console.log(
      "compare",
      dayjs("2020-01-01").isSameOrBefore(dayjs("2020-01-02"))
    );
  }
}
```

デプロイした後にコンポーネントを salesforce 組織に配置し、デベロッパーツールの `Console` を確認すると `dayjs 2022-12-20T08:32:57+09:00` と `compare true` のログが出力されライブラリが使えていることが確認できます。

## 依存関係のファイルを分割してバンドルする

先ほどの「cjs を mjs にバンドルする」で行った工程はライブラリ内の依存関係も含め、1 ファイルにバンドルして出力する方法です。

1 ファイルにまとめることでファイルの文字数が増えてしまうのを回避するために rollup.js では [code splitting](https://rollupjs.org/guide/en/#code-splitting) という機能を提供してます。

こちらを試してみましょう。

検証に使うライブラリとして [xml-js](https://www.npmjs.com/package/xml-js) を使ってみます。
`npm install xml-js` でインストールしておいてください。

依存関係のあるパッケージを以下のコマンドで確認できます。

```bash
$ npm info xml-js

# 実行結果
xml-js@1.6.11 | MIT | deps: 1 | versions: 49
A convertor between XML text and Javascript object / JSON text.
https://github.com/nashwaan/xml-js#readme

keywords: XML, xml, js, JSON, json, cdata, CDATA, doctype, processing instruction, Javascript, js2xml, json2xml, xml2js, xml2json, transform, transformer, transforming, transformation, convert, convertor, converting, conversion, parse, parser, parsing

bin: xml-js

dist
.tarball: https://registry.npmjs.org/xml-js/-/xml-js-1.6.11.tgz
.shasum: 927d2f6947f7f1c19a316dd8eea3614e8b18f8e9
.integrity: sha512-7rVi2KMfwfWFl+GpPg6m80IVMWXLRjO+PxTq7V2CDhoGak0wzYzFgUY2m4XJ47OGdXd8eLE8EmwfAmdjw7lC1g==
.unpackedSize: 420.6 kB

dependencies:
sax: ^1.2.4

maintainers:
- nashwaan <ysf953@gmail.com>

dist-tags:
latest: 1.6.11

published over a year ago by nashwaan <ysf953@gmail.com>
```

実行結果の `dependencies:` に依存関係が表示されます。`xml-js` の依存関係には `sax` が存在するようです。

こちらの情報を元にファイルを分割しない場合と分割した場合の差分を確認してみます。

まずは分割しない方法です。こちらは先ほどやったのでサクッと確認します。

ライブラリのエクスポートファイルを作成します。

```js: ./node_modules.lwc/xmlJs.js
import * as convert from "xml-js";

export {
  convert,
};
```

先ほどの `rollup.config.js` に以下の編集を加え、`npx rollup -c --bundleConfigAsCjs` を実行します。

```js:./rollup.config.js
// 省略
export default [
  {
    input: [
      "./node_modules.lwc/xmlJs.js",
    ],
    output: {
      dir: "./force-app/main/default/lwc/xmlJs",
      format: "es"
    },
    plugins: plugins,
  },
]
```

以下のコマンドでファイルサイズを確認します。

```bash
$ ls -la force-app/main/default/lwc/xmlJs/*

# 実行結果
-rw-r--r--  1 user  staff  87764 12 20 10:45 force-app/main/default/lwc/xmlJs/xmlJs.js
```

バンドルされたファイルの文字数は `87764` です。

続いて分割した場合を試してみます。

依存関係のライブラリのエクスポートファイルを作成します。こちらは `xml-js` インストール時に一緒に導入されているので改めてインストール必要はありません。

```js: ./node_modules.lwc/sax.js
import sax from "sax";

export {
  sax,
};
```

`rollup.config.js` に以下の記述を追加します。

```diff js:./rollup.config.js
// 省略
export default [
  {
    input: [
      "./node_modules.lwc/xmlJs.js",
+     "./node_modules.lwc/sax.js"
    ],
    output: {
      dir: "./force-app/main/default/lwc/xmlJs",
      format: "es"
    },
    plugins: plugins,
  },
]
```

上記のように依存関係のあるライブラリを分割したい場合は、`input` プロパティの配列に依存しているライブラリも含めることであとは rollup.js 側でいい感じに分割してくれます。

では、`npx rollup -c --bundleConfigAsCjs` を実行してファイルサイズを再度確認してみます。

```bash
$ ls -la force-app/main/default/lwc/xmlJs/*

# 実行結果
-rw-r--r--  1 user  staff  75200 12 20 11:04 force-app/main/default/lwc/xmlJs/sax-1f0e83fa.js
-rw-r--r--  1 user  staff     41 12 20 11:04 force-app/main/default/lwc/xmlJs/sax.js
-rw-r--r--  1 user  staff  12219 12 20 11:04 force-app/main/default/lwc/xmlJs/xmlJs.js
```

依存ライブラリと元のライブライりのファイルに分割されて 1 ファイルの文字数が減っていることが確認できます。

分割したライブラリが問題なく使えるか確認してみます。

「バンドルしたファイルを LWC で使ってみる」を参考に `force-app/main/default/lwc/xmlJs/xmlJs.js-meta.xml` を作成し、`usePackages.js` コンポーネントに以下の修正を加えます。

```js:./force-app/main/default/lwc/usePackages/usePackages.js
import { LightningElement } from "lwc";
import { convert } from "c/xmlJs";

export default class UsePackages extends LightningElement {
  connectedCallback() {
    const xmlObj = {
      declaration: {
        attributes: {
          version: "1.0",
          encoding: "utf-8"
        }
      },
      elements: [
        {
          type: "element",
          name: "parent",
          attributes: {
            attr: "attr_val"
          },
          elements: [
            {
              type: "element",
              name: "chid",
              elements: [
                {
                  type: "text",
                  text: "text"
                }
              ]
            }
          ]
        }
      ]
    };
    const xml = convert.js2xml(xmlObj, {
      compact: false,
      ignoreComment: true,
      spaces: 4
    });
    console.log(xml);
  }
```

デプロイした後にデベロッパーツールの `Console` を確認すると以下の xml 文字列が表示されていることが確認できます

```xml
<?xml version="1.0" encoding="utf-8"?>
<parent attr="attr_val">
    <child>text</child>
</parent>
```

## npm ライブラリの LWC 化をスクリプトで自動化する

最後におまけでここまでの npm ライブラリを LWC にする作業をスクリプトにしたのでよかったら試してみてください。

独自で作成する必要があるのは以下の通りで `package.lwc.json` の作成のみであとはスクリプトを実行すれば完了という流れです。

1. LWC 化したいライブラリを `package.lwc.json` に定義
2. スクリプト実行

スクリプトで実行される内容は以下です。

- `package.lwc.json` をベースに `npm install` を実行する
- `package.lwc.json` をベースにライブラリのエクスポートファイルを `./node_modules.lwc` 配下に作成
  - 依存関係が存在した場合は一緒に出力する
- `package.lwc.json` をベースに `rollup.config.js` を作成する
  - 依存関係が存在した場合は code splitting の設定に沿ってファイルが作成される
- `npx rollup -c --bundleConfigAsCjs` を実行
- 作成されたライブラリの LWC に対して `js-meta.xml` を作成する
- 作成した LWC のファイルサイズが 131,072 を超えるファイルが存在したらログを出力する

続いて、スクリプトを使うための準備を以下に記載していきます。

### スクリプト実行のための npm ライブラリをインストール

以下のコマンドでライブラリをインストールします。

```bash
$ npm install zx mustache xml-js
```

各種ライブラリの概要と使用目的は以下です。

- [zx]()
  - ターミナルで実行するコマンドを js ファイルで実行できるようにするライブラリ。rollup のコマンド実行などに使用
- [mustache]()
  - 文字列に変数値を埋め込むためのライブラリ。作成するファイルの値をライブラリによって変更するために使用
- [xml-js]()
  - XML 操作のライブラリ。`js-meta.xml` の作成に使用(上から順に読んだ方は既にインストール済みです)

### スクリプト関連のフォルダと空ファイルを作成

`createLwcPackages`フォルダを作成し、中のファイルを空で作成しておいてください。

```bash
.
├── README.md
├── config
├── createLwcPackages
│   ├── create.js-meta.xml.js
│   ├── package.lwc.json
│   ├── script.js
│   └── template.js
├── force-app
├── jest.config.js
├── package-lock.json
├── package.json
└── sfdx-project.json
```

各種ファイルの中身を書いていきます。

### package.lwc.json の定義

他のファイルはコピペで終了なのですが、ここだけは自分が使いたいライブラリに沿って定義する必要があります。

今回は既に例で使用した `dayjs` と `xml-js` を例に書き方を説明しますので、まずは以下の編集を加えてください。

```json:package.lwc.json
{
  "dependencies": [
    {
      "lib": "dayjs",
      "syntax": "dayjs",
      "filename": "dayjs",
      "exportName": "dayjs",
      "version": "",
      "plugins": [
        {
          "lib": "dayjs/plugin/isSameOrBefore",
          "syntax": "isSameOrBefore",
          "exportName": "isSameOrBefore"
        }
      ]
    },
    {
      "lib": "xml-js",
      "syntax": "* as convert",
      "exportName": "convert",
      "filename": "xmlJs",
      "version": "",
      "plugins": []
    }
  ]
}
```

説明は以下です。

- `lib`
  - npm コマンド実行時にここで定義した値を参照します。
  - エクスポートファイルを作成するときの `import xxx from "yyy";` の `yyy` の値に適用されます
- `syntax`
  - エクスポートファイルを作成するときの `import xxx from "yyy";` の `xxx` の値に適用されます
- `filename`
  - `rollup.config.js` の `input` と `output.dir` の値の設定時に使用します
- `exportName`
  - エクスポートファイルを作成するときの `export { xxx }` の `xxx` の値に適用されます
- `version`
  - npm インストールでバージョンを指定したい場合はここに値を設定します
- `plugins`
  - `dayjs` のようなライブラリ内にプラグインが用意されていて、別々でインポートする必要がある場合はここに定義。プロパティの概要は前述と同様
  - 定義するとエクスポートファイル作成時に `export { xxx, yyy }` のように一緒にエクスポートされる

### スクリプトのファイル内容を記述

スクリプト実行用の各ファイルに内容を書いていきます。こちらはコピペで問題ないです。(長いので折りたたんでおきます)

:::details js-meta.xml を生成するコード

```js:./createLwcPackages/create.js-meta.xml.js
import * as convert from "xml-js";

export const createMetaXml = () => {
  const xmlObj = {
    declaration: {
      attributes: {
        version: "1.0",
        encoding: "utf-8"
      }
    },
    elements: [
      {
        type: "element",
        name: "LightningComponentBundle",
        attributes: {
          xmlns: "http://soap.sforce.com/2006/04/metadata"
        },
        elements: [
          {
            type: "element",
            name: "apiVersion",
            elements: [
              {
                type: "text",
                text: "55.0"
              }
            ]
          },
          {
            type: "element",
            name: "isExposed",
            elements: [
              {
                type: "text",
                text: "true"
              }
            ]
          }
        ]
      }
    ]
  };

  return convert.js2xml(xmlObj, {
    compact: false,
    ignoreComment: true,
    spaces: 4
  });
};
```

:::

:::details 出力されるファイル群に使用するテンプレート

```js:./createLwcPackages/template.js
export const EXPORT = `
import {{syntax}} from "{{lib}}";
{{#plugins}}
import {{syntax}} from "{{{lib}}}";
{{/plugins}}

export {
  {{exportName}},
  {{#plugins}}
  {{exportName}},
  {{/plugins}}
};`;

export const ROLL_UP = `
import commonjs from "@rollup/plugin-commonjs";
import nodeResolve from "@rollup/plugin-node-resolve";
import nodeBuiltins from "rollup-plugin-node-builtins";
import nodeGlobals from "rollup-plugin-node-globals";
import terser from "@rollup/plugin-terser";

const plugins = [
  commonjs(),
  nodeBuiltins(),
  nodeGlobals({ buffer: false }),
  nodeResolve(),
  terser()
];

export default [
  {{#rollUps}}
  {
    input: [
      {{#input}}
      "{{{.}}}",
      {{/input}}
    ],
    output: {
      dir: "{{{output}}}",
      format: "es"
    },
    plugins: plugins,
  },
  {{/rollUps}}
]`;
```

:::

:::details スクリプトのエントリーポイントファイル

```js:./createLwcPackages/script.js
import Mustache from "mustache";
import fs from "fs";
import { $ } from "zx";
import { createMetaXml } from "./create.js-meta.xml.js";
import { EXPORT, ROLL_UP } from "./template.js";

const npmInstall = async (lib, version) => {
  const target = version ? `${lib}@${version}` : lib;
  await $`npm install ${target}`;
};

const createExportFile = async (pkg, path, template) => {
  const output = Mustache.render(template, pkg);
  await fs.writeFile(path, output, (error) => {
    if (error) throw error;
  });
  console.log(path + " created.");
};

const handleExport = async (pkg) => {
  // ライブラリのexportファイルを生成
  const exportDir = `${process.cwd()}/node_modules.lwc/`;
  await createExportFile(pkg, `${exportDir}${pkg.filename}.js`, EXPORT);

  // 依存関係の調査
  const { stdout } = await $`npm info ${pkg.lib}`;
  const npmInfo = stdout
    .split(/[\r|\n|\r\n]+/)
    .map((v) => v.trim())
    .filter((v) => v);

  // 依存パッケージが存在した場合は別でexportファイルを生成
  const hasDependencies = npmInfo.indexOf("dependencies:");
  const dependenceNames = [];
  if (hasDependencies !== -1) {
    const endDependenciesRowNum = npmInfo.indexOf("maintainers:");
    const dependencies = npmInfo
      .slice(hasDependencies + 1, endDependenciesRowNum)
      .map((v) => v.split(":")[0]);

    for (const dependence of dependencies) {
      const obj = {
        lib: dependence,
        syntax: dependence,
        // ケバブケースはキャメルケースに変換
        filename:
          dependence.indexOf("-") === -1
            ? dependence
            : dependence
                .split("-")
                .filter((v) => v !== "-")
                .map((str, i) =>
                  i === 0
                    ? str
                    : str[0].toUpperCase() + str.slice(1).toLocaleLowerCase()
                )
                .join(""),
        version: "",
        exportName: dependence,
        plugins: []
      };
      await createExportFile(obj, `${exportDir}${obj.filename}.js`, EXPORT);
      dependenceNames.push(obj.filename);
    }
  }

  const inputDir = "./node_modules.lwc/";
  const outputDir = "./force-app/main/default/lwc/";
  return {
    input: [
      ...[`${inputDir}${pkg.filename}.js`],
      ...dependenceNames.map((v) => inputDir + v + ".js")
    ],
    output: outputDir + pkg.filename
  };
};

(async function () {
  try {
    const packages = JSON.parse(
      fs.readFileSync(
        process.cwd() + "/createLwcPackages/package.lwc.json",
        "utf8"
      )
    );

    // ライブラリをインストール
    for (const pkg of packages.dependencies) {
      const { lib, version } = pkg;
      await npmInstall(lib, version);
    }

    // インストールしたライブラリのexportファイルを作成し、rollup.configに記載するライブラリのi/oオブジェクトを返却する
    const rollUpConfig = [];
    const moduleDir = process.cwd() + "/node_modules.lwc";
    if (!fs.existsSync(moduleDir)) {
      fs.mkdirSync(moduleDir);
    }
    for (const pkg of packages.dependencies) {
      /**
       * @return {
       *  input: string[],
       *  output: string,
       * }[]
       */
      const config = await handleExport(pkg);
      rollUpConfig.push(config);
    }

    // rollup.config.jsを作成
    const path = `${process.cwd()}/rollup.config.js`;
    await createExportFile({ rollUps: rollUpConfig }, path, ROLL_UP);

    // rollUp実行
    await $`npx rollup -c --bundleConfigAsCjs`;

    for (const { output } of rollUpConfig) {
      const dir = output.slice(2);

      // meta xmlファイル生成
      const xmlPath = `${dir}/${dir.split("/").slice(-1)}.js-meta.xml`;
      await fs.writeFile(xmlPath, createMetaXml(), (error) => {
        if (error) throw error;
      });

      // rollupにより作成されたlwcフォルダ内のファイルの文字数をチェック
      const { stdout } = await $`ls -la ${dir}/*`;
      const fileSizes = stdout
        .split(/[\r|\n|\r\n]+/)
        .map((v) => v.trim())
        .filter((v) => v)
        .map((v) => {
          const fileSize = v.split(/[\s|\t]+/)[4];
          return Number(fileSize);
        });

      const isSizeOver = fileSizes.some((size) => size > 131072);
      if (isSizeOver) {
        console.error(`${output} is size over.`);
        // lwcが仕様変更した？のかファイルサイズ超えたファイルもデプロイできたのでコメントアウト
        // await $`rm -rf ${output}`;
      }
    }
    await $`exit 1`;
  } catch (processOutput) {
    console.error(processOutput);
  }
})();
```

:::

### スクリプトを実行

これで準備完了なので、実行してみましょう。
上から読んで既に LWC ができてしまってる方は一旦以下のを削除してから実行してみてください。

- `rollup.config.js`
- `node_modules.lwc/` フォルダごと削除
- rollup により作成された `./force-app/main/default/lwc/{lib}` をフォルダごと削除

消したら以下のフォルダ構成になっているかと思います。

```bash
.
├── README.md
├── config
├── createLwcPackages
│   ├── create.js-meta.xml.js
│   ├── package.lwc.json
│   ├── script.js
│   └── template.js
├── force-app
│   └── main
│       └── default
│           ├── lwc
│           │   ├── jsconfig.json
│           │   └── usePackages
│           │       ├── __tests__
│           │       │   └── usePackages.test.js
│           │       ├── usePackages.html
│           │       ├── usePackages.js
│           │       └── usePackages.js-meta.xml
│           ├── objects
│           ├── permissionsets
│           ├── staticresources
│           ├── tabs
│           └── triggers
├── jest.config.js
├── package-lock.json
├── package.json
└── sfdx-project.json
```

確認したら以下のコマンドを実行

```bash
$ node ./createLwcPackages/script.js
```

諸々ファイルが生成されて以下のフォルダ構成に変わってることが確認できます。

```diff bash
  .
  ├── README.md
  ├── config
  ├── createLwcPackages
  │   ├── create.js-meta.xml.js
  │   ├── package.lwc.json
  │   ├── script.js
  │   └── template.js
  ├── force-app
  │   └── main
  │       └── default
  │           ├── lwc
+ │           │   ├── dayjs
+ │           │   │   ├── dayjs.js
+ │           │   │   └── dayjs.js-meta.xml
  │           │   ├── jsconfig.json
  │           │   ├── usePackages
  │           │   │   ├── __tests__
  │           │   │   │   └── usePackages.test.js
  │           │   │   ├── usePackages.html
  │           │   │   ├── usePackages.js
  │           │   │   └── usePackages.js-meta.xml
+ │           │   └── xmlJs
+ │           │       ├── sax-1f0e83fa.js
+ │           │       ├── sax.js
+ │           │       ├── xmlJs.js
+ │           │       └── xmlJs.js-meta.xml
  │           ├── objects
  │           ├── permissionsets
  │           ├── staticresources
  │           ├── tabs
  │           └── triggers
  ├── jest.config.js
+ ├── node_modules.lwc
+ │   ├── dayjs.js
+ │   ├── sax.js
+ │   └── xmlJs.js
  ├── package-lock.json
  ├── package.json
+ ├── rollup.config.js
  └── sfdx-project.json
```

こちらで再度デプロイして `usePackages` で動作確認してみてください。問題なく動くかと思います。

# さいごに

さいごまで読んでいただきありがとうございます。

今回のサンプルは「[GitHub](https://github.com/akkie-i/lwc-samples)」にもまとめているので併せてご確認ください。

npm パッケージを LWC で使う方法のまとめは以下です。

- 静的リソースとしての利用

  - メリット

    - 初期導入が簡単
    - もう一方の方法よりは大きいファイルサイズを扱える
      ::: details コラム：どこまでのファイルサイズを扱えるか
      過去に静的リソースでアップロードをした時にもファイルサイズオーバーのエラーが発生した記憶があるので静的リソースでも限界は多分ありそうです。

      試しにサイズが大きめのライブラリである [pdfmake](https://unpkg.com/pdfmake@0.2.6/build/pdfmake.min.js) とフォント設定用の [vfs_fonts](https://unpkg.com/pdfmake@0.2.6/build/vfs_fonts.js) を静的リソースとしてアップロードしたところアップロードは出来ました。

      サイズは以下です。

      ```bash
      $ ls -la force-app/main/default/staticresources/*

      #出力結果
      -rw-r--r--  1 user  staff  1356885 12 16 17:33 force-app/main/default/staticresources/Pdfmake.js
      -rw-r--r--  1 user  staff      205 12 16 17:36 force-app/main/default/staticresources/Pdfmake.resource-meta.xml
      -rw-r--r--  1 user  staff   798329 12 16 17:41 force-app/main/default/staticresources/PdfmakeVfsFonts.js
      -rw-r--r--  1 user  staff      205 12 16 17:42 force-app/main/default/staticresources/PdfmakeVfsFonts.resource-meta.xml
      ```

      1,000,000 行を超えても大丈夫なのである程度のライブラリは扱えそうですが、こちらでもファイルサイズオーバーのエラーが発生した場合はそのライブラリの使用は諦める必要ありかもです。
      :::

  - デメリット
    - ライブラリ導入毎に静的リソースを生成するのが手間
    - 非同期で読み込むので状態管理しないといけない
    - 複数読み込みでのパフォーマンス低下の懸念

- ES Module 形式でバンドルし LWC 化して利用
  - メリット
    - 複数のライブラリを管理する時楽
    - 非同期で読み込む必要がない
    - 非同期読み込みでないのでパフォーマンスの向上が見込める
  - デメリット
    - 初期導入に手間がかかる
    - 複数のコンポーネント間で同じライブラリを使ってる場合コードが重複して肥大化するので場合によってはパフォーマンスを下げる可能性がある。この場合は共通ライブラリを作ったりなどの別のアプローチが必要
    - LWC のファイルサイズ上限 `131,072` を超えると使えない(現在は使えるかも)。この場合は静的リソースとして利用する

お互いメリットデメリットはあるのでプロジェクトに応じて使い分けてください。

それでは、また次の記事でお会いしましょう。
