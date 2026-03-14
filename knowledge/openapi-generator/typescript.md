# OpenAPI Generator TypeScript

## TypeScript Fetch

```bash
npx @openapitools/openapi-generator-cli generate \
  -i openapi.yaml \
  -g typescript-fetch \
  -o ./src/api-client \
  --additional-properties=\
supportsES6=true,\
typescriptThreePlus=true,\
withInterfaces=true,\
useSingleRequestParameter=true
```

### Additional Properties

| Property | Default | Description |
|----------|---------|-------------|
| supportsES6 | false | Use ES6+ features |
| typescriptThreePlus | false | TypeScript 3+ features |
| withInterfaces | false | Generate interfaces |
| useSingleRequestParameter | false | Wrap params in object |
| prefixParameterInterfaces | false | Prefix param interfaces |
| npmName | - | Package name |
| npmVersion | 1.0.0 | Package version |

### Usage

```typescript
import { Configuration, UsersApi } from './api-client';

const config = new Configuration({
  basePath: 'https://api.example.com',
  accessToken: async () => getToken(),
  headers: { 'X-Custom': 'value' },
});

const api = new UsersApi(config);
const users = await api.listUsers({ status: 'active' });
```

## TypeScript Axios

```bash
npx @openapitools/openapi-generator-cli generate \
  -i openapi.yaml \
  -g typescript-axios \
  -o ./src/api-client \
  --additional-properties=\
supportsES6=true,\
withSeparateModelsAndApi=true,\
withInterfaces=true
```

### Additional Properties

| Property | Default | Description |
|----------|---------|-------------|
| withSeparateModelsAndApi | false | Separate models/api dirs |
| apiPackage | api | API directory name |
| modelPackage | model | Models directory name |
| withInterfaces | false | Generate interfaces |
| withNodeImports | false | Node.js imports |

### Usage

```typescript
import { Configuration, UsersApi } from './api-client';
import axios from 'axios';

const axiosInstance = axios.create();
axiosInstance.interceptors.request.use((config) => {
  config.headers.Authorization = `Bearer ${getToken()}`;
  return config;
});

const config = new Configuration({
  basePath: 'https://api.example.com',
});

const api = new UsersApi(config, undefined, axiosInstance);
const { data: users } = await api.listUsers('active');
```

## Configuration Options

```typescript
import { Configuration } from './api-client';

const config = new Configuration({
  basePath: 'https://api.example.com',

  // Bearer token (string or function)
  accessToken: 'my-token',
  accessToken: () => localStorage.getItem('token'),
  accessToken: async () => await refreshAndGetToken(),

  // API key
  apiKey: 'my-api-key',
  apiKey: (name: string) => getApiKey(name),

  // Basic auth
  username: 'user',
  password: 'pass',

  // Custom headers
  headers: {
    'X-Custom-Header': 'value',
  },

  // Middleware
  middleware: [
    {
      pre: async (context) => context,
      post: async (context) => context.response,
    },
  ],
});
```

## Error Handling

### typescript-fetch

```typescript
import { ResponseError } from './api-client';

try {
  await api.getUserById({ id: '123' });
} catch (error) {
  if (error instanceof ResponseError) {
    console.log(error.response.status);
    const body = await error.response.json();
    console.log(body.message);
  }
}
```

### typescript-axios

```typescript
import { AxiosError } from 'axios';

try {
  await api.getUserById('123');
} catch (error) {
  if (error instanceof AxiosError) {
    console.log(error.response?.status);
    console.log(error.response?.data);
  }
}
```

## Custom Templates

```bash
# Extract templates
npx @openapitools/openapi-generator-cli author template \
  -g typescript-fetch \
  -o ./templates

# Use custom templates
npx @openapitools/openapi-generator-cli generate \
  -i openapi.yaml \
  -g typescript-fetch \
  -o ./src/api-client \
  -t ./templates
```

## Selective Generation

```bash
# Only models
npx @openapitools/openapi-generator-cli generate \
  -i openapi.yaml \
  -g typescript-fetch \
  -o ./src/api-client \
  --global-property=models

# Only APIs
npx @openapitools/openapi-generator-cli generate \
  -i openapi.yaml \
  -g typescript-fetch \
  -o ./src/api-client \
  --global-property=apis

# Both without supporting files
npx @openapitools/openapi-generator-cli generate \
  -i openapi.yaml \
  -g typescript-fetch \
  -o ./src/api-client \
  --global-property=models,apis
```

## Model Customization

```bash
# Add suffix
--model-name-suffix=Dto

# Add prefix
--model-name-prefix=Api

# Map types
--type-mappings=DateTime=Date
```

## Configuration File

```json
{
  "generator-cli": {
    "version": "7.0.0",
    "generators": {
      "client": {
        "generatorName": "typescript-fetch",
        "output": "./src/api-client",
        "inputSpec": "./openapi.yaml",
        "additionalProperties": {
          "supportsES6": true,
          "typescriptThreePlus": true,
          "withInterfaces": true,
          "useSingleRequestParameter": true,
          "npmName": "@myorg/api-client",
          "npmVersion": "1.0.0"
        },
        "typeMappings": {
          "DateTime": "Date"
        }
      }
    }
  }
}
```
