# TanStack Query Context

Comprehensive guide to TanStack Query (React Query) for data fetching and caching.

## Setup

```bash
npm install @tanstack/react-query
npm install -D @tanstack/eslint-plugin-query
```

## Provider Setup

```tsx
// app/providers.tsx
'use client'

import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'
import { useState } from 'react'

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 60 * 1000, // 1 minute
            gcTime: 5 * 60 * 1000, // 5 minutes (formerly cacheTime)
            retry: 1,
            refetchOnWindowFocus: false,
          },
        },
      })
  )

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  )
}
```

## Basic Queries

### Simple Query
```tsx
import { useQuery } from '@tanstack/react-query'

function Users() {
  const { data, isLoading, error, isError, refetch } = useQuery({
    queryKey: ['users'],
    queryFn: async () => {
      const res = await fetch('/api/users')
      if (!res.ok) throw new Error('Failed to fetch')
      return res.json()
    },
  })

  if (isLoading) return <div>Loading...</div>
  if (isError) return <div>Error: {error.message}</div>

  return (
    <ul>
      {data.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  )
}
```

### Query with Parameters
```tsx
function User({ userId }: { userId: string }) {
  const { data, isLoading } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
    enabled: !!userId, // Only run when userId exists
  })

  // ...
}
```

### Dependent Queries
```tsx
function UserPosts({ userId }: { userId: string }) {
  const userQuery = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  })

  const postsQuery = useQuery({
    queryKey: ['posts', userId],
    queryFn: () => fetchUserPosts(userId),
    enabled: !!userQuery.data, // Only run after user is fetched
  })

  // ...
}
```

### Parallel Queries
```tsx
import { useQueries } from '@tanstack/react-query'

function UserData({ userIds }: { userIds: string[] }) {
  const userQueries = useQueries({
    queries: userIds.map((id) => ({
      queryKey: ['user', id],
      queryFn: () => fetchUser(id),
    })),
  })

  const allLoaded = userQueries.every((q) => q.isSuccess)
  const users = userQueries.map((q) => q.data)

  // ...
}
```

## Mutations

### Basic Mutation
```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query'

function CreateUser() {
  const queryClient = useQueryClient()

  const mutation = useMutation({
    mutationFn: (newUser: CreateUserInput) => {
      return fetch('/api/users', {
        method: 'POST',
        body: JSON.stringify(newUser),
      }).then((res) => res.json())
    },
    onSuccess: (data) => {
      // Invalidate and refetch
      queryClient.invalidateQueries({ queryKey: ['users'] })
    },
    onError: (error) => {
      console.error('Failed to create user:', error)
    },
  })

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault()
        mutation.mutate({ name: 'John', email: 'john@example.com' })
      }}
    >
      <button type="submit" disabled={mutation.isPending}>
        {mutation.isPending ? 'Creating...' : 'Create User'}
      </button>
    </form>
  )
}
```

### Optimistic Updates
```tsx
const mutation = useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    // Cancel outgoing refetches
    await queryClient.cancelQueries({ queryKey: ['todos'] })

    // Snapshot previous value
    const previousTodos = queryClient.getQueryData(['todos'])

    // Optimistically update
    queryClient.setQueryData(['todos'], (old) =>
      old.map((todo) =>
        todo.id === newTodo.id ? { ...todo, ...newTodo } : todo
      )
    )

    // Return context with snapshot
    return { previousTodos }
  },
  onError: (err, newTodo, context) => {
    // Rollback on error
    queryClient.setQueryData(['todos'], context.previousTodos)
  },
  onSettled: () => {
    // Always refetch after mutation
    queryClient.invalidateQueries({ queryKey: ['todos'] })
  },
})
```

## Query Keys

### Key Factory Pattern
```typescript
// lib/queries/keys.ts
export const userKeys = {
  all: ['users'] as const,
  lists: () => [...userKeys.all, 'list'] as const,
  list: (filters: UserFilters) => [...userKeys.lists(), filters] as const,
  details: () => [...userKeys.all, 'detail'] as const,
  detail: (id: string) => [...userKeys.details(), id] as const,
}

export const postKeys = {
  all: ['posts'] as const,
  lists: () => [...postKeys.all, 'list'] as const,
  list: (filters: PostFilters) => [...postKeys.lists(), filters] as const,
  details: () => [...postKeys.all, 'detail'] as const,
  detail: (id: string) => [...postKeys.details(), id] as const,
  byUser: (userId: string) => [...postKeys.all, 'user', userId] as const,
}
```

### Using Key Factory
```tsx
// Queries
useQuery({
  queryKey: userKeys.detail(userId),
  queryFn: () => fetchUser(userId),
})

// Invalidation
queryClient.invalidateQueries({ queryKey: userKeys.all })
queryClient.invalidateQueries({ queryKey: userKeys.detail(userId) })
```

## Custom Hooks

```typescript
// hooks/useUser.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { userKeys } from '@/lib/queries/keys'
import { fetchUser, updateUser, deleteUser } from '@/lib/api'

export function useUser(userId: string) {
  return useQuery({
    queryKey: userKeys.detail(userId),
    queryFn: () => fetchUser(userId),
    enabled: !!userId,
  })
}

export function useUpdateUser() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: updateUser,
    onSuccess: (data, variables) => {
      queryClient.setQueryData(userKeys.detail(variables.id), data)
      queryClient.invalidateQueries({ queryKey: userKeys.lists() })
    },
  })
}

export function useDeleteUser() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: deleteUser,
    onSuccess: (_, userId) => {
      queryClient.removeQueries({ queryKey: userKeys.detail(userId) })
      queryClient.invalidateQueries({ queryKey: userKeys.lists() })
    },
  })
}
```

## Infinite Queries

```tsx
import { useInfiniteQuery } from '@tanstack/react-query'

function Posts() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
  } = useInfiniteQuery({
    queryKey: ['posts'],
    queryFn: ({ pageParam = 0 }) =>
      fetch(`/api/posts?offset=${pageParam}&limit=10`).then((r) => r.json()),
    initialPageParam: 0,
    getNextPageParam: (lastPage, allPages) =>
      lastPage.hasMore ? allPages.length * 10 : undefined,
  })

  const posts = data?.pages.flatMap((page) => page.posts) ?? []

  return (
    <div>
      {posts.map((post) => (
        <Post key={post.id} post={post} />
      ))}
      <button
        onClick={() => fetchNextPage()}
        disabled={!hasNextPage || isFetchingNextPage}
      >
        {isFetchingNextPage
          ? 'Loading more...'
          : hasNextPage
          ? 'Load More'
          : 'Nothing more to load'}
      </button>
    </div>
  )
}
```

## Prefetching

```tsx
// Prefetch on hover
function PostLink({ postId }: { postId: string }) {
  const queryClient = useQueryClient()

  const prefetchPost = () => {
    queryClient.prefetchQuery({
      queryKey: ['post', postId],
      queryFn: () => fetchPost(postId),
      staleTime: 5 * 60 * 1000, // 5 minutes
    })
  }

  return (
    <Link
      href={`/posts/${postId}`}
      onMouseEnter={prefetchPost}
      onFocus={prefetchPost}
    >
      View Post
    </Link>
  )
}
```

## Server-Side Rendering (Next.js)

```tsx
// app/users/page.tsx
import { dehydrate, HydrationBoundary, QueryClient } from '@tanstack/react-query'

export default async function UsersPage() {
  const queryClient = new QueryClient()

  await queryClient.prefetchQuery({
    queryKey: ['users'],
    queryFn: fetchUsers,
  })

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <UsersList />
    </HydrationBoundary>
  )
}
```

## Query Options

```typescript
// Reusable query options
export const userQueryOptions = (userId: string) => ({
  queryKey: userKeys.detail(userId),
  queryFn: () => fetchUser(userId),
  staleTime: 5 * 60 * 1000,
  gcTime: 10 * 60 * 1000,
  enabled: !!userId,
})

// Use in component
useQuery(userQueryOptions(userId))

// Use for prefetching
queryClient.prefetchQuery(userQueryOptions(userId))
```

## Best Practices

1. **Use query key factories** - Consistent, type-safe keys
2. **Extract to custom hooks** - Reusable, testable
3. **Configure staleTime** - Reduce unnecessary refetches
4. **Optimistic updates** - Better UX for mutations
5. **Prefetch on intent** - Hover, focus events
6. **Handle loading/error states** - Always show feedback
7. **Use React Query Devtools** - Debug in development
8. **Invalidate strategically** - Don't over-invalidate
