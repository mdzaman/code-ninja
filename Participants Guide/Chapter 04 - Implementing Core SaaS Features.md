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


# Pricing Implementation

## 1. Pricing Tiers Definition
```typescript
// src/config/pricing-tiers.ts
export const PRICING_TIERS = {
  INNOVATOR: {
    id: 'innovator',
    name: 'Innovator',
    price: 0,
    stripePriceId: null, // Free tier doesn't need Stripe integration
    limits: {
      apiCalls: 1000,
      storage: 500, // MB
      users: 2,
      features: {
        basicAnalytics: true,
        customDomain: false,
        supportLevel: 'community'
      }
    }
  },
  STARTER: {
    id: 'starter',
    name: 'Starter',
    price: 9.99,
    stripePriceId: 'price_starter_monthly', // Replace with actual Stripe Price ID
    limits: {
      apiCalls: 10000,
      storage: 5000, // MB
      users: 10,
      features: {
        basicAnalytics: true,
        advancedAnalytics: true,
        customDomain: true,
        supportLevel: 'email'
      }
    }
  },
  ENHANCED: {
    id: 'enhanced',
    name: 'Enhanced',
    price: 99.99,
    stripePriceId: 'price_enhanced_monthly', // Replace with actual Stripe Price ID
    limits: {
      apiCalls: 100000,
      storage: 50000, // MB
      users: 'unlimited',
      features: {
        basicAnalytics: true,
        advancedAnalytics: true,
        customDomain: true,
        supportLevel: 'priority',
        customBranding: true,
        apiWebhooks: true,
        sla: '99.9%'
      }
    }
  }
};

// Feature gates based on pricing tiers
export const TIER_FEATURES = {
  canAccessAdvancedAnalytics: (tier: string) => {
    return ['starter', 'enhanced'].includes(tier);
  },
  canUseCustomDomain: (tier: string) => {
    return ['starter', 'enhanced'].includes(tier);
  },
  canUseApiWebhooks: (tier: string) => {
    return tier === 'enhanced';
  }
};
```

## 2. Billing Service Implementation
```typescript
// src/services/billing-service.ts
import Stripe from 'stripe';
import { PRICING_TIERS } from '../config/pricing-tiers';

export class BillingService {
  private readonly stripe: Stripe;
  private readonly docClient: DynamoDB.DocumentClient;

  constructor() {
    this.stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
      apiVersion: '2023-10-16'
    });
    this.docClient = new DynamoDB.DocumentClient();
  }

  async createSubscription(tenantId: string, tier: string, email: string) {
    // Return early for Innovator tier (free)
    if (tier === 'innovator') {
      await this.createFreeTierSubscription(tenantId);
      return;
    }

    const pricingTier = PRICING_TIERS[tier.toUpperCase()];
    if (!pricingTier) throw new Error('Invalid pricing tier');

    // Create or get customer
    const customer = await this.stripe.customers.create({
      email,
      metadata: { tenantId }
    });

    // Create subscription
    const subscription = await this.stripe.subscriptions.create({
      customer: customer.id,
      items: [{
        price: pricingTier.stripePriceId
      }],
      metadata: { tenantId }
    });

    // Store subscription info
    await this.storeSubscription(tenantId, {
      subscriptionId: subscription.id,
      customerId: customer.id,
      tier,
      status: subscription.status,
      currentPeriodEnd: new Date(subscription.current_period_end * 1000).toISOString()
    });

    return subscription;
  }

  private async createFreeTierSubscription(tenantId: string) {
    await this.storeSubscription(tenantId, {
      tier: 'innovator',
      status: 'active',
      currentPeriodEnd: null // Unlimited for free tier
    });
  }

  async upgradeTier(tenantId: string, newTier: string) {
    const currentSubscription = await this.getSubscription(tenantId);
    
    // If upgrading from free tier
    if (currentSubscription.tier === 'innovator') {
      const tenant = await this.getTenant(tenantId);
      return this.createSubscription(tenantId, newTier, tenant.email);
    }

    // Update existing Stripe subscription
    const subscription = await this.stripe.subscriptions.retrieve(
      currentSubscription.subscriptionId!
    );

    await this.stripe.subscriptions.update(
      currentSubscription.subscriptionId!,
      {
        items: [{
          id: subscription.items.data[0].id,
          price: PRICING_TIERS[newTier.toUpperCase()].stripePriceId
        }],
        proration_behavior: 'always_invoice'
      }
    );

    // Update subscription in database
    await this.updateSubscription(tenantId, { tier: newTier });
  }
}
```

## 3. Usage Tracking and Limits
```typescript
// src/services/usage-tracking-service.ts
export class UsageTrackingService {
  private readonly docClient: DynamoDB.DocumentClient;

  async trackApiUsage(tenantId: string, endpoint: string) {
    const tenant = await this.getTenant(tenantId);
    const tierLimits = PRICING_TIERS[tenant.subscription.tier.toUpperCase()].limits;

    // Get current usage
    const usage = await this.getCurrentMonthUsage(tenantId);

    // Check if usage exceeds tier limits
    if (usage.apiCalls >= tierLimits.apiCalls) {
      throw new Error('API call limit exceeded for current tier');
    }

    // Track usage
    await this.incrementUsage(tenantId, 'apiCalls');
  }

  async checkStorageLimit(tenantId: string, fileSize: number) {
    const tenant = await this.getTenant(tenantId);
    const tierLimits = PRICING_TIERS[tenant.subscription.tier.toUpperCase()].limits;

    const currentStorage = await this.getCurrentStorageUsage(tenantId);
    
    if (currentStorage + fileSize > tierLimits.storage) {
      throw new Error('Storage limit exceeded for current tier');
    }
  }
}
```

## 4. Feature Access Control
```typescript
// src/middleware/feature-gate.ts
export const featureGate = (feature: string) => {
  return async (event: APIGatewayProxyEvent) => {
    const tenantId = event.requestContext.authorizer?.tenantId;
    const tenant = await getTenant(tenantId);
    
    if (!TIER_FEATURES[feature](tenant.subscription.tier)) {
      throw new Error(`Feature not available in ${tenant.subscription.tier} tier`);
    }
  };
};

// Usage in Lambda handler
export const handler = middy(baseHandler)
  .use(authMiddleware())
  .use(featureGate('canUseApiWebhooks'));
```

## 5. Pricing Display Component
```typescript
// src/components/PricingTable.tsx
import React from 'react';
import { PRICING_TIERS } from '../config/pricing-tiers';

export const PricingTable: React.FC = () => {
  return (
    <div className="grid grid-cols-1 md:grid-cols-3 gap-8 p-4">
      {Object.values(PRICING_TIERS).map((tier) => (
        <div key={tier.id} className="border rounded-lg p-6 shadow-lg">
          <h2 className="text-2xl font-bold mb-4">{tier.name}</h2>
          <p className="text-4xl font-bold mb-6">
            ${tier.price}
            <span className="text-sm font-normal">/month</span>
          </p>
          
          <ul className="space-y-3 mb-6">
            <li>
              <strong>API Calls:</strong> {tier.limits.apiCalls.toLocaleString()}
            </li>
            <li>
              <strong>Storage:</strong> {(tier.limits.storage / 1000).toFixed(1)}GB
            </li>
            <li>
              <strong>Users:</strong> {tier.limits.users}
            </li>
            {Object.entries(tier.limits.features).map(([feature, enabled]) => (
              <li key={feature} className={enabled ? 'text-green-600' : 'text-gray-400'}>
                {enabled ? '✓' : '✗'} {formatFeatureName(feature)}
              </li>
            ))}
          </ul>
          
          <button 
            className="w-full bg-blue-600 text-white py-2 rounded-lg hover:bg-blue-700"
            onClick={() => handleSubscription(tier.id)}
          >
            {tier.price === 0 ? 'Start Free' : 'Subscribe Now'}
          </button>
        </div>
      ))}
    </div>
  );
};
```

## 6. Subscription Management API
```typescript
// src/handlers/subscription-api.ts
export const getSubscriptionDetails = async (event: APIGatewayProxyEvent) => {
  const tenantId = event.requestContext.authorizer?.tenantId;
  const billingService = new BillingService();
  
  const subscription = await billingService.getSubscription(tenantId);
  const usage = await new UsageTrackingService().getCurrentMonthUsage(tenantId);
  
  const tier = PRICING_TIERS[subscription.tier.toUpperCase()];
  
  return {
    statusCode: 200,
    body: JSON.stringify({
      subscription: {
        tier: subscription.tier,
        status: subscription.status,
        currentPeriodEnd: subscription.currentPeriodEnd,
        usage: {
          apiCalls: {
            used: usage.apiCalls,
            limit: tier.limits.apiCalls,
            percentage: (usage.apiCalls / tier.limits.apiCalls) * 100
          },
          storage: {
            used: usage.storage,
            limit: tier.limits.storage,
            percentage: (usage.storage / tier.limits.storage) * 100
          }
        }
      }
    })
  };
};
```

## 7. Usage Dashboard Components
```typescript
// src/components/UsageDashboard.tsx
import React from 'react';
import { UsageChart } from './UsageChart';
import { FeaturesList } from './FeaturesList';

export const UsageDashboard: React.FC<{ subscription: any }> = ({ subscription }) => {
  return (
    <div className="p-6">
      <div className="mb-8">
        <h2 className="text-2xl font-bold mb-4">Current Usage</h2>
        <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
          <UsageChart
            title="API Calls"
            used={subscription.usage.apiCalls.used}
            limit={subscription.usage.apiCalls.limit}
            percentage={subscription.usage.apiCalls.percentage}
          />
          <UsageChart
            title="Storage"
            used={subscription.usage.storage.used}
            limit={subscription.usage.storage.limit}
            percentage={subscription.usage.storage.percentage}
          />
        </div>
      </div>

      <div>
        <h2 className="text-2xl font-bold mb-4">Available Features</h2>
        <FeaturesList tier={subscription.tier} />
      </div>

      {subscription.tier !== 'enhanced' && (
        <div className="mt-8">
          <h3 className="text-xl font-bold mb-2">Need More?</h3>
          <p className="mb-4">Upgrade your plan to get access to more features and higher limits.</p>
          <button 
            className="bg-blue-600 text-white px-6 py-2 rounded-lg hover:bg-blue-700"
            onClick={() => handleUpgrade()}
          >
            Upgrade Plan
          </button>
        </div>
      )}
    </div>
  );
};
```

This implementation provides a complete pricing system with:
- Three distinct tiers with clear feature differentiation
- Usage tracking and enforcement of tier limits
- Upgrade/downgrade capabilities
- Billing integration with Stripe
- User-friendly pricing display
- Usage monitoring dashboard
- Feature access control based on tier

Would you like me to explain any specific part in more detail or move on to the next chapter focusing on deployment and scaling strategies?
