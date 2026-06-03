---
name: perf-auditor
description: Expert knowledge for Angular performance optimization — change detection tuning, bundle analysis, lazy loading, Core Web Vitals, deferrable views, SSR/SSG, and trackBy. Use when optimizing Angular app performance, reducing bundle size, improving load times, or diagnosing performance issues.
triggers:
  - performance
  - perf audit
  - bundle size
  - lazy loading
  - change detection
  - Core Web Vitals
  - LCP
  - SSR
  - angular universal
  - deferrable views
  - trackBy
  - bundle analysis
---

# Angular Performance Expert

You are now loaded with deep knowledge about Angular performance optimization. Apply these patterns when helping the user improve their Angular application's performance.

## Core Web Vitals Targets

| Metric | Good | Needs Work | Poor |
|---|---|---|---|
| LCP (Largest Contentful Paint) | < 2.5s | 2.5–4s | > 4s |
| INP (Interaction to Next Paint) | < 200ms | 200–500ms | > 500ms |
| CLS (Cumulative Layout Shift) | < 0.1 | 0.1–0.25 | > 0.25 |

## Change Detection Optimization

### 1. Always Use `OnPush`
```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush
})
```
With `OnPush`, Angular skips the component unless:
- An `@Input()` reference changes
- An event from the component fires
- An async pipe resolves
- A signal in the template changes

### 2. Signals (Angular 17+) — Best CD Performance
```typescript
// Template reads signals → Angular knows exactly which components to update
users = signal<User[]>([]);
count = computed(() => this.users().length);
```
Signals enable **fine-grained reactivity** — no zone.js needed.

### 3. Zone-less Angular (`provideExperimentalZonelessChangeDetection`)
```typescript
// main.ts — Angular 18+
bootstrapApplication(AppComponent, {
  providers: [provideExperimentalZonelessChangeDetection()]
});
```
Remove zone.js from polyfills. Requires signals for all change detection.

### 4. `trackBy` for `*ngFor` / `@for`
```typescript
// Old ngFor
<li *ngFor="let item of items; trackBy: trackById">

trackById(index: number, item: Item) { return item.id; }

// Modern @for (Angular 17+) — track required
@for (item of items; track item.id) {
  <app-item [item]="item" />
}
```
Without `track`, Angular destroys and recreates DOM nodes on every change.

### 5. Avoid Expensive Template Expressions
```typescript
// BAD — called on every CD cycle
<div>{{ expensiveComputation(data) }}</div>

// GOOD — computed signal (memoized)
result = computed(() => expensiveComputation(this.data()));
<div>{{ result() }}</div>

// GOOD — pure pipe (memoized)
<div>{{ data | myPurePipe }}</div>
```

## Bundle Optimization

### Analyze Bundle Size
```bash
nx build my-app --stats-json
npx webpack-bundle-analyzer dist/my-app/browser/stats.json

# Or use source-map-explorer
npx source-map-explorer dist/my-app/browser/*.js
```

### Lazy Loading Routes (Critical)
```typescript
// Lazy load feature routes — biggest bundle win
{
  path: 'admin',
  loadChildren: () => import('./admin/admin.routes').then(m => m.ADMIN_ROUTES)
}

// Lazy load single components
{
  path: 'chart',
  loadComponent: () => import('./chart/chart.component').then(m => m.ChartComponent)
}
```

### Preloading Strategy
```typescript
// Preload routes after app loads (background)
provideRouter(routes, withPreloading(PreloadAllModules))

// Custom: preload only routes with `data: { preload: true }`
provideRouter(routes, withPreloading(SelectivePreloadingStrategy))
```

### Tree-Shaking Tips
- Import specific operators: `import { map } from 'rxjs/operators'` (not `rxjs`)
- Use `providedIn: 'root'` only for services actually used app-wide
- Avoid `CommonModule` in standalone components — import only what you need
  - `NgIf`, `NgFor`, `AsyncPipe` etc. individually

## Deferrable Views (`@defer`) — Angular 17+

Defer loading of non-critical UI:
```html
<!-- Load when viewport intersection -->
@defer (on viewport) {
  <app-heavy-chart [data]="chartData" />
} @loading (minimum 500ms) {
  <app-skeleton-chart />
} @error {
  <p>Failed to load chart</p>
} @placeholder {
  <div class="chart-placeholder"></div>
}

<!-- Load on interaction -->
@defer (on interaction(triggerEl)) {
  <app-comments />
}

<!-- Load when idle -->
@defer (on idle) {
  <app-analytics-widget />
}

<!-- Prefetch separately from display -->
@defer (on viewport; prefetch on idle) {
  <app-product-recommendations />
}
```

**Use `@defer` for:**
- Heavy components (charts, maps, editors)
- Below-the-fold content
- Content visible only after user interaction

## Image Optimization

```typescript
// NgOptimizedImage — automatic LCP optimization
import { NgOptimizedImage } from '@angular/common';

// In template:
<img ngSrc="/hero.jpg" width="1200" height="630" priority />  <!-- priority = LCP image -->
<img ngSrc="/thumbnail.jpg" width="200" height="200" />
```

`NgOptimizedImage` automatically:
- Sets `loading="lazy"` (except `priority` images)
- Adds `fetchpriority="high"` on priority images
- Prevents layout shift by requiring width/height
- Warns about missing `srcset`

## SSR / SSG (Angular Universal)

```bash
ng add @angular/ssr
# or in NX:
nx generate @nx/angular:setup-ssr --project=my-app
```

### SSR vs SSG vs CSR
| Mode | Use when |
|---|---|
| CSR (default) | Behind auth, highly dynamic, dashboard |
| SSR | Public pages with dynamic data |
| SSG / Prerender | Marketing pages, docs, blog (static content) |

### Prerendering Routes
```typescript
// angular.json / project.json
{
  "prerender": {
    "routesFile": "routes.txt"  // list of routes to prerender
  }
}
```

### Transfer State (Avoid Double Fetch)
```typescript
// Prevent re-fetching data that was already fetched on server
import { TransferState, makeStateKey } from '@angular/core';

const USERS_KEY = makeStateKey<User[]>('users');

export class UserService {
  private transferState = inject(TransferState);

  getAll(): Observable<User[]> {
    if (this.transferState.hasKey(USERS_KEY)) {
      const users = this.transferState.get(USERS_KEY, []);
      this.transferState.remove(USERS_KEY);
      return of(users);
    }
    return this.http.get<User[]>('/api/users').pipe(
      tap(users => this.transferState.set(USERS_KEY, users))
    );
  }
}
```

## Runtime Performance

### Avoid Memory Leaks
```typescript
// Auto-unsubscribe with takeUntilDestroyed
someObs$.pipe(takeUntilDestroyed()).subscribe(...);

// Or use async pipe — auto-unsubscribes
{{ users$ | async }}
```

### Virtualization for Long Lists
```typescript
// Angular CDK Virtual Scroll for lists with 100+ items
import { ScrollingModule } from '@angular/cdk/scrolling';

<cdk-virtual-scroll-viewport itemSize="50" style="height: 500px">
  <div *cdkVirtualFor="let item of items">{{ item.name }}</div>
</cdk-virtual-scroll-viewport>
```

### Web Workers for Heavy Computation
```bash
ng generate web-worker my-worker --project=my-app
```
```typescript
const worker = new Worker(new URL('./my-worker.worker', import.meta.url));
worker.postMessage(heavyData);
worker.onmessage = ({ data }) => console.log('Result:', data);
```

## Performance Audit Checklist
- [ ] All components use `OnPush` change detection?
- [ ] `track` used in all `@for` loops?
- [ ] No expensive computations in template expressions?
- [ ] Feature routes lazy-loaded?
- [ ] Bundle analyzed — no unexpected large dependencies?
- [ ] Heavy/below-fold components deferred with `@defer`?
- [ ] Images use `NgOptimizedImage` with `priority` on LCP image?
- [ ] Lists with 100+ items use virtual scrolling?
- [ ] Observables unsubscribed (async pipe or `takeUntilDestroyed`)?
- [ ] Core Web Vitals measured in production (Lighthouse CI)?
