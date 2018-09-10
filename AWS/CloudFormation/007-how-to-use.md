# AWS CloudFormationの使い方

AWS CloudFormationを利用するため、まず初めはスタックを作成するためのテンプレートを手に入れなけれる必要がある。テンプレートを入手する方法は大まかに3つだ。

1. AWSなどで公開されているサンプルテンプレートを使用する
2. クイックスタート（公式チュートリアル）で公開されているテンプレートを使用する
3. 自分でテンプレートを作成する
  - YAMLかJSON形式のテンプレートを作成する
  - CloudFormation Designerを利用する
  - CloudFormerを利用する

テンプレートを手に入れる方法は様々だが、まずはCloudFormationでスタックを作成することが目的のため、S3バケットを作成する最小のテンプレートを利用しよう。

```json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Resources": {
    "MyS3Bucket": {
      "Type": "AWS::S3::Bucket"
    }
  }
}
```

```yaml
AWSTemplateFormatVersion: 2010-09-09
Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket
```

`MyS3Bucket` 内に `Properties: {BucketName: ` を含めることでバケット名を指定することはできるのだが、ここでは指定せずAWSの自動生成に頼る。自動生成では `#{スタック名}-#{リソース名}-#{乱数}` の形式の名前が設定されるため重複することがないためだ。


