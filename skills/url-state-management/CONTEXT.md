# URL State Management

Using URL search parameters as the most powerful global state manager - shareable, bookmarkable, and browser-native.

## Why URL State?

- **Shareable** - Copy URL, share exact view
- **Bookmarkable** - Save for later
- **Browser navigation** - Back/forward works
- **SSR-friendly** - State available on first render
- **Debugging** - Visible in address bar

## React Router useSearchParams

```tsx
import { useSearchParams } from 'react-router'

function ProductList() {
  const [searchParams, setSearchParams] = useSearchParams()
  
  // Read params
  const page = parseInt(searchParams.get('page') ?? '1')
  const sort = searchParams.get('sort') ?? 'name'
  const filter = searchParams.get('filter') ?? 'all'
  
  // Update single param (preserves others)
  const setPage = (newPage: number) => {
    setSearchParams(prev => {
      prev.set('page', String(newPage))
      return prev
    })
  }
  
  // Update multiple params
  const applyFilters = (filters: Record<string, string>) => {
    setSearchParams(prev => {
      Object.entries(filters).forEach(([key, value]) => {
        if (value) {
          prev.set(key, value)
        } else {
          prev.delete(key)
        }
      })
      return prev
    })
  }
  
  return (
    <div>
      <Filters
        sort={sort}
        filter={filter}
        onChange={applyFilters}
      />
      <Products page={page} sort={sort} filter={filter} />
      <Pagination page={page} onPageChange={setPage} />
    </div>
  )
}
```

## TanStack Router (Type-Safe)

```tsx
import { createFileRoute } from '@tanstack/react-router'
import { z } from 'zod'

// Define search param schema
const productSearchSchema = z.object({
  page: z.number().default(1),
  sort: z.enum(['name', 'price', 'date']).default('name'),
  direction: z.enum(['asc', 'desc']).default('asc'),
  category: z.string().optional(),
  minPrice: z.number().optional(),
  maxPrice: z.number().optional(),
})

export const Route = createFileRoute('/products/')({
  validateSearch: productSearchSchema,
  component: ProductsPage,
})

function ProductsPage() {
  // Fully typed!
  const { page, sort, direction, category, minPrice, maxPrice } = Route.useSearch()
  const navigate = Route.useNavigate()
  
  const updateSearch = (updates: Partial<z.infer<typeof productSearchSchema>>) => {
    navigate({
      search: (prev) => ({ ...prev, ...updates }),
    })
  }
  
  return (
    <div>
      <select
        value={sort}
        onChange={(e) => updateSearch({ sort: e.target.value as any, page: 1 })}
      >
        <option value="name">Name</option>
        <option value="price">Price</option>
        <option value="date">Date</option>
      </select>
      
      <Products filters={{ category, minPrice, maxPrice }} sort={sort} page={page} />
      
      <button onClick={() => updateSearch({ page: page + 1 })}>
        Next Page
      </button>
    </div>
  )
}
```

## Custom Hook Pattern

```tsx
function useQueryParams<T extends Record<string, string | number | boolean | undefined>>() {
  const [searchParams, setSearchParams] = useSearchParams()
  
  const params = useMemo(() => {
    const result: Record<string, string> = {}
    searchParams.forEach((value, key) => {
      result[key] = value
    })
    return result as T
  }, [searchParams])
  
  const setParams = useCallback((
    updates: Partial<T> | ((prev: T) => Partial<T>)
  ) => {
    setSearchParams(prev => {
      const currentParams = Object.fromEntries(prev.entries())
      const newParams = typeof updates === 'function' 
        ? updates(currentParams as T)
        : updates
      
      Object.entries(newParams).forEach(([key, value]) => {
        if (value === undefined || value === null || value === '') {
          prev.delete(key)
        } else {
          prev.set(key, String(value))
        }
      })
      
      return prev
    })
  }, [setSearchParams])
  
  return [params, setParams] as const
}

// Usage
function Filters() {
  const [params, setParams] = useQueryParams<{
    search?: string
    category?: string
    page?: string
  }>()
  
  return (
    <input
      value={params.search ?? ''}
      onChange={(e) => setParams({ search: e.target.value, page: '1' })}
    />
  )
}
```

## nuqs (Type-Safe URL State)

```tsx
import { useQueryState, parseAsInteger, parseAsStringEnum } from 'nuqs'

function ProductList() {
  // Type-safe with parser
  const [page, setPage] = useQueryState('page', parseAsInteger.withDefault(1))
  const [sort, setSort] = useQueryState('sort', 
    parseAsStringEnum(['name', 'price', 'date']).withDefault('name')
  )
  
  return (
    <div>
      <button onClick={() => setPage(page + 1)}>
        Next (Page {page})
      </button>
      <select 
        value={sort} 
        onChange={(e) => setSort(e.target.value as any)}
      >
        <option value="name">Name</option>
        <option value="price">Price</option>
      </select>
    </div>
  )
}
```

### nuqs with Object State

```tsx
import { useQueryStates, parseAsInteger, parseAsString } from 'nuqs'

const searchParamsParsers = {
  page: parseAsInteger.withDefault(1),
  sort: parseAsString.withDefault('name'),
  q: parseAsString,
}

function ProductList() {
  const [{ page, sort, q }, setParams] = useQueryStates(searchParamsParsers)
  
  return (
    <div>
      <input
        value={q ?? ''}
        onChange={(e) => setParams({ q: e.target.value, page: 1 })}
      />
      <Pagination 
        page={page} 
        onPageChange={(p) => setParams({ page: p })}
      />
    </div>
  )
}
```

## Debounced Search

```tsx
function SearchInput() {
  const [searchParams, setSearchParams] = useSearchParams()
  const [localQuery, setLocalQuery] = useState(searchParams.get('q') ?? '')
  
  // Debounce URL updates
  useEffect(() => {
    const timer = setTimeout(() => {
      setSearchParams(prev => {
        if (localQuery) {
          prev.set('q', localQuery)
        } else {
          prev.delete('q')
        }
        prev.set('page', '1') // Reset page on search
        return prev
      })
    }, 300)
    
    return () => clearTimeout(timer)
  }, [localQuery, setSearchParams])
  
  return (
    <input
      value={localQuery}
      onChange={(e) => setLocalQuery(e.target.value)}
      placeholder="Search..."
    />
  )
}
```

## URL + TanStack Query

```tsx
function ProductList() {
  const [searchParams] = useSearchParams()
  
  const page = parseInt(searchParams.get('page') ?? '1')
  const sort = searchParams.get('sort') ?? 'name'
  
  // URL state drives query
  const { data } = useQuery({
    queryKey: ['products', { page, sort }],
    queryFn: () => fetchProducts({ page, sort }),
  })
  
  return <ProductGrid products={data} />
}
```

## Sync with Form State

```tsx
function FilterForm() {
  const [searchParams, setSearchParams] = useSearchParams()
  
  // Initialize form from URL
  const defaultValues = {
    category: searchParams.get('category') ?? '',
    minPrice: searchParams.get('minPrice') ?? '',
    maxPrice: searchParams.get('maxPrice') ?? '',
  }
  
  const { register, handleSubmit } = useForm({ defaultValues })
  
  const onSubmit = (data: typeof defaultValues) => {
    setSearchParams(prev => {
      Object.entries(data).forEach(([key, value]) => {
        if (value) prev.set(key, value)
        else prev.delete(key)
      })
      prev.set('page', '1') // Reset pagination
      return prev
    })
  }
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <select {...register('category')}>
        <option value="">All Categories</option>
        <option value="electronics">Electronics</option>
      </select>
      <input {...register('minPrice')} placeholder="Min Price" />
      <input {...register('maxPrice')} placeholder="Max Price" />
      <button type="submit">Apply Filters</button>
    </form>
  )
}
```

## Replace vs Push

```tsx
function Filters() {
  const [searchParams, setSearchParams] = useSearchParams()
  
  // Replace history (don't add to back stack)
  const updateFilter = (key: string, value: string) => {
    setSearchParams(prev => {
      prev.set(key, value)
      return prev
    }, { replace: true }) // User won't have to click back through every filter change
  }
}
```

## Best Practices

1. **Use for shareable state** - Filters, pagination, tabs, modals
2. **Reset pagination on filter change** - Set page=1 when filtering
3. **Debounce text inputs** - Don't update URL on every keystroke
4. **Replace for minor updates** - Use replace: true for filters
5. **Type your params** - Use zod/nuqs for validation
6. **Provide defaults** - Handle missing params gracefully
7. **Clean up empty params** - Delete params when value is empty
8. **SSR consideration** - Read params in loaders for SEO
