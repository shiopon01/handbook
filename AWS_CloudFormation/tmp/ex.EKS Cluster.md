# EKS Cluster

0からEKSのクラスターを作成するためのテンプレート。このテンプレートではクラスターとそれに必要なIAM Role、VPCが作成されるだけであり、ワーカーノードのインスタンスは作成されない。

※ EKSは現在オレゴンとバージニア北部でしか提供されていないので、CloudFormationでのEKSクラスター作成もこの2つのリージョンでしか行えない。リージョンを間違えるとリソースのTypeが不正だとエラーが出るので注意。

# Template (YAML Only)

作成されるリソースの一覧。作成時のパラメーター `ClusterName` には自分が設定したい好きなEKSクラスターの名前を入力する。

- AWS::IAM::Role
- AWS::EC2::VPC
- AWS::EC2::InternetGateway
- AWS::EC2::RouteTable
- AWS::EC2::Route
- AWS::EC2::Subnet
- AWS::EC2::SubnetRouteTableAssociation
- AWS::EC2::SecurityGroup
- AWS::EKS::Cluster

Subnetは3つあるので、Subnet関連のリソースも全て3つずつ作成される。このテンプレートで作成されるリソースはVPCとその基本的な設定、EKSクラスター用のIAM Role、実際のEKSクラスターとなる。

Outputsではクラスターの情報、VPCやサブネットの情報を出力しているが、これはワーカーノードを作成したり、kubectlからクラスターを操作する場合に必要になる。

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: Amazon EKS - Cluster

Parameters:

  ## Cluster VPC

  VpcBlock:
    Type: String
    Default: 192.168.0.0/16
    Description: The CIDR range for the VPC. This should be a valid private (RFC 1918) CIDR range.

  Subnet01Block:
    Type: String
    Default: 192.168.64.0/18
    Description: CidrBlock for subnet 01 within the VPC

  Subnet02Block:
    Type: String
    Default: 192.168.128.0/18
    Description: CidrBlock for subnet 02 within the VPC

  Subnet03Block:
    Type: String
    Default: 192.168.192.0/18
    Description: CidrBlock for subnet 03 within the VPC

  ## Cluster

  ClusterName:
    Type: String
    Description: EKSCluster Name

Metadata:
  AWS::CloudFormation::Interface:
    ParameterGroups:
      - Label:
          default: "Cluster Configuration"
        Parameters:
          - ClusterName
          - VpcBlock
          - Subnet01Block
          - Subnet02Block
          - Subnet03Block

Resources:

  ## IAM Role

  EKSServiceRole:
    Type: AWS::IAM::Role
    Properties:
      Path: /
      RoleName: eksServiceRole
      AssumeRolePolicyDocument:
        Statement:
          - Effect: Allow
            Action: sts:AssumeRole
            Principal:
              Service:
                - eks.amazonaws.com
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
        - arn:aws:iam::aws:policy/AmazonEKSServicePolicy

  ## VPC

  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcBlock
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
      - Key: Name
        Value: !Sub ${AWS::StackName}-VPC

  InternetGateway:
    Type: AWS::EC2::InternetGateway

  VPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    DependsOn: [InternetGateway, VPC]
    Properties:
      InternetGatewayId: !Ref InternetGateway
      VpcId: !Ref VPC

  RouteTable:
    Type: AWS::EC2::RouteTable
    DependsOn: VPC
    Properties:
      VpcId: !Ref VPC
      Tags:
      - Key: Name
        Value: Public Subnets
      - Key: Network
        Value: Public

  Route:
    Type: AWS::EC2::Route
    DependsOn: [VPCGatewayAttachment, RouteTable, InternetGateway]
    Properties:
      RouteTableId: !Ref RouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  Subnet01:
    Type: AWS::EC2::Subnet
    DependsOn: VPC
    Metadata:
      Comment: Subnet 01
    Properties:
      AvailabilityZone:
        Fn::Select:
        - '0'
        - Fn::GetAZs:
            Ref: AWS::Region
      CidrBlock:
        Ref: Subnet01Block
      VpcId:
        Ref: VPC
      Tags:
        - Key: Name
          Value: !Sub ${AWS::StackName}-Subnet01

  Subnet02:
    Type: AWS::EC2::Subnet
    DependsOn: VPC
    Metadata:
      Comment: Subnet 02
    Properties:
      AvailabilityZone:
        Fn::Select:
        - '1'
        - Fn::GetAZs:
            Ref: AWS::Region
      CidrBlock:
        Ref: Subnet02Block
      VpcId:
        Ref: VPC
      Tags:
        - Key: Name
          Value: !Sub ${AWS::StackName}-Subnet02

  Subnet03:
    Type: AWS::EC2::Subnet
    DependsOn: VPC
    Metadata:
      Comment: Subnet 03
    Properties:
      AvailabilityZone:
        Fn::Select:
        - '2'
        - Fn::GetAZs:
            Ref: AWS::Region
      CidrBlock:
        Ref: Subnet03Block
      VpcId:
        Ref: VPC
      Tags:
        - Key: Name
          Value: !Sub ${AWS::StackName}-Subnet03

  Subnet01RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    DependsOn: [Subnet01, RouteTable]
    Properties:
      SubnetId: !Ref Subnet01
      RouteTableId: !Ref RouteTable

  Subnet02RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    DependsOn: [Subnet02, RouteTable]
    Properties:
      SubnetId: !Ref Subnet02
      RouteTableId: !Ref RouteTable

  Subnet03RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    DependsOn: [Subnet03, RouteTable]
    Properties:
      SubnetId: !Ref Subnet03
      RouteTableId: !Ref RouteTable

  ControlPlaneSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    DependsOn: VPC
    Properties:
      GroupDescription: Cluster communication with worker nodes
      VpcId: !Ref VPC

  ## EKS Cluster

  EKSCluster:
    Type: AWS::EKS::Cluster
    Properties:
      Name: !Ref ClusterName
      Version: "1.10"
      RoleArn: !GetAtt EKSServiceRole.Arn
      ResourcesVpcConfig:
        SecurityGroupIds:
          - !Ref ControlPlaneSecurityGroup
        SubnetIds:
          - !Ref Subnet01
          - !Ref Subnet02
          - !Ref Subnet03

Outputs:

  Cluster:
    Description: Cluster Name
    Value: !Ref EKSCluster

  VpcId:
    Description: The VPC Id
    Value: !Ref VPC

  SubnetIds:
    Description: All subnets in the VPC
    Value: !Join [ ",", [ !Ref Subnet01, !Ref Subnet02, !Ref Subnet03 ] ]

  ClusterControlPlaneSecurityGroup:
    Description: Cluster Name
    Value: !Ref ControlPlaneSecurityGroup

  ClusterCertificateAuthorityData:
    Description: Cluster CertificateAuthorityData
    Value: !GetAtt EKSCluster.CertificateAuthorityData

  ClusterEndpoint:
    Description: Cluster Endpoint
    Value: !GetAtt EKSCluster.Endpoint
```


## 参考

- [Amazon EKS クラスターの作成](https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/create-cluster.html)