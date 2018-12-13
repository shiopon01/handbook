# API Gateway with Lambda Template (CORS-Setting included)

これはLambdaと、それを呼び出すAPI Gatewayを作成するテンプレートだ。CORSや最低限のステータスコード設定も含まれているから、このテンプレートをベースにAPIを作成すればAPIを作成するたびにしていた煩わしい設定は必要なくなる。

このテンプレートではLambdaの作成、APIの作成からデプロイまでを自動で行う。が、APIのステージのURLはCloudFormationから取得できないので、API GatewayのAWSコンソール（ブラウザ）から自分で確認してもらいたい。

# Template (YAML Only)

作成されるリソースの一覧。ほとんどがLambdaとAPI Gatewayに関連するものであるが、Lambda用のIAM Roleなどもいくつか作成される。

- AWS::IAM::Role
- AWS::IAM::Policy
-	AWS::Lambda::Function
- AWS::ApiGateway::RestApi
- AWS::ApiGateway::Resource
- AWS::ApiGateway::Method
- AWS::ApiGateway::Method (Option Method)
- AWS::Lambda::Permission
- AWS::ApiGateway::Deployment

`".*BadRequest.*"` を含むレンスポんスをエラーとして認識するので、Lambdaからエラーを返したいときは `throw new Error("BadRequest: " + JSON.stringify(err))` などでプログラムを終了させれば良い。

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: API Gateway Template

Parameters:

  FunctionName:
    Type: String
    Default: DemoFunction
    Description: FunctionName

  FunctionPolicyName:
    Type: String
    Default: DemoFunctionPolicy
    Description: FunctionPolicyName

  ApiName:
    Type: String
    Default: DemoApi
    Description: ApiName

Resources:

  # Lambda

  FunctionRole:
    Type: AWS::IAM::Role
    Properties:
      Path: /
      AssumeRolePolicyDocument:
        Statement:
        - Action:
          - sts:AssumeRole
          Effect: Allow
          Principal:
            Service:
            - lambda.amazonaws.com

  FunctionPolicy:
    Type: AWS::IAM::Policy
    Properties:
      PolicyName: !Ref FunctionPolicyName
      Roles:
        - !Ref FunctionRole
      PolicyDocument:
        Statement:
          - Effect: Allow
            Action:
              - logs:*
            Resource: "arn:aws:logs:*:*:*"

  Function:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: !Sub FunctionName
      Description: Demo Function
      Runtime: nodejs6.10
      Handler: index.handler
      Code:
        ZipFile: !Sub |
          exports.handler = (event, context, callback) => {
            callback(null, {message: process.env['Demo']})
          }
        # S3Bucket: !Ref S3Bucket
        # S3Key: zip/demo.js.zip
      Environment:
        Variables:
          Demo: Hello?
      MemorySize: 128
      Role: !GetAtt FunctionRole.Arn
      Timeout: 60

  # API Gateway

  ApiGateway:
    Type: AWS::ApiGateway::RestApi
    Properties:
      Description: Rest Api
      Name: !Ref ApiName

  Resource:
    Type: AWS::ApiGateway::Resource
    Properties:
      RestApiId: !Ref ApiGateway
      ParentId: !GetAtt ApiGateway.RootResourceId
      PathPart: demo

  Method:
    Type: AWS::ApiGateway::Method
    Properties:
      RestApiId: !Ref ApiGateway
      ResourceId: !Ref Resource
      HttpMethod: GET
      AuthorizationType: NONE
      Integration:
        Type: AWS
        IntegrationHttpMethod: POST
        Uri: !Sub arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${Function.Arn}/invocations
        IntegrationResponses:
          - StatusCode: 200
            ResponseParameters:
              method.response.header.Access-Control-Allow-Origin: "'*'"
          - StatusCode: 400
            ResponseParameters:
              method.response.header.Access-Control-Allow-Origin: "'*'"
            SelectionPattern: ".*BadRequest.*"
      MethodResponses:
        - StatusCode: 200
          ResponseModels:
            application/json: 'Empty'
          ResponseParameters:
            method.response.header.Access-Control-Allow-Origin: false
        - StatusCode: 400
          ResponseModels:
            application/json: 'Empty'
          ResponseParameters:
            method.response.header.Access-Control-Allow-Origin: false

  OptionsMethod:
    Type: AWS::ApiGateway::Method
    Properties:
      RestApiId: !Ref ApiGateway
      ResourceId: !Ref Resource
      HttpMethod: OPTIONS
      AuthorizationType: NONE
      Integration:
        Type: MOCK
        IntegrationResponses:
        - StatusCode: 200
          ResponseParameters:
            method.response.header.Access-Control-Allow-Headers: "'Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token,X-Requested-With,X-Requested-By'"
            method.response.header.Access-Control-Allow-Methods: "'OPTIONS'"
            method.response.header.Access-Control-Allow-Origin: "'*'"
          ResponseTemplates:
            application/json: ''
        PassthroughBehavior: WHEN_NO_MATCH
        RequestTemplates:
          application/json: '{"statusCode": 200}'
      MethodResponses:
      - StatusCode: 200
        ResponseModels:
          application/json: 'Empty'
        ResponseParameters:
            method.response.header.Access-Control-Allow-Headers: false
            method.response.header.Access-Control-Allow-Methods: false
            method.response.header.Access-Control-Allow-Origin: false

  FunctionInvokePermission:
    Type: AWS::Lambda::Permission
    Properties:
      Action: lambda:InvokeFunction
      FunctionName: !Ref Function
      Principal: apigateway.amazonaws.com
      SourceArn: {"Fn::Join" : ["", ["arn:aws:execute-api:", {"Ref": "AWS::Region"}, ":", {"Ref": "AWS::AccountId"}, ":", {"Ref": "ApiGateway"}, "/*/GET/demo"]]}

  # Deployment

  ApiDeployment:
    Type: AWS::ApiGateway::Deployment
    DependsOn: [Method, OptionsMethod]
    Properties:
      Description: Sample Deployment
      RestApiId: !Ref ApiGateway
      StageName: prd

Outputs:

  Name:
    Description: Stack name
    Value: !Sub ${AWS::StackName}

  RestApi:
    Value: !Ref ApiGateway
    Export:
      Name: !Sub ${AWS::StackName}-RestApi

  RootResourceId:
    Value: !GetAtt ApiGateway.RootResourceId
    Export:
      Name: !Sub ${AWS::StackName}-RootResourceId
```
