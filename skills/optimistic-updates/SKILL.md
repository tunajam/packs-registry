# Optimistic Updates

Patterns for providing instant UI feedback before server confirmation, with proper error handling and rollback.

## Core Concept

1. Update UI immediately (optimistically)
2. Send request to server
3. On success: Keep the optimistic update
4. On error: Rollback to previous state

## TanStack Query Pattern

### Via UI (Simple)

```tsx
const addTodoMutation = useMutation({
  mutationFn: (newTodo: string) => 
    fetch('/api/todos', {
      method: 'POST',
      body: JSON.stringify({ text: newTodo }),
    }),
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] })
  },
})

function TodoList() {
  const { data: todos } = useQuery({ queryKey: ['todos'], queryFn: fetchTodos })
  const { mutate, variables, isPending, isError } = addTodoMutation

  return (
    <ul>
      {todos?.map(todo => <li key={todo.id}>{todo.text}</li>)}
      
      {/* Show optimistic item while pending */}
      {isPending && (
        <li style={{ opacity: 0.5 }}>{variables}</li>
      )}
      
      {/* Show failed item with retry */}
      {isError && (
        <li style={{ color: 'red' }}>
          {variables}
          <button onClick={() => mutate(variables)}>Retry</button>
        </li>
      )}
    </ul>
  )
}
```

### Via Cache (With Rollback)

```tsx
const updateTodoMutation = useMutation({
  mutationFn: updateTodo,
  
  onMutate: async (newTodo) => {
    // Cancel outgoing refetches
    await queryClient.cancelQueries({ queryKey: ['todos', newTodo.id] })
    
    // Snapshot previous value
    const previousTodo = queryClient.getQueryData(['todos', newTodo.id])
    
    // Optimistically update cache
    queryClient.setQueryData(['todos', newTodo.id], newTodo)
    
    // Return context for rollback
    return { previousTodo }
  },
  
  onError: (err, newTodo, context) => {
    // Rollback on error
    if (context?.previousTodo) {
      queryClient.setQueryData(['todos', newTodo.id], context.previousTodo)
    }
    toast.error('Failed to update todo')
  },
  
  onSettled: () => {
    // Refetch to ensure sync
    queryClient.invalidateQueries({ queryKey: ['todos'] })
  },
})
```

### List Operations

```tsx
const deleteTodoMutation = useMutation({
  mutationFn: (todoId: string) => deleteTodo(todoId),
  
  onMutate: async (todoId) => {
    await queryClient.cancelQueries({ queryKey: ['todos'] })
    
    const previousTodos = queryClient.getQueryData<Todo[]>(['todos'])
    
    // Optimistically remove from list
    queryClient.setQueryData<Todo[]>(['todos'], (old) => 
      old?.filter(todo => todo.id !== todoId)
    )
    
    return { previousTodos }
  },
  
  onError: (err, todoId, context) => {
    // Restore the list
    queryClient.setQueryData(['todos'], context?.previousTodos)
  },
  
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] })
  },
})
```

## SWR Pattern

```tsx
import useSWR, { mutate } from 'swr'

function TodoList() {
  const { data: todos } = useSWR('/api/todos', fetcher)

  async function addTodo(text: string) {
    const optimisticTodo = { id: 'temp-' + Date.now(), text, completed: false }
    
    // Optimistic update
    mutate('/api/todos', [...(todos || []), optimisticTodo], false)
    
    try {
      // Actual request
      await fetch('/api/todos', {
        method: 'POST',
        body: JSON.stringify({ text }),
      })
      
      // Revalidate
      mutate('/api/todos')
    } catch (error) {
      // Rollback
      mutate('/api/todos', todos, false)
      toast.error('Failed to add todo')
    }
  }
}
```

### useSWRMutation

```tsx
import useSWRMutation from 'swr/mutation'

function TodoList() {
  const { data: todos } = useSWR('/api/todos', fetcher)
  
  const { trigger } = useSWRMutation('/api/todos', addTodoFn, {
    optimisticData: (currentData, { arg }) => [
      ...(currentData || []),
      { id: 'temp', text: arg.text, completed: false }
    ],
    rollbackOnError: true,
    populateCache: true,
    revalidate: true,
  })

  return (
    <button onClick={() => trigger({ text: 'New Todo' })}>
      Add Todo
    </button>
  )
}
```

## Zustand Pattern

```tsx
interface TodoStore {
  todos: Todo[]
  pendingTodos: Map<string, Todo>
  addTodo: (text: string) => Promise<void>
}

const useTodoStore = create<TodoStore>((set, get) => ({
  todos: [],
  pendingTodos: new Map(),

  addTodo: async (text: string) => {
    const tempId = `temp-${Date.now()}`
    const optimisticTodo = { id: tempId, text, completed: false }
    
    // Add to pending (shows with different styling)
    set((state) => ({
      pendingTodos: new Map(state.pendingTodos).set(tempId, optimisticTodo)
    }))
    
    try {
      const newTodo = await createTodo(text)
      
      // Move from pending to real todos
      set((state) => {
        const pending = new Map(state.pendingTodos)
        pending.delete(tempId)
        return {
          todos: [...state.todos, newTodo],
          pendingTodos: pending,
        }
      })
    } catch (error) {
      // Remove from pending (rollback)
      set((state) => {
        const pending = new Map(state.pendingTodos)
        pending.delete(tempId)
        return { pendingTodos: pending }
      })
      throw error
    }
  },
}))

function TodoList() {
  const { todos, pendingTodos } = useTodoStore()
  
  return (
    <ul>
      {todos.map(todo => <TodoItem key={todo.id} todo={todo} />)}
      {[...pendingTodos.values()].map(todo => (
        <TodoItem key={todo.id} todo={todo} isPending />
      ))}
    </ul>
  )
}
```

## React Reducer Pattern

```tsx
type State = {
  todos: Todo[]
  optimisticTodos: Map<string, Todo>
}

type Action =
  | { type: 'OPTIMISTIC_ADD'; todo: Todo }
  | { type: 'CONFIRM_ADD'; tempId: string; todo: Todo }
  | { type: 'ROLLBACK_ADD'; tempId: string }

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'OPTIMISTIC_ADD':
      return {
        ...state,
        optimisticTodos: new Map(state.optimisticTodos).set(
          action.todo.id,
          action.todo
        ),
      }
    
    case 'CONFIRM_ADD': {
      const optimistic = new Map(state.optimisticTodos)
      optimistic.delete(action.tempId)
      return {
        todos: [...state.todos, action.todo],
        optimisticTodos: optimistic,
      }
    }
    
    case 'ROLLBACK_ADD': {
      const optimistic = new Map(state.optimisticTodos)
      optimistic.delete(action.tempId)
      return { ...state, optimisticTodos: optimistic }
    }
  }
}
```

## Toggle Pattern (Like Button)

```tsx
function LikeButton({ postId, initialLiked }: { postId: string; initialLiked: boolean }) {
  const [liked, setLiked] = useState(initialLiked)
  const [isPending, setIsPending] = useState(false)

  async function toggleLike() {
    const previousLiked = liked
    
    // Optimistic update
    setLiked(!liked)
    setIsPending(true)
    
    try {
      await fetch(`/api/posts/${postId}/like`, {
        method: liked ? 'DELETE' : 'POST',
      })
    } catch (error) {
      // Rollback
      setLiked(previousLiked)
      toast.error('Failed to update')
    } finally {
      setIsPending(false)
    }
  }

  return (
    <button 
      onClick={toggleLike} 
      disabled={isPending}
      className={liked ? 'liked' : ''}
    >
      {liked ? '❤️' : '🤍'}
    </button>
  )
}
```

## Form with Optimistic Submission

```tsx
function CommentForm({ postId }: { postId: string }) {
  const [optimisticComments, setOptimisticComments] = useState<Comment[]>([])
  const { data: comments } = useQuery({ queryKey: ['comments', postId] })

  async function handleSubmit(formData: FormData) {
    const text = formData.get('text') as string
    const tempId = `temp-${Date.now()}`
    
    const optimisticComment = {
      id: tempId,
      text,
      author: currentUser,
      createdAt: new Date().toISOString(),
    }
    
    // Add optimistically
    setOptimisticComments(prev => [...prev, optimisticComment])
    
    try {
      await addComment(postId, text)
      // Remove from optimistic, real data will come from refetch
      setOptimisticComments(prev => prev.filter(c => c.id !== tempId))
      queryClient.invalidateQueries({ queryKey: ['comments', postId] })
    } catch (error) {
      setOptimisticComments(prev => prev.filter(c => c.id !== tempId))
      toast.error('Failed to post comment')
    }
  }

  const allComments = [...(comments || []), ...optimisticComments]
  
  return (
    <div>
      {allComments.map(comment => (
        <Comment 
          key={comment.id} 
          comment={comment}
          isPending={comment.id.startsWith('temp-')}
        />
      ))}
      <form action={handleSubmit}>
        <input name="text" required />
        <button type="submit">Post</button>
      </form>
    </div>
  )
}
```

## Best Practices

1. **Visual feedback** - Show pending state (opacity, spinner)
2. **Always handle errors** - Rollback and notify user
3. **Use temp IDs** - Distinguish optimistic from real items
4. **Revalidate on settle** - Ensure eventual consistency
5. **Cancel ongoing requests** - Prevent race conditions
6. **Consider offline** - Queue mutations for later
7. **Test rollbacks** - Ensure error handling works
