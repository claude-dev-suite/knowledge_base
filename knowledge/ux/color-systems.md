# Color Systems in UX

Color is functional infrastructure, not just aesthetics. It communicates meaning, builds trust, signals status, and guides attention — all before a user reads a single word.

---

## Color Psychology by Domain

| Color | Psychological effect | Primary domains | Avoid when |
|-------|---------------------|----------------|------------|
| **Blue** | Trust, security, reliability, calm | Fintech, SaaS, healthcare, B2B | Warmth/excitement needed |
| **Green** | Growth, success, nature, health | E-commerce ("add to cart"), wellness, finance (positive metrics) | Danger/urgency |
| **Red** | Urgency, energy, danger, passion | Error states, alerts, limited offers, destructive actions | Primary brand (perceived aggression) |
| **Orange** | Warmth, creativity, optimism, enthusiasm | Consumer, food, lifestyle, "Buy now" CTAs | Professional/enterprise contexts |
| **Yellow** | Optimism, caution, attention | Warning states, highlights, construction | Body text (poor contrast) |
| **Purple** | Premium, wisdom, creativity, mystery | Luxury, creative tools, DeFi/crypto | Mass-market or budget positioning |
| **Earthy tones** | Calm, groundedness, approachability | Consumer lifestyle, sustainability brands | High-tech, precision industries |
| **Black** | Sophistication, authority, premium | Luxury, fashion, premium tech | Approachability-first brands |
| **White** | Cleanliness, simplicity, space | Healthcare, minimal UI, premium | Content-heavy pages (eye strain) |

---

## WCAG Contrast Requirements (Must meet AA minimum)

| Text type | Minimum ratio (AA) | Enhanced (AAA) |
|-----------|-------------------|----------------|
| Normal text (< 18px regular, or < 14px bold) | **4.5:1** | 7:1 |
| Large text (≥ 18px regular, or ≥ 14px bold) | **3:1** | 4.5:1 |
| UI components (borders, icons, inputs) | **3:1** | — |
| Decorative elements | No requirement | — |
| Logo/brand | No requirement | — |

### Contrast testing tools
- Browser DevTools (Accessibility panel)
- [WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/)
- Figma plugins: Contrast, A11y — Color Contrast Checker

### Color blindness considerations

Affect ~8% of males, ~0.5% of females:
- **Deuteranopia** (most common): Red-green confusion
- **Protanopia**: Red appears very dark, confusion with green/yellow
- **Tritanopia** (rare): Blue-yellow confusion

**Never use color as the only indicator** of status, error, or required action. Always pair with:
- An icon or symbol
- Text label
- Pattern or shape difference

```tsx
{/* Bad: color-only error indicator */}
<input className={error ? "border-red-500" : "border-gray-300"} />

{/* Good: color + icon + text */}
<input
  className={error ? "border-destructive" : "border-input"}
  aria-invalid={!!error}
  aria-describedby={error ? "field-error" : undefined}
/>
{error && (
  <p id="field-error" className="flex items-center gap-1 text-sm text-destructive">
    <AlertCircleIcon className="h-4 w-4" aria-hidden="true" />
    {error}
  </p>
)}
```

---

## Design Token Color Architecture

### 3-layer token system

```css
/* ──────────────────────────────────────────────── */
/* LAYER 1: Primitive tokens (never use directly in components) */
/* ──────────────────────────────────────────────── */
:root {
  /* Blue scale */
  --color-blue-50:  #eff6ff;
  --color-blue-100: #dbeafe;
  --color-blue-400: #60a5fa;
  --color-blue-500: #3b82f6;
  --color-blue-600: #2563eb;
  --color-blue-700: #1d4ed8;

  /* Red scale */
  --color-red-400:  #f87171;
  --color-red-500:  #ef4444;
  --color-red-600:  #dc2626;

  /* Neutral scale */
  --color-neutral-50:  #f8fafc;
  --color-neutral-100: #f1f5f9;
  --color-neutral-200: #e2e8f0;
  --color-neutral-400: #94a3b8;
  --color-neutral-500: #64748b;
  --color-neutral-700: #334155;
  --color-neutral-800: #1e293b;
  --color-neutral-900: #0f172a;
}

/* ──────────────────────────────────────────────── */
/* LAYER 2: Semantic tokens (use these in all components) */
/* ──────────────────────────────────────────────── */
:root {
  /* Interactive */
  --color-interactive-default:  var(--color-blue-600);
  --color-interactive-hover:    var(--color-blue-700);
  --color-interactive-disabled: var(--color-neutral-400);

  /* Status */
  --color-success:   #16a34a;
  --color-warning:   #d97706;
  --color-error:     var(--color-red-600);
  --color-info:      var(--color-blue-600);

  /* Text */
  --color-text-primary:   var(--color-neutral-900);
  --color-text-secondary: var(--color-neutral-500);
  --color-text-disabled:  var(--color-neutral-400);
  --color-text-inverse:   var(--color-neutral-50);

  /* Surface */
  --color-surface-base:     var(--color-neutral-50);
  --color-surface-elevated: #ffffff;
  --color-surface-overlay:  var(--color-neutral-100);

  /* Border */
  --color-border-default: var(--color-neutral-200);
  --color-border-strong:  var(--color-neutral-400);
}

/* Dark mode: remap semantic layer only */
[data-theme="dark"] {
  --color-text-primary:     var(--color-neutral-50);
  --color-text-secondary:   var(--color-neutral-400);
  --color-surface-base:     var(--color-neutral-900);
  --color-surface-elevated: var(--color-neutral-800);
  --color-surface-overlay:  var(--color-neutral-800);
  --color-border-default:   var(--color-neutral-700);
}
```

---

## Dark Mode vs Light Mode

### Research findings (2024–2025 studies)

| Context | Recommended mode | Why |
|---------|-----------------|-----|
| Bright environment (office, daytime) | Light mode | Higher contrast in ambient light |
| Low-light environment (evening, dim room) | Dark mode | Reduces glare, easier on eyes |
| Complex cognitive tasks | Light mode | Better performance on hard tasks |
| Casual / entertainment use | Dark mode | User preference (79.7% on mobile) |
| Long-form reading | Light mode | Better for sustained focus |

**Key finding**: No statistically significant difference in eye fatigue between modes when used in appropriate lighting conditions. The context of use determines which is better.

**User preference**: 79.7% of users prefer dark mode on their phones. Always provide both.

### Implementation

```css
/* Automatic (system preference) */
@media (prefers-color-scheme: dark) {
  :root {
    --color-text-primary:   var(--color-neutral-50);
    --color-surface-base:   var(--color-neutral-900);
    /* ... */
  }
}

/* Manual override with data-theme attribute */
[data-theme="dark"] {
  --color-text-primary:   var(--color-neutral-50);
  --color-surface-base:   var(--color-neutral-900);
}
```

---

## 2025 Palette Trends

Based on design industry reports (Smashing Magazine, MockFlow, Material Design):

1. **Earthy tones + vibrant accents**: Browns, greens, ochres as base with coral, turquoise, or electric blue accents. Grounds the interface, adds warmth.

2. **Warm neutrals**: Warm beige, pale pink, peach as surface colors. Psychologically inviting, gentler on eyes than pure white, still accessible.

3. **Expressive color** (Material Design 3 direction): More vibrant, dynamic interfaces. Moving away from muted/corporate palettes toward personality-driven color.

4. **Biomorphic palettes**: Colors inspired by natural materials — terracotta, sage, sand, stone. Reduces digital fatigue.

---

## Cultural Color Considerations

Color meaning varies significantly across cultures. Always research before using color symbolically in global products:

| Color | Western | East Asian | Middle Eastern | Notes |
|-------|---------|-----------|----------------|-------|
| White | Purity, wedding | Mourning, death | Purity | Context-dependent |
| Red | Danger, passion | Luck, prosperity | Danger, caution | Most variable |
| Green | Nature, go, money | No strong association | Islam, nature | Generally positive |
| Black | Sophistication, mourning | Neutral | Death, mourning | Negative in some contexts |
| Yellow | Caution, cowardice | Royalty (China), courage (Japan) | Royalty, happiness | Positive in Asia |

---

## References

- MockFlow — [Color Psychology in UI Design: Trends and Insights for 2025](https://mockflow.com/blog/color-psychology-in-ui-design)
- Smashing Magazine — [The Psychology of Color in UX and Digital Products](https://www.smashingmagazine.com/2025/08/psychology-color-ux-design-digital-products/)
- ACM ETRA 2025 — [An Eye Tracking Study on Dark and Light Themes](https://dl.acm.org/doi/10.1145/3715669.3725879)
- PMC — [Immediate Effects of Light Mode and Dark Mode on Visual Fatigue](https://pmc.ncbi.nlm.nih.gov/articles/PMC12027292/)
- W3C — [WCAG 2.2 Success Criterion 1.4.3 Contrast](https://www.w3.org/WAI/WCAG22/quickref/#contrast-minimum)
