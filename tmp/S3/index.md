以下メモ


# Amazon Simple Storage Service

Amazon Simple Storage Serviceは、インターネット用のストレージサービス。ユーザーはいつでも、安全に、どこからでも、容量制限無くデータを保存することができる。

# ユースケース

- コンテンツ配信や補完サーバー
  - Web・画像・動画などのメディアコンテンツ
  - JavaScriptを活用した2Tier Webシステム
- ログ&データハブストレージ
  - ログや分析データ保管用ストレージ
  - データロード元
- バックアップやディザスタリカバリ
  - データバックアップストレージ
  - 拠点間レプリケーション
- データレイク
  - センサーから取得したデータ

# 特徴

- インターネット対応のストレージ
- 常にオンラインで、HTTPでアクセス可能
- 容量無制限 (1ファイル5GBまで)
- 高い耐久性
  - イレブンナインと呼ばれる、99.999999999%の耐久性
- 安価なストレージ
  - 容量単価: 月額1GB/3円
- スケーラブルで安定した性能
  - データ容量に依存しない性能

# 課金

- 容量単価: 月額1GB/3円
- S3の通信 (OUT): 月額1GB/4円

# 登場人物

## バケット

(バケットはあんまり沢山作るものではなく、アクセス制御したいから分けるというものではない。（アクセス制御するときはIAMでわりと細かく設定できる）)

## オブジェクト

## オブジェクトキー

S3内のオブジェクトにアクセスする時は、 `オブジェクトキー` と呼ばれる文字列を利用する。

```
http://[バケット名].s3.amazonaws.com/[オブジェクトキー]
```

オブジェクトキーにはバケットのディレクトリやファイル名が入るが、実際にはディレクトリなどとして処理されず、オブジェクトキーはただの識別子 (文字列) として処理される。間にスラッシュが挟まっていても実際にはなんの意味もない。


# ストレージクラス

Amazon S3には複数のストレージクラスが多数存在し、用途に応じて格納する場所を使い分けることができる。使い分けることで、保存のコストを抑えることが可能。

## STANDARD

デフォルトのストレージクラス。

## STANDARD-IA（標準低頻度アクセスストレージ）

スタンダードに比べ、格納コストが安価。いつでもアクセス可能だが、データの読み出し容量に対して課金される。

低頻度アクセスのデータに最適のクラスであり、いざという時は多少のお金を払っても欲しい場合用。

## Glacier (アーカイブ)

長期アーカイブサービスであり、最も低コストだがデータの取り出しにコストと時間を要する。呼び出してからアクセス可能になるまでの時間によって課金される。 (早くても数分、遅くて数時間)

時間がかかるので、データベースなどの呼び出しには向いていない。データを呼び出すタイミングが分かっている、調査データ、実験データなどに最適。

## ライフサイクル

S3に置いて何日放置されたらS3 IAに移動、さらに何日経ったらGlacierに移動、そこから何日経ったら削除というライフサイクルが設定できる。

# Data Consistencyモデル

Amazon S3はデータを複数の場所に複製することで高い可用性を実現している。そのため、データの更新・削除には `Eventual Consistency Readモデル` (結果整合性) が採用されている。

|オペレーション|Consistencyモデル|挙動|
|-|-|-|
|新規登録 (New PUTs)|Consistency Read|登録後、即時データが参照できる|
|更新 (Overwrite PUTs)|Eventual Consistency Read (結果整合性)|更新直後は更新前のデータが参照される可能性がある|
|削除 (DELETE)|Eventual Consistency Read (結果整合性)|削除直後は削除前のデータが参照される可能性がある|

同じオブジェクトへの複数同時書き込み制御のためのロック処理は行われず、タイムスタンプが更新される。ロック処理が行われないため、読み込みの待ち時間は小さくなる。


# アクセス制御

バケット、もしくはオブジェクトへの細かいアクセス権限の設定が可能。
アカウント同士のクロスアカウントアクセスの設定も行える。

## ユーザーポリシー

IAMポリシーやIAMロールを使用することで、IAMからバケットへのアクセス管理が可能。

## バケットポリシー

バケットポリシーを使用することで、S3バケット毎にアクセス権限を設定できる。任意のIPアドレスからのリクエストのみ許可など。

[アクセス権限] タブから [バケットポリシー] を選択することで自由に記述可能。
以下サンプル。

```
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Sid":"AddPerm",
      "Effect":"Allow",
      "Principal": "*",
      "Action":["s3:GetObject"],
      "Resource":["arn:aws:s3:::examplebucket/*"]
    }
  ]
}
```

## ACL (アクセスコントロールリスト)

各バケットおよびオブジェクトへのアクセス権限を設定できる。 (ACLには、バケット単位のACLと、オブジェクト単位のACLが存在する)

バケットACLは、バケット内のオブジェクト全てに影響を与える。オブジェクトに個別でACLが設定されていた場合は、オブジェクトACLが優先される。また、ユーザーポリシーやバケットポリシーが設定されている場合は、こちらが優先される。
ACLはバケットやオブジェクトに対して100個まで指定可能。

バケットのアクセス権限を設定するには、[アクセス権限] タブから [アクセスコントロールリスト] を選択すればよい。
同様に、オブジェクトのアクセス権限を設定したい場合は、そのファイルを選択した時に表示される [アクセス権限] の項目を選択する。


# S3の機能

## イベント通知

バケットにてイベントが発生した際に、Amazon SNS、Amazon SQS、Amazon Lambdaに対して通知を送信することが可能。これにより、シームレスなシステム連携を実現している。

## Pre-signed Object URL (署名付きURL)

Pre-signed URLを利用することで、セキュアにS3とデータをやり取りすることが可能になる。

AWS SDKで生成される署名されたURLを利用し、S3上のプライベートなオブジェクトに一定時間アクセスすることが可能。生成する際は、有効期限、対象バケットかオブジェクトの指定と、GET/PUTのどちらかを指定する必要がある。

## Webサイトホスティング

静的なWebサイトをS3のみでホスティングすることができる。
`Amazon Route 53` のAlias設定でドメイン名とS3のバケット名を紐付ければ、独自ドメインを指定することもできる。他にも、CORSの設定や、任意のドメインへのリダイレクトなども可能。

WebサーバーとしてAmazon S3を利用する場合は、同時に `Amazon CloudFront` との利用が推奨されている。CloudFrontと利用することで、 `AWS WAF` の使用もできるようになる。

## インベント

S3の指定したバケットに入っているオブジェクトのリストを、CSVファイルで取得できる。
[管理] タブから [インベントリ] を選択することで取得可能。

## オブジェクトのバージョン管理

ユーザーやアプリケーションの誤操作による削除対策に有効。バケットに対して設定可能で、バージョン保管されているオブジェクトの任意のバージョンを参照することができる。ただし、バージョニングにより保管されているオブジェクト分も課金が発生。

もしオブジェクトを完全に削除してしまった場合、同じ名前のオブジェクトを置けば、過去のバージョンを指定して取り出すことができる。バケットを削除した場合は、古いバージョンも全て削除される。

## クロスリージョンレプリケーション

異なるリージョン間のS3バケットオブジェクトのレプリケーション (複製) を実施できる。
双方向レプリケーションなども可能だが、リージョン間データ転送費用が発生。

## 暗号化

サーバーやクライアントが、データを暗号化してセキュリティを高めることができる。

- サーバー再度暗号化
  - AWSのサーバーリソースを利用して格納データの暗号化処理を実施
  - 暗号化種別
     - SSE-S3: AWSが管理する鍵を利用して暗号化
     - SSE-KMS: `AWS Key Management Service` の鍵を利用して暗号化
     - SSE-C: ユーザーが提供した鍵を利用して暗号化
- クライアントサイド暗号化
  - 暗号化プロセスはユーザ管理
  - クライアント側で暗号化したデータをS3にアップロード
  - 暗号化種別
     - `AWS Key Management Service` で管理された鍵を利用して暗号化
     - クライアントが管理する鍵を利用して暗号化

## VPC Endpoint

S3と同一リージョンにあるVPC内のプライベートサブネット上で稼働するサービスから、NAT GatewayやNATインスタンスを経由せずに直接S3とセキュアな通信を行える。

## S3 support for IPv6

IPv4とIPv6の両方を、デュアルスタックエンドポイントにて無料でサポート。Webサイトホスティングでは利用できない。

|type|url|
|-|-|
|IPv4|http://[バケット名].s3.amazonaws.com/|
|IPv6|http://[バケット名].s3.<b>dualstack</b>.amazonaws.com/|


# 参考

- https://aws.amazon.com/jp/s3/
- http://docs.aws.amazon.com/ja_jp/AmazonS3/latest/dev/Welcome.html
