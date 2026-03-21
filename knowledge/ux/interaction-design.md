# Interaction Design

Interaction design defines how users interact with interfaces through motion, feedback, loading patterns, and input design. Good interaction design is largely invisible — users feel it works, but cannot explain why.

---

## Animation Timing (Research-Backed)

The most common mistake in UI animation is animations that are **too slow**. Users perceive lag at >500ms consistently.

| Interaction type | Duration | Easing | Notes |
|-----------------|----------|--------|-------|
| Hover state (color, opacity) | 80–120ms | ease-out | Should feel instant |
| Focus ring appearance | 80ms | ease-out | Immediate feedback |
| Tap/click pulse | 80–100ms | ease-out | Scale or opacity micro-pulse |
| Toggle (checkbox, switch) | 150ms | ease-in-out | State change, not directional |
| Dropdown / menu open | 150–200ms | ease-out | Element entering |
| Dropdown / menu close | 100–150ms | ease-in | Faster than opening |
| Modal / dialog open | 200–250ms | ease-out | Entering from below or fade |
| Modal / dialog close | 150–200ms | ease-in | Dismiss faster than open |
| Toast notification in | 200–300ms | ease-out | |
| Toast notification out | 150ms | ease-in | |
| Page transition | 300–400ms | ease-in-out | Maximum for full-page |
| **Hard upper limit** | **500ms** | — | **Never exceed** — felt as lag |

### Easing curves

```css
:root {
  /* Material Design / Google standard */
  --ease-standard:   cubic-bezier(0.2, 0, 0, 1);     /* default for most */
  --ease-emphasized: cubic-bezier(0.2, 0, 0, 1);     /* large movements */
  --ease-out:        cubic-bezier(0, 0, 0.2, 1);     /* elements entering */
  --ease-in:         cubic-bezier(0.4, 0, 1, 1);     /* elements leaving */
  --ease-in-out:     cubic-bezier(0.4, 0, 0.2, 1);   /* transforming in place */
}
```

**Rules**:
- **ease-out** = entering elements (decelerates into place — feels responsive, like it arrived for you)
- **ease-in** = exiting elements (accelerates away — feels natural, like it's going somewhere)
- **Never linear** — looks mechanical and cheap
- Exiting animations should be **faster** than entering animations

---

## prefers-reduced-motion (Safety-Critical)

Parallax, large animations, auto-playing effects, and scroll-jacking can cause **nausea, dizziness, and seizures** in users with vestibular disorders (estimated 35% of adults over 40 have some vestibular dysfunction).

This is not an accessibility bonus — it is a **safety requirement**.

### Global CSS reset (include in every project)

```css
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration:        0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration:       0.01ms !important;
    scroll-behavior:           auto !important;
  }
}
```

### Pattern for purposeful animations (preserve feedback, remove decoration)

```css
/* The animation provides useful feedback — preserve it in reduced form */
.progress-bar {
  transition: width 300ms ease-out;
}

@media (prefers-reduced-motion: reduce) {
  .progress-bar {
    transition: none;  /* Jump to final state — still shows progress */
  }
}

/* Decorative animation — remove entirely */
.hero-float {
  animation: float 3s ease-in-out infinite;
}

@media (prefers-reduced-motion: reduce) {
  .hero-float {
    animation: none;
  }
}
```

### React hook for conditional animation

```tsx
function useReducedMotion(): boolean {
  const [reducedMotion, setReducedMotion] = useState(
    () => window.matchMedia('(prefers-reduced-motion: reduce)').matches
  );

  useEffect(() => {
    const mq = window.matchMedia('(prefers-reduced-motion: reduce)');
    const handler = (e: MediaQueryListEvent) => setReducedMotion(e.matches);
    mq.addEventListener('change', handler);
    return () => mq.removeEventListener('change', handler);
  }, []);

  return reducedMotion;
}

// Usage
function AnimatedCard() {
  const reducedMotion = useReducedMotion();
  return (
    <div
      className={cn(
        'card',
        !reducedMotion && 'transition-transform duration-200 hover:scale-[1.02]'
      )}
    >
      ...
    </div>
  );
}
```

---

## Microinteraction Patterns

Gartner (2024): 75% of customer-facing applications will use microinteractions as a standard pattern. They are no longer optional — they are expected.

### Success feedback
```css
@keyframes success-pulse {
  0%   { transform: scale(1); }
  50%  { transform: scale(1.05); }
  100% { transform: scale(1); }
}

.success-state {
  animation: success-pulse 300ms ease-out;
  color: var(--color-success);
}
```

### Error shake (reduced motion: color only)
```css
@keyframes error-shake {
  0%, 100% { transform: translateX(0); }
  25%       { transform: translateX(-4px); }
  75%       { transform: translateX(4px); }
}

.error-state {
  animation: error-shake 300ms ease-in-out;
  border-color: var(--color-destructive);
}

@media (prefers-reduced-motion: reduce) {
  .error-state {
    animation: none;
    /* Color + icon still communicate error — no motion needed */
  }
}
```

### Loading button state
```tsx
function SubmitButton({ loading, children }: { loading: boolean; children: React.ReactNode }) {
  return (
    <button
      disabled={loading}
      aria-busy={loading}
      className={cn('btn-primary', loading && 'opacity-70 cursor-not-allowed')}
    >
      {loading ? (
        <span className="flex items-center gap-2">
          <span className="animate-spin h-4 w-4 border-2 border-current border-t-transparent rounded-full" aria-hidden="true" />
          Processing...
        </span>
      ) : children}
    </button>
  );
}
```

---

## Loading States

### Decision matrix

| Scenario | Best pattern | Avoid | Why |
|----------|-------------|-------|-----|
| Page / section initial load | Skeleton screen | Spinner | Skeleton: +20–30% perceived speed, prevents CLS |
| Button action (low-risk, fast) | Optimistic UI | Blocking overlay | Instant feedback, non-blocking |
| Button action (uncertain outcome) | Spinner on button + disable | Optimistic UI | Prevents false positives |
| File upload | Progress bar with % | Spinner | Users need progress estimate |
| Long operation (>3s) | Progress bar + estimated time | Any other | Reduces abandonment by ~30% |
| Background fetch (invisible) | Nothing | Spinner | Don't draw attention to infrastructure |

### Skeleton screen pattern

Skeleton screens feel **20–30% faster** than spinners for the same wait time because:
1. They establish layout expectations (no "jumping")
2. They mimic the final content structure
3. Users can start processing the page structure before content arrives

```tsx
// Base skeleton component
function Skeleton({ className }: { className?: string }) {
  return (
    <div
      className={cn('animate-pulse rounded-md bg-muted', className)}
      aria-hidden="true"
    />
  );
}

// Card skeleton — mirrors actual card structure exactly
function ProductCardSkeleton() {
  return (
    <div className="rounded-lg border bg-card p-4 space-y-3">
      <Skeleton className="h-48 w-full rounded-md" />   {/* image */}
      <div className="space-y-2">
        <Skeleton className="h-5 w-3/4" />              {/* title */}
        <Skeleton className="h-4 w-1/2" />              {/* subtitle */}
      </div>
      <div className="flex items-center justify-between">
        <Skeleton className="h-6 w-20" />               {/* price */}
        <Skeleton className="h-9 w-24 rounded-md" />    {/* button */}
      </div>
    </div>
  );
}
```

### Optimistic UI

Update the interface immediately, assuming the server request will succeed. Revert only on error.

**When to use**: Low-risk, reversible, frequent actions: liking, bookmarking, toggling, marking complete, reordering.

**When NOT to use**: Payments, deletions, irreversible actions, low-connectivity scenarios.

```tsx
function useOptimisticToggle<T>(
  initialValue: T,
  serverAction: (value: T) => Promise<void>
) {
  const [value, setValue] = useState(initialValue);
  const [error, setError] = useState<string | null>(null);

  async function toggle(newValue: T) {
    const previousValue = value;
    setValue(newValue);          // Optimistic update
    setError(null);

    try {
      await serverAction(newValue);
    } catch {
      setValue(previousValue);   // Revert on failure
      setError('Failed. Please try again.');
    }
  }

  return { value, toggle, error };
}
```

---

## CLS (Cumulative Layout Shift) Prevention

**Target**: CLS < 0.1 (Google Core Web Vitals "Good" threshold).
- Good: < 0.1
- Needs improvement: 0.1–0.25
- Poor: > 0.25

**60% of CLS is caused by images without explicit dimensions.**

### Image dimension reservation

```html
<!-- Always set width + height — browser reserves space before load -->
<img
  src="product.jpg"
  width="800"
  height="600"
  alt="Product name"
  loading="lazy"
  decoding="async"
/>
```

```css
/* Aspect ratio reservation for responsive images */
.image-container {
  aspect-ratio: 16 / 9;
  background: hsl(var(--muted));
  overflow: hidden;
}

.image-container img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

/* Avatar — always fixed dimensions */
.avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  flex-shrink: 0;
  background: hsl(var(--muted));
}
```

### Font CLS prevention

```css
/* Prevent font swap layout shift */
@font-face {
  font-family: 'Inter';
  font-display: swap;  /* or 'optional' for zero CLS */
  src: url('inter.woff2') format('woff2');
  /* size-adjust and ascent-override match fallback metrics */
  size-adjust: 100.06%;
  ascent-override: 105%;
}
```

### Dynamic content (ads, async components)

```css
/* Reserve minimum space for content that loads asynchronously */
.ad-slot          { min-height: 250px; container-type: inline-size; }
.async-card-list  { min-height: 400px; }
.comment-section  { min-height: 200px; }
```

---

## References

- NNGroup — [Executing UX Animations: Duration and Motion Characteristics](https://www.nngroup.com/articles/animation-duration/)
- Web.dev — [Cumulative Layout Shift (CLS)](https://web.dev/articles/cls)
- web.dev — [prefers-reduced-motion: Sometimes less movement is more](https://web.dev/articles/prefers-reduced-motion)
- Gartner — Microinteractions forecast (2024)
- LogRocket — [Skeleton Loading Screen Design](https://blog.logrocket.com/ux-design/skeleton-loading-screen-design/)
