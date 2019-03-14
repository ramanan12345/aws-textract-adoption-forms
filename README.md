# Textract Adoption Form

Consuming and processing WA Animals adoption forms using AWS Textract and placing that data in DynamoDB

## Serverless Setup

```bash
## Project Setup
mkdir waanimals-adoption-textract
cd waanimals-adoption-textract
serverless create --template aws-python3 --name waanimals-adoption-textract
```

### Requirements [Workaround]

Currently the boto3 client deployed to Lambda doesn't include textract. We'll need to force an update on the client using python requirements

```bash
serverless plugin install -n serverless-python-requirements
```

Create a `requirements.txt` file and add the following to it

```bash
boto3>=1.9.111
```

Also in the next section take careful note of the lines below

```bash
pythonRequirements:
  dockerizePip: non-linux
  noDeploy: []
```

`noDeploy` tells the `serverless-python-requirements` plugin to include boto3 and not omit it. More information can be found [HERE](https://github.com/UnitedIncome/serverless-python-requirements#omitting-packages)

### Update serverless.yml

```yaml
service: waanimals-adoption-textract

custom:
  pythonRequirements:
    dockerizePip: non-linux
    noDeploy: []
  PRIMARY_KEY: Microchip Number
  ADOPTION_FORM_TABLE: waanimals-adoption-forms-table
  ADOPTION_FORM_BUCKET: waanimals-adoption-forms
  ADOPTION_EMAIL_BUCKET: waanimals-adoption-emails
  SES_RULE_SET_NAME: waanimals-adoption-form-rule
  SES_RULE_SET_EMAIL: contact@devopstar.com

provider:
  name: aws
  runtime: python3.7
  iamRoleStatements:
    - Effect: Allow
      Action:
        - dynamodb:PutItem
      Resource:
        - { "Fn::GetAtt": ["AdoptionDynamoDBTable", "Arn"] }
    - Effect: Allow
      Action:
        - s3:GetObject
        - s3:DeleteObject
      Resource:
        - arn:aws:s3:::${self:custom.ADOPTION_EMAIL_BUCKET}/*
    - Effect: Allow
      Action:
        - s3:GetObject
        - s3:PutObject
      Resource:
        - arn:aws:s3:::${self:custom.ADOPTION_FORM_BUCKET}/*
    - Effect: Allow
      Action:
        - textract:AnalyzeDocument
      Resource: "*"
  environment:
    ADOPTION_FORM_TABLE: ${self:custom.ADOPTION_FORM_TABLE}
    ADOPTION_FORM_BUCKET: ${self:custom.ADOPTION_FORM_BUCKET}

functions:
  textract:
    handler: handler.textract
    events:
      - s3:
          bucket: ${self:custom.ADOPTION_FORM_BUCKET}
          event: s3:ObjectCreated:*
  ses:
    handler: handler.ses
    events:
      - s3:
          bucket: ${self:custom.ADOPTION_EMAIL_BUCKET}
          event: s3:ObjectCreated:*

resources:
  Resources:

    AdoptionDynamoDBTable:
      Type: AWS::DynamoDB::Table
      Properties:
        AttributeDefinitions:
          - AttributeName: ${self:custom.PRIMARY_KEY}
            AttributeType: S
        KeySchema:
          - AttributeName: ${self:custom.PRIMARY_KEY}
            KeyType: HASH
        ProvisionedThroughput:
          ReadCapacityUnits: 1
          WriteCapacityUnits: 1
        TableName: ${self:custom.ADOPTION_FORM_TABLE}

    S3EMailBucketPermissions:
      Type: AWS::S3::BucketPolicy
      Properties:
        Bucket: ${self:custom.ADOPTION_EMAIL_BUCKET}
        PolicyDocument:
          Statement:
            - Principal:
                Service: ses.amazonaws.com
              Action:
                - s3:PutObject
              Effect: Allow
              Sid: AllowSESPuts
              Resource:
                Fn::Join: ['', ['arn:aws:s3:::', "${self:custom.ADOPTION_EMAIL_BUCKET}", '/*'] ]
              Condition:
                StringEquals:
                  "aws:Referer": { Ref: AWS::AccountId }

    S3SMailProcessRule:
      Type: AWS::SES::ReceiptRule
      Properties:
          RuleSetName: default-rule-set
          Rule:
            Name: ${self:custom.SES_RULE_SET_NAME}
            Enabled: true
            Recipients:
              - ${self:custom.SES_RULE_SET_EMAIL}
            Actions:
              - S3Action:
                  BucketName: ${self:custom.ADOPTION_EMAIL_BUCKET}
                  ObjectKeyPrefix: ""

plugins:
  - serverless-python-requirements
```
