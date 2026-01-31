# XState State Machines

Actor-based state management & orchestration for complex app logic. XState uses event-driven programming, state machines, and statecharts to handle complex logic in predictable, robust, and visual ways.

## Installation

```bash
npm install xstate @xstate/react
```

## Core Concepts

### Basic State Machine

```tsx
import { createMachine, createActor } from 'xstate'

const toggleMachine = createMachine({
  id: 'toggle',
  initial: 'inactive',
  states: {
    inactive: {
      on: { TOGGLE: 'active' }
    },
    active: {
      on: { TOGGLE: 'inactive' }
    }
  }
})

// Create actor (instance)
const toggleActor = createActor(toggleMachine)

// Subscribe to state changes
toggleActor.subscribe((state) => {
  console.log(state.value) // 'inactive' or 'active'
})

// Start the actor
toggleActor.start()

// Send events
toggleActor.send({ type: 'TOGGLE' })
```

### With Context (Extended State)

```tsx
import { createMachine, assign } from 'xstate'

const counterMachine = createMachine({
  id: 'counter',
  initial: 'active',
  context: {
    count: 0,
  },
  states: {
    active: {
      on: {
        INCREMENT: {
          actions: assign({
            count: ({ context }) => context.count + 1
          })
        },
        DECREMENT: {
          actions: assign({
            count: ({ context }) => context.count - 1
          })
        }
      }
    }
  }
})
```

## React Integration

### useActor Hook

```tsx
import { useMachine } from '@xstate/react'

function Toggle() {
  const [state, send] = useMachine(toggleMachine)
  
  return (
    <button onClick={() => send({ type: 'TOGGLE' })}>
      {state.value === 'active' ? 'ON' : 'OFF'}
    </button>
  )
}
```

### With Context Values

```tsx
function Counter() {
  const [state, send] = useMachine(counterMachine)
  
  return (
    <div>
      <span>{state.context.count}</span>
      <button onClick={() => send({ type: 'INCREMENT' })}>+</button>
      <button onClick={() => send({ type: 'DECREMENT' })}>-</button>
    </div>
  )
}
```

## Hierarchical (Nested) States

```tsx
const lightMachine = createMachine({
  id: 'light',
  initial: 'green',
  states: {
    green: {
      on: { TIMER: 'yellow' }
    },
    yellow: {
      on: { TIMER: 'red' }
    },
    red: {
      initial: 'walk',
      states: {
        walk: {
          on: { PED_TIMER: 'wait' }
        },
        wait: {
          on: { PED_TIMER: 'stop' }
        },
        stop: {}
      },
      on: { TIMER: 'green' }
    }
  }
})

// Access nested state
state.value // { red: 'walk' }
state.matches('red.walk') // true
```

## Parallel States

```tsx
const wordMachine = createMachine({
  id: 'word',
  type: 'parallel',
  states: {
    bold: {
      initial: 'off',
      states: {
        on: { on: { TOGGLE_BOLD: 'off' } },
        off: { on: { TOGGLE_BOLD: 'on' } }
      }
    },
    italic: {
      initial: 'off',
      states: {
        on: { on: { TOGGLE_ITALIC: 'off' } },
        off: { on: { TOGGLE_ITALIC: 'on' } }
      }
    }
  }
})

// State value is an object
// { bold: 'on', italic: 'off' }
```

## Guards (Conditional Transitions)

```tsx
const paymentMachine = createMachine({
  id: 'payment',
  initial: 'idle',
  context: {
    amount: 0,
  },
  states: {
    idle: {
      on: {
        SUBMIT: {
          target: 'processing',
          guard: ({ context }) => context.amount > 0
        }
      }
    },
    processing: {
      // ...
    }
  }
})
```

## Actions

```tsx
const machine = createMachine({
  // ...
  states: {
    active: {
      entry: ['logEntry', 'notifyUser'],
      exit: ['logExit'],
      on: {
        SUBMIT: {
          target: 'submitted',
          actions: ['saveData', 'sendAnalytics']
        }
      }
    }
  }
}, {
  actions: {
    logEntry: () => console.log('Entered active'),
    logExit: () => console.log('Exited active'),
    saveData: ({ context, event }) => saveToDb(context),
    sendAnalytics: () => analytics.track('submitted'),
    notifyUser: () => toast.success('Active!'),
  }
})
```

## Async with Promises (Invoke)

```tsx
const fetchMachine = createMachine({
  id: 'fetch',
  initial: 'idle',
  context: {
    data: null,
    error: null,
  },
  states: {
    idle: {
      on: { FETCH: 'loading' }
    },
    loading: {
      invoke: {
        src: 'fetchData',
        onDone: {
          target: 'success',
          actions: assign({
            data: ({ event }) => event.output
          })
        },
        onError: {
          target: 'failure',
          actions: assign({
            error: ({ event }) => event.error
          })
        }
      }
    },
    success: {},
    failure: {
      on: { RETRY: 'loading' }
    }
  }
}, {
  actors: {
    fetchData: fromPromise(async () => {
      const response = await fetch('/api/data')
      return response.json()
    })
  }
})
```

## Spawned Actors

```tsx
import { createMachine, assign, spawn } from 'xstate'

const parentMachine = createMachine({
  context: {
    children: [],
  },
  on: {
    SPAWN_CHILD: {
      actions: assign({
        children: ({ context, spawn }) => [
          ...context.children,
          spawn(childMachine)
        ]
      })
    }
  }
})
```

## History States

```tsx
const paymentMachine = createMachine({
  id: 'payment',
  initial: 'method',
  states: {
    method: {
      initial: 'card',
      states: {
        card: { on: { SWITCH: 'cash' } },
        cash: { on: { SWITCH: 'card' } },
        hist: { type: 'history' } // remembers last child
      },
      on: { NEXT: 'review' }
    },
    review: {
      on: { BACK: 'method.hist' } // returns to last method
    }
  }
})
```

## Delayed Transitions

```tsx
const lightMachine = createMachine({
  id: 'light',
  initial: 'green',
  states: {
    green: {
      after: {
        3000: 'yellow' // transition after 3 seconds
      }
    },
    yellow: {
      after: {
        1000: 'red'
      }
    },
    red: {
      after: {
        2000: 'green'
      }
    }
  }
})
```

## Visual Debugging

### Stately Studio

Create and visualize machines at [state.new](https://state.new)

### Inspector

```tsx
import { createBrowserInspector } from '@statelyai/inspect'

const inspector = createBrowserInspector()

const actor = createActor(machine, {
  inspect: inspector.inspect,
})
```

## TypeScript

```tsx
import { createMachine, assign } from 'xstate'

interface Context {
  count: number
  user: User | null
}

type Events =
  | { type: 'INCREMENT' }
  | { type: 'DECREMENT' }
  | { type: 'SET_USER'; user: User }

const machine = createMachine({
  types: {} as {
    context: Context
    events: Events
  },
  context: {
    count: 0,
    user: null,
  },
  // ...
})
```

## When to Use XState

- Complex UI flows (multi-step wizards, auth)
- Async workflows with multiple states
- Feature flags with state dependencies
- Form validation with complex rules
- Game state management
- WebSocket connection management
- Any logic that benefits from visualization

## Best Practices

1. **Model the states first** - Think about states before events
2. **Use guards for conditions** - Keep transitions predictable
3. **Visualize your machine** - Use Stately Studio
4. **Keep context minimal** - Only store what changes behavior
5. **Use actors for isolation** - Spawn for independent logic
6. **Type your events** - Union types for event payloads
