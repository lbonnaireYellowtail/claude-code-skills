---
name: bff-designer
description: Expert knowledge for Backend for Frontend (BFF) pattern — API shaping, aggregation, auth handling, contract-first design, and caching strategies. Use when designing the BFF layer, shaping APIs for Angular clients, or deciding what belongs in BFF vs backend microservices.
triggers:
  - BFF
  - backend for frontend
  - API shaping
  - API gateway
  - aggregation layer
  - client-specific API
  - BFF pattern
  - API contract
---

# Backend for Frontend (BFF) Expert

You are now loaded with deep knowledge about the BFF pattern. Apply these principles when helping design or implement the BFF layer for Angular applications.

## What is a BFF?

A BFF is a **dedicated backend service per frontend client** that:
- Shapes API responses to exactly what the UI needs (no under/over-fetching)
- Aggregates data from multiple microservices into one response
- Handles auth/session on behalf of the frontend
- Acts as a security boundary — the frontend never talks to microservices directly

```
Angular App
    ↓
  BFF (Node.js / NestJS)    ← one BFF per client type
    ↓         ↓       ↓
Service A  Service B  Service C
```

## Core Responsibilities

### 1. API Shaping
Transform backend response to match UI requirements exactly:
```typescript
// Backend returns a generic user object with 40 fields
// BFF shapes it to what the dashboard component needs
function shapeUserForDashboard(user: BackendUser): DashboardUser {
  return {
    id: user.id,
    displayName: `${user.firstName} ${user.lastName}`,
    avatarUrl: user.profileImage?.url ?? DEFAULT_AVATAR,
    role: user.permissions.includes('admin') ? 'Admin' : 'User',
    lastActive: formatRelativeTime(user.lastActivityAt),
  };
}
```

**Why this matters**: Reduces frontend logic, decouples UI from backend schema changes.

### 2. Aggregation
Combine multiple service calls into one frontend request:
```typescript
// Angular makes ONE call: GET /bff/user-profile
// BFF makes THREE calls in parallel
async function getUserProfile(userId: string): Promise<UserProfileView> {
  const [user, orders, preferences] = await Promise.all([
    userService.getById(userId),
    orderService.getRecentOrders(userId, { limit: 5 }),
    preferenceService.get(userId),
  ]);
  return mergeUserProfile(user, orders, preferences);
}
```

**Benefit**: Fewer round trips, faster page loads, no waterfall requests in Angular.

### 3. Authentication & Session
The BFF is the auth boundary:
```
Browser → BFF (validates session cookie / JWT) → Microservices (machine-to-machine token)
```

```typescript
// BFF middleware: validate session, attach service token
app.use(async (req, res, next) => {
  const session = await validateSession(req.cookies['session']);
  if (!session) return res.status(401).json({ error: 'Unauthorized' });
  req.user = session.user;
  req.serviceToken = await getM2MToken(); // injected into downstream calls
  next();
});
```

**Pattern**: Frontend uses cookies (BFF manages them); BFF uses machine-to-machine tokens for services.

### 4. Data Transformation & Normalization
```typescript
// Normalize dates, currencies, enums from backend to frontend conventions
function normalizeOrder(backendOrder: Order): UIOrder {
  return {
    ...backendOrder,
    createdAt: new Date(backendOrder.created_at),    // snake_case → camelCase
    amount: Money.fromCents(backendOrder.amount_cents), // cents → display value
    status: ORDER_STATUS_MAP[backendOrder.status],    // enum mapping
  };
}
```

## Contract-First Design

Design the API contract from the frontend's perspective, not the backend's:

1. **Start with the UI component** — what data does it need?
2. **Define the response schema** first (OpenAPI / TypeScript types)
3. **Generate types** for both Angular and BFF from the same schema
4. **Build the BFF endpoint** to fulfill the contract
5. **Aggregate backend data** to fill the contract

```typescript
// Shared types package (in NX monorepo)
// libs/shared/api-types/src/lib/user-profile.ts
export interface UserProfileResponse {
  id: string;
  displayName: string;
  avatarUrl: string;
  recentOrders: OrderSummary[];
  preferences: UserPreferences;
}
```

## Response Caching

### Where to Cache in BFF
```typescript
// Cache at BFF level — not in Angular (too ephemeral)
// Use Redis or in-memory for BFF caching

// Per-request memoization (same request, multiple service calls)
const cache = new Map<string, Promise<any>>();

function cachedServiceCall<T>(key: string, fn: () => Promise<T>): Promise<T> {
  if (!cache.has(key)) {
    cache.set(key, fn());
  }
  return cache.get(key)!;
}

// Redis for shared cache (across BFF instances)
async function getCachedUserProfile(userId: string) {
  const cached = await redis.get(`user-profile:${userId}`);
  if (cached) return JSON.parse(cached);
  const profile = await fetchUserProfile(userId);
  await redis.setex(`user-profile:${userId}`, 300, JSON.stringify(profile)); // 5min TTL
  return profile;
}
```

### Cache Invalidation Signals
- User-triggered actions (update profile → invalidate user cache)
- Event-driven (subscribe to service events via message bus)
- Time-based TTL as fallback

## Error Handling at BFF

Map backend errors to client-friendly errors:
```typescript
// Don't leak internal errors to frontend
function mapServiceError(error: ServiceError): BFFError {
  switch (error.code) {
    case 'USER_NOT_FOUND': return { status: 404, message: 'User not found' };
    case 'PERMISSION_DENIED': return { status: 403, message: 'Access denied' };
    default:
      logger.error('Unexpected service error', error);
      return { status: 500, message: 'Something went wrong' };
  }
}
```

**Never expose**: stack traces, internal service names, database error messages.

## BFF in NX Monorepo

```
apps/
  my-angular-app/
  my-bff/              # NestJS BFF application
libs/
  shared/
    api-types/          # shared TypeScript types (Angular + BFF use this)
    api-client/         # Angular HttpClient service layer
```

Generate BFF app: `nx generate @nx/nest:app my-bff`

## Request Patterns

### Parallel Aggregation (default)
```typescript
const [a, b, c] = await Promise.all([serviceA(), serviceB(), serviceC()]);
```

### Sequential with Dependencies
```typescript
const user = await userService.get(id);
const orders = await orderService.getByUser(user.organizationId); // needs user first
```

### Graceful Degradation
```typescript
const [user, recommendations] = await Promise.allSettled([
  userService.get(id),
  recommendationService.get(id), // non-critical — don't fail if this fails
]);
return {
  user: user.status === 'fulfilled' ? user.value : null,
  recommendations: recommendations.status === 'fulfilled' ? recommendations.value : [],
};
```

## Decision: What Belongs in BFF vs Frontend

| Put in BFF | Put in Angular |
|---|---|
| Auth token management | UI state (open/closed, selected tab) |
| Multi-service aggregation | Derived display values from a single response |
| Sensitive data transformation | Client-side filtering/sorting of loaded data |
| Response caching | HTTP request deduplication (shareReplay) |
| Error mapping | Error display logic |
| Rate limiting | Optimistic updates |

## Checklist
- [ ] Frontend makes single BFF call per page (not multiple service calls)?
- [ ] BFF response shaped exactly for UI needs (no over-fetching)?
- [ ] Auth/session managed at BFF — not exposed to frontend?
- [ ] Shared API types package used by both Angular and BFF?
- [ ] Errors mapped to client-friendly messages (no internal details)?
- [ ] Parallel aggregation used where service calls are independent?
- [ ] Graceful degradation for non-critical service failures?
- [ ] Caching strategy defined for frequently read, infrequently changed data?
