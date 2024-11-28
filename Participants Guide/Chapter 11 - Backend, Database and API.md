# Chapter 11: Building Scalable Backend Services

## Table of Contents
1. Backend Architecture Setup
2. Database Design and Management
3. API Development and Documentation
4. Service Layer Implementation
5. Event-Driven Architecture
6. Caching Strategies

## 1. Backend Architecture Setup

### 1.1 Service Layer Setup
```typescript
// src/core/service-factory.ts
export class ServiceFactory {
  private static instance: ServiceFactory;
  private services: Map<string, any> = new Map();

  // Database connections
  private readonly dynamoClient: AWS.DynamoDB.DocumentClient;
  private readonly mongoClient: MongoClient;
  private readonly redis: Redis;

  constructor() {
    // Initialize database connections
    this.dynamoClient = new AWS.DynamoDB.DocumentClient();
    this.mongoClient = new MongoClient(process.env.MONGODB_URI!);
    this.redis = new Redis(process.env.REDIS_URL);

    // Register services
    this.registerServices();
  }

  private async registerServices() {
    // Core services
    this.services.set('userService', new UserService(this.dynamoClient));
    this.services.set('authService', new AuthService(this.dynamoClient));
    this.services.set('tenantService', new TenantService(this.dynamoClient));
    
    // Cache service
    const cacheService = new CacheService(this.redis);
    this.services.set('cacheService', cacheService);
  }

  static getInstance(): ServiceFactory {
    if (!ServiceFactory.instance) {
      ServiceFactory.instance = new ServiceFactory();
    }
    return ServiceFactory.instance;
  }
}
```

### 1.2 API Router Setup
```typescript
// src/api/router.ts
export class APIRouter {
  private readonly app: Express;
  private readonly serviceFactory: ServiceFactory;

  constructor() {
    this.app = express();
    this.serviceFactory = ServiceFactory.getInstance();
    this.setupMiddleware();
    this.setupRoutes();
  }

  private setupMiddleware() {
    // Global middleware
    this.app.use(cors());
    this.app.use(express.json());
    this.app.use(compression());
    
    // Security middleware
    this.app.use(helmet());
    this.app.use(rateLimit({
      windowMs: 15 * 60 * 1000,
      max: 100
    }));

    // Custom middleware
    this.app.use(this.authMiddleware.bind(this));
    this.app.use(this.tenantMiddleware.bind(this));
  }

  private setupRoutes() {
    // API version prefix
    const v1Router = express.Router();

    // Mount route handlers
    v1Router.use('/users', new UserRouter(this.serviceFactory).router);
    v1Router.use('/tenants', new TenantRouter(this.serviceFactory).router);
    v1Router.use('/auth', new AuthRouter(this.serviceFactory).router);

    this.app.use('/api/v1', v1Router);
  }
}
```

## 2. Database Design and Management

### 2.1 Database Manager
```typescript
// src/database/database-manager.ts
export class DatabaseManager {
  private readonly config: DatabaseConfig;
  private connections: Map<string, any> = new Map();

  constructor(config: DatabaseConfig) {
    this.config = config;
  }

  async initializeDatabases() {
    // Initialize primary database
    await this.initializePrimaryDatabase();
    
    // Initialize read replicas
    if (this.config.useReplicas) {
      await this.initializeReadReplicas();
    }
    
    // Initialize cache
    if (this.config.useCache) {
      await this.initializeCache();
    }
  }

  private async initializePrimaryDatabase() {
    switch (this.config.primaryDatabase.type) {
      case 'dynamodb':
        await this.initializeDynamoDB();
        break;
      case 'mongodb':
        await this.initializeMongoDB();
        break;
      default:
        throw new Error(`Unsupported database type: ${this.config.primaryDatabase.type}`);
    }
  }

  async executeWithRetry<T>(operation: () => Promise<T>): Promise<T> {
    const maxRetries = this.config.retryAttempts || 3;
    let lastError: Error;

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        return await operation();
      } catch (error) {
        lastError = error;
        await this.exponentialBackoff(attempt);
      }
    }

    throw lastError!;
  }
}
```

### 2.2 Data Model Manager
```typescript
// src/database/model-manager.ts
export class ModelManager {
  private readonly schemas: Map<string, Schema> = new Map();
  private readonly validators: Map<string, SchemaValidator> = new Map();

  registerModel(name: string, schema: Schema) {
    // Register schema
    this.schemas.set(name, schema);
    
    // Create validator
    this.validators.set(name, new SchemaValidator(schema));
    
    // Generate migrations if needed
    if (this.shouldMigrate(name, schema)) {
      this.generateMigration(name, schema);
    }
  }

  async validateData(modelName: string, data: any): Promise<ValidationResult> {
    const validator = this.validators.get(modelName);
    if (!validator) {
      throw new Error(`No validator found for model: ${modelName}`);
    }

    return validator.validate(data);
  }

  private shouldMigrate(name: string, newSchema: Schema): boolean {
    const currentSchema = this.getCurrentSchema(name);
    return this.schemaHasChanges(currentSchema, newSchema);
  }
}
```

## 3. API Development and Documentation

### 3.1 API Documentation Generator
```typescript
// src/api/docs-generator.ts
export class APIDocsGenerator {
  private readonly config: SwaggerConfig;
  private specs: OpenAPISpec;

  constructor(config: SwaggerConfig) {
    this.config = config;
    this.specs = this.initializeSpecs();
  }

  generateDocs() {
    // Generate base documentation
    this.generateBaseDoc();
    
    // Add routes
    this.addRoutes();
    
    // Add models
    this.addModels();
    
    // Add security schemes
    this.addSecuritySchemes();
    
    return this.specs;
  }

  private addRoutes() {
    const routes = this.scanRoutes();
    
    routes.forEach(route => {
      this.addPathItem(route);
      this.addOperations(route);
      this.addParameters(route);
      this.addResponses(route);
    });
  }

  private scanRoutes(): Route[] {
    // Scan project for route definitions
    const routeScanner = new RouteScanner();
    return routeScanner.scan(this.config.sourcePath);
  }
}
```

### 3.2 API Versioning Manager
```typescript
// src/api/version-manager.ts
export class APIVersionManager {
  private readonly versions: Map<string, RouterConfig> = new Map();

  registerVersion(version: string, config: RouterConfig) {
    this.validateVersion(version);
    this.versions.set(version, config);
  }

  getRouter(version: string): Router {
    const config = this.versions.get(version);
    if (!config) {
      throw new Error(`API version ${version} not found`);
    }

    const router = express.Router();
    
    // Apply version-specific middleware
    config.middleware.forEach(middleware => {
      router.use(middleware);
    });
    
    // Mount routes
    config.routes.forEach(route => {
      this.mountRoute(router, route);
    });
    
    return router;
  }

  private mountRoute(router: Router, route: RouteConfig) {
    const { method, path, handler, middleware = [] } = route;
    
    router[method](path, ...middleware, handler);
  }
}
```

## 4. Service Layer Implementation

### 4.1 Base Service
```typescript
// src/services/base-service.ts
export abstract class BaseService {
  protected readonly db: DatabaseManager;
  protected readonly cache: CacheService;
  protected readonly logger: Logger;

  constructor(
    db: DatabaseManager,
    cache: CacheService,
    logger: Logger
  ) {
    this.db = db;
    this.cache = cache;
    this.logger = logger;
  }

  protected async withTransaction<T>(
    operation: (transaction: Transaction) => Promise<T>
  ): Promise<T> {
    const transaction = await this.db.beginTransaction();
    
    try {
      const result = await operation(transaction);
      await transaction.commit();
      return result;
    } catch (error) {
      await transaction.rollback();
      throw error;
    }
  }

  protected async withCache<T>(
    key: string,
    operation: () => Promise<T>,
    options: CacheOptions = {}
  ): Promise<T> {
    const cached = await this.cache.get(key);
    if (cached) {
      return cached as T;
    }

    const result = await operation();
    await this.cache.set(key, result, options);
    return result;
  }
}
```

### 4.2 Error Handler
```typescript
// src/core/error-handler.ts
export class ErrorHandler {
  private readonly logger: Logger;
  private readonly metrics: MetricsService;

  constructor(logger: Logger, metrics: MetricsService) {
    this.logger = logger;
    this.metrics = metrics;
  }

  handleError(error: any): APIResponse {
    // Log error
    this.logger.error('Error occurred', {
      error,
      stack: error.stack,
      timestamp: new Date().toISOString()
    });

    // Track metric
    this.metrics.incrementCounter('errors', {
      type: error.constructor.name
    });

    // Determine response
    if (error instanceof ValidationError) {
      return this.handleValidationError(error);
    }

    if (error instanceof AuthorizationError) {
      return this.handleAuthError(error);
    }

    // Default error response
    return {
      status: 500,
      body: {
        error: 'Internal Server Error',
        message: 'An unexpected error occurred'
      }
    };
  }
}
```

## Learning Resources

### 1. Backend Development
- **Node.js and Express**
  - Node.js Documentation: https://nodejs.org/docs/
  - Express.js Guide: https://expressjs.com/guide/
  - Advanced Node.js: https://frontendmasters.com/courses/advanced-node-js/

- **TypeScript**
  - Official Documentation: https://www.typescriptlang.org/docs/
  - TypeScript Deep Dive: https://basarat.gitbook.io/typescript/

### 2. Database Management
- **DynamoDB**
  - AWS DynamoDB Documentation: https://docs.aws.amazon.com/dynamodb/
  - DynamoDB Design Patterns: https://www.youtube.com/watch?v=HaEPXoXVf2k

- **MongoDB**
  - MongoDB University: https://university.mongodb.com/
  - MongoDB Best Practices: https://www.mongodb.com/blog/channel/best-practices

### 3. API Development
- **REST API Design**
  - REST API Tutorial: https://restfulapi.net/
  - API Design Guide: https://cloud.google.com/apis/design

- **GraphQL**
  - GraphQL Documentation: https://graphql.org/learn/
  - How to GraphQL: https://www.howtographql.com/

### 4. Additional Resources
1. **YouTube Channels**
   - Traversy Media
   - Ben Awad
   - Academind

2. **Online Courses**
   - Backend Development on Udemy
   - AWS for Backend Development
   - MongoDB Complete Developer Guide

3. **Books**
   - "Clean Architecture" by Robert C. Martin
   - "Designing Data-Intensive Applications" by Martin Kleppmann
   - "Node.js Design Patterns" by Mario Casciaro

Would you like me to continue with Chapter 12, which could focus on implementing specific backend features and integrations, or would you like me to expand on any particular aspect of Chapter 11?
