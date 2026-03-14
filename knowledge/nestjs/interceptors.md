# NestJS Interceptors - Comprehensive Guide

## Overview

Interceptors are a powerful feature in NestJS inspired by the Aspect-Oriented Programming (AOP) technique. They enable you to:

- Bind extra logic before/after method execution
- Transform the result returned from a function
- Transform the exception thrown from a function
- Extend basic function behavior
- Completely override a function depending on specific conditions (e.g., for caching purposes)

Interceptors have access to the `ExecutionContext` instance, which provides information about the current execution context, and the `CallHandler`, which allows you to invoke the route handler method at some point in your interceptor.

---

## 1. Interceptor Basics

### What is an Interceptor?

An interceptor is a class annotated with the `@Injectable()` decorator and implements the `NestInterceptor` interface. Interceptors wrap the request/response stream, allowing you to execute logic both before and after the execution of the final route handler.

### Basic Interceptor Structure

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log('Before...');

    const now = Date.now();
    return next
      .handle()
      .pipe(tap(() => console.log(`After... ${Date.now() - now}ms`)));
  }
}
```

### Key Components

1. **ExecutionContext**: Extends `ArgumentsHost`, providing additional details about the current execution process
2. **CallHandler**: Interface that wraps the execution stream, allowing you to implement custom logic before/after method execution
3. **Observable**: Interceptors work with RxJS Observables, giving you powerful stream manipulation capabilities

### When to Use Interceptors

- Logging request/response data
- Transforming responses to a standard format
- Caching responses
- Handling timeouts
- Adding retry logic
- Measuring performance
- Exception mapping

---

## 2. NestInterceptor Interface

### Interface Definition

```typescript
export interface NestInterceptor<T = any, R = any> {
  intercept(
    context: ExecutionContext,
    next: CallHandler<T>,
  ): Observable<R> | Promise<Observable<R>>;
}
```

### Type Parameters

- **T**: The type of the incoming value (typically the route handler's return type)
- **R**: The type of the outgoing value (the transformed response)

### Complete Interface Implementation

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

// Define response wrapper interface
interface Response<T> {
  data: T;
  statusCode: number;
  timestamp: string;
}

@Injectable()
export class TransformInterceptor<T>
  implements NestInterceptor<T, Response<T>>
{
  intercept(
    context: ExecutionContext,
    next: CallHandler<T>,
  ): Observable<Response<T>> {
    const ctx = context.switchToHttp();
    const response = ctx.getResponse();

    return next.handle().pipe(
      map((data) => ({
        data,
        statusCode: response.statusCode,
        timestamp: new Date().toISOString(),
      })),
    );
  }
}
```

### Async Interceptors

Interceptors can also return a Promise that resolves to an Observable:

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AsyncInterceptor implements NestInterceptor {
  async intercept(
    context: ExecutionContext,
    next: CallHandler,
  ): Promise<Observable<any>> {
    // Perform async operations before handler execution
    await this.someAsyncSetup();

    return next.handle();
  }

  private async someAsyncSetup(): Promise<void> {
    // Async initialization logic
  }
}
```

---

## 3. Call Handler and RxJS

### Understanding CallHandler

The `CallHandler` interface implements the `handle()` method, which returns an Observable. This Observable wraps the response stream from the route handler.

```typescript
export interface CallHandler<T = any> {
  handle(): Observable<T>;
}
```

### Importance of Calling handle()

If you don't call `handle()`, the route handler method won't be executed at all. This is useful for completely overriding behavior (e.g., returning cached data).

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable, of } from 'rxjs';

@Injectable()
export class CacheInterceptor implements NestInterceptor {
  private cache = new Map<string, any>();

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const cacheKey = request.url;

    // If cached, return cached value without calling handler
    if (this.cache.has(cacheKey)) {
      return of(this.cache.get(cacheKey));
    }

    // Otherwise, call handler and cache result
    return next.handle().pipe(
      tap((data) => {
        this.cache.set(cacheKey, data);
      }),
    );
  }
}
```

### RxJS Operators in Interceptors

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import {
  tap,
  map,
  catchError,
  timeout,
  retry,
  finalize,
  delay,
} from 'rxjs/operators';

@Injectable()
export class ComprehensiveInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const startTime = Date.now();

    return next.handle().pipe(
      // Add timeout
      timeout(5000),

      // Retry on failure
      retry(3),

      // Transform response
      map((data) => ({ success: true, data })),

      // Handle errors
      catchError((err) => {
        if (err instanceof TimeoutError) {
          return throwError(() => new Error('Request timed out'));
        }
        return throwError(() => err);
      }),

      // Log completion
      tap({
        next: (data) => console.log('Response:', data),
        error: (err) => console.error('Error:', err),
      }),

      // Cleanup
      finalize(() => {
        console.log(`Request completed in ${Date.now() - startTime}ms`);
      }),
    );
  }
}
```

### Stream Manipulation Examples

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable, merge, from } from 'rxjs';
import { map, mergeMap, toArray, filter } from 'rxjs/operators';

// Example: Flattening array responses
@Injectable()
export class FlattenInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      mergeMap((data) => {
        if (Array.isArray(data)) {
          return from(data);
        }
        return of(data);
      }),
      filter((item) => item !== null),
      toArray(),
    );
  }
}

// Example: Combining multiple streams
@Injectable()
export class EnrichmentInterceptor implements NestInterceptor {
  constructor(private readonly metadataService: MetadataService) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      mergeMap((data) => {
        const metadata$ = this.metadataService.getMetadata(data.id);
        return metadata$.pipe(
          map((metadata) => ({
            ...data,
            metadata,
          })),
        );
      }),
    );
  }
}
```

---

## 4. Response Transformation

### Basic Response Transformation

Transform the raw response from route handlers into a standardized format:

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface StandardResponse<T> {
  success: boolean;
  data: T;
  message: string;
  timestamp: string;
}

@Injectable()
export class TransformResponseInterceptor<T>
  implements NestInterceptor<T, StandardResponse<T>>
{
  intercept(
    context: ExecutionContext,
    next: CallHandler<T>,
  ): Observable<StandardResponse<T>> {
    return next.handle().pipe(
      map((data) => ({
        success: true,
        data,
        message: 'Request successful',
        timestamp: new Date().toISOString(),
      })),
    );
  }
}
```

### Conditional Transformation

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';
import { Reflector } from '@nestjs/core';

export const SKIP_TRANSFORM_KEY = 'skipTransform';

@Injectable()
export class ConditionalTransformInterceptor implements NestInterceptor {
  constructor(private reflector: Reflector) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const skipTransform = this.reflector.getAllAndOverride<boolean>(
      SKIP_TRANSFORM_KEY,
      [context.getHandler(), context.getClass()],
    );

    if (skipTransform) {
      return next.handle();
    }

    return next.handle().pipe(
      map((data) => ({
        data,
        meta: {
          timestamp: new Date().toISOString(),
          path: context.switchToHttp().getRequest().url,
        },
      })),
    );
  }
}
```

### Pagination Response Transformation

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface PaginatedResponse<T> {
  data: T[];
  pagination: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
    hasNext: boolean;
    hasPrevious: boolean;
  };
}

@Injectable()
export class PaginationInterceptor<T> implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const page = parseInt(request.query.page, 10) || 1;
    const limit = parseInt(request.query.limit, 10) || 10;

    return next.handle().pipe(
      map((result: { data: T[]; total: number }) => {
        const totalPages = Math.ceil(result.total / limit);

        return {
          data: result.data,
          pagination: {
            page,
            limit,
            total: result.total,
            totalPages,
            hasNext: page < totalPages,
            hasPrevious: page > 1,
          },
        };
      }),
    );
  }
}
```

---

## 5. Response Mapping

### Null/Undefined Handling

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  NotFoundException,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable()
export class NotFoundInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      map((data) => {
        if (data === undefined || data === null) {
          throw new NotFoundException('Resource not found');
        }
        return data;
      }),
    );
  }
}
```

### Data Exclusion Interceptor

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';
import { classToPlain } from 'class-transformer';

@Injectable()
export class ExcludeNullInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      map((data) => this.removeNullValues(data)),
    );
  }

  private removeNullValues(obj: any): any {
    if (obj === null || obj === undefined) {
      return obj;
    }

    if (Array.isArray(obj)) {
      return obj.map((item) => this.removeNullValues(item));
    }

    if (typeof obj === 'object') {
      return Object.entries(obj).reduce((acc, [key, value]) => {
        if (value !== null && value !== undefined) {
          acc[key] = this.removeNullValues(value);
        }
        return acc;
      }, {} as Record<string, any>);
    }

    return obj;
  }
}
```

### Class Serialization Interceptor

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  UseInterceptors,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';
import { plainToInstance, ClassConstructor } from 'class-transformer';

export function Serialize<T>(dto: ClassConstructor<T>) {
  return UseInterceptors(new SerializeInterceptor(dto));
}

@Injectable()
export class SerializeInterceptor<T> implements NestInterceptor {
  constructor(private readonly dto: ClassConstructor<T>) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      map((data) => {
        return plainToInstance(this.dto, data, {
          excludeExtraneousValues: true,
        });
      }),
    );
  }
}

// Usage with DTO
import { Expose, Exclude } from 'class-transformer';

export class UserResponseDto {
  @Expose()
  id: number;

  @Expose()
  email: string;

  @Expose()
  name: string;

  @Exclude()
  password: string;

  @Exclude()
  createdAt: Date;
}

// In controller
@Controller('users')
export class UsersController {
  @Get(':id')
  @Serialize(UserResponseDto)
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(+id);
  }
}
```

---

## 6. Exception Mapping

### Basic Exception Mapping

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class ErrorsInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      catchError((err) => {
        if (err instanceof HttpException) {
          return throwError(() => err);
        }

        // Transform unknown errors to internal server error
        return throwError(
          () =>
            new HttpException(
              'Internal server error',
              HttpStatus.INTERNAL_SERVER_ERROR,
            ),
        );
      }),
    );
  }
}
```

### Detailed Exception Mapping

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  HttpException,
  HttpStatus,
  Logger,
} from '@nestjs/common';
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';
import { TypeORMError } from 'typeorm';

@Injectable()
export class DatabaseExceptionInterceptor implements NestInterceptor {
  private readonly logger = new Logger(DatabaseExceptionInterceptor.name);

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      catchError((error) => {
        this.logger.error(`Error occurred: ${error.message}`, error.stack);

        if (error instanceof TypeORMError) {
          return this.handleDatabaseError(error);
        }

        if (error instanceof HttpException) {
          return throwError(() => error);
        }

        return throwError(
          () =>
            new HttpException(
              {
                statusCode: HttpStatus.INTERNAL_SERVER_ERROR,
                message: 'An unexpected error occurred',
                error: 'Internal Server Error',
              },
              HttpStatus.INTERNAL_SERVER_ERROR,
            ),
        );
      }),
    );
  }

  private handleDatabaseError(error: TypeORMError): Observable<never> {
    const errorMessage = error.message;

    // Handle duplicate entry
    if (errorMessage.includes('duplicate')) {
      return throwError(
        () =>
          new HttpException(
            {
              statusCode: HttpStatus.CONFLICT,
              message: 'Resource already exists',
              error: 'Conflict',
            },
            HttpStatus.CONFLICT,
          ),
      );
    }

    // Handle foreign key constraint
    if (errorMessage.includes('foreign key')) {
      return throwError(
        () =>
          new HttpException(
            {
              statusCode: HttpStatus.BAD_REQUEST,
              message: 'Referenced resource does not exist',
              error: 'Bad Request',
            },
            HttpStatus.BAD_REQUEST,
          ),
      );
    }

    return throwError(
      () =>
        new HttpException(
          {
            statusCode: HttpStatus.INTERNAL_SERVER_ERROR,
            message: 'Database error occurred',
            error: 'Internal Server Error',
          },
          HttpStatus.INTERNAL_SERVER_ERROR,
        ),
    );
  }
}
```

### Business Logic Exception Interceptor

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

// Custom business exceptions
export class InsufficientFundsException extends Error {
  constructor(message: string = 'Insufficient funds') {
    super(message);
    this.name = 'InsufficientFundsException';
  }
}

export class ResourceLockedException extends Error {
  constructor(message: string = 'Resource is locked') {
    super(message);
    this.name = 'ResourceLockedException';
  }
}

@Injectable()
export class BusinessExceptionInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      catchError((error) => {
        if (error instanceof InsufficientFundsException) {
          return throwError(
            () =>
              new HttpException(
                {
                  statusCode: HttpStatus.PAYMENT_REQUIRED,
                  message: error.message,
                  errorCode: 'INSUFFICIENT_FUNDS',
                },
                HttpStatus.PAYMENT_REQUIRED,
              ),
          );
        }

        if (error instanceof ResourceLockedException) {
          return throwError(
            () =>
              new HttpException(
                {
                  statusCode: HttpStatus.LOCKED,
                  message: error.message,
                  errorCode: 'RESOURCE_LOCKED',
                },
                HttpStatus.LOCKED,
              ),
          );
        }

        return throwError(() => error);
      }),
    );
  }
}
```

---

## 7. Cache Interceptor

### Simple In-Memory Cache

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable, of } from 'rxjs';
import { tap } from 'rxjs/operators';

interface CacheEntry {
  data: any;
  expiry: number;
}

@Injectable()
export class SimpleCacheInterceptor implements NestInterceptor {
  private cache = new Map<string, CacheEntry>();
  private readonly ttl = 60000; // 1 minute default TTL

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();

    // Only cache GET requests
    if (request.method !== 'GET') {
      return next.handle();
    }

    const cacheKey = this.generateCacheKey(request);
    const cachedEntry = this.cache.get(cacheKey);

    if (cachedEntry && cachedEntry.expiry > Date.now()) {
      return of(cachedEntry.data);
    }

    return next.handle().pipe(
      tap((data) => {
        this.cache.set(cacheKey, {
          data,
          expiry: Date.now() + this.ttl,
        });
      }),
    );
  }

  private generateCacheKey(request: any): string {
    return `${request.url}:${JSON.stringify(request.query)}`;
  }
}
```

### Advanced Cache with Redis

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  Inject,
} from '@nestjs/common';
import { Observable, of, from } from 'rxjs';
import { tap, switchMap } from 'rxjs/operators';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';
import { Reflector } from '@nestjs/core';

export const CACHE_TTL_KEY = 'cache_ttl';
export const CACHE_KEY_PREFIX = 'cache_key_prefix';

@Injectable()
export class RedisCacheInterceptor implements NestInterceptor {
  constructor(
    @Inject(CACHE_MANAGER) private cacheManager: Cache,
    private reflector: Reflector,
  ) {}

  async intercept(
    context: ExecutionContext,
    next: CallHandler,
  ): Promise<Observable<any>> {
    const request = context.switchToHttp().getRequest();

    // Only cache GET requests
    if (request.method !== 'GET') {
      return next.handle();
    }

    const ttl = this.reflector.getAllAndOverride<number>(CACHE_TTL_KEY, [
      context.getHandler(),
      context.getClass(),
    ]) || 300; // 5 minutes default

    const keyPrefix = this.reflector.getAllAndOverride<string>(
      CACHE_KEY_PREFIX,
      [context.getHandler(), context.getClass()],
    ) || 'api';

    const cacheKey = `${keyPrefix}:${request.url}`;

    const cachedData = await this.cacheManager.get(cacheKey);
    if (cachedData) {
      return of(cachedData);
    }

    return next.handle().pipe(
      tap(async (data) => {
        await this.cacheManager.set(cacheKey, data, ttl * 1000);
      }),
    );
  }
}

// Decorators for cache configuration
import { SetMetadata } from '@nestjs/common';

export const CacheTTL = (ttl: number) => SetMetadata(CACHE_TTL_KEY, ttl);
export const CacheKeyPrefix = (prefix: string) =>
  SetMetadata(CACHE_KEY_PREFIX, prefix);

// Usage in controller
@Controller('products')
export class ProductsController {
  @Get()
  @CacheTTL(600) // 10 minutes
  @CacheKeyPrefix('products')
  findAll() {
    return this.productsService.findAll();
  }
}
```

### Cache Invalidation Interceptor

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  Inject,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';

@Injectable()
export class CacheInvalidationInterceptor implements NestInterceptor {
  constructor(@Inject(CACHE_MANAGER) private cacheManager: Cache) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();

    // Only invalidate on mutating operations
    if (['POST', 'PUT', 'PATCH', 'DELETE'].includes(request.method)) {
      return next.handle().pipe(
        tap(async () => {
          // Invalidate related cache keys
          const resourcePath = request.url.split('/')[1]; // e.g., 'users'
          const keys = await this.cacheManager.store.keys(`${resourcePath}:*`);
          await Promise.all(keys.map((key) => this.cacheManager.del(key)));
        }),
      );
    }

    return next.handle();
  }
}
```

---

## 8. Logging Interceptor

### Comprehensive Request/Response Logger

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  Logger,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap, catchError } from 'rxjs/operators';
import { v4 as uuidv4 } from 'uuid';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  private readonly logger = new Logger('HTTP');

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const response = context.switchToHttp().getResponse();
    const { method, url, body, query, params, headers } = request;

    const requestId = uuidv4();
    const userAgent = headers['user-agent'] || 'Unknown';
    const ip = request.ip || request.connection.remoteAddress;
    const startTime = Date.now();

    // Attach request ID for tracing
    request.requestId = requestId;
    response.setHeader('X-Request-ID', requestId);

    this.logger.log({
      type: 'REQUEST',
      requestId,
      method,
      url,
      query,
      params,
      body: this.sanitizeBody(body),
      userAgent,
      ip,
      timestamp: new Date().toISOString(),
    });

    return next.handle().pipe(
      tap((data) => {
        const duration = Date.now() - startTime;

        this.logger.log({
          type: 'RESPONSE',
          requestId,
          method,
          url,
          statusCode: response.statusCode,
          duration: `${duration}ms`,
          responseSize: JSON.stringify(data).length,
          timestamp: new Date().toISOString(),
        });
      }),
      catchError((error) => {
        const duration = Date.now() - startTime;

        this.logger.error({
          type: 'ERROR',
          requestId,
          method,
          url,
          statusCode: error.status || 500,
          errorMessage: error.message,
          duration: `${duration}ms`,
          stack: error.stack,
          timestamp: new Date().toISOString(),
        });

        throw error;
      }),
    );
  }

  private sanitizeBody(body: any): any {
    if (!body) return body;

    const sensitiveFields = ['password', 'token', 'secret', 'authorization'];
    const sanitized = { ...body };

    for (const field of sensitiveFields) {
      if (sanitized[field]) {
        sanitized[field] = '[REDACTED]';
      }
    }

    return sanitized;
  }
}
```

### Structured JSON Logger

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

interface LogEntry {
  timestamp: string;
  level: string;
  requestId: string;
  method: string;
  path: string;
  statusCode: number;
  duration: number;
  userId?: string;
  action?: string;
  resource?: string;
}

@Injectable()
export class StructuredLoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const response = context.switchToHttp().getResponse();
    const startTime = process.hrtime.bigint();

    return next.handle().pipe(
      tap(() => {
        const endTime = process.hrtime.bigint();
        const duration = Number(endTime - startTime) / 1e6; // Convert to ms

        const logEntry: LogEntry = {
          timestamp: new Date().toISOString(),
          level: 'info',
          requestId: request.requestId || 'unknown',
          method: request.method,
          path: request.url,
          statusCode: response.statusCode,
          duration: Math.round(duration * 100) / 100,
          userId: request.user?.id,
          action: this.extractAction(context),
          resource: this.extractResource(context),
        };

        // Output as JSON for log aggregators
        console.log(JSON.stringify(logEntry));
      }),
    );
  }

  private extractAction(context: ExecutionContext): string {
    return context.getHandler().name;
  }

  private extractResource(context: ExecutionContext): string {
    return context.getClass().name.replace('Controller', '');
  }
}
```

---

## 9. Timeout Interceptor

### Basic Timeout Interceptor

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  RequestTimeoutException,
} from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { timeout, catchError } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  constructor(private readonly timeoutMs: number = 5000) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(this.timeoutMs),
      catchError((err) => {
        if (err instanceof TimeoutError) {
          return throwError(
            () => new RequestTimeoutException('Request timed out'),
          );
        }
        return throwError(() => err);
      }),
    );
  }
}
```

### Configurable Timeout with Metadata

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  RequestTimeoutException,
  SetMetadata,
} from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { timeout, catchError } from 'rxjs/operators';
import { Reflector } from '@nestjs/core';

export const TIMEOUT_KEY = 'request_timeout';
export const Timeout = (ms: number) => SetMetadata(TIMEOUT_KEY, ms);

@Injectable()
export class ConfigurableTimeoutInterceptor implements NestInterceptor {
  private readonly defaultTimeout = 30000; // 30 seconds default

  constructor(private reflector: Reflector) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const timeoutValue =
      this.reflector.getAllAndOverride<number>(TIMEOUT_KEY, [
        context.getHandler(),
        context.getClass(),
      ]) || this.defaultTimeout;

    return next.handle().pipe(
      timeout(timeoutValue),
      catchError((err) => {
        if (err instanceof TimeoutError) {
          return throwError(
            () =>
              new RequestTimeoutException(
                `Request exceeded timeout of ${timeoutValue}ms`,
              ),
          );
        }
        return throwError(() => err);
      }),
    );
  }
}

// Usage in controller
@Controller('reports')
export class ReportsController {
  @Get('generate')
  @Timeout(120000) // 2 minutes for report generation
  generateReport() {
    return this.reportsService.generate();
  }
}
```

### Timeout with Progress Tracking

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  RequestTimeoutException,
  Logger,
} from '@nestjs/common';
import { Observable, throwError, TimeoutError, race, timer } from 'rxjs';
import { map, catchError, tap } from 'rxjs/operators';

@Injectable()
export class ProgressTimeoutInterceptor implements NestInterceptor {
  private readonly logger = new Logger(ProgressTimeoutInterceptor.name);

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const timeoutMs = 30000;
    const warningMs = 20000;

    // Warning timer
    const warningTimer$ = timer(warningMs).pipe(
      tap(() => {
        this.logger.warn(
          `Request ${request.url} is taking longer than ${warningMs}ms`,
        );
      }),
    );

    // Timeout timer
    const timeout$ = timer(timeoutMs).pipe(
      map(() => {
        throw new TimeoutError();
      }),
    );

    // Subscribe to warning (non-blocking)
    warningTimer$.subscribe();

    return race(next.handle(), timeout$).pipe(
      catchError((err) => {
        if (err instanceof TimeoutError) {
          this.logger.error(`Request ${request.url} timed out after ${timeoutMs}ms`);
          return throwError(
            () => new RequestTimeoutException('Request processing timed out'),
          );
        }
        return throwError(() => err);
      }),
    );
  }
}
```

---

## 10. Retry Interceptor

### Simple Retry Interceptor

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  Logger,
} from '@nestjs/common';
import { Observable, throwError } from 'rxjs';
import { retry, catchError } from 'rxjs/operators';

@Injectable()
export class RetryInterceptor implements NestInterceptor {
  private readonly logger = new Logger(RetryInterceptor.name);
  private readonly maxRetries = 3;

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      retry(this.maxRetries),
      catchError((error) => {
        this.logger.error(
          `Request failed after ${this.maxRetries} retries: ${error.message}`,
        );
        return throwError(() => error);
      }),
    );
  }
}
```

### Exponential Backoff Retry

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  Logger,
} from '@nestjs/common';
import { Observable, throwError, timer } from 'rxjs';
import { retryWhen, mergeMap, catchError } from 'rxjs/operators';

@Injectable()
export class ExponentialBackoffInterceptor implements NestInterceptor {
  private readonly logger = new Logger(ExponentialBackoffInterceptor.name);
  private readonly maxRetries = 3;
  private readonly initialDelay = 1000; // 1 second

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      retryWhen((errors) =>
        errors.pipe(
          mergeMap((error, index) => {
            const retryAttempt = index + 1;

            if (retryAttempt > this.maxRetries) {
              return throwError(() => error);
            }

            const delayMs = this.initialDelay * Math.pow(2, index);

            this.logger.warn(
              `Retry attempt ${retryAttempt}/${this.maxRetries} after ${delayMs}ms`,
            );

            return timer(delayMs);
          }),
        ),
      ),
      catchError((error) => {
        this.logger.error(
          `All ${this.maxRetries} retry attempts failed: ${error.message}`,
        );
        return throwError(() => error);
      }),
    );
  }
}
```

### Conditional Retry with Jitter

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  Logger,
  HttpException,
} from '@nestjs/common';
import { Observable, throwError, timer } from 'rxjs';
import { retryWhen, mergeMap } from 'rxjs/operators';

interface RetryConfig {
  maxRetries: number;
  initialDelay: number;
  maxDelay: number;
  retryableStatuses: number[];
}

@Injectable()
export class ConditionalRetryInterceptor implements NestInterceptor {
  private readonly logger = new Logger(ConditionalRetryInterceptor.name);
  private readonly config: RetryConfig = {
    maxRetries: 3,
    initialDelay: 500,
    maxDelay: 10000,
    retryableStatuses: [408, 429, 500, 502, 503, 504],
  };

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      retryWhen((errors) =>
        errors.pipe(
          mergeMap((error, index) => {
            const retryAttempt = index + 1;

            // Check if we should retry this error
            if (!this.shouldRetry(error, retryAttempt)) {
              return throwError(() => error);
            }

            const delay = this.calculateDelay(index);

            this.logger.warn(
              `Retrying request (attempt ${retryAttempt}/${this.config.maxRetries}) ` +
                `after ${delay}ms due to: ${error.message}`,
            );

            return timer(delay);
          }),
        ),
      ),
    );
  }

  private shouldRetry(error: any, attempt: number): boolean {
    if (attempt > this.config.maxRetries) {
      return false;
    }

    if (error instanceof HttpException) {
      return this.config.retryableStatuses.includes(error.getStatus());
    }

    // Retry network errors
    if (error.code === 'ECONNREFUSED' || error.code === 'ETIMEDOUT') {
      return true;
    }

    return false;
  }

  private calculateDelay(attempt: number): number {
    // Exponential backoff with jitter
    const exponentialDelay = this.config.initialDelay * Math.pow(2, attempt);
    const jitter = Math.random() * 1000; // Add up to 1 second of jitter
    return Math.min(exponentialDelay + jitter, this.config.maxDelay);
  }
}
```

---

## 11. Request/Response Transformation

### Request Header Injection

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { v4 as uuidv4 } from 'uuid';

@Injectable()
export class RequestEnrichmentInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();

    // Add correlation ID
    request.correlationId = request.headers['x-correlation-id'] || uuidv4();

    // Add request timestamp
    request.timestamp = new Date();

    // Normalize headers
    request.normalizedHeaders = {
      userAgent: request.headers['user-agent'],
      acceptLanguage: request.headers['accept-language'] || 'en',
      contentType: request.headers['content-type'],
    };

    return next.handle();
  }
}
```

### Response Header Modification

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class ResponseHeadersInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const response = context.switchToHttp().getResponse();
    const request = context.switchToHttp().getRequest();
    const startTime = Date.now();

    return next.handle().pipe(
      tap(() => {
        // Add custom headers
        response.setHeader('X-Request-ID', request.correlationId || 'unknown');
        response.setHeader('X-Response-Time', `${Date.now() - startTime}ms`);
        response.setHeader('X-Powered-By', 'NestJS');

        // Security headers
        response.setHeader('X-Content-Type-Options', 'nosniff');
        response.setHeader('X-Frame-Options', 'DENY');
        response.setHeader('X-XSS-Protection', '1; mode=block');
      }),
    );
  }
}
```

### Request Body Transformation

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import * as sanitizeHtml from 'sanitize-html';

@Injectable()
export class SanitizeInputInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();

    if (request.body) {
      request.body = this.sanitizeObject(request.body);
    }

    return next.handle();
  }

  private sanitizeObject(obj: any): any {
    if (typeof obj === 'string') {
      return sanitizeHtml(obj.trim(), {
        allowedTags: [],
        allowedAttributes: {},
      });
    }

    if (Array.isArray(obj)) {
      return obj.map((item) => this.sanitizeObject(item));
    }

    if (obj !== null && typeof obj === 'object') {
      const sanitized: any = {};
      for (const [key, value] of Object.entries(obj)) {
        sanitized[key] = this.sanitizeObject(value);
      }
      return sanitized;
    }

    return obj;
  }
}
```

### Complete Request/Response Wrapper

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

interface ApiResponse<T> {
  success: boolean;
  data: T;
  meta: {
    timestamp: string;
    path: string;
    method: string;
    duration: number;
    version: string;
  };
}

@Injectable()
export class ApiResponseInterceptor<T>
  implements NestInterceptor<T, ApiResponse<T>>
{
  private readonly apiVersion = '1.0.0';

  intercept(
    context: ExecutionContext,
    next: CallHandler<T>,
  ): Observable<ApiResponse<T>> {
    const request = context.switchToHttp().getRequest();
    const startTime = Date.now();

    return next.handle().pipe(
      map((data) => ({
        success: true,
        data,
        meta: {
          timestamp: new Date().toISOString(),
          path: request.url,
          method: request.method,
          duration: Date.now() - startTime,
          version: this.apiVersion,
        },
      })),
    );
  }
}
```

---

## 12. Performance Timing

### Basic Performance Timer

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  Logger,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class PerformanceInterceptor implements NestInterceptor {
  private readonly logger = new Logger('Performance');

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const { method, url } = request;
    const startTime = process.hrtime.bigint();

    return next.handle().pipe(
      tap(() => {
        const endTime = process.hrtime.bigint();
        const durationNs = Number(endTime - startTime);
        const durationMs = durationNs / 1e6;

        this.logger.log(
          `${method} ${url} - ${durationMs.toFixed(2)}ms`,
        );

        // Log slow requests
        if (durationMs > 1000) {
          this.logger.warn(
            `Slow request detected: ${method} ${url} took ${durationMs.toFixed(2)}ms`,
          );
        }
      }),
    );
  }
}
```

### Detailed Performance Metrics

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  Logger,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap, finalize } from 'rxjs/operators';

interface PerformanceMetrics {
  requestId: string;
  method: string;
  path: string;
  controller: string;
  handler: string;
  startTime: Date;
  endTime: Date;
  duration: number;
  memoryUsage: {
    heapUsed: number;
    heapTotal: number;
    external: number;
  };
  statusCode: number;
}

@Injectable()
export class DetailedPerformanceInterceptor implements NestInterceptor {
  private readonly logger = new Logger('PerformanceMetrics');

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const response = context.switchToHttp().getResponse();
    const startTime = new Date();
    const startMemory = process.memoryUsage();
    const startHrTime = process.hrtime.bigint();

    const controller = context.getClass().name;
    const handler = context.getHandler().name;

    return next.handle().pipe(
      finalize(() => {
        const endTime = new Date();
        const endHrTime = process.hrtime.bigint();
        const endMemory = process.memoryUsage();

        const metrics: PerformanceMetrics = {
          requestId: request.requestId || 'unknown',
          method: request.method,
          path: request.url,
          controller,
          handler,
          startTime,
          endTime,
          duration: Number(endHrTime - startHrTime) / 1e6,
          memoryUsage: {
            heapUsed: endMemory.heapUsed - startMemory.heapUsed,
            heapTotal: endMemory.heapTotal,
            external: endMemory.external,
          },
          statusCode: response.statusCode,
        };

        this.logger.log(JSON.stringify(metrics));

        // Send to metrics service (e.g., Prometheus, DataDog)
        this.recordMetrics(metrics);
      }),
    );
  }

  private recordMetrics(metrics: PerformanceMetrics): void {
    // Implementation for sending metrics to monitoring system
    // Example: prometheus client, datadog agent, etc.
  }
}
```

### Database Query Performance Tracking

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  Logger,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class QueryPerformanceInterceptor implements NestInterceptor {
  private readonly logger = new Logger('QueryPerformance');

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const queryLog: { query: string; duration: number }[] = [];

    // Hook into query logging (example with TypeORM)
    const originalQuery = this.captureQueries(queryLog);

    return next.handle().pipe(
      tap(() => {
        // Restore original query method
        this.restoreQueries(originalQuery);

        // Log query statistics
        if (queryLog.length > 0) {
          const totalQueryTime = queryLog.reduce(
            (sum, q) => sum + q.duration,
            0,
          );

          this.logger.log({
            totalQueries: queryLog.length,
            totalQueryTime: `${totalQueryTime.toFixed(2)}ms`,
            queries: queryLog,
          });

          // Warn about N+1 queries
          if (queryLog.length > 10) {
            this.logger.warn(
              `Potential N+1 query detected: ${queryLog.length} queries executed`,
            );
          }
        }
      }),
    );
  }

  private captureQueries(queryLog: any[]): any {
    // Implementation depends on ORM used
    // This is a placeholder for query capture logic
    return null;
  }

  private restoreQueries(original: any): void {
    // Restore original query behavior
  }
}
```

---

## 13. Binding Interceptors (Method, Controller, Global)

### Method-Level Binding

```typescript
import { Controller, Get, UseInterceptors } from '@nestjs/common';
import { LoggingInterceptor } from './interceptors/logging.interceptor';
import { CacheInterceptor } from './interceptors/cache.interceptor';

@Controller('users')
export class UsersController {
  // Single interceptor
  @Get()
  @UseInterceptors(LoggingInterceptor)
  findAll() {
    return this.usersService.findAll();
  }

  // Multiple interceptors (executed in order)
  @Get(':id')
  @UseInterceptors(LoggingInterceptor, CacheInterceptor)
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(+id);
  }
}
```

### Controller-Level Binding

```typescript
import { Controller, Get, UseInterceptors } from '@nestjs/common';
import { LoggingInterceptor } from './interceptors/logging.interceptor';

@Controller('products')
@UseInterceptors(LoggingInterceptor)
export class ProductsController {
  @Get()
  findAll() {
    return this.productsService.findAll();
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.productsService.findOne(+id);
  }
}
```

### Global Interceptors

#### Method 1: In main.ts (without DI)

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { LoggingInterceptor } from './interceptors/logging.interceptor';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Global interceptor without dependency injection
  app.useGlobalInterceptors(new LoggingInterceptor());

  await app.listen(3000);
}
bootstrap();
```

#### Method 2: In Module (with DI support)

```typescript
import { Module } from '@nestjs/common';
import { APP_INTERCEPTOR } from '@nestjs/core';
import { LoggingInterceptor } from './interceptors/logging.interceptor';

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: LoggingInterceptor,
    },
  ],
})
export class AppModule {}
```

#### Method 3: Multiple Global Interceptors with Order

```typescript
import { Module } from '@nestjs/common';
import { APP_INTERCEPTOR } from '@nestjs/core';
import { LoggingInterceptor } from './interceptors/logging.interceptor';
import { TransformInterceptor } from './interceptors/transform.interceptor';
import { TimeoutInterceptor } from './interceptors/timeout.interceptor';

@Module({
  providers: [
    // Interceptors execute in the order they are registered
    {
      provide: APP_INTERCEPTOR,
      useClass: TimeoutInterceptor,
    },
    {
      provide: APP_INTERCEPTOR,
      useClass: LoggingInterceptor,
    },
    {
      provide: APP_INTERCEPTOR,
      useClass: TransformInterceptor,
    },
  ],
})
export class AppModule {}
```

### Interceptor with Factory

```typescript
import { Module } from '@nestjs/common';
import { APP_INTERCEPTOR } from '@nestjs/core';
import { ConfigService } from '@nestjs/config';
import { TimeoutInterceptor } from './interceptors/timeout.interceptor';

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useFactory: (configService: ConfigService) => {
        const timeout = configService.get<number>('REQUEST_TIMEOUT', 30000);
        return new TimeoutInterceptor(timeout);
      },
      inject: [ConfigService],
    },
  ],
})
export class AppModule {}
```

### Excluding Routes from Global Interceptors

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  SetMetadata,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { Reflector } from '@nestjs/core';

export const SKIP_INTERCEPTOR = 'skipInterceptor';
export const SkipInterceptor = () => SetMetadata(SKIP_INTERCEPTOR, true);

@Injectable()
export class GlobalInterceptor implements NestInterceptor {
  constructor(private reflector: Reflector) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const skip = this.reflector.getAllAndOverride<boolean>(SKIP_INTERCEPTOR, [
      context.getHandler(),
      context.getClass(),
    ]);

    if (skip) {
      return next.handle();
    }

    // Normal interceptor logic
    return next.handle();
  }
}

// Usage in controller
@Controller('health')
export class HealthController {
  @Get()
  @SkipInterceptor()
  check() {
    return { status: 'ok' };
  }
}
```

---

## 14. Execution Context

### Understanding ExecutionContext

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class ContextExplorerInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    // Get the type of context (http, rpc, ws)
    const contextType = context.getType(); // 'http' | 'rpc' | 'ws'

    // Get handler method reference
    const handler = context.getHandler();
    const handlerName = handler.name;

    // Get controller class reference
    const controller = context.getClass();
    const controllerName = controller.name;

    // Get arguments host
    const args = context.getArgs();

    console.log({
      contextType,
      handlerName,
      controllerName,
      argsLength: args.length,
    });

    return next.handle();
  }
}
```

### HTTP Context

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { Request, Response } from 'express';

@Injectable()
export class HttpContextInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    if (context.getType() !== 'http') {
      return next.handle();
    }

    const ctx = context.switchToHttp();

    // Get Express Request object
    const request = ctx.getRequest<Request>();
    const {
      method,
      url,
      headers,
      body,
      query,
      params,
      ip,
      cookies,
    } = request;

    // Get Express Response object
    const response = ctx.getResponse<Response>();

    // Get next function (rarely needed in interceptors)
    const nextFn = ctx.getNext();

    console.log({
      method,
      url,
      contentType: headers['content-type'],
      bodyKeys: Object.keys(body || {}),
      queryParams: query,
      routeParams: params,
      clientIp: ip,
    });

    return next.handle();
  }
}
```

### WebSocket Context

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class WsContextInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    if (context.getType() !== 'ws') {
      return next.handle();
    }

    const wsContext = context.switchToWs();

    // Get WebSocket client
    const client = wsContext.getClient();

    // Get message data
    const data = wsContext.getData();

    // Get pattern (event name)
    const pattern = wsContext.getPattern();

    console.log({
      clientId: client.id,
      pattern,
      data,
    });

    return next.handle();
  }
}
```

### RPC (Microservices) Context

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class RpcContextInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    if (context.getType() !== 'rpc') {
      return next.handle();
    }

    const rpcContext = context.switchToRpc();

    // Get message data
    const data = rpcContext.getData();

    // Get context object (transport-specific)
    const ctx = rpcContext.getContext();

    console.log({
      data,
      context: ctx,
    });

    return next.handle();
  }
}
```

### Using Reflector with ExecutionContext

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  SetMetadata,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { Reflector } from '@nestjs/core';

// Custom metadata keys
export const ROLES_KEY = 'roles';
export const IS_PUBLIC_KEY = 'isPublic';

// Custom decorators
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);

@Injectable()
export class MetadataInterceptor implements NestInterceptor {
  constructor(private reflector: Reflector) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    // Get metadata from handler only
    const handlerRoles = this.reflector.get<string[]>(
      ROLES_KEY,
      context.getHandler(),
    );

    // Get metadata from class only
    const classRoles = this.reflector.get<string[]>(
      ROLES_KEY,
      context.getClass(),
    );

    // Get metadata from both (handler takes precedence)
    const roles = this.reflector.getAllAndOverride<string[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    // Merge metadata from both
    const mergedRoles = this.reflector.getAllAndMerge<string[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    // Check if route is public
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    console.log({
      handlerRoles,
      classRoles,
      roles,
      mergedRoles,
      isPublic,
    });

    return next.handle();
  }
}
```

---

## 15. Dependency Injection in Interceptors

### Basic Dependency Injection

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';
import { MetricsService } from '../services/metrics.service';
import { LoggerService } from '../services/logger.service';

@Injectable()
export class MetricsInterceptor implements NestInterceptor {
  constructor(
    private readonly metricsService: MetricsService,
    private readonly logger: LoggerService,
  ) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const startTime = Date.now();

    return next.handle().pipe(
      tap({
        next: () => {
          const duration = Date.now() - startTime;
          this.metricsService.recordRequestDuration(
            request.method,
            request.route.path,
            duration,
          );
          this.logger.log(`Request completed in ${duration}ms`);
        },
        error: (error) => {
          this.metricsService.incrementErrorCount(
            request.method,
            request.route.path,
          );
          this.logger.error(`Request failed: ${error.message}`);
        },
      }),
    );
  }
}
```

### Injecting Configuration

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  Inject,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { ConfigService } from '@nestjs/config';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';

@Injectable()
export class ConfigurableCacheInterceptor implements NestInterceptor {
  private readonly enabled: boolean;
  private readonly ttl: number;

  constructor(
    private readonly configService: ConfigService,
    @Inject(CACHE_MANAGER) private readonly cacheManager: Cache,
  ) {
    this.enabled = this.configService.get<boolean>('CACHE_ENABLED', true);
    this.ttl = this.configService.get<number>('CACHE_TTL', 300);
  }

  async intercept(
    context: ExecutionContext,
    next: CallHandler,
  ): Promise<Observable<any>> {
    if (!this.enabled) {
      return next.handle();
    }

    const request = context.switchToHttp().getRequest();
    const cacheKey = `cache:${request.url}`;

    const cached = await this.cacheManager.get(cacheKey);
    if (cached) {
      return of(cached);
    }

    return next.handle().pipe(
      tap(async (data) => {
        await this.cacheManager.set(cacheKey, data, this.ttl * 1000);
      }),
    );
  }
}
```

### Injecting Request-Scoped Providers

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  Scope,
  Inject,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { REQUEST } from '@nestjs/core';
import { Request } from 'express';

// Request-scoped interceptor
@Injectable({ scope: Scope.REQUEST })
export class RequestScopedInterceptor implements NestInterceptor {
  constructor(@Inject(REQUEST) private readonly request: Request) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    // Access request directly from injected dependency
    const user = this.request['user'];

    console.log(`Processing request for user: ${user?.id}`);

    return next.handle();
  }
}
```

### Custom Provider Injection

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  Inject,
} from '@nestjs/common';
import { Observable } from 'rxjs';

// Custom token
export const INTERCEPTOR_OPTIONS = 'INTERCEPTOR_OPTIONS';

export interface InterceptorOptions {
  logLevel: 'debug' | 'info' | 'warn' | 'error';
  includeBody: boolean;
  excludePaths: string[];
}

@Injectable()
export class ConfigurableInterceptor implements NestInterceptor {
  constructor(
    @Inject(INTERCEPTOR_OPTIONS) private readonly options: InterceptorOptions,
  ) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();

    // Check if path should be excluded
    if (this.options.excludePaths.includes(request.path)) {
      return next.handle();
    }

    const logData: any = {
      method: request.method,
      path: request.path,
    };

    if (this.options.includeBody) {
      logData.body = request.body;
    }

    console[this.options.logLevel](logData);

    return next.handle();
  }
}

// Register in module
@Module({
  providers: [
    {
      provide: INTERCEPTOR_OPTIONS,
      useValue: {
        logLevel: 'info',
        includeBody: true,
        excludePaths: ['/health', '/metrics'],
      },
    },
    ConfigurableInterceptor,
  ],
})
export class AppModule {}
```

### Async Provider Injection

```typescript
import { Module } from '@nestjs/common';
import { APP_INTERCEPTOR } from '@nestjs/core';
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
  imports: [ConfigModule],
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useFactory: async (configService: ConfigService) => {
        // Perform async initialization
        const options = await loadInterceptorConfig(configService);
        return new DynamicInterceptor(options);
      },
      inject: [ConfigService],
    },
  ],
})
export class AppModule {}

async function loadInterceptorConfig(configService: ConfigService) {
  // Could load from database, external service, etc.
  return {
    timeout: configService.get('TIMEOUT'),
    retries: configService.get('RETRIES'),
  };
}
```

---

## 16. Interceptor vs Middleware vs Guards vs Pipes

### Comparison Overview

| Feature | Middleware | Guards | Interceptors | Pipes |
|---------|-----------|--------|--------------|-------|
| Execution Order | 1st | 2nd | 3rd (before) / Last (after) | 4th |
| Access to ExecutionContext | No | Yes | Yes | Partial |
| Can transform request | Yes | No | Yes | Yes (parameters) |
| Can transform response | No | No | Yes | No |
| Can prevent execution | Yes | Yes | Yes | Yes (throw) |
| Can access route handler result | No | No | Yes | No |
| DI Support | Limited | Full | Full | Full |

### Execution Order Diagram

```
Request
   │
   ▼
┌──────────────┐
│  Middleware  │  ← 1. Process request, can terminate
└──────────────┘
   │
   ▼
┌──────────────┐
│    Guards    │  ← 2. Authorization, can reject
└──────────────┘
   │
   ▼
┌──────────────┐
│ Interceptors │  ← 3. Pre-processing (before handler)
│   (before)   │
└──────────────┘
   │
   ▼
┌──────────────┐
│    Pipes     │  ← 4. Transform/validate parameters
└──────────────┘
   │
   ▼
┌──────────────┐
│   Handler    │  ← 5. Route handler execution
└──────────────┘
   │
   ▼
┌──────────────┐
│ Interceptors │  ← 6. Post-processing (after handler)
│   (after)    │
└──────────────┘
   │
   ▼
Response
```

### When to Use Each

#### Middleware - Use for:
- Request/response modification before routing
- Logging (basic)
- CORS handling
- Body parsing
- Session handling
- Rate limiting (basic)

```typescript
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log(`[${new Date().toISOString()}] ${req.method} ${req.url}`);
    next();
  }
}
```

#### Guards - Use for:
- Authentication
- Authorization
- Role-based access control
- Feature flags
- License validation

```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    return this.validateRequest(request);
  }

  private validateRequest(request: any): boolean {
    return !!request.headers.authorization;
  }
}
```

#### Interceptors - Use for:
- Response transformation
- Response wrapping
- Caching
- Logging (detailed with timing)
- Exception mapping
- Timeout handling
- Retry logic
- Performance monitoring

```typescript
@Injectable()
export class TransformInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      map((data) => ({ success: true, data })),
    );
  }
}
```

#### Pipes - Use for:
- Input validation
- Input transformation
- Data sanitization
- Type conversion

```typescript
import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class ParseIntPipe implements PipeTransform<string, number> {
  transform(value: string, metadata: ArgumentMetadata): number {
    const val = parseInt(value, 10);
    if (isNaN(val)) {
      throw new BadRequestException('Validation failed');
    }
    return val;
  }
}
```

### Combining All Together

```typescript
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { APP_GUARD, APP_INTERCEPTOR, APP_PIPE } from '@nestjs/core';

@Module({
  providers: [
    // Global pipe
    {
      provide: APP_PIPE,
      useClass: ValidationPipe,
    },
    // Global guard
    {
      provide: APP_GUARD,
      useClass: AuthGuard,
    },
    // Global interceptors (order matters)
    {
      provide: APP_INTERCEPTOR,
      useClass: LoggingInterceptor,
    },
    {
      provide: APP_INTERCEPTOR,
      useClass: TransformInterceptor,
    },
  ],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware, CorrelationIdMiddleware)
      .forRoutes('*');
  }
}
```

---

## 17. Best Practices

### 1. Keep Interceptors Focused

```typescript
// BAD: Interceptor doing too many things
@Injectable()
export class EverythingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    // Logging
    // Caching
    // Transformation
    // Error handling
    // Metrics
    // All in one interceptor
  }
}

// GOOD: Single responsibility interceptors
@Injectable()
export class LoggingInterceptor implements NestInterceptor { /* ... */ }

@Injectable()
export class CacheInterceptor implements NestInterceptor { /* ... */ }

@Injectable()
export class TransformInterceptor implements NestInterceptor { /* ... */ }
```

### 2. Handle Errors Properly

```typescript
@Injectable()
export class SafeInterceptor implements NestInterceptor {
  private readonly logger = new Logger(SafeInterceptor.name);

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      catchError((error) => {
        // Log the error
        this.logger.error(`Error in interceptor: ${error.message}`, error.stack);

        // Re-throw to let exception filters handle it
        return throwError(() => error);
      }),
    );
  }
}
```

### 3. Use Proper TypeScript Generics

```typescript
// GOOD: Properly typed interceptor
@Injectable()
export class TypedTransformInterceptor<T>
  implements NestInterceptor<T, Response<T>>
{
  intercept(
    context: ExecutionContext,
    next: CallHandler<T>,
  ): Observable<Response<T>> {
    return next.handle().pipe(
      map((data) => ({
        success: true,
        data,
        timestamp: new Date().toISOString(),
      })),
    );
  }
}
```

### 4. Make Interceptors Configurable

```typescript
// Create interceptor with options
export function createLoggingInterceptor(options: LoggingOptions) {
  @Injectable()
  class ConfiguredLoggingInterceptor implements NestInterceptor {
    intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
      if (!options.enabled) {
        return next.handle();
      }
      // Logging logic using options
    }
  }
  return ConfiguredLoggingInterceptor;
}

// Or use factory provider
@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useFactory: (config: ConfigService) => {
        return new LoggingInterceptor({
          level: config.get('LOG_LEVEL'),
          includeBody: config.get('LOG_BODY'),
        });
      },
      inject: [ConfigService],
    },
  ],
})
export class AppModule {}
```

### 5. Consider Performance Impact

```typescript
@Injectable()
export class PerformantInterceptor implements NestInterceptor {
  // Cache expensive computations
  private readonly compiledPatterns: Map<string, RegExp> = new Map();

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    // Avoid creating new objects in hot paths
    const request = context.switchToHttp().getRequest();

    // Use cached values when possible
    const pattern = this.getOrCreatePattern(request.path);

    return next.handle();
  }

  private getOrCreatePattern(path: string): RegExp {
    if (!this.compiledPatterns.has(path)) {
      this.compiledPatterns.set(path, new RegExp(path));
    }
    return this.compiledPatterns.get(path)!;
  }
}
```

### 6. Test Interceptors Properly

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { ExecutionContext, CallHandler } from '@nestjs/common';
import { of } from 'rxjs';
import { LoggingInterceptor } from './logging.interceptor';

describe('LoggingInterceptor', () => {
  let interceptor: LoggingInterceptor;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [LoggingInterceptor],
    }).compile();

    interceptor = module.get<LoggingInterceptor>(LoggingInterceptor);
  });

  it('should be defined', () => {
    expect(interceptor).toBeDefined();
  });

  it('should log request and response', (done) => {
    const mockExecutionContext = {
      switchToHttp: () => ({
        getRequest: () => ({ method: 'GET', url: '/test' }),
        getResponse: () => ({ statusCode: 200 }),
      }),
      getClass: () => ({ name: 'TestController' }),
      getHandler: () => ({ name: 'testMethod' }),
    } as unknown as ExecutionContext;

    const mockCallHandler: CallHandler = {
      handle: () => of({ data: 'test' }),
    };

    const logSpy = jest.spyOn(console, 'log');

    interceptor.intercept(mockExecutionContext, mockCallHandler).subscribe({
      next: (value) => {
        expect(value).toEqual({ data: 'test' });
        expect(logSpy).toHaveBeenCalled();
        done();
      },
    });
  });
});
```

### 7. Document Interceptor Behavior

```typescript
/**
 * TransformResponseInterceptor
 *
 * Wraps all successful responses in a standardized format.
 *
 * @example
 * // Before transformation
 * { id: 1, name: 'John' }
 *
 * // After transformation
 * {
 *   success: true,
 *   data: { id: 1, name: 'John' },
 *   timestamp: '2024-01-15T10:30:00.000Z'
 * }
 *
 * @remarks
 * - Only transforms successful responses (non-error)
 * - Does not modify error responses
 * - Can be skipped using @SkipTransform() decorator
 *
 * @see {@link SkipTransform} for excluding specific routes
 */
@Injectable()
export class TransformResponseInterceptor implements NestInterceptor {
  // Implementation
}
```

### 8. Use Decorators for Cleaner Code

```typescript
// Create composite decorators
import { applyDecorators, UseInterceptors, SetMetadata } from '@nestjs/common';

export function Cached(ttl: number = 300) {
  return applyDecorators(
    SetMetadata('cache_ttl', ttl),
    UseInterceptors(CacheInterceptor),
  );
}

export function Logged() {
  return applyDecorators(UseInterceptors(LoggingInterceptor));
}

export function Timed(timeout: number = 5000) {
  return applyDecorators(
    SetMetadata('timeout', timeout),
    UseInterceptors(TimeoutInterceptor),
  );
}

// Usage
@Controller('products')
export class ProductsController {
  @Get()
  @Cached(600)
  @Logged()
  @Timed(10000)
  findAll() {
    return this.productsService.findAll();
  }
}
```

### 9. Handle Async Operations Correctly

```typescript
@Injectable()
export class AsyncInterceptor implements NestInterceptor {
  constructor(private readonly asyncService: AsyncService) {}

  async intercept(
    context: ExecutionContext,
    next: CallHandler,
  ): Promise<Observable<any>> {
    // Await async operations before returning Observable
    const config = await this.asyncService.getConfig();

    if (!config.enabled) {
      return next.handle();
    }

    return next.handle().pipe(
      mergeMap(async (data) => {
        // Handle async operations in pipe
        const enriched = await this.asyncService.enrich(data);
        return enriched;
      }),
    );
  }
}
```

### 10. Implement Graceful Degradation

```typescript
@Injectable()
export class ResilientCacheInterceptor implements NestInterceptor {
  private readonly logger = new Logger(ResilientCacheInterceptor.name);

  constructor(@Inject(CACHE_MANAGER) private cacheManager: Cache) {}

  async intercept(
    context: ExecutionContext,
    next: CallHandler,
  ): Promise<Observable<any>> {
    const request = context.switchToHttp().getRequest();
    const cacheKey = `cache:${request.url}`;

    try {
      const cached = await this.cacheManager.get(cacheKey);
      if (cached) {
        return of(cached);
      }
    } catch (error) {
      // Cache failure should not break the request
      this.logger.warn(`Cache read failed: ${error.message}`);
    }

    return next.handle().pipe(
      tap(async (data) => {
        try {
          await this.cacheManager.set(cacheKey, data, 300000);
        } catch (error) {
          // Cache write failure should not affect response
          this.logger.warn(`Cache write failed: ${error.message}`);
        }
      }),
    );
  }
}
```

---

## Summary

Interceptors are a powerful feature in NestJS that provide a clean way to:

1. **Transform responses** - Wrap data in standard formats, handle pagination
2. **Handle cross-cutting concerns** - Logging, caching, metrics
3. **Implement retry and timeout logic** - Resilient request handling
4. **Map exceptions** - Convert errors to appropriate HTTP responses
5. **Measure performance** - Track request duration and resource usage

Key points to remember:

- Interceptors execute in the order they are bound
- Use `CallHandler.handle()` to invoke the route handler
- Leverage RxJS operators for powerful stream manipulation
- Inject dependencies through the constructor
- Use `ExecutionContext` to access request/response and metadata
- Keep interceptors focused on single responsibilities
- Test interceptors thoroughly with mocked contexts

By following these patterns and best practices, you can create maintainable, reusable, and efficient interceptors that enhance your NestJS applications.
