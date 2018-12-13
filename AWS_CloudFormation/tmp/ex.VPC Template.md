# VPC Template

これはVPCと付随するSubnetを2つ作成し、インターネットゲートウェイとルートテーブルを使用してサブネットのインターネット接続までを行うテンプレートだ。セキュリティーグループでは自分自身からのインバウンドを許可し、ハマりどころを減らしている。

# Template (YAML Only)

作成されるリソースの一覧。

- AWS::EC2::VPC
- AWS::EC2::InternetGateway
- AWS::EC2::VPCGatewayAttachment
- AWS::EC2::Subnet
- AWS::EC2::RouteTable
- AWS::EC2::Route
- AWS::EC2::SubnetRouteTableAssociation
- AWS::EC2::SecurityGroup
- AWS::EC2::SecurityGroupIngress

VPCにインターネットゲートウェイを適応したりルートテーブルに設定を付与する等、設定の項目が多く、モノとして残るAWSリソースはあまり多くない。最終的に残るAWSリソースは以下の5つのみだ。（このテンプレートではSubnetを2つ作成しているので、実際はSubnetが2つ作成される）

- AWS::EC2::VPC
- AWS::EC2::InternetGateway
- AWS::EC2::Subnet
- AWS::EC2::RouteTable
- AWS::EC2::SecurityGroup

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: VPC Template

Parameters:

  VPCName:
    Type: String
    Default: Demo VPC
    Description: VPCName

  Subnet1Name:
    Type: String
    Default: Demo Subnet 1
    Description: Subnet1Name

  Subnet2Name:
    Type: String
    Default: Demo Subnet 2
    Description: Subnet2Name

  SecurityGroupName:
    Type: String
    Default: Demo SecurityGroup
    Description: SecurityGroupName

Resources:

  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Ref VPCName

  InternetGateway:
    Type: AWS::EC2::InternetGateway

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  Subnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs ]
      MapPublicIpOnLaunch: true
      CidrBlock: 10.0.0.0/24
      Tags:
        - Key: Name
          Value: !Ref Subnet1Name

  Subnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs ]
      MapPublicIpOnLaunch: true
      CidrBlock: 10.0.1.0/24
      Tags:
        - Key: Name
          Value: !Ref Subnet2Name

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachGateway
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  Subnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref Subnet1

  Subnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref Subnet2

  SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      VpcId: !Ref VPC
      GroupName: SecurityGroup
      GroupDescription: Security Group

  SecurityGroupIngress:
    Type: AWS::EC2::SecurityGroupIngress
    DependsOn: SecurityGroup
    Properties:
      GroupId: !Ref SecurityGroup
      IpProtocol: tcp
      FromPort: 0
      ToPort: 65535
      SourceSecurityGroupId: !Ref SecurityGroup

Outputs:

  Name:
    Description: Stack name
    Value: !Sub ${AWS::StackName}

  VPCID:
    Description: VPC ID
    Value: !Ref VPC
    Export:
      Name: !Sub ${AWS::StackName}-VPC

  Subnet1ID:
    Description: Subnet1 ID
    Value: !Ref Subnet1
    Export:
      Name: !Sub ${AWS::StackName}-Subnet1

  Subnet2ID:
    Description: Subnet2 ID
    Value: !Ref Subnet2
    Export:
      Name: !Sub ${AWS::StackName}-Subnet2

  SecurityGroupID:
    Description: SecurityGroup ID
    Value: !GetAtt SecurityGroup.GroupId
    Export:
      Name: !Sub ${AWS::StackName}-SecurityGroup
```
