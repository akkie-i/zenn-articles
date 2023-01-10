---
title: "【Papertrail】ログのダウンロードと検索方法"
emoji: "🔖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["heroku", "papertrail"]
published: false
---

# はじめに

アプリケーションのロギングサービスである [papertrail](https://www.papertrail.com/)
特に Heroku で開発を行う人は必ずと言っていいほど使うのではないでしょうか。

papertrail は web の管理画面からでもログの検索を行えますが、検索期間として 1 週間前までしか遡れないです。
1 週間以上前を検索するとなるとログファイルをダウンロードして検索を行うなどのアプローチが必要になります。

今回はそんなシチュエーションを想定して特定の期間のログファイルをダウンロードしてきて検索を行う方法を記事にします。

## この記事でわかること

:::message

- CLI で papertrail のログ検索を行う方法

:::

## この記事を読む上での前提条件

:::message alert

- アカウントやアクセスキーの発行は Heroku で行ってる前提です。(キャプチャの UI などが Heroku ベース)

:::

# ファイルのダウンロード

papertrail は 1 時間単位でダウンロードファイルが保存されています。

[ドキュメント](https://www.papertrail.com/help/permanent-log-archives/#download-multiple-archives-using-date)を参考にしたところ、以下のパターンでのダウンロードの方法がサンプルでありましたのでこちらの方法を順番に確認してみようかと思います。

- 特定の日付の時間帯でファイルをダウンロードする
- 特定の日付の範囲で複数ファイルをダウンロードする
- 現在時刻の n 時間前のファイルをダウンロードする
- 現在時刻から n 時間前までの複数のファイルをダウンロード
- 特定の日付から n 時間前までの複数のファイルをダウンロード

## 特定の日付の時間帯でファイルをダウンロードする

curl を使用して、API Endpoint を叩くことで取得可能です。

以下は何年何月何日何時のログを指定して取得する方法です。(例では 2022-08-31 04:00:00 から 1 時間分の取得となる)

```bash
$ curl --progress-bar --no-include -o 2022-08-31-04.tsv.gz -f -L -H \
"X-papertrail-Token: PAPERTRAIL_TOKEN" \
https://papertrailapp.com/api/v1/archives/2022-08-30-04/download
```

`PAPERTRAIL_TOKEN` は Addon 追加時に Heroku の環境変数に追加されますのでそちらから確認可能です。

![](https://storage.googleapis.com/zenn-user-upload/4d5ec1d6d8f3-20230110.png)

その他の curl オプションも確認します。(事細かに知りたい人は `curl --help all` を実行)

- `--progress-bar`
  - レスポンスまでのプログレスバーを表示する
- `--no-include`
  - response header を含まないようにする
- `-o`
  - `<ファイル名> <URL>` の形式で response のアウトプット先を指定
  - 実行コマンドでは API Endpoint を `2022-08-31-04.tsv.gz` でダウンロードする記述になっている
- `-f`
  - 200 レスポンスが返らなかったらメッセージを出す
- `-L`
  - リダイレクトする
- `-H`
  - リクエストヘッダーを設定する
  - 上の `-L` と組み合わせてヘッダーの認証情報でリダイレクト(認証処理)を実行
  - `-fLH` でまとめると楽

以降で複数パターンでのファイルダウンロードの方法を試しますが、やってることは curl のオプションや CLI を組み合わせて `https://papertrailapp.com/api/v1/archives/yyyy-mm-dd-hh/download` のエンドポイントを動的に作っているだけです。

なので、ここを基本として押さえておけばこちらの記事以外のコマンドに限らず色々カスタマイズが可能です。

## 特定の日付の範囲で複数ファイルをダウンロードする

以下のコマンドで 2022-10-01 00:00:00 から 2022-10-01 23:00:00 の範囲のファイルをダウンロードします。

```bash
$ curl -sH 'X-Papertrail-Token: PAPERTRAIL_TOKEN' \
https://papertrailapp.com/api/v1/archives.json |
grep -o '"filename":"[^"]*"' |
egrep -o '[0-9-]+' |
awk '$0 >= "2022-10-01" && $0 < "2022-10-02" {
	print "output " $0 ".tsv.gz"
	print "url https://papertrailapp.com/api/v1/archives/" $0 "/download"
}' |
curl --progress-bar -fLH 'X-Papertrail-Token: PAPERTRAIL_TOKEN' -K-
```

未出のオプション、CLI などについて確認します。

- `|`
  - 左辺の処理結果を右辺の引数として渡す
- `-s`
  - サイレントモード。curl の進捗を何も表示しない
- `grep -o`
  - 空白でない行をに絞る
- `egrep`
  - grep の拡張 `grep -E` と同じ
  - `grep -o`
    - `grep -o` と同義
- `awk`
  - 区切り文字ごとに処理する
- `-K`
  - パイプやファイルの構成を読み取ります
  - `-K-` とすることでパイプからの URL に対して処理を実行

上記を元にコマンドの解説をします。

まず、curl で `https://papertrailapp.com/api/v1/archives.json` の内容を読み取ります。

この URL の json ファイルはログのファイル情報が一元管理されたファイルです。

実際は以下のような構成になってます。

```json
[
  {
    "start":"2022-10-18T02:00:00Z",
    "end":"2022-10-18T02:59:59Z",
    "start_formatted":"Tuesday, Oct 18 at  2:00 AM UTC",
    "duration_formatted":"1 hour",
    "filename":"2022-10-18-02.tsv.gz",
    "filesize":"7259",
    "_links":{
      "download":{
        "href":"https://papertrailapp.com/api/v1/archives/2022-10-18-02/download"
      }
    }
  },
	...
]
```

次に `grep -o '"filename":"[^"]*"' |` で json 内の `filename` プロパティの情報のみを抽出してパイプします。

次に `egrep -o '[0-9-]+' |` で日付情報のみを抽出してパイプします。

`awk` 以降の処理で取得した全日付情報から指定した範囲内の時のみの制御を入れ、一致した値の URL を組み立てます。

`$0` はパイプやファイルから渡された、文字列全てという意味になります。

条件に一致した URL に対して `-K-` でダウンロードを実行する

このような流れになってます。

## 現在時刻の n 時間前のファイルをダウンロードする

以下のコマンドで現在時刻から 12 時間前のファイルをダウンロードします。

OS が Mac か Linux かで差異があるので両者乗せときます

```bash
$ curl -silent --no-include --progress-bar -o　`date -u -v-12H '+%Y-%m-%d-%H'`.tsv.gz \
-fLH "X-Papertrail-Token: PAPERTRAIL_TOKEN" \
https://papertrailapp.com/api/v1/archives/`date -u -v-12H '+%Y-%m-%d-%H'`/download
```

```bash
$ curl -silent --no-include --progress-bar　-o `date -u --date='12 hours ago' +%Y-%m-%d-%H`.tsv.gz \
-fLH "X-Papertrail-Token: PAPERTRAIL_TOKEN" \
https://papertrailapp.com/api/v1/archives/`date -u --date='12 hours ago' +%Y-%m-%d-%H`/download
```

`date` コマンドに差異があるだけですね。

試しに `date -u -v-12H '+%Y-%m-%d-%H'` の部分だけをターミナルで実行ください。

現在時刻から 12 時間前がの時間が `yyyy-mm-dd-hh` 形式で出力されることが確認できます。

この時間を API Endpoint の期間指定の部分で動的に設定するやり方ですね。

※ `-u` で UTC ベースになってますので日本時間で考えたい場合はこちらをなくすか `-j` に変更ください。

## 現在時刻から n 時間前までの複数のファイルをダウンロード

以下のコマンドは現在時刻の 1 時間前から 12 時間(12 ファイル)分ダウンロードします。

こちらも OS が Mac か Linux かで差異があるので両者乗せときます

```bash
$ seq 1 12 |
xargs -I {} date -u -v-{}H +%Y-%m-%d-%H　|
xargs -I {} curl --progress-bar --no-include -o {}.tsv.gz \
-fLH "X-Papertrail-Token: PAPERTRAIL_TOKEN" https://papertrailapp.com/api/v1/archives/{}/download
```

```bash
$ seq 1 12 |
xargs -I {} date -u --date='{} hours ago' +%Y-%m-%d-%H |
xargs -I {} curl --progress-bar --no-include -o {}.tsv.gz \
-fLH "X-Papertrail-Token: PAPERTRAIL_TOKEN" https://papertrailapp.com/api/v1/archives/{}/download
```

未出の CLI を確認します。

- `seq`
  - 開始と終了の数値を指定して連続した Sequence を生成します
  - `seq 1 12` の場合 1-12 の値を保持する 12 個の Sequence になる
- `xargs`
  - パイプやファイルの構成を読み取ります引数として受け取ることが可能です
  - `-I {}`
    - 受け取った引数を `{}` で受け取る

上記を踏まえ以下のフローを実施しています。

1. 開始値 1 の sequence を実行
2. `xargs -I {}` シーケンスの番号を受け取りその値を引いた日付を生成
3. `xargs -I {}` で生成した日付を受け取り Api Endpoint のパラメータとして使用する
4. シーケンス番号に +1 して 1.に戻る。これを終了値 12 まで繰り返す

試しに curl の実行前までの記述をターミナルで実行するとイメージしやすいと思います。

```bash
seq 1 12 |
pipe> xargs -I {} date -u -v-{}H +%Y-%m-%d-%H
2022-10-19-03
2022-10-19-02
2022-10-19-01
2022-10-19-00
2022-10-18-23
2022-10-18-22
2022-10-18-21
2022-10-18-20
2022-10-18-19
2022-10-18-18
2022-10-18-17
2022-10-18-16
```

このように-1 時間ずつ引いた日付時刻が順番に出力されています。

この結果をもとに 12 回 API を実行しているということになります。

## 特定の日付から n 時間前までの複数のファイルをダウンロード

以下のコマンドで 10-01 00:00:00 の 1 時間前 09-30 23:00:00 から 12 時間分のファイルをダウンロードします。

OS が Mac か Linux かで差異があるので両者乗せときます

```bash
$ seq 1 12 |
xargs -I {} date -ur `date -ju 10010000 +%s` -v-{}H +%Y-%m-%d-%H |
xargs -I {} curl --progress-bar --no-include -o {}.tsv.gz \
-fLH "X-Papertrail-Token: PAPERTRAIL_TOKEN" https://papertrailapp.com/api/v1/archives/{}/download
```

```bash
$ seq 1 12 |
xargs -I {} date -u --date='2022-10-01 {} hours ago' +%Y-%m-%d-%H |
xargs -I {} curl --progress-bar --no-include -o {}.tsv.gz \
-fLH "X-Papertrail-Token: PAPERTRAIL_TOKEN" https://papertrailapp.com/api/v1/archives/{}/download
```

構成は上の「現在時刻から n 時間前までの複数のファイルをダウンロード」とほぼ同じで `date` の指定方法を変えてるだけなので説明は割愛します。

# ダウンロードファイル内の検索

ダウンロードしてきたファイルから検索する方法も簡単に確認します。

tsv ファイルの zip でダウンロードされるので解凍してそのまま Excel で開いて Excel 内の検索機能を使って該当のログを探すのが 1 つ目です。

もう一つは `gzip` コマンドを使ってそのまま検索する方法です。

以下に検索の一例を記載します。

```bash
# 指定日付に絞る
gzip -cd 2022-10-01-*

# 時間帯を絞る
gzip -cd 2022-10-01-* | grep 23:

# 文言で絞る
gzip -cd 2022-10-01-* | grep 23: | grep ERROR

# tsvの出力列を指定する
gzip -cd 2022-10-01-* | grep 23: | grep ERROR | cut -f5,9,10 | tr '\t' ' '
```

# さいごに

さいごまで読んでいただきありがとうございます。

エンジニアやってるとログとにらめっこして終わる 1 日なんてザラにあるのでログの検索などはスムーズにできるようになると良いですよね。

AWS での CloudWatch でログ検索の方法などクラウドごとのログ検索方法なども今後記事にしていければと思います。

間違いの指摘やリクエストなどありましたら加筆していきたので是非、ご意見をいただけたらと思います。

それではまた次の記事でお会いしましょう。
