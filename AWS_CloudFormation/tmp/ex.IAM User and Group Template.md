# IAM User and Group Template

これはIAM UserとIAM Group、ユーザーのアクセスキーを作成するテンプレートだ。ポリシーはグループのインラインポリシー（グループポリシー）として直接記述されている。

# Template (YAML Only)

作成されるリソースの一覧。

- AWS::IAM::User
- AWS::IAM::AccessKey
- AWS::IAM::Group
- AWS::IAM::UserToGroupAddition

SDKなどからAWSリソースを操作する際に役に立つアクセスキーとシークレットキーは `Output` で出力しているのでCloudFormationの出力タブで確認できる。（本来であればこのアクセスキー、例えばEC2などに直接渡すなどして隠蔽すべきであり、出力タブに表示するのは好ましくない）

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: IAM Template

Parameters:

  GroupName:
    Type: String
    Default: DemoGroup
    Description: GroupName

  GroupPolicyName:
    Type: String
    Default: DemoGroupPolicy
    Description: GroupPolicyName

  UserName:
    Type: String
    Default: DemoUser
    Description: UserName

Resources:

  Group:
    Type: AWS::IAM::Group
    Properties:
      Path: /
      Policies:
        - PolicyName: !Ref GroupPolicyName
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - logs:*
                Resource: '*'

  User:
    Type: AWS::IAM::User
    Properties:
      UserName: !Ref UserName

  AKStreamer:
    Type: AWS::IAM::AccessKey
    DependsOn: User
    Properties:
      UserName: !Ref User

  AddUserToGroup:
    Type: AWS::IAM::UserToGroupAddition
    Properties:
      GroupName: !Ref Group
      Users:
        - !Ref User

Outputs:

  Name:
    Value: !Sub ${AWS::StackName}

  AccessKeyId:
    Value: !Ref AKStreamer
    Export:
      Name: !Sub ${AWS::StackName}-AccessKeyId

  SecretAccessKey:
    Value: !GetAtt AKStreamer.SecretAccessKey
    Export:
      Name: !Sub ${AWS::StackName}-SecretAccessKey
```