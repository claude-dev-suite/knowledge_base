# NestJS Providers

> Official Documentation: https://docs.nestjs.com/providers

## Overview

Providers are the fundamental building blocks of NestJS dependency injection (DI) system. They encapsulate reusable logic and can be injected into controllers, other providers, or modules. Services, repositories, factories, helpers, and any class decorated with `@Injectable()` can be providers.

The DI container manages provider instantiation, lifecycle, and injection, promoting loose coupling and testability.

---

## Services and @Injectable

The `@Injectable()` decorator marks a class as a provider that can be managed by the NestJS IoC container.

### Basic Service Definition

```typescript
import { Injectable } from '@nestjs/common';

@Injectable()
export class UsersService {
  private users: User[] = [];

  findAll(): User[] {
    return this.users;
  }

  findOne(id: number): User | undefined {
    return this.users.find(user => user.id === id);
  }

  create(createUserDto: CreateUserDto): User {
    const newUser = {
      id: Date.now(),
      ...createUserDto,
      createdAt: new Date(),
    };
    this.users.push(newUser);
    return newUser;
  }

  update(id: number, updateUserDto: UpdateUserDto): User {
    const userIndex = this.users.findIndex(user => user.id === id);
    if (userIndex === -1) {
      throw new NotFoundException(`User #${id} not found`);
    }
    this.users[userIndex] = { ...this.users[userIndex], ...updateUserDto };
    return this.users[userIndex];
  }

  remove(id: number): void {
    const userIndex = this.users.findIndex(user => user.id === id);
    if (userIndex === -1) {
      throw new NotFoundException(`User #${id} not found`);
    }
    this.users.splice(userIndex, 1);
  }
}
```

### Registering Providers in a Module

```typescript
import { Module } from '@nestjs/common';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

@Module({
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService], // Export for use in other modules
})
export class UsersModule {}
```

---

## Dependency Injection

### Constructor-Based Injection (Recommended)

```typescript
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { LoggerService } from './logger.service';

@Injectable()
export class UsersService {
  constructor(
    private readonly configService: ConfigService,
    private readonly logger: LoggerService,
  ) {}

  findAll(): User[] {
    this.logger.log('Fetching all users');
    const limit = this.configService.get<number>('USERS_LIMIT', 100);
    return this.users.slice(0, limit);
  }
}
```

### Controller Injection

```typescript
import { Controller, Get, Post, Body, Param } from '@nestjs/common';
import { UsersService } from './users.service';

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  findAll(): User[] {
    return this.usersService.findAll();
  }

  @Post()
  create(@Body() createUserDto: CreateUserDto): User {
    return this.usersService.create(createUserDto);
  }
}
```

### Token-Based Injection

```typescript
import { Injectable, Inject } from '@nestjs/common';

@Injectable()
export class AppService {
  constructor(
    @Inject('CONFIG_OPTIONS') private readonly config: ConfigOptions,
    @Inject('DATABASE_CONNECTION') private readonly db: Database,
  ) {}
}
```

---

## Custom Providers

Custom providers offer flexibility beyond standard class-based injection.

### useClass - Dynamic Class Selection

```typescript
// Abstract class or interface token
abstract class LoggerService {
  abstract log(message: string): void;
  abstract error(message: string, trace?: string): void;
}

// Implementations
@Injectable()
class DevelopmentLogger implements LoggerService {
  log(message: string): void {
    console.log(`[DEV] ${message}`);
  }
  error(message: string, trace?: string): void {
    console.error(`[DEV ERROR] ${message}`, trace);
  }
}

@Injectable()
class ProductionLogger implements LoggerService {
  log(message: string): void {
    // Send to logging service
  }
  error(message: string, trace?: string): void {
    // Send to error tracking service
  }
}

// Module configuration
@Module({
  providers: [
    {
      provide: LoggerService,
      useClass:
        process.env.NODE_ENV === 'production'
          ? ProductionLogger
          : DevelopmentLogger,
    },
  ],
  exports: [LoggerService],
})
export class LoggerModule {}
```

### useValue - Static Values and Mocks

```typescript
// Configuration object
const configProvider = {
  provide: 'APP_CONFIG',
  useValue: {
    apiUrl: 'https://api.example.com',
    timeout: 5000,
    retries: 3,
  },
};

// Mock for testing
const mockUsersService = {
  provide: UsersService,
  useValue: {
    findAll: jest.fn().mockResolvedValue([]),
    findOne: jest.fn().mockResolvedValue({ id: 1, name: 'Test' }),
    create: jest.fn().mockImplementation(dto => ({ id: 1, ...dto })),
  },
};

@Module({
  providers: [configProvider],
})
export class AppModule {}
```

### useFactory - Dynamic Provider Creation

```typescript
// Simple factory
const connectionFactory = {
  provide: 'DATABASE_CONNECTION',
  useFactory: async (configService: ConfigService) => {
    const dbConfig = {
      host: configService.get('DB_HOST'),
      port: configService.get('DB_PORT'),
      username: configService.get('DB_USER'),
      password: configService.get('DB_PASSWORD'),
      database: configService.get('DB_NAME'),
    };
    const connection = await createConnection(dbConfig);
    return connection;
  },
  inject: [ConfigService],
};

// Factory with multiple dependencies
const cacheFactory = {
  provide: 'CACHE_MANAGER',
  useFactory: async (
    configService: ConfigService,
    logger: LoggerService,
  ) => {
    logger.log('Initializing cache manager');
    const cacheType = configService.get('CACHE_TYPE', 'memory');

    if (cacheType === 'redis') {
      return new RedisCache({
        host: configService.get('REDIS_HOST'),
        port: configService.get('REDIS_PORT'),
      });
    }
    return new MemoryCache();
  },
  inject: [ConfigService, LoggerService],
};

@Module({
  providers: [connectionFactory, cacheFactory],
  exports: ['DATABASE_CONNECTION', 'CACHE_MANAGER'],
})
export class DatabaseModule {}
```

### useExisting - Provider Aliasing

```typescript
@Injectable()
class LoggerService {
  log(message: string): void {
    console.log(message);
  }
}

@Module({
  providers: [
    LoggerService,
    {
      provide: 'AliasedLogger',
      useExisting: LoggerService,
    },
  ],
  exports: [LoggerService, 'AliasedLogger'],
})
export class LoggerModule {}

// Both inject the same singleton instance
@Injectable()
class AppService {
  constructor(
    private readonly logger: LoggerService,
    @Inject('AliasedLogger') private readonly aliasedLogger: LoggerService,
  ) {
    console.log(logger === aliasedLogger); // true
  }
}
```

---

## Optional Providers

Use `@Optional()` when a dependency may not be available.

```typescript
import { Injectable, Optional, Inject } from '@nestjs/common';

@Injectable()
export class HttpService {
  constructor(
    @Optional() @Inject('HTTP_OPTIONS') private readonly options?: HttpOptions,
  ) {
    this.options = options ?? {
      timeout: 5000,
      retries: 3,
      baseUrl: '',
    };
  }

  async get<T>(url: string): Promise<T> {
    const fullUrl = `${this.options.baseUrl}${url}`;
    // Implementation
  }
}

@Injectable()
export class NotificationService {
  constructor(
    @Optional() private readonly emailService?: EmailService,
    @Optional() private readonly smsService?: SmsService,
  ) {}

  async notify(message: string): Promise<void> {
    if (this.emailService) {
      await this.emailService.send(message);
    }
    if (this.smsService) {
      await this.smsService.send(message);
    }
  }
}
```

---

## Property-Based Injection

Use when constructor injection is not possible (e.g., inheritance scenarios).

```typescript
import { Injectable, Inject } from '@nestjs/common';

@Injectable()
export class BaseService {
  @Inject(ConfigService)
  protected readonly configService: ConfigService;

  @Inject('LOGGER')
  protected readonly logger: LoggerService;
}

@Injectable()
export class UsersService extends BaseService {
  // configService and logger are available via property injection
  findAll(): User[] {
    this.logger.log('Finding all users');
    const limit = this.configService.get('LIMIT');
    return this.users.slice(0, limit);
  }
}
```

**Note:** Constructor injection is preferred for better testability and explicit dependencies.

---

## Provider Scopes

NestJS supports three injection scopes that control provider instantiation.

### DEFAULT Scope (Singleton)

```typescript
import { Injectable } from '@nestjs/common';

@Injectable() // Scope.DEFAULT is implicit
export class CacheService {
  private cache = new Map<string, any>();

  get(key: string): any {
    return this.cache.get(key);
  }

  set(key: string, value: any): void {
    this.cache.set(key, value);
  }
}
```

Single instance shared across the entire application lifecycle.

### REQUEST Scope

```typescript
import { Injectable, Scope, Inject } from '@nestjs/common';
import { REQUEST } from '@nestjs/core';
import { Request } from 'express';

@Injectable({ scope: Scope.REQUEST })
export class RequestContextService {
  constructor(@Inject(REQUEST) private readonly request: Request) {}

  getUserId(): string {
    return this.request.user?.id;
  }

  getTenantId(): string {
    return this.request.headers['x-tenant-id'] as string;
  }

  getCorrelationId(): string {
    return this.request.headers['x-correlation-id'] as string || uuid();
  }
}

// Usage in controller
@Controller('orders')
export class OrdersController {
  constructor(
    private readonly ordersService: OrdersService,
    private readonly requestContext: RequestContextService,
  ) {}

  @Post()
  createOrder(@Body() dto: CreateOrderDto): Promise<Order> {
    const userId = this.requestContext.getUserId();
    return this.ordersService.create(dto, userId);
  }
}
```

New instance created for each incoming request. **Caution:** Scope bubbles up the injection chain.

### TRANSIENT Scope

```typescript
import { Injectable, Scope } from '@nestjs/common';

@Injectable({ scope: Scope.TRANSIENT })
export class OperationLogger {
  private operationId: string;

  setOperationId(id: string): void {
    this.operationId = id;
  }

  log(message: string): void {
    console.log(`[${this.operationId}] ${message}`);
  }
}

// Each consumer gets its own instance
@Injectable()
export class ServiceA {
  constructor(private readonly logger: OperationLogger) {
    this.logger.setOperationId('ServiceA');
  }
}

@Injectable()
export class ServiceB {
  constructor(private readonly logger: OperationLogger) {
    this.logger.setOperationId('ServiceB');
  }
}
```

New instance created for each consumer (injection point).

### Scope Inheritance

```typescript
// REQUEST-scoped provider
@Injectable({ scope: Scope.REQUEST })
export class RequestScopedService {}

// This becomes REQUEST-scoped because it depends on RequestScopedService
@Injectable() // Inherits REQUEST scope
export class DependentService {
  constructor(private readonly requestScoped: RequestScopedService) {}
}
```

---

## Circular Dependency with forwardRef

Handle circular dependencies between providers.

```typescript
import { Injectable, Inject, forwardRef } from '@nestjs/common';

@Injectable()
export class UsersService {
  constructor(
    @Inject(forwardRef(() => PostsService))
    private readonly postsService: PostsService,
  ) {}

  async getUserWithPosts(userId: number): Promise<UserWithPosts> {
    const user = await this.findOne(userId);
    const posts = await this.postsService.findByUserId(userId);
    return { ...user, posts };
  }
}

@Injectable()
export class PostsService {
  constructor(
    @Inject(forwardRef(() => UsersService))
    private readonly usersService: UsersService,
  ) {}

  async getPostWithAuthor(postId: number): Promise<PostWithAuthor> {
    const post = await this.findOne(postId);
    const author = await this.usersService.findOne(post.authorId);
    return { ...post, author };
  }
}

// Module also needs forwardRef
@Module({
  imports: [forwardRef(() => PostsModule)],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

**Best Practice:** Refactor to avoid circular dependencies when possible.

---

## Module Reference (ModuleRef)

Access providers dynamically at runtime.

```typescript
import { Injectable, OnModuleInit } from '@nestjs/common';
import { ModuleRef } from '@nestjs/core';

@Injectable()
export class DynamicService implements OnModuleInit {
  private usersService: UsersService;

  constructor(private readonly moduleRef: ModuleRef) {}

  onModuleInit(): void {
    this.usersService = this.moduleRef.get(UsersService, { strict: false });
  }

  // Resolve transient or request-scoped providers
  async resolveScoped(): Promise<void> {
    const contextId = ContextIdFactory.create();
    const requestScoped = await this.moduleRef.resolve(
      RequestScopedService,
      contextId,
    );
  }
}
```

### Dynamic Provider Resolution

```typescript
@Injectable()
export class PaymentService {
  constructor(private readonly moduleRef: ModuleRef) {}

  async processPayment(
    method: 'stripe' | 'paypal' | 'crypto',
    amount: number,
  ): Promise<PaymentResult> {
    const processorMap = {
      stripe: StripeProcessor,
      paypal: PaypalProcessor,
      crypto: CryptoProcessor,
    };

    const processor = this.moduleRef.get(processorMap[method]);
    return processor.process(amount);
  }
}
```

### Creating Standalone Instances

```typescript
@Injectable()
export class PluginLoader {
  constructor(private readonly moduleRef: ModuleRef) {}

  async loadPlugin(pluginClass: Type<Plugin>): Promise<Plugin> {
    // Create instance with DI resolution
    const plugin = await this.moduleRef.create(pluginClass);
    await plugin.initialize();
    return plugin;
  }
}
```

---

## Lazy Loading Services

Load modules on-demand to improve startup performance.

```typescript
import { Injectable } from '@nestjs/common';
import { LazyModuleLoader } from '@nestjs/core';

@Injectable()
export class ReportService {
  constructor(private readonly lazyModuleLoader: LazyModuleLoader) {}

  async generateComplexReport(data: ReportData): Promise<Report> {
    // Load heavy module only when needed
    const { ReportingModule } = await import('./reporting/reporting.module');
    const moduleRef = await this.lazyModuleLoader.load(() => ReportingModule);

    const reportGenerator = moduleRef.get(ReportGeneratorService);
    return reportGenerator.generate(data);
  }

  async exportToPdf(report: Report): Promise<Buffer> {
    const { PdfModule } = await import('./pdf/pdf.module');
    const moduleRef = await this.lazyModuleLoader.load(() => PdfModule);

    const pdfService = moduleRef.get(PdfService);
    return pdfService.generate(report);
  }
}
```

---

## Async Providers

Handle asynchronous initialization.

```typescript
// Database connection
const databaseProviders = [
  {
    provide: 'DATABASE_CONNECTION',
    useFactory: async (): Promise<Connection> => {
      const connection = await createConnection({
        type: 'postgres',
        host: process.env.DB_HOST,
        port: parseInt(process.env.DB_PORT, 10),
        database: process.env.DB_NAME,
        synchronize: process.env.NODE_ENV !== 'production',
      });
      await connection.runMigrations();
      return connection;
    },
  },
];

// External API client with authentication
const apiClientProvider = {
  provide: 'EXTERNAL_API_CLIENT',
  useFactory: async (configService: ConfigService): Promise<ApiClient> => {
    const client = new ApiClient({
      baseUrl: configService.get('API_BASE_URL'),
    });

    // Authenticate and get token
    await client.authenticate({
      clientId: configService.get('API_CLIENT_ID'),
      clientSecret: configService.get('API_CLIENT_SECRET'),
    });

    return client;
  },
  inject: [ConfigService],
};

// Multiple async dependencies
const complexProvider = {
  provide: 'ANALYTICS_SERVICE',
  useFactory: async (
    db: Connection,
    cache: CacheManager,
    config: ConfigService,
  ): Promise<AnalyticsService> => {
    const service = new AnalyticsService(db, cache);
    await service.loadModels(config.get('ML_MODELS_PATH'));
    await service.warmupCache();
    return service;
  },
  inject: ['DATABASE_CONNECTION', 'CACHE_MANAGER', ConfigService],
};

@Module({
  providers: [
    ...databaseProviders,
    apiClientProvider,
    complexProvider,
  ],
  exports: ['DATABASE_CONNECTION', 'EXTERNAL_API_CLIENT', 'ANALYTICS_SERVICE'],
})
export class InfrastructureModule {}
```

---

## Best Practices

### 1. Single Responsibility

```typescript
// Good: Focused service
@Injectable()
export class UserValidationService {
  validateEmail(email: string): boolean { /* ... */ }
  validatePassword(password: string): ValidationResult { /* ... */ }
}

@Injectable()
export class UserPersistenceService {
  save(user: User): Promise<User> { /* ... */ }
  findById(id: number): Promise<User> { /* ... */ }
}

// Bad: God service
@Injectable()
export class UserService {
  validateEmail() { /* ... */ }
  hashPassword() { /* ... */ }
  sendEmail() { /* ... */ }
  generateReport() { /* ... */ }
  processPayment() { /* ... */ }
}
```

### 2. Use Injection Tokens for Interfaces

```typescript
// Define token
export const PAYMENT_GATEWAY = Symbol('PAYMENT_GATEWAY');

// Interface
export interface PaymentGateway {
  charge(amount: number): Promise<PaymentResult>;
  refund(transactionId: string): Promise<RefundResult>;
}

// Implementation
@Injectable()
export class StripeGateway implements PaymentGateway {
  async charge(amount: number): Promise<PaymentResult> { /* ... */ }
  async refund(transactionId: string): Promise<RefundResult> { /* ... */ }
}

// Registration
@Module({
  providers: [
    {
      provide: PAYMENT_GATEWAY,
      useClass: StripeGateway,
    },
  ],
})
export class PaymentModule {}

// Injection
@Injectable()
export class OrderService {
  constructor(
    @Inject(PAYMENT_GATEWAY) private readonly paymentGateway: PaymentGateway,
  ) {}
}
```

### 3. Prefer Constructor Injection

```typescript
// Good: Explicit dependencies
@Injectable()
export class OrderService {
  constructor(
    private readonly usersService: UsersService,
    private readonly inventoryService: InventoryService,
    private readonly paymentService: PaymentService,
  ) {}
}

// Avoid: Hidden dependencies
@Injectable()
export class OrderService {
  @Inject() usersService: UsersService;
  @Inject() inventoryService: InventoryService;
}
```

### 4. Handle Cleanup with Lifecycle Hooks

```typescript
import { Injectable, OnModuleDestroy } from '@nestjs/common';

@Injectable()
export class DatabaseService implements OnModuleDestroy {
  private connection: Connection;

  async onModuleDestroy(): Promise<void> {
    if (this.connection) {
      await this.connection.close();
    }
  }
}
```

---

## Common Pitfalls

### 1. Scope Bubble Effect

```typescript
// REQUEST-scoped
@Injectable({ scope: Scope.REQUEST })
export class RequestLogger {}

// This becomes REQUEST-scoped too!
@Injectable()
export class UserService {
  constructor(private readonly logger: RequestLogger) {}
}

// Solution: Use durable providers or restructure
@Injectable({ scope: Scope.REQUEST, durable: true })
export class DurableRequestLogger {}
```

### 2. Missing Provider Registration

```typescript
// Error: Nest can't resolve dependencies
@Module({
  providers: [UsersService], // Missing UsersRepository
})
export class UsersModule {}

// Fix: Register all dependencies
@Module({
  providers: [UsersService, UsersRepository],
})
export class UsersModule {}
```

### 3. Circular Module Imports

```typescript
// Problem: Modules importing each other
@Module({
  imports: [PostsModule], // PostsModule imports UsersModule
})
export class UsersModule {}

// Solution: Use forwardRef
@Module({
  imports: [forwardRef(() => PostsModule)],
})
export class UsersModule {}
```

### 4. Async Provider Timing

```typescript
// Problem: Using provider before it's ready
@Injectable()
export class AppService implements OnModuleInit {
  constructor(
    @Inject('ASYNC_CONNECTION') private connection: Connection,
  ) {}

  // Safe: Module is fully initialized
  async onModuleInit(): Promise<void> {
    await this.connection.query('SELECT 1');
  }
}
```

### 5. Testing with Custom Providers

```typescript
describe('UsersService', () => {
  let service: UsersService;
  let mockRepository: jest.Mocked<UsersRepository>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        UsersService,
        {
          provide: UsersRepository,
          useValue: {
            findOne: jest.fn(),
            save: jest.fn(),
            delete: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get(UsersService);
    mockRepository = module.get(UsersRepository);
  });

  it('should find user by id', async () => {
    mockRepository.findOne.mockResolvedValue({ id: 1, name: 'Test' });
    const result = await service.findOne(1);
    expect(result.name).toBe('Test');
  });
});
```
