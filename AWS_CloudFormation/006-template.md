# テンプレート

AWS CloudFormationからAWSリソースを作成しようと考えているなら、まずJSONかYAML形式のテンプレートファイルを用意しなければならない。（なぜなら、スタックはテンプレートが無いと作成できないからだ。）

これは、S3バケットを作成するための最小の簡単なテンプレートだ。

```json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Resources": {
    "MyS3Bucket": {
      "Type": "AWS::S3::Bucket",
      "Properties": {
        "BucketName": "my-s3-bucket-1234567890"
      }
    }
  }
}
```

```yaml
AWSTemplateFormatVersion: 2010-09-09

Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: my-s3-bucket-1234567890
```

テンプレートファイルはいくつかのセクションによって成立している。

## AWSTemplateFormatVersion - テンプレートのバージョン

まず、 初めの `AWSTemplateFormatVersion` 。これはAWSがこのテキストファイルをパースする際に用いる情報で記述は任意であるが、習慣としてふつうは記述される。（現時点では `2010-09-09` 以外存在していないので、実質決め打ち状態。）

## Resources - 作成するAWSリソース

次に、 `Resources` 。ここには作成したいAWSリソースをオブジェクトの形式（key-value）で宣言していく。AWSのリソースのキー、今回なら `MyS3Bucket` は、重複しないのであれば基本は自由に決めて構わない。プログラミングの変数のようなもので、何のリソースか把握できるような説明的な名前が良いだろう。

その下にある `Type` には、この宣言で作成するAWSリソースのプロパティタイプを記述する。S3のバケットなら `AWS::S3::Bucket` だし、DynamoDBのテーブルなら `AWS::DynamoDB::Table` と指定する。ここでプロパティタイプに指定する文字列はAWSのドキュメントに全て記載されているので、他のプロパティタイプはドキュメントを参照してもらいたい。（大量にある。）

- [AWS リソースプロパティタイプ - AWS CloudFormation](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html)

プロパティタイプと一緒に設定する `Properties` の値もほとんどはドキュメントを見ながら設定する。この部分の値はプロパティタイプによってまるっきり違うためだ。次のリンクはプロパティタイプ `AWS::S3::Bucket` に設定できるプロパティのドキュメント。

- [AWS::S3::Bucket - AWS CloudFormation](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-s3-bucket.html)

今回は `Properties` 内で `BucketName` を明示的に設定しているが、ドキュメントを見ると、 `AWS::S3::Bucket` に必須のプロパティは無いらしい。このテンプレートから作成されるバケットの名前は `my-s3-bucket-1234567890` になるが、名前を指定子なかった場合は `#{スタック名}-#{リソース名}-#{乱数}` の名前のバケットが作成される。

さて、まずはこの程度の知識で良いので、実際にCloudFormationでS3バケットを作成してみよう。
