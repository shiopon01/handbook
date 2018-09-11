# 擬似パラメーター

`Fn::Ref` で参照できるものにはParametersセクションで宣言されたパラメーターと、Resourcesセクションで宣言したAWSリソースが返す値の2つがあるが、パラメーターには自分で定義したもの以外にもAWSが用意した **擬似パラメーター** というものが存在する。これはコンポーネントの命名や環境によっての条件分岐に有用だ。

次の表は、擬似パラメーターの一覧だ。擬似パラメーターからは現在のアカウントの情報、現在のスタックの情報を取得することができる。

| 擬似パラメーター名    | 値の詳細                          | 値の例                                                  |
| --------------------- | --------------------------------- | ------------------------------------------------------- |
| AWS::AccountId        | AWSアカウントのID                 | 123456789012                                            |
| AWS::StackName        | スタックの名前                    | demo-stack                                              |
| AWS::StackId          | スタックのID                      | arn:aws:cloudformation:us-west-2:123456789012:stack/... |
| AWS::Region           | AWSリージョンの識別子             | us-west-2                                               |
| AWS::NoValue          | NoValueを指定したプロパティを削除 | 値を返さない                                            |
| AWS::NotificationARNs | スタックの通知先のARNのリスト     | [arn:aws:sns:us-east-1:123456789012:MyTopic]            |

例えば、テンプレート内で `Fn::Ref: AWS::AccountId` と指定することでアカウントのIDが返される。アカウントのIDはユニークなものであるため、ユニークでなければならないS3バケット名などに使用できる。

次のテンプレートはアカウントIDを含むS3バケットを作成するテンプレートだ。これで、 `mybucket-123456789012（AWSアカウントID）` という名前のS3バケットが作成される。

```json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Resources": {
    "MyS3Bucket": {
      "Type": "AWS::S3::Bucket",
      "Properties": {
        "BucketName": {
          "Fn::Join": [
            "",
            [
              "mybucket-",
              {"Ref": "AWS::AccountId"}
            ]
          ]
        }
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
      BucketName:
        Fn::Join:
          - ''
          - - 'mybucket-'
            - !Ref AWS::AccountId
```

## AWS::NoValue

**AWS::NoValue** は少し特殊な擬似パラメーターで、この値を設定したAWSリソースのプロパティーを削除することができる。

例えば次のテンプレートは、 `MyS3Bucket` にプロパティー `BucketName` を設定していながらも、 `BucketName` を削除してバケット名をデフォルトの名前にする例である。（実用的ではない。）

```json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Resources": {
    "MyS3Bucket": {
      "Type": "AWS::S3::Bucket",
      "Properties": {
        "BucketName": { "Ref": "AWS::NoValue" }
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
      BucketName: !Ref AWS::NoValue
```

一見役に立たないようにも見えるが、AWS::NoValueが輝くのは **Fn::If** と併用した時だ。 Fn::Ifは指定された条件が `true` の時は1つめの値を、 `false` の時は2つめの値を返す組み込み関数で、どちらかでNoValueを返すことで期待する動作を実現することが出来る。

次のテンプレートはこの **Fn::If** を併用し、ParametersからS3のバケット名が渡された場合はその名前で、渡されなかった（空文字列だった）時はデフォルトの名前を利用するテンプレートだ。おそらく、 **AWS::NoValue** を使うときははほとんどこの使い方だろう。

```json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Parameters": {
    "BucketName": {
      "Type": "String",
      "Description": "It's a bucket name."
    }
  },
  "Conditions": {
    "BucketNameIsBlank": {
      "Fn::Equals": [{ "Ref": "BucketName" }, ""]
    }
  },
  "Resources": {
    "MyS3Bucket": {
      "Type": "AWS::S3::Bucket",
      "Properties": {
        "BucketName": {
          "Fn::If": [
            "BucketNameIsBlank"
            { "Ref": "AWS::NoValue" },
            { "Ref": "BucketName" }
          ]
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
    Description: It's a bucket name.

Conditions:
  BucketNameIsBlank:
    !Equals [!Ref BucketName, '']

Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !If
      - BucketNameIsBlank
      - !Ref AWS::NoValue
      - !Ref BucketName
```

## 参考

- [擬似パラメーター参照 - AWS CloudFormation](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/pseudo-parameter-reference.html)