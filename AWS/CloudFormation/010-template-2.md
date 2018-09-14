# もっとテンプレート

前回説明したセクションは `AWSTemplateFormatVersion` と `Resources` の2つ。最低限、これだけあればテンプレートとして成立する2つだ。

ここでは、テンプレート（1）に引き続き、AWS CloudFormationで使用できるテンプレートの構文について説明する。CloudFormationのより詳しいテンプレートの書き方について求めているのであれば、この記事だ。 `Mappings` や `Conditions` についてさらに詳しい話は別の記事に書くかもしれない。

今回もこのS3バケットを作成するだけのテンプレートを基本として、各セクションの説明を行う。

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

スタック（テンプレート）を説明したいのであれば **Description** セクションが利用できる。テンプレートに記述するかは任意だが、プログラムでのコメントのように、書いておいたほうが良いもののひとつではあるだろう。必ず `AWSTemplateFormatVersion` セクションの後に記述するという決まりがある。

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

プログラムの関数で言う引数のようなもので、 **Parameters** を設定することでテンプレートの値を自由に変更できる。Parameters内で引数の名前を設定し、その下で型（ **Type** ）やデフォルト値（ **Default** ）などを設定する。スタックの説明と同じように、ここにも **Default** がいる。必須のキーは `Type` のみ。（設定できるキーは多いため、公式ドキュメント参照のこと。）

テンプレート内でParametersの値は組み込み関数 `Ref` などで参照できる。

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

習慣として `{"マッピング名": {"キー": {"名前": "値"} } }` のような、3つの値を指定して値を取り出す構造が一般的だ。キーにはよく、擬似パラメーター `AWS::Region` が利用される。（これによって、リージョンごとに使用する値を変更することができる。）

テンプレート内でマッピングの値は組み込み関数 `FindMap` を使って参照できる。

```json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Mappings" : {
    "RegionMap" : {
      "us-east-1" : { "S3" : "us-east-1-bucket"},
      "us-west-1" : { "S3" : "us-west-1-bucket"}
    }
  },
  "Resources": {
    "MyS3Bucket": {
      "Type": "AWS::S3::Bucket",
      "Properties": {
        "BucketName": {
          "Fn::FindInMap": ["RegionMap", {"Ref": "AWS::Region"}, "S3"]
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
      BucketName: !FindInMap ["RegionMap", !Ref AWS::Region, "S3"]
```

## Conditions - 条件分岐

CloudFormationでは、条件に応じてResourceセクションに記述されたAWSリソース「作る」か「作らない」かを選択できる。それはAWSリソースに `Condition` キーと、その値に **Conditions** セクションで定義した条件分岐を設定するだけで良い。AWSリソースは設定した条件分岐が `true` であれば作成され、 `false` であるなら作成されない。

CloudFormationには、使用する条件分岐は全てConditionsセクションに記述する決まりがあるため、AWSリソースの `Condition` キーを使用する場合はこのセクションも必要だ。条件分岐させようと思うとテンプレートや環境によっての変動値も必要になるので、自然とParametersセクションも利用することになるだろう。

さて、このConditionsセクション内では値にもとづいた条件分岐を行わなければならないので、 `And` や `Or` などの条件分岐を行うための関数が必要だ。そこでCloudFormationでは、専用の **条件関数** が用意されている。（ `Fn::If` に限り、Resourcesセクション、Outputsセクションでも利用できる。）

- Fn::If
- Fn::And
- Fn::Or
- Fn::Equals
- Fn::Not

ここでは例として、`Fn::Equals` のみ説明する。これは単純で、配列の1つめの値と2つめの値が等しければ `true` 、そうでなければ `false` を返す関数だ。

次のテンプレートは、パラメーターBucketCreateに渡された文字列が `true` であればResourceセクションのS3バケットが作成されるテンプレートだ。

```json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Parameters": {
    "BucketCreate": {
        "Type": "String",
        "Default": "true",
        "Description": "create bucket?"
      }
    },
  "Conditions": {
    "CreateBucketResource": {
      "Fn::Equals": [{"Ref": "BucketCreate"}, "true"]
    }
  },
  "Resources": {
    "MyS3Bucket": {
      "Type": "AWS::S3::Bucket",
      "Condition": "CreateBucketResource"
    }
  }
}
```

```yaml
AWSTemplateFormatVersion: 2010-09-09

Parameters:
  BucketCreate:
    Type: String
    Default: 'true'
    Description: create bucket?

Conditions:
  CreateBucketResource:
    !Equals [!Ref BucketCreate, 'true']

Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket
    Condition: CreateBucketResource
```

## Outputs - 出力

**Outputs** セクションでは、スタックが出力する値のことだ。バケットの名前、データベースのURL、その他AWSサービスのARNなど、CloudFormationで構築したインフラストラクチャを利用した次の処理に必要な値などを出力することができる。

次テンプレートはS3バケットを作成し、そのS3バケット名をOutputsセクションで出力するテンプレートだ。出力された値はAWSマネジメントコンソールの対象スタック選択後、出力タブで確認することが出来る。

![スタックの出力](/img/aws-cf-template-2-002.png "スタックの出力")

```yaml
AWSTemplateFormatVersion: 2010-09-09

Resources:
  MyS3Bucket:
    Type: 'AWS::S3::Bucket'

Outputs:
  S3BucketName:
    Description: Hello
    Value: !Ref MyS3Bucket
```

出力された値は他のスタックから `Fn::ImportValue` を使って参照することもできるが、詳しくは別の記事に記述する。ここでは、純粋なスタックからの出力のみを扱った。

## 参考

- [テンプレートの分析 - AWS CloudFormation](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/template-anatomy.html)