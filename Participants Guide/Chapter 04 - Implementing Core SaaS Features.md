# Chapter 4: Implementing Core SaaS Features

## Table of Contents
1. Multi-Tenancy Implementation
2. User Management & Authentication
3. Billing Integration
4. API Development
5. Rate Limiting & Quotas
6. Feature Flags & Tenant Configuration

## 1. Multi-Tenancy Implementation

### 1.1 DynamoDB Single-Table Design for Multi-Tenancy
```typescript
// infrastructure/database/table-design.ts
export const TableSchema = {
  // Primary keys for different entity types
  KEYS: {
    tenant: (tenantId: string) => ({ PK: `TENANT#${tenantId}`, SK: `#METADATA` }),
    user: (tenantId: string, userId: string) => ({ 
      PK: `TENANT#${tenantId}`, 
      SK: `USER#${userId}`,
      GSI1PK: `USER#${userId}`,
      GSI1SK: `TENANT#${tenantId}`
    }),
    subscription: (tenantId: string) => ({
      PK: `TENANT#${tenantId}`,
      SK: `SUB#ACTIVE`,
      GSI1PK: `SUB#ACTIVE`,
      GSI1SK: `TENANT#${tenantId}`
    })
  },

  // Access patterns
  INDEXES: {
    primary: {
      pk: 'PK',
      sk: 'SK'
    },
    gsi1: {
      pk: 'GSI1PK',
      sk: 'GSI1SK',
      name: 'GSI1'
    }
  }
};

// Example tenant isolation middleware
export const tenantIsolation = async (event: APIGatewayProxyEvent) => {
  const tenantId = event.requestContext.authorizer?.tenantId;
  if (!tenantId) throw new Error('Tenant context missing');
  
  // Validate tenant access
  const tenant = await getTenant(tenantId);
  if (!tenant.active) throw new Error('Tenant inactive');
  
  return {
    tenantId,
    tenant
  };
};
```

### 1.2 Tenant Service Implementation
```typescript
// src/services/tenant-service.ts
import { DynamoDB } from 'aws-sdk';
import { TableSchema } from '../database/table-design';

export class TenantService {
  private readonly docClient: DynamoDB.DocumentClient;
  
  constructor() {
    this.docClient = new DynamoDB.DocumentClient();
  }

  async createTenant(name: string, plan: string): Promise<string> {
    const tenantId = generateUUID();
    const timestamp = new Date().toISOString();

    await this.docClient.put({
      TableName: process.env.TABLE_NAME!,
      Item: {
        ...TableSchema.KEYS.tenant(tenantId),
        name,
        plan,
        createdAt: timestamp,
        updatedAt: timestamp,
        status: 'ACTIVE',
        settings: getDefaultSettings(plan)
      }
    }).promise();

    return tenantId;
  }

  async getTenantSettings(tenantId: string) {
    const result = await this.docClient.get({
      TableName: process.env.TABLE_NAME!,
      Key: TableSchema.KEYS.tenant(tenantId)
    }).promise();

    return result.Item;
  }
}
```

## 2. User Management & Authentication

### 2.1 Cognito Setup with Custom Attributes
```typescript
// infrastructure/stacks/auth-stack.ts
import * as cognito from 'aws-cdk-lib/aws-cognito';

export class AuthStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const userPool = new cognito.UserPool(this, 'SaasUserPool', {
      userPoolName: 'saas-user-pool',
      selfSignUpEnabled: true,
      signInAliases: {
        email: true
      },
      standardAttributes: {
        email: {
          required: true,
          mutable: true
        }
      },
      customAttributes: {
        tenantId: new cognito.StringAttribute({ mutable: false }),
        role: new cognito.StringAttribute({ mutable: true })
      },
      passwordPolicy: {
        minLength: 12,
        requireLowercase: true,
        requireUppercase: true,
        requireDigits: true,
        requireSymbols: true
      },
      accountRecovery: cognito.AccountRecovery.EMAIL_ONLY
    });

    // Add app client
    const userPoolClient = userPool.addClient('SaasAppClient', {
      oAuth: {
        flows: {
          authorizationCodeGrant: true
        },
        scopes: [
          cognito.OAuthScope.EMAIL,
          cognito.OAuthScope.OPENID,
          cognito.OAuthScope.PROFILE
        ],
        callbackUrls: ['http://localhost:3000/callback']
      }
    });
  }
}
```

### 2.2 User Management Service
```typescript
// src/services/user-service.ts
import { CognitoIdentityServiceProvider } from 'aws-sdk';
import { TenantService } from './tenant-service';

export class UserService {
  private readonly cognito: CognitoIdentityServiceProvider;
  private readonly tenantService: TenantService;

  constructor() {
    this.cognito = new CognitoIdentityServiceProvider();
    this.tenantService = new TenantService();
  }

  async createUser(email: string, tenantId: string, role: string) {
    // Validate tenant
    const tenant = await this.tenantService.getTenantSettings(tenantId);
    if (!tenant) throw new Error('Invalid tenant');

    // Create Cognito user
    const response = await this.cognito.adminCreateUser({
      UserPoolId: process.env.USER_POOL_ID!,
      Username: email,
      UserAttributes: [
        { Name: 'email', Value: email },
        { Name: 'custom:tenantId', Value: tenantId },
        { Name: 'custom:role', Value: role }
      ]
    }).promise();

    // Store user in DynamoDB
    await this.storeUserInDB(email, tenantId, role, response.User?.Username!);

    return response.User;
  }

  private async storeUserInDB(email: string, tenantId: string, role: string, userId: string) {
    const timestamp = new Date().toISOString();

    await this.docClient.put({
      TableName: process.env.TABLE_NAME!,
      Item: {
        ...TableSchema.KEYS.user(tenantId, userId),
        email,
        role,
        createdAt: timestamp,
        updatedAt: timestamp,
        status: 'ACTIVE'
      }
    }).promise();
  }
}
```

## 3. Billing Integration

### 3.1 Stripe Integration Setup
```typescript
// src/services/billing-service.ts
import Stripe from 'stripe';

export class BillingService {
  private readonly stripe: Stripe;

  constructor() {
    this.stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
      apiVersion: '2023-10-16'
    });
  }

  async createSubscription(tenantId: string, plan: string, email: string) {
    // Create or get customer
    const customer = await this.stripe.customers.create({
      email,
      metadata: {
        tenantId
      }
    });

    // Create subscription
    const subscription = await this.stripe.subscriptions.create({
      customer: customer.id,
      items: [{
        price: getPlanPriceId(plan)
      }],
      metadata: {
        tenantId
      }
    });

    // Store subscription info in DynamoDB
    await this.storeSubscription(tenantId, subscription.id, plan);

    return subscription;
  }

  async handleWebhook(event: APIGatewayProxyEvent) {
    const sig = event.headers['stripe-signature'];
    const webhookSecret = process.env.STRIPE_WEBHOOK_SECRET!;

    try {
      const stripeEvent = this.stripe.webhooks.constructEvent(
        event.body!,
        sig!,
        webhookSecret
      );

      switch (stripeEvent.type) {
        case 'invoice.payment_succeeded':
          await this.handlePaymentSuccess(stripeEvent.data.object);
          break;
        case 'invoice.payment_failed':
          await this.handlePaymentFailure(stripeEvent.data.object);
          break;
      }

      return {
        statusCode: 200,
        body: JSON.stringify({ received: true })
      };
    } catch (err) {
      console.error('Webhook Error:', err.message);
      throw err;
    }
  }
}
```

### 3.2 Usage Tracking and Billing Metrics
```typescript
// src/services/usage-tracking-service.ts
export class UsageTrackingService {
  private readonly cloudWatch: CloudWatch;

  constructor() {
    this.cloudWatch = new CloudWatch();
  }

  async trackApiUsage(tenantId: string, endpoint: string) {
    await this.cloudWatch.putMetricData({
      Namespace: 'SaaS/Usage',
      MetricData: [{
        MetricName: 'ApiCalls',
        Value: 1,
        Unit: 'Count',
        Dimensions: [
          { Name: 'TenantId', Value: tenantId },
          { Name: 'Endpoint', Value: endpoint }
        ]
      }]
    }).promise();
  }

  async getMonthlyUsage(tenantId: string): Promise<Usage> {
    const endTime = new Date();
    const startTime = new Date();
    startTime.setMonth(startTime.getMonth() - 1);

    const response = await this.cloudWatch.getMetricStatistics({
      Namespace: 'SaaS/Usage',
      MetricName: 'ApiCalls',
      Dimensions: [{ Name: 'TenantId', Value: tenantId }],
      StartTime: startTime,
      EndTime: endTime,
      Period: 2592000, // 30 days in seconds
      Statistics: ['Sum']
    }).promise();

    return processUsageData(response);
  }
}
```

## 4. Rate Limiting & Quotas

### 4.1 API Gateway Usage Plans
```typescript
// infrastructure/stacks/api-stack.ts
const api = new apigateway.RestApi(this, 'SaasApi', {
  restApiName: 'saas-api'
});

const usagePlan = api.addUsagePlan('UsagePlan', {
  name: 'Standard',
  throttle: {
    rateLimit: 10,
    burstLimit: 20
  },
  quota: {
    limit: 1000,
    period: apigateway.Period.MONTH
  }
});

// Create API key per tenant
const createApiKey = (tenantId: string) => {
  const key = api.addApiKey(`TenantKey-${tenantId}`);
  usagePlan.addApiKey(key);
  return key;
};
```

### 4.2 Lambda-Based Rate Limiting
```typescript
// src/middleware/rate-limit.ts
import { RateLimiter } from 'redis-rate-limiter';
import { createClient } from 'redis';

export const rateLimiter = new RateLimiter({
  redis: createClient({
    url: process.env.REDIS_URL
  }),
  key: (req) => `${req.tenantId}:${req.path}`,
  rate: '100/hour'
});

export const rateLimitMiddleware = async (event: APIGatewayProxyEvent) => {
  const tenant = event.requestContext.authorizer?.tenantId;
  const path = event.path;

  try {
    await rateLimiter.pass({
      tenantId: tenant,
      path
    });
  } catch (error) {
    throw new Error('Rate limit exceeded');
  }
};
```

## 5. Feature Flags & Tenant Configuration

### 5.1 Feature Flag Service
```typescript
// src/services/feature-flag-service.ts
export class FeatureFlagService {
  private readonly docClient: DynamoDB.DocumentClient;

  constructor() {
    this.docClient = new DynamoDB.DocumentClient();
  }

  async getTenantFeatures(tenantId: string): Promise<Features> {
    const tenant = await this.docClient.get({
      TableName: process.env.TABLE_NAME!,
      Key: TableSchema.KEYS.tenant(tenantId)
    }).promise();

    return {
      ...getDefaultFeatures(),
      ...tenant.Item?.features
    };
  }

  async isFeatureEnabled(tenantId: string, featureKey: string): Promise<boolean> {
    const features = await this.getTenantFeatures(tenantId);
    return features[featureKey]?.enabled ?? false;
  }

  async updateFeature(tenantId: string, featureKey: string, enabled: boolean) {
    await this.docClient.update({
      TableName: process.env.TABLE_NAME!,
      Key: TableSchema.KEYS.tenant(tenantId),
      UpdateExpression: 'SET features.#key = :value',
      ExpressionAttributeNames: {
        '#key': featureKey
      },
      ExpressionAttributeValues: {
        ':value': { enabled }
      }
    }).promise();
  }
}
```

### 5.2 Tenant Configuration Management
```typescript
// src/services/tenant-config-service.ts
export class TenantConfigService {
  private readonly docClient: DynamoDB.DocumentClient;
  private readonly cache: NodeCache;

  constructor() {
    this.docClient = new DynamoDB.DocumentClient();
    this.cache = new NodeCache({ stdTTL: 300 }); // 5-minute cache
  }

  async getConfig(tenantId: string): Promise<TenantConfig> {
    const cacheKey = `config:${tenantId}`;
    let config = this.cache.get<TenantConfig>(cacheKey);

    if (!config) {
      const result = await this.docClient.get({
        TableName: process.env.TABLE_NAME!,
        Key: TableSchema.KEYS.tenant(tenantId)
      }).promise();

      config = result.Item?.config;
      this.cache.set(cacheKey, config);
    }

    return config;
  }

  async updateConfig(tenantId: string, updates: Partial<TenantConfig>) {
    await this.docClient.update({
      TableName: process.env.TABLE_NAME!,
      Key: TableSchema.KEYS.tenant(tenantId),
      UpdateExpression: 'SET config = :config',
      ExpressionAttributeValues: {
        ':config': {
          ...(await this.getConfig(tenantId)),
          ...updates
        }
      }
    }).promise();

    this.cache.del(`config:${tenantId}`);
  }
}
```

This chapter covers the essential SaaS features implementation. Would you like me to proceed with Chapter 5, which could focus on deployment strategies and CI/CD pipelines, or would you like me to expand on any particular aspect of Chapter 4?
