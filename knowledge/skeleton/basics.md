# Skeleton UI

> Official Documentation: https://www.skeleton.dev/

## Overview

Skeleton is a UI component library for Svelte and SvelteKit, built on Tailwind CSS. It provides accessible, themeable components with a focus on developer experience and design consistency.

Key features:
- **Tailwind CSS integration** - Built on utility-first CSS
- **Design tokens** - Consistent theming with CSS variables
- **Accessible** - WAI-ARIA compliant components
- **Dark mode** - First-class dark mode support
- **Svelte 5 ready** - Works with runes and legacy syntax

---

## Installation

### New SvelteKit Project

```bash
# Create new SvelteKit project
npm create svelte@latest my-app
cd my-app

# Install Skeleton and Tailwind
npm install -D @skeletonlabs/skeleton @skeletonlabs/tw-plugin
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### Existing SvelteKit Project

```bash
npm install -D @skeletonlabs/skeleton @skeletonlabs/tw-plugin
```

### Tailwind Configuration

```javascript
// tailwind.config.js
import { join } from 'path';
import { skeleton } from '@skeletonlabs/tw-plugin';

/** @type {import('tailwindcss').Config} */
export default {
  darkMode: 'class',
  content: [
    './src/**/*.{html,js,svelte,ts}',
    join(require.resolve('@skeletonlabs/skeleton'), '../**/*.{html,js,svelte,ts}')
  ],
  theme: {
    extend: {}
  },
  plugins: [
    skeleton({
      themes: {
        preset: [
          { name: 'skeleton', enhancements: true },
          { name: 'modern', enhancements: true },
          { name: 'crimson', enhancements: true },
          { name: 'hamlindigo', enhancements: true },
          { name: 'gold-nouveau', enhancements: true },
          { name: 'vintage', enhancements: true },
          { name: 'rocket', enhancements: true },
          { name: 'sahara', enhancements: true },
          { name: 'seafoam', enhancements: true },
        ]
      }
    })
  ]
};
```

### App Layout Setup

```svelte
<!-- src/routes/+layout.svelte -->
<script>
  import '../app.postcss';
  import { AppShell, AppBar, AppRail, AppRailAnchor } from '@skeletonlabs/skeleton';

  let { children } = $props();
</script>

<AppShell>
  <svelte:fragment slot="header">
    <AppBar>
      <svelte:fragment slot="lead">
        <strong class="text-xl">My App</strong>
      </svelte:fragment>
    </AppBar>
  </svelte:fragment>

  <svelte:fragment slot="sidebarLeft">
    <AppRail>
      <AppRailAnchor href="/" selected={$page.url.pathname === '/'}>
        Home
      </AppRailAnchor>
      <AppRailAnchor href="/about">About</AppRailAnchor>
    </AppRail>
  </svelte:fragment>

  {@render children()}
</AppShell>
```

### Global Styles

```css
/* src/app.postcss */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

```html
<!-- src/app.html -->
<!DOCTYPE html>
<html lang="en" class="dark">
  <head>
    <meta charset="utf-8" />
    <link rel="icon" href="%sveltekit.assets%/favicon.png" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    %sveltekit.head%
  </head>
  <body data-theme="skeleton">
    <div style="display: contents">%sveltekit.body%</div>
  </body>
</html>
```

---

## Theming System

### Preset Themes

Available themes: `skeleton`, `modern`, `crimson`, `hamlindigo`, `gold-nouveau`, `vintage`, `rocket`, `sahara`, `seafoam`, `wintry`

```html
<!-- Apply theme via data attribute -->
<body data-theme="modern">
```

### Theme Switching

```svelte
<script lang="ts">
  import { setModeCurrent, getModeOsPrefers, setModeUserPrefers, setInitialClassState } from '@skeletonlabs/skeleton';

  let currentTheme = $state('skeleton');

  function setTheme(theme: string) {
    currentTheme = theme;
    document.body.setAttribute('data-theme', theme);
  }
</script>

<select bind:value={currentTheme} onchange={() => setTheme(currentTheme)}>
  <option value="skeleton">Skeleton</option>
  <option value="modern">Modern</option>
  <option value="crimson">Crimson</option>
  <option value="rocket">Rocket</option>
</select>
```

### Dark Mode Toggle

```svelte
<script lang="ts">
  import { LightSwitch } from '@skeletonlabs/skeleton';
</script>

<LightSwitch />
```

### Custom Theme

```javascript
// tailwind.config.js
skeleton({
  themes: {
    custom: [
      {
        name: 'my-theme',
        properties: {
          // Surface colors
          '--color-surface-50': '255 255 255',
          '--color-surface-100': '244 244 245',
          '--color-surface-200': '228 228 231',
          '--color-surface-300': '212 212 216',
          '--color-surface-400': '161 161 170',
          '--color-surface-500': '113 113 122',
          '--color-surface-600': '82 82 91',
          '--color-surface-700': '63 63 70',
          '--color-surface-800': '39 39 42',
          '--color-surface-900': '24 24 27',

          // Primary color
          '--color-primary-50': '240 253 250',
          '--color-primary-100': '204 251 241',
          '--color-primary-200': '153 246 228',
          '--color-primary-300': '94 234 212',
          '--color-primary-400': '45 212 191',
          '--color-primary-500': '20 184 166',
          '--color-primary-600': '13 148 136',
          '--color-primary-700': '15 118 110',
          '--color-primary-800': '17 94 89',
          '--color-primary-900': '19 78 74',

          // Additional tokens
          '--theme-font-family-base': 'Inter, sans-serif',
          '--theme-font-family-heading': 'Inter, sans-serif',
          '--theme-rounded-base': '8px',
          '--theme-rounded-container': '12px',
          '--theme-border-base': '1px',
        }
      }
    ]
  }
})
```

---

## Design Tokens

### Color Classes

```svelte
<!-- Background variants -->
<div class="bg-surface-50">Light surface</div>
<div class="bg-surface-500">Medium surface</div>
<div class="bg-surface-900">Dark surface</div>

<div class="bg-primary-500">Primary</div>
<div class="bg-secondary-500">Secondary</div>
<div class="bg-tertiary-500">Tertiary</div>
<div class="bg-success-500">Success</div>
<div class="bg-warning-500">Warning</div>
<div class="bg-error-500">Error</div>

<!-- Text colors -->
<p class="text-primary-500">Primary text</p>
<p class="text-surface-900 dark:text-surface-50">Adaptive text</p>

<!-- Gradient backgrounds -->
<div class="bg-gradient-to-br from-primary-500 to-secondary-500">Gradient</div>
<div class="variant-gradient-primary-secondary">Preset gradient</div>
```

### Variant Classes

```svelte
<!-- Filled variants -->
<button class="variant-filled">Default</button>
<button class="variant-filled-primary">Primary</button>
<button class="variant-filled-secondary">Secondary</button>
<button class="variant-filled-tertiary">Tertiary</button>
<button class="variant-filled-success">Success</button>
<button class="variant-filled-warning">Warning</button>
<button class="variant-filled-error">Error</button>
<button class="variant-filled-surface">Surface</button>

<!-- Ghost variants -->
<button class="variant-ghost">Ghost</button>
<button class="variant-ghost-primary">Ghost Primary</button>

<!-- Soft variants -->
<button class="variant-soft">Soft</button>
<button class="variant-soft-primary">Soft Primary</button>

<!-- Ringed variants -->
<button class="variant-ringed">Ringed</button>
<button class="variant-ringed-primary">Ringed Primary</button>
```

### Typography

```svelte
<!-- Headings -->
<h1 class="h1">Heading 1</h1>
<h2 class="h2">Heading 2</h2>
<h3 class="h3">Heading 3</h3>
<h4 class="h4">Heading 4</h4>

<!-- Text styles -->
<p class="text-sm">Small text</p>
<p class="text-base">Base text</p>
<p class="text-lg">Large text</p>
<p class="text-xl">Extra large text</p>

<!-- Font weights -->
<p class="font-light">Light</p>
<p class="font-normal">Normal</p>
<p class="font-medium">Medium</p>
<p class="font-semibold">Semibold</p>
<p class="font-bold">Bold</p>

<!-- Anchors -->
<a href="/" class="anchor">Styled link</a>
```

---

## Layout Components

### AppShell

```svelte
<script>
  import { AppShell, AppBar, AppRail } from '@skeletonlabs/skeleton';
</script>

<AppShell
  slotSidebarLeft="bg-surface-500/5 w-56"
  slotSidebarRight="bg-surface-500/5 w-56"
  slotPageHeader="bg-surface-500/5 p-4"
  slotPageFooter="bg-surface-500/5 p-4"
>
  <svelte:fragment slot="header">
    <AppBar>Header</AppBar>
  </svelte:fragment>

  <svelte:fragment slot="sidebarLeft">
    Sidebar Left
  </svelte:fragment>

  <svelte:fragment slot="sidebarRight">
    Sidebar Right
  </svelte:fragment>

  <svelte:fragment slot="pageHeader">
    Page Header
  </svelte:fragment>

  <!-- Main content (default slot) -->
  <div class="container mx-auto p-8">
    Main Content
  </div>

  <svelte:fragment slot="pageFooter">
    Page Footer
  </svelte:fragment>

  <svelte:fragment slot="footer">
    App Footer
  </svelte:fragment>
</AppShell>
```

### AppBar

```svelte
<script>
  import { AppBar } from '@skeletonlabs/skeleton';
</script>

<AppBar
  gridColumns="grid-cols-3"
  slotDefault="place-self-center"
  slotTrail="place-content-end"
>
  <svelte:fragment slot="lead">
    <button class="btn btn-sm variant-ghost-surface">Menu</button>
  </svelte:fragment>

  <strong class="text-xl uppercase">App Title</strong>

  <svelte:fragment slot="trail">
    <a href="/login" class="btn btn-sm variant-ghost-surface">Login</a>
  </svelte:fragment>
</AppBar>
```

### AppRail

```svelte
<script>
  import { AppRail, AppRailAnchor, AppRailTile } from '@skeletonlabs/skeleton';
  import { page } from '$app/stores';

  let currentTile = $state(0);
</script>

<AppRail>
  <svelte:fragment slot="lead">
    <AppRailAnchor href="/">Logo</AppRailAnchor>
  </svelte:fragment>

  <AppRailAnchor href="/" selected={$page.url.pathname === '/'}>
    <svelte:fragment slot="lead">🏠</svelte:fragment>
    <span>Home</span>
  </AppRailAnchor>

  <AppRailAnchor href="/about" selected={$page.url.pathname === '/about'}>
    <svelte:fragment slot="lead">ℹ️</svelte:fragment>
    <span>About</span>
  </AppRailAnchor>

  <hr class="opacity-30" />

  <AppRailTile bind:group={currentTile} name="tile-1" value={0}>
    <svelte:fragment slot="lead">📊</svelte:fragment>
    <span>Tile 1</span>
  </AppRailTile>

  <svelte:fragment slot="trail">
    <AppRailAnchor href="/settings">⚙️</AppRailAnchor>
  </svelte:fragment>
</AppRail>
```

---

## Form Elements

### Input Fields

```svelte
<script>
  let value = $state('');
  let email = $state('');
</script>

<!-- Basic input -->
<input
  type="text"
  class="input"
  placeholder="Enter text..."
  bind:value
/>

<!-- Variants -->
<input type="text" class="input variant-form-material" placeholder="Material style" />

<!-- With label -->
<label class="label">
  <span>Email Address</span>
  <input type="email" class="input" placeholder="you@example.com" bind:value={email} />
</label>

<!-- Textarea -->
<textarea class="textarea" rows="4" placeholder="Enter description..." />

<!-- Select -->
<select class="select">
  <option value="1">Option 1</option>
  <option value="2">Option 2</option>
  <option value="3">Option 3</option>
</select>

<!-- Checkbox -->
<label class="flex items-center space-x-2">
  <input class="checkbox" type="checkbox" />
  <span>Accept terms</span>
</label>

<!-- Radio -->
<label class="flex items-center space-x-2">
  <input class="radio" type="radio" name="group" value="1" />
  <span>Option 1</span>
</label>

<!-- Range slider -->
<input type="range" class="range" min="0" max="100" />
```

### File Input

```svelte
<script>
  import { FileDropzone, FileButton } from '@skeletonlabs/skeleton';

  let files = $state<FileList | undefined>();
</script>

<!-- File button -->
<FileButton name="files" bind:files accept="image/*">
  Upload Image
</FileButton>

<!-- Dropzone -->
<FileDropzone name="files" bind:files>
  <svelte:fragment slot="lead">📁</svelte:fragment>
  <svelte:fragment slot="message">Upload a file or drag and drop</svelte:fragment>
  <svelte:fragment slot="meta">PNG, JPG, or GIF up to 10MB</svelte:fragment>
</FileDropzone>
```

---

## Best Practices

### Component Composition

```svelte
<!-- Good: Compose components -->
<Card>
  <header class="card-header">
    <h3 class="h3">Title</h3>
  </header>
  <section class="p-4">
    Content here
  </section>
  <footer class="card-footer">
    <button class="btn variant-filled">Action</button>
  </footer>
</Card>

<!-- Good: Use design tokens consistently -->
<div class="bg-surface-100 dark:bg-surface-800 rounded-container-token p-4">
  <p class="text-surface-900 dark:text-surface-50">Adaptive content</p>
</div>
```

### Accessibility

```svelte
<!-- Always use semantic HTML -->
<nav aria-label="Main navigation">
  <AppRail>...</AppRail>
</nav>

<!-- Provide labels for interactive elements -->
<button class="btn variant-filled" aria-label="Submit form">
  Submit
</button>

<!-- Use focus-visible for keyboard navigation -->
<button class="btn variant-filled focus-visible:ring-2">
  Focusable
</button>
```

### Responsive Design

```svelte
<!-- Mobile-first approach -->
<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
  <Card>Item 1</Card>
  <Card>Item 2</Card>
  <Card>Item 3</Card>
</div>

<!-- Hide/show based on screen size -->
<AppRail class="hidden md:flex">...</AppRail>
<nav class="md:hidden">Mobile nav</nav>
```

---

## Related Topics

- [Skeleton Components](components.md) - Full component reference
- [Svelte SKILL](../../frontend-frameworks/svelte/SKILL.md) - Svelte patterns
- [Tailwind Utilities](../tailwind/utilities.md) - Tailwind CSS reference
