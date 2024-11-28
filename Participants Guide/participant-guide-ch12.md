# Chapter 12: Advanced Backend Features and Integrations

## Table of Contents
1. Message Queue System
2. Real-time Communication
3. Search Engine Integration
4. Payment Processing
5. Email and Notification System
6. File Storage and CDN Integration

## 1. Message Queue System

### 1.1 Queue Manager
```typescript
// src/services/queue/queue-manager.ts
export class QueueManager {
  private readonly sqs: AWS.SQS;
  private readonly eventBridge: AWS.EventBridge;
  private readonly queues: Map<string, Queue>;

  async publishMessage(queueName: string, message: any): Promise<void> {
    const queue = this.queues.get(queueName);
    if (!queue) throw new Error(`Queue ${queueName} not found`);

    try {
      await this.sqs.sendMessage({
        QueueUrl: queue.url,
        MessageBody: JSON.stringify(message),
        MessageAttributes: this.buildMessageAttributes(message)
      }).promise();

      await this.trackMessageMetrics(queueName, 'published');
    } catch (error) {
      await this.handlePublishError(error, queueName, message);
      throw error;
    }
  }

  async processMessages(queueName: string, handler: MessageHandler): Promise<void> {
    const queue = this.queues.get(queueName);
    
    while (true) {
      const messages = await this.sqs.receiveMessage({
        QueueUrl: queue!.url,
        MaxNumberOfMessages: 10,
        WaitTimeSeconds: 20
      }).promise();

      await Promise.all(messages.Messages?.map(async (message) => {
        try {
          await handler(JSON.parse(message.Body));
          await this.deleteMessage(queue!.url, message.ReceiptHandle!);
        } catch (error) {
          await this.handleProcessingError(error, queue!.url, message);
        }
      }) ?? []);
    }
  }
}
```

### 1.2 Event Bus Implementation
```typescript
// src/services/events/event-bus.ts
export class EventBus {
  private readonly eventBridge: AWS.EventBridge;
  private readonly logger: Logger;

  async publishEvent(event: ApplicationEvent): Promise<void> {
    try {
      await this.eventBridge.putEvents({
        Entries: [{
          Source: event.source,
          DetailType: event.type,
          Detail: JSON.stringify(event.payload),
          EventBusName: this.getEventBusName(event)
        }]
      }).promise();

      await this.logEventPublished(event);
    } catch (error) {
      await this.handleEventPublishError(error, event);
      throw error;
    }
  }

  async subscribeToEvents(
    pattern: EventPattern,
    handler: EventHandler
  ): Promise<void> {
    const rule = await this.eventBridge.putRule({
      Name: this.generateRuleName(pattern),
      EventPattern: JSON.stringify(pattern),
      State: 'ENABLED'
    }).promise();

    await this.eventBridge.putTargets({
      Rule: rule.RuleName!,
      Targets: [{
        Id: `${rule.RuleName}-target`,
        Arn: handler.functionArn
      }]
    }).promise();
  }
}
```

## 2. Real-time Communication

### 2.1 WebSocket Manager
```typescript
// src/services/websocket/websocket-manager.ts
export class WebSocketManager {
  private readonly apiGateway: AWS.ApiGatewayManagementApi;
  private readonly dynamodb: AWS.DynamoDB.DocumentClient;
  private readonly connectionTable: string;

  async handleConnection(event: APIGatewayWebSocketEvent): Promise<void> {
    switch (event.requestContext.routeKey) {
      case '$connect':
        await this.handleConnect(event);
        break;
      case '$disconnect':
        await this.handleDisconnect(event);
        break;
      default:
        await this.handleMessage(event);
    }
  }

  async broadcastMessage(message: any, filter?: ConnectionFilter): Promise<void> {
    const connections = await this.getActiveConnections(filter);
    
    await Promise.all(
      connections.map(async (connection) => {
        try {
          await this.sendMessage(connection.id, message);
        } catch (error) {
          if (error.statusCode === 410) {
            await this.removeConnection(connection.id);
          }
        }
      })
    );
  }

  private async sendMessage(connectionId: string, message: any): Promise<void> {
    await this.apiGateway.postToConnection({
      ConnectionId: connectionId,
      Data: JSON.stringify(message)
    }).promise();
  }
}
```

## 3. Search Engine Integration

### 3.1 Search Service
```typescript
// src/services/search/search-service.ts
export class SearchService {
  private readonly elasticsearch: Client;
  private readonly indexPrefix: string;

  async indexDocument(
    index: string,
    document: any,
    options: IndexOptions = {}
  ): Promise<void> {
    const indexName = `${this.indexPrefix}_${index}`;
    
    await this.elasticsearch.index({
      index: indexName,
      body: this.prepareDocument(document),
      refresh: options.immediate ? 'wait_for' : false,
      ...this.getIndexOptions(options)
    });
  }

  async search(
    index: string,
    query: SearchQuery,
    options: SearchOptions = {}
  ): Promise<SearchResult> {
    const indexName = `${this.indexPrefix}_${index}`;
    
    const searchParams = {
      index: indexName,
      body: {
        query: this.buildElasticsearchQuery(query),
        aggs: this.buildAggregations(options.aggregations),
        sort: this.buildSortCriteria(options.sort)
      },
      ...this.getSearchOptions(options)
    };

    const response = await this.elasticsearch.search(searchParams);
    return this.processSearchResponse(response);
  }

  private buildElasticsearchQuery(query: SearchQuery): any {
    return {
      bool: {
        must: this.buildMustClauses(query),
        filter: this.buildFilterClauses(query),
        should: this.buildShouldClauses(query),
        must_not: this.buildMustNotClauses(query)
      }
    };
  }
}
```

## 4. Payment Processing

### 4.1 Payment Service
```typescript
// src/services/payment/payment-service.ts
export class PaymentService {
  private readonly stripe: Stripe;
  private readonly dynamodb: AWS.DynamoDB.DocumentClient;
  private readonly eventBus: EventBus;

  async processPayment(
    paymentIntent: PaymentIntent
  ): Promise<PaymentResult> {
    const transaction = await this.beginTransaction();
    
    try {
      // Create Stripe payment intent
      const stripeIntent = await this.stripe.paymentIntents.create({
        amount: paymentIntent.amount,
        currency: paymentIntent.currency,
        customer: await this.getOrCreateCustomer(paymentIntent.customerId),
        payment_method: paymentIntent.paymentMethodId,
        confirm: true,
        metadata: this.buildPaymentMetadata(paymentIntent)
      });

      // Record payment
      await this.recordPayment(transaction, {
        ...paymentIntent,
        stripeIntentId: stripeIntent.id,
        status: stripeIntent.status
      });

      await transaction.commit();

      // Emit payment event
      await this.eventBus.publishEvent({
        type: 'payment.completed',
        payload: {
          paymentId: paymentIntent.id,
          status: stripeIntent.status
        }
      });

      return {
        success: true,
        transactionId: stripeIntent.id,
        status: stripeIntent.status
      };
    } catch (error) {
      await transaction.rollback();
      await this.handlePaymentError(error, paymentIntent);
      throw error;
    }
  }

  async refundPayment(
    refundRequest: RefundRequest
  ): Promise<RefundResult> {
    const transaction = await this.beginTransaction();
    
    try {
      const refund = await this.stripe.refunds.create({
        payment_intent: refundRequest.paymentIntentId,
        amount: refundRequest.amount,
        reason: refundRequest.reason
      });

      await this.recordRefund(transaction, {
        ...refundRequest,
        stripeRefundId: refund.id,
        status: refund.status
      });

      await transaction.commit();

      await this.eventBus.publishEvent({
        type: 'payment.refunded',
        payload: {
          refundId: refund.id,
          status: refund.status
        }
      });

      return {
        success: true,
        refundId: refund.id,
        status: refund.status
      };
    } catch (error) {
      await transaction.rollback();
      await this.handleRefundError(error, refundRequest);
      throw error;
    }
  }
}
```

## 5. Email and Notification System

### 5.1 Notification Service
```typescript
// src/services/notification/notification-service.ts
export class NotificationService {
  private readonly ses: AWS.SES;
  private readonly sns: AWS.SNS;
  private readonly templateEngine: TemplateEngine;

  async sendEmail(
    notification: EmailNotification
  ): Promise<NotificationResult> {
    const template = await this.templateEngine.render(
      notification.template,
      notification.data
    );

    try {
      await this.ses.sendEmail({
        Destination: {
          ToAddresses: [notification.recipient]
        },
        Message: {
          Subject: {
            Data: template.subject
          },
          Body: {
            Html: {
              Data: template.html
            },
            Text: {
              Data: template.text
            }
          }
        },
        Source: this.getSourceEmail(notification)
      }).promise();

      await this.recordNotification('email', notification);
      return { success: true };
    } catch (error) {
      await this.handleEmailError(error, notification);
      throw error;
    }
  }

  async sendPushNotification(
    notification: PushNotification
  ): Promise<NotificationResult> {
    try {
      await this.sns.publish({
        TopicArn: this.getPushTopicArn(notification),
        Message: JSON.stringify({
          default: notification.message,
          APNS: this.formatAPNSMessage(notification),
          FCM: this.formatFCMMessage(notification)
        }),
        MessageStructure: 'json'
      }).promise();

      await this.recordNotification('push', notification);
      return { success: true };
    } catch (error) {
      await this.handlePushError(error, notification);
      throw error;
    }
  }
}
```

## 6. File Storage and CDN Integration

### 6.1 Storage Service
```typescript
// src/services/storage/storage-service.ts
export class StorageService {
  private readonly s3: AWS.S3;
  private readonly cloudfront: AWS.CloudFront;
  private readonly dynamodb: AWS.DynamoDB.DocumentClient;

  async uploadFile(
    file: FileUpload,
    options: UploadOptions = {}
  ): Promise<FileMetadata> {
    const key = this.generateFileKey(file);
    const metadata = this.prepareFileMetadata(file, key);

    try {
      // Upload to S3
      await this.s3.upload({
        Bucket: this.getStorageBucket(options),
        Key: key,
        Body: file.content,
        ContentType: file.contentType,
        Metadata: metadata,
        ...this.getUploadOptions(options)
      }).promise();

      // Record file metadata
      await this.recordFileMetadata(metadata);

      // Invalidate CDN cache if necessary
      if (options.invalidateCache) {
        await this.invalidateCDNCache([key]);
      }

      return {
        ...metadata,
        url: this.generateFileUrl(key, options)
      };
    } catch (error) {
      await this.handleUploadError(error, file);
      throw error;
    }
  }

  async deleteFile(
    fileId: string,
    options: DeleteOptions = {}
  ): Promise<void> {
    const metadata = await this.getFileMetadata(fileId);
    
    try {
      // Delete from S3
      await this.s3.deleteObject({
        Bucket: metadata.bucket,
        Key: metadata.key
      }).promise();

      // Delete metadata
      await this.deleteFileMetadata(fileId);

      // Invalidate CDN cache if necessary
      if (options.invalidateCache) {
        await this.invalidateCDNCache([metadata.key]);
      }
    } catch (error) {
      await this.handleDeleteError(error, fileId);
      throw error;
    }
  }

  private async invalidateCDNCache(paths: string[]): Promise<void> {
    await this.cloudfront.createInvalidation({
      DistributionId: this.getDistributionId(),
      InvalidationBatch: {
        CallerReference: Date.now().toString(),
        Paths: {
          Quantity: paths.length,
          Items: paths.map(path => `/${path}`)
        }
      }
    }).promise();
  }
}
```

Would you like me to continue with Chapter 13, which could focus on implementing advanced monitoring, logging, and observability features, or would you like me to expand on any particular aspect of Chapter 12?