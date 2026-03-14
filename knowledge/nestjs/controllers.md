# NestJS Controllers

> Official Documentation: https://docs.nestjs.com/controllers

## Overview

Controllers are responsible for handling incoming HTTP requests and returning responses to the client. A controller's purpose is to receive specific requests for the application. The routing mechanism controls which controller receives which requests. Each controller can have multiple routes, and different routes can perform different actions.

---

## 1. Controller Basics and Routing

### The @Controller Decorator

The `@Controller()` decorator defines a basic controller. The optional route path prefix groups related routes and minimizes repetitive code.

```typescript
import { Controller, Get } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
```

### HTTP Method Decorators

NestJS provides decorators for all standard HTTP methods:

| Decorator     | HTTP Method | Description                    |
|---------------|-------------|--------------------------------|
| `@Get()`      | GET         | Retrieve resources             |
| `@Post()`     | POST        | Create new resources           |
| `@Put()`      | PUT         | Replace entire resource        |
| `@Patch()`    | PATCH       | Partial resource update        |
| `@Delete()`   | DELETE      | Remove resources               |
| `@Options()`  | OPTIONS     | Preflight requests             |
| `@Head()`     | HEAD        | Like GET but no response body  |
| `@All()`      | ALL         | Handles all HTTP methods       |

```typescript
import { Controller, Get, Post, Put, Patch, Delete } from '@nestjs/common';

@Controller('users')
export class UsersController {
  @Get()
  findAll() { return 'Get all users'; }

  @Get(':id')
  findOne() { return 'Get one user'; }

  @Post()
  create() { return 'Create user'; }

  @Put(':id')
  replace() { return 'Replace user'; }

  @Patch(':id')
  update() { return 'Update user'; }

  @Delete(':id')
  remove() { return 'Remove user'; }
}
```

---

## 2. Request Object Decorators

### Available Parameter Decorators

| Decorator                  | Express Equivalent        |
|----------------------------|---------------------------|
| `@Request()`, `@Req()`     | `req`                     |
| `@Response()`, `@Res()`    | `res`                     |
| `@Next()`                  | `next`                    |
| `@Session()`               | `req.session`             |
| `@Param(key?: string)`     | `req.params` / `req.params[key]` |
| `@Body(key?: string)`      | `req.body` / `req.body[key]` |
| `@Query(key?: string)`     | `req.query` / `req.query[key]` |
| `@Headers(name?: string)`  | `req.headers` / `req.headers[name]` |
| `@Ip()`                    | `req.ip`                  |
| `@HostParam()`             | `req.hosts`               |

### Usage Examples

```typescript
import {
  Controller,
  Get,
  Post,
  Req,
  Body,
  Param,
  Query,
  Headers,
  Ip,
} from '@nestjs/common';
import { Request } from 'express';
import { CreateCatDto } from './dto/create-cat.dto';

@Controller('cats')
export class CatsController {
  // Full request object access
  @Get()
  findAll(@Req() request: Request): string {
    console.log(request.url);
    console.log(request.method);
    return 'All cats';
  }

  // Route parameters
  @Get(':id')
  findOne(@Param('id') id: string): string {
    return `Cat #${id}`;
  }

  // Multiple route parameters
  @Get(':userId/posts/:postId')
  findUserPost(
    @Param('userId') userId: string,
    @Param('postId') postId: string,
  ): string {
    return `User ${userId}, Post ${postId}`;
  }

  // Query parameters
  @Get()
  findFiltered(
    @Query('page') page: number = 1,
    @Query('limit') limit: number = 10,
    @Query() allQuery: Record<string, any>,
  ) {
    return { page, limit, allQuery };
  }

  // Request body
  @Post()
  create(@Body() createCatDto: CreateCatDto) {
    return createCatDto;
  }

  // Specific body property
  @Post('name')
  createWithName(@Body('name') name: string) {
    return `Created cat named ${name}`;
  }

  // Headers
  @Get('auth')
  checkAuth(
    @Headers('authorization') auth: string,
    @Headers() allHeaders: Record<string, string>,
  ) {
    return { auth, headerCount: Object.keys(allHeaders).length };
  }

  // Client IP
  @Get('ip')
  getClientIp(@Ip() ip: string) {
    return { clientIp: ip };
  }
}
```

---

## 3. Response Handling

### Standard Approach (Recommended)

NestJS automatically serializes return values to JSON and sets appropriate headers.

```typescript
@Controller('cats')
export class CatsController {
  @Get()
  findAll(): Cat[] {
    return this.catsService.findAll();
  }

  @Get(':id')
  async findOne(@Param('id') id: string): Promise<Cat> {
    return this.catsService.findOne(id);
  }
}
```

### @HttpCode Decorator

POST requests return 201 by default; other methods return 200. Override with `@HttpCode()`.

```typescript
import { Controller, Post, HttpCode, HttpStatus } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  @HttpCode(HttpStatus.NO_CONTENT)  // 204
  create() {
    return;
  }

  @Post('accepted')
  @HttpCode(202)
  createAsync() {
    return { message: 'Processing' };
  }
}
```

### @Header Decorator

Set custom response headers using `@Header()`.

```typescript
import { Controller, Get, Header } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  @Header('Cache-Control', 'max-age=3600')
  @Header('X-Custom-Header', 'custom-value')
  findAll() {
    return this.catsService.findAll();
  }
}
```

### @Redirect Decorator

Redirect responses to different URLs.

```typescript
import { Controller, Get, Redirect, Query } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get('docs')
  @Redirect('https://docs.nestjs.com', 302)
  getDocs() {
    // Static redirect
  }

  @Get('dynamic')
  @Redirect('https://default.com', 301)
  dynamicRedirect(@Query('version') version: string) {
    // Override redirect by returning object
    if (version === 'v2') {
      return { url: 'https://v2.example.com', statusCode: 302 };
    }
  }
}
```

### Library-Specific Response (@Res)

Use `@Res()` for direct response object access. **Warning:** This bypasses interceptors and serialization.

```typescript
import { Controller, Get, Post, Res, HttpStatus } from '@nestjs/common';
import { Response } from 'express';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(@Res() res: Response) {
    res.status(HttpStatus.OK).json({ message: 'All cats' });
  }

  // Use passthrough to retain NestJS features while accessing response
  @Get('passthrough')
  findWithPassthrough(@Res({ passthrough: true }) res: Response) {
    res.header('X-Custom', 'value');
    return { message: 'Response with custom header' };
  }
}
```

---

## 4. Route Parameters and Wildcards

### Basic Route Parameters

```typescript
@Controller('users')
export class UsersController {
  @Get(':id')
  findOne(@Param('id') id: string) {
    return `User ${id}`;
  }

  // Get all params as object
  @Get(':category/:id')
  findByCategory(@Param() params: { category: string; id: string }) {
    return `Category: ${params.category}, ID: ${params.id}`;
  }
}
```

### Route Wildcards

Use asterisk `*` as a wildcard to match any combination of characters.

```typescript
@Controller('files')
export class FilesController {
  // Matches: /files/any/path/here
  @Get('*')
  findAll() {
    return 'Wildcard route';
  }

  // Matches: /files/ab_cd, /files/aXYZcd, etc.
  @Get('ab*cd')
  findPattern() {
    return 'Pattern matched';
  }
}
```

### Regular Expression Routes

```typescript
@Controller()
export class AppController {
  // Matches numeric IDs only
  @Get('users/:id(\\d+)')
  findNumericUser(@Param('id') id: string) {
    return `Numeric user ID: ${id}`;
  }
}
```

---

## 5. Sub-Domain Routing

Handle requests based on the host/subdomain using `@Controller()` with host option.

```typescript
import { Controller, Get, HostParam } from '@nestjs/common';

@Controller({ host: 'admin.example.com' })
export class AdminController {
  @Get()
  index(): string {
    return 'Admin panel';
  }
}

// Dynamic subdomain routing
@Controller({ host: ':account.example.com' })
export class AccountController {
  @Get()
  getAccount(@HostParam('account') account: string): string {
    return `Account: ${account}`;
  }

  @Get('dashboard')
  getDashboard(@HostParam('account') account: string): string {
    return `Dashboard for ${account}`;
  }
}
```

---

## 6. Async Handlers and Observables

### Async/Await (Promises)

NestJS natively supports async functions.

```typescript
import { Controller, Get, Param } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }

  @Get(':id')
  async findOne(@Param('id') id: string): Promise<Cat> {
    const cat = await this.catsService.findOne(id);
    if (!cat) {
      throw new NotFoundException(`Cat #${id} not found`);
    }
    return cat;
  }

  @Post()
  async create(@Body() dto: CreateCatDto): Promise<Cat> {
    return this.catsService.create(dto);
  }
}
```

### RxJS Observables

NestJS automatically subscribes to Observables and emits the last value.

```typescript
import { Controller, Get } from '@nestjs/common';
import { Observable, of } from 'rxjs';
import { map, delay } from 'rxjs/operators';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(): Observable<Cat[]> {
    return of(this.catsService.findAll());
  }

  @Get('delayed')
  findDelayed(): Observable<Cat[]> {
    return of(this.cats).pipe(
      delay(1000),
      map(cats => cats.filter(cat => cat.active)),
    );
  }
}
```

---

## 7. Request Payloads with DTOs

### Data Transfer Objects

DTOs define the structure of data for network transfer. Use classes (not interfaces) for runtime validation.

```typescript
// dto/create-cat.dto.ts
import { IsString, IsInt, IsOptional, Min, Max, IsArray } from 'class-validator';

export class CreateCatDto {
  @IsString()
  readonly name: string;

  @IsInt()
  @Min(0)
  @Max(30)
  readonly age: number;

  @IsString()
  readonly breed: string;

  @IsArray()
  @IsString({ each: true })
  @IsOptional()
  readonly tags?: string[];
}

// dto/update-cat.dto.ts
import { PartialType } from '@nestjs/mapped-types';
import { CreateCatDto } from './create-cat.dto';

export class UpdateCatDto extends PartialType(CreateCatDto) {}

// dto/query-cat.dto.ts
import { IsOptional, IsInt, Min, IsString } from 'class-validator';
import { Type } from 'class-transformer';

export class QueryCatDto {
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  page?: number = 1;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  limit?: number = 10;

  @IsOptional()
  @IsString()
  search?: string;
}
```

### Using DTOs in Controllers

```typescript
import { Controller, Get, Post, Put, Body, Param, Query } from '@nestjs/common';
import { CreateCatDto } from './dto/create-cat.dto';
import { UpdateCatDto } from './dto/update-cat.dto';
import { QueryCatDto } from './dto/query-cat.dto';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(@Query() query: QueryCatDto) {
    return this.catsService.findAll(query);
  }

  @Post()
  create(@Body() createCatDto: CreateCatDto) {
    return this.catsService.create(createCatDto);
  }

  @Put(':id')
  update(@Param('id') id: string, @Body() updateCatDto: UpdateCatDto) {
    return this.catsService.update(id, updateCatDto);
  }
}
```

---

## 8. Handling Errors

### Built-in HTTP Exceptions

```typescript
import {
  Controller,
  Get,
  Post,
  Param,
  Body,
  NotFoundException,
  BadRequestException,
  ConflictException,
  ForbiddenException,
  UnauthorizedException,
  InternalServerErrorException,
} from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get(':id')
  async findOne(@Param('id') id: string) {
    const cat = await this.catsService.findOne(id);
    if (!cat) {
      throw new NotFoundException(`Cat with ID ${id} not found`);
    }
    return cat;
  }

  @Post()
  async create(@Body() dto: CreateCatDto) {
    const existing = await this.catsService.findByName(dto.name);
    if (existing) {
      throw new ConflictException('Cat with this name already exists');
    }
    return this.catsService.create(dto);
  }

  @Get('protected')
  protectedRoute() {
    throw new ForbiddenException('Access denied');
  }
}
```

### Custom Exception Response

```typescript
throw new BadRequestException({
  statusCode: 400,
  message: 'Validation failed',
  errors: [
    { field: 'name', message: 'Name is required' },
    { field: 'age', message: 'Age must be positive' },
  ],
});
```

### All Built-in Exceptions

| Exception                       | Status Code |
|---------------------------------|-------------|
| `BadRequestException`           | 400         |
| `UnauthorizedException`         | 401         |
| `NotFoundException`             | 404         |
| `ForbiddenException`            | 403         |
| `NotAcceptableException`        | 406         |
| `RequestTimeoutException`       | 408         |
| `ConflictException`             | 409         |
| `GoneException`                 | 410         |
| `PayloadTooLargeException`      | 413         |
| `UnsupportedMediaTypeException` | 415         |
| `UnprocessableEntityException`  | 422         |
| `InternalServerErrorException`  | 500         |
| `NotImplementedException`       | 501         |
| `BadGatewayException`           | 502         |
| `ServiceUnavailableException`   | 503         |
| `GatewayTimeoutException`       | 504         |

---

## 9. Full Resource Sample

Complete CRUD controller with all common patterns:

```typescript
import {
  Controller,
  Get,
  Post,
  Put,
  Patch,
  Delete,
  Body,
  Param,
  Query,
  HttpCode,
  HttpStatus,
  Header,
  ParseIntPipe,
  ValidationPipe,
  NotFoundException,
  UseGuards,
  UseInterceptors,
} from '@nestjs/common';
import { CatsService } from './cats.service';
import { CreateCatDto } from './dto/create-cat.dto';
import { UpdateCatDto } from './dto/update-cat.dto';
import { QueryCatDto } from './dto/query-cat.dto';
import { Cat } from './entities/cat.entity';
import { AuthGuard } from '../auth/auth.guard';
import { LoggingInterceptor } from '../common/logging.interceptor';

@Controller('cats')
@UseInterceptors(LoggingInterceptor)
export class CatsController {
  constructor(private readonly catsService: CatsService) {}

  // GET /cats
  @Get()
  @Header('Cache-Control', 'max-age=60')
  async findAll(@Query(ValidationPipe) query: QueryCatDto): Promise<Cat[]> {
    return this.catsService.findAll(query);
  }

  // GET /cats/:id
  @Get(':id')
  async findOne(@Param('id', ParseIntPipe) id: number): Promise<Cat> {
    const cat = await this.catsService.findOne(id);
    if (!cat) {
      throw new NotFoundException(`Cat #${id} not found`);
    }
    return cat;
  }

  // POST /cats
  @Post()
  @HttpCode(HttpStatus.CREATED)
  @UseGuards(AuthGuard)
  async create(@Body(ValidationPipe) createCatDto: CreateCatDto): Promise<Cat> {
    return this.catsService.create(createCatDto);
  }

  // PUT /cats/:id (full replacement)
  @Put(':id')
  @UseGuards(AuthGuard)
  async replace(
    @Param('id', ParseIntPipe) id: number,
    @Body(ValidationPipe) createCatDto: CreateCatDto,
  ): Promise<Cat> {
    return this.catsService.replace(id, createCatDto);
  }

  // PATCH /cats/:id (partial update)
  @Patch(':id')
  @UseGuards(AuthGuard)
  async update(
    @Param('id', ParseIntPipe) id: number,
    @Body(ValidationPipe) updateCatDto: UpdateCatDto,
  ): Promise<Cat> {
    return this.catsService.update(id, updateCatDto);
  }

  // DELETE /cats/:id
  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  @UseGuards(AuthGuard)
  async remove(@Param('id', ParseIntPipe) id: number): Promise<void> {
    const deleted = await this.catsService.remove(id);
    if (!deleted) {
      throw new NotFoundException(`Cat #${id} not found`);
    }
  }

  // GET /cats/:id/owner
  @Get(':id/owner')
  async findOwner(@Param('id', ParseIntPipe) id: number) {
    return this.catsService.findOwner(id);
  }

  // POST /cats/:id/feed
  @Post(':id/feed')
  @HttpCode(HttpStatus.OK)
  async feedCat(@Param('id', ParseIntPipe) id: number) {
    return this.catsService.feed(id);
  }
}
```

---

## 10. Library-Specific Approach

### Express-Specific Features

```typescript
import { Controller, Get, Req, Res, Next } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Controller('express')
export class ExpressController {
  @Get()
  handleRequest(
    @Req() req: Request,
    @Res({ passthrough: true }) res: Response,
  ) {
    // Access Express-specific features
    res.cookie('session', 'value', { httpOnly: true });
    res.header('X-Custom', 'value');

    return { message: 'Using Express features' };
  }

  @Get('stream')
  streamResponse(@Res() res: Response) {
    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');

    let count = 0;
    const interval = setInterval(() => {
      res.write(`data: ${count++}\n\n`);
      if (count > 5) {
        clearInterval(interval);
        res.end();
      }
    }, 1000);
  }
}
```

### Fastify-Specific Features

```typescript
import { Controller, Get, Req, Res } from '@nestjs/common';
import { FastifyRequest, FastifyReply } from 'fastify';

@Controller('fastify')
export class FastifyController {
  @Get()
  handleRequest(
    @Req() req: FastifyRequest,
    @Res({ passthrough: true }) reply: FastifyReply,
  ) {
    reply.header('X-Custom', 'value');
    return { message: 'Using Fastify' };
  }
}
```

---

## 11. Best Practices and Common Pitfalls

### Best Practices

```typescript
// 1. Keep controllers thin - delegate to services
@Controller('cats')
export class CatsController {
  constructor(private readonly catsService: CatsService) {}

  @Get()
  findAll() {
    // Good: Delegate to service
    return this.catsService.findAll();

    // Bad: Business logic in controller
    // const cats = await this.catRepository.find();
    // return cats.filter(cat => cat.active);
  }
}

// 2. Use proper HTTP status codes
@Post()
@HttpCode(HttpStatus.CREATED)  // 201 for resource creation
create(@Body() dto: CreateCatDto) {}

@Delete(':id')
@HttpCode(HttpStatus.NO_CONTENT)  // 204 for successful deletion
remove(@Param('id') id: string) {}

// 3. Use ParseIntPipe for numeric parameters
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {}

// 4. Use ValidationPipe for DTOs
@Post()
create(@Body(ValidationPipe) dto: CreateCatDto) {}

// 5. Group related routes with route prefixes
@Controller('api/v1/cats')  // Versioned API
export class CatsV1Controller {}
```

### Common Pitfalls

```typescript
// Pitfall 1: Using @Res() without passthrough breaks interceptors
// Bad
@Get()
findAll(@Res() res: Response) {
  return this.catsService.findAll();  // Won't work!
}

// Good
@Get()
findAll(@Res({ passthrough: true }) res: Response) {
  res.header('X-Custom', 'value');
  return this.catsService.findAll();  // Works with interceptors
}

// Pitfall 2: Forgetting to handle not found cases
// Bad
@Get(':id')
findOne(@Param('id') id: string) {
  return this.catsService.findOne(id);  // Returns null/undefined
}

// Good
@Get(':id')
async findOne(@Param('id') id: string) {
  const cat = await this.catsService.findOne(id);
  if (!cat) {
    throw new NotFoundException(`Cat #${id} not found`);
  }
  return cat;
}

// Pitfall 3: Not typing route parameters
// Bad - id is string, might cause issues
@Get(':id')
findOne(@Param('id') id: string) {
  return this.repo.findOne(id);  // Might fail if repo expects number
}

// Good - explicit conversion
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {
  return this.repo.findOne(id);
}

// Pitfall 4: Circular dependencies
// Bad - Controller importing another controller
// Good - Use services for shared logic

// Pitfall 5: Not registering controller in module
// Always add to controllers array in @Module()
@Module({
  controllers: [CatsController],  // Don't forget this!
  providers: [CatsService],
})
export class CatsModule {}
```

### API Documentation with Swagger

```typescript
import { ApiTags, ApiOperation, ApiResponse, ApiParam } from '@nestjs/swagger';

@ApiTags('cats')
@Controller('cats')
export class CatsController {
  @Get(':id')
  @ApiOperation({ summary: 'Find a cat by ID' })
  @ApiParam({ name: 'id', type: Number, description: 'Cat ID' })
  @ApiResponse({ status: 200, description: 'Cat found', type: Cat })
  @ApiResponse({ status: 404, description: 'Cat not found' })
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.catsService.findOne(id);
  }
}
```

---

## Quick Reference

```typescript
// Essential imports
import {
  Controller,
  Get, Post, Put, Patch, Delete,
  Param, Query, Body, Headers, Req, Res,
  HttpCode, HttpStatus, Header, Redirect,
  ParseIntPipe, ParseUUIDPipe, ValidationPipe,
  NotFoundException, BadRequestException,
  UseGuards, UseInterceptors,
} from '@nestjs/common';

// Controller template
@Controller('resource')
export class ResourceController {
  constructor(private readonly service: ResourceService) {}

  @Get()                                    // GET /resource
  findAll(@Query() query: QueryDto) {}

  @Get(':id')                               // GET /resource/:id
  findOne(@Param('id', ParseIntPipe) id: number) {}

  @Post()                                   // POST /resource
  @HttpCode(HttpStatus.CREATED)
  create(@Body(ValidationPipe) dto: CreateDto) {}

  @Patch(':id')                             // PATCH /resource/:id
  update(@Param('id') id: string, @Body() dto: UpdateDto) {}

  @Delete(':id')                            // DELETE /resource/:id
  @HttpCode(HttpStatus.NO_CONTENT)
  remove(@Param('id') id: string) {}
}
```
