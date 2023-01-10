---
title: "【AWS】MFAユーザでaws cliを使用する"
emoji: "📲"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "mfa", "cli"]
published: false
---

# はじめに

aws cli を使おうとして時、IAM の権限設定は問題ないのになんでかアクセスが拒否される。

```bash
$ aws s3 ls --profile PROFILE_NAME

# 結果
An error occurred (AccessDenied) when calling the ListBuckets operation: Access Denied
```

そんな時は MFA の設定があるかどうかを確認してみましょう。

https://aws.amazon.com/jp/premiumsupport/knowledge-center/authenticate-mfa-cli/

# MFA の確認

MFA が設定してある場合、AWS のマネジメントコンソール画面から `IAM` を検索。
`ユーザー` → `認証情報` から `MFAデバイスの割り当て` で MFA の ARN が付与されていることが確認できます。

![](https://storage.googleapis.com/zenn-user-upload/22839672ed4c-20230110.png)

# 一時的なアクセス情報を発行する

MFA が設定されている場合は `aws sts` コマンドで有効期限付きの一時的なアクセス情報を発行する必要があります。

アクセス情報の発行には以下のコマンドを各オプションの説明に沿って適宜値を修正して実行します。

```bash
$ aws sts get-session-token \
--serial-number MFAに割り当てられたIAM \
--token-code MFAに割り当てられたIAM \
--profile AWSアカウントのプロフィール名
```

各オプションで設定する値の概要は以下です。

- `MFAに割り当てられたIAM`
  - 先ほどの「MFA の確認」の項にて添付したキャプチャの `MFAデバイスの割り当て` 記載の ARN 情報
- `MFA認証に使用してるアプリのコード`
  - 自身の MFA に使用している多要素認証用のアプリのコード。(google Authenticator など)
    ![](https://storage.googleapis.com/zenn-user-upload/24a8cfee53c9-20230110.jpeg)
- `AWSアカウントのプロフィール名`
  - `.aws/credentials` に記載されている以下の `xxx` の箇所
  ```
  [xxx]
  aws_access_key_id = *****
  aws_secret_access_key = *****
  ```
  - `xxx` の値が `default` になっている場合は省略しても大丈夫です。

:::message

- CLI コマンドで明示的に `--profile` で接続先を指定しない場合、`default` の認証情報を参照して CLI を実行します
- デフォルト接続はコマンド量が減っていい面もありますが、プロフィール指定する必要があるのに忘れてしまった場合、誤った接続先にコマンドを実行してしまうなどの事故も起こり得ます。
- 事故を起こさないように基本は `default` の設定は使わないで `--profile` の指定がない場合はエラーになるようにしておく方が安全です。
- プロフィール名 `$ aws configure --profile profile_name` で新規追加するか、`.aws/credentials` を直接編集します。

:::

実行結果として以下のレスポンスが返ってきます。

```bash
{
    "Credentials": {
        "AccessKeyId": "*****",
        "SecretAccessKey": "*****",
        "SessionToken": "*****",
        "Expiration": "2022-10-19T19:38:53+00:00"
    }
}
```

レスポンスの値を元に `.aws/credentials` の値を修正します。

```text: .aws/credentials
[xxx]
aws_access_key_id = AccessKeyId
aws_secret_access_key = SecretAccessKey
aws_session_token = SessionToken
```

動作確認として適当なコマンドを実行して問題なくレスポンスが返ればオッケーです。

```bash
$ aws s3 ls --profile {xxx}
```

# さいごに

さいごまで読んでいただきありがとうございます。

MFA アカウントで CLI を実行する場合はこんな感じでいくつかの工程が発生しますし、一時的な権限のため期限が切れれば再度同じことを行わなければ行けないので少々面倒ですね。

コンソール画面での作業用の MFA ユーザー、プログラムアクセス用の MFA なしのユーザーと分けて管理すると幾分楽ですが、何にせよ MFA なしのアカウントが 1 つはできてしまうことになるのでこの辺はどう運用していくかはプロジェクト、チーム単位での検討が必要そうですね。

間違いの指摘やリクエストなどありましたら加筆していきたので是非、ご意見をいただけたらと思います。

それではまた次の記事でお会いしましょう。
