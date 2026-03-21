# Mobile UX

Mobile UX differs fundamentally from desktop: users interact with thumbs, often one-handed, in varied lighting, while in motion. Every decision — touch target size, navigation placement, information density — must account for these physical and contextual constraints.

---

## Touch Target Sizes

The human thumb pad is approximately 25mm × 40mm. The tip used for precise tapping is about 9–10mm.

| Element | Minimum (CSS) | Recommended | Notes |
|---------|--------------|-------------|-------|
| Primary button | 44×44px | 48×48px | Full-width on mobile is ideal |
| Icon button | 44×44px tap area | 48×48px | Use padding to expand tap area visually |
| Navigation item | 44px height | 48px | Always use bottom navigation |
| List item | 44px height | 48px | `py-3` minimum on rows |
| Input field | 44px height | 48px | Never smaller — causes zoom on iOS |
| Checkbox / radio | 44×44px | Include label as tap target | Expand with `<label>` wrapping |
| Link in text | 24px minimum | Increase padding | Very hard to tap precisely in flowing text |

### Expanding tap areas without visual size change

```css
/* Method 1: Padding (simplest) */
.icon-button {
  padding: 12px;  /* 48px total for a 24px icon */
}

/* Method 2: Pseudo-element (no layout impact) */
.small-target {
  position: relative;
}
.small-target::after {
  content: '';
  position: absolute;
  inset: -8px;  /* Expands tap area 8px in each direction */
}

/* Method 3: min-height / min-width */
.touch-item {
  min-height: 44px;
  min-width: 44px;
  display: flex;
  align-items: center;
  justify-content: center;
}
```

---

## Thumb Zones (One-Handed Use)

Research shows users hold phones with a single hand 49% of the time, and use their thumb as the primary input tool.

### Zone map for standard phone (~6" screen)

```
┌─────────────────────────┐
│  ✗ ✗  Hard to reach     │  Corners — misclick-prone, avoid critical actions
│  ✗                      │  Top right especially difficult for right-hand users
│─────────────────────────│
│  ~                      │
│  ~  Stretch zone        │  Requires deliberate reach
│  ~  (moderate effort)   │
│─────────────────────────│
│  ✓  Natural zone        │  Comfortable, thumb moves here naturally
│  ✓✓ ✓✓ Primary zone    │  Bottom 40% — most comfortable, most reliable
│  [════ Navigation ════] │  Bottom bar — optimal position
└─────────────────────────┘
```

### Design decisions based on thumb zones

| Action type | Placement | Reasoning |
|------------|-----------|-----------|
| Primary CTA (submit, continue, buy) | Bottom third | Thumb-natural, less misclick risk |
| Destructive action (delete, cancel subscription) | Middle or top | Requires deliberate reach = intentional tap |
| Navigation tabs | Bottom bar | Highest engagement (1.5x vs hamburger) |
| Search bar | Top OR bottom (test both) | Varies by app type |
| Secondary actions | Middle | Accessible but not accidental |

### Foldable phones (2024–2025)

Samsung Galaxy Fold, Pixel Fold, and other foldables change the thumb zone significantly:
- In folded mode: standard phone thumb zones apply
- In unfolded mode: center of the large screen becomes hard-to-reach, edges become more accessible
- Design recommendation: test on actual foldable devices; container queries help adapt layouts

---

## Bottom Navigation vs Hamburger Menu

Research consistently shows that visible navigation dramatically outperforms hidden navigation:

| Metric | Bottom nav (visible) | Hamburger (hidden) |
|--------|---------------------|-------------------|
| User engagement | **1.5× higher** | Baseline |
| Feature discoverability | High | **~20% lower** |
| Task completion speed | Faster | Slower |
| Icons only (no label) | Moderate | — |
| Icons + labels | **+75% engagement** vs icon-only | — |

### When to use each pattern

| Pattern | Use when | Avoid when |
|---------|----------|------------|
| **Bottom navigation** | 3–5 primary destinations | >5 destinations (use hybrid) |
| **Tab bar (top)** | Web apps with horizontal flow | Mobile-primary apps |
| **Hamburger** | >8 items, secondary items only | Primary navigation |
| **Hybrid** | 4 primary + many secondary | Simple apps |

### Bottom navigation implementation

```tsx
const navItems = [
  { href: '/',        icon: HomeIcon,    label: 'Home' },
  { href: '/search',  icon: SearchIcon,  label: 'Search' },
  { href: '/saved',   icon: BookmarkIcon, label: 'Saved' },
  { href: '/profile', icon: UserIcon,    label: 'Profile' },
];

function BottomNav() {
  const pathname = usePathname();
  return (
    <nav
      className="fixed bottom-0 left-0 right-0 z-50 border-t bg-background safe-area-pb"
      aria-label="Primary navigation"
    >
      <div className="flex h-16 items-stretch">
        {navItems.map(({ href, icon: Icon, label }) => (
          <Link
            key={href}
            href={href}
            className={cn(
              'flex flex-1 flex-col items-center justify-center gap-1 text-xs',
              'min-h-[44px] transition-colors',
              pathname === href
                ? 'text-primary'
                : 'text-muted-foreground hover:text-foreground'
            )}
            aria-current={pathname === href ? 'page' : undefined}
          >
            <Icon className="h-5 w-5" aria-hidden="true" />
            <span>{label}</span>
          </Link>
        ))}
      </div>
    </nav>
  );
}
```

---

## Responsive Design Patterns (2025)

### Container queries (universally supported — Chrome 105+, Safari 16+, Firefox 110+)

Container queries enable component-level responsiveness, responding to the container size rather than the viewport. They are preferred over viewport media queries for reusable components.

```css
/* Define a container */
.card-container {
  container-type: inline-size;
  container-name: card;
}

/* Style based on container width */
@container card (min-width: 300px) {
  .card { flex-direction: row; }
}

@container card (min-width: 500px) {
  .card {
    grid-template-columns: 120px 1fr auto;
    gap: 1rem;
  }
}
```

### CSS subgrid (universally supported — all major browsers 2024)

Enables child elements to participate in the parent grid's track layout:

```css
.product-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 1rem;
}

.product-card {
  display: grid;
  grid-template-rows: auto 1fr auto;  /* image / content / footer */
  grid-row: span 3;                   /* each card spans 3 rows */
}

.product-grid {
  grid-template-rows: subgrid;  /* cards share the parent's row tracks */
}
```

### Fluid spacing with clamp

```css
/* Fluid section padding — no abrupt breakpoints */
.section {
  padding-block: clamp(2rem, 5vw, 6rem);
  padding-inline: clamp(1rem, 5vw, 3rem);
}

/* Fluid gap */
.card-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(min(280px, 100%), 1fr));
  gap: clamp(0.75rem, 2vw, 1.5rem);
}
```

---

## Mobile Performance

Mobile users are on slower networks and lower-powered devices. Performance is UX.

| Metric | Impact | Target |
|--------|--------|--------|
| First Contentful Paint (FCP) | First impression | < 1.8s |
| Largest Contentful Paint (LCP) | Main content visible | < 2.5s |
| Time to Interactive (TTI) | User can interact | < 3.8s |
| Cumulative Layout Shift (CLS) | Visual stability | < 0.1 |
| First Input Delay (FID) / INP | Responsiveness | < 200ms INP |

**Business impact**: A 1-second delay in page load time reduces conversions by **7%**. 61% of users leave mobile sites that aren't optimized.

---

## References

- Smashing Magazine — [Bottom Navigation Pattern For Mobile Web Pages: A Better Alternative?](https://www.smashingmagazine.com/2019/08/bottom-navigation-pattern-mobile-web-pages/)
- BrightHR Engineering — [Designing for Thumbs: Why Optimising for Thumb Usability Matters](https://engineering.brighthr.com/blog-post/designing-for-thumbs)
- Google Material Design — [Navigation](https://m3.material.io/components/navigation-bar/overview)
- Apple HIG — [Tab Bars](https://developer.apple.com/design/human-interface-guidelines/tab-bars)
- MDN — [CSS Container Queries](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_containment/Container_queries)
- web.dev — [Core Web Vitals](https://web.dev/vitals/)
