# NestJS Pipes

> Official Documentation: https://docs.nestjs.com/pipes

## Overview

Pipes are classes annotated with the `@Injectable()` decorator that implement the `PipeTransform` interface. They operate on arguments processed by a route handler, executing just before the method is invoked. Pipes serve two primary purposes:

1. **Transformation**: Convert input data to a desired format (e.g., string to integer)
2. **Validation**: Evaluate input data and throw exceptions if invalid

NestJS inserts pipes in the arguments processing phase, allowing clean separation of concerns.

---

## PipeTransform Interface

Every pipe must implement the `PipeTransform` interface, which requires a single method:

```typescript
import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class ExamplePipe implements PipeTransform<T, R> {
  transform(value: T, metadata: ArgumentMetadata): R {
    // value: the argument value being processed
    // metadata: information about the argument
    return transformedValue;
  }
}
```

The `ArgumentMetadata` interface provides context about the argument:

```typescript
interface ArgumentMetadata {
  type: 'body' | 'query' | 'param' | 'custom';  // Argument source
  metatype?: Type<unknown>;                       // Type declared in route handler
  data?: string;                                  // String passed to decorator (e.g., 'id' in @Param('id'))
}
```

---

## Built-in Pipes

NestJS provides eight built-in pipes from `@nestjs/common`:

### ParseIntPipe

Converts string to integer. Throws `BadRequestException` if conversion fails.

```typescript
import { ParseIntPipe } from '@nestjs/common';

@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {
  // id is guaranteed to be a number
  return this.usersService.findOne(id);
}

// With custom error handling
@Get(':id')
findOne(
  @Param('id', new ParseIntPipe({
    errorHttpStatusCode: HttpStatus.NOT_ACCEPTABLE,
    exceptionFactory: (error) => new NotAcceptableException('ID must be numeric')
  }))
  id: number
) {
  return this.usersService.findOne(id);
}
```

### ParseBoolPipe

Converts string to boolean. Accepts 'true' and 'false' strings.

```typescript
import { ParseBoolPipe } from '@nestjs/common';

@Get()
findAll(@Query('active', ParseBoolPipe) active: boolean) {
  return this.usersService.findAll({ active });
}

// Optional boolean with default
@Get()
findAll(
  @Query('active', new DefaultValuePipe(true), ParseBoolPipe)
  active: boolean
) {
  return this.usersService.findAll({ active });
}
```

### ParseFloatPipe

Converts string to floating-point number.

```typescript
import { ParseFloatPipe } from '@nestjs/common';

@Get()
findByPrice(@Query('price', ParseFloatPipe) price: number) {
  return this.productsService.findByPrice(price);
}
```

### ParseArrayPipe

Parses and validates array values. Useful for query parameters.

```typescript
import { ParseArrayPipe } from '@nestjs/common';

@Get()
findByIds(
  @Query('ids', new ParseArrayPipe({ items: Number, separator: ',' }))
  ids: number[]
) {
  // GET /items?ids=1,2,3 -> ids = [1, 2, 3]
  return this.itemsService.findByIds(ids);
}

// With DTO validation
@Post('bulk')
createBulk(
  @Body(new ParseArrayPipe({ items: CreateItemDto }))
  items: CreateItemDto[]
) {
  return this.itemsService.createMany(items);
}

// Optional array handling
@Get()
findByTags(
  @Query('tags', new ParseArrayPipe({ items: String, optional: true }))
  tags?: string[]
) {
  return this.itemsService.findByTags(tags ?? []);
}
```

### ParseUUIDPipe

Validates string is a valid UUID.

```typescript
import { ParseUUIDPipe } from '@nestjs/common';

@Get(':uuid')
findByUuid(@Param('uuid', ParseUUIDPipe) uuid: string) {
  return this.usersService.findByUuid(uuid);
}

// Specify UUID version
@Get(':uuid')
findByUuid(
  @Param('uuid', new ParseUUIDPipe({ version: '4' }))
  uuid: string
) {
  return this.usersService.findByUuid(uuid);
}
```

### ParseEnumPipe

Validates value is a member of a specified enum.

```typescript
import { ParseEnumPipe } from '@nestjs/common';

enum UserRole {
  Admin = 'admin',
  User = 'user',
  Guest = 'guest',
}

@Get('role/:role')
findByRole(@Param('role', new ParseEnumPipe(UserRole)) role: UserRole) {
  return this.usersService.findByRole(role);
}
```

### DefaultValuePipe

Provides a default value when the parameter is undefined or null.

```typescript
import { DefaultValuePipe, ParseIntPipe } from '@nestjs/common';

@Get()
findAll(
  @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
  @Query('limit', new DefaultValuePipe(10), ParseIntPipe) limit: number,
  @Query('sort', new DefaultValuePipe('createdAt')) sort: string,
) {
  return this.usersService.findAll({ page, limit, sort });
}
```

### ParseFilePipe

Validates uploaded files with configurable validators.

```typescript
import {
  ParseFilePipe,
  MaxFileSizeValidator,
  FileTypeValidator
} from '@nestjs/common';

@Post('upload')
@UseInterceptors(FileInterceptor('file'))
uploadFile(
  @UploadedFile(
    new ParseFilePipe({
      validators: [
        new MaxFileSizeValidator({ maxSize: 5 * 1024 * 1024 }), // 5MB
        new FileTypeValidator({ fileType: /(jpg|jpeg|png|gif)$/ }),
      ],
    }),
  )
  file: Express.Multer.File,
) {
  return this.filesService.upload(file);
}

// Optional file upload
@Post('upload')
@UseInterceptors(FileInterceptor('file'))
uploadFile(
  @UploadedFile(
    new ParseFilePipe({
      fileIsRequired: false,
      validators: [
        new MaxFileSizeValidator({ maxSize: 5 * 1024 * 1024 }),
      ],
    }),
  )
  file?: Express.Multer.File,
) {
  return file ? this.filesService.upload(file) : null;
}
```

---

## Binding Pipes

Pipes can be bound at different levels:

### Parameter Level

Apply to a specific parameter:

```typescript
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {
  return this.usersService.findOne(id);
}

// Multiple pipes (executed left to right)
@Get(':id')
findOne(
  @Param('id', new DefaultValuePipe(1), ParseIntPipe)
  id: number
) {
  return this.usersService.findOne(id);
}
```

### Method Level

Apply to all parameters of a method:

```typescript
@Post()
@UsePipes(ValidationPipe)
create(@Body() createDto: CreateUserDto) {
  return this.usersService.create(createDto);
}

// Multiple pipes
@Post()
@UsePipes(TrimPipe, ValidationPipe)
create(@Body() createDto: CreateUserDto) {
  return this.usersService.create(createDto);
}
```

### Controller Level

Apply to all methods in a controller:

```typescript
@Controller('users')
@UsePipes(ValidationPipe)
export class UsersController {
  @Post()
  create(@Body() dto: CreateUserDto) {}

  @Put(':id')
  update(@Param('id') id: string, @Body() dto: UpdateUserDto) {}
}
```

### Global Level

Apply to every route handler in the application:

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(3000);
}
```

---

## Global Pipes

### Basic Global Setup

```typescript
// main.ts
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
      transformOptions: {
        enableImplicitConversion: true,
      },
    }),
  );

  await app.listen(3000);
}
```

### Dependency Injection with Global Pipes

Global pipes registered in `main.ts` cannot inject dependencies. Use module registration:

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { APP_PIPE } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';

@Module({
  providers: [
    {
      provide: APP_PIPE,
      useClass: ValidationPipe,
    },
  ],
})
export class AppModule {}

// With factory for configuration
@Module({
  providers: [
    {
      provide: APP_PIPE,
      useFactory: () => new ValidationPipe({
        whitelist: true,
        transform: true,
      }),
    },
  ],
})
export class AppModule {}
```

---

## Custom Pipes

### Transformation Pipe

```typescript
import {
  PipeTransform,
  Injectable,
  ArgumentMetadata,
  BadRequestException
} from '@nestjs/common';

@Injectable()
export class ParseDatePipe implements PipeTransform<string, Date> {
  transform(value: string, metadata: ArgumentMetadata): Date {
    if (!value) {
      throw new BadRequestException('Date is required');
    }

    const date = new Date(value);

    if (isNaN(date.getTime())) {
      throw new BadRequestException(`Invalid date format: ${value}`);
    }

    return date;
  }
}

// Usage
@Get()
findByDate(@Query('date', ParseDatePipe) date: Date) {
  return this.eventsService.findByDate(date);
}
```

### Configurable Transformation Pipe

```typescript
import { PipeTransform, Injectable, BadRequestException } from '@nestjs/common';

interface TrimPipeOptions {
  lowercase?: boolean;
  maxLength?: number;
}

@Injectable()
export class TrimPipe implements PipeTransform<string, string> {
  constructor(private readonly options: TrimPipeOptions = {}) {}

  transform(value: string): string {
    if (typeof value !== 'string') {
      return value;
    }

    let result = value.trim();

    if (this.options.lowercase) {
      result = result.toLowerCase();
    }

    if (this.options.maxLength && result.length > this.options.maxLength) {
      throw new BadRequestException(
        `Value exceeds maximum length of ${this.options.maxLength}`
      );
    }

    return result;
  }
}

// Usage
@Get()
search(@Query('q', new TrimPipe({ lowercase: true, maxLength: 100 })) query: string) {
  return this.searchService.search(query);
}
```

### Validation Pipe

```typescript
import {
  PipeTransform,
  Injectable,
  ArgumentMetadata,
  BadRequestException,
} from '@nestjs/common';

@Injectable()
export class PositiveIntPipe implements PipeTransform<string, number> {
  transform(value: string, metadata: ArgumentMetadata): number {
    const num = parseInt(value, 10);

    if (isNaN(num)) {
      throw new BadRequestException(
        `${metadata.data} must be a number`
      );
    }

    if (num <= 0) {
      throw new BadRequestException(
        `${metadata.data} must be a positive integer`
      );
    }

    return num;
  }
}
```

### Async Validation Pipe

```typescript
import {
  PipeTransform,
  Injectable,
  NotFoundException,
} from '@nestjs/common';
import { UsersService } from './users.service';

@Injectable()
export class UserExistsPipe implements PipeTransform<string, Promise<string>> {
  constructor(private readonly usersService: UsersService) {}

  async transform(value: string): Promise<string> {
    const user = await this.usersService.findOne(value);

    if (!user) {
      throw new NotFoundException(`User with ID ${value} not found`);
    }

    return value;
  }
}

// Usage
@Get(':id')
findOne(@Param('id', ParseUUIDPipe, UserExistsPipe) id: string) {
  return this.usersService.findOne(id);
}
```

---

## Validation with class-validator

### Installation

```bash
npm install class-validator class-transformer
```

### Basic DTO Validation

```typescript
// create-user.dto.ts
import {
  IsEmail,
  IsString,
  MinLength,
  MaxLength,
  IsOptional,
  IsEnum,
  IsNumber,
  Min,
  Max,
} from 'class-validator';

export enum UserRole {
  Admin = 'admin',
  User = 'user',
}

export class CreateUserDto {
  @IsString()
  @MinLength(2)
  @MaxLength(50)
  name: string;

  @IsEmail({}, { message: 'Please provide a valid email address' })
  email: string;

  @IsString()
  @MinLength(8, { message: 'Password must be at least 8 characters' })
  password: string;

  @IsOptional()
  @IsString()
  @MaxLength(500)
  bio?: string;

  @IsOptional()
  @IsEnum(UserRole)
  role?: UserRole;

  @IsOptional()
  @IsNumber()
  @Min(0)
  @Max(150)
  age?: number;
}
```

### Nested Object Validation

```typescript
import { Type } from 'class-transformer';
import { ValidateNested, IsArray, IsString, IsPostalCode } from 'class-validator';

class AddressDto {
  @IsString()
  street: string;

  @IsString()
  city: string;

  @IsPostalCode('US')
  zipCode: string;
}

export class CreateUserDto {
  @IsString()
  name: string;

  @ValidateNested()
  @Type(() => AddressDto)
  address: AddressDto;

  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => AddressDto)
  shippingAddresses: AddressDto[];
}
```

### Conditional Validation

```typescript
import { ValidateIf, IsString, IsNotEmpty } from 'class-validator';

export class CreateAccountDto {
  @IsString()
  type: 'personal' | 'business';

  @ValidateIf((o) => o.type === 'business')
  @IsString()
  @IsNotEmpty()
  companyName?: string;

  @ValidateIf((o) => o.type === 'business')
  @IsString()
  taxId?: string;

  @ValidateIf((o) => o.type === 'personal')
  @IsString()
  firstName?: string;

  @ValidateIf((o) => o.type === 'personal')
  @IsString()
  lastName?: string;
}
```

### Custom Validation Decorator

```typescript
import {
  registerDecorator,
  ValidationOptions,
  ValidatorConstraint,
  ValidatorConstraintInterface,
  ValidationArguments,
} from 'class-validator';
import { Injectable } from '@nestjs/common';
import { UsersService } from './users.service';

@ValidatorConstraint({ name: 'isUniqueEmail', async: true })
@Injectable()
export class IsUniqueEmailConstraint implements ValidatorConstraintInterface {
  constructor(private readonly usersService: UsersService) {}

  async validate(email: string): Promise<boolean> {
    const user = await this.usersService.findByEmail(email);
    return !user;
  }

  defaultMessage(args: ValidationArguments): string {
    return `Email ${args.value} is already registered`;
  }
}

export function IsUniqueEmail(validationOptions?: ValidationOptions) {
  return function (object: object, propertyName: string) {
    registerDecorator({
      target: object.constructor,
      propertyName,
      options: validationOptions,
      constraints: [],
      validator: IsUniqueEmailConstraint,
    });
  };
}

// Usage
export class CreateUserDto {
  @IsEmail()
  @IsUniqueEmail()
  email: string;
}
```

### ValidationPipe Options

```typescript
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,              // Strip properties not in DTO
    forbidNonWhitelisted: true,   // Throw error on extra properties
    transform: true,              // Transform payloads to DTO instances
    transformOptions: {
      enableImplicitConversion: true,  // Auto-convert primitive types
    },
    disableErrorMessages: false,  // Show validation errors (disable in production)
    validateCustomDecorators: true,
    stopAtFirstError: false,      // Collect all errors vs stop at first
    exceptionFactory: (errors) => {
      // Custom error formatting
      const messages = errors.map(err => ({
        field: err.property,
        errors: Object.values(err.constraints || {}),
      }));
      return new BadRequestException({ errors: messages });
    },
  }),
);
```

---

## Transformation Use Cases

### Using class-transformer

```typescript
import { Transform, Exclude, Expose, Type } from 'class-transformer';
import { IsString, IsNumber, IsDate } from 'class-validator';

export class QueryDto {
  @Transform(({ value }) => value?.toLowerCase().trim())
  @IsString()
  search: string;

  @Transform(({ value }) => parseInt(value, 10))
  @IsNumber()
  page: number;

  @Transform(({ value }) => value === 'true')
  active: boolean;

  @Type(() => Date)
  @IsDate()
  startDate: Date;
}

export class UserResponseDto {
  @Expose()
  id: string;

  @Expose()
  name: string;

  @Expose()
  email: string;

  @Exclude()
  password: string;

  @Expose()
  @Transform(({ value }) => value?.toISOString())
  createdAt: Date;
}
```

### Transform Pipe for Sanitization

```typescript
import { PipeTransform, Injectable } from '@nestjs/common';
import * as sanitizeHtml from 'sanitize-html';

@Injectable()
export class SanitizeHtmlPipe implements PipeTransform {
  transform(value: any): any {
    if (typeof value === 'string') {
      return sanitizeHtml(value, {
        allowedTags: ['b', 'i', 'em', 'strong', 'a', 'p', 'br'],
        allowedAttributes: { a: ['href'] },
      });
    }

    if (typeof value === 'object' && value !== null) {
      return this.sanitizeObject(value);
    }

    return value;
  }

  private sanitizeObject(obj: Record<string, any>): Record<string, any> {
    const sanitized: Record<string, any> = {};

    for (const [key, val] of Object.entries(obj)) {
      sanitized[key] = this.transform(val);
    }

    return sanitized;
  }
}
```

---

## Schema Validation

### Zod Validation

```bash
npm install zod
```

```typescript
import { PipeTransform, BadRequestException, Injectable } from '@nestjs/common';
import { ZodSchema, ZodError } from 'zod';

@Injectable()
export class ZodValidationPipe implements PipeTransform {
  constructor(private schema: ZodSchema) {}

  transform(value: unknown) {
    try {
      return this.schema.parse(value);
    } catch (error) {
      if (error instanceof ZodError) {
        throw new BadRequestException({
          message: 'Validation failed',
          errors: error.errors.map(e => ({
            path: e.path.join('.'),
            message: e.message,
          })),
        });
      }
      throw error;
    }
  }
}

// Define schema
import { z } from 'zod';

export const createUserSchema = z.object({
  name: z.string().min(2).max(50),
  email: z.string().email(),
  password: z.string().min(8),
  age: z.number().positive().optional(),
});

export type CreateUserDto = z.infer<typeof createUserSchema>;

// Usage
@Post()
@UsePipes(new ZodValidationPipe(createUserSchema))
create(@Body() createUserDto: CreateUserDto) {
  return this.usersService.create(createUserDto);
}
```

### Joi Validation

```bash
npm install joi
```

```typescript
import { PipeTransform, Injectable, BadRequestException } from '@nestjs/common';
import { ObjectSchema } from 'joi';

@Injectable()
export class JoiValidationPipe implements PipeTransform {
  constructor(private schema: ObjectSchema) {}

  transform(value: any) {
    const { error, value: validatedValue } = this.schema.validate(value, {
      abortEarly: false,
      stripUnknown: true,
    });

    if (error) {
      throw new BadRequestException({
        message: 'Validation failed',
        errors: error.details.map(detail => ({
          path: detail.path.join('.'),
          message: detail.message,
        })),
      });
    }

    return validatedValue;
  }
}

// Define schema
import * as Joi from 'joi';

export const createUserSchema = Joi.object({
  name: Joi.string().min(2).max(50).required(),
  email: Joi.string().email().required(),
  password: Joi.string().min(8).required(),
  age: Joi.number().positive().optional(),
});

// Usage
@Post()
@UsePipes(new JoiValidationPipe(createUserSchema))
create(@Body() createUserDto: any) {
  return this.usersService.create(createUserDto);
}
```

---

## Best Practices

### 1. Use Global ValidationPipe

```typescript
// main.ts - recommended configuration
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
    transformOptions: {
      enableImplicitConversion: true,
    },
  }),
);
```

### 2. Create Reusable DTOs

```typescript
// Extend base DTOs for consistency
export class BaseDto {
  @IsOptional()
  @IsUUID()
  id?: string;
}

export class CreateUserDto extends BaseDto {
  @IsString()
  name: string;
}

// Use PartialType for update DTOs
import { PartialType } from '@nestjs/mapped-types';

export class UpdateUserDto extends PartialType(CreateUserDto) {}
```

### 3. Chain Pipes Appropriately

```typescript
// Order matters: default -> parse -> validate
@Get(':id')
findOne(
  @Param('id', new DefaultValuePipe('1'), ParseIntPipe)
  id: number
) {}
```

### 4. Custom Error Messages

```typescript
export class CreateUserDto {
  @IsEmail({}, { message: 'Invalid email format' })
  @IsNotEmpty({ message: 'Email is required' })
  email: string;

  @MinLength(8, { message: 'Password must be at least 8 characters' })
  password: string;
}
```

### 5. Separate Transformation and Validation Logic

```typescript
// Transformation pipe
@Injectable()
export class NormalizeEmailPipe implements PipeTransform {
  transform(value: any) {
    if (value?.email) {
      value.email = value.email.toLowerCase().trim();
    }
    return value;
  }
}

// Apply before validation
@Post()
@UsePipes(NormalizeEmailPipe, ValidationPipe)
create(@Body() dto: CreateUserDto) {}
```

---

## Common Pitfalls

### 1. Forgetting transform: true

```typescript
// Without transform, DTOs remain plain objects
// WRONG: dto instanceof CreateUserDto === false
app.useGlobalPipes(new ValidationPipe());

// CORRECT: dto instanceof CreateUserDto === true
app.useGlobalPipes(new ValidationPipe({ transform: true }));
```

### 2. Missing @Type() for Nested Objects

```typescript
// WRONG: nested validation won't work
class CreateUserDto {
  @ValidateNested()
  address: AddressDto;
}

// CORRECT: @Type() enables proper instantiation
class CreateUserDto {
  @ValidateNested()
  @Type(() => AddressDto)
  address: AddressDto;
}
```

### 3. Async Validators Not Working

```typescript
// Ensure constraint class is injectable and registered
@ValidatorConstraint({ async: true })
@Injectable()
export class IsUniqueConstraint implements ValidatorConstraintInterface {
  constructor(private service: MyService) {} // Requires DI
}

// Register in module providers
@Module({
  providers: [IsUniqueConstraint],
})
export class AppModule {}
```

### 4. Pipe Order Confusion

```typescript
// Pipes execute left-to-right in decorators
@Param('id', PipeA, PipeB, PipeC) // PipeA -> PipeB -> PipeC

// For @UsePipes, execution is also left-to-right
@UsePipes(PipeA, PipeB) // PipeA runs first
```

### 5. Global Pipe Not Injecting Dependencies

```typescript
// WRONG: Cannot inject dependencies
// main.ts
app.useGlobalPipes(new CustomPipeWithDeps()); // deps are undefined

// CORRECT: Use module-based registration
@Module({
  providers: [
    {
      provide: APP_PIPE,
      useClass: CustomPipeWithDeps,
    },
  ],
})
export class AppModule {}
```

### 6. Not Handling Optional Parameters

```typescript
// Handle undefined/null values in custom pipes
@Injectable()
export class ParseOptionalIntPipe implements PipeTransform {
  transform(value: string | undefined): number | undefined {
    if (value === undefined || value === null || value === '') {
      return undefined;
    }

    const num = parseInt(value, 10);
    if (isNaN(num)) {
      throw new BadRequestException('Invalid number');
    }
    return num;
  }
}
```
