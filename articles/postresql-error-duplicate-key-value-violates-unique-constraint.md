---
title: "【PostgreSQL】シーケンスの自動採番が重複してエラーが出る時の確認観点"
emoji: "📀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["postgreSQL"]
published: false
---

# はじめに

PostgreSQL を使っていてレコードを新規登録しようとしたら以下のエラーが発生しました。

```
ERROR:  duplicate key value violates unique constraint "テーブル名_pkey"
```

今回はこちらのエラーが発生した時に行なった対処方法を記事にします。

# エラー発生時の状況

少し昔のことなので事細かには覚えていないですが、CSV データを何回も入れ直したりとしている内にひょんな時に以下のテーブルのように id の最大値の間に欠番が存在していたことに気がつきました。

| id  | name   |
| --- | ------ |
| 1   | ichiro |
| 2   | jiro   |
| 3   | saburo |
| 5   | shiro  |

# エラー解消までの確認観点

紆余曲折色々調べた記憶がありますが、まとめると確認観点は以下 2 点でまとめられます。

1. テーブル id の最大値とシーケンスの最終番号で整合性が取れているか。
2. 調べてるシーケンスはあっているか。

冒頭のエラー文をググると概ね 1 に対しての対処法が上がってきます。
1 の対処法で解決したならばこの記事は書かなかったのでしょうが、解決できなかったため本記事を執筆する経緯に至りました。

補足説明や事象の再現をしながらやったことをつらつら書いていこうと思います。

## シーケンスを理解する

PostgreSQL では主キーを省略してレコードを `INSERT` すると [シーケンス](https://www.postgresql.jp/document/14/html/functions-sequence.html) を用い、連番で id が採番されます。

シーケンスはテーブルを作成した時のテーブル名と主キーに選択したカラムを用い、`テーブル名_カラム名_seq` の名前で作成されます。

`users` テーブルの主キーを `id` にした場合ですと `users_id_seq` ですね。

ちなみにシーケンス一覧を確認したい場合は以下の SQL を実行すると確認できます。

```sql
SELECT
  c.relname
FROM
  pg_class c
LEFT join
  pg_user u
ON
  c.relowner = u.usesysid
WHERE
  c.relkind = 'S';
```

採番には `select nextval('シーケンス名')` を実行するとシーケンスを次の値に進め、その値を返し採番します。

`nextval` によって採番されたシーケンスの Latest の値は以下のコマンドで確認します。

```sql
SELECT * FROM シーケンス名
```

`last_value` というカラムにシーケンス番号が入ってますので、レコードが 1 件だった場合、`last_value` の値は `1` で、`select nextval('シーケンス名')` を実行すると `2` が返ってくるのが採番の状況が健康な状態です。

![](https://storage.googleapis.com/zenn-user-upload/7236aab7ac4c-20230111.png)

## テーブル id の最大値とシーケンスの最終番号で整合性が取れているか

シーケンスの概要を理解したところで、本題に戻ります。

「エラー発生時の状況」で書いたようなテーブル id の最大値とシーケンスの最終番号で整合性が取れていないテーブルの状況を作り、エラー再現、修正する流れを確認します。

まず適当なデータベースを作成し、以下のコマンドでシーケンスを作成します。

```sql
CREATE SEQUENCE users_id_seq
```

併せて `users` テーブルを作成し `id` を主キーでシーケンスに `users_id_seq` を指定し、あと `name` の String 型のカラムを作成します。

![](https://storage.googleapis.com/zenn-user-upload/3817fb61d0b6-20230111.png)

※シーケンスはテーブル作成時に自動で作成されますが、今回は明示的に作成してます。

`id` を指定しないデータを 3 件作成し、4 件目に `id = 5` のレコードを作成します。

![](https://storage.googleapis.com/zenn-user-upload/9a07c88b61ab-20230111.png)

この状態でまずはテーブル id の最大値を取得します。以下のコマンドを実行ください。

```sql
SELECT MAX(id) FROM users
```

`5` の値が取得できます。
次にシーケンスの Latest の値を取得します。以下のコマンドを実行ください。

```sql
SELECT * FROM users_id_seq
```

`last_value` が `3` で取得できます。

これで整合性が取れてない状況になりました。
id が 4 のレコードは挿入できますが、5 で id が重複して衝突します。

![](https://storage.googleapis.com/zenn-user-upload/d9a2a2f38d77-20230111.png)

冒頭のエラーになりました。

このズレを治してみます。

カレントのシーケンスが `4` なのを `5` にしてあげることで次の採番は `6` からになるのでズレを修正できますね。こちらを以下のコマンドで実行します。

```sql
SELECT setval('users_id_seq', (SELECT max(id) from users))
```

`setval` の結果が `5` になるので無事後続のレコードが挿入できて、`6` からの採番になります。

![](https://storage.googleapis.com/zenn-user-upload/01cd8efdeeca-20230111.png)

## 調べてるシーケンスはあっているか

同じエラーに直面して、ここまでの内容で無事解決した方はヨシ！ですが、当時の僕はここで終了ではなかったのです。

確かに、テーブル id の最大値とシーケンスの最終番号のズレがあることを確認できた。`setval` で採番も正常な状況に戻した。

でも、まだ `duplicate key value` のエラーが発生する、嫌になって一回レコードを全件削除し、 `setval` を `1` にして初期の状態に戻して新規レコードを登録してみたら採番が `1` からじゃない！なんやねん！

こんな状況になってました。

これ何が起きてたのかというとシーケンスがもう 1 つできてて 1 テーブル間で 2 つのシーケンスが共存してました。

詳細には以下キャプチャのようなカオスな状況に(いつの間にこうなってたのかはわからんです。)

![](https://storage.googleapis.com/zenn-user-upload/587d8b6c4287-20230111.jpg)

キャプチャの状態になっていたため、`users_id_seq` のシーケンス番号を修正したところで `users_id_seq2` が存在していたため、重複エラーにもかかるし初期状態にしてレコードを登録しても採番が初期化しなかった原因だったようです。

この状態になっていたら解決方法は Structure から id のシーケンス名を `users_id_seq` に戻せば直るのですがせっかくなので `users_id_seq2` の存在に気づかない状態という前提で事象を再現してみようかと思います。

冒頭と同じく 3 件のデータは id 入力なしで 4 件目のデータは id を `5` にし、重複エラーが発生する状況にします。

以下のコマンドを順番に実行し、新しいシーケンスの作成及びシーケンス番号を `3` に統一します。

```sql
CREATE SEQUENCE users_id_seq2;
SELECT setval('users_id_seq', 3);
SELECT setval('users_id_seq2', 3);
```

この状態で id の設定なしで 2 件データを登録を試みて、2 件目で重複エラーが発生することを確認します。(`users_id_seq2` のカレント値が `4` で止まってる状態)

![](https://storage.googleapis.com/zenn-user-upload/edfe712ea885-20230111.png)

ここで `users_id_seq` を戻しても `user_id_seq2` の値は `4` のままなのでまた重複エラーが発生しますし、データを全件削除して以下のコマンドでシーケンスを初期化しても採番は `5` から始まります。

```sql
SELECT setval('users_id_seq', 1, false)
```

![](https://storage.googleapis.com/zenn-user-upload/2a1ef2f5b67e-20230111.png)

原因がわかった段階ではこうなるのは当たり前って話なのですが、当時のシーケンスが複数になってるという状況に気付かなかった僕にとっては意味不明って嘆いていたのを覚えてます。

PS. DB クライアントの Structure って偉大ですよね。テーブル情報が見やすくなってたから違和感に気づけたけど SQL 直で叩いて確認してたら一生気づかなかった気がします。

## おまけ、考えるのがだるくなった方へ

同じ状況になった人がこの記事で解決できることを祈りますが、それでもダメだった。もうここに時間使ってられない。

そんな時はもう [TRUNCATE](https://www.postgresql.jp/document/14/html/sql-truncate.html) しましょう。

`RESTART IDENTITY` することでシーケンスも含め完全にテーブルが初期化されるので 1 からやり直しましょう。

```sql
# テーブルの初期化
TRUNCATE TABLE {tableName} RESTART IDENTITY;

# 外部キー制約のある場合
TRUNCATE TABLE {tableName} RESTART IDENTITY CASCADE;
```

# さいごに

さいごまで読んでいただきありがとうございます。

ちょっと前にハマってメモしていたものを発掘し、事象を再現しながら書いたのでまとめるというよりは状況を書くみたいな感じで読みづらいところがあるかもしれませんが誰かの役に立てば幸いです。

間違いの指摘やリクエストなどありましたら加筆していきたので是非、ご意見をいただけたらと思います。

それではまた次の記事でお会いしましょう。
