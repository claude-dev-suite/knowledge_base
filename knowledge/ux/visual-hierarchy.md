# UX Visual Hierarchy

Visual hierarchy is the arrangement of elements to guide users' attention in order of importance. Combined with research on how users scan screens, it is the foundation of effective interface design.

---

## Fluid Typography Scale

Use `clamp(min, preferred, max)` for fluid type that scales between breakpoints without abrupt jumps.

```css
:root {
  /* Adapts between ~375px and ~1280px viewport width */
  --text-xs:   clamp(0.75rem,  0.70rem + 0.25vw, 0.875rem);   /*  12–14px */
  --text-sm:   clamp(0.875rem, 0.82rem + 0.30vw, 1rem);        /*  14–16px */
  --text-base: clamp(1rem,     0.92rem + 0.40vw, 1.125rem);    /*  16–18px */
  --text-lg:   clamp(1.125rem, 1.00rem + 0.60vw, 1.375rem);    /*  18–22px */
  --text-xl:   clamp(1.25rem,  1.10rem + 0.75vw, 1.75rem);     /*  20–28px */
  --text-2xl:  clamp(1.5rem,   1.25rem + 1.25vw, 2.25rem);     /*  24–36px */
  --text-3xl:  clamp(1.875rem, 1.50rem + 1.90vw, 3rem);        /*  30–48px */
  --text-4xl:  clamp(2.25rem,  1.75rem + 2.50vw, 3.75rem);     /*  36–60px */
}
```

### Typography rules

| Property | Desktop | Mobile | Notes |
|----------|---------|--------|-------|
| Body text size | 16–18px | 16px min | Never below 16px for continuous reading |
| Line height (body) | 1.5–1.6 | 1.5 | Improves readability by 20% |
| Line height (headings) | 1.1–1.3 | 1.2 | Tighter for display text |
| Line length | 65–75 chars | 35–50 chars | 45–90 char range acceptable |
| Letter spacing (body) | 0–0.02em | 0–0.015em | Minimal; less than 0.02em |
| Letter spacing (headings) | −0.02 to −0.04em | −0.02em | Slight negative for large text |

**Research finding**: Proper letter/line spacing makes text **20% easier to read** vs. defaults and reduces eye strain by **30%**.

---

## Spacing System (4px base grid)

All spacing values should be multiples of 4px. This creates visual rhythm and consistency.

```css
:root {
  --space-0:  0;
  --space-px: 1px;
  --space-1:  0.25rem;  /*  4px */
  --space-2:  0.5rem;   /*  8px */
  --space-3:  0.75rem;  /* 12px */
  --space-4:  1rem;     /* 16px */
  --space-5:  1.25rem;  /* 20px */
  --space-6:  1.5rem;   /* 24px */
  --space-8:  2rem;     /* 32px */
  --space-10: 2.5rem;   /* 40px */
  --space-12: 3rem;     /* 48px */
  --space-16: 4rem;     /* 64px */
  --space-20: 5rem;     /* 80px */
  --space-24: 6rem;     /* 96px */
  --space-32: 8rem;     /* 128px */
}
```

### Spacing application guide

```
Component internal padding:    var(--space-3) to var(--space-4)   (12–16px)
Between related components:    var(--space-6) to var(--space-8)   (24–32px)
Between sections:              var(--space-12) to var(--space-16) (48–64px)
Page horizontal padding (mob): var(--space-4)                     (16px min)
Page horizontal padding (desk): var(--space-8) to var(--space-16) (32–64px)
```

---

## User Scanning Patterns (NNGroup Eye-Tracking Research)

Studies of over 200 users using eye-tracking equipment revealed four primary scanning patterns:

### F-Pattern
**What**: Users make two horizontal eye movements across the top, then scan down the left side in a vertical movement, forming an "F" shape.

**When it occurs**: Text-heavy pages, blog articles, product listings, unfocused browsing.

**Layout implications**:
- Place most important content in the first two paragraphs
- Front-load headings — first 2 words carry 80% of scanning weight
- Key information should be in the left 30% of the layout
- Avoid burying calls-to-action at the bottom of long text blocks

### Z-Pattern
**What**: Users scan horizontally across the top, then diagonally to the bottom-left, then horizontally across the bottom.

**When it occurs**: Pages with minimal text, landing pages, marketing pages.

**Layout implications**:
- Logo/brand: top-left
- Primary navigation: top-right
- Key message/image: middle
- Call to action: bottom-right

### Layer-Cake Pattern
**What**: Users focus only on headings and subheadings, skipping body text entirely until they find something relevant.

**When it occurs**: Motivated users looking for specific information, documentation, reference pages.

**Layout implications**:
- Use meaningful, specific subheadings (not generic "Introduction", "Overview")
- Bold key terms in body text to act as visual anchors
- The layer-cake is the **most effective pattern to design for** after initial F-pattern entry

### Spotted Pattern
**What**: Users jump directly to specific visual elements: images, bold text, links, or highlighted words.

**When it occurs**: Users searching for a specific item (price, date, name), product listings.

**Layout implications**:
- Use visual anchors for key data (prices, dates, names)
- Bold actionable text and CTAs
- Images draw attention — position them intentionally

---

## Cognitive Load Reduction

### Miller's Law
Working memory holds **7 ± 2 items** (chunks). This is a hard constraint of human cognition.

**Application rules**:
- Navigation: maximum 7 top-level items (5 recommended)
- Form fields per screen: 5–7 maximum
- Select/dropdown options before a search is needed: 7
- Dashboard KPIs per view: 5–9 maximum

### Progressive Disclosure

Reveal information gradually, based on user need:

| Type | When to use | Example |
|------|------------|---------|
| **Staged disclosure** | Linear wizard flows | Onboarding steps |
| **Contextual disclosure** | Details on current action | Tooltip, expandable row |
| **Feature-based** | Advanced functionality | "Advanced settings" accordion |
| **Interaction-triggered** | On demand | Accordion, "Show more", hover |

### Hick's Law
Decision time increases with log(n+1) of choices. Reduce options at every decision point.

**Practical implications**:
- Limit radio button groups to ≤5 options (use dropdown for more)
- Offer smart defaults — 80% of users take the default
- Group related choices to reduce perceived quantity

### Chunking
Break information into meaningful groups:
- Phone: `06 1234 5678` not `0612345678`
- Credit card: `4444 4444 4444 4444` not `4444444444444444`
- Long content: sections with headings, not walls of text

---

## Visual Weight Principles

Visual weight determines the first element users look at. Hierarchy requires ONE primary element, 1–2 secondary, and the rest tertiary.

| Factor | High weight | Low weight |
|--------|-------------|------------|
| **Size** | Larger than surrounding elements | Proportional or smaller |
| **Contrast** | High contrast against background | Low contrast |
| **Color** | Saturated, warm (red/orange/yellow) | Desaturated, cool, gray |
| **Position** | Top of page, left side, center | Bottom, right, edges |
| **Isolation** | Surrounded by whitespace | Crowded by other elements |
| **Motion** | Animated, moving | Static |
| **Shape** | Irregular, complex, pointing | Regular, rectangular |
| **Texture/detail** | Highly detailed | Simple, flat |

**Common mistake**: Multiple "primary" buttons or headline elements on one screen create visual noise and user confusion. One thing should be the most important — always.

---

## Whitespace as Design Tool

Whitespace (negative space) is not empty — it actively communicates relationship, importance, and quality.

### Gestalt Proximity
Elements physically close together are perceived as related. Use spacing to group and separate:

```
Related group:    [Label][Input][Helper text]  ← tight spacing
Unrelated items:  [Group A]          [Group B] ← generous spacing
```

### Macro vs Micro Whitespace

**Micro whitespace**: Padding inside components (buttons, cells, inputs).
- Too little: cramped, hard to read
- Right amount: comfortable, scannable

**Macro whitespace**: Space between sections and components.
- Too little: overwhelming, low-quality perception
- Right amount: premium feel, clear hierarchy
- Research: increasing whitespace around text increases comprehension by **20%**

### Whitespace signals quality
Studies show users rate pages with generous whitespace as more "credible", "professional", and "modern" — even when the content is identical to crowded versions.

---

## References

- NNGroup — [Text Scanning Patterns: Eye-Tracking Evidence](https://www.nngroup.com/articles/text-scanning-patterns-eyetracking/) (2006, updated 2023)
- NNGroup — [F-Shaped Pattern of Reading on the Web](https://www.nngroup.com/articles/f-shaped-pattern-reading-web-content/)
- Readability Matters — [How Important is X-Height for Font Legibility?](https://readabilitymatters.org/articles/research-highlight-how-important-is-x-height-for-font-legibility)
- ADOC Studio — [Typography Best Practices](https://www.adoc-studio.app/blog/typography-guide)
- Miller, G.A. (1956) — "The Magical Number Seven, Plus or Minus Two", Psychological Review
