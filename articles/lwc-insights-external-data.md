---
title: "【salesforce】InsightsExternalData APIをLWCで使う"
emoji: "⚡️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["salesforce", "EinsteinAnalytics", "LWC", "InsightsExternalDataAPI"]
published: true
---

# はじめに

[こちらの記事](https://zenn.dev/akkie1030/articles/apex-insights-external-data)で取り扱いました salesforce の [Insights External Data API](https://developer.salesforce.com/docs/atlas.ja-jp.bi_dev_guide_ext_data.meta/bi_dev_guide_ext_data/bi_ext_data_overview.htm) の内容を Lightning Web Component(以降 LWC)で実装しましたので記事にします。

## この記事でわかること

:::message

- LWC から InsightsExternalData API を使って Einstein Analytics へ CSV をアップロードする方法

:::

## この記事を読む上での前提条件

:::message alert

- Einstein Analytics を試せるデベロッパー環境は準備済みの前提です
  - 用意されてない方は [こちら](https://trailhead.salesforce.com/ja/promo/orgs/analytics-de) よりアカウントを発行してください。
- [前回記事](https://zenn.dev/akkie1030/articles/apex-insights-external-data)はすでに読まれて Insights External Data API の知識は習得済みの前提です。

:::

## この記事で取り扱わないこと

:::message alert

- What is LWC 的な概念やアーキテクチャの説明とかはこの記事では扱いません
- JavaScript の基本的な記法など解説は扱いません

:::

## 環境情報

- sfdx-cli v7.152.0
- npm v.8.9.0
- node v18.2.0

## GitHub

https://github.com/akkie-i/lwc-samples/tree/main/force-app/main/default/lwc/csvUploadToEA

# 機能一覧

[Insights External Data API](https://developer.salesforce.com/docs/atlas.ja-jp.bi_dev_guide_ext_data.meta/bi_dev_guide_ext_data/bi_ext_data_overview.htm) の仕様をもとに以下の機能で実装していこうかと思います。

- アップロード先の Einstein Analytics のアプリケーション一覧を取得しコンボボックスを生成
  :::details 動作イメージ
  ![](https://storage.googleapis.com/zenn-user-upload/8a3879c6ddfb-20221229.gif)
  :::
- 選択したアプリケーションのデータセット(edgemartAlias)の有無で表示を切り替え
  :::details 動作イメージ
  ![](https://storage.googleapis.com/zenn-user-upload/07264f0adb2c-20221229.gif)
  :::
  - データセットあり：
    - アプリケーションの参照関係にあるデータセット一覧を取得しコンボボックスを生成
    - 選択したデータセットの最終アップロード時の MetadataJson からユニーク制約の有無を判定しオペレーションのコンボボックスを生成
      - ユニーク制約なし：`Overwrite`, `Append`
      - ユニーク制約あり：`Upsert`, `Delete`, `Overwrite`
      - MetadataJson に値なし：`Overwrite`
  - データセットなし
    - 新規登録用データセット名入力インプットエリア生成
    - 新規登録用メタデータ入力テキストエリア生成
    - オペレーションは `Overwrite` で固定
- オペレーションのコンボボックスにて `Overwrite` を選択時に既存の MetadataJson をセットし編集可能にする
  :::details 操作イメージ
  ![](https://storage.googleapis.com/zenn-user-upload/defb0087c2f5-20221229.gif)
  :::
- CSV アップロード用インプットエリア生成
- アプリケーション、データセット、CSV の全データの入力が揃った送信ボタンを表示しアップロード可能にする
  :::details 操作イメージ
  ![](https://storage.googleapis.com/zenn-user-upload/32b6b71549b2-20221229.gif)
  :::

# 実装

では機能一覧に沿って実装を進めていきます。

今回 Apex の実装部分は [前回記事](https://zenn.dev/akkie1030/articles/apex-insights-external-data) でほぼできているので特段説明などは行いません。
以下のコードをコピペしてあらかじめ作成しておいてください。

:::details Apex コード

```java: force-app/main/default/classes/controllers/FolderController.cls
public with sharing class FolderController {
    @AuraEnabled(cacheable=true)
    public static List<Folder> getFolders() {
        return [
            SELECT
                Id,
                Name,
                Type,
                DeveloperName,
                IsReadonly
            FROM
                Folder
            WHERE
                Type='Insights'
        ];
    }
}
```

```java: force-app/main/default/classes/controllers/InsightsExternalDataController.cls
public with sharing class InsightsExternalDataController {
    @AuraEnabled
    public static Map<String, InsightsExternalData> getEdgemartAliasesWithMetadataJson(String edgemartContainer) {
        Map<String, InsightsExternalData> response = new  Map<String, InsightsExternalData>();
        List<InsightsExternalData> externalData = new List<InsightsExternalData>();
        externalData = [
            SELECT
                Id,
                MetadataJson,
                EdgemartAlias,
                LastModifiedDate
            FROM
                InsightsExternalData
            WHERE
                EdgemartContainer =:edgemartContainer
            AND
                Status = 'Completed'
            Order By
                LastModifiedDate DESC
            LIMIT 1000
        ];

        if (externalData.size() == 0) {
            response.put('', null);
        }

        for (InsightsExternalData d : externalData) {
            String encode = System.EncodingUtil.base64Encode(d.MetadataJson);
            response.put(encode, d);
        }

        return response;
    }

    @AuraEnabled
    public static InsightsExternalData createBody(String edgemartAlias, String edgemartContainer, String metaDataJsonString, String Operation) {
        // InsightsExternalDataオブジェクトを作成
        InsightsExternalData externalData = new InsightsExternalData();
        externalData.put('Format', 'Csv');
        externalData.put('EdgemartAlias', edgemartAlias);
        if (metaDataJsonString != '') {
            // メタデータはBlob型に変換する必要がある
            Blob jsonDecode = System.EncodingUtil.base64Decode(metaDataJsonString);
            externalData.put('MetadataJson', jsonDecode);
        }
        externalData.put('Operation', Operation);
        externalData.put('Action', 'None');
        externalData.put('EdgemartContainer', edgemartContainer);
        try {
            // InsightsExternalDataを追加
            insert externalData;
            return externalData;
        } catch(DmlException e) {
            throw e;
        }
    }
    @AuraEnabled
    public static void createParts(String externalDataId, String[] csvFileString, Integer partNumberOffset) {

        try {
            List<InsightsExternalDataPart> externalDataParts = new List<InsightsExternalDataPart>();
            for (Integer i = 0; i < csvFileString.size(); i++) {
                Integer partNumber = partNumberOffset + i + 1;
                Blob csvFile = System.EncodingUtil.base64Decode(csvFileString[i]);
                InsightsExternalDataPart externalDataPart = new InsightsExternalDataPart();
                externalDataPart.put('InsightsExternalDataId', externalDataId);
                externalDataPart.put('DataFile', csvFile);
                externalDataPart.put('PartNumber', partNumber);
                externalDataParts.add(externalDataPart);
            }
            // InsightsExternalDatapartを追加
            insert externalDataParts;
        } catch(DmlException e) {
            throw e;
        }
    }

    @AuraEnabled
    public static void updateBody(String externalDataId) {
        try {
            InsightsExternalData[] processes = [
                SELECT Action FROM InsightsExternalData WHERE Id =:externalDataId
            ];
            for (InsightsExternalData externalData : processes) {
                externalData.Action = 'Process';
            }
            update processes;
        } catch(DmlException e) {
            throw e;
        }
    }
}
```

:::

## コンポーネントの作成

vscode 上で `command⌘` + `shift⇧` + `p`でコマンドパレットを開きます。
パレット内に `sfdx`と入力し、候補で検出された `SFDX: Create Lightning Web Component`を選択し以下 2 つのコンポーネントを作成します。

![](https://storage.googleapis.com/zenn-user-upload/78e2e9ac95b9-20221108.png)

- `csvUploadToEA`
  - メインのアップロードを行うコンポーネント。追加でこちらのフォルダに `metadataJson.js` を作成しておいてください。
- `utils`
  - メインコンポーネント内で使用する関数系を別に切り出したコンポーネント。こちらは HTML は必要ないです。

コンポーネントが作成できたら、それぞれの生成された `js-meta.xml` の方を編集ください。

:::details js-meta.xml の修正

```xml: csvUploadToEA.js-meta.xml
<?xml version="1.0" encoding="UTF-8" ?>
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

```xml: utils.js-meta.xml
<?xml version="1.0" encoding="UTF-8" ?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>55.0</apiVersion>
    <isExposed>true</isExposed>
</LightningComponentBundle>
```

:::

## ライブラリのインストール

以下の npm ライブラリをインストールして使えるようにしてください。

- [buffer](https://www.npmjs.com/package/buffer)
  - データのやり取りに Base64 エンコード/デコードを行うので相互変換用に使用
- [lodash](https://www.npmjs.com/package/lodash)
  - 配列操作などで実装コストを下げるためのライブラリとして使用
- [jsonschema](https://www.npmjs.com/package/jsonschema)
  - MetadataJson の形式をチェックするバリデーション用として使用

ライブラリの導入方法は以下の記事を参照ください。(本記事ではライブラリを LWC 化する方法で使用します)

[【Salesforce】LWC で npm ライブラリを利用する方法](https://zenn.dev/akkie1030/articles/lwc-use-npm-packages)

## フォルダ構成の確認

下準備は完了したので、詳細の実装に入る前に以下のフォルダ構成で雛形ファイルが生成されていることを確認ください。(必要ない箇所は省略してます)

:::details フォルダー構成

```bash
.
├── createLwcPackages
│   ├── create.js-meta.xml.js
│   ├── package.lwc.json
│   ├── script.js
│   └── template.js
├── force-app
│   └── main
│       └── default
│           ├── classes
│           │   └── controllers
│           │       ├── FolderController.cls
│           │       ├── FolderController.cls-meta.xml
│           │       ├── InsightsExternalDataController.cls
│           │       └── InsightsExternalDataController.cls-meta.xml
│           └── lwc
│               ├── buffer
│               │   ├── buffer.js
│               │   └── buffer.js-meta.xml
│               ├── csvUploadToEA
│               │   ├── __tests__
│               │   │   └── csvUploadToEA.test.js
│               │   ├── csvUploadToEA.html
│               │   ├── csvUploadToEA.js
│               │   ├── csvUploadToEA.js-meta.xml
│               │   └── metadataJson.js
│               ├── jsonschema
│               │   ├── jsonschema.js
│               │   └── jsonschema.js-meta.xml
│               ├── lodash
│               │   ├── lodash.js
│               │   └── lodash.js-meta.xml
│               └── utils
│                   ├── __tests__
│                   │   └── utils.test.js
│                   ├── utils.js
│                   └── utils.js-meta.xml
├── node_modules.lwc
│   ├── buffer.js
│   ├── jsonschema.js
│   └── lodash.js
├── package-lock.json
├── package.json
└── rollup.config.js
```

:::

## アップロード先のアプリケーション一覧コンボボックスの実装

html に以下の編集を加えビューを作成します。

```html: force-app/main/default/lwc/csvUploadToEA/csvUploadToEA.html
<template>
  <lightning-card title="CSVアップロード" icon-name="custom:custom15">
    <div class="slds-var-p-horizontal_medium">
      <lightning-combobox
        name="EinsteinAnalyticsApplication"
        label="Einstein Analytics Application"
        placeholder="アプリケーションを選択してください"
        options={edgemartContainers}
        onchange={handleApplicationChange}
      >
      </lightning-combobox>
    </div>
  </lightning-card>
</template>
```

html 内に仕込まれている `edgemartContainers` と `handleApplicationChange` の js を実装します。

`edgemartContainers` はコンボボックスのオプションのため、最初にデータ取得してセットできてれば良いので `connectedCallback`に実装します。

```js: force-app/main/default/lwc/csvUploadToEA/csvUploadToEA.js
import { LightningElement, track } from "lwc";
import getFolders from "@salesforce/apex/FolderController.getFolders";

export default class CsvUploadToEA extends LightningElement {
  @track edgemartContainers = [];

  async connectedCallback() {
    const edgemartContainers = await getFolders();
    this.edgemartContainers = edgemartContainers.map((f) => {
      return {
        label: f.Name,
        value: f.Id
      };
    });
  }
}
```

### アプリケーションの選択結果で表示が切り替わるロジックを実装

次にコンボボックスを選択した時に発火する `handleApplicationChange` の実装です。

ここは選択したアプリケーションの参照関係を持つデータセットが存在するかで表示が切り替わるのでロジックを持たせます。

まずはアプリケーションに参照関係を持つデータセットを取得する処理を書きます。
js ファイルに以下の修正を加えてください。

```js: force-app/main/default/lwc/csvUploadToEA/csvUploadToEA.js
import { LightningElement, track } from "lwc";
import getFolders from "@salesforce/apex/FolderController.getFolders";
import getEdgemartAliasesWithMetadataJson from "@salesforce/apex/InsightsExternalDataController.getEdgemartAliasesWithMetadataJson";
import {
  base64Decode,
} from "c/utils";

export default class CsvUploadToEA extends LightningElement {
  @track edgemartContainers = [];
  metadataJsonMap = {};
  edgemartContainer = "";

  async handleApplicationChange(event) {
    /**
     * 選択したアプリケーションのIDからデータセットを取得する。以下の形式でリターン
     * {'base64エンコードされたMetadataJson': 'データセットのrow'}
     */
    this.edgemartContainer = event.detail.value;
    const EdgemartAliasesWithMetadataJson =
      await getEdgemartAliasesWithMetadataJson({
        edgemartContainer: this.edgemartContainer
      });

    /**
     * データセットの取得結果から以下のオブジェクト構成にマップする
     * [{ alias: 'データセット名', metaDataJson: 'デコードした生のMetadataJson' }]
     */
    this.metadataJsonMap = Object.keys(EdgemartAliasesWithMetadataJson)
      .map((keyOfEncodedMeta) => {
        const edgemartAlias =
          EdgemartAliasesWithMetadataJson[keyOfEncodedMeta].EdgemartAlias;
        return {
          alias: edgemartAlias,
          metaDataJson: base64Decode(keyOfEncodedMeta)
        };
      })
      /**
       * 重複は不要なのでトリミング、Apex側で実行日付順でソートしてるので
       * 最初にヒットしたデータセットを残せば最新のデータセットに絞ったマップデータができる
       */
      .reduce((prev, { alias, metaDataJson }) => {
        const found = prev.find((v) => v.alias === alias);
        if (!found) {
          prev.push({ alias, metaDataJson });
        }
        return prev;
      }, []);
  }
}
```

js 内の `base64Decode` は `utils` の方に実装してますので以下の修正も加えてください。

```js: force-app/main/default/lwc/utils/utils.js
import { Buffer } from "c/buffer";
/**
 *
 * @param {string} text
 * @returns {string}
 */
const base64Decode = (text) => {
  return Buffer.from(text, "base64").toString();
};

export {
  base64Decode,
};
```

これで `this.metadataJsonMap` にデータセットの取得結果が格納されたのでこちらで表示のハンドリングを行います。js ファイルに以下の修正を加えてください。

```diff js: force-app/main/default/lwc/csvUploadToEA/csvUploadToEA.js
import { LightningElement, track } from "lwc";
// 省略

export default class CsvUploadToEA extends LightningElement {
  // 省略
+  @track edgemartAliases = [];
+  edgemartAlias = "";
+  operation = "";
+  metaDataJson = "";
+  isFirstUpload = false;
+  isAfterFirstUpload = false;
+  isOperationByOverwrite = false;
+  isInputCompleted = false;
+  isSelectedEdgemartAlias = false;
+  isSelectedApplication = false;
+  file = null;

+  handleProgress() {
+    this.isSelectedApplication = !!this.edgemartContainer;
+    const inputs = [
+      this.edgemartContainer,
+      this.edgemartAlias,
+      this.operation,
+      this.file
+    ];
+
+    this.isInputCompleted = inputs.every((input) => input);
+  }
+
+  setOperation(operation) {
+    this.operation = operation;
+    this.isOperationByOverwrite = this.operation === "Overwrite";
+  }
+
+  setEdgemartAliases() {
+    this.edgemartAliases = this.metadataJsonMap.map(({ alias }) => {
+      return {
+        label: alias,
+        value: alias
+      };
+    });
+  }
+
+  handleFormDisplay(isFirstUpload, isAfterFirstUpload) {
+    this.isFirstUpload = isFirstUpload;
+    this.isAfterFirstUpload = isAfterFirstUpload;
+    this.metaDataJson = "";
+    this.edgemartAlias = "";
+    this.isSelectedEdgemartAlias = false;
+  }

  async handleApplicationChange(event) {
    // 省略

    this.metadataJsonMap = // 省略

+    if (!this.metadataJsonMap.length) {
+      this.handleFormDisplay(true, false);
+      this.setOperation("Overwrite");
+    } else {
+      this.handleFormDisplay(false, true);
+      this.setEdgemartAliases();
+      this.setOperation("");
+    }
+    this.handleProgress();
  }
}
```

if 文内でコールしてる関数の概要は以下です。

- `handleFormDisplay`
  - 初回アップロードか初回以降アップロードで表示が切り替わるため切り替えのフラグなどをセットする関数
- `setOperation`
  - アップロード処理のオペレーションを設定する関数。初回アップロードの場合は `Overwrite` しかできないのでここで設定、初回以降はオペレーションのコンボボックスにより決まるのでここでは空で設定
- `setEdgemartAliases`
  - `metadataJsonMap` を使いデータセットコンボボックス用の変数を作る関数
- `handleProgress`
  - 必要な入力値が全部揃ったら `アップロード` ボタン表示の切り替えを行う関数。各アクションごとに実行する

ここで次のアクションを行うための変数の設定系が完了しましたので次のステップに移ります。

## 初回アップロード時の画面処理を実装

新規アップロードの場合はデータセットが 1 つもないため、データセット名を決める必要がありますのでインプットエリアを配置します。

```html: force-app/main/default/lwc/csvUploadToEA/csvUploadToEA.html
<template>
  <lightning-card title="CSVアップロード" icon-name="custom:custom15">
    <div class="slds-var-p-horizontal_medium">
      <!-- 省略 -->

      <template if:true={edgemartContainer}>
        <!-- 初回アップロードの場合 -->
        <template if:true={isFirstUpload}>
          <lightning-input
            type="text"
            name="dataset-name"
            label="データセット名"
            placeholder="半角英数字のみ、連続した_、末尾の_は不可"
            onchange={handleNewEdgemartAliasChange}
            class="slds-var-m-top_small"
          ></lightning-input>
        </template>
      </template>
    </div>
  </lightning-card>
</template>
```

対応する関数 `handleNewEdgemartAliasChange` は以下です。こちらはイベントの値を変数に格納しているだけですので特段説明はございません。

```js: force-app/main/default/lwc/csvUploadToEA/csvUploadToEA.js
import { LightningElement, track } from "lwc";
// 省略

export default class CsvUploadToEA extends LightningElement {

  // 省略

  handleNewEdgemartAliasChange(event) {
    this.edgemartAlias = event.detail.value;
    this.handleProgress();
  }
}
```

次に初回および、オペレーションを `Overwrite` で選択した時は MetadataJson は変更できるようにしたいのでその処理を書いていきます。

```html: force-app/main/default/lwc/csvUploadToEA/csvUploadToEA.html
<template>
  <lightning-card title="CSVアップロード" icon-name="custom:custom15">
    <div class="slds-var-p-horizontal_medium">
      <!-- 省略 -->

      <template if:true={edgemartContainer}>
        <!-- 初回アップロードの場合 -->
        <!-- 省略 -->

        <!-- operationがOverwriteの場合 -->
        <template if:true={isOperationByOverwrite}>
          <div class="slds-var-m-top_medium" style="position: relative">
            <div>
              <lightning-textarea
                name="meta-data"
                label="メタデータ"
                placeholder="空白、またはJson形式のCSVヘッダの構造情報を記載"
                value={metaDataJson}
              ></lightning-textarea>
            </div>
            <div style="position: absolute; top: -8px; right: 0">
              <lightning-button
                label="テンプレート作成"
                title="createTemplate"
                onclick={setTemplateMetadataJson}
              ></lightning-button>
            </div>
          </div>
        </template>
      </template>
    </div>
  </lightning-card>
</template>
```

特にアクションを持たないテキストエリアと、MetadataJson の雛形をテキストエリアに入力するボタンアクションである `setTemplateMetadataJson` を設定します。

`setTemplateMetadataJson`の実装は以下です。ただ変数を転記してるだけなのでここも特段説明はございません。

```js: force-app/main/default/lwc/csvUploadToEA/csvUploadToEA.js
import { LightningElement, track } from "lwc";
import { templateMetadataJson } from "./metadataJson";
// 省略

export default class CsvUploadToEA extends LightningElement {

  // 省略

  setTemplateMetadataJson() {
    this.metaDataJson = JSON.stringify(templateMetadataJson);
  }
}
```

```js: force-app/main/default/lwc/csvUploadToEA/metadataJson.js
export const templateMetadataJson = {
  fileFormat: {
    charsetName: "UTF-8",
    fieldsDelimitedBy: ",",
    linesTerminatedBy: "\n"
  },
  objects: [
    {
      connector: "CSV",
      fullyQualifiedName: "sample_csv",
      label: "sample.csv",
      name: "sample_csv",
      fields: [
        {
          fullyQualifiedName: "Column1",
          name: "Column1",
          type: "Text",
          label: "名前"
        }
      ]
    }
  ]
};
```

これで以下の実装ができました。

![](https://storage.googleapis.com/zenn-user-upload/1f77df2bc63f-20221229.png)

## 初回以降アップロードの画面処理を実装

初回以降のアップロードの場合はすでに実行されたデータセットのコンボボックスが表示されます。

```html: force-app/main/default/lwc/csvUploadToEA/csvUploadToEA.html
<template>
  <lightning-card title="CSVアップロード" icon-name="custom:custom15">
    <div class="slds-var-p-horizontal_medium">
      <!-- 省略 -->

      <template if:true={edgemartContainer}>
        <!-- 初回アップロードの場合 -->
        <!-- 省略 -->

        <!-- 初回以降のアップロードの場合 -->
        <template if:true={isAfterFirstUpload}>
          <lightning-combobox
            name="dataset"
            label="データセット"
            placeholder="アップロード先のデータセットを選択してください"
            options={edgemartAliases}
            onchange={handleEdgemartAliasChange}
            class="slds-var-m-top_small"
          >
          </lightning-combobox>
        </template>
      </template>
    </div>
  </lightning-card>
</template>
```

コード内の `edgemartAliases` ですが、ここはアプリケーション選択時の `handleApplicationChange` にて設定されています。

`handleEdgemartAliasChange` の実装は以下です。

```js: force-app/main/default/lwc/csvUploadToEA/csvUploadToEA.js
import { LightningElement, track } from "lwc";
// 省略

export default class CsvUploadToEA extends LightningElement {
  operations
  // 省略

  setOperations() {
    const operations = {
      hasUnique: [
        { label: "Upsert", value: "Upsert" },
        { label: "Delete", value: "Delete" },
        { label: "Overwrite", value: "Overwrite" }
      ],
      noUnique: [
        { label: "Overwrite", value: "Overwrite" },
        { label: "Append", value: "Append" }
      ],
      noMetaData: [{ label: "Overwrite", value: "Overwrite" }]
    };
    const fields = this.metaDataJson
      ? JSON.parse(this.metaDataJson).objects[0].fields
      : null;

    this.operations = fields?.some((field) => field.isUniqueId)
      ? operations.hasUnique
      : fields
      ? operations.noUnique
      : operations.noMetaData;
  }

  handleEdgemartAliasChange(event) {
    this.edgemartAlias = event.detail.value;
    this.metaDataJson = this.metadataJsonMap.find(
      ({ alias }) => alias === this.edgemartAlias
    ).metaDataJson;
    this.setOperations();
    this.isSelectedEdgemartAlias = true;
    this.handleProgress();
  }
}
```

`handleEdgemartAliasChange` で生成された `metaDataJsonMap` から EdgemartAlias 名に一致する MetadataJson を取得し、その取得した json をもとに `setOperations` でオペレーションのコンボボックスに使用する変数の値を設定します。

MetadataJson の形式は以下 3 つのパターンが考えられ、パターンによって実行できるアクションも決まります。([参考](https://developer.salesforce.com/docs/atlas.ja-jp.240.0.bi_dev_guide_ext_data.meta/bi_dev_guide_ext_data/bi_ext_data_object_externaldata.htm))

- MetadataJsonJson 内に `isUniqueId: true`の項目を保持する
  - ユニークキー制約があるときは `Upsert`, `Delete` はできるが `Append` はできない
- MetadataJsonJson 内に `isUniqueId: true`の項目を保持しない
  - ユニークキー制約がないときは `Append` はできるが `Upsert`, `Delete` はできない
- MetadataJsonJson の値がない
  - `Overwrite` は MetadataJson がなくとも実行できるので MetadataJson を保持しない可能性もある
  - MetadataJson がない場合は `Overwrite` しかできない

この処理によって以下の html の値と表示のハンドリングが決定します。

```html: force-app/main/default/lwc/csvUploadToEA/csvUploadToEA.html
<template>
  <lightning-card title="CSVアップロード" icon-name="custom:custom15">
    <div class="slds-var-p-horizontal_medium">
      <!-- 省略 -->

      <template if:true={edgemartContainer}>
        <!-- 初回アップロードの場合 -->
        <!-- 省略 -->

        <!-- 初回以降のアップロードの場合 -->
        <template if:true={isAfterFirstUpload}>
          <!-- 省略 -->

          <!-- 最終アップロードのMetaDataJsonによってオペレーションが
              が変わるためデータセットを選択するまで非活性
          -->
          <template if:false={isSelectedEdgemartAlias}>
            <lightning-combobox
              name="operation"
              label="オペレーション"
              value="operation"
              placeholder="実行する処理を選択してください"
              options={operations}
              class="slds-var-m-top_small"
              disabled=""
            >
            </lightning-combobox>
          </template>
          <template if:true={isSelectedEdgemartAlias}>
            <lightning-combobox
              name="operation"
              label="オペレーション"
              value={operation}
              placeholder="実行する処理を選択してください"
              options={operations}
              onchange={handleOperationsChange}
              class="slds-var-m-top_small"
            >
            </lightning-combobox>
          </template>
        </template>
      </template>
    </div>
  </lightning-card>
</template>
```

`handleOperationsChange` の実装は以下です。イベントの値によってオペレーションを設定しているだけですので特段説明はございません。

```js: force-app/main/default/lwc/csvUploadToEA/csvUploadToEA.js
import { LightningElement, track } from "lwc";
import { templateMetadataJson } from "./metadataJson";
// 省略

export default class CsvUploadToEA extends LightningElement {

  // 省略

  handleOperationsChange(event) {
    this.setOperation(event.detail.value);
    this.handleProgress();
  }
}
```

これで以下の実装ができました。

![](https://storage.googleapis.com/zenn-user-upload/ab2ce4b1e26d-20221229.png)

## CSV アップロード用インプットエリア実装

CSV を tmp でアップロードする以下の箇所です。

![](https://storage.googleapis.com/zenn-user-upload/c5a9ff647e93-20221229.png)

html, js はそれぞれ以下です。アップロードしたファイルオブジェクトをそのまま変数に格納するだけなので特段説明はございません。

```html: force-app/main/default/lwc/csvUploadToEA/csvUploadToEA.html
<template>
  <lightning-card title="CSVアップロード" icon-name="custom:custom15">
    <div class="slds-var-p-horizontal_medium">
      <!-- 省略 -->

      <template if:true={edgemartContainer}>
        <!-- 省略 -->

        <template if:true={isSelectedApplication}>
          <lightning-input
            type="file"
            name="csv"
            label="CSV"
            onchange={handleCsvUpload}
            accept="text/csv"
            class="slds-var-m-top_small"
          ></lightning-input>
          <span>{fileName}</span>
        </template>
      </template>
    </div>
  </lightning-card>
</template>
```

```js: force-app/main/default/lwc/csvUploadToEA/csvUploadToEA.js
import { LightningElement, track } from "lwc";
import { templateMetadataJson } from "./metadataJson";
// 省略

export default class CsvUploadToEA extends LightningElement {
  file = null;
  fileName = "";
  // 省略

  handleCsvUpload(event) {
    this.file = event.detail.files[0];
    this.fileName = this.file.name;
    this.handleProgress();
  }
}
```

## アップロード処理の実装

最後にメインのアップロード処理です。
ここまでの入力内容が揃ったらボタンが表示される。ボタンを押したらスピナーとアップロード進行のプログレスバーが表示されるようにした html です。

```html: force-app/main/default/lwc/csvUploadToEA/csvUploadToEA.html
<template>
  <lightning-card title="CSVアップロード" icon-name="custom:custom15">
    <div class="slds-var-p-horizontal_medium">
      <!-- 省略 -->

      <template if:true={edgemartContainer}>
        <!-- 省略 -->
        <template if:true={isInputCompleted}>
          <div class="slds-var-m-top_small">
            <lightning-button
              variant="brand"
              label="アップロード"
              title="Primary action"
              onclick={handleUpload}
              class="slds-var-m-left_x-small"
            >
            </lightning-button>
          </div>
        </template>
      </template>

      <div if:true={isLoading}>
        <lightning-spinner
          alternative-text="Loading..."
          variant="brand"
          class="slds-var-m-bottom_small"
        ></lightning-spinner>

        <div class="slds-is-relative" style="top: 10px">
          <lightning-progress-bar
            value={loadingBar}
            size="medium"
          ></lightning-progress-bar>
        </div>
      </div>
    </div>
  </lightning-card>
</template>
```

`handleUpload` の実装は以下に続きます。

### バリデーション処理の実装

バリデーションとして以下の 2 点を実装します。

- 初回アップロード時のデータセット名の形式チェックバリデーション
- MetaDataJson の形式チェックバリデーション

順番に確認します。

#### 初回アップロード時のデータセット名の形式チェックバリデーション

項目名には以下の制限があります。

![](https://storage.googleapis.com/zenn-user-upload/b04838b6bd40-20221229.png)

まとめると先頭の文字が半角英字で、`_` 以外の特殊記号は使われておらず、末尾が `_` で終了してない。
とすればオッケーかなと思います。

まず上記のチェックする関数を `utils` に登録します。

```js: force-app/main/default/lwc/utils/utils.js
/**
 *
 * @param {string} input
 * @returns {boolean}
 */
const sfCustomObjectNameRule = (input) => {
  return (
    // 先頭が英字で始まる
    input.match(/^[a-zA-Z].*$/) &&
    // 半角英数字_以外は使えない
    input.match(/^[a-zA-Z0-9\\_]*?$/) &&
    // _で終わらない
    input.match(/^(?!.*_$).*$/)
  );
};

export {
  base64Decode,
  sfCustomObjectNameRule,
};
```

こちらをメインのコンポーネント側にインポートして EdgemartAlias に対して実行します。
バリデーションを突破できなかった場合はそこで処理を終了してエラーメッセージのポップアップを表示します。

```js: force-app/main/default/lwc/csvUploadToEA/csvUploadToEA.js
import { LightningElement, track } from "lwc";
import {
  base64Decode,
  sfCustomObjectNameRule,
  showToastMessage
} from "c/utils";

export default class CsvUploadToEA extends LightningElement {
  // 省略

  validation() {
    if (this.isFirstUpload) {
      return sfCustomObjectNameRule(this.edgemartAlias);
    }
    return true;
  }

  async handleUpload() {
    try {
      this.isLoading = true;
      const isPassed = this.validation();
      if (!isPassed) {
        showToastMessage(
          this,
          "Upload Failed",
          "データセット名、またはメタデータの形式が正しくありません",
          "error"
        );
        return;
      }
    } catch (e) {
      console.error("upload error", e);
    }
  }
}
```

ポップアップメッセージの処理はどこでも使えるように `utils` に登録してますので以下の修正をしてください。

```js: force-app/main/default/lwc/utils/utils.js
import { ShowToastEvent } from "lightning/platformShowToastEvent";
/**
 *
 * @param {any} that
 * @param {string} title
 * @param {string} message
 * @param {info | success | warning | error} variant
 * @returns
 */
const showToastMessage = (that, title, message, variant) => {
  that.dispatchEvent(new ShowToastEvent({ title, message, variant }));
};

export {
  base64Decode,
  sfCustomObjectNameRule,
  showToastMessage
};
```

#### MetaDataJson の形式チェックバリデーション

MetadataJson は field の箇所は CSV の内容によって変わりますが後はフォーマットは決まってますのでフォーマット通りの json になっているか、また EdgemartAlias と同じ命名の制限があったりするのでそのあたりのバリデーションを実装します。(詳しい項目などは[こちら](https://developer.salesforce.com/docs/atlas.ja-jp.240.0.bi_dev_guide_ext_data_format.meta/bi_dev_guide_ext_data_format/bi_ext_data_schema_reference.htm)を参照)

まずは json のフォーマットチェックに使うスキーマを `metadataJson.js`に追記します。(長いから折りたたみます)

:::details metadataJson.js

```js: force-app/main/default/lwc/csvUploadToEA/metadataJson.js
export const schema = {
  type: "object",
  required: ["fileFormat", "objects"],
  properties: {
    fileFormat: {
      type: "object",
      properties: {
        charsetName: {
          type: "string",
          pattern: "UTF-8",
          required: true
        },
        fieldsDelimitedBy: {
          type: "string",
          pattern: /^[,]$/,
          required: true
        },
        linesTerminatedBy: {
          type: "string",
          pattern: "\n",
          required: true
        }
      }
    },
    objects: {
      type: "array",
      minItems: 1,
      items: {
        type: "object",
        required: [
          "connector",
          "fullyQualifiedName",
          "name",
          "label",
          "fields"
        ],
        properties: {
          connector: {
            type: "string",
            pattern: "CSV"
          },
          fullyQualifiedName: {
            type: "string",
            format: "sfCustomObjectNameRule"
          },
          name: {
            type: "string",
            format: "sfCustomObjectNameRule"
          },
          label: {
            type: "string"
          },
          fields: {
            type: "array",
            minItems: 1,
            items: {
              type: "object",
              required: ["fullyQualifiedName", "name", "type", "label"],
              properties: {
                fullyQualifiedName: {
                  type: "string",
                  format: "sfCustomObjectNameRule"
                },
                name: {
                  type: "string",
                  format: "sfCustomObjectNameRule"
                },
                type: {
                  type: "string",
                  pattern: /^Text$|^Numeric$|^Date$/
                },
                label: {
                  type: "string"
                }
              }
            }
          }
        }
      }
    }
  }
};
```

:::

細かい使用方法は[ライブラリ側](https://github.com/tdegrunt/jsonschema)にお任せしますが、基本プロパティの必須チェックと型チェックは全てに適用しており、プロパティの値に制限をかけたいものは `pattern` で正規表現チェックを行うか、`format` でチェック関数をプロトタイプに追加して実行してチェックを行ってます。

`format`に使用する関数は先ほどの `sfCustomObjectNameRule` の使い回しで、プロトタイプの追加は以下のようにして実装してます。

```js: force-app/main/default/lwc/utils/utils.js
import { jsonschema } from "c/jsonschema";
const Validator = jsonschema.Validator;
/**
 *
 * @param {string} funcName
 * @param {() => boolean} func
 * @return {void}
 */
const extendValidator = (funcName, func) => {
  Validator.prototype.customFormats[funcName] = func;
};

export {
  base64Decode,
  sfCustomObjectNameRule,
  showToastMessage,
  extendValidator
};
```

```diff js: force-app/main/default/lwc/csvUploadToEA/csvUploadToEA.js
import { LightningElement, track } from "lwc";
import {
  base64Decode
  sfCustomObjectNameRule,
  showToastMessage,
+  extendValidator
} from "c/utils";

export default class CsvUploadToEA extends LightningElement {
  // 省略

  async connectedCallback() {
+    extendValidator("sfCustomObjectNameRule", sfCustomObjectNameRule);
    const edgemartContainers = await getFolders();
    this.edgemartContainers = edgemartContainers.map((f) => {
      return {
        label: f.Name,
        value: f.Id
      };
    });
  }
}
```

これを実行する処理を以下に実装します。

```js: force-app/main/default/lwc/utils/utils.js
/**
 *
 * @param {object} target
 * @param {object} schema
 * @returns {boolean}
 */
const isValidJsonFormat = (target, schema) => {
  const validator = new Validator();
  return validator.validate(target, schema).valid;
};

export {
  base64Decode,
  sfCustomObjectNameRule,
  showToastMessage,
  extendValidator,
  isValidJsonFormat
};
```

アップロード処理側に適用させます。

```diff js: force-app/main/default/lwc/csvUploadToEA/csvUploadToEA.js
import { LightningElement, track } from "lwc";
import { templateMetadataJson, schema } from "./metadataJson";
import {
  base64Decode
  sfCustomObjectNameRule,
  showToastMessage,
  extendValidator,
+  isValidJsonFormat
} from "c/utils";

export default class CsvUploadToEA extends LightningElement {
  // 省略

  validation() {
    if (this.isFirstUpload) {
      return sfCustomObjectNameRule(this.edgemartAlias);
    }
+    if (this.isOperationByOverwrite) {
+      this.metaDataJson =
+        this.template.querySelector("lightning-textarea").value;
+      if (this.metaDataJson) {
+        return isValidJsonFormat(JSON.parse(this.metaDataJson), schema);
+      }
+    }
    return true;
  }
}
```

MetadataJson の値が書きかわってる可能性があるのは `Overwrite` の時だけで `Overwrite` の値は空の可能性もあるのでその制御を加えた上で条件に一致した場合バリデーションを実行しています。

### InsightExternalData 作成の実装

バリデーションを通過したら InsightExternalData を作成します。
以下の修正をおこなってください。

```js: force-app/main/default/lwc/csvUploadToEA/csvUploadToEA.js
import { LightningElement, track } from "lwc";
import createBody from "@salesforce/apex/InsightsExternalDataController.createBody";
import {
  base64Decode
  sfCustomObjectNameRule,
  showToastMessage,
  extendValidator,
  isValidJsonFormat,
  base64Encode
} from "c/utils";

export default class CsvUploadToEA extends LightningElement {
  // 省略

  async handleUpload() {
    try {
      // 省略

      const externalData = await createBody({
        edgemartAlias: this.edgemartAlias,
        edgemartContainer: this.edgemartContainer,
        metaDataJsonString: this.metaDataJson
          ? await base64Encode(this.metaDataJson)
          : "",
        Operation: this.operation
      });
      const externalDataId = externalData.Id;

    } catch (e) {
      console.error("upload error", e);
    }
  }
}
```

Apex を実行して、発行される Id を後続の処理で使用するために変数に格納しておきます。

また、`base64Encode` の処理を `utils` の方にも追加します。

```js: force-app/main/default/lwc/utils/utils.js
/**
 * @param  {...string} parts
 * @returns {string}
 */
const base64Encode = (...parts) => {
  return new Promise((resolve) => {
    const reader = new FileReader();
    reader.onload = () => {
      const offset = reader.result.indexOf(",") + 1;
      resolve(reader.result.slice(offset));
    };
    reader.readAsDataURL(new Blob(parts));
  });
};

export {
  base64Decode,
  sfCustomObjectNameRule,
  showToastMessage,
  extendValidator,
  isValidJsonFormat,
  base64Encode
};
```

### InsightsExternalDataPart 作成の実装

次に実際の CSV の 行データを登録する InsightsExternalDataPart の登録です。

ここのデータ量は結構多くなることが見込まれるので、チャンクして処理を分割させていこうと思います。

分割はまず CSV の行は 1000 件で分割します。
1 万行あったと仮定して、これだと InsightsExternalDataPart への登録処理を行う SOQL が 10 回発行されてしまいます。

ちょっと心もとないのでこれをさらに 10 個のかたまりにして 1000×10 の配列で Apex に渡し、登録処理はバルクで実行します。

これなら SOQL も 1 回の発行でおさまるのでまぁ良いパフォーマンスになるかと思います。

また、これを行うため CSV を配列に変換する処理と、配列を分割する処理を `utils` に書きますので先にそちらを実装します。

また、一回登録するごとに 1 秒スリープさせたいので遅延処理も追加します。(これは実際なくてもいいです。少ないデータ数だとプログレスバーの進行がほぼ見えずすぐ終わってしまうのでサンプルとして見栄えをよくするために実装してます。)

```js: force-app/main/default/lwc/utils/utils.js
import { _ } from "c/lodash";

/**
 * @param {object} fileObject
 * @returns {Array<string>}
 */
const convertCsvToArray = (fileObject) => {
  return new Promise((resolve) => {
    const fileReader = new FileReader();
    fileReader.onload = function (fileLoadedEvent) {
      // テキスト化したCSVの中身を取得
      const textFromFileLoaded = fileLoadedEvent.target.result;
      resolve(textFromFileLoaded.split(/[\r|\n|\r\n]+/).filter((v) => v));
    };
    // ファイルオブジェクトをテキストで読み込み
    fileReader.readAsText(fileObject, "UTF-8");
  });
};

/**
 *
 * @param {Array<string | number | boolean | null | undefined>} array
 * @param {number} size
 * @returns {Array<Array<string | number | boolean | null | undefined>>}
 */
const arrayChunk = (array, size) => {
  return _.chunk(array, size).map((v) => [...v]);
};

/**
 *
 * @param {number} second
 * @returns {Promise<any>}
 */
const delay = (second) => {
  const ms = second * 1000;
  // eslint-disable-next-line @lwc/lwc/no-async-operation
  return new Promise((resolve) => setTimeout(resolve, ms));
};

export {
  base64Encode,
  base64Decode,
  convertCsvToArray,
  arrayChunk,
  showToastMessage,
  sfCustomObjectNameRule,
  extendValidator,
  isValidJsonFormat,
  delay
};
```

次に実際の登録処理の実装です。以下の修正を加えてください。

```js: force-app/main/default/lwc/csvUploadToEA/csvUploadToEA.js
import { LightningElement, track } from "lwc";
// 省略
import {
  base64Decode,
  base64Encode,
  convertCsvToArray,
  arrayChunk,
  showToastMessage,
  sfCustomObjectNameRule,
  extendValidator,
  isValidJsonFormat,
  delay
} from "c/utils";
import createParts from "@salesforce/apex/InsightsExternalDataController.createParts";

export default class CsvUploadToEA extends LightningElement {
  // 省略

  async handleUpload() {
    try {
      // 省略

      // csvを1万行数で分割
      const csvArray = await convertCsvToArray(this.file);
      const csvHead = csvArray[0];
      const csvRows = csvArray.slice(1, csvArray.length);
      const csvChunkRows = arrayChunk(csvRows, 1000);

      // 1000*10の1万のデータでExternalDataPartの登録処理を実行
      let partNumberOffset = 0;
      const partNumberLimit = 10;
      let csvParts = [];
      let counter = 0;
      for await (const csvChunkRow of csvChunkRows) {
        /**
         * InsightsExternalDataApiの仕様として
         * 複数partNumberでデータ登録するとき最初は
         * ヘッダーあり、以降は先頭に改行がある構造にする必要がある
         */
        const prefix = !counter ? [csvHead] : ["\n"];
        csvParts.push([...prefix, ...csvChunkRow]);
        const isChunkLimit = csvParts.length >= partNumberLimit;
        const isLastChunk = counter === csvChunkRows.length - 1;

        if (isLastChunk || isChunkLimit) {
          const csvFileString = await Promise.all(
            csvParts
              .map((part) => part.join("\n"))
              .map(async (v) => {
                const encodedCsvPart = await base64Encode(v);
                return encodedCsvPart;
              })
          );

          await createParts({
            externalDataId,
            csvFileString,
            partNumberOffset
          });
          if (isLastChunk) {
            this.loadingBar = 100;
            break;
          } else if (isChunkLimit) {
            partNumberOffset += partNumberLimit;
            csvParts = [];
          }
        }
        this.loadingBar = (100 / csvChunkRows.length) * counter + 1;
        counter++;
        await delay(1);
      }

    } catch (e) {
      console.error("upload error", e);
    }
  }
}
```

冒頭で書いた通り分割をしながら登録するようにしてます。

### InsightExternalData API 実行と初期化処理の実装

最後に InsightExternalData のアクションを更新することで API を実行します。
完了した時、何がしかで失敗した時にメッセージも表示されるようにしましょう。

```js: force-app/main/default/lwc/csvUploadToEA/csvUploadToEA.js
import { LightningElement, track } from "lwc";
// 省略

export default class CsvUploadToEA extends LightningElement {
  // 省略

  async handleUpload() {
    try {
      // 省略

      // アップロード
      updateBody({ externalDataId });
      showToastMessage(
        this,
        "Upload Success",
        "アップロードに成功しました",
        "success"
      );
    } catch (e) {
      console.error("upload error", e);
      showToastMessage(
        this,
        "Upload Failed",
        "アップロードに失敗しました",
        "error"
      );
    }
  }
}
```

あと、`this` の変数であったり画面で入力した値がそのままだとよろしくないので全部初期化してお行儀良く処理を終了させて終わりです。

```js: force-app/main/default/lwc/csvUploadToEA/csvUploadToEA.js
import { LightningElement, track } from "lwc";
// 省略

export default class CsvUploadToEA extends LightningElement {
  // 省略

  clearComboBox() {
    this.template.querySelectorAll("lightning-combobox").forEach((each) => {
      each.value = "";
    });
  }

  clearInputArea() {
    this.template.querySelectorAll("lightning-input").forEach((each) => {
      each.value = "";
    });
  }

  clearInputData() {
    this.edgemartContainer = "";
    this.edgemartAlias = "";
    this.operation = "";
    this.file = null;
    this.fileName = "";
    this.metaDataJson = "";
    this.loadingBar = 0;
    this.isLoading = false;
  }

  initialize() {
    this.clearInputData();
    this.clearComboBox();
    this.clearInputArea();
    this.handleProgress();
  }

  async handleUpload() {
    try {
      // 省略
    } catch (e) {
      console.error("upload error", e);
      showToastMessage(
        this,
        "Upload Failed",
        "アップロードに失敗しました",
        "error"
      );
    } finally {
      this.initialize();
    }
  }
}
```

# さいごに

さいごまで読んでいただきありがとうございます。

書きながらちょっと長めのプログラムの説明をするの難しいなぁと感じていたのですが、読みやすさとかはいかができたでしょうか。

不明点やわかりにくいところなどあったら教えていただけると幸いです。

今回のサンプルは「GitHub」にもまとめているので併せてご確認ください。

間違いの指摘やリクエストなどありましたら加筆していきたので是非、ご意見をいただけたらと思います。

それではまた次の記事でお会いしましょう。
