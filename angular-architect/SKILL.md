---
name: angular-architect
description: Expert knowledge for Angular application architecture — component patterns, change detection, standalone APIs, dependency injection, routing, and module structure. Use when designing Angular features, structuring components, choosing patterns, or refactoring architecture.
triggers:
  - angular architecture
  - angular component pattern
  - smart dumb component
  - container presentational
  - OnPush
  - standalone component
  - angular DI
  - angular module
  - angular routing
  - feature module
---

# Angular Architecture Expert

You are now loaded with deep knowledge about Angular application architecture. Apply these patterns when helping the user design or build Angular applications.

## Core Architecture Principles

- **Separation of concerns**: UI (dumb), orchestration (smart), data (services), logic (utils)
- **Unidirectional data flow**: data down (inputs), events up (outputs)
- **OnPush by default**: opt-in to full CD only when necessary
- **Standalone first** (Angular 15+): prefer standalone over NgModule unless sharing across lazy routes

## Component Patterns

### Smart (Container) Component
- Knows about services, state, routing
- Handles side effects (HTTP, store dispatch)
- Passes data down to dumb children via `@Input()`
- Listens to events from children via `@Output()`

```typescript
@Component({
  selector: 'app-user-dashboard',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<app-user-list [users]="users()" (select)="onSelect($event)" />`
})
export class UserDashboardComponent {
  private userService = inject(UserService);
  users = toSignal(this.userService.getAll());

  onSelect(id: string) {
    this.router.navigate(['/users', id]);
  }
}
```

### Dumb (Presentational) Component
- No service injection (exception: pure formatting services)
- Pure function of inputs → rendered output
- Communicates only via `@Input()` / `@Output()`
- Always `OnPush`

```typescript
@Component({
  selector: 'app-user-list',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    @for (user of users; track user.id) {
      <app-user-card [user]="user" (click)="select.emit(user.id)" />
    }
  `
})
export class UserListComponent {
  @Input({ required: true }) users!: User[];
  @Output() select = new EventEmitter<string>();
}
```

## Change Detection Strategy

### OnPush Rules
With `OnPush`, Angular only checks when:
1. An `@Input()` reference changes
2. An event originated from the component or a descendant
3. An async pipe resolves
4. `ChangeDetectorRef.markForCheck()` is called
5. A signal used in the template changes

**Use `OnPush` on every component.** Exceptions are rare.

### Triggering CD Manually
```typescript
// inject when needed
private cdr = inject(ChangeDetectorRef);

// after async updates not tracked by Angular
this.cdr.markForCheck();

// for one-off imperative updates
this.cdr.detectChanges();
```

## Standalone Components (Angular 17+)

```typescript
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet, CommonModule, MyFeatureComponent],
  template: `<router-outlet />`
})
export class AppComponent {}
```

Bootstrap:
```typescript
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    provideHttpClient(withInterceptors([authInterceptor])),
    provideAnimations(),
  ]
});
```

## Routing Patterns

### Lazy Loading (standalone)
```typescript
export const routes: Routes = [
  {
    path: 'users',
    loadComponent: () => import('./users/users.component').then(m => m.UsersComponent),
  },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.routes').then(m => m.ADMIN_ROUTES),
  }
];
```

### Route Guards (functional)
```typescript
export const authGuard: CanActivateFn = (route, state) => {
  const auth = inject(AuthService);
  return auth.isLoggedIn() ? true : inject(Router).createUrlTree(['/login']);
};
```

### Route-Level Data Loading (Resolvers)
```typescript
export const userResolver: ResolveFn<User> = (route) =>
  inject(UserService).getById(route.paramMap.get('id')!);
```

## Dependency Injection

### Providing Services
```typescript
// Singleton (root scope) — preferred
@Injectable({ providedIn: 'root' })
export class UserService {}

// Feature scope (lazy-loaded route)
@Injectable({ providedIn: 'platform' })
export class PlatformService {}

// Component scope
@Component({ providers: [LocalService] })
export class MyComponent {}
```

### Injection Tokens
```typescript
const API_URL = new InjectionToken<string>('API_URL');

// provide
providers: [{ provide: API_URL, useValue: 'https://api.example.com' }]

// inject
const apiUrl = inject(API_URL);
```

### `inject()` vs Constructor Injection
Prefer `inject()` in modern Angular — works in constructors, class fields, and standalone functions:
```typescript
export class MyComponent {
  private service = inject(MyService);  // preferred
}
```

## Content Projection

```typescript
// Single slot
<ng-content />

// Named slots
<ng-content select="[header]" />
<ng-content select="[body]" />

// With fallback (Angular 18+)
<ng-content>Default content</ng-content>
```

## Host Binding & Directives

```typescript
@Directive({
  selector: '[appHighlight]',
  standalone: true,
  host: {
    '[class.highlighted]': 'isHighlighted',
    '(mouseenter)': 'onEnter()',
  }
})
export class HighlightDirective {
  isHighlighted = false;
  onEnter() { this.isHighlighted = true; }
}
```

## Architecture Decision Rules

| Situation | Pattern |
|---|---|
| Fetching data, dispatching actions | Smart component |
| Rendering data, emitting events | Dumb component |
| Cross-component state | Service + signals/RxJS |
| Shared UI across features | `ui` library |
| Feature-specific state | `data-access` library |
| Page with routing | `feature` library |
| Pure logic, no Angular | `util` library |

## Common Anti-Patterns to Avoid

- Injecting services into dumb components
- Using `Default` change detection
- Putting business logic in components (belongs in services)
- Deeply nested component trees without `OnPush`
- God components (> 200 lines of template)
- Importing everything in `AppModule` (prefer lazy loading)

## Checklist
- [ ] Every component uses `ChangeDetectionStrategy.OnPush`?
- [ ] Smart/dumb separation enforced?
- [ ] Services injected only in smart components or other services?
- [ ] Routes lazy-loaded?
- [ ] `inject()` used instead of constructor injection?
- [ ] Libraries follow domain/type organization?
- [ ] No circular dependencies between libraries?
