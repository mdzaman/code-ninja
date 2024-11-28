# Chapter 3: Building Cost-Effective and Secure Serverless Applications

## Table of Contents
1. Application Architecture and Design
2. Development Workflow and Tools
3. Security Implementation
4. Cost Optimization Strategies
5. Testing Framework Setup
6. Performance Optimization
7. Monitoring and Observability

## 1. Application Architecture and Design

### 1.1 Serverless Architecture Best Practices
```typescript
// infrastructure/stacks/app-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as apigateway from 'aws-cdk-lib/aws-apigateway';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as cognito from 'aws-cdk-lib/aws-cognito';

export class ServerlessAppStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Single-table DynamoDB design for cost optimization
    const table = new dynamodb.Table(this, 'AppTable', {
      partitionKey: { name: 'PK', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'SK', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      timeToLiveAttribute: 'TTL',
      // Enable point-in-time recovery for production
      pointInTimeRecovery: true,
    });

    // Add GSIs for access patterns
    table.addGlobalSecondaryIndex({
      indexName: 'GSI1',
      partitionKey: { name: 'GSI1PK', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'GSI1SK', type: dynamodb.AttributeType.STRING },
    });

    // Create Cognito User Pool with enhanced security
    const userPool = new cognito.UserPool(this, 'UserPool', {
      selfSignUpEnabled: true,
      passwordPolicy: {
        minLength: 12,
        requireLowercase: true,
        requireUppercase: true,
        requireDigits: true,
        requireSymbols: true,
      },
      accountRecovery: cognito.AccountRecovery.EMAIL_ONLY,
      removalPolicy: cdk.RemovalPolicy.RETAIN,
    });

    // API Lambda with optimized settings
    const apiHandler = new lambda.Function(this, 'ApiHandler', {
      runtime: lambda.Runtime.NODEJS_18_X,
      code: lambda.Code.fromAsset('src/lambda'),
      handler: 'api.handler',
      memorySize: 1024, // Optimize based on testing
      timeout: cdk.Duration.seconds(29), // API Gateway limit is 30s
      environment: {
        TABLE_NAME: table.tableName,
        USER_POOL_ID: userPool.userPoolId,
      },
      tracing: lambda.Tracing.ACTIVE, // X-Ray tracing
      bundling: {
        minify: true, // Reduce cold start time
        sourceMap: true, // Enable proper stack traces
      },
    });
  }
}
```

### 1.2 Testing Tools Setup
```javascript
// jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  collectCoverage: true,
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
    },
  },
  setupFiles: ['./test/setup.ts'],
};
```

```typescript
// test/setup.ts
import { DynamoDB } from 'aws-sdk';
import * as AWSMock from 'aws-sdk-mock';

// Mock AWS services for testing
AWSMock.setSDKInstance(DynamoDB);
```

## 2. Development Workflow and Tools

### 2.1 Local Development Setup
```yaml
# docker-compose.yml for local development
version: '3.8'
services:
  dynamodb-local:
    image: amazon/dynamodb-local
    ports:
      - "8000:8000"
    command: ["-jar", "DynamoDBLocal.jar", "-sharedDb"]
    volumes:
      - dynamodb_data:/home/dynamodblocal/data

  localstack:
    image: localstack/localstack
    ports:
      - "4566:4566"
    environment:
      - SERVICES=s3,sqs,lambda
      - DEBUG=1
      - DATA_DIR=/tmp/localstack/data
    volumes:
      - localstack_data:/tmp/localstack

volumes:
  dynamodb_data:
  localstack_data:
```

### 2.2 Serverless Testing Framework
```typescript
// test/api.test.ts
import { APIGatewayProxyEvent } from 'aws-lambda';
import { handler } from '../src/lambda/api';

describe('API Handler Tests', () => {
  beforeEach(() => {
    // Setup test environment
  });

  test('should handle user creation', async () => {
    const event: APIGatewayProxyEvent = {
      httpMethod: 'POST',
      path: '/users',
      body: JSON.stringify({
        email: 'test@example.com',
        name: 'Test User',
      }),
      headers: {},
      multiValueHeaders: {},
      isBase64Encoded: false,
      pathParameters: null,
      queryStringParameters: null,
      multiValueQueryStringParameters: null,
      stageVariables: null,
      requestContext: {} as any,
      resource: '',
    };

    const response = await handler(event);
    expect(response.statusCode).toBe(201);
  });
});
```

## 3. Security Implementation

### 3.1 Lambda Authorization Layer
```typescript
// src/lambda/layers/auth/index.ts
import { APIGatewayProxyEvent } from 'aws-lambda';
import { CognitoJwtVerifier } from 'aws-jwt-verify';

export const authorizer = async (event: APIGatewayProxyEvent) => {
  const verifier = CognitoJwtVerifier.create({
    userPoolId: process.env.USER_POOL_ID!,
    clientId: process.env.CLIENT_ID!,
    tokenUse: 'access',
  });

  try {
    const token = event.headers.Authorization?.split(' ')[1];
    if (!token) throw new Error('No token provided');

    const payload = await verifier.verify(token);
    return payload;
  } catch (error) {
    throw new Error('Unauthorized');
  }
};
```

### 3.2 API Security Headers
```typescript
// src/lambda/middleware/security-headers.ts
export const securityHeaders = {
  'Strict-Transport-Security': 'max-age=31536000; includeSubDomains',
  'Content-Security-Policy': "default-src 'self'",
  'X-Frame-Options': 'DENY',
  'X-Content-Type-Options': 'nosniff',
  'Referrer-Policy': 'strict-origin-when-cross-origin',
  'X-XSS-Protection': '1; mode=block',
};
```

## 4. Cost Optimization Strategies

### 4.1 DynamoDB Optimization
```typescript
// src/lambda/data/repository.ts
import { DynamoDB } from 'aws-sdk';
import { DocumentClient } from 'aws-sdk/clients/dynamodb';

export class Repository {
  private readonly client: DocumentClient;
  private readonly tableName: string;

  constructor() {
    this.client = new DynamoDB.DocumentClient();
    this.tableName = process.env.TABLE_NAME!;
  }

  async batchGet(keys: string[]) {
    // Implement exponential backoff for batch operations
    const results = [];
    for (let i = 0; i < keys.length; i += 100) {
      const batch = keys.slice(i, i + 100);
      const response = await this.client.batchGet({
        RequestItems: {
          [this.tableName]: {
            Keys: batch.map(key => ({ PK: key, SK: key })),
          },
        },
      }).promise();
      results.push(...(response.Responses?.[this.tableName] || []));
    }
    return results;
  }
}
```

### 4.2 Lambda Performance Optimization
```typescript
// src/lambda/api.ts
import middy from '@middy/core';
import correlation from '@middy/http-cors';
import errorHandler from '@middy/http-error-handler';
import validator from '@middy/validator';

const baseHandler = async (event) => {
  // Implementation
};

// Optimize Lambda with middleware
export const handler = middy(baseHandler)
  .use(correlation())
  .use(validator({ inputSchema }))
  .use(errorHandler());
```

## 5. Monitoring and Observability

### 5.1 CloudWatch Metrics Setup
```typescript
// src/lambda/monitoring/metrics.ts
import { CloudWatch } from 'aws-sdk';

export class Metrics {
  private cloudwatch: CloudWatch;

  constructor() {
    this.cloudwatch = new CloudWatch();
  }

  async recordMetric(name: string, value: number, unit: string) {
    await this.cloudwatch.putMetricData({
      Namespace: 'ServerlessApp',
      MetricData: [{
        MetricName: name,
        Value: value,
        Unit: unit,
        Timestamp: new Date(),
      }],
    }).promise();
  }
}
```

### 5.2 X-Ray Tracing Implementation
```typescript
// src/lambda/tracing/tracer.ts
import * as AWSXRay from 'aws-xray-sdk-core';
import { DynamoDB, S3 } from 'aws-sdk';

// Wrap AWS SDK clients with X-Ray
export const dynamodb = AWSXRay.captureAWSClient(new DynamoDB.DocumentClient());
export const s3 = AWSXRay.captureAWSClient(new S3());

// Create custom subsegments
export const trace = async (name: string, fn: () => Promise<any>) => {
  const segment = AWSXRay.getSegment();
  const subsegment = segment.addNewSubsegment(name);
  
  try {
    const result = await fn();
    subsegment.close();
    return result;
  } catch (error) {
    subsegment.addError(error);
    subsegment.close();
    throw error;
  }
};
```

## Best Practices Summary

1. **Cost Optimization**
   - Use pay-per-request pricing for low-traffic applications
   - Implement proper caching strategies
   - Optimize Lambda memory settings through testing
   - Use single-table DynamoDB design

2. **Security**
   - Implement proper authentication and authorization
   - Use security headers
   - Enable AWS WAF for API Gateway
   - Implement proper error handling and logging
   - Use AWS Secrets Manager for sensitive data

3. **Performance**
   - Optimize Lambda cold starts
   - Implement proper connection pooling
   - Use compression where appropriate
   - Implement caching strategies

4. **Testing**
   - Write unit tests for all Lambda functions
   - Implement integration tests
   - Use local development environment
   - Implement proper error handling

5. **Monitoring**
   - Use X-Ray for tracing
   - Implement custom metrics
   - Set up proper alerting
   - Monitor costs regularly

## Required Tools for Development

1. **Local Development**
   - Docker Desktop
   - LocalStack
   - DynamoDB Local
   - AWS SAM CLI

2. **Testing**
   - Jest
   - AWS SDK Mock
   - Postman/Thunder Client
   - Artillery (load testing)

3. **Monitoring**
   - AWS X-Ray
   - CloudWatch
   - Grafana (optional)
   - Lumigo/Thundra (optional)

4. **Security**
   - OWASP ZAP
   - AWS IAM Policy Simulator
   - git-secrets
   - AWS Security Hub

5. **Development**
   - TypeScript
   - ESLint
   - Prettier
   - Husky (git hooks)
