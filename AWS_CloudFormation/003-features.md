# 特徴

## Cumbersome CloudFormation...

ユースケースでも述べた用に大変に便利なCloudFormationではあるが、利用している中でいくつか面倒な点は発見できる。これは、利用者が意識して対応しなければならない点だ。

- S3: テンプレートから作成したS3バケットは中身が空でないと `スタックの削除` での削除が失敗する
- IAM: 他のリソースに同じテンプレートで作成したIAMロールを適用する際、事前に設定しておかねばならない必須のIAMポリシーが存在する場合がある。その際、IAMロールへのポリシー適用を待つため、対象のリソースには `DependsOn` オプションでポリシー設定を待つ記述が必要になる
- VPC: VPCには依存関係が多いためか、テンプレートから作成したままの状態のAWSリソースでも `スタックの削除` が失敗する場合がある。これを回避するために、作成時には必要なくとも、リソースの削除の順番を明示化しておく目的で `DependsOn` オプションを多用しなければならない
- VPC: テンプレートから作成したVPCに手動でサブネットやインターネットゲートウェイを追加した時は、それらをまた手動で削除しないとスタックを削除できない
- そもそも、新しいAWSサービスは未対応のことがある（Rekognitionなど）

S3バケットの作成を含むスタックを削除する場合は、事前にバケットを空にしておかなければならない。（対象のバケットを手動で削除しておいても良い。）このS3バケット削除失敗のエラー `DELETE_FAILED` は、CloudTrailの `証跡` 機能でS3にログを保存している場合に頻発する。または、他のバケットのアクセスログを対象のバケットに保存している場合にも。

VPCリソースでの依存関係とは、 `VPC` に対しての `Subnet` 、 `Subnet` に対しての `RouteTable` などのことだ。スタックの削除失敗を回避するため、いくつかの必要なコンポーネントに対しては `DependsOn` で作成・削除の順番を指定してやる必要がある。そしてVPCにはちょっとした追加を手動でしてしまいがちだが、必ずCloudFormationの更新からVPCの設定を行うようにする。

そしてわかりやすく一番困る問題は、比較的新しいサービスがCloudFormationでサポートされていない問題だ。しかしこの対応速度、サービスによってバラバラであり、比較的新しい `Media Packages` のシリーズは既に対応されている。だが、もっと前から出ている `Rekognition` や `Kinesis Video Streams` は対応されていない。AWS CLIやSDKからは新しめのサービスも作成することができるので、対応されていなかった場合はここから作成するしかない。