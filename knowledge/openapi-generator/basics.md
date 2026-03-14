# OpenAPI Generator Basics

openapi-generator-cli generates complete API clients from OpenAPI specifications.

## Installation

```bash
npm install -D @openapitools/openapi-generator-cli
```

## Basic Usage

```bash
# Generate TypeScript Fetch client
npx @openapitools/openapi-generator-cli generate \
  -i openapi.yaml \
  -g typescript-fetch \
  -o ./src/api-client

# Generate TypeScript Axios client
npx @openapitools/openapi-generator-cli generate \
  -i openapi.yaml \
  -g typescript-axios \
  -o ./src/api-client
```

## Available Generators

| Generator | Description |
|-----------|-------------|
| typescript-fetch | Native fetch API |
| typescript-axios | Axios HTTP client |
| typescript-node | Node.js HTTP client |
| typescript-angular | Angular HttpClient |
| java | Java client |
| kotlin | Kotlin client |
| python | Python client |
| go | Go client |

List all generators:

```bash
npx @openapitools/openapi-generator-cli list
```

## Configuration File

```json
// openapitools.json
{
  "$schema": "https://raw.githubusercontent.com/OpenAPITools/openapi-generator-cli/master/apps/generator-cli/src/config.schema.json",
  "spaces": 2,
  "generator-cli": {
    "version": "7.0.0",
    "generators": {
      "client": {
        "generatorName": "typescript-fetch",
        "output": "#{cwd}/src/api-client",
        "inputSpec": "#{cwd}/openapi.yaml",
        "additionalProperties": {
          "supportsES6": true,
          "typescriptThreePlus": true,
          "withInterfaces": true
        }
      }
    }
  }
}
```

Run with config:

```bash
npx @openapitools/openapi-generator-cli generate
```

## Common Options

```bash
-i, --input-spec      # Input OpenAPI spec (file or URL)
-g, --generator-name  # Generator to use
-o, --output          # Output directory
-c, --config          # Configuration file
-t, --template-dir    # Custom templates directory
--additional-properties  # Generator-specific options
--global-property     # Global properties
```

## Additional Properties

### typescript-fetch

```bash
--additional-properties=\
supportsES6=true,\
typescriptThreePlus=true,\
withInterfaces=true,\
useSingleRequestParameter=true
```

### typescript-axios

```bash
--additional-properties=\
supportsES6=true,\
withSeparateModelsAndApi=true,\
withInterfaces=true
```

## Generated Structure

```
src/api-client/
├── apis/
│   ├── UsersApi.ts
│   └── PostsApi.ts
├── models/
│   ├── User.ts
│   └── CreateUserDto.ts
├── runtime.ts
└── index.ts
```

## Client Usage

```typescript
import { Configuration, UsersApi } from './api-client';

const config = new Configuration({
  basePath: 'https://api.example.com',
  accessToken: () => localStorage.getItem('token') || '',
});

const usersApi = new UsersApi(config);

// List
const users = await usersApi.listUsers({ status: 'active' });

// Get
const user = await usersApi.getUserById({ id: '123' });

// Create
const newUser = await usersApi.createUser({
  createUserDto: { name: 'John', email: 'john@example.com' },
});

// Update
await usersApi.updateUser({
  id: '123',
  updateUserDto: { name: 'Updated' },
});

// Delete
await usersApi.deleteUser({ id: '123' });
```

## Validate Spec

```bash
npx @openapitools/openapi-generator-cli validate -i openapi.yaml
```

## Package Script

```json
{
  "scripts": {
    "generate:client": "openapi-generator-cli generate"
  }
}
```
