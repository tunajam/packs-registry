# Pub/Sub Patterns

Event-driven patterns for decoupling components. Publish/Subscribe allows loose coupling between event producers and consumers.

## Core Concepts

- **Publisher**: Emits events without knowing who listens
- **Subscriber**: Reacts to events without knowing who publishes
- **Event Bus**: Central hub that routes events

## Native EventTarget

Modern, browser-native approach:

```ts
// Create event bus
const eventBus = new EventTarget()

// Subscribe
eventBus.addEventListener('user:login', (e: CustomEvent) => {
  console.log('User logged in:', e.detail)
})

// Publish
eventBus.dispatchEvent(
  new CustomEvent('user:login', { detail: { userId: '123' } })
)
```

### Typed EventTarget

```ts
type EventMap = {
  'user:login': { userId: string; email: string }
  'user:logout': { userId: string }
  'cart:update': { items: CartItem[] }
}

class TypedEventTarget<T extends Record<string, unknown>> extends EventTarget {
  on<K extends keyof T>(type: K, listener: (event: CustomEvent<T[K]>) => void) {
    this.addEventListener(type as string, listener as EventListener)
    return () => this.removeEventListener(type as string, listener as EventListener)
  }

  emit<K extends keyof T>(type: K, detail: T[K]) {
    this.dispatchEvent(new CustomEvent(type as string, { detail }))
  }
}

const bus = new TypedEventTarget<EventMap>()

// Fully typed!
bus.on('user:login', (e) => {
  console.log(e.detail.userId) // typed as string
})

bus.emit('user:login', { userId: '123', email: 'test@example.com' })
```

## EventEmitter Pattern

Classic Node.js-style emitter:

```ts
type Listener<T = unknown> = (data: T) => void

class EventEmitter<Events extends Record<string, unknown>> {
  private listeners = new Map<keyof Events, Set<Listener>>()

  on<K extends keyof Events>(event: K, listener: Listener<Events[K]>) {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set())
    }
    this.listeners.get(event)!.add(listener as Listener)
    
    // Return unsubscribe function
    return () => this.off(event, listener)
  }

  off<K extends keyof Events>(event: K, listener: Listener<Events[K]>) {
    this.listeners.get(event)?.delete(listener as Listener)
  }

  emit<K extends keyof Events>(event: K, data: Events[K]) {
    this.listeners.get(event)?.forEach((listener) => listener(data))
  }

  once<K extends keyof Events>(event: K, listener: Listener<Events[K]>) {
    const wrapper = (data: Events[K]) => {
      listener(data)
      this.off(event, wrapper)
    }
    this.on(event, wrapper)
  }
}

// Usage
interface AppEvents {
  'user:login': { userId: string }
  'theme:change': { theme: 'light' | 'dark' }
}

const emitter = new EventEmitter<AppEvents>()

const unsubscribe = emitter.on('user:login', ({ userId }) => {
  console.log('User logged in:', userId)
})

emitter.emit('user:login', { userId: '123' })
unsubscribe()
```

## Observer Pattern

Object-oriented approach:

```ts
interface Observer<T> {
  update(data: T): void
}

class Subject<T> {
  private observers = new Set<Observer<T>>()

  subscribe(observer: Observer<T>) {
    this.observers.add(observer)
    return () => this.unsubscribe(observer)
  }

  unsubscribe(observer: Observer<T>) {
    this.observers.delete(observer)
  }

  notify(data: T) {
    this.observers.forEach((observer) => observer.update(data))
  }
}

// Usage
class UserLoginObserver implements Observer<{ userId: string }> {
  update(data: { userId: string }) {
    console.log('User logged in:', data.userId)
  }
}

const loginSubject = new Subject<{ userId: string }>()
const observer = new UserLoginObserver()

loginSubject.subscribe(observer)
loginSubject.notify({ userId: '123' })
```

## Message Bus (Complex Routing)

For larger applications with namespaced events:

```ts
type MessageHandler<T = unknown> = (message: T, metadata: MessageMetadata) => void

interface MessageMetadata {
  topic: string
  timestamp: number
  source?: string
}

class MessageBus {
  private handlers = new Map<string, Set<MessageHandler>>()
  private wildcardHandlers = new Set<MessageHandler>()

  subscribe<T>(topic: string, handler: MessageHandler<T>) {
    if (topic === '*') {
      this.wildcardHandlers.add(handler as MessageHandler)
    } else {
      if (!this.handlers.has(topic)) {
        this.handlers.set(topic, new Set())
      }
      this.handlers.get(topic)!.add(handler as MessageHandler)
    }

    return () => this.unsubscribe(topic, handler)
  }

  unsubscribe<T>(topic: string, handler: MessageHandler<T>) {
    if (topic === '*') {
      this.wildcardHandlers.delete(handler as MessageHandler)
    } else {
      this.handlers.get(topic)?.delete(handler as MessageHandler)
    }
  }

  publish<T>(topic: string, message: T, source?: string) {
    const metadata: MessageMetadata = {
      topic,
      timestamp: Date.now(),
      source,
    }

    // Notify topic-specific handlers
    this.handlers.get(topic)?.forEach((handler) => handler(message, metadata))

    // Notify wildcard handlers
    this.wildcardHandlers.forEach((handler) => handler(message, metadata))

    // Notify parent topic handlers (e.g., 'user:*' for 'user:login')
    const parts = topic.split(':')
    for (let i = parts.length - 1; i > 0; i--) {
      const parentTopic = parts.slice(0, i).join(':') + ':*'
      this.handlers.get(parentTopic)?.forEach((handler) => handler(message, metadata))
    }
  }
}

// Usage
const bus = new MessageBus()

// Subscribe to specific topic
bus.subscribe('user:login', (data, meta) => {
  console.log(`[${meta.topic}] User:`, data)
})

// Subscribe to all user events
bus.subscribe('user:*', (data, meta) => {
  console.log(`[User Event] ${meta.topic}:`, data)
})

// Subscribe to everything
bus.subscribe('*', (data, meta) => {
  console.log(`[All] ${meta.topic}:`, data)
})

bus.publish('user:login', { userId: '123' }, 'auth-service')
```

## Async Event Handling

```ts
class AsyncEventEmitter<Events extends Record<string, unknown>> {
  private listeners = new Map<keyof Events, Set<(data: any) => Promise<void>>>()

  on<K extends keyof Events>(
    event: K,
    listener: (data: Events[K]) => Promise<void>
  ) {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set())
    }
    this.listeners.get(event)!.add(listener)
    return () => this.listeners.get(event)?.delete(listener)
  }

  async emit<K extends keyof Events>(event: K, data: Events[K]) {
    const listeners = this.listeners.get(event)
    if (!listeners) return

    // Parallel execution
    await Promise.all([...listeners].map((listener) => listener(data)))
  }

  async emitSerial<K extends keyof Events>(event: K, data: Events[K]) {
    const listeners = this.listeners.get(event)
    if (!listeners) return

    // Sequential execution
    for (const listener of listeners) {
      await listener(data)
    }
  }
}
```

## Browser Custom Events

For cross-component communication in vanilla JS:

```ts
// Define event types
declare global {
  interface WindowEventMap {
    'app:theme-change': CustomEvent<{ theme: 'light' | 'dark' }>
    'app:user-login': CustomEvent<{ userId: string }>
  }
}

// Publish
window.dispatchEvent(
  new CustomEvent('app:theme-change', { 
    detail: { theme: 'dark' },
    bubbles: true,
  })
)

// Subscribe
window.addEventListener('app:theme-change', (e) => {
  console.log('Theme changed to:', e.detail.theme)
})
```

## Middleware Pattern

Add cross-cutting concerns:

```ts
type Middleware<T> = (data: T, next: () => void) => void

class MiddlewareEventEmitter<Events extends Record<string, unknown>> {
  private listeners = new Map<keyof Events, Set<(data: any) => void>>()
  private middleware: Middleware<unknown>[] = []

  use(middleware: Middleware<unknown>) {
    this.middleware.push(middleware)
  }

  on<K extends keyof Events>(event: K, listener: (data: Events[K]) => void) {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set())
    }
    this.listeners.get(event)!.add(listener)
    return () => this.listeners.get(event)?.delete(listener)
  }

  emit<K extends keyof Events>(event: K, data: Events[K]) {
    const execute = (index: number) => {
      if (index < this.middleware.length) {
        this.middleware[index](data, () => execute(index + 1))
      } else {
        this.listeners.get(event)?.forEach((listener) => listener(data))
      }
    }
    execute(0)
  }
}

// Usage - add logging middleware
const emitter = new MiddlewareEventEmitter<AppEvents>()

emitter.use((data, next) => {
  console.log('Event received:', data)
  next()
})

emitter.use((data, next) => {
  // Could add validation, transformation, etc.
  next()
})
```

## Best Practices

1. **Use typed events** - Prevent typos and wrong payloads
2. **Return unsubscribe functions** - Prevent memory leaks
3. **Namespace events** - `domain:action` format
4. **Handle errors** - Don't let one listener break others
5. **Document event contracts** - What data each event carries
6. **Consider ordering** - Parallel vs sequential handlers
7. **Clean up subscriptions** - Especially in React useEffect
