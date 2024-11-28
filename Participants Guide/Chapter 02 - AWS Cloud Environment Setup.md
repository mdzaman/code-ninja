# Chapter 2: AWS Cloud Development Setup & Best Practices

## Table of Contents
1. AWS Account Setup & Security Best Practices
2. IDE Configuration for AWS Development
3. Essential AWS Development Tools
4. Infrastructure as Code Setup
5. Local Development Environment Configuration
6. CI/CD Pipeline Tools
7. Monitoring and Debugging Tools

## 1. AWS Account Setup & Security Best Practices

### 1.1 Initial AWS Account Configuration
```bash
# Create dedicated IAM users instead of using root account
aws iam create-user --user-name dev-user

# Create development environment specific credentials
aws iam create-access-key --user-name dev-user

# Set up MFA for additional security
aws iam enable-mfa-device --user-name dev-user --serial-number arn:aws:iam::ACCOUNT-ID:mfa/dev-user --authentication-code1 CODE1 --authentication-code2 CODE2
```

### 1.2 AWS Credentials Configuration
```bash
# Configure AWS CLI with named profiles
aws configure --profile dev
# Enter your AWS Access Key ID
# Enter your AWS Secret Access Key
# Enter default region (e.g., us-east-1)
# Enter default output format (json recommended)
```

## 2. IDE Configuration for AWS Development

### 2.1 VS Code Extensions for AWS Development

Essential extensions and their purposes:
```plaintext
1. AWS Toolkit
   - Direct AWS service integration
   - CloudFormation template validation
   - Lambda function debugging
   Installation: ext install amazonwebservices.aws-toolkit-vscode

2. Thunder Client
   - Testing API Gateway endpoints
   - Save and share API collections
   Installation: ext install rangav.vscode-thunder-client

3. YAML
   - CloudFormation template editing
   - SAM template validation
   Installation: ext install redhat.vscode-yaml

4. Python
   - Lambda function development
   - Code linting and formatting
   Installation: ext install ms-python.python

5. ESLint
   - JavaScript/TypeScript code quality
   - Automated code formatting
   Installation: ext install dbaeumer.vscode-eslint
```

### 2.2 VS Code Settings for AWS Development
```json
{
  "aws.telemetry": false,
  "aws.profile": "dev",
  "editor.formatOnSave": true,
  "python.linting.enabled": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  }
}
```

## 3. Essential AWS Development Tools

### 3.1 AWS SAM CLI Installation
```bash
# macOS
brew tap aws/tap
brew install aws-sam-cli

# Windows (PowerShell as Administrator)
choco install aws-sam-cli

# Verify installation
sam --version
```

### 3.2 AWS CDK Setup
```bash
# Install AWS CDK Toolkit
npm install -g aws-cdk

# Verify installation
cdk --version

# Bootstrap CDK in your AWS account
cdk bootstrap aws://ACCOUNT-NUMBER/REGION
```

### 3.3 AWS Systems Manager Session Manager Plugin
```bash
# macOS
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/mac/sessionmanager-bundle.zip" -o "sessionmanager-bundle.zip"
unzip sessionmanager-bundle.zip
sudo ./sessionmanager-bundle/install -i /usr/local/sessionmanagerplugin -b /usr/local/bin/session-manager-plugin

# Windows (PowerShell as Administrator)
choco install session-manager-plugin
```

## 4. Infrastructure as Code Setup

### 4.1 Setting up AWS CDK Project
```bash
# Create new CDK project
mkdir my-saas-app
cd my-saas-app
cdk init app --language typescript

# Install essential dependencies
npm install @aws-cdk/aws-lambda @aws-cdk/aws-apigateway @aws-cdk/aws-dynamodb
```

### 4.2 Basic CDK Stack Example
```typescript
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as apigateway from 'aws-cdk-lib/aws-apigateway';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';

export class MySaasStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // DynamoDB Table
    const table = new dynamodb.Table(this, 'SaasTable', {
      partitionKey: { name: 'id', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      removalPolicy: cdk.RemovalPolicy.DESTROY, // For development only
    });

    // Lambda Function
    const handler = new lambda.Function(this, 'SaasHandler', {
      runtime: lambda.Runtime.NODEJS_18_X,
      code: lambda.Code.fromAsset('lambda'),
      handler: 'index.handler',
      environment: {
        TABLE_NAME: table.tableName,
      },
    });

    // Grant Lambda permissions to DynamoDB
    table.grantReadWriteData(handler);

    // API Gateway
    const api = new apigateway.RestApi(this, 'SaasApi', {
      restApiName: 'SaaS Service',
      deployOptions: {
        stageName: 'dev',
      },
    });

    api.root.addMethod('ANY', new apigateway.LambdaIntegration(handler));
  }
}
```

## 5. Local Development Environment Configuration

### 5.1 Docker Setup for Local Development
```bash
# Install Docker
# macOS
brew install --cask docker

# Windows
choco install docker-desktop

# Pull necessary images
docker pull amazon/dynamodb-local
docker pull amazon/aws-sam-cli-emulator
```

### 5.2 Local Development Docker Compose
```yaml
# docker-compose.yml
version: '3.8'
services:
  dynamodb-local:
    image: amazon/dynamodb-local
    ports:
      - "8000:8000"
    command: ["-jar", "DynamoDBLocal.jar", "-sharedDb"]
    
  sam-local:
    image: amazon/aws-sam-cli-emulator
    volumes:
      - .:/app
    working_dir: /app
    depends_on:
      - dynamodb-local
```

## 6. CI/CD Pipeline Tools

### 6.1 GitHub Actions Workflow
```yaml
# .github/workflows/deploy.yml
name: Deploy SaaS Application

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
          
      - name: Install dependencies
        run: npm install
        
      - name: Build and deploy
        run: |
          npm run build
          cdk deploy --require-approval never
```

## 7. Monitoring and Debugging Tools

### 7.1 AWS X-Ray Setup
```typescript
// Add to your Lambda function
import * as aws from 'aws-sdk';
import AWSXRay from 'aws-xray-sdk-core';

const dynamodb = AWSXRay.captureAWSClient(new aws.DynamoDB.DocumentClient());
```

### 7.2 CloudWatch Logs Insights Queries
```sql
# Error monitoring query
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 20

# Performance monitoring query
fields @timestamp, @message
| filter @type = "REPORT"
| stats avg(@duration), max(@duration), min(@duration) by bin(5m)
```

## Best Practices Summary

1. **Security**
   - Use IAM roles with least privilege
   - Enable MFA for all users
   - Rotate access keys regularly
   - Use AWS Secrets Manager for sensitive data

2. **Infrastructure**
   - Use Infrastructure as Code (CDK/CloudFormation)
   - Implement proper tagging strategy
   - Use separate AWS accounts for different environments

3. **Development**
   - Follow the AWS Well-Architected Framework
   - Implement proper error handling and logging
   - Use staged deployments
   - Implement automated testing

4. **Monitoring**
   - Set up proper alerting
   - Use X-Ray for distributed tracing
   - Implement custom metrics for business KPIs
