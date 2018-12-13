# [WIP]

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: Cognito Template

Parameters:

  UserPoolName:
    Type: String
    Default: userPool
    Description: userPoolName

  UserPoolClientName:
    Type: String
    Default: userPoolClient
    Description: userPoolClientName

  IdentityPoolName:
    Type: String
    Default: identityPool
    Description: identityPoolName

  CognitoAuthPolicyName:
    Type: String
    Default: CognitoAuthPolicy
    Description: CognitoAuthPolicyName

  CognitoUnauthPolicyName:
    Type: String
    Default: CognitoUnauthPolicy
    Description: CognitoUnauthPolicyName

Resources:

  UserPool:
    Type: AWS::Cognito::UserPool
    Properties:
      UserPoolName: !Ref UserPoolName
      Schema:
        - Name: email
          StringAttributeConstraints:
            MinLength: 0
            MaxLength: 2048
          Required: true
          AttributeDataType: String
          Mutable: true
      AliasAttributes: ['email']
      AutoVerifiedAttributes: ['email']
      EmailVerificationSubject: 'Your verification code'
      EmailVerificationMessage: 'Your confirmation code is {####}.'
      MfaConfiguration: OFF
      Policies:
        PasswordPolicy:
          MinimumLength: 8
          RequireLowercase: false
          RequireNumbers: false
          RequireSymbols: false
          RequireUppercase: false

  UserPoolClient:
    Type: AWS::Cognito::UserPoolClient
    Properties:
      ClientName: !Ref UserPoolClientName
      UserPoolId: !Ref UserPool
      GenerateSecret: true

  # Federated identity

  IdentityPool:
    Type: AWS::Cognito::IdentityPool
    Properties:
      IdentityPoolName: !Ref IdentityPoolName
      AllowUnauthenticatedIdentities: false
      CognitoIdentityProviders:
        - ClientId: !Ref UserPoolClient
          ProviderName: !GetAtt UserPool.ProviderName

  ## Auth Role

  CognitoAuthRole:
    Type: AWS::IAM::Role
    Properties:
      Path: /
      AssumeRolePolicyDocument:
        Statement:
          - Effect: Allow
            Principal:
              Federated: cognito-identity.amazonaws.com
            Action: sts:AssumeRoleWithWebIdentity
            Condition:
              StringEquals:
                cognito-identity.amazonaws.com:aud: !Ref IdentityPool
              ForAnyValue:StringLike:
                cognito-identity.amazonaws.com:amr: authenticated

  CognitoAuthPolicy:
    Type: AWS::IAM::Policy
    Properties:
      PolicyName: !Ref CognitoAuthPolicyName
      PolicyDocument:
        Statement:
          - Effect: Allow
            Action:
              - mobileanalytics:PutEvents
              - 'cognito-sync:*'
              - 'cognito-identity:*'
            Resource:
              - '*'
      Roles:
        - !Ref CognitoAuthRole

  ## Unauth Role

  CognitoUnauthRole:
    Type: AWS::IAM::Role
    Properties:
      Path: /
      AssumeRolePolicyDocument:
        Statement:
          - Effect: Allow
            Principal:
              Federated: cognito-identity.amazonaws.com
            Action: sts:AssumeRoleWithWebIdentity
            Condition:
              StringEquals:
                cognito-identity.amazonaws.com:aud: !Ref IdentityPool
              ForAnyValue:StringLike:
                cognito-identity.amazonaws.com:amr: unauthenticated

  CognitoUnauthPolicy:
    Type: AWS::IAM::Policy
    Properties:
      PolicyName: !Ref CognitoUnauthPolicyName
      PolicyDocument:
        Statement:
          - Effect: Allow
            Action:
              - mobileanalytics:PutEvents
              - cognito-sync:*
            Resource:
              - "*"
      Roles:
        - !Ref CognitoUnauthRole

  ## Attachment

  IdentityPoolRoleAttachment:
    Type: AWS::Cognito::IdentityPoolRoleAttachment
    Properties:
      IdentityPoolId: !Ref IdentityPool
      Roles:
        authenticated: !GetAtt CognitoAuthRole.Arn
        unauthenticated: !GetAtt CognitoUnauthRole.Arn

Outputs:

  Name:
    Value: !Sub ${AWS::StackName}

  UserPoolId:
    Value: !Ref UserPool
    Export:
      Name: !Sub ${AWS::StackName}-UserPoolId

  UserPoolClientId:
    Value: !Ref UserPoolClient
    Export:
      Name: !Sub ${AWS::StackName}-UserPoolClientId

  IdPoolId:
    Value: !Ref IdentityPool
    Export:
      Name: !Sub ${AWS::StackName}-IdPoolId
```