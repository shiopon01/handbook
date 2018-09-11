# Commentary of Intrinsic functions

CloudFormationでは、テンプレート内で使えるいくつかの組み込み関数が提供されている。これは例えば、定数のように事前に定義しておいた値を抜き出してきたり、値をBase64に変換したり、文字列を結合したり、別のスタックでエクスポートされた値を取得する場合などに利用される。

公式のドキュメントから引用すると組み込み関数はこれだけ存在する。その中で、条件関数以外を頻繁に利用する順にだいたいで並べるとこのような順番になると思う。1つのテンプレートで何度も利用するのは特に上から5つ、6つだが、それ以外も全く使わないわけではない。

- Ref
- Fn::GetAtt
- Fn::Join
- Fn::Sub
- Fn::ImportValue
- Fn::FindInMap
- Fn::GetAZs
- Fn::Cidr
- Fn::Select
- Fn::Split
- Fn::Base64
- 条件関数
  - Fn::If
  - Fn::Or
  - Fn::And
  - Fn::Equals
  - Fn::Not

このページでは、それぞれの組み込み関数を解説していく。

# Ref

CloudFormationのテンプレートで一番よく見かけ、よく使う組み込み関数はこの **Ref** だ。Refは指定したパラメーターの値、または指定したコンポーネントから作成されたAWSリソースの値を取り出すことができる。

次の例は、S3の名前をRefでパラメーターから取り出した値にし、テンプレートの出力にRefでAWSリソースから取り出したバケット名を指定するテンプレートだ。補足として、組み込み関数のほとんどは完全な形の構文と短縮した形の構文が存在しており、この例ではその両方を取り入れている。Refでは、 `Ref:` で指定するのが完全形の構文で、 `!Ref` で指定するのが短縮形の構文となる。主に利用するのは短縮形の構文だが、完全形の構文も覚えておいたほうが良い。

```yaml
AWSTemplateFormatVersion: 2010-09-09
Parameters:

  S3BucketName: # パラメータ
    Type: String
    Default: my-s3-bucket-1234567890
    Description: Your S3 Bucket Name

Resources:

  MyS3Bucket: # コンポーネント（AWSリソース）
    Type: AWS::S3::Bucket
    Properties:
      BucketName:
        Ref: S3BucketName # Ref（完全形の構文）でパラメーターの値を取得

Outputs:

  MyOutput:
    Value: !Ref MyS3Bucket # Ref（短縮形の構文）でAWSリソースの値を取得
```

AWSリソースから取り出せる値はコンポーネントの `Type` によってそれぞれ全く違っていて、例えば `AWS::S3::Bucket` ではバケット名が返されるが、物によってはARNが返されるものなどがある。ほとんどはAWSサービス内部で特に利用するためのリソース名などが返されるが、テンプレートを作成する場合、このあたりは [Refのドキュメント](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-ref.html) を見ながら進めるしかない。


# Fn::GetAtt

RefはAWSリソースから決められた1つの値しか取れないため、本当に欲しい値（ARNなど）を取得できないことが多い。そこで利用できるのが **Fn::GetAtt** で、これはAWSリソースからRefで取れない値を取り出すための組み込み関数になる。これもRefに続いてテンプレート内で良く使われる。

次の例は、作成したS3バケットの `名前` 、 `ドメイン名` 、 `Webサイトとして公開した場合のURL` を出力するテンプレートだ。GetAttにもRefと同じように短縮形があり、完全形の構文は `Fn::GetAtt:` で、短縮形の構文は `!GetAtt` となる。完全形の場合は引数に配列を渡すが、短縮形の場合はリソースと属性をドットで繋ぐだけで良い。Refと同じくこちらも短縮形が多く使われるが、AWSのサンプルでは完全形もよく見かけるので覚えておいたほうが良い。

```yaml
AWSTemplateFormatVersion: 2010-09-09
Resources:

  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: mybucket-123456789012

Outputs:

  BucketName:
    Value: !Ref MyS3Bucket # バケット名はRefで取得

  BucketDomainName:
    Value:
      Fn::GetAtt: # GetAtt（完全形の構文）でS3のドメイン名を取得
        - MyS3Bucket
        - DomainName

  BucketWebsiteURL:
    Value: !GetAtt MyS3Bucket.WebsiteURL # GetAtt（短縮形の構文）でWebサイトとして公開した場合のURLを取得
```

これもRefと同じようにAWSリソースから取り出せる値や属性名はコンポーネントの `Type` によって全く違っているため、テンプレートを作成する際は [Fn::GetAttのドキュメント](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-getatt.html) を見ながら進める。

# Fn::Join

**Fn::Join** は区切り記号を挟んで文字列配列を連結するための組み込み関数だ。引数には2つの要素を含んだ配列を渡し、その配列の1つめの要素には区切り記号（文字列）、2つめの要素には連結させたい文字列配列を入れておく。Refの例で出たJoinでは、`"mybucket-"` と `!Ref AWS::AccountId` が `""（空文字列）` で結合されていることが分かる。

```yaml
AWSTemplateFormatVersion: 2010-09-09
Resources:

  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName:
        Fn::Join:
          - ''
          - - 'mybuket-'
            - !Ref AWS::AccountId
```

YAMLなので分かり辛いかもしれないが、実際の配列にすると `["", ["mybucket-", !Ref AWS::AccountId] ]` となる。YAMLのテンプレート内の配列はこの形で書き換えることも可能。どちらの形もよく利用されており、場面によって使い分けられていたりするので一概にどちらかが良いとは言えない。また、Joinにも短縮形があり、完全形の構文が `Fn::Join:` で短縮形の構文が `!Join` だが、YAML的にキーの後に続けて `!Join` を置けるかどうか、くらいの違いしか無い。 `!Join` と `[]` の配列の形を組み合わせることで、キーの値として1行でJoinを記述することが可能。

```yaml
AWSTemplateFormatVersion: 2010-09-09
Resources:

  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Join ["", ["mybucket-", {"Ref": "AWS::AccountId"}] ]
```

# Fn::Sub

**Fn::Sub** 特定の文字列を指定した値に置き換えることができる組み込み関数で、これもまた大変便利な関数だ。Refと同じくらいよく使う言っても過言ではない。Subでは空文字列や短い文字で結合するような複雑でないJoinを置き換えることも可能だが、結局はどちらも欲しい文字列を生成する同じ用途の関数なので、これは可読性で選択することになる。

Subには文字列とハッシュ型の変数マップを渡すことで目的の文字列を生成することができ、先述した、S3バケットの名前にJoinを使ってAWSアカウントIDを含める方法を `Fn::Sub:` で実現するとこうなる。変数は文字列内に `${変数名}` を含めることで埋め込むことが出来て、これは例えるならC言語のprintfに似ている。Subにも他の関数と同じく短縮形の構文があり、完全形の構文は `Fn::Sub:` で短縮形の構文が `!Sub` だ。

```yaml
AWSTemplateFormatVersion: 2010-09-09
Resources:

  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName:
        Fn::Sub: # Sub（完全形の構文）で、変数 UniqueString を変数マップの値で置き換える
          - mybuket-${UniqueString}
          - UniqueString: !Ref AWS::AccountId
```

これで `BucketName` は `mybuket-123456789012（AWSアカウントID）` になる。変数マップのキーの値は今回は擬似パラメーターからAWSアカウントIDを取得しているが、実際はRefでなくても例えばただの文字列やJoinで結合した文字列など、最終的に出力されるものが文字列であるなら何でも良い。変数マップはハッシュであるので、 `{ UniqueString: !Ref AWS::AccountId }` という書き方も可能だ。

Subには2つの利用方法（記述方法）があり、1つめは先述した例の文字列と変数マップを渡して目的の文字列を生成する方法。もうひとつは、文字列だけを渡す方法だ。Subに変数を含む文字列だけを渡すと、変数名（これはコンポーネント名になる）から自動で `Ref` を使った時のパラメーターやリソースの値、または `GetAtt` を使った時の属性に置き換えてくれる。これはES6などで言うところのテンプレート文字列のようなものだ。

1. 文字列内の変数を変数マップのパラメーターでマッピングする使い方
2. 文字列内の変数を既存のテンプレートパラメーター、リソースの値、リソースの属性で置き換えする使い方

下の例は、短縮形の構文かつ、Subに文字列だけを渡す2の使い方でS3バケットを作成する例だ。Subを使うときはだいたいこの組み合わせ、この使い方で使われていて、1の使い方はあまり見かけない。Outputsの部分では、Subを使ったリソースの属性の取得方法を紹介する。Refの値は変数名に取得したいリソースの値のコンポーネント名を付けるだけで良いが、リソースの属性を取得する際は `.（ドット）` で属性名を繋げなければならない。

```yaml
AWSTemplateFormatVersion: 2010-09-09
Resources:

  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub mybuket-${AWS::AccountId} # Sub（短縮形の構文）の2の使い方でアカウントIDを取得（Refの代わり）

Outputs:

  BucketDomainName:
    Value: !Sub ${MyS3Bucket.DomainName} # Sub（短縮形の構文）の2の使い方でS3のドメイン名を取得（GetAttの代わり）

  BucketWebsiteURL:
    Value:
      !Sub # Sub（短縮形の構文）の1の使い方で、WebsiteURLを取得（GetAttの代わり）
        - ${WebsiteURL}
        - { WebsiteURL: !GetAtt MyS3Bucket.WebsiteURL }
```

# Fn::ImportValue




# [WIP]

# 参考

- [組み込み関数リファレンス](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference.html)
- [Refのドキュメント](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-ref.html)
- [Fn::GetAttのドキュメント](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-getatt.html)
- [Fn::Joinのドキュメント](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-join.html)
- [Fn::Subのドキュメント](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-sub.html)
- [Fn::ImportValueのドキュメント](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-importvalue.html)