---
name: angular-signals
description: Expert knowledge for Angular Signals — reactive primitives, computed values, effects, signal inputs/outputs, signal queries, Resource API, and RxJS interop. Use when building reactive Angular features, migrating from observables, or designing state management with signals.
triggers:
  - angular signals
  - signal()
  - computed()
  - effect()
  - signal input
  - toSignal
  - fromSignal
  - resource api
  - angular reactive
  - linkedSignal
---

# Angular Signals Expert

You are now loaded with deep knowledge about Angular Signals. Apply these patterns when helping the user build reactive Angular applications.

## Core Primitives

### `signal()` — Writable reactive value
```typescript
import { signal } from '@angular/core';

const count = signal(0);          // WritableSignal<number>
count();                          // read: 0
count.set(5);                     // set to 5
count.update(n => n + 1);        // update based on current
count.mutate(arr => arr.push(1)); // mutate in place (for objects/arrays)
```

### `computed()` — Derived value (memoized)
```typescript
import { computed } from '@angular/core';

const count = signal(3);
const doubled = computed(() => count() * 2);  // lazy, memoized
doubled();  // 6 — only recalculates when count changes
```

- Computed signals are **lazy**: only evaluated when read
- Automatically tracks dependencies — no manual subscription
- Cannot be written to

### `effect()` — Side effects
```typescript
import { effect } from '@angular/core';

// In a component/service (injection context)
effect(() => {
  console.log('count changed:', count());
  // any signal read here is tracked
});
```

**Rules for `effect()`:**
- Must run in injection context (constructor, class field, or `runInInjectionContext`)
- Cannot write to signals by default (use `allowSignalWrites: true` if needed, but prefer `linkedSignal` or `computed`)
- Cleanup: return a function or use `onCleanup`

```typescript
effect((onCleanup) => {
  const sub = someObservable.subscribe();
  onCleanup(() => sub.unsubscribe());
});
```

## Signal Inputs (Angular 17.1+)

Replace `@Input()` decorator with `input()` function:
```typescript
import { input } from '@angular/core';

export class UserCardComponent {
  // optional input (User | undefined)
  user = input<User>();

  // required input
  userId = input.required<string>();

  // with alias
  theme = input<string>('light', { alias: 'colorTheme' });

  // transform
  disabled = input(false, { transform: booleanAttribute });
}
```

Signal inputs are **read-only signals** — access like `this.userId()`.

## Signal Outputs (Angular 17.3+)

```typescript
import { output } from '@angular/core';

export class ButtonComponent {
  clicked = output<void>();
  selected = output<string>();

  onClick() {
    this.clicked.emit();
    this.selected.emit('value');
  }
}
```

## Signal Queries (Angular 17.2+)

Replace `@ViewChild` / `@ContentChild` with signal-based queries:
```typescript
import { viewChild, viewChildren, contentChild, contentChildren } from '@angular/core';

export class ParentComponent {
  // single
  header = viewChild<ElementRef>('headerRef');           // Signal<ElementRef | undefined>
  header = viewChild.required<ElementRef>('headerRef'); // Signal<ElementRef>

  // multiple
  items = viewChildren<ItemComponent>(ItemComponent);   // Signal<readonly ItemComponent[]>

  // content projection
  tabs = contentChildren<TabComponent>(TabComponent);
}
```

Access with `this.header()` — reactive, no need for `ngAfterViewInit`.

## `linkedSignal()` (Angular 19+)

For writable derived state that resets when source changes:
```typescript
import { linkedSignal } from '@angular/core';

const options = signal(['a', 'b', 'c']);
const selected = linkedSignal(() => options()[0]); // resets when options changes

selected.set('b');  // writable
```

## RxJS Interop

### Observable → Signal
```typescript
import { toSignal } from '@angular/core/rxjs-interop';

@Component({ ... })
export class MyComponent {
  private userService = inject(UserService);

  // converts observable to signal
  users = toSignal(this.userService.getAll(), { initialValue: [] });

  // with requireSync (observable must emit synchronously)
  config = toSignal(this.configService.config$, { requireSync: true });
}
```

### Signal → Observable
```typescript
import { toObservable } from '@angular/core/rxjs-interop';

count = signal(0);
count$ = toObservable(this.count);  // Observable<number>

// useful for debouncing signal-based search
searchTerm = signal('');
debouncedSearch$ = toObservable(this.searchTerm).pipe(
  debounceTime(300),
  distinctUntilChanged()
);
```

### `takeUntilDestroyed()`
```typescript
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

someObservable.pipe(
  takeUntilDestroyed()  // auto-unsubscribes on component destroy
).subscribe(...)
```

## Resource API (Angular 19+)

For async data loading with signals:
```typescript
import { resource } from '@angular/core';

export class UserComponent {
  userId = input.required<string>();

  userResource = resource({
    request: () => ({ id: this.userId() }),
    loader: ({ request }) => fetch(`/api/users/${request.id}`).then(r => r.json()),
  });

  // access state
  user = this.userResource.value;       // Signal<User | undefined>
  isLoading = this.userResource.isLoading; // Signal<boolean>
  error = this.userResource.error;      // Signal<unknown>

  // imperative refresh
  refresh() { this.userResource.reload(); }
}
```

Use `httpResource()` for Angular HttpClient integration:
```typescript
import { httpResource } from '@angular/common/http';

userResource = httpResource<User>(() => `/api/users/${this.userId()}`);
```

## State Management with Signals

### Simple Local State
```typescript
// In component — no store needed for local state
export class CartComponent {
  private items = signal<CartItem[]>([]);
  count = computed(() => this.items().length);
  total = computed(() => this.items().reduce((sum, i) => sum + i.price, 0));

  addItem(item: CartItem) {
    this.items.update(items => [...items, item]);
  }
}
```

### Service-Based State (Shared)
```typescript
@Injectable({ providedIn: 'root' })
export class CartStore {
  private _items = signal<CartItem[]>([]);

  // expose as readonly
  items = this._items.asReadonly();
  count = computed(() => this._items().length);

  addItem(item: CartItem) {
    this._items.update(items => [...items, item]);
  }
}
```

## Signals vs Observables — When to Use Which

| Use Signals | Use Observables |
|---|---|
| Component state | WebSocket streams |
| Derived/computed values | Complex async operators (retry, race) |
| Template bindings | Multiple subscribers with different lifecycles |
| Service state | Event streams (fromEvent, interval) |
| Simple async data (resource) | HTTP with complex error handling |

**Rule of thumb**: Start with signals. Reach for observables when you need stream operators or multi-cast behavior.

## Checklist
- [ ] Signal inputs used instead of `@Input()` decorator?
- [ ] `viewChild`/`contentChild` signal queries used?
- [ ] `toSignal()` used to bridge HTTP observables to templates?
- [ ] `takeUntilDestroyed()` used for observable subscriptions?
- [ ] `effect()` avoided for state derivation (use `computed()` instead)?
- [ ] Service state exposed as `readonly` signals?
- [ ] `resource()` or `httpResource()` for async loading?
