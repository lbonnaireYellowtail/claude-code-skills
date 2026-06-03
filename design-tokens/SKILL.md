---
name: design-tokens
description: Expert knowledge for design tokens and design systems — CSS custom properties, token hierarchy (primitive/semantic/component), Angular theming, dark mode, and design-to-code workflows. Use when setting up a design system, implementing theming, adding dark mode, or establishing token architecture.
triggers:
  - design tokens
  - design system
  - CSS variables
  - theming
  - dark mode
  - CSS custom properties
  - token hierarchy
  - primitive tokens
  - semantic tokens
  - angular theme
---

# Design Tokens Expert

You are now loaded with deep knowledge about design tokens and design systems. Apply these patterns when helping the user implement theming and design systems in Angular.

## What Are Design Tokens?

Design tokens are **named variables that store design decisions** — colors, spacing, typography, shadows. They bridge design tools (Figma) and code, ensuring consistency.

```
Design Tool (Figma) → Tokens → CSS Variables → Components
```

## Token Hierarchy (3 Levels)

### Level 1: Primitive Tokens (raw values)
```css
/* No semantic meaning — just raw values */
:root {
  --color-blue-100: #dbeafe;
  --color-blue-500: #3b82f6;
  --color-blue-900: #1e3a5f;

  --spacing-1: 4px;
  --spacing-2: 8px;
  --spacing-4: 16px;
  --spacing-8: 32px;

  --font-size-sm: 0.875rem;
  --font-size-base: 1rem;
  --font-size-lg: 1.125rem;

  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 16px;
  --radius-full: 9999px;
}
```

### Level 2: Semantic Tokens (intent-based, reference primitives)
```css
/* Semantic = what it's FOR, not what it IS */
:root {
  /* Colors */
  --color-primary: var(--color-blue-500);
  --color-primary-hover: var(--color-blue-600);
  --color-surface: #ffffff;
  --color-surface-raised: var(--color-gray-50);
  --color-text-default: var(--color-gray-900);
  --color-text-muted: var(--color-gray-500);
  --color-border: var(--color-gray-200);
  --color-error: var(--color-red-500);
  --color-success: var(--color-green-500);

  /* Spacing */
  --spacing-xs: var(--spacing-1);
  --spacing-sm: var(--spacing-2);
  --spacing-md: var(--spacing-4);
  --spacing-lg: var(--spacing-8);

  /* Typography */
  --font-size-body: var(--font-size-base);
  --font-size-caption: var(--font-size-sm);
  --font-size-heading: var(--font-size-lg);
}
```

### Level 3: Component Tokens (component-specific overrides)
```css
/* Scoped to specific components */
:root {
  --button-border-radius: var(--radius-md);
  --button-padding-x: var(--spacing-md);
  --button-padding-y: var(--spacing-sm);
  --button-font-size: var(--font-size-body);

  --card-border-radius: var(--radius-lg);
  --card-padding: var(--spacing-lg);
  --card-shadow: 0 1px 3px rgba(0,0,0,0.1);

  --input-border-color: var(--color-border);
  --input-focus-ring: 0 0 0 2px var(--color-primary);
}
```

**Rule**: Components should reference semantic or component tokens, never primitive tokens directly.

## Dark Mode Implementation

### CSS `prefers-color-scheme` + Data Attribute
```css
/* Default: light theme (semantic tokens) */
:root {
  --color-surface: #ffffff;
  --color-text-default: #111827;
  --color-border: #e5e7eb;
  --color-primary: #3b82f6;
}

/* Dark theme — override semantic tokens */
[data-theme="dark"] {
  --color-surface: #111827;
  --color-text-default: #f9fafb;
  --color-border: #374151;
  --color-primary: #60a5fa; /* lighter for dark bg contrast */
}

/* System preference */
@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) {
    --color-surface: #111827;
    --color-text-default: #f9fafb;
    /* ... */
  }
}
```

### Angular Theme Service
```typescript
@Injectable({ providedIn: 'root' })
export class ThemeService {
  private theme = signal<'light' | 'dark' | 'system'>('system');

  activeTheme = computed(() => {
    const theme = this.theme();
    if (theme !== 'system') return theme;
    return window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light';
  });

  constructor() {
    effect(() => {
      document.documentElement.setAttribute('data-theme', this.activeTheme());
    });
  }

  setTheme(theme: 'light' | 'dark' | 'system') {
    this.theme.set(theme);
    localStorage.setItem('theme', theme);
  }
}
```

## Angular Material Theming (M3)

```typescript
// styles.scss
@use '@angular/material' as mat;

$theme: mat.define-theme((
  color: (
    theme-type: light,
    primary: mat.$azure-palette,
    tertiary: mat.$blue-palette,
  ),
  typography: (
    brand-family: 'Inter',
    bold-weight: 700,
  ),
  density: (scale: 0),
));

html {
  @include mat.all-component-themes($theme);
}

// Dark theme variant
html[data-theme="dark"] {
  $dark-theme: mat.define-theme((
    color: (theme-type: dark, primary: mat.$azure-palette)
  ));
  @include mat.all-component-colors($dark-theme);
}
```

## Token Files Structure (NX Monorepo)

```
libs/shared/design-tokens/
  src/
    lib/
      tokens/
        primitive.css        # raw values
        semantic.css         # intent-based
        component/
          button.css
          card.css
          input.css
      index.ts
  package.json
```

## Figma → Code Workflow

### Option 1: Manual (simple projects)
1. Design tokens defined in Figma as variables
2. Export manually to CSS custom properties
3. Check in to shared design-tokens library

### Option 2: Style Dictionary (automated)
```json
// tokens/colors.json
{
  "color": {
    "blue": {
      "500": { "value": "#3b82f6", "type": "color" }
    },
    "primary": { "value": "{color.blue.500}", "type": "color" }
  }
}
```

```javascript
// style-dictionary.config.js
module.exports = {
  source: ['tokens/**/*.json'],
  platforms: {
    css: {
      transformGroup: 'css',
      prefix: '',
      buildPath: 'src/styles/',
      files: [{ destination: 'tokens.css', format: 'css/variables' }]
    }
  }
};
```

### Option 3: Tokens Studio for Figma + Style Dictionary
Figma plugin exports tokens directly → Style Dictionary transforms → CSS/SCSS/TS

## Using Tokens in Angular Components

```scss
// button.component.scss
.btn {
  border-radius: var(--button-border-radius);
  padding: var(--button-padding-y) var(--button-padding-x);
  font-size: var(--button-font-size);
  background: var(--color-primary);
  color: white;

  &:hover {
    background: var(--color-primary-hover);
  }

  &--secondary {
    background: transparent;
    border: 1px solid var(--color-border);
    color: var(--color-text-default);
  }
}
```

## TypeScript Token Types (optional, for runtime use)
```typescript
// design-tokens.ts
export const SPACING = {
  xs: 'var(--spacing-xs)',
  sm: 'var(--spacing-sm)',
  md: 'var(--spacing-md)',
  lg: 'var(--spacing-lg)',
} as const;

// For dynamic styles (inline)
@Component({
  template: `<div [style.padding]="spacing.md">...</div>`
})
export class MyComponent {
  spacing = SPACING;
}
```

## Checklist
- [ ] Three-tier token hierarchy defined (primitive → semantic → component)?
- [ ] Components use semantic/component tokens (never raw values)?
- [ ] Dark mode works via `data-theme` attribute on `<html>`?
- [ ] System preference respected via `prefers-color-scheme`?
- [ ] Token theme preference persisted to `localStorage`?
- [ ] Design tokens in a shared NX library?
- [ ] Color contrast meets WCAG AA in both light and dark themes?
- [ ] Token naming is semantic (what it's FOR, not what it IS)?
