# SWR Data Fetching

React Hooks for data fetching. SWR (stale-while-revalidate) returns cached data first, then fetches fresh data, and finally updates.

## Installation

```bash
npm install swr
```

## Basic Usage

```tsx
import useSWR from 'swr'

// Define a fetcher
const fetcher = (url: string) => fetch(url).then(res => res.json())

function Profile() {
  const { data, error, isLoading } = useSWR('/api/user', fetcher)

  if (error) return <div>Failed to load</div>
  if (isLoading) return <div>Loading...</div>
  
  return <div>Hello {data.name}!</div>
}
```

## Custom Fetcher

```tsx
// With axios
import axios from 'axios'

const fetcher = (url: string) => axios.get(url).then(res => res.data)

// With GraphQL
const fetcher = (query: string) =>
  request('/api/graphql', query).then(res => res.data)

// Multiple arguments
const fetcher = ([url, token]: [string, string]) =>
  fetch(url, { headers: { Authorization: token } }).then(res => res.json())

useSWR(['/api/user', token], fetcher)
```

## Reusable Hooks

```tsx
function useUser(id: string) {
  const { data, error, isLoading } = useSWR(
    `/api/users/${id}`,
    fetcher
  )

  return {
    user: data,
    isLoading,
    isError: error,
  }
}

// Components stay independent
function Avatar({ id }) {
  const { user, isLoading } = useUser(id)
  if (isLoading) return <Spinner />
  return <img src={user.avatar} />
}

function Username({ id }) {
  const { user, isLoading } = useUser(id)
  if (isLoading) return <Spinner />
  return <span>{user.name}</span>
}

// Both components use same hook - request is deduped!
```

## Global Configuration

```tsx
import { SWRConfig } from 'swr'

function App() {
  return (
    <SWRConfig 
      value={{
        fetcher: (url) => fetch(url).then(res => res.json()),
        refreshInterval: 3000,
        revalidateOnFocus: true,
        dedupingInterval: 2000,
        errorRetryCount: 3,
      }}
    >
      <YourApp />
    </SWRConfig>
  )
}
```

## Conditional Fetching

```tsx
// Conditional - pass null to disable
const { data } = useSWR(
  userId ? `/api/users/${userId}` : null,
  fetcher
)

// Dependent fetching
function MyProjects() {
  const { data: user } = useSWR('/api/user', fetcher)
  const { data: projects } = useSWR(
    () => '/api/projects?uid=' + user.id,  // throws if user is undefined
    fetcher
  )
}
```

## Mutation (Local State Update)

```tsx
import useSWR, { mutate } from 'swr'

function Profile() {
  const { data } = useSWR('/api/user', fetcher)

  async function updateName(newName: string) {
    // Optimistically update local data
    mutate('/api/user', { ...data, name: newName }, false)
    
    // Send update to API
    await updateUser({ name: newName })
    
    // Revalidate to ensure data is in sync
    mutate('/api/user')
  }
}
```

### useSWRMutation (v2+)

```tsx
import useSWRMutation from 'swr/mutation'

async function updateUser(url: string, { arg }: { arg: { name: string } }) {
  return fetch(url, {
    method: 'POST',
    body: JSON.stringify(arg),
  }).then(res => res.json())
}

function Profile() {
  const { data } = useSWR('/api/user', fetcher)
  const { trigger, isMutating } = useSWRMutation('/api/user', updateUser)

  return (
    <button
      disabled={isMutating}
      onClick={() => trigger({ name: 'New Name' })}
    >
      Update Name
    </button>
  )
}
```

## Optimistic Updates

```tsx
import useSWRMutation from 'swr/mutation'

function TodoList() {
  const { data: todos } = useSWR('/api/todos', fetcher)
  
  const { trigger } = useSWRMutation('/api/todos', addTodo, {
    optimisticData: (currentData) => [
      ...currentData,
      { id: 'temp', title: 'New Todo', completed: false }
    ],
    rollbackOnError: true,
    populateCache: true,
    revalidate: false, // Don't revalidate after mutation
  })
}
```

## Pagination

```tsx
function Repos() {
  const [page, setPage] = useState(1)
  
  const { data, isLoading } = useSWR(
    `/api/repos?page=${page}`,
    fetcher,
    { keepPreviousData: true }
  )

  return (
    <div>
      {data?.repos.map(repo => <Repo key={repo.id} {...repo} />)}
      <button onClick={() => setPage(p => p + 1)}>Next</button>
    </div>
  )
}
```

## Infinite Loading

```tsx
import useSWRInfinite from 'swr/infinite'

const getKey = (pageIndex: number, previousPageData: any) => {
  if (previousPageData && !previousPageData.length) return null
  return `/api/items?page=${pageIndex}&limit=10`
}

function Items() {
  const { data, size, setSize, isLoading } = useSWRInfinite(getKey, fetcher)

  const items = data ? data.flat() : []
  const isLoadingMore = isLoading || (size > 0 && !data[size - 1])

  return (
    <div>
      {items.map(item => <Item key={item.id} {...item} />)}
      <button
        disabled={isLoadingMore}
        onClick={() => setSize(size + 1)}
      >
        Load More
      </button>
    </div>
  )
}
```

## Prefetching

```tsx
import { preload } from 'swr'

// Preload before render
preload('/api/user', fetcher)

// On hover
<Link
  onMouseEnter={() => preload('/api/data', fetcher)}
  href="/data"
>
  Go to Data
</Link>
```

## Subscription (Real-time)

```tsx
import useSWRSubscription from 'swr/subscription'

function Messages() {
  const { data, error } = useSWRSubscription('/api/messages', (key, { next }) => {
    const socket = new WebSocket(key)
    
    socket.onmessage = (event) => {
      next(null, JSON.parse(event.data))
    }
    
    socket.onerror = (error) => next(error)
    
    return () => socket.close()
  })
}
```

## Error Handling

```tsx
const { data, error, isLoading } = useSWR('/api/user', fetcher, {
  onError: (error, key) => {
    if (error.status !== 403 && error.status !== 404) {
      // Handle error (e.g., Sentry)
    }
  },
  errorRetryCount: 3,
  errorRetryInterval: 5000,
  shouldRetryOnError: (error) => error.status !== 401,
})
```

## Middleware

```tsx
import useSWR, { SWRConfig } from 'swr'

// Logger middleware
const logger = (useSWRNext) => (key, fetcher, config) => {
  const swr = useSWRNext(key, fetcher, config)
  
  useEffect(() => {
    console.log('SWR:', key, swr.data)
  }, [key, swr.data])
  
  return swr
}

<SWRConfig value={{ use: [logger] }}>
  <App />
</SWRConfig>
```

## TypeScript

```tsx
interface User {
  id: string
  name: string
  email: string
}

function useUser(id: string) {
  return useSWR<User, Error>(
    `/api/users/${id}`,
    fetcher
  )
}

const { data } = useUser('1')
// data is User | undefined
```

## Best Practices

1. **Create custom hooks** - Encapsulate and reuse fetching logic
2. **Use conditional keys** - Pass null to pause fetching
3. **Configure globally** - Set fetcher and options in SWRConfig
4. **Use keepPreviousData** - Better UX during pagination
5. **Dedupe requests** - SWR does this automatically
6. **Handle errors gracefully** - Use onError and error boundaries
7. **Prefetch on interaction** - Improve perceived performance
