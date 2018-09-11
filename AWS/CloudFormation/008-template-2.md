# テンプレート 2

前回説明したセクションは `AWSTemplateFormatVersion` と `Resources` の2つ。最低限、これだけあればテンプレートとして成立する2つだ。

ここでは、テンプレート（1）に引き続き、AWS CloudFormationで使用できるテンプレートの構文について説明する。CloudFormationのより詳しいテンプレートの書き方について求めているのであれば、この記事だ。

今回もこのS3バケットを作成するだけのテンプレートを使ってセクションの説明を行う。

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

## Description - スタックの説明

スタック（テンプレート）を説明したいのであれば **Description** セクションが利用できる。テンプレートに記述するかは任意だが、プログラムでのコメントのように、書いておいたほうが良いもののひとつではあるだろう。必ず `AWSTemplateFormatVersion` セクションの後に記述決まりがある。

```json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Description": "S3バケットを作成",
  "Resources": {
    "MyS3Bucket": {
      "Type": "AWS::S3::Bucket"
    }
  }
}
```

```yaml
AWSTemplateFormatVersion: 2010-09-09

Description: S3バケットを作成

Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket
```

## Parameters - スタックへの引数

関数で言う引数のようなもので、 **Parameters** を設定することでテンプレートの値を自由に変更できる。Parameters内でパラメーターの名前を設定し、その下で型（ **Type** ）やデフォルト値（ **Default** ）などを設定する。スタックの説明と同じように、ここにも **Default** がいる。必須のキーは `Type` のみ。（設定できるキーは多いため、公式ドキュメント参照のこと。）

```json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Parameters": {
    "BucketName": {
      "Type": "String",
      "Default": "param-my-s3-bucket",
      "Description": "It's a bucket name."
    }
  },
  "Resources": {
    "MyS3Bucket": {
      "Type": "AWS::S3::Bucket",
      "Properties": {
        "BucketName": {
          "Ref": "BucketName"
        }
      }
    }
  }
}
```

```yaml
AWSTemplateFormatVersion: 2010-09-09

Parameters:
  BucketName:
    Type: String
    Default: param-my-s3-bucket
    Description: It's a bucket name.

Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Ref BucketName
```

Parametersを設定していた場合、スタックの作成でパラメーターの値を設定できる。このテンプレートであればパラメーター `BucketName` があるので、Webでの作成ではこのパラメーターを入力するテキストボックスが表示されている。

![テンプレートの引数](/img/aws-cf-template-2-001.png "テンプレートの引数")

実際に使用するキーのほとんどは `Type` `Default` `Description` の3つ。バリデーションによって、 `AllowedValues` `MaxValue` `MinLength` などを増やそう。

- [Parameters - 公式ドキュメント](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html)

## Mappings - スタックでの定数


## 参考

- [テンプレートの分析 - AWS CloudFormation](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/template-anatomy.html)