# GraphQL Code Generator Basics

## Installation

```bash
npm install -D @graphql-codegen/cli
npx graphql-codegen init
```

## Configuration

```typescript
// codegen.ts
import type { CodegenConfig } from '@graphql-codegen/cli';

const config: CodegenConfig = {
  schema: 'http://localhost:4000/graphql',
  documents: ['src/**/*.tsx', 'src/**/*.graphql'],
  generates: {
    './src/gql/': {
      preset: 'client',
    },
  },
};

export default config;
```

## Run

```bash
npx graphql-codegen
npx graphql-codegen --watch
```

## Client Preset (Recommended)

```typescript
const config: CodegenConfig = {
  schema: 'http://localhost:4000/graphql',
  documents: ['src/**/*.tsx'],
  generates: {
    './src/gql/': {
      preset: 'client',
      config: {
        documentMode: 'string',
      },
    },
  },
};
```

## Usage

```typescript
import { gql } from '../gql';
import { useQuery } from '@apollo/client';

const GET_USER = gql(`
  query GetUser($id: ID!) {
    user(id: $id) {
      id
      name
    }
  }
`);

function UserProfile({ id }: { id: string }) {
  const { data } = useQuery(GET_USER, { variables: { id } });
  return <div>{data?.user?.name}</div>;
}
```

## Fragments

```typescript
import { gql, FragmentType, getFragmentData } from '../gql';

const USER_FRAGMENT = gql(`
  fragment UserCard on User {
    id
    name
  }
`);

function UserCard({ user }: { user: FragmentType<typeof USER_FRAGMENT> }) {
  const data = getFragmentData(USER_FRAGMENT, user);
  return <div>{data.name}</div>;
}
```

## Schema Sources

```typescript
// URL
schema: 'http://localhost:4000/graphql'

// Local file
schema: './schema.graphql'

// Multiple sources
schema: ['./schema.graphql', './schema-ext.graphql']

// With headers
schema: {
  'http://localhost:4000/graphql': {
    headers: {
      Authorization: 'Bearer token',
    },
  },
}
```

## Plugins

| Plugin | Description |
|--------|-------------|
| typescript | Base TypeScript types |
| typescript-operations | Operation types |
| typescript-react-query | React Query hooks |
| typescript-urql | urql hooks |
| typed-document-node | TypedDocumentNode |

## Package Scripts

```json
{
  "scripts": {
    "codegen": "graphql-codegen",
    "codegen:watch": "graphql-codegen --watch"
  }
}
```
