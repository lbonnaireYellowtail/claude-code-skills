---
name: a11y-reviewer
description: Expert knowledge for web accessibility — WCAG 2.2 compliance, ARIA patterns, keyboard navigation, screen reader support, color contrast, and Angular CDK a11y module. Use when reviewing or implementing accessible Angular components, auditing UI for accessibility issues, or implementing ARIA patterns.
triggers:
  - accessibility
  - a11y
  - WCAG
  - ARIA
  - screen reader
  - keyboard navigation
  - color contrast
  - accessible component
  - aria-label
  - focus management
---

# Accessibility Expert

You are now loaded with deep knowledge about web accessibility. Apply these patterns when helping the user build or review accessible Angular components.

## WCAG 2.2 Overview

**Four principles (POUR):**
- **Perceivable**: Content can be perceived by all users (text alternatives, captions, adaptable)
- **Operable**: UI is operable (keyboard accessible, no seizures, navigable)
- **Understandable**: Content is understandable (readable, predictable, input assistance)
- **Robust**: Content is robust enough for assistive technologies

**Conformance levels:**
- **A**: Minimum (must have)
- **AA**: Standard target (required by most regulations)
- **AAA**: Enhanced (nice to have)

## Common ARIA Patterns

### Roles
```html
<!-- Landmark roles (use semantic HTML when possible) -->
<nav aria-label="Main navigation">...</nav>
<main>...</main>
<aside aria-label="Related content">...</aside>

<!-- Widget roles -->
<div role="button" tabindex="0" (click)="..." (keydown.enter)="..." (keydown.space)="...">
  Click me
</div>

<!-- Composite widgets -->
<ul role="listbox" aria-label="Options">
  <li role="option" [attr.aria-selected]="isSelected">Option 1</li>
</ul>
```

### ARIA Attributes
```html
<!-- Labels -->
<button aria-label="Close dialog">×</button>
<button aria-labelledby="dialog-title">Submit</button>
<input aria-describedby="email-hint" />
<span id="email-hint">Enter your work email address</span>

<!-- State -->
<button [attr.aria-expanded]="isOpen" [attr.aria-controls]="menuId">Menu</button>
<div [id]="menuId" [attr.aria-hidden]="!isOpen">...</div>

<!-- Live regions -->
<div aria-live="polite">{{ statusMessage }}</div>         <!-- non-urgent updates -->
<div aria-live="assertive" role="alert">{{ error }}</div> <!-- urgent/errors -->
```

### ARIA Decision Tree
1. Is there a native HTML element that does what you need? → **Use it**
2. Can you modify a native element with ARIA? → **Do that**
3. Must you build a custom widget? → **Implement full ARIA pattern**

## Keyboard Navigation

### Required Keyboard Access
Every interactive element must be:
- **Focusable**: reachable via Tab/Shift+Tab
- **Operable**: triggerable via keyboard (Enter for buttons/links, Space for toggles)
- **Visible focus**: focus indicator always visible

### Focus Management Patterns

```typescript
// Programmatic focus after action
@ViewChild('dialogTitle') dialogTitle!: ElementRef;

openDialog() {
  this.isOpen = true;
  // Focus first element after render
  afterNextRender(() => {
    this.dialogTitle.nativeElement.focus();
  });
}
```

### Focus Trap (Dialogs/Modals)
Use Angular CDK's `FocusTrap`:
```typescript
import { FocusTrapFactory } from '@angular/cdk/a11y';

export class DialogComponent implements AfterViewInit {
  private focusTrap = inject(FocusTrapFactory);
  private trap!: FocusTrap;

  ngAfterViewInit() {
    this.trap = this.focusTrap.create(this.el.nativeElement);
    this.trap.focusInitialElement();
  }

  ngOnDestroy() { this.trap.destroy(); }
}
```

Or use CDK Dialog/Overlay which handles this automatically.

### Keyboard Event Handling
```typescript
// Handle both Enter and Space for custom buttons
@HostListener('keydown', ['$event'])
onKeydown(event: KeyboardEvent) {
  if (event.key === 'Enter' || event.key === ' ') {
    event.preventDefault();
    this.activate();
  }
}

// Arrow key navigation in lists/menus
onKeydown(event: KeyboardEvent, index: number) {
  switch (event.key) {
    case 'ArrowDown': this.focusItem(index + 1); break;
    case 'ArrowUp': this.focusItem(index - 1); break;
    case 'Home': this.focusItem(0); break;
    case 'End': this.focusItem(this.items.length - 1); break;
    case 'Escape': this.close(); break;
  }
}
```

## Color Contrast Requirements

| Text type | AA | AAA |
|---|---|---|
| Normal text (< 18pt) | 4.5:1 | 7:1 |
| Large text (≥ 18pt or 14pt bold) | 3:1 | 4.5:1 |
| UI components & graphics | 3:1 | — |

**Never rely on color alone** to convey information (e.g., error states must also use text/icon).

Check contrast: `color-contrast()` CSS function (future) or tools like WebAIM Contrast Checker.

## Angular CDK Accessibility Module

```typescript
import { A11yModule } from '@angular/cdk/a11y';

// In your standalone component:
imports: [A11yModule]
```

### Key CDK Tools

```typescript
// FocusMonitor — track focus origin
export class MyComponent {
  private focusMonitor = inject(FocusMonitor);

  ngAfterViewInit() {
    this.focusMonitor.monitor(this.el).subscribe(origin => {
      // origin: 'keyboard' | 'mouse' | 'touch' | 'program' | null
      this.showFocusRing = origin === 'keyboard';
    });
  }
}

// LiveAnnouncer — programmatic screen reader announcements
export class MyComponent {
  private announcer = inject(LiveAnnouncer);

  onSave() {
    this.save();
    this.announcer.announce('Changes saved successfully', 'polite');
  }
}

// cdkTrapFocus directive
<div cdkTrapFocus cdkTrapFocusAutoCapture="true">
  <!-- focus trapped here -->
</div>
```

## Common Angular Accessibility Patterns

### Form Error Messages
```html
<input
  id="email"
  type="email"
  [attr.aria-invalid]="emailControl.invalid && emailControl.touched"
  [attr.aria-describedby]="emailControl.invalid ? 'email-error' : null"
/>
<span id="email-error" role="alert" *ngIf="emailControl.invalid && emailControl.touched">
  {{ getEmailError() }}
</span>
```

### Loading States
```html
<button [disabled]="isLoading" [attr.aria-busy]="isLoading">
  <span *ngIf="!isLoading">Submit</span>
  <span *ngIf="isLoading">
    <span aria-hidden="true">...</span>
    <span class="sr-only">Loading...</span>
  </span>
</button>
```

### Skip Links
```html
<!-- First element in body -->
<a class="skip-link" href="#main-content">Skip to main content</a>
<main id="main-content">...</main>
```
```css
.skip-link {
  position: absolute;
  transform: translateY(-100%);
}
.skip-link:focus {
  transform: translateY(0);
}
```

### Visually Hidden (Screen Reader Only)
```css
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

## Testing Accessibility

### Automated Tools
- **axe-core**: `@axe-core/angular` or run in Cypress with `cypress-axe`
- **Lighthouse**: accessibility audit in DevTools / CI
- **ESLint a11y**: `eslint-plugin-angular-template-accessibility`

### Manual Testing
1. **Keyboard only**: Tab through entire page, verify all interactions work
2. **Screen reader**: VoiceOver (macOS/iOS), NVDA (Windows), TalkBack (Android)
3. **Zoom 200%**: no content lost or overlapping
4. **High contrast mode**: UI remains usable

### Cypress + axe
```typescript
import 'cypress-axe';

it('has no accessibility violations', () => {
  cy.visit('/my-page');
  cy.injectAxe();
  cy.checkA11y();
});
```

## Audit Checklist
- [ ] All images have meaningful `alt` text (or `alt=""` if decorative)?
- [ ] All form inputs have associated `<label>` elements?
- [ ] Color contrast meets AA (4.5:1 normal, 3:1 large text)?
- [ ] No information conveyed by color alone?
- [ ] All interactive elements keyboard accessible?
- [ ] Focus indicator always visible?
- [ ] Focus trapped in modals/dialogs?
- [ ] Dynamic content announced via `aria-live` or `role="alert"`?
- [ ] Skip navigation link present?
- [ ] Page has logical heading hierarchy (h1 → h2 → h3)?
- [ ] Custom widgets implement full ARIA pattern (role + state + keyboard)?
- [ ] Error messages associated with inputs via `aria-describedby`?
