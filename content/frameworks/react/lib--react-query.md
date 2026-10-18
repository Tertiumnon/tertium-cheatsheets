# React Query (TanStack Query)

React Query is a powerful data-fetching and state management library for React. It simplifies managing server state, caching, synchronization, and mutations with minimal configuration.

## Key Features

- **Automatic Caching:** Smart caching and deduplication
- **Background Refetching:** Keep data fresh automatically
- **Request Deduplication:** Combine multiple identical requests
- **Pagination & Lazy Loading:** Built-in patterns
- **Optimistic Updates:** Update UI before server response
- **Mutations:** Handle create/update/delete operations
- **Infinite Queries:** Infinite scroll and pagination
- **DevTools:** Debug queries with built-in devtools

## Installation

```bash
# Using npm
npm install @tanstack/react-query

# Using yarn
yarn add @tanstack/react-query

# For DevTools
npm install @tanstack/react-query-devtools
```

## Basic Setup

### QueryClient Configuration

```typescript
// lib/queryClient.ts
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5, // 5 minutes
      gcTime: 1000 * 60 * 10, // 10 minutes (cache time)
      retry: 1,
      refetchOnWindowFocus: false,
    },
    mutations: {
      retry: 1,
    },
  },
});
```

### Provider Setup

```typescript
// app.tsx
import { QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import { queryClient } from './lib/queryClient';

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <YourApp />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}

export default App;
```

## Basic Query

### useQuery Hook

Fetch data with automatic caching and refetching:

```typescript
// hooks/useUsers.ts
import { useQuery } from '@tanstack/react-query';
import { apiClient } from '@/lib/api';

interface User {
  id: string;
  name: string;
  email: string;
}

export function useUsers() {
  return useQuery({
    queryKey: ['users'],
    queryFn: async () => {
      const response = await apiClient.get<User[]>('/users');
      return response.data;
    },
    staleTime: 1000 * 60 * 5, // Custom stale time
  });
}

// Usage in component
function UsersList() {
  const { data: users, isLoading, error } = useUsers();

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <ul>
      {users?.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

### Query with Parameters

```typescript
// hooks/useUser.ts
import { useQuery } from '@tanstack/react-query';

interface User {
  id: string;
  name: string;
  email: string;
}

export function useUser(id: string | null) {
  return useQuery({
    queryKey: ['users', id], // Include params in key
    queryFn: async () => {
      if (!id) return null;
      const response = await fetch(`/api/users/${id}`);
      return response.json();
    },
    enabled: !!id, // Only fetch if id exists
  });
}

// Usage
function UserDetail({ userId }: { userId: string }) {
  const { data: user } = useUser(userId);
  return <div>{user?.name}</div>;
}
```

### Dependent Queries

```typescript
// Chain queries that depend on each other
export function useUserWithPosts(userId: string | null) {
  const userQuery = useQuery({
    queryKey: ['users', userId],
    queryFn: () => fetchUser(userId!),
    enabled: !!userId,
  });

  const postsQuery = useQuery({
    queryKey: ['users', userId, 'posts'],
    queryFn: () => fetchUserPosts(userId!),
    enabled: !!userQuery.data, // Only fetch if user data is available
  });

  return { userQuery, postsQuery };
}
```

## Mutations

### Basic Mutation

Create, update, or delete data:

```typescript
// hooks/useCreateUser.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { apiClient } from '@/lib/api';

interface CreateUserInput {
  name: string;
  email: string;
}

export function useCreateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (data: CreateUserInput) => {
      const response = await apiClient.post('/users', data);
      return response.data;
    },
    onSuccess: (newUser) => {
      // Invalidate users list to refetch
      queryClient.invalidateQueries({ queryKey: ['users'] });
      
      // Or manually update cache
      queryClient.setQueryData(['users'], (old: User[]) => [
        ...old,
        newUser,
      ]);
    },
    onError: (error) => {
      console.error('Failed to create user:', error);
    },
  });
}

// Usage in component
function CreateUserForm() {
  const { mutate: createUser, isPending, error } = useCreateUser();

  const handleSubmit = (formData: CreateUserInput) => {
    createUser(formData);
  };

  return (
    <form onSubmit={(e) => {
      e.preventDefault();
      handleSubmit({ name: 'John', email: 'john@example.com' });
    }}>
      {error && <div className="error">{error.message}</div>}
      <button disabled={isPending}>
        {isPending ? 'Creating...' : 'Create User'}
      </button>
    </form>
  );
}
```

### Mutation with Optimistic Update

Update UI immediately, revert on error:

```typescript
export function useUpdateUser(userId: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (updatedData: Partial<User>) =>
      apiClient.put(`/users/${userId}`, updatedData),
    
    onMutate: async (updatedData) => {
      // Cancel any outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['users', userId] });

      // Snapshot the previous data
      const previousData = queryClient.getQueryData(['users', userId]);

      // Optimistically update the cache
      queryClient.setQueryData(['users', userId], (old: User) => ({
        ...old,
        ...updatedData,
      }));

      return { previousData }; // Return context for rollback
    },
    
    onError: (error, variables, context) => {
      // Rollback on error
      if (context?.previousData) {
        queryClient.setQueryData(['users', userId], context.previousData);
      }
    },
    
    onSuccess: () => {
      // Refetch to ensure sync
      queryClient.invalidateQueries({ queryKey: ['users', userId] });
    },
  });
}
```

## Advanced Patterns

### Infinite Queries (Pagination)

```typescript
import { useInfiniteQuery } from '@tanstack/react-query';

interface UsersResponse {
  data: User[];
  nextCursor?: string;
}

export function useInfiniteUsers() {
  return useInfiniteQuery({
    queryKey: ['users'],
    queryFn: async ({ pageParam }) => {
      const response = await fetch(
        `/api/users?cursor=${pageParam || ''}`
      );
      return response.json() as UsersResponse;
    },
    initialPageParam: null,
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  });
}

// Usage
function InfiniteUsersList() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteUsers();

  return (
    <>
      {data?.pages.map((page, i) => (
        <div key={i}>
          {page.data.map(user => (
            <UserCard key={user.id} user={user} />
          ))}
        </div>
      ))}
      <button
        onClick={() => fetchNextPage()}
        disabled={!hasNextPage || isFetchingNextPage}
      >
        {isFetchingNextPage ? 'Loading...' : 'Load More'}
      </button>
    </>
  );
}
```

### Prefetching Data

```typescript
import { useQueryClient } from '@tanstack/react-query';

function UsersList() {
  const queryClient = useQueryClient();

  const handleHover = (userId: string) => {
    // Prefetch user detail when hovering
    queryClient.prefetchQuery({
      queryKey: ['users', userId],
      queryFn: () => fetchUser(userId),
    });
  };

  return (
    <div>
      {users?.map(user => (
        <div
          key={user.id}
          onMouseEnter={() => handleHover(user.id)}
        >
          {user.name}
        </div>
      ))}
    </div>
  );
}
```

### Background Refetching

```typescript
export function useUsersWithRefresh() {
  return useQuery({
    queryKey: ['users'],
    queryFn: fetchUsers,
    staleTime: 1000 * 60, // Stale after 1 minute
    refetchInterval: 1000 * 60 * 5, // Refetch every 5 minutes
    refetchOnWindowFocus: true, // Refetch when window regains focus
    refetchOnMount: true, // Refetch when component mounts
  });
}
```

### Combining Queries (useQueries)

```typescript
import { useQueries } from '@tanstack/react-query';

interface UserIds {
  ids: string[];
}

export function useMultipleUsers({ ids }: UserIds) {
  return useQueries({
    queries: ids.map(id => ({
      queryKey: ['users', id],
      queryFn: () => fetchUser(id),
    })),
  });
}

// Usage
function UserProfiles({ userIds }: { userIds: string[] }) {
  const queries = useMultipleUsers({ ids: userIds });

  if (queries.some(q => q.isLoading)) return <div>Loading...</div>;

  return (
    <div>
      {queries.map((query, idx) => (
        <UserCard key={idx} user={query.data} />
      ))}
    </div>
  );
}
```

## Query Invalidation

Manually trigger refetch of cached data:

```typescript
import { useQueryClient } from '@tanstack/react-query';

function UserActions() {
  const queryClient = useQueryClient();

  const handleDeleteUser = async (userId: string) => {
    await deleteUserApi(userId);

    // Invalidate specific query
    queryClient.invalidateQueries({
      queryKey: ['users', userId],
    });

    // Invalidate entire users list
    queryClient.invalidateQueries({
      queryKey: ['users'],
    });

    // Invalidate with predicate
    queryClient.invalidateQueries({
      predicate: (query) =>
        query.queryKey[0] === 'users',
    });
  };

  return <button onClick={() => handleDeleteUser('123')}>Delete</button>;
}
```

## Error Handling

```typescript
function useUserWithErrorHandling(userId: string) {
  return useQuery({
    queryKey: ['users', userId],
    queryFn: async () => {
      const response = await fetch(`/api/users/${userId}`);
      
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`);
      }
      
      return response.json();
    },
    retry: (failureCount, error) => {
      // Don't retry on 404
      if (error instanceof Error && error.message.includes('404')) {
        return false;
      }
      // Retry up to 3 times
      return failureCount < 3;
    },
  });
}
```

## Query Client Methods

### Common Operations

```typescript
const queryClient = useQueryClient();

// Get cached data
const cachedData = queryClient.getQueryData(['users', userId]);

// Set cached data
queryClient.setQueryData(['users', userId], newData);

// Invalidate and refetch
queryClient.invalidateQueries({ queryKey: ['users'] });

// Remove query from cache
queryClient.removeQueries({ queryKey: ['users'] });

// Fetch query
const data = await queryClient.fetchQuery({
  queryKey: ['users'],
  queryFn: fetchUsers,
});

// Prefetch query
await queryClient.prefetchQuery({
  queryKey: ['users'],
  queryFn: fetchUsers,
});
```

## Custom Hooks Pattern

Create reusable query hooks:

```typescript
// hooks/api/users.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { apiClient } from '@/lib/api';

export function useUsers(options?: { enabled?: boolean }) {
  return useQuery({
    queryKey: ['users'],
    queryFn: () => apiClient.get('/users'),
    enabled: options?.enabled,
  });
}

export function useUser(id: string) {
  return useQuery({
    queryKey: ['users', id],
    queryFn: () => apiClient.get(`/users/${id}`),
  });
}

export function useCreateUser() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (data: any) => apiClient.post('/users', data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });
}

export function useUpdateUser(id: string) {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (data: any) => apiClient.put(`/users/${id}`, data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users', id] });
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });
}

export function useDeleteUser(id: string) {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: () => apiClient.delete(`/users/${id}`),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });
}
```

## DevTools

```typescript
// app.tsx
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <YourApp />
      {/* DevTools panel - drag to resize */}
      <ReactQueryDevtools
        initialIsOpen={false}
        buttonPosition="bottom-right"
      />
    </QueryClientProvider>
  );
}
```

## Best Practices

- **Query Keys:** Use array format for nested keys: `['users', userId, 'posts']`
- **Stale Time:** Set appropriate stale time for your data
- **Cache Time:** Keep data longer than stale time for background refetch
- **Error Handling:** Implement proper error handling in components
- **Prefetching:** Prefetch data on hover or route navigation
- **Optimistic Updates:** Update UI before server response for better UX
- **Query Invalidation:** Invalidate related queries on mutations
- **Custom Hooks:** Create hooks for API calls to reuse across components
- **DevTools:** Use DevTools for debugging in development
- **Retry Logic:** Configure retry behavior based on error type

## Common Query Key Patterns

```typescript
// Users
['users'] // All users
['users', userId] // Specific user
['users', userId, 'posts'] // User's posts
['users', { filters, pagination }] // With parameters

// Posts
['posts'] // All posts
['posts', postId] // Specific post
['posts', { status: 'published' }] // Filtered posts

// Infinite queries
['posts', 'infinite'] // Infinite posts
['posts', 'infinite', { category: 'tech' }] // Infinite with filters
```

## References & Sources

### Official Documentation
- React Query (TanStack Query): https://tanstack.com/query/latest
- React Query Documentation: https://tanstack.com/query/latest/docs/react/overview
- DevTools: https://tanstack.com/query/latest/docs/react/devtools

### Guides & Tutorials
- Query Key Factory: https://tkdodo.eu/blog/effective-react-query-keys
- React Query Course: https://tkdodo.eu/blog
- Important Defaults: https://tkdodo.eu/blog/react-query-as-a-state-manager

### Related Libraries
- **SWR:** Alternative data fetching library
- **Axios:** HTTP client for API calls
- **Zustand:** State management (complement to React Query)
- **Zod:** Schema validation

### Key Concepts
- Server State vs Client State
- Cache Invalidation
- Optimistic Updates
- Request Deduplication
- Stale While Revalidate (SWR) Pattern

### Note
React Query (now TanStack Query) is designed for managing server state. Use it alongside client state management (Context, Redux) for complete state management.
