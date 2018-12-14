以下メモ


# Amazon Simple Notification Service (SNS)

マルチプロトコルに対応したフルマネージド型のPub/Sub メッセージング/モバイル通知サービス。

# ユースケース

- 何かを知らせるため、メールや通知を送りたい
- iOSやAndroudなどのモバイル端末にプッシュ通知を送りたい
- SQSと連携して、複数種類の処理を非同期かつ並列に実行したい

# 特徴

## Pub/Sub メッセージングモデル

非同期メッセージパラダイムの一つであり、出版側 (パブリッシャー) が特定の購読側 (サブスクライバー) を想定せずにメッセージを送信することができるメッセージングモデル。

トピックベースシステムにおいて、メッセージの送信は「`トピック`」と呼ばれる名前付き論理チャネルに対して行われる。これは購読者リストに当たるもので、購読側は、購読しているトピックに向けて送信された全メッセージを受信できる。同じトピックを購読している各購読者は全員同じメッセージを受け取ることになる。

購読するためにはまず、サブスクライバーがトピックに対してサブスクライブ (登録) を行い、メッセージを受け取れるようにしなければならない。これによって、1対多、多対1、および多対多の通信を実現する。

## Amazon SQSとの違い

Amazon SQS及びAmazon SNSは共にAWS内のメッセージングサービスだが、開発者にそれぞれ異なった利点を提供する。

Amazon SNSを利用すれば、「プッシュ」メカニズムを利用して、複数の受信者にタイミングが重要なメッセージを送信することができる。そのため、更新を定期的に確認や、メッセージを取得するために「ポーリング」する必要が無くなる。

Amazon SQSは、分散型アプリケーションが使用するメッセージキューサービスであり、「ポーリングモデル」を通じてメッセージを交換し、コンポーネントの送受信を切り離すために使用することができる。

## モバイル通知用プロバイダとしての利用

モバイル通知の場合、大量のデバイスへの通知の管理運用は、トークン作成やそれぞれ異なるモバイルプラットフォーム (Apple APNS, Google GCM, Amazon ADMなど) のせいで、大きな負担になりやすい。

そこで、プラットフォームごとのAPIを抽象化した中間プロバイダを利用することによって、どのプラットフォームへのアプリに対しても簡単に、信頼性のある通知の管理ができるようにしたい。

このプロバイダとして、`Amazon SNS モバイルプッシュ`を利用できる。 (利用には、プッシュ通知サービスに接続するための必須認証情報、プッシュ通知サービスから受け取ったデバイストークンまたは登録ID、およびプッシュ通知を受け取るモバイルアプリケーションが必要)

## TTL

TTL (Time To Live)をサポートし、メッセージ単位で生存期間を設定できる。
何らかの理由 (モバイルデバイスがオフになっているなど) で、指定したTTL内にメッセージが配信されなかった場合、そのメッセージは破棄され、以降その配信は試みられない。 (例えば、飛行機を降りた後に受け取る、既に終わったフラッシュセールのメッセージなどに利用できる)

# 課金

- 無料枠:
  - モバイルプッシュ通知: 1,000,000件
  - Email/Email-JSON: 1,000件
  - HTTP/HTTPS: 100,000件
  - Simple Queue Service (SQS): 無料

- リクエスト単価:
  - モバイルプッシュ通知: 1,000,000件 あたり 0.5USD
  - Email/Email-JSON: 1,000件 あたり 2USD
  - HTTP/HTTPS: 100,000件 あたり 0.6USD
  - Simple Queue Service (SQS): 無料

- データ転送 (OUT) :
  - 最初の1GB/月: 0USD/GB
  - 10TBまで/月: 0.140USD/GB


# 登場人物

## エンドポイント

メッセージを送信するためのAmazon SNS上での識別子。これには、Amazon SNSから通知メッセージを受信できるモバイルアプリ、Webサーバー、Eメールアドレス、またはAmazon SQSが当てはまる。

## トピック

複数のエンドポイントを束ねてグルーピング化した論理チャネル。
トピックにメッセージを発行することで、トピックに登録されている全てのサブスクライバーに対してメッセージを通知する。

このトピックに対して発行される`Topic ARN`を利用することで出版や登録が可能になる。

- 1トピックあたり最大1,000万サブスクリプション
- 3,000トピックまで作成可能

# はじめかた

1. (Pub) トピックの作成
2. (Sub) トピックへのSubscribe
3. (Pub) トピックへ向けてメッセージをPublish

まず、Amazon SNSコンソールでトピックを作成する。この時に`Topic ARN` (Amazon Resource Name) が生成される。同時にリトライポリシーなども設定可能。

ここで生成された `Topic ARN` に対して配信することで、そのトピックに登録されているエンドポイント全てにメッセージを配信できる。また、購読側がサブスクライブを行う場合にも、`Topic ARN`を利用する。

購読側からのトピックへのサブスクライブは、AWS SDKから行うことが出来る。Amazon SNSコンソールから購読側に、購読のリクエストを送ることも可能。

```ruby
# 購読側がトピックに登録する
resp = client.confirm_subscription({
  topic_arn: "topicARN", # required
  token: "token" # required
})
```

また、メッセージの送信も同じようにAWS SDKから行える。これもAmazon SNSコンソールから実行することができる。

```ruby
# 出版側がメッセージを出版する
resp = sns.publish(
  target_arn: "topicARN", # required
  message: message, # required
  message_structure: "json" # required
)
```


# 連携

AWSの様々なサービスと連携して通知可能。

- Amazon CloudWatch
  - Billing Alertの通知
- Amazon SES
  - Bounce/Complaintのフィードバック通知
- Amazon S3
  - ファイルがアップロードされた時に通知
- Amazon Elastic Transcoder
  - 動画変換処理完了/失敗時の通知

## Amazon SQSとの連携

Amazon SQSと連携することで、[Cloud Design Pattern: Fanoutパターン](http://aws.clouddesignpattern.org/index.php/CDP:Fanout%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3)を利用することができる。(Amazon SNSからAmazon SQSへの通知は無料)

Amazon SQSからAmazon SNSへのサブスクライブは、Amazon SQSコンソールの`キュー操作`から行える。

# Amazon SNSでできないこと

- 時間や条件を指定して自動的に配信する
  - 予約配信や条件設定による自動配信は行えない
  - Amazon SNSはリクエストを受けたらすぐに通知の配信を開始するため
- 配信した結果を観察する
  - 送信履歴からなにかを分析したり、ユーザの行動を解析することはできない
  - CloudWatchで、配信した数は確認できる

# まとめ

信頼性が高く、スケーラビリティに優れ、十分に管理されたプッシュメッセージングサービス。
簡単にコスト効率良く、マルチプロトコルで通知ができる。また、AWSの様々なサービスとの連携が可能。

# 参考

- https://aws.amazon.com/jp/sns/details/
- http://docs.aws.amazon.com/ja_jp/sns/latest/dg/welcome.html
- http://aws.clouddesignpattern.org/index.php/CDP:Fanout%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3





