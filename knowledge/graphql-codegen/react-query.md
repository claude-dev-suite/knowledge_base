# GraphQL Codegen React Query

## Installation

```bash
npm install -D @graphql-codegen/typescript @graphql-codegen/typescript-operations \
  @graphql-codegen/typescript-react-query
npm install @tanstack/react-query graphql-request
```

## Configuration

```typescript
// codegen.ts
const config: CodegenConfig = {
  schema: 'http://localhost:4000/graphql',
  documents: ['src/**/*.graphql'],
  generates: {
    './src/gql/index.ts': {
      plugins: [
        'typescript',
        'typescript-operations',
        'typescript-react-query',
      ],
      config: {
        fetcher: './fetcher#fetcher',
        reactQueryVersion: 5,
        exposeQueryKeys: true,
        addInfiniteQuery: true,
      },
    },
  },
};
```

## Fetcher

```typescript
// src/gql/fetcher.ts
import { GraphQLClient } from 'graphql-request';

const client = new GraphQLClient('http://localhost:4000/graphql');

export const fetcher = <TData, TVariables>(
  query: string,
  variables?: TVariables
): (() => Promise<TData>) => {
  return async () => {
    const token = localStorage.getItem('token');
    return client.request<TData>(query, variables, {
      Authorization: token ? `Bearer ${token}` : '',
    });
  };
};
```

## Generated Hooks

```typescript
import { useGetUserQuery, useCreateUserMutation } from './gql';

// Query
const { data, isLoading } = useGetUserQuery({ id: '123' });

// Mutation
const { mutate } = useCreateUserMutation();
mutate({ input: { name: 'John', email: 'john@example.com' } });

// With options
const { data } = useGetUserQuery({ id }, {
  staleTime: 5 * 60 * 1000,
  enabled: !!id,
});

// Infinite Query
const { data, fetchNextPage } = useInfiniteGetUsersQuery({ limit: 10 }, {
  getNextPageParam: (last, pages) => ({ offset: pages.length * 10 }),
});
```

## Query Keys

```typescript
// Get query key
const key = useGetUserQuery.getKey({ id: '123' });

// Prefetch
queryClient.prefetchQuery({
  queryKey: useGetUserQuery.getKey({ id: '123' }),
  queryFn: useGetUserQuery.fetcher({ id: '123' }),
});

// Invalidate
queryClient.invalidateQueries({ queryKey: ['GetUsers'] });
```

## Config Options

```typescript
{
  config: {
    fetcher: './fetcher#fetcher',
    reactQueryVersion: 5,
    exposeQueryKeys: true,
    exposeFetcher: true,
    addInfiniteQuery: true,
    addSuspenseQuery: true,
  },
}
```
