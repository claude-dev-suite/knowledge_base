# Tailwind v3 → Utilities Delta

## Not Available in Tailwind v3

- CSS-first configuration
- Native `@import 'tailwindcss'`
- Container queries (native)
- 3D transforms utilities
- `@starting-style` support
- Automatic content detection

## Major Changes: Configuration

### Setup

```css
/* Tailwind v4 - CSS-first */
@import "tailwindcss";

/* Optional: Add theme customization */
@theme {
  --color-primary: oklch(0.7 0.15 200);
  --font-sans: "Inter", sans-serif;
}


/* Tailwind v3 - PostCSS + Config */
/* tailwind.config.js required */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### Configuration

```javascript
// Tailwind v3 - JavaScript config
// tailwind.config.js
module.exports = {
  content: ['./src/**/*.{js,jsx,ts,tsx}'],
  theme: {
    extend: {
      colors: {
        primary: '#3490dc',
      },
      fontFamily: {
        sans: ['Inter', 'sans-serif'],
      },
    },
  },
  plugins: [],
}


// Tailwind v4 - CSS-only config
/* No JavaScript config needed */
@import "tailwindcss";

@theme {
  --color-primary: #3490dc;
  --font-family-sans: "Inter", sans-serif;
}
```

### Content Detection

```javascript
// Tailwind v3 - Manual content paths
module.exports = {
  content: [
    './src/**/*.{js,jsx,ts,tsx}',
    './public/index.html',
  ],
  // ...
}

// Tailwind v4 - Automatic detection
// No configuration needed!
// Tailwind automatically finds your source files
```

## Syntax Differences

### Custom Properties

```css
/* Tailwind v4 - Native CSS variables */
.btn {
  background-color: var(--color-primary);
  font-family: var(--font-family-sans);
}

/* Tailwind v3 - Use theme() function */
.btn {
  background-color: theme('colors.primary');
  font-family: theme('fontFamily.sans');
}
```

### Container Queries

```html
<!-- Tailwind v4 - Native support -->
<div class="@container">
  <div class="@md:flex @lg:grid">
    Container query responsive
  </div>
</div>

<!-- Tailwind v3 - Plugin required -->
<!-- npm install @tailwindcss/container-queries -->
<div class="@container">
  <div class="@md:flex @lg:grid">
    Container query responsive
  </div>
</div>

<!-- tailwind.config.js -->
plugins: [
  require('@tailwindcss/container-queries'),
]
```

### 3D Transforms

```html
<!-- Tailwind v4 - Built-in 3D utilities -->
<div class="rotate-x-45 rotate-y-30 translate-z-10 perspective-1000">
  3D transformed element
</div>

<!-- Tailwind v3 - Custom utilities needed -->
<style>
  .rotate-x-45 { transform: rotateX(45deg); }
  .rotate-y-30 { transform: rotateY(30deg); }
</style>
```

### Color Palette

```css
/* Tailwind v4 - P3 color gamut support */
@theme {
  --color-blue-500: oklch(0.6 0.2 250);
}

/* Tailwind v3 - RGB/Hex only */
// tailwind.config.js
colors: {
  blue: {
    500: '#3b82f6',
  },
}
```

## Still Current in Tailwind v3

- All utility classes (flex, grid, spacing, etc.)
- Responsive variants (sm:, md:, lg:, etc.)
- State variants (hover:, focus:, etc.)
- Dark mode support
- Arbitrary values `[value]`
- @apply directive
- Plugins system

## Recommendations for Tailwind v3 Users

1. **Keep using tailwind.config.js** - Still fully supported
2. **Use plugins** for container queries
3. **Custom utilities** for 3D transforms
4. **Plan migration** for simpler CSS-only setup

## Migration Path

1. Remove tailwind.config.js (optional)
2. Replace @tailwind directives with @import
3. Move theme config to @theme block
4. Remove content array (automatic detection)
5. Update color values to CSS variables

