# NestJS Modules

> Official Documentation: https://docs.nestjs.com/modules

## Overview

Modules are the fundamental building blocks for organizing a NestJS application. They encapsulate related functionality, providers, controllers, and other modules into cohesive units. Every NestJS application has at least one module - the root module - which serves as the entry point for the application's dependency graph.

## Module Basics and @Module Decorator

The `@Module()` decorator provides metadata that NestJS uses to organize the application structure.

### Basic Module Structure

```typescript
import { Module } from '@nestjs/common';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { UsersRepository } from './users.repository';

@Module({
  imports: [],           // Modules to import
  controllers: [UsersController],  // Controllers handled by this module
  providers: [UsersService, UsersRepository],  // Services available in this module
  exports: [UsersService],  // Services to expose to other modules
})
export class UsersModule {}
```

### Module Properties Reference

| Property      | Type       | Description                                                    |
|---------------|------------|----------------------------------------------------------------|
| `imports`     | `Module[]` | List of imported modules that export providers needed here     |
| `controllers` | `Class[]`  | Controllers to be instantiated within this module              |
| `providers`   | `Provider[]` | Providers that will be instantiated and shared within module |
| `exports`     | `Provider[]` | Subset of providers available to other importing modules     |

### Root Module (AppModule)

```typescript
import { Module } from '@nestjs/common';
import { UsersModule } from './users/users.module';
import { AuthModule } from './auth/auth.module';
import { DatabaseModule } from './database/database.module';

@Module({
  imports: [
    DatabaseModule,
    UsersModule,
    AuthModule,
  ],
  controllers: [],
  providers: [],
})
export class AppModule {}
```

## Feature Modules

Feature modules organize code related to specific features, keeping the codebase modular and maintainable.

### Creating a Feature Module

```typescript
// products/products.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ProductsController } from './products.controller';
import { ProductsService } from './products.service';
import { Product } from './entities/product.entity';

@Module({
  imports: [TypeOrmModule.forFeature([Product])],
  controllers: [ProductsController],
  providers: [ProductsService],
  exports: [ProductsService],
})
export class ProductsModule {}
```

### Feature Module with Multiple Components

```typescript
// orders/orders.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { OrdersController } from './controllers/orders.controller';
import { OrderItemsController } from './controllers/order-items.controller';
import { OrdersService } from './services/orders.service';
import { OrderItemsService } from './services/order-items.service';
import { OrdersRepository } from './repositories/orders.repository';
import { Order } from './entities/order.entity';
import { OrderItem } from './entities/order-item.entity';
import { ProductsModule } from '../products/products.module';

@Module({
  imports: [
    TypeOrmModule.forFeature([Order, OrderItem]),
    ProductsModule,  // Import to use ProductsService
  ],
  controllers: [OrdersController, OrderItemsController],
  providers: [OrdersService, OrderItemsService, OrdersRepository],
  exports: [OrdersService],
})
export class OrdersModule {}
```

## Shared Modules

Shared modules contain common functionality used across multiple feature modules.

### Creating a Shared Module

```typescript
// shared/shared.module.ts
import { Module } from '@nestjs/common';
import { LoggerService } from './services/logger.service';
import { ValidationService } from './services/validation.service';
import { CacheService } from './services/cache.service';
import { DateHelperService } from './services/date-helper.service';

@Module({
  providers: [
    LoggerService,
    ValidationService,
    CacheService,
    DateHelperService,
  ],
  exports: [
    LoggerService,
    ValidationService,
    CacheService,
    DateHelperService,
  ],
})
export class SharedModule {}
```

### Using Shared Modules

```typescript
// users/users.module.ts
import { Module } from '@nestjs/common';
import { SharedModule } from '../shared/shared.module';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

@Module({
  imports: [SharedModule],
  controllers: [UsersController],
  providers: [UsersService],
})
export class UsersModule {}
```

## Global Modules (@Global)

Global modules make providers available application-wide without explicit imports.

### Creating a Global Module

```typescript
// config/config.module.ts
import { Global, Module } from '@nestjs/common';
import { ConfigService } from './config.service';
import { EnvironmentService } from './environment.service';

@Global()
@Module({
  providers: [
    ConfigService,
    EnvironmentService,
  ],
  exports: [
    ConfigService,
    EnvironmentService,
  ],
})
export class ConfigModule {}
```

### Global Module Best Practices

```typescript
// database/database.module.ts
import { Global, Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigService } from '../config/config.service';

@Global()
@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        type: 'postgres',
        host: config.get('DB_HOST'),
        port: config.get('DB_PORT'),
        username: config.get('DB_USER'),
        password: config.get('DB_PASSWORD'),
        database: config.get('DB_NAME'),
        autoLoadEntities: true,
        synchronize: config.get('NODE_ENV') !== 'production',
      }),
    }),
  ],
  exports: [TypeOrmModule],
})
export class DatabaseModule {}
```

**Note:** Use `@Global()` sparingly. Overuse leads to tight coupling and makes the application harder to test and maintain.

## Dynamic Modules

Dynamic modules allow runtime configuration of module behavior.

### forRoot Pattern (Singleton Configuration)

```typescript
// mailer/mailer.module.ts
import { DynamicModule, Module } from '@nestjs/common';
import { MailerService } from './mailer.service';

export interface MailerModuleOptions {
  host: string;
  port: number;
  secure: boolean;
  auth: {
    user: string;
    pass: string;
  };
}

@Module({})
export class MailerModule {
  static forRoot(options: MailerModuleOptions): DynamicModule {
    return {
      module: MailerModule,
      global: true,  // Optional: make globally available
      providers: [
        {
          provide: 'MAILER_OPTIONS',
          useValue: options,
        },
        MailerService,
      ],
      exports: [MailerService],
    };
  }
}
```

### forRootAsync Pattern (Async Configuration)

```typescript
// mailer/mailer.module.ts
import { DynamicModule, Module, ModuleMetadata } from '@nestjs/common';
import { MailerService } from './mailer.service';
import { MailerModuleOptions } from './interfaces/mailer-options.interface';

export interface MailerModuleAsyncOptions extends Pick<ModuleMetadata, 'imports'> {
  useFactory: (...args: any[]) => Promise<MailerModuleOptions> | MailerModuleOptions;
  inject?: any[];
}

@Module({})
export class MailerModule {
  static forRoot(options: MailerModuleOptions): DynamicModule {
    return {
      module: MailerModule,
      global: true,
      providers: [
        {
          provide: 'MAILER_OPTIONS',
          useValue: options,
        },
        MailerService,
      ],
      exports: [MailerService],
    };
  }

  static forRootAsync(options: MailerModuleAsyncOptions): DynamicModule {
    return {
      module: MailerModule,
      global: true,
      imports: options.imports || [],
      providers: [
        {
          provide: 'MAILER_OPTIONS',
          useFactory: options.useFactory,
          inject: options.inject || [],
        },
        MailerService,
      ],
      exports: [MailerService],
    };
  }
}

// Usage with ConfigService
@Module({
  imports: [
    MailerModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: async (config: ConfigService) => ({
        host: config.get('MAIL_HOST'),
        port: config.get('MAIL_PORT'),
        secure: config.get('MAIL_SECURE'),
        auth: {
          user: config.get('MAIL_USER'),
          pass: config.get('MAIL_PASS'),
        },
      }),
    }),
  ],
})
export class AppModule {}
```

### forFeature Pattern (Per-Feature Configuration)

```typescript
// events/events.module.ts
import { DynamicModule, Module } from '@nestjs/common';
import { EventsService } from './events.service';
import { EventHandler } from './interfaces/event-handler.interface';

@Module({})
export class EventsModule {
  static forRoot(): DynamicModule {
    return {
      module: EventsModule,
      global: true,
      providers: [EventsService],
      exports: [EventsService],
    };
  }

  static forFeature(handlers: EventHandler[]): DynamicModule {
    return {
      module: EventsModule,
      providers: [
        ...handlers,
        {
          provide: 'EVENT_HANDLERS',
          useValue: handlers,
        },
      ],
    };
  }
}

// Usage
@Module({
  imports: [
    EventsModule.forFeature([
      UserCreatedHandler,
      UserUpdatedHandler,
      UserDeletedHandler,
    ]),
  ],
})
export class UsersModule {}
```

## Module Re-exporting

Modules can re-export imported modules, making their providers available to consumers.

### Basic Re-exporting

```typescript
// core/core.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '../config/config.module';
import { LoggingModule } from '../logging/logging.module';
import { CacheModule } from '../cache/cache.module';

@Module({
  imports: [
    ConfigModule,
    LoggingModule,
    CacheModule,
  ],
  exports: [
    ConfigModule,
    LoggingModule,
    CacheModule,
  ],
})
export class CoreModule {}

// Now any module importing CoreModule gets access to all three modules
@Module({
  imports: [CoreModule],
})
export class UsersModule {}
```

### Combining Local and Re-exported Providers

```typescript
// common/common.module.ts
import { Module } from '@nestjs/common';
import { HttpModule } from '@nestjs/axios';
import { CommonService } from './common.service';
import { UtilityService } from './utility.service';

@Module({
  imports: [HttpModule],
  providers: [CommonService, UtilityService],
  exports: [
    HttpModule,        // Re-export imported module
    CommonService,     // Export local provider
    UtilityService,    // Export local provider
  ],
})
export class CommonModule {}
```

## Circular Dependencies Handling

Circular dependencies occur when two modules depend on each other. NestJS provides mechanisms to handle them.

### Using forwardRef()

```typescript
// users/users.module.ts
import { Module, forwardRef } from '@nestjs/common';
import { UsersService } from './users.service';
import { AuthModule } from '../auth/auth.module';

@Module({
  imports: [forwardRef(() => AuthModule)],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}

// auth/auth.module.ts
import { Module, forwardRef } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [forwardRef(() => UsersModule)],
  providers: [AuthService],
  exports: [AuthService],
})
export class AuthModule {}
```

### Circular Provider Dependencies

```typescript
// users/users.service.ts
import { Injectable, Inject, forwardRef } from '@nestjs/common';
import { AuthService } from '../auth/auth.service';

@Injectable()
export class UsersService {
  constructor(
    @Inject(forwardRef(() => AuthService))
    private authService: AuthService,
  ) {}
}

// auth/auth.service.ts
import { Injectable, Inject, forwardRef } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(
    @Inject(forwardRef(() => UsersService))
    private usersService: UsersService,
  ) {}
}
```

### Avoiding Circular Dependencies (Recommended)

```typescript
// Extract shared logic into a separate module
// shared/shared-user.service.ts
@Injectable()
export class SharedUserService {
  // Common user operations used by both Auth and Users
}

// shared/shared.module.ts
@Module({
  providers: [SharedUserService],
  exports: [SharedUserService],
})
export class SharedModule {}

// Now both modules import SharedModule instead of each other
```

## Module Reference

`ModuleRef` provides access to the internal provider container for dynamic provider resolution.

### Basic ModuleRef Usage

```typescript
import { Injectable, OnModuleInit } from '@nestjs/common';
import { ModuleRef } from '@nestjs/core';
import { UsersService } from './users.service';

@Injectable()
export class DynamicService implements OnModuleInit {
  private usersService: UsersService;

  constructor(private moduleRef: ModuleRef) {}

  onModuleInit() {
    this.usersService = this.moduleRef.get(UsersService, { strict: false });
  }
}
```

### Resolving Scoped Providers

```typescript
import { Injectable, Scope } from '@nestjs/common';
import { ModuleRef, ContextIdFactory } from '@nestjs/core';
import { Request } from 'express';

@Injectable()
export class RequestContextService {
  constructor(private moduleRef: ModuleRef) {}

  async getRequestScopedProvider(request: Request) {
    const contextId = ContextIdFactory.getByRequest(request);

    // Register the request object for DI
    this.moduleRef.registerRequestByContextId(request, contextId);

    // Resolve the request-scoped provider
    return this.moduleRef.resolve(RequestScopedService, contextId);
  }
}

@Injectable({ scope: Scope.REQUEST })
export class RequestScopedService {
  // This service is instantiated per request
}
```

### Creating Providers Dynamically

```typescript
import { Injectable } from '@nestjs/common';
import { ModuleRef } from '@nestjs/core';

@Injectable()
export class StrategyFactory {
  constructor(private moduleRef: ModuleRef) {}

  async createStrategy(type: string) {
    switch (type) {
      case 'email':
        return this.moduleRef.create(EmailStrategy);
      case 'sms':
        return this.moduleRef.create(SmsStrategy);
      default:
        throw new Error(`Unknown strategy type: ${type}`);
    }
  }
}
```

## Lazy Loading Modules

Lazy loading defers module initialization until needed, improving startup time.

### Basic Lazy Loading

```typescript
import { Injectable } from '@nestjs/common';
import { LazyModuleLoader } from '@nestjs/core';

@Injectable()
export class ReportingService {
  constructor(private lazyModuleLoader: LazyModuleLoader) {}

  async generateReport() {
    // Module is loaded only when this method is called
    const { ReportingModule } = await import('./reporting/reporting.module');
    const moduleRef = await this.lazyModuleLoader.load(() => ReportingModule);

    const reportGenerator = moduleRef.get(ReportGenerator);
    return reportGenerator.generate();
  }
}
```

### Lazy Loading with Controller Registration

```typescript
// Note: Lazy-loaded modules cannot register controllers, routes, or GraphQL resolvers
// They are best suited for background tasks, cron jobs, or worker processes

import { Injectable } from '@nestjs/common';
import { LazyModuleLoader } from '@nestjs/core';

@Injectable()
export class BackgroundJobService {
  constructor(private lazyModuleLoader: LazyModuleLoader) {}

  async processHeavyTask(data: any) {
    const { HeavyProcessingModule } = await import('./heavy-processing/heavy-processing.module');
    const moduleRef = await this.lazyModuleLoader.load(() => HeavyProcessingModule);

    const processor = moduleRef.get(HeavyProcessor);
    return processor.process(data);
  }
}
```

## Best Practices and Common Pitfalls

### Best Practices

**1. Single Responsibility**
```typescript
// Good: Focused module
@Module({
  controllers: [PaymentsController],
  providers: [PaymentsService, PaymentGatewayService],
})
export class PaymentsModule {}

// Bad: Bloated module with unrelated concerns
@Module({
  controllers: [PaymentsController, UsersController, OrdersController],
  providers: [PaymentsService, UsersService, OrdersService, EmailService],
})
export class EverythingModule {}
```

**2. Explicit Exports**
```typescript
// Good: Only export what's needed
@Module({
  providers: [UsersService, UsersRepository, PasswordHasher],
  exports: [UsersService], // Only expose the service
})
export class UsersModule {}
```

**3. Prefer Composition Over Global Modules**
```typescript
// Good: Explicit dependencies
@Module({
  imports: [LoggerModule, ConfigModule],
  providers: [UsersService],
})
export class UsersModule {}

// Avoid: Too many global modules make dependencies unclear
```

**4. Organize by Feature, Not by Type**
```
// Good: Feature-based structure
src/
  users/
    users.module.ts
    users.controller.ts
    users.service.ts
  products/
    products.module.ts
    products.controller.ts
    products.service.ts

// Bad: Type-based structure
src/
  controllers/
    users.controller.ts
    products.controller.ts
  services/
    users.service.ts
    products.service.ts
```

### Common Pitfalls

**1. Forgetting to Export Providers**
```typescript
// Problem: Provider not exported
@Module({
  providers: [UsersService],
  // Missing: exports: [UsersService]
})
export class UsersModule {}

// Error in another module:
// "Nest can't resolve dependencies of UsersService"
```

**2. Circular Dependency Without forwardRef**
```typescript
// Problem: Direct circular import
@Module({
  imports: [AuthModule], // AuthModule also imports UsersModule
})
export class UsersModule {}

// Solution: Use forwardRef
@Module({
  imports: [forwardRef(() => AuthModule)],
})
export class UsersModule {}
```

**3. Importing Module Instead of Its Exports**
```typescript
// Problem: Trying to inject module instead of its provider
constructor(private usersModule: UsersModule) {} // Wrong!

// Solution: Inject the exported provider
constructor(private usersService: UsersService) {} // Correct!
```

**4. Overusing @Global()**
```typescript
// Problem: Everything is global
@Global()
@Module({ ... })
export class UsersModule {}

@Global()
@Module({ ... })
export class ProductsModule {}

// This defeats the purpose of modular architecture
// Only use @Global() for truly application-wide services (Config, Logger)
```

**5. Not Handling Async Configuration**
```typescript
// Problem: Synchronous config that needs async values
static forRoot(options: Options): DynamicModule {
  return {
    providers: [{ provide: 'OPTIONS', useValue: options }],
  };
}

// Solution: Provide forRootAsync for async scenarios
static forRootAsync(options: AsyncOptions): DynamicModule {
  return {
    providers: [{
      provide: 'OPTIONS',
      useFactory: options.useFactory,
      inject: options.inject,
    }],
  };
}
```

### Module Checklist

- [ ] Each module has a single, clear responsibility
- [ ] Dependencies are explicitly imported
- [ ] Only necessary providers are exported
- [ ] `@Global()` is used sparingly and intentionally
- [ ] Dynamic modules provide both sync and async configuration
- [ ] Circular dependencies are resolved or refactored away
- [ ] Feature modules are self-contained and testable
