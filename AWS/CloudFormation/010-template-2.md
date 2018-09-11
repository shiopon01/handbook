# テンプレート 2

前回説明したセクションは `AWSTemplateFormatVersion` と `Resources` の2つ。最低限、これだけあればテンプレートとして成立する2つだ。

ここでは、テンプレート（1）に引き続き、AWS CloudFormationで使用できるテンプレートの構文について説明する。CloudFormationのより詳しいテンプレートの書き方について求めているのであれば、この記事だ。 `Mappings` や `Conditions` についてさらに詳しい話は別の記事に書く。

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

## Description - 説明

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

## Parameters - 引数

プログラムの関数で言う引数のようなもので、 **Parameters** を設定することでテンプレートの値を自由に変更できる。Parameters内でパラメーターの名前を設定し、その下で型（ **Type** ）やデフォルト値（ **Default** ）などを設定する。スタックの説明と同じように、ここにも **Default** がいる。必須のキーは `Type` のみ。（設定できるキーは多いため、公式ドキュメント参照のこと。）

テンプレート内で、パラメーターの値は組み込み関数 `Ref` などで参照できる。

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

- [Parameters - AWS CloudFormation](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html)

## Mappings - マップの定数

**Mappings** セクションで指定できるマップは、プログラムで言うところのマップ型の定数のようなものだ。Mapping内ではまず作成したいマッピングの名前を宣言し、その下には値と紐付くキーを設定していく。

例えば、 `RegionMap` というマッピングを宣言し、マッピング以下のキーでは各リージョンの識別子を設定する。キー以下の値も名前と値のペアとすることで、1つのキーに対して複数の値を設定することができる。

習慣として `{"マッピング名": {"キー": {"名前": "値"} } }` のような、3つの値を指定して値を取り出す構造が一般的だ。

テンプレート内でマッピングの値は組み込み関数 `FindMap` を使って参照できる。

```json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Mappings" : {
    "RegionMap" : {
      "us-east-1" : { "S3" : "us-east-1-bucket"},
      "us-west-1" : { "S3" : "us-west-1-bucket"}
    }
  }
  "Resources": {
    "MyS3Bucket": {
      "Type": "AWS::S3::Bucket",
      "Properties": {
        "BucketName": {
          "Fn::FindInMap": ["RegionMap", "us-east-1", "S3"]
        }
      }
    }
  }
}
```

```yaml
AWSTemplateFormatVersion: 2010-09-09

Mappings:
  RegionMap:
    us-east-1:
      S3: us-east-1-bucket
    us-west-1:
      S3: us-west-1-bucket

Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !FindInMap ["RegionMap", "us-east-1", "S3"]
```

## Conditions - 条件分岐


## Outputs - 出力

## 参考

- [テンプレートの分析 - AWS CloudFormation](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/template-anatomy.html)