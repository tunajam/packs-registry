# Event-Driven React Patterns

Patterns for event handling, cross-component communication, and cleanup in React applications.

## useEventListener Hook

Generic hook for DOM events with cleanup:

```tsx
import { useEffect, useRef } from 'react'

function useEventListener<K extends keyof WindowEventMap>(
  eventName: K,
  handler: (event: WindowEventMap[K]) => void,
  element: HTMLElement | Window = window
) {
  const savedHandler = useRef(handler)

  useEffect(() => {
    savedHandler.current = handler
  }, [handler])

  useEffect(() => {
    const listener = (event: Event) => savedHandler.current(event as WindowEventMap[K])
    element.addEventListener(eventName, listener)
    return () => element.removeEventListener(eventName, listener)
  }, [eventName, element])
}

// Usage
function Component() {
  useEventListener('keydown', (e) => {
    if (e.key === 'Escape') closeModal()
  })

  useEventListener('resize', () => {
    console.log('Window resized')
  })
}
```

## useCustomEvent Hook

For custom event communication:

```tsx
import { useEffect, useCallback } from 'react'

function useCustomEvent<T>(
  eventName: string,
  handler?: (data: T) => void
) {
  useEffect(() => {
    if (!handler) return

    const listener = (event: CustomEvent<T>) => handler(event.detail)
    window.addEventListener(eventName, listener as EventListener)
    return () => window.removeEventListener(eventName, listener as EventListener)
  }, [eventName, handler])

  const emit = useCallback((data: T) => {
    window.dispatchEvent(new CustomEvent(eventName, { detail: data }))
  }, [eventName])

  return emit
}

// Usage - Publisher
function Publisher() {
  const emitThemeChange = useCustomEvent<{ theme: string }>('theme:change')
  
  return (
    <button onClick={() => emitThemeChange({ theme: 'dark' })}>
      Dark Mode
    </button>
  )
}

// Usage - Subscriber
function Subscriber() {
  useCustomEvent<{ theme: string }>('theme:change', ({ theme }) => {
    console.log('Theme changed to:', theme)
  })
  
  return null
}
```

## Event Bus Context

Scoped event bus for React tree:

```tsx
import { createContext, useContext, useRef, useCallback } from 'react'

type Listener<T = unknown> = (data: T) => void

interface EventBus {
  on: <T>(event: string, listener: Listener<T>) => () => void
  emit: <T>(event: string, data: T) => void
}

const EventBusContext = createContext<EventBus | null>(null)

export function EventBusProvider({ children }: { children: React.ReactNode }) {
  const listenersRef = useRef(new Map<string, Set<Listener>>())

  const on = useCallback(<T,>(event: string, listener: Listener<T>) => {
    if (!listenersRef.current.has(event)) {
      listenersRef.current.set(event, new Set())
    }
    listenersRef.current.get(event)!.add(listener as Listener)
    
    return () => {
      listenersRef.current.get(event)?.delete(listener as Listener)
    }
  }, [])

  const emit = useCallback(<T,>(event: string, data: T) => {
    listenersRef.current.get(event)?.forEach((listener) => listener(data))
  }, [])

  return (
    <EventBusContext.Provider value={{ on, emit }}>
      {children}
    </EventBusContext.Provider>
  )
}

export function useEventBus() {
  const context = useContext(EventBusContext)
  if (!context) throw new Error('useEventBus must be used within EventBusProvider')
  return context
}

// Custom hook for subscribing
export function useSubscribe<T>(event: string, handler: (data: T) => void) {
  const { on } = useEventBus()
  
  useEffect(() => {
    return on(event, handler)
  }, [event, handler, on])
}

// Usage
function App() {
  return (
    <EventBusProvider>
      <Publisher />
      <Subscriber />
    </EventBusProvider>
  )
}

function Publisher() {
  const { emit } = useEventBus()
  return (
    <button onClick={() => emit('notification', { message: 'Hello!' })}>
      Notify
    </button>
  )
}

function Subscriber() {
  useSubscribe<{ message: string }>('notification', ({ message }) => {
    toast(message)
  })
  return null
}
```

## useOnClickOutside

Classic pattern for modals/dropdowns:

```tsx
import { useEffect, RefObject } from 'react'

function useOnClickOutside<T extends HTMLElement>(
  ref: RefObject<T>,
  handler: (event: MouseEvent | TouchEvent) => void
) {
  useEffect(() => {
    const listener = (event: MouseEvent | TouchEvent) => {
      if (!ref.current || ref.current.contains(event.target as Node)) {
        return
      }
      handler(event)
    }

    document.addEventListener('mousedown', listener)
    document.addEventListener('touchstart', listener)
    
    return () => {
      document.removeEventListener('mousedown', listener)
      document.removeEventListener('touchstart', listener)
    }
  }, [ref, handler])
}

// Usage
function Dropdown() {
  const [isOpen, setIsOpen] = useState(false)
  const ref = useRef<HTMLDivElement>(null)
  
  useOnClickOutside(ref, () => setIsOpen(false))
  
  return (
    <div ref={ref}>
      <button onClick={() => setIsOpen(!isOpen)}>Toggle</button>
      {isOpen && <div className="dropdown-content">...</div>}
    </div>
  )
}
```

## useKeyPress

Handle keyboard shortcuts:

```tsx
function useKeyPress(targetKey: string, handler: () => void) {
  useEffect(() => {
    const listener = (event: KeyboardEvent) => {
      if (event.key === targetKey) {
        handler()
      }
    }
    
    window.addEventListener('keydown', listener)
    return () => window.removeEventListener('keydown', listener)
  }, [targetKey, handler])
}

// With modifiers
function useHotkey(
  keys: { key: string; ctrl?: boolean; shift?: boolean; alt?: boolean },
  handler: () => void
) {
  useEffect(() => {
    const listener = (event: KeyboardEvent) => {
      if (
        event.key.toLowerCase() === keys.key.toLowerCase() &&
        (!keys.ctrl || event.ctrlKey || event.metaKey) &&
        (!keys.shift || event.shiftKey) &&
        (!keys.alt || event.altKey)
      ) {
        event.preventDefault()
        handler()
      }
    }
    
    window.addEventListener('keydown', listener)
    return () => window.removeEventListener('keydown', listener)
  }, [keys, handler])
}

// Usage
function App() {
  useHotkey({ key: 's', ctrl: true }, () => {
    saveDocument()
  })
  
  useHotkey({ key: 'k', ctrl: true }, () => {
    openCommandPalette()
  })
}
```

## useWindowEvent

Window events with cleanup:

```tsx
function useWindowEvent<K extends keyof WindowEventMap>(
  event: K,
  handler: (event: WindowEventMap[K]) => void,
  options?: AddEventListenerOptions
) {
  const handlerRef = useRef(handler)
  handlerRef.current = handler

  useEffect(() => {
    const listener = (e: WindowEventMap[K]) => handlerRef.current(e)
    window.addEventListener(event, listener, options)
    return () => window.removeEventListener(event, listener, options)
  }, [event, options])
}

// Usage
function Component() {
  useWindowEvent('online', () => toast.success('Back online!'))
  useWindowEvent('offline', () => toast.error('Connection lost'))
  useWindowEvent('beforeunload', (e) => {
    if (hasUnsavedChanges) {
      e.preventDefault()
      e.returnValue = ''
    }
  })
}
```

## useMediaQuery

React to media query changes:

```tsx
function useMediaQuery(query: string) {
  const [matches, setMatches] = useState(() => 
    window.matchMedia(query).matches
  )

  useEffect(() => {
    const mediaQuery = window.matchMedia(query)
    const handler = (e: MediaQueryListEvent) => setMatches(e.matches)
    
    mediaQuery.addEventListener('change', handler)
    return () => mediaQuery.removeEventListener('change', handler)
  }, [query])

  return matches
}

// Usage
function Component() {
  const isMobile = useMediaQuery('(max-width: 768px)')
  const prefersDark = useMediaQuery('(prefers-color-scheme: dark)')
  
  return isMobile ? <MobileLayout /> : <DesktopLayout />
}
```

## useVisibilityChange

Track tab visibility:

```tsx
function useVisibilityChange(handler: (isVisible: boolean) => void) {
  useEffect(() => {
    const listener = () => handler(document.visibilityState === 'visible')
    document.addEventListener('visibilitychange', listener)
    return () => document.removeEventListener('visibilitychange', listener)
  }, [handler])
}

// Usage
function VideoPlayer() {
  const [isPlaying, setIsPlaying] = useState(true)
  
  useVisibilityChange((isVisible) => {
    if (!isVisible && isPlaying) {
      pauseVideo()
    }
  })
}
```

## useIntersectionObserver

Visibility detection:

```tsx
function useIntersectionObserver(
  ref: RefObject<Element>,
  options: IntersectionObserverInit = {}
) {
  const [entry, setEntry] = useState<IntersectionObserverEntry | null>(null)

  useEffect(() => {
    if (!ref.current) return

    const observer = new IntersectionObserver(
      ([entry]) => setEntry(entry),
      options
    )
    
    observer.observe(ref.current)
    return () => observer.disconnect()
  }, [ref, options.threshold, options.root, options.rootMargin])

  return entry
}

// Usage - Lazy load images
function LazyImage({ src, alt }: { src: string; alt: string }) {
  const ref = useRef<HTMLDivElement>(null)
  const entry = useIntersectionObserver(ref, { threshold: 0.1 })
  const isVisible = entry?.isIntersecting

  return (
    <div ref={ref}>
      {isVisible ? (
        <img src={src} alt={alt} />
      ) : (
        <div className="placeholder" />
      )}
    </div>
  )
}
```

## Event Cleanup Pattern

Always clean up in useEffect:

```tsx
// ❌ Bad - no cleanup
useEffect(() => {
  window.addEventListener('resize', handleResize)
}, [])

// ✅ Good - with cleanup
useEffect(() => {
  window.addEventListener('resize', handleResize)
  return () => window.removeEventListener('resize', handleResize)
}, [])

// ✅ Better - with AbortController
useEffect(() => {
  const controller = new AbortController()
  
  window.addEventListener('resize', handleResize, { signal: controller.signal })
  window.addEventListener('scroll', handleScroll, { signal: controller.signal })
  document.addEventListener('keydown', handleKey, { signal: controller.signal })
  
  return () => controller.abort()
}, [])
```

## Best Practices

1. **Always clean up** - Return cleanup functions from useEffect
2. **Use refs for handlers** - Avoid stale closures
3. **Debounce/throttle** - Don't fire too frequently
4. **Scope appropriately** - Window vs element vs context
5. **Type your events** - Leverage TypeScript
6. **Consider AbortController** - Modern cleanup pattern
7. **Extract to hooks** - Reusable event logic
