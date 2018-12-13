# ECS Setting

# [WIP]

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: ECS Template

Parameters:

  VPCName:
    Type: String
    Default: ECS VPC
    Description: VPCName

  Subnet1Name:
    Type: String
    Default: ECS Subnet 1
    Description: Subnet1Name

  Subnet2Name:
    Type: String
    Default: ECS Subnet 2
    Description: Subnet2Name

  ALBSecurityGroupName:
    Type: String
    Default: ALB SecurityGroup
    Description: ALBSecurityGroupName

  EC2SecurityGroupName:
    Type: String
    Default: EC2 SecurityGroup
    Description: EC2SecurityGroupName

  ECSClusterName:
    Type: String
    Default: ECSCluster
    Description: ECSClusterName

Resources:

  # Networking

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

  SecurityGroupForALB:
    Type: AWS::EC2::SecurityGroup
    Properties:
      VpcId: !Ref VPC
      GroupName: !Ref ALBSecurityGroupName
      GroupDescription: Application Load Balancer Security Group
      SecurityGroupIngress:
        - CidrIp: 0.0.0.0/0
          IpProtocol: tcp
          FromPort: 80
          ToPort: 80

  SecurityGroupForEC2:
    Type: AWS::EC2::SecurityGroup
    Properties:
      VpcId: !Ref VPC
      GroupName: !Ref EC2SecurityGroupName
      GroupDescription: EC2 Security Group
      SecurityGroupIngress:
        - SourceSecurityGroupId: !Ref SecurityGroupForALB
          IpProtocol: tcp
          FromPort: 0
          ToPort: 65535

  # Application Load Balancer

  LoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Subnets:
        - !Ref Subnet1
        - !Ref Subnet2
      SecurityGroups:
        - !Ref SecurityGroupForALB

  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    DependsOn: LoadBalancer
    Properties:
      VpcId: !Ref VPC
      Port: 80
      Protocol: HTTP
      TargetType: instance
      TargetGroupAttributes:
        - Key: deregistration_delay.timeout_seconds
          Value: 30
      Matcher:
        HttpCode: 200-299
      HealthCheckIntervalSeconds: 10
      HealthCheckPath: /
      HealthCheckProtocol: HTTP
      HealthCheckTimeoutSeconds: 5
      HealthyThresholdCount: 2

  LoadBalancerListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref LoadBalancer
      Port: 80
      Protocol: HTTP
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref TargetGroup

  ListenerRule:
    Type: AWS::ElasticLoadBalancingV2::ListenerRule
    Properties:
      ListenerArn: !Ref LoadBalancerListener
      Priority: 1
      Conditions:
        - Field: path-pattern
          Values:
            - /
      Actions:
        - TargetGroupArn: !Ref TargetGroup
          Type: forward

  # ECS Cluster

  ECSRole:
    Type: AWS::IAM::Role
    Properties:
      Path: /
      AssumeRolePolicyDocument:
        Statement:
          - Action: sts:AssumeRole
            Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role

  InstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Condition: EC2
    Properties:
      Path: /
      Roles:
        - !Ref ECSRole

  Cluster:
    Type: AWS::ECS::Cluster
    Properties:
      ClusterName: !Ref ECSClusterName





  AutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Condition: EC2
    Properties:
      VPCZoneIdentifier: !Ref Subnets
      LaunchConfigurationName: !Ref LaunchConfiguration
      MinSize: !Ref ClusterSize
      MaxSize: !Ref ClusterSize
      DesiredCapacity: !Ref ClusterSize
      Tags:
        - Key: Name
          Value: !Sub ${AWS::StackName} - ECS Host
          PropagateAtLaunch: true
    CreationPolicy:
      ResourceSignal:
        Timeout: PT15M
    UpdatePolicy:
      AutoScalingRollingUpdate:
        MinInstancesInService: 1
        MaxBatchSize: 1
        PauseTime: PT15M
        WaitOnResourceSignals: true

  LaunchConfiguration:
    Type: AWS::AutoScaling::LaunchConfiguration
    Condition: EC2
    Metadata:
      AWS::CloudFormation::Init:
        config:
          commands:
            01_add_instance_to_cluster:
                command: !Sub echo ECS_CLUSTER=${Cluster} > /etc/ecs/ecs.config
          files:
            "/etc/cfn/cfn-hup.conf":
              mode: 000400
              owner: root
              group: root
              content: !Sub |
                [main]
                stack=${AWS::StackId}
                region=${AWS::Region}
            "/etc/cfn/hooks.d/cfn-auto-reloader.conf":
              content: !Sub |
                [cfn-auto-reloader-hook]
                triggers=post.update
                path=Resources.ContainerInstances.Metadata.AWS::CloudFormation::Init
                action=/opt/aws/bin/cfn-init -v --region ${AWS::Region} --stack ${AWS::StackName} --resource LaunchConfiguration
          services:
            sysvinit:
              cfn-hup:
                enabled: true
                ensureRunning: true
                files:
                  - /etc/cfn/cfn-hup.conf
                  - /etc/cfn/hooks.d/cfn-auto-reloader.conf
    Properties:
      ImageId: !FindInMap [ AWSRegionToAMI, !Ref "AWS::Region", AMI ]
      InstanceType: !Ref InstanceType
      IamInstanceProfile: !Ref InstanceProfile
      SecurityGroups:
        - !Ref SecurityGroup
      UserData:
        "Fn::Base64": !Sub |
          #!/bin/bash
          yum install -y aws-cfn-bootstrap
          /opt/aws/bin/cfn-init -v --region ${AWS::Region} --stack ${AWS::StackName} --resource LaunchConfiguration
          /opt/aws/bin/cfn-signal -e $? --region ${AWS::Region} --stack ${AWS::StackName} --resource AutoScalingGroup
```