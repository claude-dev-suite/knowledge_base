# Tailwind CSS Customization

This comprehensive guide covers all aspects of customizing Tailwind CSS, from basic configuration to advanced plugin development and optimization strategies.

---

## Table of Contents

1. [tailwind.config.js Structure](#tailwindconfigjs-structure)
2. [Theme Configuration](#theme-configuration)
3. [Colors](#colors)
4. [Spacing Scale](#spacing-scale)
5. [Typography](#typography)
6. [Breakpoints](#breakpoints)
7. [Custom Utilities](#custom-utilities)
8. [Plugins](#plugins)
9. [Content Configuration](#content-configuration)
10. [Presets](#presets)
11. [Dark Mode Configuration](#dark-mode-configuration)
12. [Arbitrary Values](#arbitrary-values)
13. [Important Modifier](#important-modifier)
14. [Prefix Configuration](#prefix-configuration)
15. [Safelist and Blocklist](#safelist-and-blocklist)
16. [CSS Variables Integration](#css-variables-integration)
17. [Best Practices](#best-practices)

---

## tailwind.config.js Structure

The configuration file is the heart of Tailwind CSS customization. It controls every aspect of the framework.

### Basic Structure

```javascript
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
module.exports = {
  // Files to scan for class names
  content: [
    './src/**/*.{js,ts,jsx,tsx,mdx}',
    './components/**/*.{js,ts,jsx,tsx,mdx}',
    './app/**/*.{js,ts,jsx,tsx,mdx}',
  ],

  // Theme customization
  theme: {
    // Override default values
    screens: {},
    colors: {},
    spacing: {},

    // Extend default values
    extend: {
      colors: {},
      spacing: {},
      fontFamily: {},
    },
  },

  // Plugins to include
  plugins: [],

  // Additional options
  darkMode: 'class',
  prefix: '',
  important: false,
  separator: ':',
  presets: [],
  safelist: [],
  blocklist: [],
  corePlugins: {},
  future: {},
  experimental: {},
}
```

### ESM Configuration (tailwind.config.mjs)

```javascript
// tailwind.config.mjs
/** @type {import('tailwindcss').Config} */
export default {
  content: ['./src/**/*.{js,ts,jsx,tsx}'],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

### TypeScript Configuration (tailwind.config.ts)

```typescript
// tailwind.config.ts
import type { Config } from 'tailwindcss'

const config: Config = {
  content: ['./src/**/*.{js,ts,jsx,tsx,mdx}'],
  theme: {
    extend: {
      backgroundImage: {
        'gradient-radial': 'radial-gradient(var(--tw-gradient-stops))',
        'gradient-conic': 'conic-gradient(from 180deg at 50% 50%, var(--tw-gradient-stops))',
      },
    },
  },
  plugins: [],
}

export default config
```

### Configuration File Location

Tailwind looks for configuration files in this order:
1. `tailwind.config.js`
2. `tailwind.config.cjs`
3. `tailwind.config.mjs`
4. `tailwind.config.ts`

You can specify a custom path:

```bash
npx tailwindcss -c ./config/tailwind.config.js -i ./src/input.css -o ./dist/output.css
```

---

## Theme Configuration

### Extend vs Override

Understanding the difference between extending and overriding is crucial:

**Extending (Recommended)**: Adds to the default values

```javascript
module.exports = {
  theme: {
    extend: {
      colors: {
        brand: '#5865F2',  // Added alongside existing colors
      },
      spacing: {
        '128': '32rem',    // Added alongside existing spacing
      },
    },
  },
}
```

**Overriding**: Replaces the default values entirely

```javascript
module.exports = {
  theme: {
    // This REPLACES all default colors
    colors: {
      transparent: 'transparent',
      current: 'currentColor',
      white: '#ffffff',
      black: '#000000',
      brand: '#5865F2',
    },
    // This REPLACES all default spacing
    spacing: {
      '0': '0px',
      '1': '4px',
      '2': '8px',
      '4': '16px',
    },
  },
}
```

### Accessing Default Theme Values

```javascript
const defaultTheme = require('tailwindcss/defaultTheme')
const colors = require('tailwindcss/colors')

module.exports = {
  theme: {
    extend: {
      fontFamily: {
        sans: ['Inter var', ...defaultTheme.fontFamily.sans],
      },
      colors: {
        primary: colors.indigo,
        secondary: colors.cyan,
        neutral: colors.slate,
      },
    },
  },
}
```

### Referencing Other Theme Values

```javascript
module.exports = {
  theme: {
    extend: {
      // Using a function to reference other values
      colors: ({ theme }) => ({
        primary: theme('colors.blue.500'),
        secondary: theme('colors.gray.500'),
      }),

      // Reference in spacing
      height: ({ theme }) => ({
        screen: '100vh',
        'screen-navbar': `calc(100vh - ${theme('spacing.16')})`,
      }),

      // Reference in typography
      fontSize: {
        'body': ['1rem', { lineHeight: '1.5', letterSpacing: '-0.01em' }],
      },
    },
  },
}
```

### Disabling Core Plugins

```javascript
module.exports = {
  corePlugins: {
    float: false,           // Disable float utilities
    objectFit: false,       // Disable object-fit utilities
    objectPosition: false,  // Disable object-position utilities

    // Disable all core plugins (for fully custom builds)
    // preflight: false,
  },
}
```

---

## Colors

### Custom Color Palette

```javascript
module.exports = {
  theme: {
    extend: {
      colors: {
        // Single color value
        brand: '#5865F2',
        accent: '#FF6B6B',

        // Full color scale (11 shades recommended)
        primary: {
          50: '#eff6ff',
          100: '#dbeafe',
          200: '#bfdbfe',
          300: '#93c5fd',
          400: '#60a5fa',
          500: '#3b82f6',
          600: '#2563eb',
          700: '#1d4ed8',
          800: '#1e40af',
          900: '#1e3a8a',
          950: '#172554',
        },

        // Using DEFAULT for base value
        secondary: {
          light: '#67e8f9',
          DEFAULT: '#22d3ee',
          dark: '#06b6d4',
        },

        // Semantic colors
        success: {
          50: '#f0fdf4',
          500: '#22c55e',
          700: '#15803d',
        },
        warning: {
          50: '#fffbeb',
          500: '#f59e0b',
          700: '#b45309',
        },
        error: {
          50: '#fef2f2',
          500: '#ef4444',
          700: '#b91c1c',
        },
        info: {
          50: '#eff6ff',
          500: '#3b82f6',
          700: '#1d4ed8',
        },
      },
    },
  },
}
```

### Using Tailwind's Color Palette

```javascript
const colors = require('tailwindcss/colors')

module.exports = {
  theme: {
    extend: {
      colors: {
        // Use built-in colors
        primary: colors.violet,
        secondary: colors.pink,
        neutral: colors.slate,

        // Create aliases
        danger: colors.red,
        success: colors.emerald,
        warning: colors.amber,

        // Mix and match
        gray: colors.zinc,  // Replace gray with zinc
      },
    },
  },
}
```

### Color Opacity Modifier

When defining colors, Tailwind automatically generates opacity variants:

```html
<!-- Using built-in opacity modifiers -->
<div class="bg-primary-500/50">50% opacity</div>
<div class="bg-primary-500/75">75% opacity</div>
<div class="text-brand/90">90% opacity</div>
```

### CSS Variables for Dynamic Colors

```javascript
module.exports = {
  theme: {
    extend: {
      colors: {
        // HSL format (recommended for opacity support)
        background: 'hsl(var(--background) / <alpha-value>)',
        foreground: 'hsl(var(--foreground) / <alpha-value>)',

        // Without opacity support
        surface: 'var(--surface)',

        // Component-specific colors
        card: {
          DEFAULT: 'hsl(var(--card) / <alpha-value>)',
          foreground: 'hsl(var(--card-foreground) / <alpha-value>)',
        },
        muted: {
          DEFAULT: 'hsl(var(--muted) / <alpha-value>)',
          foreground: 'hsl(var(--muted-foreground) / <alpha-value>)',
        },
        accent: {
          DEFAULT: 'hsl(var(--accent) / <alpha-value>)',
          foreground: 'hsl(var(--accent-foreground) / <alpha-value>)',
        },
        destructive: {
          DEFAULT: 'hsl(var(--destructive) / <alpha-value>)',
          foreground: 'hsl(var(--destructive-foreground) / <alpha-value>)',
        },
        border: 'hsl(var(--border) / <alpha-value>)',
        input: 'hsl(var(--input) / <alpha-value>)',
        ring: 'hsl(var(--ring) / <alpha-value>)',
      },
    },
  },
}
```

---

## Spacing Scale

### Extending the Spacing Scale

```javascript
module.exports = {
  theme: {
    extend: {
      spacing: {
        // Numeric values
        '13': '3.25rem',
        '15': '3.75rem',
        '18': '4.5rem',
        '22': '5.5rem',
        '128': '32rem',
        '144': '36rem',

        // Semantic values
        'header': '64px',
        'sidebar': '280px',
        'footer': '120px',
        'navbar': '4rem',
        'content': '1200px',

        // Fractional values
        '1/2': '50%',
        '1/3': '33.333333%',
        '2/3': '66.666667%',
        '1/4': '25%',
        '3/4': '75%',

        // Viewport units
        'screen-1/2': '50vh',
        'screen-3/4': '75vh',

        // Calc values
        'full-minus-header': 'calc(100% - 64px)',
        'screen-minus-nav': 'calc(100vh - 4rem)',
      },
    },
  },
}
```

### Custom Width and Height

```javascript
module.exports = {
  theme: {
    extend: {
      // Width utilities
      width: {
        'prose': '65ch',
        'content': '1200px',
        'sidebar': '280px',
        'modal': '500px',
        'modal-lg': '800px',
      },

      // Height utilities
      height: {
        'header': '64px',
        'screen-nav': 'calc(100vh - 64px)',
        'screen-safe': 'calc(100vh - env(safe-area-inset-bottom))',
      },

      // Min/Max dimensions
      minWidth: {
        'xs': '20rem',
        'screen-sm': '640px',
      },
      maxWidth: {
        '8xl': '88rem',
        '9xl': '96rem',
        'prose-lg': '75ch',
        'content': '1200px',
      },
      minHeight: {
        'screen-nav': 'calc(100vh - 64px)',
      },
      maxHeight: {
        'modal': '90vh',
        'dropdown': '300px',
      },
    },
  },
}
```

### Negative Spacing

Tailwind automatically generates negative variants for spacing:

```html
<div class="-mt-4">Negative margin top</div>
<div class="-ml-2">Negative margin left</div>
<div class="-translate-y-1/2">Negative translate</div>
```

---

## Typography

### Font Family

```javascript
const defaultTheme = require('tailwindcss/defaultTheme')

module.exports = {
  theme: {
    extend: {
      fontFamily: {
        // System font stack with custom font
        sans: ['Inter var', ...defaultTheme.fontFamily.sans],

        // Monospace fonts
        mono: ['JetBrains Mono', 'Fira Code', ...defaultTheme.fontFamily.mono],

        // Display fonts
        display: ['Cal Sans', 'Inter var', 'sans-serif'],
        heading: ['Poppins', 'sans-serif'],

        // Body fonts
        body: ['Source Sans Pro', 'system-ui', 'sans-serif'],

        // Special purpose fonts
        serif: ['Merriweather', 'Georgia', 'serif'],
        handwriting: ['Caveat', 'cursive'],
      },
    },
  },
}
```

### Font Size

```javascript
module.exports = {
  theme: {
    extend: {
      fontSize: {
        // Simple values
        'xxs': '0.625rem',
        '2.5xl': '1.75rem',

        // With line height
        'body': ['1rem', '1.75'],
        'body-lg': ['1.125rem', '1.75'],

        // With object configuration
        'display-sm': ['2.25rem', {
          lineHeight: '2.5rem',
          letterSpacing: '-0.02em',
          fontWeight: '700',
        }],
        'display': ['3rem', {
          lineHeight: '1.1',
          letterSpacing: '-0.02em',
          fontWeight: '800',
        }],
        'display-lg': ['3.75rem', {
          lineHeight: '1',
          letterSpacing: '-0.025em',
          fontWeight: '800',
        }],
        'display-xl': ['4.5rem', {
          lineHeight: '1',
          letterSpacing: '-0.025em',
          fontWeight: '800',
        }],

        // Responsive typography
        'hero': ['clamp(2.5rem, 5vw, 4rem)', {
          lineHeight: '1.1',
          fontWeight: '800',
        }],
      },
    },
  },
}
```

### Font Weight

```javascript
module.exports = {
  theme: {
    extend: {
      fontWeight: {
        'hairline': '100',
        'extralight': '200',
        'light': '300',
        'normal': '400',
        'medium': '500',
        'semibold': '600',
        'bold': '700',
        'extrabold': '800',
        'black': '900',
      },
    },
  },
}
```

### Letter Spacing

```javascript
module.exports = {
  theme: {
    extend: {
      letterSpacing: {
        tightest: '-0.075em',
        tighter: '-0.05em',
        tight: '-0.025em',
        normal: '0',
        wide: '0.025em',
        wider: '0.05em',
        widest: '0.1em',
        'mega-wide': '0.25em',
      },
    },
  },
}
```

### Line Height

```javascript
module.exports = {
  theme: {
    extend: {
      lineHeight: {
        'extra-tight': '1.1',
        'tight': '1.25',
        'snug': '1.375',
        'normal': '1.5',
        'relaxed': '1.625',
        'loose': '2',
        'extra-loose': '2.5',
        '12': '3rem',
      },
    },
  },
}
```

---

## Breakpoints

### Custom Breakpoints

```javascript
module.exports = {
  theme: {
    // Override default screens
    screens: {
      'xs': '475px',
      'sm': '640px',
      'md': '768px',
      'lg': '1024px',
      'xl': '1280px',
      '2xl': '1536px',
      '3xl': '1920px',
    },

    // Or extend existing ones
    extend: {
      screens: {
        'xs': '475px',
        '3xl': '1920px',
        '4xl': '2560px',
      },
    },
  },
}
```

### Max-Width Breakpoints

```javascript
module.exports = {
  theme: {
    screens: {
      // Max-width breakpoints (mobile-first alternative)
      '2xl': { 'max': '1535px' },
      'xl': { 'max': '1279px' },
      'lg': { 'max': '1023px' },
      'md': { 'max': '767px' },
      'sm': { 'max': '639px' },
    },
  },
}
```

### Range Breakpoints

```javascript
module.exports = {
  theme: {
    screens: {
      'sm': '640px',
      'md': '768px',
      'lg': '1024px',
      'xl': '1280px',
      '2xl': '1536px',

      // Range breakpoint
      'md-only': { 'min': '768px', 'max': '1023px' },
      'tablet': { 'min': '768px', 'max': '1279px' },
    },
  },
}
```

### Raw Media Queries

```javascript
module.exports = {
  theme: {
    extend: {
      screens: {
        // Portrait orientation
        'portrait': { 'raw': '(orientation: portrait)' },

        // Landscape orientation
        'landscape': { 'raw': '(orientation: landscape)' },

        // Print styles
        'print': { 'raw': 'print' },

        // High DPI screens
        'retina': { 'raw': '(-webkit-min-device-pixel-ratio: 2), (min-resolution: 192dpi)' },

        // Reduced motion
        'motion-safe': { 'raw': '(prefers-reduced-motion: no-preference)' },
        'motion-reduce': { 'raw': '(prefers-reduced-motion: reduce)' },

        // Hover capability
        'hover-hover': { 'raw': '(hover: hover)' },
        'hover-none': { 'raw': '(hover: none)' },

        // Touch devices
        'touch': { 'raw': '(pointer: coarse)' },
        'stylus': { 'raw': '(pointer: fine)' },
      },
    },
  },
}
```

---

## Custom Utilities

### Adding Static Utilities

```javascript
const plugin = require('tailwindcss/plugin')

module.exports = {
  plugins: [
    plugin(function({ addUtilities }) {
      addUtilities({
        // Text utilities
        '.text-balance': {
          'text-wrap': 'balance',
        },
        '.text-pretty': {
          'text-wrap': 'pretty',
        },

        // Scrollbar utilities
        '.scrollbar-hide': {
          '-ms-overflow-style': 'none',
          'scrollbar-width': 'none',
          '&::-webkit-scrollbar': {
            display: 'none',
          },
        },
        '.scrollbar-thin': {
          'scrollbar-width': 'thin',
          '&::-webkit-scrollbar': {
            width: '8px',
            height: '8px',
          },
        },

        // Content visibility
        '.content-auto': {
          'content-visibility': 'auto',
        },
        '.content-hidden': {
          'content-visibility': 'hidden',
        },

        // Writing mode
        '.writing-vertical-rl': {
          'writing-mode': 'vertical-rl',
        },
        '.writing-vertical-lr': {
          'writing-mode': 'vertical-lr',
        },

        // Text orientation
        '.text-orientation-mixed': {
          'text-orientation': 'mixed',
        },
        '.text-orientation-upright': {
          'text-orientation': 'upright',
        },
      })
    }),
  ],
}
```

### Adding Dynamic Utilities

```javascript
const plugin = require('tailwindcss/plugin')

module.exports = {
  plugins: [
    plugin(function({ matchUtilities, theme }) {
      // Dynamic text-shadow utility
      matchUtilities(
        {
          'text-shadow': (value) => ({
            textShadow: value,
          }),
        },
        { values: theme('textShadow') }
      )

      // Dynamic animation delay
      matchUtilities(
        {
          'animation-delay': (value) => ({
            animationDelay: value,
          }),
        },
        { values: theme('transitionDelay') }
      )

      // Dynamic backdrop filters
      matchUtilities(
        {
          'backdrop-saturate': (value) => ({
            '--tw-backdrop-saturate': `saturate(${value})`,
            'backdrop-filter': 'var(--tw-backdrop-blur) var(--tw-backdrop-saturate)',
          }),
        },
        { values: { '0': '0', '50': '.5', '100': '1', '150': '1.5', '200': '2' } }
      )
    }),
  ],
  theme: {
    extend: {
      textShadow: {
        sm: '0 1px 2px var(--tw-shadow-color)',
        DEFAULT: '0 2px 4px var(--tw-shadow-color)',
        lg: '0 8px 16px var(--tw-shadow-color)',
        none: 'none',
      },
    },
  },
}
```

### Adding Variants

```javascript
const plugin = require('tailwindcss/plugin')

module.exports = {
  plugins: [
    plugin(function({ addVariant }) {
      // Simple variants
      addVariant('hocus', ['&:hover', '&:focus'])
      addVariant('group-hocus', [':merge(.group):hover &', ':merge(.group):focus &'])

      // Children selectors
      addVariant('children', '& > *')
      addVariant('children-hover', '& > *:hover')

      // Sibling selectors
      addVariant('sibling', '& + *')
      addVariant('siblings', '& ~ *')

      // State variants
      addVariant('not-first', '&:not(:first-child)')
      addVariant('not-last', '&:not(:last-child)')
      addVariant('not-disabled', '&:not(:disabled)')

      // Data attribute variants
      addVariant('data-active', '&[data-active="true"]')
      addVariant('data-state-open', '&[data-state="open"]')
      addVariant('data-state-closed', '&[data-state="closed"]')

      // ARIA variants
      addVariant('aria-expanded', '&[aria-expanded="true"]')
      addVariant('aria-selected', '&[aria-selected="true"]')
      addVariant('aria-current', '&[aria-current="page"]')

      // Custom media queries
      addVariant('supports-backdrop', '@supports (backdrop-filter: blur(0))')
      addVariant('supports-grid', '@supports (display: grid)')
    }),
  ],
}
```

---

## Plugins

### Official Plugins

```javascript
module.exports = {
  plugins: [
    // Forms - Reset form styles for easier customization
    require('@tailwindcss/forms'),

    // Typography - Beautiful typographic defaults
    require('@tailwindcss/typography'),

    // Aspect Ratio - Utilities for aspect ratio
    require('@tailwindcss/aspect-ratio'),

    // Container Queries - Size-based container styling
    require('@tailwindcss/container-queries'),
  ],
}
```

### Typography Plugin Configuration

```javascript
module.exports = {
  plugins: [
    require('@tailwindcss/typography'),
  ],
  theme: {
    extend: {
      typography: ({ theme }) => ({
        DEFAULT: {
          css: {
            '--tw-prose-body': theme('colors.gray[700]'),
            '--tw-prose-headings': theme('colors.gray[900]'),
            '--tw-prose-lead': theme('colors.gray[600]'),
            '--tw-prose-links': theme('colors.blue[600]'),
            '--tw-prose-bold': theme('colors.gray[900]'),
            '--tw-prose-counters': theme('colors.gray[500]'),
            '--tw-prose-bullets': theme('colors.gray[300]'),
            '--tw-prose-hr': theme('colors.gray[200]'),
            '--tw-prose-quotes': theme('colors.gray[900]'),
            '--tw-prose-quote-borders': theme('colors.gray[200]'),
            '--tw-prose-captions': theme('colors.gray[500]'),
            '--tw-prose-code': theme('colors.gray[900]'),
            '--tw-prose-pre-code': theme('colors.gray[200]'),
            '--tw-prose-pre-bg': theme('colors.gray[800]'),
            '--tw-prose-th-borders': theme('colors.gray[300]'),
            '--tw-prose-td-borders': theme('colors.gray[200]'),

            // Custom styles
            maxWidth: '75ch',
            a: {
              textDecoration: 'none',
              '&:hover': {
                textDecoration: 'underline',
              },
            },
            'code::before': {
              content: '""',
            },
            'code::after': {
              content: '""',
            },
          },
        },
        // Dark mode variant
        invert: {
          css: {
            '--tw-prose-body': theme('colors.gray[300]'),
            '--tw-prose-headings': theme('colors.white'),
            '--tw-prose-links': theme('colors.blue[400]'),
          },
        },
      }),
    },
  },
}
```

### Forms Plugin Configuration

```javascript
module.exports = {
  plugins: [
    require('@tailwindcss/forms')({
      strategy: 'class', // or 'base' for global styles
    }),
  ],
}
```

### Creating Custom Plugins

```javascript
const plugin = require('tailwindcss/plugin')

// Full-featured plugin example
const customPlugin = plugin(
  // Plugin function
  function({ addBase, addComponents, addUtilities, matchUtilities, theme }) {
    // Add base styles
    addBase({
      'h1': {
        fontSize: theme('fontSize.2xl'),
        fontWeight: theme('fontWeight.bold'),
      },
      'h2': {
        fontSize: theme('fontSize.xl'),
        fontWeight: theme('fontWeight.semibold'),
      },
    })

    // Add component styles
    addComponents({
      '.btn': {
        padding: `${theme('spacing.2')} ${theme('spacing.4')}`,
        borderRadius: theme('borderRadius.md'),
        fontWeight: theme('fontWeight.semibold'),
        transition: 'all 150ms ease',
        '&:focus': {
          outline: 'none',
          boxShadow: `0 0 0 3px ${theme('colors.blue.500')}40`,
        },
      },
      '.btn-primary': {
        backgroundColor: theme('colors.blue.500'),
        color: theme('colors.white'),
        '&:hover': {
          backgroundColor: theme('colors.blue.600'),
        },
      },
      '.btn-secondary': {
        backgroundColor: theme('colors.gray.200'),
        color: theme('colors.gray.800'),
        '&:hover': {
          backgroundColor: theme('colors.gray.300'),
        },
      },
      '.card': {
        backgroundColor: theme('colors.white'),
        borderRadius: theme('borderRadius.lg'),
        padding: theme('spacing.6'),
        boxShadow: theme('boxShadow.md'),
      },
    })

    // Add utility styles
    addUtilities({
      '.drag-none': {
        '-webkit-user-drag': 'none',
        'user-drag': 'none',
      },
      '.tap-highlight-transparent': {
        '-webkit-tap-highlight-color': 'transparent',
      },
    })

    // Add dynamic utilities
    matchUtilities(
      {
        'bg-gradient': (value) => ({
          backgroundImage: `linear-gradient(${value})`,
        }),
      },
      {
        values: {
          'to-t': 'to top',
          'to-tr': 'to top right',
          'to-r': 'to right',
          'to-br': 'to bottom right',
          'to-b': 'to bottom',
          'to-bl': 'to bottom left',
          'to-l': 'to left',
          'to-tl': 'to top left',
        },
      }
    )
  },
  // Plugin configuration (optional)
  {
    theme: {
      extend: {
        // Default theme extensions for this plugin
      },
    },
  }
)

module.exports = {
  plugins: [customPlugin],
}
```

---

## Content Configuration

### Basic Content Patterns

```javascript
module.exports = {
  content: [
    // HTML files
    './public/**/*.html',
    './src/**/*.html',

    // JavaScript/TypeScript files
    './src/**/*.{js,ts,jsx,tsx}',

    // Vue files
    './src/**/*.vue',

    // Svelte files
    './src/**/*.svelte',

    // MDX files
    './src/**/*.mdx',
    './content/**/*.mdx',

    // PHP files
    './resources/**/*.blade.php',

    // Template files
    './templates/**/*.twig',
  ],
}
```

### Framework-Specific Configurations

```javascript
// Next.js
module.exports = {
  content: [
    './pages/**/*.{js,ts,jsx,tsx,mdx}',
    './components/**/*.{js,ts,jsx,tsx,mdx}',
    './app/**/*.{js,ts,jsx,tsx,mdx}',
    './src/**/*.{js,ts,jsx,tsx,mdx}',
  ],
}

// Nuxt.js
module.exports = {
  content: [
    './components/**/*.{js,vue,ts}',
    './layouts/**/*.vue',
    './pages/**/*.vue',
    './plugins/**/*.{js,ts}',
    './nuxt.config.{js,ts}',
    './app.vue',
  ],
}

// Laravel
module.exports = {
  content: [
    './resources/**/*.blade.php',
    './resources/**/*.js',
    './resources/**/*.vue',
  ],
}
```

### Scanning Node Modules

```javascript
module.exports = {
  content: [
    './src/**/*.{js,ts,jsx,tsx}',

    // Include specific packages
    './node_modules/@heroicons/**/*.js',
    './node_modules/flowbite/**/*.js',
    './node_modules/preline/dist/*.js',

    // Include UI library components
    'node_modules/daisyui/dist/**/*.js',
  ],
}
```

### Raw Content

```javascript
module.exports = {
  content: [
    './src/**/*.{js,ts,jsx,tsx}',

    // Raw content for dynamic classes
    {
      raw: '<div class="bg-red-500 bg-blue-500 bg-green-500"></div>',
      extension: 'html',
    },

    // Function for complex scenarios
    {
      raw: () => {
        // Dynamic content generation
        const colors = ['red', 'blue', 'green', 'yellow']
        const classes = colors.map(c => `bg-${c}-500 text-${c}-700`)
        return `<div class="${classes.join(' ')}"></div>`
      },
      extension: 'html',
    },
  ],
}
```

### Transform Content

```javascript
module.exports = {
  content: {
    files: ['./src/**/*.{js,ts,jsx,tsx}'],

    // Transform content before scanning
    transform: {
      md: (content) => {
        // Transform markdown content
        return content.replace(/class="([^"]*)"/g, 'class="$1"')
      },
      jsx: (content) => {
        // Transform JSX content (e.g., handle clsx/classnames)
        return content
      },
    },

    // Extract classes from custom syntax
    extract: {
      md: (content) => {
        // Custom class extraction logic
        return content.match(/class="([^"]*)"/g) || []
      },
    },
  },
}
```

---

## Presets

### Creating a Preset

```javascript
// my-preset.js
module.exports = {
  theme: {
    extend: {
      colors: {
        brand: {
          50: '#eff6ff',
          500: '#3b82f6',
          900: '#1e3a8a',
        },
      },
      fontFamily: {
        sans: ['Inter var', 'system-ui', 'sans-serif'],
      },
      borderRadius: {
        DEFAULT: '0.5rem',
      },
    },
  },
  plugins: [
    require('@tailwindcss/forms'),
    require('@tailwindcss/typography'),
  ],
}
```

### Using Presets

```javascript
// tailwind.config.js
module.exports = {
  presets: [
    require('./my-preset.js'),
  ],
  // Overrides and extensions
  theme: {
    extend: {
      colors: {
        // This extends the preset's colors
        accent: '#ff6b6b',
      },
    },
  },
}
```

### Multiple Presets

```javascript
module.exports = {
  presets: [
    require('./brand-preset.js'),    // Brand colors and fonts
    require('./component-preset.js'), // Component styles
    require('./utility-preset.js'),   // Custom utilities
  ],
  theme: {
    extend: {
      // Project-specific overrides
    },
  },
}
```

### Disabling Default Preset

```javascript
module.exports = {
  // Start from scratch without Tailwind defaults
  presets: [],
  theme: {
    // Define everything from scratch
    colors: {
      transparent: 'transparent',
      current: 'currentColor',
      white: '#ffffff',
      black: '#000000',
    },
    spacing: {
      '0': '0',
      '1': '0.25rem',
      '2': '0.5rem',
      '4': '1rem',
      '8': '2rem',
    },
  },
}
```

---

## Dark Mode Configuration

### Class-Based Dark Mode

```javascript
module.exports = {
  darkMode: 'class',
  theme: {
    extend: {},
  },
}
```

```html
<!-- Toggle dark mode by adding/removing class on html element -->
<html class="dark">
  <body class="bg-white dark:bg-gray-900">
    <h1 class="text-black dark:text-white">Hello World</h1>
    <p class="text-gray-600 dark:text-gray-400">Content here</p>
    <button class="bg-blue-500 dark:bg-blue-600 hover:bg-blue-600 dark:hover:bg-blue-700">
      Click me
    </button>
  </body>
</html>
```

### Media-Based Dark Mode

```javascript
module.exports = {
  darkMode: 'media', // Uses prefers-color-scheme
  theme: {
    extend: {},
  },
}
```

### Selector-Based Dark Mode

```javascript
module.exports = {
  darkMode: ['selector', '[data-theme="dark"]'],
  theme: {
    extend: {},
  },
}
```

```html
<html data-theme="dark">
  <body class="bg-white dark:bg-gray-900">
    <!-- Content -->
  </body>
</html>
```

### Custom Dark Mode Selector

```javascript
module.exports = {
  darkMode: ['variant', [
    '@media (prefers-color-scheme: dark) { &:not(.light *) }',
    '&:is(.dark *)',
  ]],
}
```

### Dark Mode with CSS Variables

```css
/* globals.css */
:root {
  --background: 0 0% 100%;
  --foreground: 222.2 84% 4.9%;
  --card: 0 0% 100%;
  --card-foreground: 222.2 84% 4.9%;
  --primary: 221.2 83.2% 53.3%;
  --primary-foreground: 210 40% 98%;
  --secondary: 210 40% 96.1%;
  --secondary-foreground: 222.2 47.4% 11.2%;
  --muted: 210 40% 96.1%;
  --muted-foreground: 215.4 16.3% 46.9%;
  --accent: 210 40% 96.1%;
  --accent-foreground: 222.2 47.4% 11.2%;
  --destructive: 0 84.2% 60.2%;
  --destructive-foreground: 210 40% 98%;
  --border: 214.3 31.8% 91.4%;
  --input: 214.3 31.8% 91.4%;
  --ring: 221.2 83.2% 53.3%;
  --radius: 0.5rem;
}

.dark {
  --background: 222.2 84% 4.9%;
  --foreground: 210 40% 98%;
  --card: 222.2 84% 4.9%;
  --card-foreground: 210 40% 98%;
  --primary: 217.2 91.2% 59.8%;
  --primary-foreground: 222.2 47.4% 11.2%;
  --secondary: 217.2 32.6% 17.5%;
  --secondary-foreground: 210 40% 98%;
  --muted: 217.2 32.6% 17.5%;
  --muted-foreground: 215 20.2% 65.1%;
  --accent: 217.2 32.6% 17.5%;
  --accent-foreground: 210 40% 98%;
  --destructive: 0 62.8% 30.6%;
  --destructive-foreground: 210 40% 98%;
  --border: 217.2 32.6% 17.5%;
  --input: 217.2 32.6% 17.5%;
  --ring: 224.3 76.3% 48%;
}
```

---

## Arbitrary Values

### Basic Arbitrary Values

```html
<!-- Arbitrary colors -->
<div class="bg-[#1da1f2]">Custom hex color</div>
<div class="bg-[rgb(255,115,0)]">Custom RGB color</div>
<div class="bg-[hsl(200,100%,50%)]">Custom HSL color</div>

<!-- Arbitrary spacing -->
<div class="m-[13px]">Custom margin</div>
<div class="p-[clamp(1rem,5vw,3rem)]">Clamp padding</div>
<div class="w-[calc(100%-2rem)]">Calc width</div>

<!-- Arbitrary font sizes -->
<p class="text-[22px]">Custom font size</p>
<p class="text-[clamp(1rem,2vw,1.5rem)]">Responsive font</p>

<!-- Arbitrary line height -->
<p class="leading-[3rem]">Custom line height</p>

<!-- Arbitrary letter spacing -->
<p class="tracking-[0.2em]">Custom tracking</p>
```

### Arbitrary Properties

```html
<!-- Custom CSS properties -->
<div class="[mask-type:luminance]">Mask type</div>
<div class="[text-wrap:balance]">Text wrap</div>
<div class="[writing-mode:vertical-rl]">Vertical text</div>

<!-- Grid properties -->
<div class="[grid-template-columns:repeat(auto-fill,minmax(200px,1fr))]">
  Responsive grid
</div>

<!-- Custom animations -->
<div class="[animation:spin_2s_linear_infinite]">Custom animation</div>

<!-- Viewport units -->
<div class="h-[100dvh]">Dynamic viewport height</div>
<div class="w-[50svw]">Small viewport width</div>
```

### Arbitrary Variants

```html
<!-- Custom states -->
<div class="[&:nth-child(3)]:bg-red-500">Third child</div>
<div class="[&>*]:p-4">All children padding</div>
<div class="[&_p]:text-blue-500">Descendant paragraphs</div>

<!-- Attribute selectors -->
<div class="[&[data-active]]:bg-blue-500">Data active</div>
<input class="[&:not(:placeholder-shown)]:border-green-500">

<!-- Complex selectors -->
<div class="hover:[&:not(:active)]:scale-105">Hover but not active</div>
<div class="group-hover/item:[&:not(:first-child)]:hidden">Complex group</div>

<!-- Media queries -->
<div class="[@media(min-width:900px)]:grid-cols-3">Custom breakpoint</div>
<div class="[@supports(display:grid)]:grid">Supports query</div>

<!-- Container queries -->
<div class="@container">
  <div class="[@container(min-width:400px)]:flex">Container responsive</div>
</div>
```

### Handling Whitespace in Arbitrary Values

```html
<!-- Use underscores for spaces -->
<div class="bg-[url('/my_image.png')]">Background image</div>
<div class="content-['hello_world']">Content with space</div>
<div class="grid-cols-[1fr_500px_2fr]">Grid columns with spaces</div>

<!-- Or use var() for complex values -->
<div class="[--my-var:calc(100%_-_2rem)] w-[var(--my-var)]">Using CSS var</div>
```

---

## Important Modifier

### Global Important

```javascript
module.exports = {
  important: true,
}
```

This makes all utilities `!important`:

```css
/* Generated CSS */
.bg-red-500 {
  background-color: rgb(239 68 68) !important;
}
```

### Selector Strategy

```javascript
module.exports = {
  important: '#app',
}
```

This increases specificity instead of using `!important`:

```css
/* Generated CSS */
#app .bg-red-500 {
  background-color: rgb(239 68 68);
}
```

### Per-Utility Important

```html
<!-- Use ! prefix for individual utilities -->
<div class="!bg-red-500">Always red background</div>
<div class="!p-4">Always this padding</div>
<div class="hover:!text-blue-500">Important on hover</div>

<!-- Useful for overriding third-party styles -->
<div class="library-component !rounded-lg !shadow-none">
  Override library styles
</div>
```

---

## Prefix Configuration

### Adding a Prefix

```javascript
module.exports = {
  prefix: 'tw-',
}
```

Usage with prefix:

```html
<div class="tw-bg-red-500 tw-text-white tw-p-4">
  Prefixed classes
</div>

<!-- Negative values -->
<div class="tw--mt-4">Negative margin with prefix</div>

<!-- Responsive and state variants -->
<div class="tw-md:tw-flex tw-hover:tw-bg-blue-500">
  Variants with prefix
</div>
```

### Prefix with Arbitrary Values

```html
<!-- Arbitrary values still work -->
<div class="tw-bg-[#1da1f2] tw-p-[13px]">
  Arbitrary values with prefix
</div>
```

---

## Safelist and Blocklist

### Safelist Configuration

```javascript
module.exports = {
  safelist: [
    // Specific classes
    'bg-red-500',
    'bg-green-500',
    'bg-blue-500',
    'text-center',

    // Pattern matching
    {
      pattern: /bg-(red|green|blue)-(100|500|900)/,
    },

    // Pattern with variants
    {
      pattern: /bg-(red|green|blue)-(100|500|900)/,
      variants: ['hover', 'focus', 'dark', 'dark:hover'],
    },

    // Text colors
    {
      pattern: /text-(red|green|blue|yellow)-(400|500|600)/,
      variants: ['hover', 'group-hover'],
    },

    // Dynamic component colors
    {
      pattern: /^(bg|text|border)-(primary|secondary|success|warning|error)/,
      variants: ['hover', 'focus', 'active', 'disabled'],
    },

    // Size utilities
    {
      pattern: /^(w|h)-(1\/2|1\/3|2\/3|1\/4|3\/4)/,
      variants: ['sm', 'md', 'lg'],
    },
  ],
}
```

### Blocklist Configuration

```javascript
module.exports = {
  blocklist: [
    // Block specific utilities
    'container',
    'collapse',

    // Block patterns (using regex in safelist, strings in blocklist)
  ],
}
```

### Dynamic Safelist

```javascript
module.exports = {
  safelist: [
    // Generate safelist dynamically
    ...['red', 'green', 'blue', 'yellow'].flatMap(color =>
      ['100', '300', '500', '700', '900'].map(shade => `bg-${color}-${shade}`)
    ),

    // Status colors
    ...['success', 'warning', 'error', 'info'].flatMap(status => [
      `bg-${status}`,
      `text-${status}`,
      `border-${status}`,
    ]),
  ],
}
```

---

## CSS Variables Integration

### Defining CSS Variables in Tailwind

```javascript
const plugin = require('tailwindcss/plugin')

module.exports = {
  plugins: [
    plugin(function({ addBase, theme }) {
      addBase({
        ':root': {
          '--color-primary': theme('colors.blue.500'),
          '--color-secondary': theme('colors.gray.500'),
          '--spacing-base': theme('spacing.4'),
          '--border-radius': theme('borderRadius.lg'),
          '--font-family-sans': theme('fontFamily.sans').join(', '),
        },
      })
    }),
  ],
}
```

### Using CSS Variables in Theme

```javascript
module.exports = {
  theme: {
    extend: {
      colors: {
        // With opacity support
        primary: 'rgb(var(--color-primary) / <alpha-value>)',
        secondary: 'rgb(var(--color-secondary) / <alpha-value>)',

        // HSL format (recommended)
        background: 'hsl(var(--background) / <alpha-value>)',
        foreground: 'hsl(var(--foreground) / <alpha-value>)',

        // Without opacity support
        surface: 'var(--surface-color)',
      },
      spacing: {
        'dynamic': 'var(--spacing-dynamic)',
      },
      borderRadius: {
        'theme': 'var(--radius)',
      },
    },
  },
}
```

### Complete CSS Variables System

```css
/* globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    /* Colors - using HSL for opacity support */
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;

    --card: 0 0% 100%;
    --card-foreground: 222.2 84% 4.9%;

    --popover: 0 0% 100%;
    --popover-foreground: 222.2 84% 4.9%;

    --primary: 221.2 83.2% 53.3%;
    --primary-foreground: 210 40% 98%;

    --secondary: 210 40% 96.1%;
    --secondary-foreground: 222.2 47.4% 11.2%;

    --muted: 210 40% 96.1%;
    --muted-foreground: 215.4 16.3% 46.9%;

    --accent: 210 40% 96.1%;
    --accent-foreground: 222.2 47.4% 11.2%;

    --destructive: 0 84.2% 60.2%;
    --destructive-foreground: 210 40% 98%;

    --border: 214.3 31.8% 91.4%;
    --input: 214.3 31.8% 91.4%;
    --ring: 221.2 83.2% 53.3%;

    /* Spacing */
    --radius: 0.5rem;
    --header-height: 4rem;
    --sidebar-width: 16rem;

    /* Typography */
    --font-sans: 'Inter var', system-ui, sans-serif;
    --font-mono: 'JetBrains Mono', monospace;
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;

    --card: 222.2 84% 4.9%;
    --card-foreground: 210 40% 98%;

    --popover: 222.2 84% 4.9%;
    --popover-foreground: 210 40% 98%;

    --primary: 217.2 91.2% 59.8%;
    --primary-foreground: 222.2 47.4% 11.2%;

    --secondary: 217.2 32.6% 17.5%;
    --secondary-foreground: 210 40% 98%;

    --muted: 217.2 32.6% 17.5%;
    --muted-foreground: 215 20.2% 65.1%;

    --accent: 217.2 32.6% 17.5%;
    --accent-foreground: 210 40% 98%;

    --destructive: 0 62.8% 30.6%;
    --destructive-foreground: 210 40% 98%;

    --border: 217.2 32.6% 17.5%;
    --input: 217.2 32.6% 17.5%;
    --ring: 224.3 76.3% 48%;
  }
}
```

---

## Best Practices

### 1. Configuration Organization

```javascript
// tailwind.config.js - Well-organized configuration

/** @type {import('tailwindcss').Config} */
module.exports = {
  // Content sources
  content: [
    './src/**/*.{js,ts,jsx,tsx,mdx}',
    './components/**/*.{js,ts,jsx,tsx,mdx}',
  ],

  // Dark mode strategy
  darkMode: 'class',

  // Theme customization
  theme: {
    // Override defaults sparingly
    screens: {
      'xs': '475px',
      'sm': '640px',
      'md': '768px',
      'lg': '1024px',
      'xl': '1280px',
      '2xl': '1536px',
    },

    // Extend is preferred over override
    extend: {
      // Group related customizations
      colors: {
        // Brand colors
        brand: {
          50: '#eff6ff',
          500: '#3b82f6',
          900: '#1e3a8a',
        },
        // Semantic colors using CSS variables
        background: 'hsl(var(--background) / <alpha-value>)',
        foreground: 'hsl(var(--foreground) / <alpha-value>)',
      },

      fontFamily: {
        sans: ['Inter var', 'system-ui', 'sans-serif'],
        mono: ['JetBrains Mono', 'monospace'],
      },

      // Semantic spacing
      spacing: {
        'header': '4rem',
        'sidebar': '16rem',
      },

      // Animation
      animation: {
        'fade-in': 'fadeIn 0.5s ease-out',
      },
      keyframes: {
        fadeIn: {
          '0%': { opacity: '0' },
          '100%': { opacity: '1' },
        },
      },
    },
  },

  // Plugins
  plugins: [
    require('@tailwindcss/forms'),
    require('@tailwindcss/typography'),
  ],
}
```

### 2. Color Naming Conventions

```javascript
// Good: Semantic naming
colors: {
  primary: { /* shades */ },
  secondary: { /* shades */ },
  success: { /* shades */ },
  warning: { /* shades */ },
  error: { /* shades */ },
  info: { /* shades */ },
}

// Good: Component-based naming
colors: {
  background: 'var(--background)',
  foreground: 'var(--foreground)',
  card: { DEFAULT: 'var(--card)', foreground: 'var(--card-foreground)' },
  button: { DEFAULT: 'var(--button)', hover: 'var(--button-hover)' },
}

// Avoid: Generic color names for semantic purposes
// Bad: blue for primary, red for errors
```

### 3. Spacing Consistency

```javascript
// Use consistent spacing scale
spacing: {
  // Keep numeric values for flexibility
  '18': '4.5rem',
  '128': '32rem',

  // Use semantic names for fixed dimensions
  'header': '64px',
  'sidebar': '280px',
  'modal': '500px',

  // Avoid too many custom values
}
```

### 4. Component Extraction

```javascript
// Extract repeated patterns into components
const plugin = require('tailwindcss/plugin')

module.exports = {
  plugins: [
    plugin(function({ addComponents, theme }) {
      addComponents({
        // Button base
        '.btn': {
          display: 'inline-flex',
          alignItems: 'center',
          justifyContent: 'center',
          padding: `${theme('spacing.2')} ${theme('spacing.4')}`,
          borderRadius: theme('borderRadius.md'),
          fontWeight: theme('fontWeight.medium'),
          transition: 'all 150ms ease',
          '&:focus': {
            outline: 'none',
            ringWidth: '2px',
            ringColor: theme('colors.blue.500'),
          },
          '&:disabled': {
            opacity: '0.5',
            cursor: 'not-allowed',
          },
        },
        // Button variants
        '.btn-primary': {
          backgroundColor: theme('colors.blue.500'),
          color: theme('colors.white'),
          '&:hover:not(:disabled)': {
            backgroundColor: theme('colors.blue.600'),
          },
        },
        '.btn-secondary': {
          backgroundColor: theme('colors.gray.100'),
          color: theme('colors.gray.900'),
          '&:hover:not(:disabled)': {
            backgroundColor: theme('colors.gray.200'),
          },
        },
      })
    }),
  ],
}
```

### 5. Safelist Patterns for Dynamic Classes

```javascript
// Only safelist what you actually need
safelist: [
  // Dynamic status colors
  {
    pattern: /^(bg|text|border)-(success|warning|error|info)(-\d+)?$/,
    variants: ['hover', 'dark'],
  },

  // Dynamic grid columns (if used dynamically)
  {
    pattern: /^grid-cols-(1|2|3|4|6|12)$/,
    variants: ['sm', 'md', 'lg'],
  },
]

// Avoid safelisting entire color palettes unnecessarily
```

### 6. Performance Optimization

```javascript
module.exports = {
  // Only scan necessary files
  content: [
    './src/**/*.{js,ts,jsx,tsx}',
    // Don't include node_modules unless needed
  ],

  // Disable unused core plugins
  corePlugins: {
    float: false,        // If not using floats
    clear: false,        // If not using clears
    sepia: false,        // If not using sepia filter
    // ... other unused plugins
  },

  // Use presets for shared configurations
  presets: [
    require('./tailwind-preset.js'),
  ],
}
```

### 7. Dark Mode Best Practices

```html
<!-- Group light and dark classes together -->
<div class="
  bg-white text-gray-900
  dark:bg-gray-900 dark:text-white
">
  <h1 class="
    text-gray-800
    dark:text-gray-100
  ">
    Title
  </h1>
</div>

<!-- Use CSS variables for complex theming -->
<div class="bg-background text-foreground">
  Automatically adapts to theme
</div>
```

### 8. Responsive Design Patterns

```html
<!-- Mobile-first approach -->
<div class="
  flex flex-col gap-4
  md:flex-row md:gap-6
  lg:gap-8
">
  <!-- Content -->
</div>

<!-- Use container queries for component-level responsiveness -->
<div class="@container">
  <div class="
    grid grid-cols-1
    @md:grid-cols-2
    @lg:grid-cols-3
  ">
    <!-- Cards -->
  </div>
</div>
```

### 9. Maintainable Custom Utilities

```javascript
// Group related utilities in plugins
const customUtilities = plugin(function({ addUtilities }) {
  addUtilities({
    // Text utilities
    '.text-balance': { 'text-wrap': 'balance' },
    '.text-pretty': { 'text-wrap': 'pretty' },

    // Scrollbar utilities
    '.scrollbar-hide': {
      '-ms-overflow-style': 'none',
      'scrollbar-width': 'none',
      '&::-webkit-scrollbar': { display: 'none' },
    },

    // Layout utilities
    '.center-absolute': {
      position: 'absolute',
      top: '50%',
      left: '50%',
      transform: 'translate(-50%, -50%)',
    },
  })
})
```

### 10. Documentation and Team Standards

```javascript
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
module.exports = {
  /*
   * Content Configuration
   * ---------------------
   * Add all template files that use Tailwind classes.
   * Be specific to avoid scanning unnecessary files.
   */
  content: [
    './src/**/*.{js,ts,jsx,tsx,mdx}',
  ],

  theme: {
    extend: {
      /*
       * Brand Colors
       * ------------
       * Primary: Main brand color (blue)
       * Secondary: Supporting color (gray)
       * Accent: Highlight color (amber)
       */
      colors: {
        primary: { /* ... */ },
        secondary: { /* ... */ },
        accent: { /* ... */ },
      },

      /*
       * Typography
       * ----------
       * Sans: Inter for body text
       * Mono: JetBrains Mono for code
       * Display: Cal Sans for headings
       */
      fontFamily: {
        sans: ['Inter var', 'system-ui', 'sans-serif'],
        mono: ['JetBrains Mono', 'monospace'],
        display: ['Cal Sans', 'sans-serif'],
      },
    },
  },
}
```

---

## Additional Resources

- [Tailwind CSS Documentation](https://tailwindcss.com/docs)
- [Tailwind CSS GitHub](https://github.com/tailwindlabs/tailwindcss)
- [Tailwind UI](https://tailwindui.com) - Official component library
- [Heroicons](https://heroicons.com) - Icons by Tailwind Labs
- [Headless UI](https://headlessui.com) - Unstyled accessible components

---

## Version Compatibility

This documentation is based on Tailwind CSS v3.x. Some features may differ in earlier or later versions:

- **v3.0+**: JIT mode is default, arbitrary values support
- **v3.1+**: First-party TypeScript types
- **v3.2+**: Container queries plugin
- **v3.3+**: Extended color palette, logical properties
- **v3.4+**: Dynamic viewport units, subgrid support
