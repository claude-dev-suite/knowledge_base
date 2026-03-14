# Tailwind CSS Responsive Design

A comprehensive guide to building responsive interfaces with Tailwind CSS, covering breakpoints, mobile-first design, and advanced responsive patterns.

---

## Table of Contents

1. [Breakpoint Overview](#breakpoint-overview)
2. [Mobile-First Approach](#mobile-first-approach)
3. [Targeting Specific Breakpoints](#targeting-specific-breakpoints)
4. [Custom Breakpoints](#custom-breakpoints)
5. [Responsive Typography](#responsive-typography)
6. [Responsive Spacing](#responsive-spacing)
7. [Responsive Flexbox and Grid](#responsive-flexbox-and-grid)
8. [Show/Hide Elements](#showhide-elements)
9. [Container Queries](#container-queries)
10. [Responsive Images](#responsive-images)
11. [Responsive Navigation Patterns](#responsive-navigation-patterns)
12. [Responsive Tables](#responsive-tables)
13. [Responsive Forms](#responsive-forms)
14. [Responsive Cards and Layouts](#responsive-cards-and-layouts)
15. [Media Query Features](#media-query-features)
16. [Arbitrary Breakpoints](#arbitrary-breakpoints)
17. [Best Practices](#best-practices)

---

## Breakpoint Overview

Tailwind CSS provides five default breakpoints that target common device sizes. These breakpoints use `min-width` media queries, meaning styles apply at that breakpoint **and above**.

### Default Breakpoints

| Prefix | Min Width | CSS Media Query | Common Devices |
|--------|-----------|-----------------|----------------|
| `sm` | 640px | `@media (min-width: 640px)` | Large phones, small tablets |
| `md` | 768px | `@media (min-width: 768px)` | Tablets |
| `lg` | 1024px | `@media (min-width: 1024px)` | Small laptops, tablets (landscape) |
| `xl` | 1280px | `@media (min-width: 1280px)` | Laptops, desktops |
| `2xl` | 1536px | `@media (min-width: 1536px)` | Large desktops, monitors |

### Understanding Breakpoint Ranges

```
0px          640px        768px        1024px       1280px       1536px
|------------|------------|------------|------------|------------|------------>
   (base)        sm           md           lg           xl          2xl
```

### Basic Usage

```html
<!-- No prefix = applies to all screen sizes (mobile-first base) -->
<div class="w-full">Full width on all screens</div>

<!-- sm: prefix = applies at 640px and above -->
<div class="w-full sm:w-1/2">Half width on small screens and up</div>

<!-- md: prefix = applies at 768px and above -->
<div class="w-full md:w-1/3">Third width on medium screens and up</div>

<!-- lg: prefix = applies at 1024px and above -->
<div class="w-full lg:w-1/4">Quarter width on large screens and up</div>

<!-- xl: prefix = applies at 1280px and above -->
<div class="w-full xl:w-1/6">Sixth width on extra-large screens and up</div>

<!-- 2xl: prefix = applies at 1536px and above -->
<div class="w-full 2xl:w-1/12">Twelfth width on 2xl screens and up</div>
```

### Breakpoint Behavior Examples

```html
<!-- This element has different widths at each breakpoint -->
<div class="
  w-full        /* 0px - 639px: full width */
  sm:w-3/4      /* 640px - 767px: 75% width */
  md:w-2/3      /* 768px - 1023px: 66% width */
  lg:w-1/2      /* 1024px - 1279px: 50% width */
  xl:w-1/3      /* 1280px - 1535px: 33% width */
  2xl:w-1/4     /* 1536px+: 25% width */
">
  Responsive width element
</div>
```

---

## Mobile-First Approach

Tailwind CSS uses a mobile-first breakpoint system. This means unprefixed utilities take effect on all screen sizes, while prefixed utilities only take effect at the specified breakpoint **and above**.

### Core Concept

```html
<!-- DON'T think "hide on mobile" -->
<!-- DO think "show on desktop" -->

<!-- Mobile-first: Start with mobile styles, then add larger screen overrides -->
<div class="text-sm md:text-base lg:text-lg xl:text-xl">
  Text starts small, grows at each breakpoint
</div>
```

### Mobile-First Layout Pattern

```html
<!-- Stack on mobile, horizontal on larger screens -->
<div class="flex flex-col md:flex-row gap-4">
  <div class="w-full md:w-1/2 lg:w-1/3">Column 1</div>
  <div class="w-full md:w-1/2 lg:w-1/3">Column 2</div>
  <div class="w-full md:w-full lg:w-1/3">Column 3</div>
</div>
```

### Why Mobile-First?

1. **Progressive Enhancement**: Start with a baseline experience and enhance for larger screens
2. **Performance**: Mobile devices often have slower connections; simpler base styles load faster
3. **Cleaner Code**: Reduces the need for max-width queries and overrides
4. **Better UX**: Forces consideration of mobile experience first

### Mobile-First vs Desktop-First

```html
<!-- Mobile-First (Tailwind's approach) -->
<div class="p-4 md:p-6 lg:p-8">
  <!-- Starts with p-4, expands for larger screens -->
</div>

<!-- If Tailwind were desktop-first (it's not!), you'd write: -->
<!-- <div class="p-8 md-down:p-6 sm-down:p-4"> -->
<!-- This is NOT how Tailwind works! -->
```

### Common Mobile-First Patterns

```html
<!-- Navigation: Hamburger on mobile, full nav on desktop -->
<nav class="flex flex-col md:flex-row md:items-center md:justify-between">
  <div class="flex items-center justify-between">
    <a href="/" class="text-xl font-bold">Logo</a>
    <button class="md:hidden">Menu</button>
  </div>
  <div class="hidden md:flex md:gap-6">
    <a href="#">Link 1</a>
    <a href="#">Link 2</a>
    <a href="#">Link 3</a>
  </div>
</nav>

<!-- Card Grid: Single column mobile, multi-column desktop -->
<div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-4">
  <div class="bg-white p-4 rounded shadow">Card 1</div>
  <div class="bg-white p-4 rounded shadow">Card 2</div>
  <div class="bg-white p-4 rounded shadow">Card 3</div>
  <div class="bg-white p-4 rounded shadow">Card 4</div>
</div>

<!-- Sidebar: Below content on mobile, beside content on desktop -->
<div class="flex flex-col lg:flex-row">
  <main class="flex-1 p-4 lg:p-6">Main Content</main>
  <aside class="w-full lg:w-80 p-4 bg-gray-100 order-first lg:order-last">
    Sidebar
  </aside>
</div>
```

---

## Targeting Specific Breakpoints

Sometimes you need styles that only apply within a specific range, not "this breakpoint and above."

### Max-Width Breakpoints

Define max-width breakpoints in your configuration:

```javascript
// tailwind.config.js
module.exports = {
  theme: {
    screens: {
      'sm': '640px',
      'md': '768px',
      'lg': '1024px',
      'xl': '1280px',
      '2xl': '1536px',
      // Max-width breakpoints
      'max-sm': { 'max': '639px' },
      'max-md': { 'max': '767px' },
      'max-lg': { 'max': '1023px' },
      'max-xl': { 'max': '1279px' },
      'max-2xl': { 'max': '1535px' },
    },
  },
}
```

```html
<!-- Only applies below 768px -->
<div class="max-md:text-center">
  Centered text on mobile only
</div>

<!-- Only applies below 640px -->
<div class="max-sm:hidden">
  Hidden on very small screens only
</div>
```

### Range Breakpoints

Target a specific range between two breakpoints:

```javascript
// tailwind.config.js
module.exports = {
  theme: {
    screens: {
      // Standard breakpoints
      'sm': '640px',
      'md': '768px',
      'lg': '1024px',
      'xl': '1280px',
      '2xl': '1536px',
      // Range breakpoints
      'sm-only': { 'min': '640px', 'max': '767px' },
      'md-only': { 'min': '768px', 'max': '1023px' },
      'lg-only': { 'min': '1024px', 'max': '1279px' },
      'xl-only': { 'min': '1280px', 'max': '1535px' },
      // Custom ranges
      'tablet': { 'min': '640px', 'max': '1023px' },
      'laptop': { 'min': '1024px', 'max': '1279px' },
    },
  },
}
```

```html
<!-- Only applies between 640px and 767px -->
<div class="sm-only:bg-blue-500">
  Blue background on small screens only
</div>

<!-- Only applies between 768px and 1023px -->
<div class="md-only:flex-row md-only:gap-8">
  Special layout for medium screens only
</div>

<!-- Tablet range (640px - 1023px) -->
<div class="tablet:text-center tablet:p-6">
  Centered with more padding on tablets
</div>
```

### Combining Min and Max

```html
<!-- Show different content at different breakpoints -->
<div>
  <span class="max-sm:inline hidden">Mobile</span>
  <span class="sm-only:inline hidden">Small</span>
  <span class="md-only:inline hidden">Medium</span>
  <span class="lg-only:inline hidden">Large</span>
  <span class="xl:inline hidden">Extra Large+</span>
</div>
```

---

## Custom Breakpoints

Tailwind allows complete customization of breakpoints to match your design requirements.

### Adding New Breakpoints

```javascript
// tailwind.config.js
module.exports = {
  theme: {
    screens: {
      'xs': '475px',    // Extra small devices
      'sm': '640px',    // Small devices
      'md': '768px',    // Medium devices
      'lg': '1024px',   // Large devices
      'xl': '1280px',   // Extra large devices
      '2xl': '1536px',  // 2X large devices
      '3xl': '1920px',  // Full HD monitors
      '4xl': '2560px',  // QHD monitors
    },
  },
}
```

```html
<!-- Using custom xs breakpoint -->
<div class="text-xs xs:text-sm sm:text-base">
  Even more granular control
</div>

<!-- Using custom 3xl breakpoint -->
<div class="max-w-7xl 3xl:max-w-[2000px]">
  Wider container on large monitors
</div>
```

### Replacing Default Breakpoints

```javascript
// tailwind.config.js - Complete replacement
module.exports = {
  theme: {
    screens: {
      'mobile': '320px',
      'tablet': '768px',
      'laptop': '1024px',
      'desktop': '1440px',
      'wide': '1920px',
    },
  },
}
```

```html
<!-- Using custom semantic breakpoints -->
<div class="mobile:p-4 tablet:p-6 laptop:p-8 desktop:p-10 wide:p-12">
  Custom breakpoint names
</div>
```

### Extending Breakpoints

```javascript
// tailwind.config.js - Add to existing breakpoints
module.exports = {
  theme: {
    extend: {
      screens: {
        'xs': '475px',
        '3xl': '1920px',
        // Raw media queries
        'tall': { 'raw': '(min-height: 800px)' },
        'retina': { 'raw': '(-webkit-min-device-pixel-ratio: 2)' },
        'hover-supported': { 'raw': '(hover: hover)' },
      },
    },
  },
}
```

```html
<!-- Using raw media query breakpoints -->
<div class="tall:min-h-screen">
  Full height on tall screens
</div>

<div class="hover-supported:hover:bg-blue-500">
  Hover effects only on devices that support hover
</div>
```

### Device-Specific Breakpoints

```javascript
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      screens: {
        // iPhone SE
        'iphone-se': '375px',
        // iPad
        'ipad': '768px',
        'ipad-pro': '1024px',
        // Common desktop sizes
        'laptop': '1366px',
        'desktop-hd': '1920px',
        'desktop-4k': '3840px',
      },
    },
  },
}
```

---

## Responsive Typography

Create fluid, readable typography across all screen sizes.

### Responsive Font Sizes

```html
<!-- Responsive heading hierarchy -->
<h1 class="text-2xl sm:text-3xl md:text-4xl lg:text-5xl xl:text-6xl font-bold">
  Main Heading
</h1>

<h2 class="text-xl sm:text-2xl md:text-3xl lg:text-4xl font-semibold">
  Section Heading
</h2>

<h3 class="text-lg sm:text-xl md:text-2xl lg:text-3xl font-medium">
  Subsection Heading
</h3>

<p class="text-sm sm:text-base md:text-lg leading-relaxed">
  Body text that scales appropriately across devices.
</p>
```

### Responsive Line Height

```html
<!-- Tighter line height on mobile, looser on desktop -->
<p class="leading-tight sm:leading-normal md:leading-relaxed lg:leading-loose">
  Long paragraph text that benefits from more breathing room on larger screens
  where lines are longer and harder to track.
</p>

<!-- Hero text with responsive line height -->
<h1 class="text-4xl md:text-6xl leading-none md:leading-tight">
  Big Bold<br />Hero Title
</h1>
```

### Responsive Letter Spacing

```html
<!-- Tighter tracking on small screens, wider on large -->
<h1 class="tracking-tight sm:tracking-normal md:tracking-wide lg:tracking-wider">
  Spaced Heading
</h1>

<!-- Uppercase text with responsive tracking -->
<span class="uppercase tracking-widest sm:tracking-[0.2em] md:tracking-[0.3em]">
  Brand Name
</span>
```

### Responsive Text Alignment

```html
<!-- Centered on mobile, left-aligned on desktop -->
<div class="text-center md:text-left">
  <h2 class="text-2xl font-bold">Title</h2>
  <p>Description text</p>
</div>

<!-- Right-aligned on larger screens -->
<div class="text-left lg:text-right">
  <span class="text-sm text-gray-500">Published: Jan 2024</span>
</div>
```

### Responsive Font Weight

```html
<!-- Lighter weight on mobile for readability -->
<h1 class="font-medium md:font-semibold lg:font-bold">
  Responsive Weight Heading
</h1>
```

### Fluid Typography with Clamp (Arbitrary Values)

```html
<!-- Fluid font size using clamp -->
<h1 class="text-[clamp(2rem,5vw,4rem)]">
  Fluid Typography
</h1>

<!-- Fluid with Tailwind breakpoints as fallback -->
<h1 class="text-2xl sm:text-[clamp(1.5rem,4vw,3rem)] lg:text-5xl">
  Hybrid Fluid Typography
</h1>
```

### Complete Typography Example

```html
<article class="max-w-prose mx-auto px-4 sm:px-6 lg:px-8">
  <header class="mb-8 md:mb-12">
    <h1 class="
      text-3xl sm:text-4xl md:text-5xl lg:text-6xl
      font-bold
      leading-tight sm:leading-tight md:leading-snug
      tracking-tight
      mb-4
    ">
      Article Title That May Span Multiple Lines
    </h1>
    <p class="
      text-lg sm:text-xl md:text-2xl
      text-gray-600
      leading-relaxed
    ">
      A compelling subtitle that introduces the article content.
    </p>
  </header>

  <div class="
    prose prose-sm sm:prose-base lg:prose-lg xl:prose-xl
    prose-headings:font-semibold
    prose-p:leading-relaxed
  ">
    <!-- Article content -->
  </div>
</article>
```

---

## Responsive Spacing

Control padding, margin, and gap responsively for optimal layouts.

### Responsive Padding

```html
<!-- Section padding that grows with screen size -->
<section class="py-8 sm:py-12 md:py-16 lg:py-20 xl:py-24">
  <div class="px-4 sm:px-6 md:px-8 lg:px-12">
    Section content
  </div>
</section>

<!-- Card with responsive padding -->
<div class="p-4 sm:p-6 md:p-8 lg:p-10 bg-white rounded-lg shadow">
  Card content with breathing room
</div>

<!-- Asymmetric responsive padding -->
<div class="pt-8 pb-16 md:pt-12 md:pb-24 lg:pt-16 lg:pb-32 px-4 md:px-8">
  Hero section with more bottom padding
</div>
```

### Responsive Margin

```html
<!-- Section margin -->
<section class="my-8 sm:my-12 md:my-16 lg:my-20">
  Content section
</section>

<!-- Auto margins for centering with responsive width -->
<div class="w-full md:w-3/4 lg:w-2/3 mx-auto">
  Centered container
</div>

<!-- Responsive negative margins -->
<div class="-mx-4 sm:-mx-6 md:-mx-8 lg:mx-0">
  Full-bleed on mobile, contained on desktop
</div>
```

### Responsive Gap

```html
<!-- Flex container with responsive gap -->
<div class="flex flex-wrap gap-2 sm:gap-4 md:gap-6 lg:gap-8">
  <div>Item 1</div>
  <div>Item 2</div>
  <div>Item 3</div>
</div>

<!-- Grid with responsive gap -->
<div class="grid grid-cols-2 lg:grid-cols-4 gap-4 sm:gap-6 lg:gap-8">
  <div>Card 1</div>
  <div>Card 2</div>
  <div>Card 3</div>
  <div>Card 4</div>
</div>

<!-- Different row and column gaps -->
<div class="grid grid-cols-3 gap-x-4 gap-y-8 md:gap-x-6 md:gap-y-12">
  <!-- Items -->
</div>
```

### Responsive Space Utilities

```html
<!-- Vertical spacing between children -->
<div class="space-y-4 sm:space-y-6 md:space-y-8">
  <div>Section 1</div>
  <div>Section 2</div>
  <div>Section 3</div>
</div>

<!-- Horizontal spacing on larger screens -->
<div class="space-y-4 md:space-y-0 md:space-x-6 flex flex-col md:flex-row">
  <div>Column 1</div>
  <div>Column 2</div>
</div>
```

### Complete Spacing Example

```html
<!-- Page layout with comprehensive responsive spacing -->
<div class="min-h-screen">
  <!-- Header -->
  <header class="py-4 sm:py-6 px-4 sm:px-6 lg:px-8">
    <nav class="flex items-center gap-4 sm:gap-6 md:gap-8">
      <!-- Nav items -->
    </nav>
  </header>

  <!-- Main content -->
  <main class="py-8 sm:py-12 md:py-16 lg:py-20">
    <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
      <!-- Hero -->
      <section class="mb-12 sm:mb-16 md:mb-20 lg:mb-24">
        <h1 class="mb-4 sm:mb-6 md:mb-8">Heading</h1>
        <p class="mb-6 sm:mb-8 md:mb-10">Description</p>
      </section>

      <!-- Content grid -->
      <section class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6 md:gap-8 lg:gap-10">
        <!-- Cards -->
      </section>
    </div>
  </main>

  <!-- Footer -->
  <footer class="py-8 sm:py-12 md:py-16 mt-auto">
    <div class="px-4 sm:px-6 lg:px-8">
      <!-- Footer content -->
    </div>
  </footer>
</div>
```

---

## Responsive Flexbox and Grid

Build flexible layouts that adapt to any screen size.

### Responsive Flexbox Direction

```html
<!-- Column on mobile, row on tablet+ -->
<div class="flex flex-col md:flex-row gap-4">
  <div class="flex-1">Item 1</div>
  <div class="flex-1">Item 2</div>
  <div class="flex-1">Item 3</div>
</div>

<!-- Reverse order on certain screens -->
<div class="flex flex-col md:flex-row-reverse">
  <div class="md:w-1/2">Content (appears first on mobile)</div>
  <div class="md:w-1/2">Image (appears first on desktop)</div>
</div>
```

### Responsive Flex Wrap

```html
<!-- No wrap on mobile, wrap on larger screens -->
<div class="flex overflow-x-auto md:flex-wrap gap-4">
  <div class="flex-shrink-0 w-64 md:w-auto md:flex-1">Card 1</div>
  <div class="flex-shrink-0 w-64 md:w-auto md:flex-1">Card 2</div>
  <div class="flex-shrink-0 w-64 md:w-auto md:flex-1">Card 3</div>
  <div class="flex-shrink-0 w-64 md:w-auto md:flex-1">Card 4</div>
</div>
```

### Responsive Flex Alignment

```html
<!-- Different alignments at different breakpoints -->
<div class="flex flex-col md:flex-row items-center md:items-start lg:items-center justify-center md:justify-between">
  <div>Logo</div>
  <nav class="flex gap-4">
    <a href="#">Link</a>
  </nav>
</div>
```

### Responsive Grid Columns

```html
<!-- Classic responsive grid -->
<div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-6">
  <div class="bg-gray-100 p-4">Card 1</div>
  <div class="bg-gray-100 p-4">Card 2</div>
  <div class="bg-gray-100 p-4">Card 3</div>
  <div class="bg-gray-100 p-4">Card 4</div>
</div>

<!-- 12-column grid system -->
<div class="grid grid-cols-12 gap-4">
  <div class="col-span-12 md:col-span-8 lg:col-span-9">Main content</div>
  <div class="col-span-12 md:col-span-4 lg:col-span-3">Sidebar</div>
</div>

<!-- Auto-fit grid that adapts to content -->
<div class="grid grid-cols-[repeat(auto-fit,minmax(250px,1fr))] gap-6">
  <div>Auto-sized card 1</div>
  <div>Auto-sized card 2</div>
  <div>Auto-sized card 3</div>
</div>
```

### Responsive Grid Rows

```html
<!-- Different row configurations -->
<div class="grid grid-rows-3 md:grid-rows-2 lg:grid-rows-1 grid-flow-col gap-4">
  <div>Item 1</div>
  <div>Item 2</div>
  <div>Item 3</div>
</div>
```

### Responsive Grid Span

```html
<!-- Elements that span different amounts at different breakpoints -->
<div class="grid grid-cols-6 gap-4">
  <div class="col-span-6 md:col-span-3 lg:col-span-2">
    Full on mobile, half on tablet, third on desktop
  </div>
  <div class="col-span-6 md:col-span-3 lg:col-span-4">
    Full on mobile, half on tablet, two-thirds on desktop
  </div>
</div>

<!-- Featured item that spans multiple rows -->
<div class="grid grid-cols-2 md:grid-cols-3 gap-4">
  <div class="col-span-2 md:col-span-1 md:row-span-2 h-64 md:h-auto">
    Featured
  </div>
  <div>Regular 1</div>
  <div>Regular 2</div>
  <div class="col-span-2 md:col-span-1">Regular 3</div>
</div>
```

### Complex Layout Example

```html
<!-- Magazine-style layout -->
<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4 lg:gap-6">
  <!-- Featured article - full width on mobile, 2 cols on tablet, 2 cols + 2 rows on desktop -->
  <article class="md:col-span-2 lg:row-span-2">
    <img src="featured.jpg" class="w-full h-48 md:h-64 lg:h-full object-cover" />
    <h2 class="text-xl md:text-2xl lg:text-3xl font-bold mt-4">Featured Story</h2>
  </article>

  <!-- Regular articles -->
  <article class="flex md:flex-col gap-4">
    <img src="article1.jpg" class="w-24 md:w-full h-24 md:h-32 object-cover" />
    <h3 class="font-semibold">Article 1</h3>
  </article>

  <article class="flex md:flex-col gap-4">
    <img src="article2.jpg" class="w-24 md:w-full h-24 md:h-32 object-cover" />
    <h3 class="font-semibold">Article 2</h3>
  </article>

  <article class="flex md:flex-col gap-4 md:col-span-2 lg:col-span-1">
    <img src="article3.jpg" class="w-24 md:w-full h-24 md:h-32 object-cover" />
    <h3 class="font-semibold">Article 3</h3>
  </article>

  <article class="flex md:flex-col gap-4 md:col-span-2 lg:col-span-1">
    <img src="article4.jpg" class="w-24 md:w-full h-24 md:h-32 object-cover" />
    <h3 class="font-semibold">Article 4</h3>
  </article>
</div>
```

---

## Show/Hide Elements

Control element visibility across different screen sizes.

### Basic Show/Hide

```html
<!-- Hidden on mobile, visible on medium screens and up -->
<div class="hidden md:block">
  Desktop content
</div>

<!-- Visible on mobile, hidden on medium screens and up -->
<div class="block md:hidden">
  Mobile content
</div>

<!-- Different display types -->
<span class="hidden lg:inline">Inline on large screens</span>
<div class="hidden sm:flex">Flex container on small screens+</div>
<div class="hidden md:grid md:grid-cols-3">Grid on medium screens+</div>
```

### Breakpoint-Specific Visibility

```html
<!-- Show only at specific breakpoints -->
<div>
  <!-- Only visible below 640px -->
  <span class="sm:hidden">XS Only</span>

  <!-- Only visible between 640px and 767px -->
  <span class="hidden sm:inline md:hidden">SM Only</span>

  <!-- Only visible between 768px and 1023px -->
  <span class="hidden md:inline lg:hidden">MD Only</span>

  <!-- Only visible between 1024px and 1279px -->
  <span class="hidden lg:inline xl:hidden">LG Only</span>

  <!-- Only visible between 1280px and 1535px -->
  <span class="hidden xl:inline 2xl:hidden">XL Only</span>

  <!-- Only visible at 1536px and above -->
  <span class="hidden 2xl:inline">2XL Only</span>
</div>
```

### Conditional Navigation

```html
<header class="flex items-center justify-between p-4">
  <a href="/" class="text-xl font-bold">Logo</a>

  <!-- Mobile menu button - hidden on large screens -->
  <button class="lg:hidden p-2" aria-label="Toggle menu">
    <svg class="w-6 h-6"><!-- Menu icon --></svg>
  </button>

  <!-- Desktop navigation - hidden on mobile -->
  <nav class="hidden lg:flex lg:items-center lg:gap-6">
    <a href="#" class="hover:text-blue-600">Home</a>
    <a href="#" class="hover:text-blue-600">About</a>
    <a href="#" class="hover:text-blue-600">Services</a>
    <a href="#" class="hover:text-blue-600">Contact</a>
  </nav>
</header>

<!-- Mobile navigation menu -->
<nav class="lg:hidden bg-gray-100 p-4">
  <a href="#" class="block py-2">Home</a>
  <a href="#" class="block py-2">About</a>
  <a href="#" class="block py-2">Services</a>
  <a href="#" class="block py-2">Contact</a>
</nav>
```

### Responsive Content Variations

```html
<!-- Different content for different screens -->
<div class="bg-blue-500 p-4 text-white">
  <p class="md:hidden">Tap to learn more</p>
  <p class="hidden md:block lg:hidden">Click to learn more</p>
  <p class="hidden lg:block">Hover over items to learn more</p>
</div>

<!-- Abbreviated vs full text -->
<button class="bg-blue-600 text-white px-4 py-2 rounded">
  <span class="sm:hidden">Save</span>
  <span class="hidden sm:inline md:hidden">Save Changes</span>
  <span class="hidden md:inline">Save All Changes</span>
</button>
```

### Visibility with Accessibility

```html
<!-- Visually hidden but accessible to screen readers -->
<span class="sr-only">Accessible label</span>

<!-- Hidden from screen readers but visible -->
<span aria-hidden="true" class="hidden md:inline">Decorative</span>

<!-- Skip link - visible on focus for keyboard users -->
<a href="#main" class="sr-only focus:not-sr-only focus:absolute focus:top-0 focus:left-0 bg-blue-600 text-white p-2">
  Skip to main content
</a>
```

---

## Container Queries

Container queries (Tailwind v3.2+) allow you to style elements based on the size of their container rather than the viewport.

### Basic Container Setup

```html
<!-- Define a container -->
<div class="@container">
  <!-- Child elements can use container query variants -->
  <div class="@md:flex @md:gap-4">
    Container-aware layout
  </div>
</div>

<!-- Named containers -->
<div class="@container/main">
  <div class="@lg/main:grid @lg/main:grid-cols-2">
    Named container query
  </div>
</div>
```

### Container Query Breakpoints

Default container query breakpoints:

| Variant | Minimum Width |
|---------|---------------|
| `@xs` | 20rem (320px) |
| `@sm` | 24rem (384px) |
| `@md` | 28rem (448px) |
| `@lg` | 32rem (512px) |
| `@xl` | 36rem (576px) |
| `@2xl` | 42rem (672px) |
| `@3xl` | 48rem (768px) |
| `@4xl` | 56rem (896px) |
| `@5xl` | 64rem (1024px) |
| `@6xl` | 72rem (1152px) |
| `@7xl` | 80rem (1280px) |

### Card with Container Queries

```html
<!-- Card that adapts to its container width -->
<div class="@container">
  <article class="
    flex flex-col
    @sm:flex-row @sm:items-center
    @lg:flex-col @lg:text-center
    gap-4 p-4 bg-white rounded-lg shadow
  ">
    <img
      src="image.jpg"
      class="
        w-full h-32 object-cover rounded
        @sm:w-24 @sm:h-24
        @lg:w-full @lg:h-48
      "
      alt="Card image"
    />
    <div class="flex-1">
      <h3 class="
        text-lg font-semibold
        @md:text-xl
        @lg:text-2xl
      ">
        Card Title
      </h3>
      <p class="
        text-sm text-gray-600
        @md:text-base
        @lg:mt-2
      ">
        Card description that adapts to container size.
      </p>
    </div>
  </article>
</div>
```

### Sidebar Content with Container Queries

```html
<!-- Content that works in main area or sidebar -->
<div class="grid grid-cols-1 lg:grid-cols-4 gap-6">
  <!-- Main content area -->
  <main class="lg:col-span-3 @container">
    <div class="@md:grid @md:grid-cols-2 @xl:grid-cols-3 gap-6">
      <article class="p-4 bg-white rounded shadow">Article 1</article>
      <article class="p-4 bg-white rounded shadow">Article 2</article>
      <article class="p-4 bg-white rounded shadow">Article 3</article>
    </div>
  </main>

  <!-- Sidebar - same component, different container size -->
  <aside class="@container">
    <div class="@md:grid @md:grid-cols-2 @xl:grid-cols-3 gap-4">
      <!-- In narrow sidebar, these won't hit @md breakpoint -->
      <article class="p-3 bg-gray-100 rounded">Related 1</article>
      <article class="p-3 bg-gray-100 rounded">Related 2</article>
      <article class="p-3 bg-gray-100 rounded">Related 3</article>
    </div>
  </aside>
</div>
```

### Custom Container Query Sizes

```javascript
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      containers: {
        'xs': '20rem',
        'sm': '24rem',
        'md': '28rem',
        'lg': '32rem',
        'xl': '36rem',
        '2xl': '42rem',
        'card': '400px',  // Custom named size
        'sidebar': '300px',
      },
    },
  },
}
```

---

## Responsive Images

Handle images responsively for optimal display and performance.

### Responsive Image Sizing

```html
<!-- Full width on mobile, constrained on larger screens -->
<img
  src="image.jpg"
  class="w-full md:w-3/4 lg:w-1/2 xl:w-1/3"
  alt="Responsive image"
/>

<!-- Max width with responsive adjustments -->
<img
  src="image.jpg"
  class="w-full max-w-xs sm:max-w-sm md:max-w-md lg:max-w-lg mx-auto"
  alt="Constrained image"
/>
```

### Responsive Aspect Ratios

```html
<!-- Different aspect ratios at different breakpoints -->
<div class="aspect-square sm:aspect-video md:aspect-[4/3] lg:aspect-[16/9]">
  <img src="image.jpg" class="w-full h-full object-cover" alt="Adaptive aspect ratio" />
</div>

<!-- Video container with responsive aspect ratio -->
<div class="aspect-video md:aspect-[21/9]">
  <iframe src="video.mp4" class="w-full h-full"></iframe>
</div>
```

### Responsive Object Fit

```html
<!-- Different object-fit behaviors at breakpoints -->
<div class="h-64 md:h-96">
  <img
    src="image.jpg"
    class="w-full h-full object-cover md:object-contain lg:object-cover"
    alt="Adaptive fit"
  />
</div>

<!-- Object position changes -->
<div class="h-48 md:h-64">
  <img
    src="portrait.jpg"
    class="w-full h-full object-cover object-top md:object-center"
    alt="Position shifts"
  />
</div>
```

### Responsive Image Gallery

```html
<!-- Image gallery with responsive layout -->
<div class="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 lg:grid-cols-6 gap-2 md:gap-4">
  <div class="aspect-square col-span-2 row-span-2 sm:col-span-1 sm:row-span-1 md:col-span-2 md:row-span-2">
    <img src="featured.jpg" class="w-full h-full object-cover rounded" alt="Featured" />
  </div>
  <div class="aspect-square">
    <img src="thumb1.jpg" class="w-full h-full object-cover rounded" alt="Thumbnail 1" />
  </div>
  <div class="aspect-square">
    <img src="thumb2.jpg" class="w-full h-full object-cover rounded" alt="Thumbnail 2" />
  </div>
  <!-- More thumbnails -->
</div>
```

### Responsive Background Images

```html
<!-- Hero with responsive background positioning -->
<div class="
  bg-cover bg-center
  md:bg-right
  lg:bg-center
  h-64 sm:h-80 md:h-96 lg:h-[500px]
  bg-[url('/hero-mobile.jpg')]
  md:bg-[url('/hero-desktop.jpg')]
">
  <div class="h-full flex items-center justify-center bg-black/40">
    <h1 class="text-white text-3xl md:text-5xl font-bold">Hero Title</h1>
  </div>
</div>
```

### Art Direction with Picture Element

```html
<!-- Use HTML picture element with Tailwind styling -->
<picture>
  <source media="(min-width: 1024px)" srcset="hero-desktop.jpg" />
  <source media="(min-width: 640px)" srcset="hero-tablet.jpg" />
  <img
    src="hero-mobile.jpg"
    class="w-full h-48 sm:h-64 md:h-80 lg:h-96 object-cover"
    alt="Responsive hero image"
  />
</picture>
```

---

## Responsive Navigation Patterns

Build navigation that works across all devices.

### Mobile-First Hamburger Navigation

```html
<header class="bg-white shadow">
  <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
    <div class="flex items-center justify-between h-16">
      <!-- Logo -->
      <a href="/" class="text-xl font-bold text-gray-900">Brand</a>

      <!-- Mobile menu button -->
      <button
        class="md:hidden p-2 rounded-md text-gray-600 hover:text-gray-900 hover:bg-gray-100"
        aria-label="Toggle navigation"
      >
        <svg class="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke="currentColor">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 12h16M4 18h16" />
        </svg>
      </button>

      <!-- Desktop navigation -->
      <nav class="hidden md:flex md:items-center md:space-x-8">
        <a href="#" class="text-gray-600 hover:text-gray-900">Home</a>
        <a href="#" class="text-gray-600 hover:text-gray-900">Products</a>
        <a href="#" class="text-gray-600 hover:text-gray-900">About</a>
        <a href="#" class="text-gray-600 hover:text-gray-900">Contact</a>
        <a href="#" class="bg-blue-600 text-white px-4 py-2 rounded-md hover:bg-blue-700">
          Sign Up
        </a>
      </nav>
    </div>
  </div>

  <!-- Mobile menu (toggle with JS) -->
  <nav class="md:hidden border-t border-gray-200">
    <div class="px-4 py-3 space-y-1">
      <a href="#" class="block py-2 text-gray-600 hover:text-gray-900">Home</a>
      <a href="#" class="block py-2 text-gray-600 hover:text-gray-900">Products</a>
      <a href="#" class="block py-2 text-gray-600 hover:text-gray-900">About</a>
      <a href="#" class="block py-2 text-gray-600 hover:text-gray-900">Contact</a>
      <a href="#" class="block py-2 text-blue-600 font-medium">Sign Up</a>
    </div>
  </nav>
</header>
```

### Responsive Dropdown Navigation

```html
<nav class="bg-white shadow">
  <div class="max-w-7xl mx-auto px-4">
    <div class="flex items-center h-16 space-x-8">
      <a href="/" class="font-bold">Logo</a>

      <!-- Desktop dropdown -->
      <div class="hidden md:block relative group">
        <button class="flex items-center space-x-1 text-gray-600 hover:text-gray-900">
          <span>Products</span>
          <svg class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7" />
          </svg>
        </button>

        <!-- Dropdown menu -->
        <div class="
          absolute left-0 mt-2 w-48 bg-white rounded-md shadow-lg py-1
          opacity-0 invisible group-hover:opacity-100 group-hover:visible
          transition-all duration-200
        ">
          <a href="#" class="block px-4 py-2 text-gray-700 hover:bg-gray-100">Product 1</a>
          <a href="#" class="block px-4 py-2 text-gray-700 hover:bg-gray-100">Product 2</a>
          <a href="#" class="block px-4 py-2 text-gray-700 hover:bg-gray-100">Product 3</a>
        </div>
      </div>
    </div>
  </div>
</nav>
```

### Responsive Tab Navigation

```html
<!-- Horizontal tabs on desktop, vertical on mobile -->
<div class="border-b border-gray-200">
  <nav class="
    flex flex-col
    sm:flex-row sm:space-x-8
    space-y-2 sm:space-y-0
    px-4 sm:px-0
  ">
    <a href="#" class="
      py-2 sm:py-4 px-1
      border-l-4 sm:border-l-0 sm:border-b-2
      border-blue-500 text-blue-600
      font-medium
    ">
      Tab 1
    </a>
    <a href="#" class="
      py-2 sm:py-4 px-1
      border-l-4 sm:border-l-0 sm:border-b-2
      border-transparent text-gray-500
      hover:text-gray-700 hover:border-gray-300
    ">
      Tab 2
    </a>
    <a href="#" class="
      py-2 sm:py-4 px-1
      border-l-4 sm:border-l-0 sm:border-b-2
      border-transparent text-gray-500
      hover:text-gray-700 hover:border-gray-300
    ">
      Tab 3
    </a>
  </nav>
</div>
```

### Bottom Navigation (Mobile)

```html
<!-- Fixed bottom navigation for mobile -->
<nav class="
  fixed bottom-0 left-0 right-0
  md:hidden
  bg-white border-t border-gray-200
  flex justify-around
  py-2
  z-50
">
  <a href="#" class="flex flex-col items-center text-blue-600">
    <svg class="w-6 h-6" fill="currentColor" viewBox="0 0 20 20">
      <!-- Home icon -->
    </svg>
    <span class="text-xs mt-1">Home</span>
  </a>
  <a href="#" class="flex flex-col items-center text-gray-600">
    <svg class="w-6 h-6" fill="currentColor" viewBox="0 0 20 20">
      <!-- Search icon -->
    </svg>
    <span class="text-xs mt-1">Search</span>
  </a>
  <a href="#" class="flex flex-col items-center text-gray-600">
    <svg class="w-6 h-6" fill="currentColor" viewBox="0 0 20 20">
      <!-- Profile icon -->
    </svg>
    <span class="text-xs mt-1">Profile</span>
  </a>
</nav>

<!-- Add padding to main content to account for fixed nav -->
<main class="pb-16 md:pb-0">
  <!-- Content -->
</main>
```

---

## Responsive Tables

Make tables work on all screen sizes.

### Horizontal Scroll Table

```html
<!-- Simple horizontal scroll on mobile -->
<div class="overflow-x-auto">
  <table class="min-w-full divide-y divide-gray-200">
    <thead class="bg-gray-50">
      <tr>
        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Name</th>
        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Email</th>
        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Role</th>
        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Status</th>
      </tr>
    </thead>
    <tbody class="bg-white divide-y divide-gray-200">
      <tr>
        <td class="px-6 py-4 whitespace-nowrap">John Doe</td>
        <td class="px-6 py-4 whitespace-nowrap">john@example.com</td>
        <td class="px-6 py-4 whitespace-nowrap">Admin</td>
        <td class="px-6 py-4 whitespace-nowrap">Active</td>
      </tr>
    </tbody>
  </table>
</div>
```

### Stacked Card Table (Mobile)

```html
<!-- Table on desktop, stacked cards on mobile -->
<div class="hidden md:block overflow-x-auto">
  <table class="min-w-full">
    <!-- Standard table for desktop -->
    <thead>
      <tr class="border-b">
        <th class="text-left py-3 px-4">Name</th>
        <th class="text-left py-3 px-4">Email</th>
        <th class="text-left py-3 px-4">Role</th>
        <th class="text-left py-3 px-4">Actions</th>
      </tr>
    </thead>
    <tbody>
      <tr class="border-b">
        <td class="py-3 px-4">John Doe</td>
        <td class="py-3 px-4">john@example.com</td>
        <td class="py-3 px-4">Admin</td>
        <td class="py-3 px-4">
          <button class="text-blue-600 hover:text-blue-800">Edit</button>
        </td>
      </tr>
    </tbody>
  </table>
</div>

<!-- Card layout for mobile -->
<div class="md:hidden space-y-4">
  <div class="bg-white rounded-lg shadow p-4">
    <div class="flex justify-between items-start mb-2">
      <h3 class="font-medium">John Doe</h3>
      <span class="text-xs bg-green-100 text-green-800 px-2 py-1 rounded">Admin</span>
    </div>
    <p class="text-sm text-gray-600 mb-3">john@example.com</p>
    <button class="text-sm text-blue-600">Edit</button>
  </div>
</div>
```

### Responsive Column Visibility

```html
<div class="overflow-x-auto">
  <table class="min-w-full">
    <thead>
      <tr class="border-b">
        <th class="py-3 px-4 text-left">Name</th>
        <th class="py-3 px-4 text-left hidden sm:table-cell">Email</th>
        <th class="py-3 px-4 text-left hidden md:table-cell">Phone</th>
        <th class="py-3 px-4 text-left hidden lg:table-cell">Department</th>
        <th class="py-3 px-4 text-left">Status</th>
      </tr>
    </thead>
    <tbody>
      <tr class="border-b">
        <td class="py-3 px-4">
          John Doe
          <!-- Show email under name on mobile -->
          <span class="block text-sm text-gray-500 sm:hidden">john@example.com</span>
        </td>
        <td class="py-3 px-4 hidden sm:table-cell">john@example.com</td>
        <td class="py-3 px-4 hidden md:table-cell">555-1234</td>
        <td class="py-3 px-4 hidden lg:table-cell">Engineering</td>
        <td class="py-3 px-4">
          <span class="inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium bg-green-100 text-green-800">
            Active
          </span>
        </td>
      </tr>
    </tbody>
  </table>
</div>
```

---

## Responsive Forms

Create forms that work well on all devices.

### Responsive Form Layout

```html
<form class="max-w-2xl mx-auto p-4 sm:p-6 lg:p-8">
  <!-- Single column on mobile, two columns on larger screens -->
  <div class="grid grid-cols-1 md:grid-cols-2 gap-4 md:gap-6">
    <div>
      <label class="block text-sm font-medium text-gray-700 mb-1">First Name</label>
      <input type="text" class="w-full px-3 py-2 border border-gray-300 rounded-md focus:ring-blue-500 focus:border-blue-500" />
    </div>
    <div>
      <label class="block text-sm font-medium text-gray-700 mb-1">Last Name</label>
      <input type="text" class="w-full px-3 py-2 border border-gray-300 rounded-md focus:ring-blue-500 focus:border-blue-500" />
    </div>
  </div>

  <!-- Full width fields -->
  <div class="mt-4 md:mt-6">
    <label class="block text-sm font-medium text-gray-700 mb-1">Email</label>
    <input type="email" class="w-full px-3 py-2 border border-gray-300 rounded-md focus:ring-blue-500 focus:border-blue-500" />
  </div>

  <div class="mt-4 md:mt-6">
    <label class="block text-sm font-medium text-gray-700 mb-1">Message</label>
    <textarea rows="4" class="w-full px-3 py-2 border border-gray-300 rounded-md focus:ring-blue-500 focus:border-blue-500"></textarea>
  </div>

  <!-- Responsive button layout -->
  <div class="mt-6 flex flex-col sm:flex-row sm:justify-end gap-3">
    <button type="button" class="w-full sm:w-auto px-4 py-2 border border-gray-300 rounded-md text-gray-700 hover:bg-gray-50 order-2 sm:order-1">
      Cancel
    </button>
    <button type="submit" class="w-full sm:w-auto px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700 order-1 sm:order-2">
      Submit
    </button>
  </div>
</form>
```

### Responsive Input Sizes

```html
<!-- Larger touch targets on mobile -->
<input
  type="text"
  class="
    w-full
    px-3 py-3 sm:py-2
    text-base sm:text-sm
    border border-gray-300 rounded-md
  "
  placeholder="Responsive input"
/>

<!-- Responsive select -->
<select class="
  w-full
  px-3 py-3 sm:py-2
  text-base sm:text-sm
  border border-gray-300 rounded-md
">
  <option>Option 1</option>
  <option>Option 2</option>
</select>
```

### Inline Form (Search)

```html
<!-- Stacked on mobile, inline on desktop -->
<form class="flex flex-col sm:flex-row gap-2 sm:gap-0">
  <input
    type="search"
    placeholder="Search..."
    class="
      flex-1
      px-4 py-2
      border border-gray-300
      rounded-md sm:rounded-r-none
      focus:ring-blue-500 focus:border-blue-500
    "
  />
  <button
    type="submit"
    class="
      px-6 py-2
      bg-blue-600 text-white
      rounded-md sm:rounded-l-none
      hover:bg-blue-700
    "
  >
    Search
  </button>
</form>
```

### Responsive Checkbox/Radio Groups

```html
<!-- Vertical on mobile, horizontal on larger screens -->
<fieldset>
  <legend class="text-sm font-medium text-gray-700 mb-2">Select options</legend>
  <div class="flex flex-col sm:flex-row sm:flex-wrap gap-2 sm:gap-4">
    <label class="flex items-center">
      <input type="checkbox" class="h-4 w-4 text-blue-600 border-gray-300 rounded" />
      <span class="ml-2 text-sm text-gray-700">Option 1</span>
    </label>
    <label class="flex items-center">
      <input type="checkbox" class="h-4 w-4 text-blue-600 border-gray-300 rounded" />
      <span class="ml-2 text-sm text-gray-700">Option 2</span>
    </label>
    <label class="flex items-center">
      <input type="checkbox" class="h-4 w-4 text-blue-600 border-gray-300 rounded" />
      <span class="ml-2 text-sm text-gray-700">Option 3</span>
    </label>
  </div>
</fieldset>
```

---

## Responsive Cards and Layouts

Build card-based layouts that adapt to screen size.

### Basic Responsive Card

```html
<div class="
  bg-white rounded-lg shadow
  p-4 sm:p-6
  flex flex-col sm:flex-row
  gap-4
">
  <img
    src="image.jpg"
    class="
      w-full sm:w-32 md:w-48
      h-48 sm:h-32 md:h-48
      object-cover rounded
    "
    alt="Card image"
  />
  <div class="flex-1">
    <h3 class="text-lg sm:text-xl font-semibold mb-2">Card Title</h3>
    <p class="text-gray-600 text-sm sm:text-base mb-4">
      Card description that provides additional context about this item.
    </p>
    <button class="
      w-full sm:w-auto
      px-4 py-2
      bg-blue-600 text-white rounded
      hover:bg-blue-700
    ">
      Action
    </button>
  </div>
</div>
```

### Responsive Card Grid

```html
<div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-4 md:gap-6">
  <!-- Card 1 -->
  <article class="bg-white rounded-lg shadow overflow-hidden">
    <img src="image1.jpg" class="w-full h-48 object-cover" alt="" />
    <div class="p-4 md:p-6">
      <h3 class="font-semibold text-lg mb-2">Card Title</h3>
      <p class="text-gray-600 text-sm mb-4">Brief description here.</p>
      <a href="#" class="text-blue-600 hover:text-blue-800 text-sm font-medium">
        Learn more &rarr;
      </a>
    </div>
  </article>

  <!-- More cards... -->
</div>
```

### Featured Card Layout

```html
<div class="grid grid-cols-1 lg:grid-cols-2 gap-6">
  <!-- Featured card (larger) -->
  <article class="
    lg:row-span-2
    bg-white rounded-lg shadow overflow-hidden
  ">
    <img src="featured.jpg" class="w-full h-64 lg:h-96 object-cover" alt="" />
    <div class="p-6 lg:p-8">
      <span class="text-xs text-blue-600 font-semibold uppercase tracking-wide">Featured</span>
      <h2 class="text-2xl lg:text-3xl font-bold mt-2 mb-4">Featured Article Title</h2>
      <p class="text-gray-600 mb-6">
        A longer description for the featured article that provides more context.
      </p>
      <a href="#" class="inline-flex items-center text-blue-600 hover:text-blue-800">
        Read more
        <svg class="w-4 h-4 ml-2" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7" />
        </svg>
      </a>
    </div>
  </article>

  <!-- Smaller cards -->
  <article class="bg-white rounded-lg shadow overflow-hidden flex flex-col sm:flex-row lg:flex-col">
    <img src="article1.jpg" class="w-full sm:w-48 lg:w-full h-48 object-cover" alt="" />
    <div class="p-4 lg:p-6 flex-1">
      <h3 class="font-semibold text-lg mb-2">Article Title</h3>
      <p class="text-gray-600 text-sm">Brief description.</p>
    </div>
  </article>

  <article class="bg-white rounded-lg shadow overflow-hidden flex flex-col sm:flex-row lg:flex-col">
    <img src="article2.jpg" class="w-full sm:w-48 lg:w-full h-48 object-cover" alt="" />
    <div class="p-4 lg:p-6 flex-1">
      <h3 class="font-semibold text-lg mb-2">Article Title</h3>
      <p class="text-gray-600 text-sm">Brief description.</p>
    </div>
  </article>
</div>
```

### Responsive Sidebar Layout

```html
<div class="min-h-screen flex flex-col lg:flex-row">
  <!-- Sidebar -->
  <aside class="
    w-full lg:w-64 xl:w-72
    bg-gray-800 text-white
    lg:min-h-screen
    order-2 lg:order-1
  ">
    <div class="p-4 lg:p-6">
      <h2 class="text-xl font-bold mb-4 hidden lg:block">Dashboard</h2>
      <nav class="flex lg:flex-col gap-2 overflow-x-auto lg:overflow-visible">
        <a href="#" class="flex-shrink-0 px-4 py-2 rounded bg-gray-700">Overview</a>
        <a href="#" class="flex-shrink-0 px-4 py-2 rounded hover:bg-gray-700">Analytics</a>
        <a href="#" class="flex-shrink-0 px-4 py-2 rounded hover:bg-gray-700">Reports</a>
        <a href="#" class="flex-shrink-0 px-4 py-2 rounded hover:bg-gray-700">Settings</a>
      </nav>
    </div>
  </aside>

  <!-- Main content -->
  <main class="flex-1 p-4 md:p-6 lg:p-8 order-1 lg:order-2">
    <h1 class="text-2xl md:text-3xl font-bold mb-6">Dashboard Overview</h1>

    <!-- Stats grid -->
    <div class="grid grid-cols-2 lg:grid-cols-4 gap-4 mb-8">
      <div class="bg-white p-4 md:p-6 rounded-lg shadow">
        <p class="text-gray-600 text-sm">Total Users</p>
        <p class="text-2xl md:text-3xl font-bold">12,345</p>
      </div>
      <!-- More stat cards -->
    </div>
  </main>
</div>
```

---

## Media Query Features

Beyond screen size, Tailwind supports other media query features.

### Dark Mode

```html
<!-- System preference-based dark mode -->
<div class="bg-white dark:bg-gray-900 text-gray-900 dark:text-white">
  <h1 class="text-2xl font-bold">Adaptive Content</h1>
  <p class="text-gray-600 dark:text-gray-400">
    This content adapts to system color scheme preference.
  </p>
  <button class="bg-blue-600 dark:bg-blue-500 text-white px-4 py-2 rounded">
    Action
  </button>
</div>
```

```javascript
// tailwind.config.js - Class-based dark mode
module.exports = {
  darkMode: 'class', // or 'media' for system preference
}
```

```html
<!-- Class-based dark mode (add 'dark' class to html/body) -->
<html class="dark">
  <body class="bg-white dark:bg-gray-900">
    <!-- Content -->
  </body>
</html>
```

### Print Styles

```html
<!-- Hide navigation when printing -->
<nav class="print:hidden">
  Navigation content
</nav>

<!-- Full width when printing -->
<main class="max-w-4xl mx-auto print:max-w-none">
  <article class="prose print:prose-sm">
    <!-- Article content -->
  </article>
</main>

<!-- Print-specific styles -->
<div class="bg-blue-100 print:bg-transparent print:border print:border-gray-300 p-4">
  <h2 class="text-blue-800 print:text-black">Section Title</h2>
</div>

<!-- Force page breaks -->
<section class="print:break-before-page">
  New page content
</section>

<div class="print:break-inside-avoid">
  Keep this content together on one page
</div>
```

### Orientation (Portrait/Landscape)

```html
<!-- Different layouts based on device orientation -->
<div class="
  grid grid-cols-1
  portrait:grid-cols-1
  landscape:grid-cols-2
  gap-4
">
  <div class="bg-blue-100 p-4">Column 1</div>
  <div class="bg-blue-200 p-4">Column 2</div>
</div>

<!-- Adjust spacing for orientation -->
<section class="
  py-8
  portrait:py-12
  landscape:py-4
">
  Orientation-aware content
</section>
```

### Motion Preferences

```html
<!-- Respect user's reduced motion preference -->
<button class="
  transform transition-transform duration-300
  hover:scale-105
  motion-reduce:transform-none
  motion-reduce:transition-none
">
  Animated Button
</button>

<!-- Alternative animations for reduced motion -->
<div class="
  animate-bounce
  motion-reduce:animate-none
  motion-reduce:hover:underline
">
  Bouncing element (static for reduced motion users)
</div>
```

### Pointer and Hover Capabilities

```javascript
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      screens: {
        'touch': { 'raw': '(pointer: coarse)' },
        'mouse': { 'raw': '(pointer: fine)' },
        'can-hover': { 'raw': '(hover: hover)' },
        'no-hover': { 'raw': '(hover: none)' },
      },
    },
  },
}
```

```html
<!-- Larger touch targets on touch devices -->
<button class="p-2 mouse:p-1 touch:p-4 touch:text-lg">
  Touch-friendly button
</button>

<!-- Hover effects only on devices that support hover -->
<a href="#" class="text-blue-600 can-hover:hover:text-blue-800 can-hover:hover:underline">
  Link with hover effect
</a>
```

### Contrast Preferences

```html
<!-- High contrast mode support -->
<div class="
  bg-gray-100
  contrast-more:bg-white
  contrast-more:border-2
  contrast-more:border-black
">
  <p class="
    text-gray-600
    contrast-more:text-black
  ">
    High contrast accessible content
  </p>
</div>
```

---

## Arbitrary Breakpoints

Use arbitrary values for one-off responsive needs.

### Arbitrary Min-Width

```html
<!-- Custom breakpoint at 900px -->
<div class="hidden min-[900px]:block">
  Visible at 900px and above
</div>

<!-- Multiple arbitrary breakpoints -->
<div class="
  text-sm
  min-[400px]:text-base
  min-[600px]:text-lg
  min-[900px]:text-xl
  min-[1200px]:text-2xl
">
  Fine-grained responsive text
</div>
```

### Arbitrary Max-Width

```html
<!-- Custom max-width breakpoint -->
<div class="block max-[599px]:hidden">
  Hidden below 600px
</div>

<!-- Combine with min-width for ranges -->
<div class="hidden min-[600px]:block max-[899px]:block min-[900px]:hidden">
  Only visible between 600px and 899px
</div>
```

### Complex Arbitrary Queries

```html
<!-- Specific pixel breakpoints -->
<div class="
  grid
  grid-cols-1
  min-[480px]:grid-cols-2
  min-[720px]:grid-cols-3
  min-[960px]:grid-cols-4
  min-[1200px]:grid-cols-5
">
  Precise grid breakpoints
</div>

<!-- Arbitrary value with other utilities -->
<div class="
  w-full
  min-[500px]:w-[450px]
  min-[800px]:w-[700px]
  min-[1100px]:w-[1000px]
  mx-auto
">
  Custom width at custom breakpoints
</div>
```

### Arbitrary with Container Queries

```html
<div class="@container">
  <div class="
    text-sm
    @[200px]:text-base
    @[400px]:text-lg
    @[600px]:text-xl
  ">
    Arbitrary container query breakpoints
  </div>
</div>
```

---

## Best Practices

Guidelines for effective responsive design with Tailwind CSS.

### 1. Start Mobile-First

Always design for mobile first, then add complexity for larger screens:

```html
<!-- Good: Mobile-first approach -->
<div class="flex flex-col md:flex-row">
  <!-- Starts stacked, becomes horizontal -->
</div>

<!-- Avoid: Desktop-first thinking -->
<!-- Don't think "hide on mobile", think "show on desktop" -->
```

### 2. Use Semantic Breakpoints

Choose breakpoints based on content, not devices:

```html
<!-- Good: Breakpoints based on when layout needs to change -->
<div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3">
  <!-- Changes based on content needs -->
</div>

<!-- Avoid: Arbitrary device-specific breakpoints -->
<!-- Don't target "iPhone 12" specifically -->
```

### 3. Test at All Breakpoints

```html
<!-- Ensure smooth transitions between breakpoints -->
<div class="
  text-base      /* 0-639px */
  sm:text-lg     /* 640-767px */
  md:text-xl     /* 768-1023px */
  lg:text-2xl    /* 1024-1279px */
  xl:text-3xl    /* 1280px+ */
">
  Test text at each breakpoint
</div>
```

### 4. Avoid Over-Specifying

```html
<!-- Good: Only specify what changes -->
<div class="p-4 md:p-6 lg:p-8">
  <!-- Padding increases at specific points -->
</div>

<!-- Avoid: Unnecessary repetition -->
<div class="p-4 sm:p-4 md:p-6 lg:p-8 xl:p-8 2xl:p-8">
  <!-- sm:p-4 and xl:p-8, 2xl:p-8 are redundant -->
</div>
```

### 5. Use Container Queries for Components

```html
<!-- Good: Component adapts to container, not viewport -->
<div class="@container">
  <article class="@md:flex @md:gap-4">
    <img class="w-full @md:w-48" />
    <div class="@md:flex-1">Content</div>
  </article>
</div>
```

### 6. Prioritize Content Hierarchy

```html
<!-- Good: Important content first in source order -->
<div class="flex flex-col lg:flex-row">
  <main class="flex-1 order-2 lg:order-1">Main content first in DOM</main>
  <aside class="lg:w-64 order-1 lg:order-2">Sidebar</aside>
</div>
```

### 7. Use Consistent Spacing Scale

```html
<!-- Good: Consistent spacing increments -->
<section class="py-8 md:py-12 lg:py-16">
  <div class="space-y-4 md:space-y-6 lg:space-y-8">
    <!-- Consistent 4-unit increments -->
  </div>
</section>
```

### 8. Consider Touch Targets

```html
<!-- Good: Adequate touch targets on mobile -->
<button class="
  min-h-[44px] min-w-[44px]
  p-3 sm:p-2
  text-base sm:text-sm
">
  Touch-friendly button
</button>

<a href="#" class="
  block py-3 sm:py-2
  text-base sm:text-sm
">
  Touch-friendly link
</a>
```

### 9. Optimize Images Responsively

```html
<!-- Good: Responsive images with appropriate sizes -->
<img
  src="image.jpg"
  srcset="image-sm.jpg 640w, image-md.jpg 768w, image-lg.jpg 1024w"
  sizes="(max-width: 640px) 100vw, (max-width: 1024px) 50vw, 33vw"
  class="w-full h-auto"
  alt="Responsive image"
/>
```

### 10. Document Breakpoint Decisions

```html
<!-- Add comments for complex responsive logic -->
<div class="
  grid
  grid-cols-1     /* Mobile: single column */
  sm:grid-cols-2  /* Tablet: 2 columns (cards still readable) */
  lg:grid-cols-3  /* Desktop: 3 columns (optimal for sidebar layout) */
  xl:grid-cols-4  /* Wide: 4 columns (max for this content) */
  gap-4 md:gap-6
">
```

### 11. Use Responsive Utilities Consistently

```javascript
// tailwind.config.js - Define reusable responsive patterns
module.exports = {
  theme: {
    extend: {
      spacing: {
        'section': '2rem',
        'section-md': '3rem',
        'section-lg': '4rem',
      },
    },
  },
}
```

### 12. Performance Considerations

```html
<!-- Avoid unnecessary DOM elements for responsive hiding -->
<!-- Bad: Duplicate content -->
<div class="md:hidden">Mobile content</div>
<div class="hidden md:block">Desktop content (same as mobile)</div>

<!-- Good: Single element with responsive styles -->
<div class="text-sm md:text-base p-4 md:p-6">
  Responsive content
</div>
```

---

## Quick Reference

### Breakpoint Prefixes

| Prefix | Min Width | Usage |
|--------|-----------|-------|
| (none) | 0px | Base/mobile styles |
| `sm:` | 640px | Small screens and up |
| `md:` | 768px | Medium screens and up |
| `lg:` | 1024px | Large screens and up |
| `xl:` | 1280px | Extra-large screens and up |
| `2xl:` | 1536px | 2XL screens and up |

### Common Patterns

```html
<!-- Responsive grid -->
grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4

<!-- Responsive flex direction -->
flex flex-col md:flex-row

<!-- Show/hide -->
hidden md:block    /* Hidden on mobile, visible on md+ */
block md:hidden    /* Visible on mobile, hidden on md+ */

<!-- Responsive text -->
text-sm md:text-base lg:text-lg

<!-- Responsive padding -->
p-4 md:p-6 lg:p-8

<!-- Responsive max-width container -->
max-w-7xl mx-auto px-4 sm:px-6 lg:px-8
```

### Container Query Prefixes

| Prefix | Min Width |
|--------|-----------|
| `@xs` | 320px |
| `@sm` | 384px |
| `@md` | 448px |
| `@lg` | 512px |
| `@xl` | 576px |
| `@2xl` | 672px |

### Media Feature Prefixes

| Prefix | Description |
|--------|-------------|
| `dark:` | Dark mode |
| `print:` | Print styles |
| `portrait:` | Portrait orientation |
| `landscape:` | Landscape orientation |
| `motion-reduce:` | Reduced motion preference |
| `motion-safe:` | No motion preference |
| `contrast-more:` | High contrast preference |

---

## Resources

- [Tailwind CSS Responsive Design Documentation](https://tailwindcss.com/docs/responsive-design)
- [Tailwind CSS Container Queries](https://tailwindcss.com/docs/container-queries)
- [Tailwind CSS Dark Mode](https://tailwindcss.com/docs/dark-mode)
- [Tailwind CSS Customizing Screens](https://tailwindcss.com/docs/screens)
