以下メモ


タイプごとのリクエストの例ではAWS SDK for Rubyを利用しているが、他言語のSDKでもパラメータはほとんど変わらない。 (キャメルケース、スネークケースなどの違いはある)

とりあえず、認証情報の作成とAWS Clientの作成。

``` ruby
awsClient = Aws::EC2::Client.new(
  region: region,
  credentials: Aws::Credentials.new(access_key, secret_key)
)
```

# リクエストタイプ

## リクエストタイプ: instance

```ruby
options = {
  instance_count: 1,
  spot_price: "0.005",
  type: "one-time",
  launch_specification: {
    image_id: "ami-bec974d8", # ubuntu 16.04
    instance_type: "t2.micro",
    key_name: "myKeyName",
    security_group_ids: ["sg-123456"],
    subnet_id: "subnet-123456"
  }
}

resp = awsClient.request_spot_instances(options)
```

## リクエストタイプ:  fleet

```ruby
options = {
  spot_fleet_request_config: {
    iam_fleet_role: "arn:aws:iam::123456789:role/my_ iam_fleet_role",
    launch_specifications: [
      {
        # iam_instance_profile: {
        #   arn: "arn:aws:iam::123456789:role/my_iam_instance_profile"
        # },
        image_id: "ami-bec974d8", # ubuntu 16.04
        instance_type: "t2.micro",
        key_name: "myKeyName",
        security_groups: [
          {
            group_id: "sg-123456",
          }
        ],
        subnet_id: "subnet-123456",
      }
    ],
    spot_price: "0.0152",
    target_capacity: 1,

    # option
    allocation_strategy: "lowestPrice",
    excess_capacity_termination_policy: "default",
    type: "request",
    valid_from: Time.now,
    valid_until: Time.now + 86400,
    terminate_instances_with_expiration: true
  }
}

resp = awsClient.request_spot_fleet(options)
```

# インスタンス作成後、自動的にコマンドを実行

AWS Lambdaの Node.js 6.10 で実行することを想定。

次の例は、スポットインスタンス (fleet) のリクエストを送り、インスタンスを確保できた場合はそのまま指定のコマンドを実行する例。

インスタンス作成後に実行するコマンドは、オプションの `UserData` 部分で設定する。SDKからリクエストを送るならこれはbase64の形式で送らないといけないので、変換する必要がある。

インスタンスの有効期限はJSONの `ValidFrom`, `ValidUntil` で行っている。タイムゾーンは、LambdaのENVで `{ TZ: "Asia/Tokyo"}` を設定すれば変更可能。 `TerminateInstancesWithExpiration` の設定で、有効期限が切れるとインスタンスが削除される設定になっている。

```js
let AWS = require("aws-sdk");
let ec2 = new AWS.EC2({region: 'ap-northeast-1'});

exports.handler = (event, context, callback) => {
  let userData64 = createUserData64(event.cmd)
  let date = new Date()
  let date2 = new Date()
  date2.setDate(date2.getDate() + 1);

  let options = {
    SpotFleetRequestConfig: {
      IamFleetRole: "arn:aws:iam::123456789:role/my_ iam_fleet_role",
      LaunchSpecifications: [
        {
          ImageId: "ami-da9e2cbc", // Amazon Linux
          InstanceType: "t2.micro",
          KeyName: "myKeyName",
          SecurityGroups: [
            {
              GroupId: "sg-123456",
            }
          ],
          SubnetId: "subnet-123456",
          UserData: userData64,
        }
      ],
      SpotPrice: "0.0152",
      TargetCapacity: 1,

      AllocationStrategy: "lowestPrice",
      ExcessCapacityTerminationPolicy: "default",
      Type: "request",
      ValidFrom: date,
      ValidUntil: date2,
      TerminateInstancesWithExpiration: true,
    }
  }

  ec2.requestSpotFleet(options, function(err, data) {
    if (err) {
      callback(null, err);
    } else {
      /* {SpotFleetRequestId: "sfr-73fbd2ce-aa30-494c-8788-1cee4EXAMPLE"} */
      callback(null, data);
    }
  })
}

function createUserData64 (cmd) {
  // User Data
  let userData = `#!/bin/bash
cp /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
yum update -y
nohup ` + cmd + ` &`

  // return String of base64 format
  return new Buffer(userData).toString('base64')
}
```

State Functionsに組み込んだり、そのままコマンドを動かすAPIとする等、スポットインスタンスには無限の可能性が。

# 参考

- https://aws.amazon.com/jp/sdk-for-ruby/
- http://docs.aws.amazon.com/sdkforruby/api/Aws/EC2/Types/RequestSpotFleetRequest.html
- http://docs.aws.amazon.com/sdkforruby/api/Aws/EC2/Types/SpotFleetRequestConfigData.html
