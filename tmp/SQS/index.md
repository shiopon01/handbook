以下メモ


# sqs

```
$ aws sqs receive-message --queue-url https://[region].queue.amazonaws.com/[path]
{
  "Messages": [
    {
      "Body": "HELLO!!",
      "ReceiptHandle": "AQEBNWPzcAywdJmUa...",
      "MD5OfBody": "fa6f5dcca46ab9c3c...",
      "MessageId": "4d1fcb5f-0dfd-4db0-8228-aea9441152d2"
    }
  ]
}
```




# Amazon Simple Queue Service (SQS)

マイクロサービス、分散システム、およびサーバーレスアプリケーション用の完全マネージド型メッセージキュー。プロデューサが大量のメッセージをキューに蓄えておき、コンシューマは好きな時にそれを取り出すことができる。
メッセージキューはソフトウェアの世界では古くからある概念であり、[MQ](https://ja.wikipedia.org/wiki/%E3%83%A1%E3%83%83%E3%82%BB%E3%83%BC%E3%82%B8%E3%82%AD%E3%83%A5%E3%83%BC)と略されることも。

Amazon SQSはPull型のMQサービスなので、送信者からSQSに送信されたメッセージを受信者は問い合わせをすることで取得できる。

https://aws.amazon.com/jp/sqs/

# 使用例

> A「今、データベースに大量のデータを登録したい。」
> B「今は負荷が高いので困る。一定間隔で少しずつ頼む。」

Aamazon SQSなら、Aがメッセージキューにデータを送信しておいて、Bが後々データベースの負荷を考慮しながらデータを取り出し、登録していくことが可能。

> A「AWS SDKを使ってSQSにデータを投入！」
> B「データベースの負荷を見ながらデータベースに登録していく！」

# 特徴

- 高い信頼性: 複数のサーバ/データセンタにメッセージを保持
- スケーラブル: 多数の送信者/受信者に対応
- 高スループット: メッセージが増加しても高スループット
- 低コスト: 毎月の無料枠 + 使った分だけの従量課金

## 課金

- 毎月の無料枠 + 使った分だけの従量課金
- 毎月100万キューイングリクエストまで無料
- SQSリクエスト
  - 100万件につき0.476 USD
  - 複数メッセージを1リクエストとして送信することも可能
- データ転送（送信（アウト））
  - 最初の1GB/月: 0 USD
  - 350TBまで、1GBあたり/月: 0.120 USD
- ただし、同一リージョン内のSQSとEC2インスタンスのデータ転送は無料

## 上限

- メッセージ保持期限
  - 削除されないメッセージはデフォルトで4日保持（60秒～14日の間で設定）
- メッセージの送信/受信は1度に10件まで
- インフライトメッセージ
  - 1つのキューごとに最大120,000インフライトメッセージ
  - 超えると`OverLimit`エラー
- メッセージの最大サイズは256KB
  - 画像や動画などの大容量データには適していない
  - S3に配置し、メッセージ内にパスを記述など

## 注意点

順序はベストエフォートになる。Aamazon SQSは出来る限り順序を維持しようとするが、保証はされていない。そのため利用者は、タイムスタンプやシーケンス番号をメッセージに付与する必要がある。

また、同じメッセージを複数回受信してしまう可能性も存在する。本来は可視性タイムアウトによって複数回受信することは無いはずだが、メッセージ削除時にサーバが一時故障中の場合などにその可能性が生まれる。そのため利用者は、同じメッセージを複数回処理した場合に悪影響を出さない設計を行う必要もある。


# 機能

## 知っておいたほうが良い識別子

- Queue URL
  - キューを作成する際に払い出されるURL
  - https://リージョン.queue.amazonaws.com/アカウントID/キュー名
- Message ID
  - システムで割り当てられたID
- Receipt Handle
  - メッセージと一緒に受信できる
  - メッセージをキューから削除や、可視性タイムアウト期限の変更に使用
  - 一緒に受信されたメッセージのみ、削除や期限の変更が可能
  - 同じメッセージでも受信する度に変更されるので、最新のものを使うこと

## 可視性タイムアウト（Visibility Timeout）

誰かがメッセージを受信した時、一定時間の間は他のコンシューマにそのメッセージを受信させない仕組み。
これによって、複数のコンシューマからの問い合わせで重複したメッセージを送信してしまうことがなくなる。（注意点にあったように、同じメッセージを複数回受信してしまう可能性は存在する）

コンシューマがキューからメッセージを受信してもそのメッセージはキューに残ったままであり、Amazon SQSがメッセージを自動的に削除することはない。接続の問題やコンシューマアプリケーションの問題などが原因で、コンシューマが実際にメッセージを受信できた保証がないためだ。
したがって、コンシューマはメッセージを受信して処理した後、`DeleteMessage`アクションを利用してキューからメッセージを削除する必要がある。

可視性タイムアウトはAmazon SQSがメッセージを返した時点から始まる。  タイムアウト時間（デフォルト30秒、変更可）の間に、コンシューマは`DeleteMessage`アクションを使用してメッセージを処理して削除しなければならない。
タイムアウトの期限が切れると、そのメッセージは他のコンシューマからも見えるようになり、再度受信される。一度だけ受信したい場合は、コンシューマは可視性タイムアウトの時間内にメッセージを削除する必要がある。

## 可視性タイムアウトの変更

例えば、メッセージを受信して処理を開始した時、キューの可視性タイムアウトが十分でない場合がある。その時、`ChangeMessageVisibility`アクションを使用して新しいタイムアウト値を設定することで短縮・拡張することができる。

例えば、タイムアウトが60秒であり、メッセージを受信してから15秒経過したときに、`VisibilityTimeout`を10秒に設定した`ChangeMessageVisibility`コールを送信した時、この10秒は`ChangeMessageVisibility`コールを行った時点からカウントが開始される。
したがって、タイムアウトを変更してから10秒（合計25秒）経過した後に可視性タイムアウトを変更しようとしたり、そのメッセージを削除しようとすると、エラーが発生する可能性がある。

## インフライトメッセージ（In Flight）

可視性タイムアウト期間内（削除可能）であり、まだ削除されていないメッセージをインフライトメッセージと呼ぶ。

スタンダードキューの場合、キューあたり最大120,000のインフライトメッセージが存在できる。この制限に達した場合、Amazon SQSは`OverLimit`エラーメッセージを返す。制限に到達しないようにするため、処理されたメッセージはキューから削除する必要がある。メッセージの処理に使用するキューの数を増やすこともできる。

FIFOキューの場合、キューあたり最大20,000のインフライトメッセージが存在できる。この制限に達した場合、Amazon SQSはエラーメッセージを返さない。

# 例

## メッセージを送信

```
$ aws sqs send-message\
> --queue-url https://[region].queue.amazonaws.com/[path]\
> --message-body "HELLO!!"
{
  "MD5OfMessageBody": "51b0a3256d59467f9730...",
  "MessageId": "47b28fdc-4352-4bc1-ab84-3ae745ffaa3c"
}
```

## メッセージを受信

```
$ aws sqs receive-message\
> --queue-url https://[region].queue.amazonaws.com/[path]

{
  "Messages": [
    {
      "Body": "HELLO!!",
      "ReceiptHandle": "AQEBNWPzcAywdJmUa...",
      "MD5OfBody": "fa6f5dcca46ab9c3c...",
      "MessageId": "4d1fcb5f-0dfd-4db0-8228-aea9441152d2"
    }
  ]
}
```

`max-number-of-messages` オプション (`--max-number-of-messages 10`など) を利用することで、1度に最大10件まで取得可能。

## メッセージを削除

可視性タイムアウト期間内に、メッセージと一緒に受信できる `ReceiptHandle` を送信することで削除できる。

```
$ aws sqs delete-message\
> --queue-url https://[region].queue.amazonaws.com/[path]\
> --receipt-handle AQEB4Hvser2zK9len0oBUKN...
```

# 実践的な機能

- 時間を置いてからメッセージを見せたい
  - Delay Queue と Message Times
- 何回受信されてもキューに残り続けるメッセージをなんとかしたい
  - Dead Letter Queue
- セキュリティ
  - AWS IAM連携
- モニタリング
  - AWS CloudWatchでの監視

## 時間を置いてからメッセージを見せたい

- Delay Queue
  - キューに送られた新しいメッセージを、ある一定秒の間見えなくする
  - 0～900秒でキューに設定できる
  - 設定すると、そのキューに送信されるメッセージ全てに適用

- Message Timers
  - 個々のメッセージに対して、送信されてから見えるようになるまでの時間を設定
  - 0～900秒でメッセージに設定できる
  - キューに設定されているDelay Queueを上書きできる

## 何回受信されてもキューに残り続けるメッセージをなんとかしたい

- Dead Letter Queue
  - メッセージを指定回数受信後に、自動で別のキュー（Dead Letter Queue）に移動する機能
  - デフォルトは無効で、1～1000回で設定

## セキュリティ

AWS IAMと連携することで、特定のリソースやアクセス元のみアクセス許可/拒否や、特定の時間帯のみアクセス許可などの設定を行える。

## モニタリング

AWS CloudWatchを利用して、キューに追加されたメッセージ数やその合計サイズ、コールによって返されたメッセージ数などの情報を取得できる。

# まとめ

Amazon SQSは信頼性が高く、スケーラビリティに優れ、十分に管理されたメッセージキューサービス。簡単でコスト効率も良く、疎結合で柔軟なシステムを構築することができる。

# 参考

- [Amazon Simple Queue Service 開発者ガイド](http://docs.aws.amazon.com/ja_jp/AWSSimpleQueueService/latest/SQSDeveloperGuide/Welcome.html)