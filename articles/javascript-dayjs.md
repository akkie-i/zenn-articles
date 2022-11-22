---
title: "【JavaScript】dayjsでの日付処理まとめ"
emoji: "⏰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javaScript", "npm", "dayjs"]
published: true
---

# はじめに

JavaScript を使うとみな一度はキレてるであろう [Date](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Date) の仕様。

もうライブラリなしには使いたくないなんて思うこともしばしばありますよね。
今回はそんな日付操作で人気のライブラリである [dayjs](https://day.js.org/en/) についてです。

よく使う部分をまとめた(つもり)なのでクイックリファレンス的な使い方してもらえたら幸いです。

## この記事でわかること

:::message

- dayjs を使って日付を操作する方法

:::

## この記事を読む上での前提条件

:::message alert

- コードをローカルで確認する場合、CDN を読み込むため VSCode の拡張機能 [Live Server](https://marketplace.visualstudio.com/items?itemName=ritwickdey.LiveServer)などを導入し、localhost を立てれるようにしてください。

:::

## この記事で取り扱わないこと

:::message alert

- CDN での web 画面確認が中心の記事なので node.js 実行環境での書き方などは取り扱いません。

:::

## GitHub

https://github.com/akkie-i/js-date/tree/main/src/dayjs

# dayjs vs moment.js

最初に少しだけ小話を。

dayjs と並んでよく使われているライブラリとして [moment.js](https://momentjs.com/) が挙げられます。

なんとなく dayjs が一強になっているのかなと思ってたのですが、実際は両者どのくらいの利用差があるのか [npx trends](https://npmtrends.com/dayjs-vs-moment) で確認してみました。

結果、どうやらまだ moment.js の方が利用率は高いようです。

しかし、dayjs も 6 年間という歴史の差をひっくり返す勢いで利用されておりトレンドとしてはやはり dayjs に向いてきているのかなと思いました。

どちらを使うかは機能面など状況によりけりだとは思いますが、両者を比較をすると以下の差があるようです。

- star の数の多さ：moment.js
- issue の数：dayjs
- 最終更新：dayjs
- ライブラリの軽さ：dayjs

個人的にはライブラリの軽さの面で dayjs 推しってところなのですが、いかんせん issue が多いのでエンバグの可能性を考慮すると moment.js を使っておいた方が安全という考えもできるかもしれないですね。

今回に限らずライブラリを導入するときは機能や軽さの他に、ライブラリの更新が頻繁にアップデートされているか、利用数とスター数、開発元の信頼性などを調べた上でちゃんと商用として長いスパンで使っていけそうなのかというのはチェックした上でインストールするという癖をつけておくと良いと思います。

# 導入方法

dayjs を使う方法は [npm](https://www.npmjs.com/package/dayjs) パッケージとしてインストール方法と CDN を使って利用する方法があります。
※ 今回はシンプルな HTML と JavaScript だけの構成にするために CDN を利用しています。

CDN を使う場合は主に以下の 3 サイトから対象のファイルを `script` タグで読み込んで使用します。使うものはどれでも良いと思います。

- [JSDELIVER](https://www.jsdelivr.com/package/npm/dayjs)
- [cloudflare](https://cdnjs.com/libraries/dayjs)
- [unpkg](https://unpkg.com/browse/dayjs@1.11.6/)

JSDELIVER と cloudflare はサイト上にパッケージの検索バーがありますのでそこで `dayjs` を検索し対象の `min.js` の URL をコピーし `src` で読み込みます。

JSDELIVER:
![](https://storage.googleapis.com/zenn-user-upload/ae7a99626c0f-20221121.png)

cloudflare:
![](https://storage.googleapis.com/zenn-user-upload/2f8fc4207911-20221121.png)

unpkg の場合は `https://unpkg.com/dayjs/` のように `バッケージ名/` の URL を入力するとブラウザ画面が見れますのでそちらから対象のファイルを選択し URL を取得します。

![](https://storage.googleapis.com/zenn-user-upload/88a4b51b13e5-20221121.png)

## dayjs の読み込み

**npm の場合**

```bash
$ npm i dayjs
```

```js: app.js
// ESMの場合
import dayjs from 'dayjs';
// CJSの場合
const dayjs = require("dayjs");

console.log(dayjs());
```

**CDN の場合**

```html: index.html
<body>
  <script src="https://cdn.jsdelivr.net/npm/dayjs@1/dayjs.min.js"></script>
  <script src="./app.js" type="module"></script>
</body>
```

```js: app.js
console.log(dayjs());
```

## locale の読み込み

**npm の場合**

```js: app.js
// ESMの場合
import dayjs from 'dayjs';
import ja from 'dayjs/locale/ja';

// CJSの場合
const dayjs = require("dayjs");
const ja = require("dayjs/locale/ja");

dayjs.locale(ja);
```

**CDN の場合**

```html: index.html
<body>
  <script src="https://cdn.jsdelivr.net/npm/dayjs@1/dayjs.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/dayjs@1.11.6/locale/ja.js"></script>
  <script src="./app.js" type="module"></script>
</body>
```

```js: app.js
dayjs.locale("ja");
```

## plugin の読み込み

**npm の場合**

```js: app.js
// ESMの場合
import dayjs from 'dayjs';
import isSameOrBefore from 'dayjs/plugin/isSameOrBefore';

// CJSの場合
const dayjs = require("dayjs");
const isSameOrBefore = require('dayjs/plugin/isSameOrBefore');

dayjs.extend(isSameOrBefore);
```

**CDN の場合**

```html: index.html
<body>
  <script src="https://cdn.jsdelivr.net/npm/dayjs@1/dayjs.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/dayjs@1.11.6/locale/ja.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/dayjs@1.11.6/plugin/isSameOrBefore.js"></script>
  <script src="./app.js" type="module"></script>
</body>
```

```js: app.js
dayjs.extend(dayjs_plugin_isSameOrBefore);
```

# 本日と指定日の取得

基本は `dayjs()` と宣言するだけでオブジェクトが生成されます。

引数がない場合は現在時刻をベースに、引数を渡す場合は日付形式の String や Date オブジェクトや UNIX タイムスタンプなど様々なフォーマットの引数でオブジェクトを生成できます。

どんな値を引数に渡したら parse できるのか詳細は [こちら](https://day.js.org/docs/en/parse/parse) をご確認ください。

生成したオブジェクトに対して `format()` を実行すると日付形式の文字列にフォーマットされます。(デフォルトは `YYYY-MM-DDTHH:mm:ss.SSS+09:00` )

以下サンプルです。

@[codepen](https://codepen.io/rmtgjoam/pen/dyKJayz)

::: message

dayjs でオブジェクトを生成すると date 型としては UTC ベースでの時間を保持していますが、format すると日本にいてることがバレて JST ベースの時間に変換されます。

この辺を考慮しないでデータベースに入れたりしてると思わぬところで 9 時間ずれの不具合が発生したりする可能性がありますのでご注意ください。

以下はオブジェクトをそのまま出力したデータと format で出力したデータの差異です。
`$d` は UTC、format で JST に変換されているのが確認できます。

```js
const dayjs = require("dayjs");
const d = dayjs();
console.log(d);
console.log(d.format());
```

```bash: 出力結果
M {
  '$L': 'en',
  '$d': 2022-11-21T09:31:51.558Z,
  '$x': {},
  '$y': 2022,
  '$M': 10,
  '$D': 21,
  '$W': 1,
  '$H': 18,
  '$m': 31,
  '$s': 51,
  '$ms': 558
}
2022-11-21T18:31:51+09:00
```

:::

# 曜日の取得

曜日の取得には dayjs のオブジェクトを生成し、`format` する引数に以下の値を渡すことで表現できます。
また、`locale` の設定により変換される値が変わります。([ドキュメント](https://day.js.org/docs/en/display/format))

**2022-01-01 00:00:00 の場合**

| 引数   | 結果                                         | 説明                                                    |
| ------ | -------------------------------------------- | ------------------------------------------------------- |
| `d`    | `6`                                          | 0-6 の整数に対応した曜日のキーを表示                    |
| `dd`   | locale en: `Sa`<br>locale ja: `土`           | Su-Sa または月-土のフォーマットで表示                   |
| `ddd`  | locale en: `Sat`<br>locale ja: `土`          | Sut-San または月-土のフォーマットで表示                 |
| `dddd` | locale en: `Saturday`<br>locale ja: `土曜日` | Sunday-Saturday または月曜日-土曜日のフォーマットで表示 |

実際の挙動のサンプルは以下を確認ください。

@[codepen](https://codepen.io/rmtgjoam/pen/mdKXmKG)

# 年月日時分ミリ秒ごとに取得

曜日と同じく `format` の以下の引数によって値を決定します。

**2022-01-01 00:00:00 の場合**

| 引数   | 結果                                 | 説明                                                         |
| ------ | ------------------------------------ | ------------------------------------------------------------ |
| `YY`   | `22`                                 | 年を 2 桁のフォーマットで表示                                |
| `YYYY` | `2022`                               | 年を 4 桁のフォーマットで表示                                |
| `M`    | `1`                                  | 月の 1-12 を 1-2 桁のフォーマットで表示                      |
| `MM`   | `01`                                 | 月の 01-12 を 2 桁に揃えたフォーマットで表示                 |
| `MMM`  | locale en: `Jan`<br>locale ja: `1月` | Jan-Dec または 1 月-12 月のフォーマットで表示                |
| `MMMM` | locale en: `Jan`<br>locale ja: `1月` | January-December または 1 月-12 月のフォーマットで表示       |
| `D`    | `1`                                  | 日の 1-31 を 1-2 桁のフォーマットで表示                      |
| `DD`   | `01`                                 | 日の 01-31 を 2 桁に揃えたフォーマットで表示                 |
| `H`    | `0`                                  | 時間を 24 時間表記の 0-24 の 1-2 桁のフォーマットで表示      |
| `HH`   | `00`                                 | 時間を 24 時間表記の 00-24 の 2 桁に揃えたフォーマットで表示 |
| `h`    | `0`                                  | 時間を 12 時間表記の 0-12 の 1-2 桁のフォーマットで表示      |
| `hh`   | `00`                                 | 時間を 12 時間表記の 00-12 の 2 桁に揃えたフォーマットで表示 |
| `m`    | `0`                                  | 分 の 0-59 を 1-2 桁のフォーマットで表示                     |
| `mm`   | `00`                                 | 分 の 00-59 を 2 桁に揃えたフォーマットで表示                |
| `s`    | `0`                                  | 秒 の 0-59 を 1-2 桁のフォーマットで表示                     |
| `ss`   | `00`                                 | 秒 の 00-59 を 2 桁に揃えたフォーマットで表示                |
| `SSS`  | `000`                                | ミリ秒を 000-999 のフォーマットで表示                        |

実際の挙動のサンプルは以下を確認ください。

@[codepen](https://codepen.io/rmtgjoam/pen/KKeQmBw)

# 日付の加算減算

日付を加算したり減算したりして、取得した結果に対してその日の始まりや終わりを指定する。
など実行できる関数を組み合わせることで明日や昨日など指定した日付を取得できます。

加算するには [add](https://day.js.org/docs/en/manipulate/add) を使用します。第一引数に加算する整数の値を渡し、第二引数に加算する単位を以下のフォーマットに沿って指定します。

減算するには [subtract](https://day.js.org/docs/en/manipulate/subtract) を使用します。引数の指定は `add` の使い方と同様です。

| 第二引数      | 省略形 | 説明                                                             |
| ------------- | ------ | ---------------------------------------------------------------- |
| `day`         | `d`    | 第一引数に指定した日数を足し引きする                             |
| `week`        | `w`    | 第一引数に指定した週を足し引きする                               |
| `month`       | `m`    | 第一引数に指定した月を足し引きする                               |
| `quarter`     | `q`    | 第一引数に指定した４半期を足し引きする(使用にはプラグインが必要) |
| `year`        | `y`    | 第一引数に指定した年を足し引きする                               |
| `hour`        | `h`    | 第一引数に指定した時間数を足し引きする                           |
| `minute`      | `m`    | 第一引数に指定した分数を足し引きする                             |
| `second`      | `s`    | 第一引数に指定した秒数を足し引きする                             |
| `millisecond` | `ms`   | 第一引数に指定したミリ秒数を足し引きする                         |

開始を表現するには [startOf](https://day.js.org/docs/en/manipulate/start-of) を使用します。引数を以下のフォーマットに沿って指定します。

終了の表現は [endOf](https://day.js.org/docs/en/manipulate/end-of) を使用します。引数の指定は `startOf` の使い方と同様です

| 引数      | 省略形 | 説明                                                                |
| --------- | ------ | ------------------------------------------------------------------- |
| `year`    | `y`    | 年末,年始の日付を表示する                                           |
| `quarter` | `Q`    | ４半期の開始日,終了日を表示する(使用にはプラグインが必要)           |
| `month`   | `M`    | 月の開始日,終了日を表示する                                         |
| `week`    | `w`    | 週の開始日,終了日を表示する                                         |
| `isoWeek` | -      | ISO に準拠した週の開始日,終了日を表示する(使用にはプラグインが必要) |
| `date`    | `D`    | 日の開始日,終了日を表示する                                         |
| `day`     | `d`    | 日の開始日,終了日を表示する                                         |
| `hour`    | `h`    | 時間の開始日,終了日を表示する                                       |
| `minute`  | `m`    | 分の開始日,終了日を表示する                                         |
| `second`  | `s`    | 秒の開始日,終了日を表示する                                         |

実際の挙動のサンプルは以下を確認ください。

@[codepen](https://codepen.io/rmtgjoam/pen/VwdQbBx)

# 日付を比較する

日付の比較には UNIX タイムスタンプに変換して整数比較するなどでも良いですが、dayjs では比較用のプライグインがいくつか提供されています。

日時 A が日時 B より同じか以前かを比較する `isSameOrBefore`、同じか以降かを比較する `isSameOrAfter`。
日時 A が日時 B より以前かを比較する `isBefore`、以降かを比較する `isAfter`などなど。

今回は `isSameOrBefore` で試してみます。

そのほかにも色々あるので [こちら](https://day.js.org/docs/en/plugin/plugin) から色々プラグインを探してみると良いと思います。

実際の挙動のサンプルは以下を確認ください。

@[codepen](https://codepen.io/rmtgjoam/pen/PoaQmBV)

# 日付の差分を求める

日付の差分を求めるには [diff](https://day.js.org/docs/en/display/difference) を使用します。

`{from}.diff({to}, {unit})` の形式で日付 A(`from`)と日付 B(`to`)の差分を `unit` の単位で取得します。

`unit` に指定する値は以下です。第三引数として `true` を渡すと小数点の範囲までの差分を算出します。

| unit          | 省略形 | 説明                     |
| ------------- | ------ | ------------------------ |
| `day`         | `d`    | 差分を日数で換算する     |
| `week`        | `w`    | 差分を週数で換算する     |
| `quarter`     | `Q`    | 差分を４半期数で換算する |
| `month`       | `M`    | 差分を月数で換算する     |
| `year`        | `y`    | 差分を年数で換算する     |
| `hour`        | `h`    | 差分を時間数で換算する   |
| `minute`      | `m`    | 差分を分数で換算する     |
| `second`      | `s`    | 差分を秒数で換算する     |
| `millisecond` | `ms`   | 差分をミリ秒数で換算する |

実際の挙動のサンプルは以下を確認ください。

@[codepen](https://codepen.io/rmtgjoam/pen/QWxQvVW)

# 書式のフォーマットを指定

書式のフォーマットは「曜日の取得」や「年月日時分ミリ秒ごとに取得」で扱ったフォーマットの単位を任意の区切り文字で繋げて `format` に指定することで自由に表現できます。

実際の挙動のサンプルは以下を確認ください。

@[codepen](https://codepen.io/rmtgjoam/pen/zYaRwJZ)

# UNIX タイムスタンプ

UNIX タイムスタンプには 10 桁と 13 桁で表示されるパターンがあります。
10 桁は UNIX エポック からの秒数までを算出した数値、13 桁はミリ秒までを算出した値になります。

そのため、10 から 13 桁のタイムスタンプに変換する場合は `*1000`、13 から 10 桁のタイムスタンプに変換する場合は `/1000`して相互で変換し合うことができます。

dayjs では 10 桁のタイムスタンプの取得には `unix` を使用し、13 桁のタイムスタンプの取得には `valueOf` を使用します。

逆にタイムスタンプから dayjs のオブジェクト生成するときは 13 桁のタイムスタンプを考慮した渡し方をしないとうまく変換できません。

なので 10 桁のタイムスタンプで dayjs のオブジェクトを生成するときは `*1000` した値を渡してあげる必要があります。

実際の挙動のサンプルは以下を確認ください。

@[codepen](https://codepen.io/rmtgjoam/pen/gOKvWdZ)

# さいごに

さいごまで読んでいただきありがとうございます。

今回のサンプルは「GitHub」にもまとめているので併せてご確認ください。

間違いの指摘やリクエストなどありましたら加筆していきたので是非、ご意見をいただけたらと思います。

それではまた次の記事でお会いしましょう。
