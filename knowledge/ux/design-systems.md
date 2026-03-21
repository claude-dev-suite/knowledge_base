# Design Systems & Design Tokens

Design systems provide a shared language between designers and developers. Design tokens are the foundation: named entities that store visual design attributes.

---

## W3C Design Token Specification (v1.0 — October 2025)

The W3C Design Tokens Community Group published the first stable specification in October 2025. It defines a vendor-neutral JSON format for sharing design decisions across tools (Figma, Sketch, Style Dictionary, Tokens Studio, Penpot).

**Tools implementing the spec**: Figma, Sketch, Penpot, Style Dictionary v4, Tokens Studio, Terrazzo, Supernova, zeroheight, Knapsack.

### Token JSON format

```json
{
  "color": {
    "$type": "color",
    "brand": {
      "primary":       { "$value": "#2563eb", "$description": "Primary brand blue" },
      "primary-hover": { "$value": "#1d4ed8" },
      "secondary":     { "$value": "#7c3aed" },
      "destructive":   { "$value": "#dc2626" }
    },
    "neutral": {
      "50":  { "$value": "#f8fafc" },
      "100": { "$value": "#f1f5f9" },
      "200": { "$value": "#e2e8f0" },
      "500": { "$value": "#64748b" },
      "700": { "$value": "#334155" },
      "900": { "$value": "#0f172a" }
    }
  },
  "spacing": {
    "$type": "dimension",
    "1":  { "$value": "0.25rem" },
    "2":  { "$value": "0.5rem" },
    "4":  { "$value": "1rem" },
    "8":  { "$value": "2rem" },
    "12": { "$value": "3rem" },
    "16": { "$value": "4rem" }
  },
  "font-size": {
    "$type": "dimension",
    "sm":   { "$value": "0.875rem" },
    "base": { "$value": "1rem" },
    "lg":   { "$value": "1.125rem" },
    "2xl":  { "$value": "1.5rem" }
  },
  "border-radius": {
    "$type": "dimension",
    "sm":   { "$value": "0.25rem" },
    "md":   { "$value": "0.375rem" },
    "lg":   { "$value": "0.5rem" },
    "xl":   { "$value": "0.75rem" },
    "full": { "$value": "9999px" }
  },
  "duration": {
    "$type": "duration",
    "fast":   { "$value": "100ms" },
    "normal": { "$value": "200ms" },
    "slow":   { "$value": "300ms" }
  }
}
```

### Token hierarchy (3 layers — required for theming)

```
Layer 1: Primitives (raw values — do not use in components directly)
  color.brand.primary = #2563eb
  color.neutral.900   = #0f172a

Layer 2: Semantic tokens (meaning-based — use these in all components)
  color.interactive.default = {color.brand.primary}
  color.text.primary        = {color.neutral.900}
  color.surface.background  = {color.neutral.50}

Layer 3: Component tokens (optional, for complex components)
  button.bg.default  = {color.interactive.default}
  button.text.color  = {color.neutral.50}
  input.border.color = {color.neutral.200}
```

**Key rule**: Components consume semantic tokens, never primitives. This enables complete theme swapping by changing only the semantic layer.

---

## shadcn/ui CSS Variable Convention

shadcn uses HSL values **without** the `hsl()` wrapper, which enables Tailwind's opacity modifier syntax (`bg-primary/80`, `text-foreground/50`).

```css
:root {
  /* shadcn/ui standard palette — HSL channels only */
  --background:         0 0% 100%;
  --foreground:         222.2 47.4% 11.2%;

  --card:               0 0% 100%;
  --card-foreground:    222.2 47.4% 11.2%;

  --popover:            0 0% 100%;
  --popover-foreground: 222.2 47.4% 11.2%;

  --primary:            222.2 47.4% 11.2%;
  --primary-foreground: 210 40% 98%;

  --secondary:          210 40% 96.1%;
  --secondary-foreground: 222.2 47.4% 11.2%;

  --muted:              210 40% 96.1%;
  --muted-foreground:   215.4 16.3% 46.9%;

  --accent:             210 40% 96.1%;
  --accent-foreground:  222.2 47.4% 11.2%;

  --destructive:        0 84.2% 60.2%;
  --destructive-foreground: 210 40% 98%;

  --border:             214.3 31.8% 91.4%;
  --input:              214.3 31.8% 91.4%;
  --ring:               222.2 47.4% 11.2%;

  --radius:             0.5rem;
}

.dark {
  --background:         224 71% 4%;
  --foreground:         213 31% 91%;
  --primary:            210 40% 98%;
  --primary-foreground: 222.2 47.4% 1.2%;
  --secondary:          222.2 47.4% 11.2%;
  --secondary-foreground: 210 40% 98%;
  --muted:              223 47% 11%;
  --muted-foreground:   215.4 16.3% 56.9%;
  --accent:             216 34% 17%;
  --accent-foreground:  210 40% 98%;
  --destructive:        0 63% 31%;
  --destructive-foreground: 210 40% 98%;
  --border:             216 34% 17%;
  --input:              216 34% 17%;
  --ring:               216 34% 17%;
}
```

Usage in Tailwind: `bg-background`, `text-foreground`, `text-muted-foreground`, `border-border`, `bg-primary/80`.

---

## Dark Mode Implementation

### Flash prevention (critical — place in `<head>` before React hydration)

```html
<script>
  (function() {
    const theme = localStorage.getItem('theme') || 'system';
    const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
    const dark = theme === 'dark' || (theme === 'system' && prefersDark);
    document.documentElement.classList.toggle('dark', dark);
  })();
</script>
```

### Theme toggle implementation (React)

```tsx
type Theme = 'light' | 'dark' | 'system';

function useTheme() {
  const [theme, setTheme] = useState<Theme>(() =>
    (localStorage.getItem('theme') as Theme) || 'system'
  );

  useEffect(() => {
    const root = document.documentElement;
    const media = window.matchMedia('(prefers-color-scheme: dark)');

    function apply() {
      const isDark = theme === 'dark' || (theme === 'system' && media.matches);
      root.classList.toggle('dark', isDark);
      root.setAttribute('data-theme', isDark ? 'dark' : 'light');
    }

    apply();
    if (theme === 'system') {
      media.addEventListener('change', apply);
      return () => media.removeEventListener('change', apply);
    }
  }, [theme]);

  function setAndPersist(t: Theme) {
    localStorage.setItem('theme', t);
    setTheme(t);
  }

  return { theme, setTheme: setAndPersist };
}
```

---

## Atomic Design Methodology

Brad Frost's model (2013, still foundational in 2025) — adapted for modern component libraries:

| Level | Description | shadcn/Radix equivalent | Scale |
|-------|-------------|------------------------|-------|
| **Atoms** | Smallest reusable elements | `Button`, `Input`, `Badge`, `Avatar`, `Label` | ~50–100 |
| **Molecules** | 2–3 atoms forming a unit | `FormField`, `SearchBar`, `ComboBox`, `DatePicker` | ~30–60 |
| **Organisms** | Complex, standalone sections | `DataTable`, `Header`, `Sidebar`, `CommandPalette` | ~20–40 |
| **Templates** | Page-level layout without real data | `DashboardLayout`, `AuthLayout`, `SettingsLayout` | ~5–15 |
| **Pages** | Templates with real content | `DashboardPage`, `ProfilePage`, `SettingsPage` | Unlimited |

**Practical note**: Few teams follow Atomic Design strictly. Use it as a mental model for deciding abstraction level, not as a rigid rule.

---

## Style Dictionary Configuration

Style Dictionary transforms W3C design token JSON into platform-specific code (CSS variables, Sass, Swift, Kotlin, etc.).

```javascript
// style-dictionary.config.js
import StyleDictionary from 'style-dictionary';

export default {
  source: ['tokens/**/*.json'],
  platforms: {
    css: {
      transformGroup: 'css',
      prefix: 'ds',
      buildPath: 'dist/',
      files: [
        {
          destination: 'tokens.css',
          format: 'css/variables',
          options: { outputReferences: true },
        },
      ],
    },
    js: {
      transformGroup: 'js',
      buildPath: 'dist/',
      files: [
        { destination: 'tokens.esm.js', format: 'javascript/es6' },
        { destination: 'tokens.cjs.js', format: 'javascript/module' },
      ],
    },
    ios: {
      transformGroup: 'ios-swift',
      buildPath: 'dist/ios/',
      files: [{ destination: 'Tokens.swift', format: 'ios-swift/class.swift' }],
    },
  },
};
```

---

## Design System Efficiency Data

Research-backed benefits:
- Design tokens reduce design-to-dev handoff time by **~35%**
- Shared component libraries boost development efficiency by **30–50%**
- Single source of truth: updating one token propagates to all components and platforms instantly
- Reduces QA cycles: eliminates per-component inconsistency bugs
- In 2024, ~2/3 of Figma monthly active users were non-designers (developers, PMs, etc.) — design systems enabled this democratization

---

## References

- W3C Design Tokens Community Group — [Design Tokens Specification v1.0](https://www.w3.org/community/design-tokens/)
- Brad Frost — [Atomic Design Methodology](https://atomicdesign.bradfrost.com/)
- shadcn/ui — [Theming Documentation](https://ui.shadcn.com/docs/theming)
- Style Dictionary — [Official Documentation](https://styledictionary.com/)
- Figma Blog — [Schema 2025: Design Systems For A New Era](https://www.figma.com/blog/schema-2025-design-systems-recap/)
