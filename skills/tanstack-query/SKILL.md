# TanStack Query

The missing data-fetching library for React. Makes fetching, caching, synchronizing, and updating server state a breeze.

## Installation

```bash
npm install @tanstack/react-query
```

## Setup

```tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5, // 5 minutes
      gcTime: 1000 * 60 * 30,   // 30 minutes (formerly cacheTime)
    },
  },
})

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <YourApp />
    </QueryClientProvider>
  )
}
```

## Basic Query

```tsx
import { useQuery } from '@tanstack/react-query'

function Todos() {
  const { data, isPending, isError, error, refetch } = useQuery({
    queryKey: ['todos'],
    queryFn: async () => {
      const res = await fetch('/api/todos')
      if (!res.ok) throw new Error('Failed to fetch')
      return res.json()
    },
  })

  if (isPending) return <Loading />
  if (isError) return <Error message={error.message} />

  return (
    <ul>
      {data.map(todo => <li key={todo.id}>{todo.title}</li>)}
    </ul>
  )
}
```

## Query Keys

```tsx
// Simple key
useQuery({ queryKey: ['todos'], ... })

// With variables (automatic refetch when they change)
useQuery({ queryKey: ['todos', { status, page }], ... })

// Hierarchical
useQuery({ queryKey: ['todos', todoId, 'comments'], ... })

// All of these share cache:
// ['todos'] -> all todos
// ['todos', todoId] -> specific todo
// ['todos', todoId, 'comments'] -> todo's comments
```

## Mutations

```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query'

function AddTodo() {
  const queryClient = useQueryClient()
  
  const mutation = useMutation({
    mutationFn: (newTodo: Todo) => {
      return fetch('/api/todos', {
        method: 'POST',
        body: JSON.stringify(newTodo),
      })
    },
    onSuccess: () => {
      // Invalidate and refetch
      queryClient.invalidateQueries({ queryKey: ['todos'] })
    },
  })

  return (
    <button
      onClick={() => mutation.mutate({ title: 'New Todo' })}
      disabled={mutation.isPending}
    >
      {mutation.isPending ? 'Adding...' : 'Add Todo'}
    </button>
  )
}
```

## Optimistic Updates

### Via UI (Simpler)

```tsx
const addTodoMutation = useMutation({
  mutationFn: (newTodo: string) => axios.post('/api/todos', { text: newTodo }),
  onSettled: () => queryClient.invalidateQueries({ queryKey: ['todos'] }),
})

// In render - show optimistic item while pending
<ul>
  {todos.map(todo => <li key={todo.id}>{todo.text}</li>)}
  {addTodoMutation.isPending && (
    <li style={{ opacity: 0.5 }}>{addTodoMutation.variables}</li>
  )}
</ul>
```

### Via Cache (Rollback Support)

```tsx
const mutation = useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    // Cancel outgoing refetches
    await queryClient.cancelQueries({ queryKey: ['todos', newTodo.id] })
    
    // Snapshot previous value
    const previousTodo = queryClient.getQueryData(['todos', newTodo.id])
    
    // Optimistically update
    queryClient.setQueryData(['todos', newTodo.id], newTodo)
    
    // Return context for rollback
    return { previousTodo }
  },
  onError: (err, newTodo, context) => {
    // Rollback on error
    queryClient.setQueryData(['todos', newTodo.id], context.previousTodo)
  },
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] })
  },
})
```

## Pagination

```tsx
function Projects() {
  const [page, setPage] = useState(0)
  
  const { data, isPending, isPlaceholderData } = useQuery({
    queryKey: ['projects', page],
    queryFn: () => fetchProjects(page),
    placeholderData: keepPreviousData, // Keep showing old data
  })

  return (
    <div>
      {data.projects.map(project => <Project key={project.id} {...project} />)}
      <button
        onClick={() => setPage(old => old - 1)}
        disabled={page === 0}
      >
        Previous
      </button>
      <button
        onClick={() => setPage(old => old + 1)}
        disabled={isPlaceholderData || !data?.hasMore}
      >
        Next
      </button>
    </div>
  )
}
```

## Infinite Queries

```tsx
import { useInfiniteQuery } from '@tanstack/react-query'

function Projects() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: ['projects'],
    queryFn: ({ pageParam }) => fetchProjects(pageParam),
    initialPageParam: 0,
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  })

  return (
    <>
      {data.pages.map((page, i) => (
        <React.Fragment key={i}>
          {page.projects.map(project => <Project key={project.id} {...project} />)}
        </React.Fragment>
      ))}
      <button
        onClick={() => fetchNextPage()}
        disabled={!hasNextPage || isFetchingNextPage}
      >
        {isFetchingNextPage ? 'Loading...' : hasNextPage ? 'Load More' : 'No more'}
      </button>
    </>
  )
}
```

## Prefetching

```tsx
// In a component
const queryClient = useQueryClient()

// Prefetch on hover
<Link
  to="/todos"
  onMouseEnter={() => {
    queryClient.prefetchQuery({
      queryKey: ['todos'],
      queryFn: fetchTodos,
    })
  }}
>
  Todos
</Link>

// Prefetch in loader (React Router)
export async function loader({ params }) {
  await queryClient.ensureQueryData({
    queryKey: ['todo', params.id],
    queryFn: () => fetchTodo(params.id),
  })
  return null
}
```

## Dependent Queries

```tsx
// Second query depends on first
function UserPosts({ userId }) {
  const userQuery = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  })

  const postsQuery = useQuery({
    queryKey: ['posts', userId],
    queryFn: () => fetchPosts(userId),
    enabled: !!userQuery.data, // Only run when user is loaded
  })
}
```

## Parallel Queries

```tsx
import { useQueries } from '@tanstack/react-query'

function UserStats({ userIds }) {
  const userQueries = useQueries({
    queries: userIds.map(id => ({
      queryKey: ['user', id],
      queryFn: () => fetchUser(id),
    })),
  })
  
  const isLoading = userQueries.some(q => q.isPending)
  const users = userQueries.map(q => q.data)
}
```

## Suspense Mode

```tsx
import { useSuspenseQuery } from '@tanstack/react-query'

function Todos() {
  // No need for loading states - Suspense handles it
  const { data } = useSuspenseQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
  })

  return <TodoList todos={data} />
}

// Wrap with Suspense
<Suspense fallback={<Loading />}>
  <Todos />
</Suspense>
```

## DevTools

```tsx
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <YourApp />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  )
}
```

## Custom Hooks Pattern

```tsx
// hooks/useTodos.ts
export function useTodos(filters?: TodoFilters) {
  return useQuery({
    queryKey: ['todos', filters],
    queryFn: () => fetchTodos(filters),
    staleTime: 1000 * 60 * 5,
  })
}

export function useAddTodo() {
  const queryClient = useQueryClient()
  
  return useMutation({
    mutationFn: addTodo,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] })
    },
  })
}

// Usage
function Todos() {
  const { data: todos } = useTodos({ status: 'active' })
  const addTodo = useAddTodo()
  // ...
}
```

## Best Practices

1. **Use query keys wisely** - Include all dependencies
2. **Set appropriate staleTime** - Reduce unnecessary refetches
3. **Wrap in custom hooks** - Reuse query logic
4. **Invalidate strategically** - Don't over-invalidate
5. **Use placeholderData** - Better UX during pagination
6. **Prefetch on hover** - Perceived performance boost
7. **Enable DevTools in dev** - Debug cache issues
