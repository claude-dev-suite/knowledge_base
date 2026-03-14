# NestJS Guards

> Official Documentation: https://docs.nestjs.com/guards

## Overview

Guards determine whether a given request will be handled by the route handler or not. They implement authorization logic, running after middleware but before interceptors and pipes. Guards have access to the ExecutionContext, making them ideal for implementing authentication and role-based access control.

## Guard Basics and CanActivate Interface

Every guard must implement the `CanActivate` interface with a single `canActivate()` method:

```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    return this.validateRequest(request);
  }

  private validateRequest(request: any): boolean {
    // Validate authentication token
    const authHeader = request.headers.authorization;
    if (!authHeader) {
      return false;
    }
    // Perform token validation logic
    return true;
  }
}
```

The `canActivate()` method can return:
- **boolean**: Synchronous decision
- **Promise<boolean>**: Asynchronous decision
- **Observable<boolean>**: Reactive decision

If the guard returns `false`, NestJS throws a `ForbiddenException` (HTTP 403).

## Execution Context

The `ExecutionContext` extends `ArgumentsHost` and provides additional methods to inspect the current request pipeline:

```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';

@Injectable()
export class ContextAwareGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    // Get the class (controller) the handler belongs to
    const controller = context.getClass();
    console.log('Controller:', controller.name);

    // Get the handler (method) being called
    const handler = context.getHandler();
    console.log('Handler:', handler.name);

    // Get the type of application context (http, ws, rpc)
    const contextType = context.getType();
    console.log('Context type:', contextType);

    // Switch to HTTP context for web requests
    if (contextType === 'http') {
      const httpContext = context.switchToHttp();
      const request = httpContext.getRequest();
      const response = httpContext.getResponse();
      const next = httpContext.getNext();

      return this.validateHttpRequest(request);
    }

    // Switch to WebSocket context
    if (contextType === 'ws') {
      const wsContext = context.switchToWs();
      const client = wsContext.getClient();
      const data = wsContext.getData();

      return this.validateWsClient(client);
    }

    // Switch to RPC (microservices) context
    if (contextType === 'rpc') {
      const rpcContext = context.switchToRpc();
      const data = rpcContext.getData();
      const rpcContext2 = rpcContext.getContext();

      return this.validateRpcRequest(data);
    }

    return false;
  }

  private validateHttpRequest(request: any): boolean {
    return !!request.headers.authorization;
  }

  private validateWsClient(client: any): boolean {
    return !!client.handshake?.auth?.token;
  }

  private validateRpcRequest(data: any): boolean {
    return !!data.credentials;
  }
}
```

## Role-Based Access Control

Implement role-based authorization using metadata and guards:

```typescript
// role.enum.ts
export enum Role {
  User = 'user',
  Admin = 'admin',
  Moderator = 'moderator',
  SuperAdmin = 'super_admin',
}

// roles.decorator.ts
import { SetMetadata } from '@nestjs/common';
import { Role } from './role.enum';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);

// roles.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Role } from './role.enum';
import { ROLES_KEY } from './roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    // Get required roles from metadata
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    // If no roles required, allow access
    if (!requiredRoles || requiredRoles.length === 0) {
      return true;
    }

    // Get user from request (set by auth guard)
    const { user } = context.switchToHttp().getRequest();

    if (!user || !user.roles) {
      return false;
    }

    // Check if user has any of the required roles
    return requiredRoles.some((role) => user.roles.includes(role));
  }
}
```

## Binding Guards (@UseGuards)

Guards can be bound at different levels using the `@UseGuards()` decorator:

```typescript
import { Controller, Get, Post, UseGuards } from '@nestjs/common';
import { AuthGuard } from './auth.guard';
import { RolesGuard } from './roles.guard';
import { Roles } from './roles.decorator';
import { Role } from './role.enum';

// Controller-level guard - applies to all routes
@Controller('users')
@UseGuards(AuthGuard, RolesGuard)
export class UsersController {
  @Get()
  @Roles(Role.User, Role.Admin)
  findAll() {
    return 'All users (requires User or Admin role)';
  }

  @Get(':id')
  @Roles(Role.User)
  findOne() {
    return 'Single user (requires User role)';
  }

  @Post()
  @Roles(Role.Admin)
  create() {
    return 'Create user (requires Admin role)';
  }
}

// Route-level guard - applies only to specific routes
@Controller('posts')
export class PostsController {
  @Get()
  findAll() {
    return 'Public route - no guard';
  }

  @Post()
  @UseGuards(AuthGuard)
  create() {
    return 'Protected route - auth required';
  }

  @Delete(':id')
  @UseGuards(AuthGuard, RolesGuard)
  @Roles(Role.Admin)
  remove() {
    return 'Delete post - admin only';
  }
}
```

You can also pass guard instances instead of classes:

```typescript
@UseGuards(new AuthGuard())
```

However, using classes is preferred as it enables dependency injection.

## Global Guards

Register guards globally to protect all routes:

```typescript
// Method 1: In main.ts (no dependency injection)
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { AuthGuard } from './auth/auth.guard';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalGuards(new AuthGuard());
  await app.listen(3000);
}
bootstrap();

// Method 2: Via module (with dependency injection - RECOMMENDED)
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';
import { AuthGuard } from './auth/auth.guard';
import { RolesGuard } from './auth/roles.guard';

@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: AuthGuard,
    },
    {
      provide: APP_GUARD,
      useClass: RolesGuard,
    },
  ],
})
export class AppModule {}
```

## Reflection and Metadata

Use `SetMetadata` and `Reflector` to attach and read custom metadata:

```typescript
import { SetMetadata, Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

// Define metadata keys
export const PERMISSIONS_KEY = 'permissions';
export const MIN_ROLE_LEVEL_KEY = 'minRoleLevel';

// Create metadata decorators
export const Permissions = (...permissions: string[]) =>
  SetMetadata(PERMISSIONS_KEY, permissions);
export const MinRoleLevel = (level: number) =>
  SetMetadata(MIN_ROLE_LEVEL_KEY, level);

// Guard using Reflector methods
@Injectable()
export class AdvancedRolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const { user } = context.switchToHttp().getRequest();

    // getAllAndOverride: Returns handler metadata, falls back to class metadata
    const permissions = this.reflector.getAllAndOverride<string[]>(
      PERMISSIONS_KEY,
      [context.getHandler(), context.getClass()],
    );

    // getAllAndMerge: Merges metadata from handler and class
    const allPermissions = this.reflector.getAllAndMerge<string[]>(
      PERMISSIONS_KEY,
      [context.getHandler(), context.getClass()],
    );

    // get: Gets metadata from a single target
    const handlerPermissions = this.reflector.get<string[]>(
      PERMISSIONS_KEY,
      context.getHandler(),
    );

    const classPermissions = this.reflector.get<string[]>(
      PERMISSIONS_KEY,
      context.getClass(),
    );

    // Get numeric metadata
    const minLevel = this.reflector.getAllAndOverride<number>(
      MIN_ROLE_LEVEL_KEY,
      [context.getHandler(), context.getClass()],
    );

    // Validate permissions
    if (permissions && permissions.length > 0) {
      const hasPermission = permissions.some((perm) =>
        user.permissions?.includes(perm),
      );
      if (!hasPermission) return false;
    }

    // Validate role level
    if (minLevel !== undefined && user.roleLevel < minLevel) {
      return false;
    }

    return true;
  }
}
```

## Custom Decorators for Roles/Permissions

Create reusable composite decorators:

```typescript
// auth.decorator.ts
import { applyDecorators, SetMetadata, UseGuards } from '@nestjs/common';
import { AuthGuard } from './auth.guard';
import { RolesGuard } from './roles.guard';
import { Role } from './role.enum';

// Composite decorator for common auth patterns
export function Auth(...roles: Role[]) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(AuthGuard, RolesGuard),
  );
}

// Public route decorator for global guard scenarios
export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);

// Permission-based decorator
export const PERMISSIONS_KEY = 'permissions';
export const RequirePermissions = (...permissions: string[]) =>
  SetMetadata(PERMISSIONS_KEY, permissions);

// Combined permissions decorator with guard
export function Authorized(...permissions: string[]) {
  return applyDecorators(
    SetMetadata(PERMISSIONS_KEY, permissions),
    UseGuards(AuthGuard, PermissionsGuard),
  );
}

// Usage in controller
@Controller('admin')
export class AdminController {
  @Get('dashboard')
  @Auth(Role.Admin, Role.SuperAdmin)
  getDashboard() {
    return 'Admin dashboard';
  }

  @Get('stats')
  @Authorized('read:stats', 'admin:access')
  getStats() {
    return 'Admin statistics';
  }

  @Get('health')
  @Public()
  healthCheck() {
    return 'OK';
  }
}
```

## JWT Authentication Guard

Complete JWT authentication guard with Passport integration:

```typescript
// jwt-auth.guard.ts
import {
  Injectable,
  ExecutionContext,
  UnauthorizedException,
} from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { Reflector } from '@nestjs/core';
import { IS_PUBLIC_KEY } from './public.decorator';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext) {
    // Check if route is marked as public
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    if (isPublic) {
      return true;
    }

    // Proceed with JWT validation
    return super.canActivate(context);
  }

  handleRequest(err: any, user: any, info: any, context: ExecutionContext) {
    // Handle specific JWT errors
    if (err || !user) {
      if (info?.name === 'TokenExpiredError') {
        throw new UnauthorizedException('Token has expired');
      }
      if (info?.name === 'JsonWebTokenError') {
        throw new UnauthorizedException('Invalid token');
      }
      if (info?.message === 'No auth token') {
        throw new UnauthorizedException('No authentication token provided');
      }
      throw err || new UnauthorizedException('Authentication failed');
    }
    return user;
  }
}

// jwt.strategy.ts (for reference)
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(private configService: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: configService.get<string>('JWT_SECRET'),
    });
  }

  async validate(payload: any) {
    // Return user object that will be attached to request
    return {
      id: payload.sub,
      email: payload.email,
      roles: payload.roles,
      permissions: payload.permissions,
    };
  }
}
```

## Multiple Guards Execution Order

Guards execute in a specific order based on where they are bound:

```typescript
// Execution order:
// 1. Global guards (in registration order)
// 2. Controller guards (left to right)
// 3. Route guards (left to right)

// Example demonstrating order
@Injectable()
export class LoggingGuard implements CanActivate {
  constructor(private name: string) {}

  canActivate(context: ExecutionContext): boolean {
    console.log(`Guard executed: ${this.name}`);
    return true;
  }
}

// main.ts - Global guards execute first
app.useGlobalGuards(
  new LoggingGuard('Global-1'),  // Executes 1st
  new LoggingGuard('Global-2'),  // Executes 2nd
);

// Controller with guards
@Controller('example')
@UseGuards(
  LoggingGuard('Controller-1'),  // Executes 3rd
  LoggingGuard('Controller-2'),  // Executes 4th
)
export class ExampleController {
  @Get()
  @UseGuards(
    LoggingGuard('Route-1'),     // Executes 5th
    LoggingGuard('Route-2'),     // Executes 6th
  )
  example() {
    return 'Response';
  }
}

// Output:
// Guard executed: Global-1
// Guard executed: Global-2
// Guard executed: Controller-1
// Guard executed: Controller-2
// Guard executed: Route-1
// Guard executed: Route-2
```

## Combining Guards

Different strategies for combining multiple guards:

```typescript
// Strategy 1: Sequential guards (all must pass)
@UseGuards(AuthGuard, RolesGuard, ThrottlerGuard)

// Strategy 2: Composite guard (custom logic)
@Injectable()
export class CompositeGuard implements CanActivate {
  constructor(
    private authGuard: AuthGuard,
    private rolesGuard: RolesGuard,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    // First check authentication
    const isAuthenticated = await this.authGuard.canActivate(context);
    if (!isAuthenticated) {
      return false;
    }

    // Then check roles
    return this.rolesGuard.canActivate(context);
  }
}

// Strategy 3: OR logic (any guard can pass)
@Injectable()
export class EitherAuthGuard implements CanActivate {
  constructor(
    private jwtGuard: JwtAuthGuard,
    private apiKeyGuard: ApiKeyGuard,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    try {
      // Try JWT authentication first
      const jwtResult = await this.jwtGuard.canActivate(context);
      if (jwtResult) return true;
    } catch (e) {
      // JWT failed, try API key
    }

    try {
      // Try API key authentication
      const apiKeyResult = await this.apiKeyGuard.canActivate(context);
      if (apiKeyResult) return true;
    } catch (e) {
      // API key also failed
    }

    return false;
  }
}

// Strategy 4: Conditional guard based on context
@Injectable()
export class ConditionalGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const isAdmin = this.reflector.get<boolean>('isAdminRoute', context.getHandler());
    const request = context.switchToHttp().getRequest();

    if (isAdmin) {
      return request.user?.roles?.includes('admin');
    }

    return !!request.user;
  }
}
```

## Best Practices and Common Pitfalls

### Best Practices

```typescript
// 1. Always use class-based guards for dependency injection
@Injectable()
export class BestPracticeGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private userService: UserService,
    private configService: ConfigService,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    // Can use injected services
    return true;
  }
}

// 2. Throw specific exceptions for better error handling
import { ForbiddenException, UnauthorizedException } from '@nestjs/common';

@Injectable()
export class DetailedErrorGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();

    if (!request.headers.authorization) {
      throw new UnauthorizedException('No authorization header provided');
    }

    if (!request.user) {
      throw new UnauthorizedException('Invalid credentials');
    }

    if (!this.hasRequiredRole(request.user)) {
      throw new ForbiddenException('Insufficient permissions');
    }

    return true;
  }

  private hasRequiredRole(user: any): boolean {
    return user.roles?.includes('admin');
  }
}

// 3. Use async/await for database lookups
@Injectable()
export class AsyncGuard implements CanActivate {
  constructor(private userService: UserService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const userId = request.user?.id;

    if (!userId) return false;

    // Fetch fresh user data
    const user = await this.userService.findById(userId);

    // Update request with fresh data
    request.user = user;

    return user.isActive && !user.isBanned;
  }
}

// 4. Create a CurrentUser decorator for clean controller code
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: string | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;
    return data ? user?.[data] : user;
  },
);

// Usage
@Get('profile')
getProfile(@CurrentUser() user: User) {
  return user;
}

@Get('email')
getEmail(@CurrentUser('email') email: string) {
  return { email };
}
```

### Common Pitfalls

```typescript
// PITFALL 1: Forgetting guards don't have access to response body
// Guards run BEFORE the handler, not after

// PITFALL 2: Not handling async properly
// WRONG - Promise not awaited
@Injectable()
export class BrokenGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    // This returns immediately, not waiting for the promise!
    this.asyncCheck(); // Bug: not awaited
    return true;
  }

  async asyncCheck() {
    await someAsyncOperation();
  }
}

// CORRECT - Proper async handling
@Injectable()
export class FixedGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    await this.asyncCheck();
    return true;
  }
}

// PITFALL 3: Circular dependency with global guards
// WRONG - Will cause circular dependency
@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: GuardThatNeedsService, // If this service needs something from this module
    },
  ],
})

// CORRECT - Use forwardRef if needed
@Injectable()
export class SafeGuard implements CanActivate {
  constructor(
    @Inject(forwardRef(() => UserService))
    private userService: UserService,
  ) {}
}

// PITFALL 4: Not considering guard order with roles
// WRONG - RolesGuard before AuthGuard (user not set yet)
@UseGuards(RolesGuard, AuthGuard)

// CORRECT - AuthGuard first to populate request.user
@UseGuards(AuthGuard, RolesGuard)

// PITFALL 5: Blocking public routes with global guards
// Always implement @Public() decorator pattern for global guards
@Injectable()
export class GlobalAuthGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    // ALWAYS check for public routes first
    const isPublic = this.reflector.getAllAndOverride<boolean>('isPublic', [
      context.getHandler(),
      context.getClass(),
    ]);

    if (isPublic) {
      return true;
    }

    // Then perform authentication
    return this.authenticate(context);
  }

  private authenticate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    return !!request.headers.authorization;
  }
}
```

### Guard vs Middleware vs Interceptor

| Feature | Middleware | Guard | Interceptor |
|---------|-----------|-------|-------------|
| Access to ExecutionContext | No | Yes | Yes |
| Can prevent handler execution | Yes (by not calling next) | Yes (return false) | Yes (don't call next.handle()) |
| Access to handler metadata | No | Yes | Yes |
| Can transform response | No | No | Yes |
| Best for | Logging, CORS | Auth, Authorization | Response transform, caching |

## Summary

Guards are powerful tools for implementing authentication and authorization in NestJS applications. Key points:

1. Guards implement `CanActivate` and return boolean/Promise/Observable
2. Use `ExecutionContext` to access request details and metadata
3. Bind guards globally, at controller level, or route level
4. Use `Reflector` to read custom metadata from decorators
5. Guards execute in order: Global -> Controller -> Route
6. Always place `AuthGuard` before `RolesGuard`
7. Implement `@Public()` decorator for routes that bypass global guards
8. Throw specific exceptions for better error messages
