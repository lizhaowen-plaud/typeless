# Typeless Technical Reference for AI Agents

## Project Overview

**Typeless** is a TypeScript + React Hooks + RxJS state management library designed for building type-safe, scalable React applications with minimal boilerplate.

**Core Philosophy:**
- Type safety with minimal annotations
- Event-driven architecture using RxJS
- Dynamic loading of reducers and epics (no single global registry)
- Built-in code splitting and HMR support

**Tech Stack:**
- React ^16.8 (Hooks required)
- RxJS ^6
- TypeScript
- Immer (for immutable state updates)

---

## Architecture Overview

### Core Concepts Flow
```
Module → Actions → Dispatch → [Epic + Reducer] → State Update → React Re-render
                       ↓
                   Side Effects (Epic)
```

### Module System
**Entry Point:** `createModule(symbol)` creates a module with optional actions and state.

**Module Types:**
1. **Base Module**: `[Handle]` - no state, no actions
2. **Module with Actions**: `[Handle, Actions]` - has action creators
3. **Module with State**: `[HandleWithState, StateGetter]` - has state
4. **Full Module**: `[HandleWithState, Actions, StateGetter]` - has both

**Builder Pattern:**
```typescript
createModule(Symbol('myModule'))
  .withActions({ ... })  // Optional: add actions
  .withState<TState>()   // Optional: add state
```

---

## Module System Implementation

### 1. Module Creation (`createModule.ts`)

**Key Components:**
- **Handle**: React hook that registers epic/reducer with Registry
- **Actions**: Auto-generated action creators with type inference
- **StateGetter**: Function to retrieve current module state

**Lifecycle Hooks (Auto Actions):**
- `$init`: Dispatched when module first mounts (not on HMR)
- `$mounted`: Dispatched after mount effect
- `$remounted`: Dispatched on HMR remount
- `$unmounting`: Dispatched before unmount
- `$unmounted`: Dispatched after unmount

**Implementation Details:**
- Action types are tuples: `[Symbol, 'ACTION_NAME']`
- Action names auto-converted to SCREAMING_SNAKE_CASE
- State getter has `_module` and `_store` metadata for hooks
- Epic/reducer enabled/disabled dynamically via `Store.enable()` and `Store.disable()`

### 2. Registry System (`Registry.ts`)

**Purpose:** Central coordinator for all stores and action/epic streams.

**Key Responsibilities:**
1. Store management (create/retrieve stores by symbol)
2. Action dispatching to all stores
3. RxJS stream coordination (input$ → epic processing → output$)
4. State logging (dev mode only)

**Dispatch Flow:**
```
dispatch(action)
  → ReactDom.unstable_batchedUpdates
    → notify all stores (reducers)
    → update state
    → notify listeners (trigger React re-renders)
    → input$.next(action)
      → epic processing
        → dispatch result actions (recursive)
```

**Stream Architecture:**
- `input$`: Subject receiving all dispatched actions
- `output$`: Observable of epic results
- Output subscribes to input and re-dispatches results

### 3. Store (`Store.ts`)

**Purpose:** Container for a single module's state, reducer, and epic.

**Key Features:**
- Reference counting (`usageCount`) for dynamic enable/disable
- State persistence across remounts (unless manually reset)
- Listener pattern for React component subscriptions
- Epic stream generation per action

**State Management:**
- `initState()`: Called once on first mount (skipped on HMR)
- `state` persists even when `isEnabled = false`
- `isStateInited` flag prevents state reset on remount

### 4. Epic System (`Epic.ts`)

**Purpose:** Handle side effects (async operations, action chains, etc.)

**Handler Types:**
- `on(action, handler)`: Listen to single action
- `onMany([actions], handler)`: Listen to multiple actions
- `onModule(symbol, handler)`: Listen to all actions from a module

**Handler Return Types:**
- `Observable<Action>`: Stream of actions
- `Promise<Action>`: Async action
- `Action | Action[]`: Sync action(s)
- `null`: No action

**Stream Processing:**
- Each handler wrapped in `defer()` for lazy execution
- `mergeMap` to flatten nested actions
- `catchError` to prevent epic crashes (errors thrown async)
- Results validated (must be Action or Action[])

### 5. ChainedReducer (`ChainedReducer.ts`)

**Purpose:** Fluent API for building reducers with Immer integration.

**Key Methods:**
- `on(action, handler)`: Mutate state (uses Immer)
- `onMany([actions], handler)`: Handle multiple actions
- `replace(action, handler)`: Return new state (uses Immer)
- `attach(reducer)`: Compose reducers
- `nested(prop, fn)`: Nested reducer for sub-state
- `mergePayload(action)`: Merge action payload into state

**Implementation:**
- Internal `reducerMap` by `[symbol, type]`
- Immer's `produce()` for immutability
- `asReducer()` returns function with chained methods

### 6. RxJS Integration (`rx/`)

**ofType Operator (`ofType.ts`):**
- Filter action stream by action creator
- Compares `[symbol, type]` tuples
- Supports single or array of action creators

**waitForType:**
- Similar to `ofType` but waits for first matching action

---

## Hook System Implementation

### 1. useActions (`useActions.ts`)

**Purpose:** Wrap action creators with automatic dispatch.

**Implementation:**
```typescript
useActions({ increment, decrement })
// Returns: { increment: () => dispatch(increment()), ... }
```

**Features:**
- `useMemo` optimization based on action names
- Dispatches via Registry
- Returns action for chaining

### 2. useMappedState (`useMappedState.ts`)

**Purpose:** Subscribe to multiple stores with custom selector.

**Key Features:**
- Multi-store subscription
- Equality check to prevent unnecessary re-renders
- `useLayoutEffect` for immediate subscription (no missed updates)
- Ref-based state caching

**Implementation Flow:**
```
stateGetters → extract _store → subscribe to stores
  → getMappedState() → equality check → forceUpdate if changed
```

### 3. useSelector (`useSelector.ts`)

**Purpose:** Subscribe to computed selector values.

**Implementation:**
- Thin wrapper around `useMappedState`
- Uses `selector.getStateGetters()` for dependencies

### 4. useState (on StateGetter)

**Purpose:** Subscribe to single module state.

**Implementation:**
```typescript
getModuleState.useState()
// Internally: useMappedState([getState], state => state)
```

---

## Form System (`typeless-form`)

### createForm Implementation (`createForm.tsx`)

**Architecture:**
- Wraps `createModule` with form-specific actions
- Uses FormContext for field-level access

**State Structure:**
```typescript
{
  values: TData,           // Form data
  errors: { [K]?: string }, // Validation errors
  touched: { [K]?: boolean } // Touched fields
}
```

**Actions:**
- `change(field, value)`: Update single field
- `changeMany(values)`: Update multiple fields
- `replace(values)`: Replace entire form
- `blur(field)`: Mark field as touched
- `submit()`: Validate and submit
- `reset()`: Reset to initial state
- `setErrors(errors)`: Set validation errors

**Epic Logic:**
- Auto-validate on change/blur/submit
- Submit flow: validate → setErrors → check errors → submitSucceeded/Failed
- Uses `concatObs` to ensure sequential execution

**Validation:**
- Custom validator function: `(errors, data) => errors`
- Called automatically by epic
- Errors set via `setErrors` action

---

## Router System (`typeless-router`)

### createUseRouter Implementation (`module.ts`)

**Architecture:**
- Single global module with symbol `RouterSymbol`
- Browser History API or Hash-based routing

**State Structure:**
```typescript
{
  location: RouterLocation | null,    // Current location
  prevLocation: RouterLocation | null // Previous location
}
```

**Actions:**
- `locationChange(location)`: Internal, triggered by history change
- `push(location)`: Navigate to new location
- `replace(location)`: Replace current location

**Epic Logic:**
- `$init`: Setup popstate listener, emit initial location
- `push`: Call `history.pushState`, emit locationChange
- `replace`: Call `history.replaceState`, emit locationChange
- Uses `takeUntil(dispose)` for cleanup

**History Types:**
- `browser`: Uses pathname (HTML5 History API)
- `hash`: Uses hash-based routing (#/path)

---

## Hot Module Replacement (`onHmr.tsx`)

**Global State:**
- `isHmr` flag tracks HMR state

**Lifecycle:**
1. `startHmr()`: Set flag before module reload
2. Component remounts with `isHmr = true`
3. `<Hmr>` wrapper calls `stopHmr()` after render

**Module Behavior:**
- `isHmr = false`: Dispatch `$init` and `$mounted`
- `isHmr = true`: Skip `$init`, dispatch `$remounted`, preserve state

**Usage:**
```typescript
// In HMR setup
if (module.hot) {
  module.hot.accept('./App', () => {
    startHmr();
    // Re-render App
  });
}
```

---

## Code Splitting Strategy

**Automatic Support:**
- Modules register on component mount
- Epic/reducer loaded dynamically via `handle()` hook
- No global epic/reducer file required

**Pattern:**
```typescript
// In lazy-loaded component
const Module = React.lazy(() => import('./MyModule'));

// MyModule.tsx
export default function MyModule() {
  useMyModule(); // Registers epic/reducer
  // ...
}
```

**Benefits:**
- Epic/reducer bundled with component
- Automatic cleanup on unmount
- No manual registration needed

---

## State Flow Diagrams

### Dispatch Flow
```
Action
  ↓
Registry.dispatch()
  ↓
┌─────────────────────┐
│ For each Store:     │
│ 1. Call reducer     │
│ 2. Update state     │
│ 3. Notify listeners │ → React re-render
└─────────────────────┘
  ↓
input$.next(action)
  ↓
┌────────────────────┐
│ For each Epic:     │
│ 1. Find handlers   │
│ 2. Execute handler │
│ 3. Emit actions    │
└────────────────────┘
  ↓
output$ (recursive dispatch)
```

### React Integration Flow
```
Component Mount
  ↓
useModule() → handle()
  ↓
Registry.getStore(symbol)
  ↓
store.enable({ epic, reducer })
  ↓
store.initState() (if first mount)
  ↓
dispatch($init) / dispatch($mounted)
  ↓
useState() / useActions()
  ↓
Component subscribes to store
  ↓
Component Unmount
  ↓
dispatch($unmounting)
  ↓
store.disable()
  ↓
dispatch($unmounted)
```

---

## Key Design Patterns

### 1. Symbol-based Module Identity
- Each module identified by unique Symbol
- Prevents name collisions
- Enables multiple instances of same module

### 2. Tuple Action Types
- `[Symbol, 'TYPE']` instead of string
- Module namespace built-in
- Type-safe action matching

### 3. Builder Pattern (createModule)
- Fluent API for module configuration
- Type inference cascades through chain
- Minimal annotations required

### 4. Dynamic Registration
- Components register modules on mount
- No global singleton pattern
- Supports React.lazy and code splitting

### 5. Immer Integration
- Reducers use mutable-style syntax
- Immer handles immutability
- Cleaner reducer code

### 6. RxJS for Side Effects
- Epics return Observables/Promises
- Built-in async handling
- Composable with RxJS operators

---

## Performance Optimizations

### 1. Batched Updates
- Uses `ReactDom.unstable_batchedUpdates`
- Groups state changes in single render

### 2. Reference Counting
- Stores track usage count
- Disables epic/reducer when unused
- Keeps state for quick remount

### 3. Equality Checks
- `useMappedState` uses `objectIs` by default
- Custom equality functions supported
- Prevents unnecessary re-renders

### 4. QueueScheduler
- RxJS actions processed async via queueScheduler
- Prevents stack overflow on deep epic chains

### 5. Memoization
- `useActions` memoizes dispatch wrappers
- `useMappedState` memoizes selectors
- Reduces function allocations

---

## Development Features

### 1. State Logging
- `StateLogger` in dev mode only
- Logs prevState, action, nextState
- Only logs when state actually changes

### 2. Action Logging
- Logs epic execution in dev mode
- Shows store name and action type

### 3. HMR Support
- Preserves state across reloads
- Special `$remounted` lifecycle
- `<Hmr>` component for coordination

### 4. Error Handling
- Epic errors thrown async (setTimeout)
- Prevents breaking other epics
- Errors still visible in console

---

## Common Patterns and Examples

### Basic Module
```typescript
const [useCounter, CounterActions, getCounterState] = createModule(Symbol('counter'))
  .withActions({
    increment: null,
    decrement: null,
  })
  .withState<{ count: number }>();

useCounter
  .epic()
  .on(CounterActions.increment, () => {
    // side effects
  });

useCounter
  .reducer({ count: 0 })
  .on(CounterActions.increment, state => {
    state.count++;
  });
```

### Async Data Fetching
```typescript
useModule
  .epic()
  .on(Actions.fetch, () => {
    return fetch('/api/data')
      .then(res => res.json())
      .then(data => Actions.fetchSuccess(data))
      .catch(err => Actions.fetchError(err));
  });
```

### Epic Chains
```typescript
useModule
  .epic()
  .on(Actions.start, (_, { action$ }) => {
    return Rx.concat(
      Rx.of(Actions.step1()),
      action$.pipe(
        Rx.waitForType(Actions.step1Done),
        Rx.map(() => Actions.step2())
      ),
      action$.pipe(
        Rx.waitForType(Actions.step2Done),
        Rx.map(() => Actions.complete())
      )
    );
  });
```

### Multi-Store State
```typescript
const combined = useMappedState(
  [getUserState, getPostsState],
  (user, posts) => ({
    user,
    userPosts: posts.filter(p => p.userId === user.id)
  })
);
```

---

## File Structure Reference

```
packages/
├── typeless/                 # Core library
│   ├── src/
│   │   ├── createModule.ts    # Module builder
│   │   ├── Epic.ts            # Side effects handler
│   │   ├── Store.ts           # State container
│   │   ├── Registry.ts        # Global coordinator
│   │   ├── ChainedReducer.ts  # Reducer builder
│   │   ├── useActions.ts      # Action dispatcher hook
│   │   ├── useMappedState.ts  # Multi-store selector
│   │   ├── useSelector.ts     # Computed selector
│   │   ├── onHmr.tsx          # HMR support
│   │   └── rx/                # RxJS operators
│   │       ├── ofType.ts      # Action filter
│   │       └── waitForType.ts # Action waiter
│   └── package.json
├── typeless-form/            # Form management
│   ├── src/
│   │   ├── createForm.tsx    # Form module factory
│   │   └── FormContext.ts    # React context
│   └── package.json
└── typeless-router/          # Routing
    ├── src/
    │   ├── module.ts         # Router module
    │   ├── Link.tsx          # Link component
    │   └── utils.ts          # History helpers
    └── package.json
```

---

## Quick Reference Commands

### Testing
```bash
yarn test                # Run all tests
yarn build-dev           # TypeScript build (dev)
yarn build-prod          # TypeScript build (prod)
```

### Project Structure
- Monorepo using Lerna and Yarn Workspaces
- TypeScript project references for fast builds
- Separate CJS and ESM builds

---

## Key Type Definitions

### Action Type
```typescript
type ActionType = [symbol, string];

interface Action<T = any> {
  type: ActionType;
  payload?: T;
}
```

### Action Creator
```typescript
type AC = (...args: any[]) => { payload?: any };

// Converted to:
type ConvertedAC<T> = (...args) => Action<Payload>;
```

### Reducer
```typescript
type Reducer<S> = (state: S | undefined, action: ActionLike) => S;
```

### Epic Handler
```typescript
type EpicHandler<TAC> = (
  payload: Payload,
  deps: { action$: Observable<Action> },
  action: Action
) => Observable<Action> | Promise<Action> | Action | Action[] | null;
```

---

## Dependencies

### Core (typeless)
- `immer`: Immutable state updates
- `react`, `react-dom`: Peer dependencies
- `rxjs`: Peer dependency

### Form (typeless-form)
- Depends on `typeless`

### Router (typeless-router)
- Depends on `typeless`
- Uses native History API

---

## Important Notes for AI Agents

1. **Symbol Usage**: Every module MUST use a unique Symbol. Never use strings.

2. **Action Types**: Actions are `[Symbol, string]` tuples, not plain strings.

3. **State Mutations**: In reducers, mutate state directly (Immer handles it).

4. **Epic Returns**: Always return Observable, Promise, Action, Action[], or null.

5. **Lifecycle**: Use `$init`, `$mounted`, etc. for module lifecycle hooks.

6. **HMR**: State persists by default. Reset manually if needed.

7. **Subscription**: `useMappedState` uses `useLayoutEffect` for immediate subscription.

8. **Batching**: All dispatches are batched via `unstable_batchedUpdates`.

9. **Type Safety**: Types inferred from action definitions. Minimal annotations needed.

10. **Code Splitting**: Just use React.lazy - module registration is automatic.

---

## End of Document

This document provides a complete technical reference for understanding Typeless architecture, implementation details, and usage patterns. Use this to quickly answer questions about module structure, implementation patterns, and system behavior.
