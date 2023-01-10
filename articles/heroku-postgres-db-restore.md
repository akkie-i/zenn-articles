---
title: "【Heroku】Postgresを別のPostgresにリストアする方法"
emoji: "🪣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["heroku", "postgres"]
published: false
---

# はじめに

Heroku で開発したアプリケーションで Postgres に対して、INSERT や DELETE も含め実行するクエリのパフォーマンスを計測するために元々の DB に影響しない同じ構成の DB 環境を用意したい。

今回はこんなシチュエーションの時に Heroku 上に別の DB を作成し、リストアする方法を記事にします。

Heroku は Dyno と呼ばれるアプリケーションの実行環境が存在していて、Postgres などの DB はアドオンとして Dyno にアタッチすることで追加可能です。

ひとつの Dyno 上に同じアドオンを複数アタッチすることも可能なので、以下の 2 パターンで Postgres をリストアする方法を確認していきます。

- 同じ Dyno の別の DB にデータをリストアする方法
- 別 Dyno の DB にデータをリストアする方法

# 同じ Dyno の別の Postgres にデータをリストアする方法

前提として事前に Dyno は作成してあり、アプリケーション名は `sample_app` で、Postgres も追加され適当なテーブルは存在するとします。

既にある Postgres を A としてさらに追加した Postgres を B とし A から B へリストアし DB 操作をして挙動を確認していきます。

Postgres のアドオンが追加されいる状態で再度アドオンを追加します。

![](https://storage.googleapis.com/zenn-user-upload/1474be91fa8d-20230110.png)

元々あった Postgres には `DATABASE_URL` の環境変数、追加した Postgres には `HEROKU_POSTGRESQL_IVORY_URL` の環境変数で追加されてることが確認できます。

![](https://storage.googleapis.com/zenn-user-upload/06fe70520092-20230110.png)

現状 DB の中身は A には以下キャプチャのデータが存在し、B はからの DB です。

![](https://storage.googleapis.com/zenn-user-upload/250ddc72892f-20230110.png)

リストアには `heroku pg` コマンドを使用し、以下のコマンドで A から B へデータをリストアします。

```bash
$ heroku pg:copy sample_app::DATABASE_URL HEROKU_POSTGRESQL_IVORY_URL -a sample_app
```

以下の確認が表示されますので、アプリケーション名を入力してエンター

```bash
▸    WARNING: Destructive action
 ▸    This command will remove all data from IVORY
 ▸    Data from DATABASE will then be transferred to IVORY
 ▸    To proceed, type sample_app or re-run this command
 ▸    with --confirm sample_app

> sample_app # Enterすると以下のコピー完了のログが表示
Starting copy of DATABASE to IVORY... done
Copying... done
```

以下実行コマンドの概要です。

- `heroku pg:copy` でデータをリストア
  - From To の順番でコピー元、コピー先を指定
    - From: 環境変数 `DATABASE_URL` の Postgres のデータを
    - To: 環境変数 `HEROKU_POSTGRESQL_IVORY_URL -a sample_app` へコピーする

実行結果として、DB がリストアされて、同じ内容のテーブルが作成されることが確認できます。

![](https://storage.googleapis.com/zenn-user-upload/7741ce017dff-20230110.png)

試しに B のデータを削除しても A のテーブルには影響はありません。

![](https://storage.googleapis.com/zenn-user-upload/c73dd5c555ae-20230110.png)

これで B を対象にクエリを発行すれば元の A の DB には影響を及ぼさずに色々検証ができます。

# 別 Dyno の DB にデータをリストアする方法

やり方としてはコマンドをちょっと変えるだけです。
既に作成している `sample_app` とは別に `cp_app` というアプリケーション名の Dyno を作成し `sample_app` から `cp_app` へデータをリストアする方法です。

実行コマンドは以下になります。

```bash
$ heroku pg:copy sample_app::DATABASE_URL DATABASE_URL -a cp_app
```

コマンドの構成は「同じ Dyno の別の Postgres にデータをリストアする方法」と同じですね。

違いは Dyno が同じ環境だと Postgres の環境変数が別々ですが、Dyno が別だと環境変数が同じになり `-a` でのアプリケーションの指定先が異なります。

ですが、基本はどこから(`sample_app::DATABASE_URL`)どこへ(`DATABASE_URL -a cp_app`)の構成でコマンドを構成すると覚えればスッと理解できるかと思います。

# 注意点

コマンド実行時は慎重に行ってください。
From To の指定を逆にしてしまったら元の DB のデータが空っぽになります。もし本番環境で行っていたのだとしたら大惨事です。

また有料プランの Postgres のアドオンの場合、追加すればもちろんお金がかかります。
追加する前の相談、検証が終わった後のアドオンの削除を忘れず行うようにしましょう。

# さいごに

さいごまで読んでいただきありがとうございます。

ローカル環境だけじゃわからないから、実際の稼働してる環境でパフォーマンスを計測したいというシチュエーションが発生したときに誰かの役立てば幸いです。

間違いの指摘やリクエストなどありましたら加筆していきたので是非、ご意見をいただけたらと思います。

それではまた次の記事でお会いしましょう。
