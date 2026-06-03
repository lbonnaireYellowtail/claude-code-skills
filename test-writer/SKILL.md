---
name: test-writer
description: Expert knowledge for testing Angular applications — unit tests with TestBed, signal testing, service testing, component harnesses, E2E with Cypress/Playwright, and mocking strategies. Use when writing tests for Angular components, services, or setting up test infrastructure.
triggers:
  - write tests
  - unit test
  - angular testing
  - TestBed
  - component testing
  - cypress
  - playwright
  - mock service
  - test harness
  - e2e test
---

# Angular Testing Expert

You are now loaded with deep knowledge about testing Angular applications. Apply these patterns when writing or reviewing tests.

## Testing Pyramid for Angular

```
          /\
         /E2E\          ← Cypress/Playwright: user flows (few, slow)
        /------\
       /Integr. \       ← TestBed: component integration (moderate)
      /----------\
     / Unit Tests \     ← Services, pipes, utils (many, fast)
    /--------------\
```

## Unit Testing (Services, Pipes, Utils)

### Pure Logic — No Angular Needed
```typescript
// utils.spec.ts
describe('formatCurrency', () => {
  it('formats whole dollars', () => {
    expect(formatCurrency(1000)).toBe('$1,000.00');
  });

  it('handles zero', () => {
    expect(formatCurrency(0)).toBe('$0.00');
  });
});
```

### Service Testing with TestBed
```typescript
import { TestBed } from '@angular/core/testing';
import { provideHttpClientTesting, HttpTestingController } from '@angular/common/http/testing';

describe('UserService', () => {
  let service: UserService;
  let http: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        UserService,
        provideHttpClient(),
        provideHttpClientTesting(),
      ]
    });
    service = TestBed.inject(UserService);
    http = TestBed.inject(HttpTestingController);
  });

  afterEach(() => http.verify()); // ensure no pending requests

  it('fetches users', () => {
    service.getAll().subscribe(users => {
      expect(users).toHaveLength(2);
    });

    const req = http.expectOne('/api/users');
    expect(req.request.method).toBe('GET');
    req.flush([{ id: '1' }, { id: '2' }]);
  });
});
```

## Component Testing with TestBed

### Basic Component Test
```typescript
import { ComponentFixture, TestBed } from '@angular/core/testing';

describe('UserCardComponent', () => {
  let fixture: ComponentFixture<UserCardComponent>;
  let component: UserCardComponent;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [UserCardComponent], // standalone component
    }).compileComponents();

    fixture = TestBed.createComponent(UserCardComponent);
    component = fixture.componentInstance;
  });

  it('displays user name', () => {
    fixture.componentRef.setInput('user', { name: 'Alice', email: 'alice@example.com' });
    fixture.detectChanges();

    const nameEl = fixture.nativeElement.querySelector('[data-testid="user-name"]');
    expect(nameEl.textContent.trim()).toBe('Alice');
  });

  it('emits select event on click', () => {
    fixture.componentRef.setInput('user', mockUser);
    fixture.detectChanges();

    const spy = jest.spyOn(component.select, 'emit');
    fixture.nativeElement.querySelector('[data-testid="user-card"]').click();
    expect(spy).toHaveBeenCalledWith(mockUser.id);
  });
});
```

### Testing Signals
```typescript
it('updates computed count when items change', () => {
  const component = TestBed.createComponent(CartComponent).componentInstance;

  // set signal value
  component.items.set([{ id: '1', price: 10 }]);
  expect(component.count()).toBe(1);
  expect(component.total()).toBe(10);

  component.items.update(items => [...items, { id: '2', price: 20 }]);
  expect(component.count()).toBe(2);
  expect(component.total()).toBe(30);
});
```

### Mocking Dependencies
```typescript
// Mock a service
const mockUserService = {
  getAll: jest.fn().mockReturnValue(of([mockUser])),
  getById: jest.fn().mockReturnValue(of(mockUser)),
};

await TestBed.configureTestingModule({
  imports: [UserDashboardComponent],
  providers: [
    { provide: UserService, useValue: mockUserService }
  ]
}).compileComponents();
```

### Testing Async Operations
```typescript
it('shows loading state during fetch', fakeAsync(() => {
  const subject = new Subject<User[]>();
  mockService.getAll.mockReturnValue(subject.asObservable());

  fixture.detectChanges();
  expect(fixture.nativeElement.querySelector('[data-testid="loading"]')).toBeTruthy();

  subject.next([mockUser]);
  subject.complete();
  tick();
  fixture.detectChanges();

  expect(fixture.nativeElement.querySelector('[data-testid="loading"]')).toBeFalsy();
}));

// Or with waitForAsync
it('loads data', waitForAsync(() => {
  fixture.detectChanges();
  fixture.whenStable().then(() => {
    fixture.detectChanges();
    expect(component.users().length).toBe(1);
  });
}));
```

## Component Test Harnesses (CDK)

Harnesses provide a stable abstraction over DOM:
```typescript
import { MatButtonHarness } from '@angular/material/button/testing';
import { HarnessLoader } from '@angular/cdk/testing';
import { TestbedHarnessEnvironment } from '@angular/cdk/testing/testbed';

describe('MyComponent', () => {
  let loader: HarnessLoader;

  beforeEach(async () => {
    // ... setup
    loader = TestbedHarnessEnvironment.loader(fixture);
  });

  it('can click submit button', async () => {
    const button = await loader.getHarness(MatButtonHarness.with({ text: 'Submit' }));
    await button.click();
    expect(mockService.submit).toHaveBeenCalled();
  });
});
```

## Testing Best Practices

### Use `data-testid` Attributes
```html
<!-- Stable selectors that survive CSS/DOM restructuring -->
<div data-testid="user-card">
<button data-testid="submit-btn">Submit</button>
```

```typescript
// In tests
fixture.nativeElement.querySelector('[data-testid="user-card"]')
```

Never select by: CSS class (UI concern), tag name (too generic), or text content (localization).

### Test Behavior, Not Implementation
```typescript
// BAD — testing implementation
expect(component.isLoading).toBe(true);

// GOOD — testing behavior
expect(fixture.nativeElement.querySelector('[data-testid="spinner"]')).toBeTruthy();
```

### AAA Pattern
```typescript
it('shows error when email is invalid', () => {
  // Arrange
  const emailInput = fixture.nativeElement.querySelector('[data-testid="email"]');

  // Act
  emailInput.value = 'not-an-email';
  emailInput.dispatchEvent(new Event('input'));
  fixture.detectChanges();

  // Assert
  const error = fixture.nativeElement.querySelector('[data-testid="email-error"]');
  expect(error.textContent).toContain('Invalid email');
});
```

## E2E Testing with Cypress

### Setup
```bash
nx generate @nx/cypress:configuration --project=my-app
```

### Basic Test
```typescript
// cypress/e2e/login.cy.ts
describe('Login Flow', () => {
  beforeEach(() => {
    cy.visit('/login');
  });

  it('logs in with valid credentials', () => {
    cy.get('[data-testid="email"]').type('user@example.com');
    cy.get('[data-testid="password"]').type('password123');
    cy.get('[data-testid="submit"]').click();

    cy.url().should('include', '/dashboard');
    cy.get('[data-testid="welcome-message"]').should('contain', 'Welcome');
  });

  it('shows error for invalid credentials', () => {
    cy.get('[data-testid="email"]').type('wrong@example.com');
    cy.get('[data-testid="password"]').type('wrongpass');
    cy.get('[data-testid="submit"]').click();

    cy.get('[data-testid="error-message"]').should('be.visible');
  });
});
```

### API Interception
```typescript
// Stub API calls in Cypress
cy.intercept('GET', '/api/users', { fixture: 'users.json' }).as('getUsers');
cy.visit('/users');
cy.wait('@getUsers');
cy.get('[data-testid="user-list"]').should('have.length', 3);

// Or spy on real calls
cy.intercept('POST', '/api/orders').as('createOrder');
cy.get('[data-testid="submit"]').click();
cy.wait('@createOrder').its('request.body').should('include', { productId: '123' });
```

### Custom Commands
```typescript
// cypress/support/commands.ts
Cypress.Commands.add('login', (email: string, password: string) => {
  cy.request('POST', '/api/auth/login', { email, password })
    .then(({ body }) => {
      window.localStorage.setItem('token', body.token);
    });
});

// Usage
cy.login('user@example.com', 'password123');
cy.visit('/dashboard');
```

## Testing Checklist
- [ ] Pure functions tested without Angular overhead?
- [ ] Services tested with `HttpTestingController` for HTTP?
- [ ] Components tested via behavior (DOM), not internal state?
- [ ] Signal inputs set via `fixture.componentRef.setInput()`?
- [ ] `data-testid` attributes used for stable selectors?
- [ ] Async tests use `fakeAsync`/`tick` or `waitForAsync`?
- [ ] `afterEach(() => http.verify())` for HTTP service tests?
- [ ] E2E covers critical user journeys?
- [ ] API calls stubbed in E2E for deterministic tests?
