# Tailwind CSS Utilities - Comprehensive Reference

A complete guide to Tailwind CSS utility classes covering all major categories.
Based on official Tailwind CSS documentation.

---

## Table of Contents

1. [Spacing](#spacing)
2. [Sizing](#sizing)
3. [Typography](#typography)
4. [Colors](#colors)
5. [Flexbox](#flexbox)
6. [Grid](#grid)
7. [Layout](#layout)
8. [Borders](#borders)
9. [Effects](#effects)
10. [Transitions and Animations](#transitions-and-animations)
11. [Transforms](#transforms)
12. [Filters](#filters)
13. [Interactivity](#interactivity)
14. [Accessibility](#accessibility)
15. [Best Practices and Common Patterns](#best-practices-and-common-patterns)

---

## Spacing

Spacing utilities control padding, margin, space-between, and gap properties.

### Spacing Scale Reference

| Class | Value   | Pixels |
|-------|---------|--------|
| 0     | 0       | 0px    |
| px    | 1px     | 1px    |
| 0.5   | 0.125rem| 2px    |
| 1     | 0.25rem | 4px    |
| 1.5   | 0.375rem| 6px    |
| 2     | 0.5rem  | 8px    |
| 2.5   | 0.625rem| 10px   |
| 3     | 0.75rem | 12px   |
| 3.5   | 0.875rem| 14px   |
| 4     | 1rem    | 16px   |
| 5     | 1.25rem | 20px   |
| 6     | 1.5rem  | 24px   |
| 7     | 1.75rem | 28px   |
| 8     | 2rem    | 32px   |
| 9     | 2.25rem | 36px   |
| 10    | 2.5rem  | 40px   |
| 11    | 2.75rem | 44px   |
| 12    | 3rem    | 48px   |
| 14    | 3.5rem  | 56px   |
| 16    | 4rem    | 64px   |
| 20    | 5rem    | 80px   |
| 24    | 6rem    | 96px   |
| 28    | 7rem    | 112px  |
| 32    | 8rem    | 128px  |
| 36    | 9rem    | 144px  |
| 40    | 10rem   | 160px  |
| 44    | 11rem   | 176px  |
| 48    | 12rem   | 192px  |
| 52    | 13rem   | 208px  |
| 56    | 14rem   | 224px  |
| 60    | 15rem   | 240px  |
| 64    | 16rem   | 256px  |
| 72    | 18rem   | 288px  |
| 80    | 20rem   | 320px  |
| 96    | 24rem   | 384px  |

### Padding

```html
<!-- All sides -->
<div class="p-0">padding: 0</div>
<div class="p-1">padding: 0.25rem (4px)</div>
<div class="p-2">padding: 0.5rem (8px)</div>
<div class="p-4">padding: 1rem (16px)</div>
<div class="p-6">padding: 1.5rem (24px)</div>
<div class="p-8">padding: 2rem (32px)</div>
<div class="p-12">padding: 3rem (48px)</div>
<div class="p-16">padding: 4rem (64px)</div>

<!-- Horizontal (left and right) -->
<div class="px-0">padding-left: 0; padding-right: 0</div>
<div class="px-2">padding-left: 0.5rem; padding-right: 0.5rem</div>
<div class="px-4">padding-left: 1rem; padding-right: 1rem</div>
<div class="px-8">padding-left: 2rem; padding-right: 2rem</div>

<!-- Vertical (top and bottom) -->
<div class="py-0">padding-top: 0; padding-bottom: 0</div>
<div class="py-2">padding-top: 0.5rem; padding-bottom: 0.5rem</div>
<div class="py-4">padding-top: 1rem; padding-bottom: 1rem</div>
<div class="py-8">padding-top: 2rem; padding-bottom: 2rem</div>

<!-- Individual sides -->
<div class="pt-4">padding-top: 1rem</div>
<div class="pr-4">padding-right: 1rem</div>
<div class="pb-4">padding-bottom: 1rem</div>
<div class="pl-4">padding-left: 1rem</div>

<!-- Logical properties (start/end for RTL support) -->
<div class="ps-4">padding-inline-start: 1rem</div>
<div class="pe-4">padding-inline-end: 1rem</div>
```

### Margin

```html
<!-- All sides -->
<div class="m-0">margin: 0</div>
<div class="m-2">margin: 0.5rem</div>
<div class="m-4">margin: 1rem</div>
<div class="m-8">margin: 2rem</div>
<div class="m-auto">margin: auto</div>

<!-- Horizontal (left and right) -->
<div class="mx-0">margin-left: 0; margin-right: 0</div>
<div class="mx-4">margin-left: 1rem; margin-right: 1rem</div>
<div class="mx-auto">margin-left: auto; margin-right: auto (centering)</div>

<!-- Vertical (top and bottom) -->
<div class="my-0">margin-top: 0; margin-bottom: 0</div>
<div class="my-4">margin-top: 1rem; margin-bottom: 1rem</div>
<div class="my-auto">margin-top: auto; margin-bottom: auto</div>

<!-- Individual sides -->
<div class="mt-4">margin-top: 1rem</div>
<div class="mr-4">margin-right: 1rem</div>
<div class="mb-4">margin-bottom: 1rem</div>
<div class="ml-4">margin-left: 1rem</div>

<!-- Logical properties -->
<div class="ms-4">margin-inline-start: 1rem</div>
<div class="me-4">margin-inline-end: 1rem</div>

<!-- Negative margins -->
<div class="-m-4">margin: -1rem</div>
<div class="-mt-2">margin-top: -0.5rem</div>
<div class="-mx-4">margin-left: -1rem; margin-right: -1rem</div>
```

### Space Between

Control spacing between child elements without affecting outer spacing.

```html
<!-- Horizontal space between children -->
<div class="flex space-x-0">No horizontal space</div>
<div class="flex space-x-2">0.5rem gap horizontally</div>
<div class="flex space-x-4">1rem gap horizontally</div>
<div class="flex space-x-8">2rem gap horizontally</div>

<!-- Vertical space between children -->
<div class="flex flex-col space-y-0">No vertical space</div>
<div class="flex flex-col space-y-2">0.5rem gap vertically</div>
<div class="flex flex-col space-y-4">1rem gap vertically</div>
<div class="flex flex-col space-y-8">2rem gap vertically</div>

<!-- Reverse space (for flex-row-reverse or flex-col-reverse) -->
<div class="flex flex-row-reverse space-x-4 space-x-reverse">
  Reversed horizontal spacing
</div>
<div class="flex flex-col-reverse space-y-4 space-y-reverse">
  Reversed vertical spacing
</div>

<!-- Practical example: Button group -->
<div class="flex space-x-2">
  <button class="px-4 py-2">Button 1</button>
  <button class="px-4 py-2">Button 2</button>
  <button class="px-4 py-2">Button 3</button>
</div>

<!-- Practical example: Stacked form fields -->
<div class="space-y-4">
  <input type="text" placeholder="Name">
  <input type="email" placeholder="Email">
  <input type="password" placeholder="Password">
</div>
```

### Gap

Use gap utilities with flexbox and grid layouts.

```html
<!-- Gap (all directions) -->
<div class="flex gap-0">gap: 0</div>
<div class="flex gap-1">gap: 0.25rem</div>
<div class="flex gap-2">gap: 0.5rem</div>
<div class="flex gap-4">gap: 1rem</div>
<div class="flex gap-6">gap: 1.5rem</div>
<div class="flex gap-8">gap: 2rem</div>

<!-- Row gap (horizontal in grid) -->
<div class="grid gap-x-4">column-gap: 1rem</div>
<div class="grid gap-x-8">column-gap: 2rem</div>

<!-- Column gap (vertical in grid) -->
<div class="grid gap-y-4">row-gap: 1rem</div>
<div class="grid gap-y-8">row-gap: 2rem</div>

<!-- Combined gaps -->
<div class="grid gap-x-4 gap-y-8">Different x and y gaps</div>

<!-- Practical example: Card grid -->
<div class="grid grid-cols-3 gap-6">
  <div class="bg-white p-4 shadow">Card 1</div>
  <div class="bg-white p-4 shadow">Card 2</div>
  <div class="bg-white p-4 shadow">Card 3</div>
</div>
```

---

## Sizing

Control width, height, and min/max dimensions.

### Width

```html
<!-- Fixed widths -->
<div class="w-0">width: 0</div>
<div class="w-1">width: 0.25rem (4px)</div>
<div class="w-2">width: 0.5rem (8px)</div>
<div class="w-4">width: 1rem (16px)</div>
<div class="w-8">width: 2rem (32px)</div>
<div class="w-16">width: 4rem (64px)</div>
<div class="w-32">width: 8rem (128px)</div>
<div class="w-64">width: 16rem (256px)</div>
<div class="w-96">width: 24rem (384px)</div>

<!-- Percentage widths -->
<div class="w-1/2">width: 50%</div>
<div class="w-1/3">width: 33.333%</div>
<div class="w-2/3">width: 66.667%</div>
<div class="w-1/4">width: 25%</div>
<div class="w-2/4">width: 50%</div>
<div class="w-3/4">width: 75%</div>
<div class="w-1/5">width: 20%</div>
<div class="w-2/5">width: 40%</div>
<div class="w-3/5">width: 60%</div>
<div class="w-4/5">width: 80%</div>
<div class="w-1/6">width: 16.667%</div>
<div class="w-5/6">width: 83.333%</div>
<div class="w-1/12">width: 8.333%</div>
<div class="w-11/12">width: 91.667%</div>

<!-- Special widths -->
<div class="w-auto">width: auto</div>
<div class="w-full">width: 100%</div>
<div class="w-screen">width: 100vw</div>
<div class="w-svw">width: 100svw (small viewport)</div>
<div class="w-lvw">width: 100lvw (large viewport)</div>
<div class="w-dvw">width: 100dvw (dynamic viewport)</div>
<div class="w-min">width: min-content</div>
<div class="w-max">width: max-content</div>
<div class="w-fit">width: fit-content</div>

<!-- Arbitrary values -->
<div class="w-[300px]">width: 300px</div>
<div class="w-[50vw]">width: 50vw</div>
```

### Min-Width

```html
<div class="min-w-0">min-width: 0</div>
<div class="min-w-full">min-width: 100%</div>
<div class="min-w-min">min-width: min-content</div>
<div class="min-w-max">min-width: max-content</div>
<div class="min-w-fit">min-width: fit-content</div>

<!-- Arbitrary values -->
<div class="min-w-[200px]">min-width: 200px</div>
```

### Max-Width

```html
<div class="max-w-0">max-width: 0</div>
<div class="max-w-none">max-width: none</div>
<div class="max-w-xs">max-width: 20rem (320px)</div>
<div class="max-w-sm">max-width: 24rem (384px)</div>
<div class="max-w-md">max-width: 28rem (448px)</div>
<div class="max-w-lg">max-width: 32rem (512px)</div>
<div class="max-w-xl">max-width: 36rem (576px)</div>
<div class="max-w-2xl">max-width: 42rem (672px)</div>
<div class="max-w-3xl">max-width: 48rem (768px)</div>
<div class="max-w-4xl">max-width: 56rem (896px)</div>
<div class="max-w-5xl">max-width: 64rem (1024px)</div>
<div class="max-w-6xl">max-width: 72rem (1152px)</div>
<div class="max-w-7xl">max-width: 80rem (1280px)</div>
<div class="max-w-full">max-width: 100%</div>
<div class="max-w-min">max-width: min-content</div>
<div class="max-w-max">max-width: max-content</div>
<div class="max-w-fit">max-width: fit-content</div>
<div class="max-w-prose">max-width: 65ch (readable text)</div>
<div class="max-w-screen-sm">max-width: 640px</div>
<div class="max-w-screen-md">max-width: 768px</div>
<div class="max-w-screen-lg">max-width: 1024px</div>
<div class="max-w-screen-xl">max-width: 1280px</div>
<div class="max-w-screen-2xl">max-width: 1536px</div>
```

### Height

```html
<!-- Fixed heights -->
<div class="h-0">height: 0</div>
<div class="h-1">height: 0.25rem</div>
<div class="h-4">height: 1rem</div>
<div class="h-8">height: 2rem</div>
<div class="h-16">height: 4rem</div>
<div class="h-32">height: 8rem</div>
<div class="h-64">height: 16rem</div>
<div class="h-96">height: 24rem</div>

<!-- Percentage heights -->
<div class="h-1/2">height: 50%</div>
<div class="h-1/3">height: 33.333%</div>
<div class="h-2/3">height: 66.667%</div>
<div class="h-1/4">height: 25%</div>
<div class="h-3/4">height: 75%</div>
<div class="h-1/5">height: 20%</div>
<div class="h-1/6">height: 16.667%</div>

<!-- Special heights -->
<div class="h-auto">height: auto</div>
<div class="h-full">height: 100%</div>
<div class="h-screen">height: 100vh</div>
<div class="h-svh">height: 100svh (small viewport)</div>
<div class="h-lvh">height: 100lvh (large viewport)</div>
<div class="h-dvh">height: 100dvh (dynamic viewport)</div>
<div class="h-min">height: min-content</div>
<div class="h-max">height: max-content</div>
<div class="h-fit">height: fit-content</div>
```

### Min-Height

```html
<div class="min-h-0">min-height: 0</div>
<div class="min-h-full">min-height: 100%</div>
<div class="min-h-screen">min-height: 100vh</div>
<div class="min-h-svh">min-height: 100svh</div>
<div class="min-h-lvh">min-height: 100lvh</div>
<div class="min-h-dvh">min-height: 100dvh</div>
<div class="min-h-min">min-height: min-content</div>
<div class="min-h-max">min-height: max-content</div>
<div class="min-h-fit">min-height: fit-content</div>
```

### Max-Height

```html
<div class="max-h-0">max-height: 0</div>
<div class="max-h-full">max-height: 100%</div>
<div class="max-h-screen">max-height: 100vh</div>
<div class="max-h-svh">max-height: 100svh</div>
<div class="max-h-lvh">max-height: 100lvh</div>
<div class="max-h-dvh">max-height: 100dvh</div>
<div class="max-h-min">max-height: min-content</div>
<div class="max-h-max">max-height: max-content</div>
<div class="max-h-fit">max-height: fit-content</div>
<div class="max-h-none">max-height: none</div>

<!-- Fixed max heights -->
<div class="max-h-48">max-height: 12rem</div>
<div class="max-h-64">max-height: 16rem</div>
<div class="max-h-96">max-height: 24rem</div>
```

### Size (Width and Height Together)

```html
<div class="size-0">width: 0; height: 0</div>
<div class="size-4">width: 1rem; height: 1rem</div>
<div class="size-8">width: 2rem; height: 2rem</div>
<div class="size-16">width: 4rem; height: 4rem</div>
<div class="size-full">width: 100%; height: 100%</div>

<!-- Useful for icons and avatars -->
<div class="size-6">24px square (icon)</div>
<div class="size-10">40px square (avatar)</div>
<div class="size-12">48px square (larger avatar)</div>
```

---

## Typography

Control font properties, text styling, and text layout.

### Font Size

```html
<p class="text-xs">font-size: 0.75rem (12px); line-height: 1rem</p>
<p class="text-sm">font-size: 0.875rem (14px); line-height: 1.25rem</p>
<p class="text-base">font-size: 1rem (16px); line-height: 1.5rem</p>
<p class="text-lg">font-size: 1.125rem (18px); line-height: 1.75rem</p>
<p class="text-xl">font-size: 1.25rem (20px); line-height: 1.75rem</p>
<p class="text-2xl">font-size: 1.5rem (24px); line-height: 2rem</p>
<p class="text-3xl">font-size: 1.875rem (30px); line-height: 2.25rem</p>
<p class="text-4xl">font-size: 2.25rem (36px); line-height: 2.5rem</p>
<p class="text-5xl">font-size: 3rem (48px); line-height: 1</p>
<p class="text-6xl">font-size: 3.75rem (60px); line-height: 1</p>
<p class="text-7xl">font-size: 4.5rem (72px); line-height: 1</p>
<p class="text-8xl">font-size: 6rem (96px); line-height: 1</p>
<p class="text-9xl">font-size: 8rem (128px); line-height: 1</p>
```

### Font Weight

```html
<p class="font-thin">font-weight: 100</p>
<p class="font-extralight">font-weight: 200</p>
<p class="font-light">font-weight: 300</p>
<p class="font-normal">font-weight: 400</p>
<p class="font-medium">font-weight: 500</p>
<p class="font-semibold">font-weight: 600</p>
<p class="font-bold">font-weight: 700</p>
<p class="font-extrabold">font-weight: 800</p>
<p class="font-black">font-weight: 900</p>
```

### Font Family

```html
<p class="font-sans">font-family: ui-sans-serif, system-ui, sans-serif</p>
<p class="font-serif">font-family: ui-serif, Georgia, serif</p>
<p class="font-mono">font-family: ui-monospace, monospace</p>
```

### Font Style

```html
<p class="italic">font-style: italic</p>
<p class="not-italic">font-style: normal</p>
```

### Line Height

```html
<p class="leading-none">line-height: 1</p>
<p class="leading-tight">line-height: 1.25</p>
<p class="leading-snug">line-height: 1.375</p>
<p class="leading-normal">line-height: 1.5</p>
<p class="leading-relaxed">line-height: 1.625</p>
<p class="leading-loose">line-height: 2</p>

<!-- Fixed line heights -->
<p class="leading-3">line-height: 0.75rem</p>
<p class="leading-4">line-height: 1rem</p>
<p class="leading-5">line-height: 1.25rem</p>
<p class="leading-6">line-height: 1.5rem</p>
<p class="leading-7">line-height: 1.75rem</p>
<p class="leading-8">line-height: 2rem</p>
<p class="leading-9">line-height: 2.25rem</p>
<p class="leading-10">line-height: 2.5rem</p>
```

### Letter Spacing

```html
<p class="tracking-tighter">letter-spacing: -0.05em</p>
<p class="tracking-tight">letter-spacing: -0.025em</p>
<p class="tracking-normal">letter-spacing: 0em</p>
<p class="tracking-wide">letter-spacing: 0.025em</p>
<p class="tracking-wider">letter-spacing: 0.05em</p>
<p class="tracking-widest">letter-spacing: 0.1em</p>
```

### Text Alignment

```html
<p class="text-left">text-align: left</p>
<p class="text-center">text-align: center</p>
<p class="text-right">text-align: right</p>
<p class="text-justify">text-align: justify</p>
<p class="text-start">text-align: start</p>
<p class="text-end">text-align: end</p>
```

### Text Decoration

```html
<p class="underline">text-decoration: underline</p>
<p class="overline">text-decoration: overline</p>
<p class="line-through">text-decoration: line-through</p>
<p class="no-underline">text-decoration: none</p>

<!-- Decoration style -->
<p class="decoration-solid">text-decoration-style: solid</p>
<p class="decoration-double">text-decoration-style: double</p>
<p class="decoration-dotted">text-decoration-style: dotted</p>
<p class="decoration-dashed">text-decoration-style: dashed</p>
<p class="decoration-wavy">text-decoration-style: wavy</p>

<!-- Decoration thickness -->
<p class="decoration-auto">text-decoration-thickness: auto</p>
<p class="decoration-from-font">text-decoration-thickness: from-font</p>
<p class="decoration-0">text-decoration-thickness: 0</p>
<p class="decoration-1">text-decoration-thickness: 1px</p>
<p class="decoration-2">text-decoration-thickness: 2px</p>
<p class="decoration-4">text-decoration-thickness: 4px</p>
<p class="decoration-8">text-decoration-thickness: 8px</p>

<!-- Underline offset -->
<p class="underline-offset-auto">text-underline-offset: auto</p>
<p class="underline-offset-0">text-underline-offset: 0</p>
<p class="underline-offset-1">text-underline-offset: 1px</p>
<p class="underline-offset-2">text-underline-offset: 2px</p>
<p class="underline-offset-4">text-underline-offset: 4px</p>
<p class="underline-offset-8">text-underline-offset: 8px</p>
```

### Text Transform

```html
<p class="uppercase">text-transform: uppercase</p>
<p class="lowercase">text-transform: lowercase</p>
<p class="capitalize">text-transform: capitalize</p>
<p class="normal-case">text-transform: none</p>
```

### Text Overflow

```html
<p class="truncate">Truncate with ellipsis (overflow: hidden; text-overflow: ellipsis; white-space: nowrap)</p>
<p class="text-ellipsis">text-overflow: ellipsis</p>
<p class="text-clip">text-overflow: clip</p>
```

### Text Wrap

```html
<p class="text-wrap">text-wrap: wrap</p>
<p class="text-nowrap">text-wrap: nowrap</p>
<p class="text-balance">text-wrap: balance</p>
<p class="text-pretty">text-wrap: pretty</p>
```

### Whitespace

```html
<p class="whitespace-normal">white-space: normal</p>
<p class="whitespace-nowrap">white-space: nowrap</p>
<p class="whitespace-pre">white-space: pre</p>
<p class="whitespace-pre-line">white-space: pre-line</p>
<p class="whitespace-pre-wrap">white-space: pre-wrap</p>
<p class="whitespace-break-spaces">white-space: break-spaces</p>
```

### Word Break

```html
<p class="break-normal">overflow-wrap: normal; word-break: normal</p>
<p class="break-words">overflow-wrap: break-word</p>
<p class="break-all">word-break: break-all</p>
<p class="break-keep">word-break: keep-all</p>
```

### Hyphens

```html
<p class="hyphens-none">hyphens: none</p>
<p class="hyphens-manual">hyphens: manual</p>
<p class="hyphens-auto">hyphens: auto</p>
```

### List Style

```html
<ul class="list-none">list-style-type: none</ul>
<ul class="list-disc">list-style-type: disc</ul>
<ul class="list-decimal">list-style-type: decimal</ul>

<!-- List position -->
<ul class="list-inside">list-style-position: inside</ul>
<ul class="list-outside">list-style-position: outside</ul>
```

---

## Colors

Control text, background, border, and ring colors.

### Color Palette

Tailwind includes a comprehensive color palette with shades from 50 (lightest) to 950 (darkest).

Available colors: slate, gray, zinc, neutral, stone, red, orange, amber, yellow, lime, green, emerald, teal, cyan, sky, blue, indigo, violet, purple, fuchsia, pink, rose

### Text Color

```html
<!-- Inherit and current -->
<p class="text-inherit">color: inherit</p>
<p class="text-current">color: currentColor</p>
<p class="text-transparent">color: transparent</p>

<!-- Black and white -->
<p class="text-black">color: #000</p>
<p class="text-white">color: #fff</p>

<!-- Gray scale -->
<p class="text-gray-50">Lightest gray</p>
<p class="text-gray-100">Very light gray</p>
<p class="text-gray-200">Light gray</p>
<p class="text-gray-300">Light-medium gray</p>
<p class="text-gray-400">Medium-light gray</p>
<p class="text-gray-500">Medium gray</p>
<p class="text-gray-600">Medium-dark gray</p>
<p class="text-gray-700">Dark gray</p>
<p class="text-gray-800">Very dark gray</p>
<p class="text-gray-900">Darkest gray</p>
<p class="text-gray-950">Near black gray</p>

<!-- Colors with shades -->
<p class="text-red-500">Red</p>
<p class="text-blue-500">Blue</p>
<p class="text-green-500">Green</p>
<p class="text-yellow-500">Yellow</p>
<p class="text-purple-500">Purple</p>
<p class="text-pink-500">Pink</p>
<p class="text-indigo-500">Indigo</p>
<p class="text-teal-500">Teal</p>

<!-- With opacity -->
<p class="text-blue-500/50">50% opacity blue</p>
<p class="text-blue-500/75">75% opacity blue</p>
<p class="text-black/50">50% opacity black</p>
```

### Background Color

```html
<!-- Inherit and transparent -->
<div class="bg-inherit">background-color: inherit</div>
<div class="bg-current">background-color: currentColor</div>
<div class="bg-transparent">background-color: transparent</div>

<!-- Black and white -->
<div class="bg-black">Black background</div>
<div class="bg-white">White background</div>

<!-- Gray backgrounds -->
<div class="bg-gray-50">Very light gray</div>
<div class="bg-gray-100">Light gray</div>
<div class="bg-gray-200">Soft gray</div>
<div class="bg-gray-300">Medium-light gray</div>
<div class="bg-gray-400">Medium gray</div>
<div class="bg-gray-500">Gray</div>
<div class="bg-gray-600">Medium-dark gray</div>
<div class="bg-gray-700">Dark gray</div>
<div class="bg-gray-800">Very dark gray</div>
<div class="bg-gray-900">Near black</div>

<!-- Color backgrounds -->
<div class="bg-red-500">Red</div>
<div class="bg-blue-500">Blue</div>
<div class="bg-green-500">Green</div>
<div class="bg-yellow-500">Yellow</div>
<div class="bg-purple-500">Purple</div>
<div class="bg-pink-500">Pink</div>

<!-- With opacity -->
<div class="bg-blue-500/50">50% opacity</div>
<div class="bg-black/25">25% opacity black</div>
```

### Background Gradient

```html
<!-- Gradient direction -->
<div class="bg-gradient-to-t">To top</div>
<div class="bg-gradient-to-tr">To top right</div>
<div class="bg-gradient-to-r">To right</div>
<div class="bg-gradient-to-br">To bottom right</div>
<div class="bg-gradient-to-b">To bottom</div>
<div class="bg-gradient-to-bl">To bottom left</div>
<div class="bg-gradient-to-l">To left</div>
<div class="bg-gradient-to-tl">To top left</div>

<!-- Gradient stops -->
<div class="bg-gradient-to-r from-blue-500">From blue</div>
<div class="bg-gradient-to-r from-blue-500 to-purple-500">Blue to purple</div>
<div class="bg-gradient-to-r from-blue-500 via-purple-500 to-pink-500">Blue via purple to pink</div>

<!-- Stop positions -->
<div class="bg-gradient-to-r from-blue-500 from-10%">From starts at 10%</div>
<div class="bg-gradient-to-r from-blue-500 via-purple-500 via-30% to-pink-500 to-90%">Custom positions</div>
```

### Border Color

```html
<div class="border border-inherit">border-color: inherit</div>
<div class="border border-current">border-color: currentColor</div>
<div class="border border-transparent">border-color: transparent</div>

<!-- Black and white -->
<div class="border border-black">Black border</div>
<div class="border border-white">White border</div>

<!-- Gray borders -->
<div class="border border-gray-200">Light gray border</div>
<div class="border border-gray-300">Medium-light gray border</div>
<div class="border border-gray-400">Medium gray border</div>
<div class="border border-gray-500">Gray border</div>

<!-- Color borders -->
<div class="border border-red-500">Red border</div>
<div class="border border-blue-500">Blue border</div>
<div class="border border-green-500">Green border</div>

<!-- Per-side border colors -->
<div class="border border-t-red-500 border-r-blue-500 border-b-green-500 border-l-yellow-500">
  Multi-color borders
</div>

<!-- With opacity -->
<div class="border border-blue-500/50">50% opacity border</div>
```

### Ring (Box Shadow Outline)

```html
<!-- Ring width -->
<div class="ring-0">No ring</div>
<div class="ring-1">1px ring</div>
<div class="ring-2">2px ring</div>
<div class="ring">3px ring (default)</div>
<div class="ring-4">4px ring</div>
<div class="ring-8">8px ring</div>
<div class="ring-inset">Inset ring</div>

<!-- Ring color -->
<div class="ring ring-blue-500">Blue ring</div>
<div class="ring ring-red-500">Red ring</div>
<div class="ring ring-green-500">Green ring</div>
<div class="ring ring-gray-300">Gray ring</div>

<!-- Ring with opacity -->
<div class="ring ring-blue-500/50">50% opacity ring</div>

<!-- Ring offset -->
<div class="ring ring-offset-0">No offset</div>
<div class="ring ring-offset-1">1px offset</div>
<div class="ring ring-offset-2">2px offset</div>
<div class="ring ring-offset-4">4px offset</div>
<div class="ring ring-offset-8">8px offset</div>

<!-- Ring offset color -->
<div class="ring ring-offset-2 ring-offset-gray-100">Gray offset background</div>
```

### Outline Color

```html
<div class="outline outline-inherit">outline-color: inherit</div>
<div class="outline outline-current">outline-color: currentColor</div>
<div class="outline outline-transparent">outline-color: transparent</div>
<div class="outline outline-black">Black outline</div>
<div class="outline outline-white">White outline</div>
<div class="outline outline-blue-500">Blue outline</div>
```

---

## Flexbox

Control flexible box layout.

### Display Flex

```html
<div class="flex">display: flex</div>
<div class="inline-flex">display: inline-flex</div>
```

### Flex Direction

```html
<div class="flex flex-row">flex-direction: row (default)</div>
<div class="flex flex-row-reverse">flex-direction: row-reverse</div>
<div class="flex flex-col">flex-direction: column</div>
<div class="flex flex-col-reverse">flex-direction: column-reverse</div>
```

### Flex Wrap

```html
<div class="flex flex-wrap">flex-wrap: wrap</div>
<div class="flex flex-wrap-reverse">flex-wrap: wrap-reverse</div>
<div class="flex flex-nowrap">flex-wrap: nowrap (default)</div>
```

### Justify Content

```html
<div class="flex justify-normal">justify-content: normal</div>
<div class="flex justify-start">justify-content: flex-start</div>
<div class="flex justify-end">justify-content: flex-end</div>
<div class="flex justify-center">justify-content: center</div>
<div class="flex justify-between">justify-content: space-between</div>
<div class="flex justify-around">justify-content: space-around</div>
<div class="flex justify-evenly">justify-content: space-evenly</div>
<div class="flex justify-stretch">justify-content: stretch</div>
```

### Justify Items

```html
<div class="flex justify-items-start">justify-items: start</div>
<div class="flex justify-items-end">justify-items: end</div>
<div class="flex justify-items-center">justify-items: center</div>
<div class="flex justify-items-stretch">justify-items: stretch</div>
```

### Justify Self

```html
<div class="justify-self-auto">justify-self: auto</div>
<div class="justify-self-start">justify-self: start</div>
<div class="justify-self-end">justify-self: end</div>
<div class="justify-self-center">justify-self: center</div>
<div class="justify-self-stretch">justify-self: stretch</div>
```

### Align Items

```html
<div class="flex items-start">align-items: flex-start</div>
<div class="flex items-end">align-items: flex-end</div>
<div class="flex items-center">align-items: center</div>
<div class="flex items-baseline">align-items: baseline</div>
<div class="flex items-stretch">align-items: stretch (default)</div>
```

### Align Content

```html
<div class="flex flex-wrap content-normal">align-content: normal</div>
<div class="flex flex-wrap content-start">align-content: flex-start</div>
<div class="flex flex-wrap content-end">align-content: flex-end</div>
<div class="flex flex-wrap content-center">align-content: center</div>
<div class="flex flex-wrap content-between">align-content: space-between</div>
<div class="flex flex-wrap content-around">align-content: space-around</div>
<div class="flex flex-wrap content-evenly">align-content: space-evenly</div>
<div class="flex flex-wrap content-baseline">align-content: baseline</div>
<div class="flex flex-wrap content-stretch">align-content: stretch</div>
```

### Align Self

```html
<div class="self-auto">align-self: auto</div>
<div class="self-start">align-self: flex-start</div>
<div class="self-end">align-self: flex-end</div>
<div class="self-center">align-self: center</div>
<div class="self-stretch">align-self: stretch</div>
<div class="self-baseline">align-self: baseline</div>
```

### Flex

```html
<div class="flex-1">flex: 1 1 0% (grow and shrink equally)</div>
<div class="flex-auto">flex: 1 1 auto (grow, shrink, based on content)</div>
<div class="flex-initial">flex: 0 1 auto (shrink but not grow)</div>
<div class="flex-none">flex: none (no flex)</div>
```

### Flex Grow

```html
<div class="grow">flex-grow: 1</div>
<div class="grow-0">flex-grow: 0</div>
```

### Flex Shrink

```html
<div class="shrink">flex-shrink: 1</div>
<div class="shrink-0">flex-shrink: 0</div>
```

### Flex Basis

```html
<div class="basis-0">flex-basis: 0</div>
<div class="basis-1">flex-basis: 0.25rem</div>
<div class="basis-auto">flex-basis: auto</div>
<div class="basis-full">flex-basis: 100%</div>
<div class="basis-1/2">flex-basis: 50%</div>
<div class="basis-1/3">flex-basis: 33.333%</div>
<div class="basis-1/4">flex-basis: 25%</div>
```

### Order

```html
<div class="order-first">order: -9999</div>
<div class="order-last">order: 9999</div>
<div class="order-none">order: 0</div>
<div class="order-1">order: 1</div>
<div class="order-2">order: 2</div>
<div class="order-3">order: 3</div>
<div class="order-4">order: 4</div>
<div class="order-5">order: 5</div>
<div class="order-6">order: 6</div>
<div class="order-7">order: 7</div>
<div class="order-8">order: 8</div>
<div class="order-9">order: 9</div>
<div class="order-10">order: 10</div>
<div class="order-11">order: 11</div>
<div class="order-12">order: 12</div>
```

---

## Grid

Control CSS Grid layout.

### Display Grid

```html
<div class="grid">display: grid</div>
<div class="inline-grid">display: inline-grid</div>
```

### Grid Template Columns

```html
<div class="grid grid-cols-1">1 column</div>
<div class="grid grid-cols-2">2 columns</div>
<div class="grid grid-cols-3">3 columns</div>
<div class="grid grid-cols-4">4 columns</div>
<div class="grid grid-cols-5">5 columns</div>
<div class="grid grid-cols-6">6 columns</div>
<div class="grid grid-cols-7">7 columns</div>
<div class="grid grid-cols-8">8 columns</div>
<div class="grid grid-cols-9">9 columns</div>
<div class="grid grid-cols-10">10 columns</div>
<div class="grid grid-cols-11">11 columns</div>
<div class="grid grid-cols-12">12 columns</div>
<div class="grid grid-cols-none">No columns defined</div>
<div class="grid grid-cols-subgrid">Subgrid columns</div>

<!-- Arbitrary values -->
<div class="grid grid-cols-[200px_1fr_2fr]">Custom columns</div>
```

### Grid Column Span

```html
<div class="col-auto">grid-column: auto</div>
<div class="col-span-1">grid-column: span 1 / span 1</div>
<div class="col-span-2">grid-column: span 2 / span 2</div>
<div class="col-span-3">grid-column: span 3 / span 3</div>
<div class="col-span-4">grid-column: span 4 / span 4</div>
<div class="col-span-5">grid-column: span 5 / span 5</div>
<div class="col-span-6">grid-column: span 6 / span 6</div>
<div class="col-span-7">grid-column: span 7 / span 7</div>
<div class="col-span-8">grid-column: span 8 / span 8</div>
<div class="col-span-9">grid-column: span 9 / span 9</div>
<div class="col-span-10">grid-column: span 10 / span 10</div>
<div class="col-span-11">grid-column: span 11 / span 11</div>
<div class="col-span-12">grid-column: span 12 / span 12</div>
<div class="col-span-full">grid-column: 1 / -1</div>
```

### Grid Column Start/End

```html
<div class="col-start-1">grid-column-start: 1</div>
<div class="col-start-2">grid-column-start: 2</div>
<div class="col-start-3">grid-column-start: 3</div>
<div class="col-start-auto">grid-column-start: auto</div>

<div class="col-end-1">grid-column-end: 1</div>
<div class="col-end-2">grid-column-end: 2</div>
<div class="col-end-3">grid-column-end: 3</div>
<div class="col-end-auto">grid-column-end: auto</div>
```

### Grid Template Rows

```html
<div class="grid grid-rows-1">1 row</div>
<div class="grid grid-rows-2">2 rows</div>
<div class="grid grid-rows-3">3 rows</div>
<div class="grid grid-rows-4">4 rows</div>
<div class="grid grid-rows-5">5 rows</div>
<div class="grid grid-rows-6">6 rows</div>
<div class="grid grid-rows-none">No rows defined</div>
<div class="grid grid-rows-subgrid">Subgrid rows</div>
```

### Grid Row Span

```html
<div class="row-auto">grid-row: auto</div>
<div class="row-span-1">grid-row: span 1 / span 1</div>
<div class="row-span-2">grid-row: span 2 / span 2</div>
<div class="row-span-3">grid-row: span 3 / span 3</div>
<div class="row-span-4">grid-row: span 4 / span 4</div>
<div class="row-span-5">grid-row: span 5 / span 5</div>
<div class="row-span-6">grid-row: span 6 / span 6</div>
<div class="row-span-full">grid-row: 1 / -1</div>
```

### Grid Row Start/End

```html
<div class="row-start-1">grid-row-start: 1</div>
<div class="row-start-2">grid-row-start: 2</div>
<div class="row-start-3">grid-row-start: 3</div>
<div class="row-start-auto">grid-row-start: auto</div>

<div class="row-end-1">grid-row-end: 1</div>
<div class="row-end-2">grid-row-end: 2</div>
<div class="row-end-3">grid-row-end: 3</div>
<div class="row-end-auto">grid-row-end: auto</div>
```

### Grid Auto Flow

```html
<div class="grid-flow-row">grid-auto-flow: row</div>
<div class="grid-flow-col">grid-auto-flow: column</div>
<div class="grid-flow-dense">grid-auto-flow: dense</div>
<div class="grid-flow-row-dense">grid-auto-flow: row dense</div>
<div class="grid-flow-col-dense">grid-auto-flow: column dense</div>
```

### Grid Auto Columns

```html
<div class="auto-cols-auto">grid-auto-columns: auto</div>
<div class="auto-cols-min">grid-auto-columns: min-content</div>
<div class="auto-cols-max">grid-auto-columns: max-content</div>
<div class="auto-cols-fr">grid-auto-columns: minmax(0, 1fr)</div>
```

### Grid Auto Rows

```html
<div class="auto-rows-auto">grid-auto-rows: auto</div>
<div class="auto-rows-min">grid-auto-rows: min-content</div>
<div class="auto-rows-max">grid-auto-rows: max-content</div>
<div class="auto-rows-fr">grid-auto-rows: minmax(0, 1fr)</div>
```

### Place Content

```html
<div class="place-content-center">place-content: center</div>
<div class="place-content-start">place-content: start</div>
<div class="place-content-end">place-content: end</div>
<div class="place-content-between">place-content: space-between</div>
<div class="place-content-around">place-content: space-around</div>
<div class="place-content-evenly">place-content: space-evenly</div>
<div class="place-content-baseline">place-content: baseline</div>
<div class="place-content-stretch">place-content: stretch</div>
```

### Place Items

```html
<div class="place-items-start">place-items: start</div>
<div class="place-items-end">place-items: end</div>
<div class="place-items-center">place-items: center</div>
<div class="place-items-baseline">place-items: baseline</div>
<div class="place-items-stretch">place-items: stretch</div>
```

### Place Self

```html
<div class="place-self-auto">place-self: auto</div>
<div class="place-self-start">place-self: start</div>
<div class="place-self-end">place-self: end</div>
<div class="place-self-center">place-self: center</div>
<div class="place-self-stretch">place-self: stretch</div>
```

---

## Layout

Control display, position, z-index, and overflow properties.

### Display

```html
<div class="block">display: block</div>
<div class="inline-block">display: inline-block</div>
<div class="inline">display: inline</div>
<div class="flex">display: flex</div>
<div class="inline-flex">display: inline-flex</div>
<div class="grid">display: grid</div>
<div class="inline-grid">display: inline-grid</div>
<div class="table">display: table</div>
<div class="inline-table">display: inline-table</div>
<div class="table-caption">display: table-caption</div>
<div class="table-cell">display: table-cell</div>
<div class="table-column">display: table-column</div>
<div class="table-column-group">display: table-column-group</div>
<div class="table-footer-group">display: table-footer-group</div>
<div class="table-header-group">display: table-header-group</div>
<div class="table-row-group">display: table-row-group</div>
<div class="table-row">display: table-row</div>
<div class="flow-root">display: flow-root</div>
<div class="contents">display: contents</div>
<div class="list-item">display: list-item</div>
<div class="hidden">display: none</div>
```

### Position

```html
<div class="static">position: static</div>
<div class="fixed">position: fixed</div>
<div class="absolute">position: absolute</div>
<div class="relative">position: relative</div>
<div class="sticky">position: sticky</div>
```

### Position Coordinates (Top, Right, Bottom, Left)

```html
<!-- Inset (all sides) -->
<div class="inset-0">top: 0; right: 0; bottom: 0; left: 0</div>
<div class="inset-auto">top: auto; right: auto; bottom: auto; left: auto</div>
<div class="inset-1/2">50% on all sides</div>
<div class="inset-full">100% on all sides</div>

<!-- Inset X (left and right) -->
<div class="inset-x-0">left: 0; right: 0</div>
<div class="inset-x-auto">left: auto; right: auto</div>

<!-- Inset Y (top and bottom) -->
<div class="inset-y-0">top: 0; bottom: 0</div>
<div class="inset-y-auto">top: auto; bottom: auto</div>

<!-- Individual sides -->
<div class="top-0">top: 0</div>
<div class="top-1">top: 0.25rem</div>
<div class="top-2">top: 0.5rem</div>
<div class="top-4">top: 1rem</div>
<div class="top-auto">top: auto</div>
<div class="top-1/2">top: 50%</div>
<div class="top-full">top: 100%</div>

<div class="right-0">right: 0</div>
<div class="right-4">right: 1rem</div>
<div class="right-auto">right: auto</div>

<div class="bottom-0">bottom: 0</div>
<div class="bottom-4">bottom: 1rem</div>
<div class="bottom-auto">bottom: auto</div>

<div class="left-0">left: 0</div>
<div class="left-4">left: 1rem</div>
<div class="left-auto">left: auto</div>

<!-- Logical properties -->
<div class="start-0">inset-inline-start: 0</div>
<div class="end-0">inset-inline-end: 0</div>

<!-- Negative values -->
<div class="-top-4">top: -1rem</div>
<div class="-left-2">left: -0.5rem</div>
```

### Z-Index

```html
<div class="z-0">z-index: 0</div>
<div class="z-10">z-index: 10</div>
<div class="z-20">z-index: 20</div>
<div class="z-30">z-index: 30</div>
<div class="z-40">z-index: 40</div>
<div class="z-50">z-index: 50</div>
<div class="z-auto">z-index: auto</div>

<!-- Negative z-index -->
<div class="-z-10">z-index: -10</div>

<!-- Arbitrary values -->
<div class="z-[100]">z-index: 100</div>
```

### Overflow

```html
<div class="overflow-auto">overflow: auto</div>
<div class="overflow-hidden">overflow: hidden</div>
<div class="overflow-clip">overflow: clip</div>
<div class="overflow-visible">overflow: visible</div>
<div class="overflow-scroll">overflow: scroll</div>

<!-- Overflow X (horizontal) -->
<div class="overflow-x-auto">overflow-x: auto</div>
<div class="overflow-x-hidden">overflow-x: hidden</div>
<div class="overflow-x-clip">overflow-x: clip</div>
<div class="overflow-x-visible">overflow-x: visible</div>
<div class="overflow-x-scroll">overflow-x: scroll</div>

<!-- Overflow Y (vertical) -->
<div class="overflow-y-auto">overflow-y: auto</div>
<div class="overflow-y-hidden">overflow-y: hidden</div>
<div class="overflow-y-clip">overflow-y: clip</div>
<div class="overflow-y-visible">overflow-y: visible</div>
<div class="overflow-y-scroll">overflow-y: scroll</div>
```

### Visibility

```html
<div class="visible">visibility: visible</div>
<div class="invisible">visibility: hidden</div>
<div class="collapse">visibility: collapse</div>
```

### Float

```html
<div class="float-start">float: inline-start</div>
<div class="float-end">float: inline-end</div>
<div class="float-right">float: right</div>
<div class="float-left">float: left</div>
<div class="float-none">float: none</div>
```

### Clear

```html
<div class="clear-start">clear: inline-start</div>
<div class="clear-end">clear: inline-end</div>
<div class="clear-left">clear: left</div>
<div class="clear-right">clear: right</div>
<div class="clear-both">clear: both</div>
<div class="clear-none">clear: none</div>
```

### Isolation

```html
<div class="isolate">isolation: isolate</div>
<div class="isolation-auto">isolation: auto</div>
```

### Object Fit

```html
<img class="object-contain">object-fit: contain</img>
<img class="object-cover">object-fit: cover</img>
<img class="object-fill">object-fit: fill</img>
<img class="object-none">object-fit: none</img>
<img class="object-scale-down">object-fit: scale-down</img>
```

### Object Position

```html
<img class="object-bottom">object-position: bottom</img>
<img class="object-center">object-position: center</img>
<img class="object-left">object-position: left</img>
<img class="object-left-bottom">object-position: left bottom</img>
<img class="object-left-top">object-position: left top</img>
<img class="object-right">object-position: right</img>
<img class="object-right-bottom">object-position: right bottom</img>
<img class="object-right-top">object-position: right top</img>
<img class="object-top">object-position: top</img>
```

### Aspect Ratio

```html
<div class="aspect-auto">aspect-ratio: auto</div>
<div class="aspect-square">aspect-ratio: 1 / 1</div>
<div class="aspect-video">aspect-ratio: 16 / 9</div>

<!-- Arbitrary values -->
<div class="aspect-[4/3]">aspect-ratio: 4 / 3</div>
```

### Container

```html
<div class="container">Responsive container with max-width breakpoints</div>
<div class="container mx-auto">Centered container</div>
```

### Columns

```html
<div class="columns-1">columns: 1</div>
<div class="columns-2">columns: 2</div>
<div class="columns-3">columns: 3</div>
<div class="columns-4">columns: 4</div>
<div class="columns-auto">columns: auto</div>
<div class="columns-xs">columns: 20rem</div>
<div class="columns-sm">columns: 24rem</div>
<div class="columns-md">columns: 28rem</div>
<div class="columns-lg">columns: 32rem</div>
```

### Break

```html
<!-- Break after -->
<div class="break-after-auto">break-after: auto</div>
<div class="break-after-avoid">break-after: avoid</div>
<div class="break-after-all">break-after: all</div>
<div class="break-after-avoid-page">break-after: avoid-page</div>
<div class="break-after-page">break-after: page</div>
<div class="break-after-left">break-after: left</div>
<div class="break-after-right">break-after: right</div>
<div class="break-after-column">break-after: column</div>

<!-- Break before -->
<div class="break-before-auto">break-before: auto</div>
<div class="break-before-avoid">break-before: avoid</div>
<div class="break-before-all">break-before: all</div>
<div class="break-before-avoid-page">break-before: avoid-page</div>
<div class="break-before-page">break-before: page</div>
<div class="break-before-left">break-before: left</div>
<div class="break-before-right">break-before: right</div>
<div class="break-before-column">break-before: column</div>

<!-- Break inside -->
<div class="break-inside-auto">break-inside: auto</div>
<div class="break-inside-avoid">break-inside: avoid</div>
<div class="break-inside-avoid-page">break-inside: avoid-page</div>
<div class="break-inside-avoid-column">break-inside: avoid-column</div>
```

### Box Decoration Break

```html
<div class="box-decoration-clone">box-decoration-break: clone</div>
<div class="box-decoration-slice">box-decoration-break: slice</div>
```

### Box Sizing

```html
<div class="box-border">box-sizing: border-box</div>
<div class="box-content">box-sizing: content-box</div>
```

---

## Borders

Control border width, radius, style, and color.

### Border Width

```html
<!-- All sides -->
<div class="border-0">border-width: 0</div>
<div class="border">border-width: 1px</div>
<div class="border-2">border-width: 2px</div>
<div class="border-4">border-width: 4px</div>
<div class="border-8">border-width: 8px</div>

<!-- Individual sides -->
<div class="border-t-0">border-top-width: 0</div>
<div class="border-t">border-top-width: 1px</div>
<div class="border-t-2">border-top-width: 2px</div>
<div class="border-t-4">border-top-width: 4px</div>
<div class="border-t-8">border-top-width: 8px</div>

<div class="border-r-0">border-right-width: 0</div>
<div class="border-r">border-right-width: 1px</div>
<div class="border-r-2">border-right-width: 2px</div>
<div class="border-r-4">border-right-width: 4px</div>

<div class="border-b-0">border-bottom-width: 0</div>
<div class="border-b">border-bottom-width: 1px</div>
<div class="border-b-2">border-bottom-width: 2px</div>
<div class="border-b-4">border-bottom-width: 4px</div>

<div class="border-l-0">border-left-width: 0</div>
<div class="border-l">border-left-width: 1px</div>
<div class="border-l-2">border-left-width: 2px</div>
<div class="border-l-4">border-left-width: 4px</div>

<!-- Horizontal (left and right) -->
<div class="border-x-0">border-left: 0; border-right: 0</div>
<div class="border-x">border-left: 1px; border-right: 1px</div>
<div class="border-x-2">border-left: 2px; border-right: 2px</div>

<!-- Vertical (top and bottom) -->
<div class="border-y-0">border-top: 0; border-bottom: 0</div>
<div class="border-y">border-top: 1px; border-bottom: 1px</div>
<div class="border-y-2">border-top: 2px; border-bottom: 2px</div>

<!-- Logical properties -->
<div class="border-s">border-inline-start-width: 1px</div>
<div class="border-e">border-inline-end-width: 1px</div>
```

### Border Radius

```html
<!-- All corners -->
<div class="rounded-none">border-radius: 0</div>
<div class="rounded-sm">border-radius: 0.125rem (2px)</div>
<div class="rounded">border-radius: 0.25rem (4px)</div>
<div class="rounded-md">border-radius: 0.375rem (6px)</div>
<div class="rounded-lg">border-radius: 0.5rem (8px)</div>
<div class="rounded-xl">border-radius: 0.75rem (12px)</div>
<div class="rounded-2xl">border-radius: 1rem (16px)</div>
<div class="rounded-3xl">border-radius: 1.5rem (24px)</div>
<div class="rounded-full">border-radius: 9999px</div>

<!-- Individual corners -->
<div class="rounded-t-lg">Top corners only</div>
<div class="rounded-r-lg">Right corners only</div>
<div class="rounded-b-lg">Bottom corners only</div>
<div class="rounded-l-lg">Left corners only</div>

<div class="rounded-tl-lg">Top-left corner</div>
<div class="rounded-tr-lg">Top-right corner</div>
<div class="rounded-br-lg">Bottom-right corner</div>
<div class="rounded-bl-lg">Bottom-left corner</div>

<!-- Logical corners -->
<div class="rounded-s-lg">Start corners (left in LTR)</div>
<div class="rounded-e-lg">End corners (right in LTR)</div>
<div class="rounded-ss-lg">Start-start corner</div>
<div class="rounded-se-lg">Start-end corner</div>
<div class="rounded-es-lg">End-start corner</div>
<div class="rounded-ee-lg">End-end corner</div>
```

### Border Style

```html
<div class="border border-solid">border-style: solid</div>
<div class="border border-dashed">border-style: dashed</div>
<div class="border border-dotted">border-style: dotted</div>
<div class="border border-double">border-style: double</div>
<div class="border border-hidden">border-style: hidden</div>
<div class="border-none">border-style: none</div>
```

### Divide Width (Between Children)

```html
<!-- Horizontal divide (between columns) -->
<div class="divide-x">1px dividers between horizontal children</div>
<div class="divide-x-0">No horizontal dividers</div>
<div class="divide-x-2">2px horizontal dividers</div>
<div class="divide-x-4">4px horizontal dividers</div>
<div class="divide-x-8">8px horizontal dividers</div>
<div class="divide-x-reverse">Reverse horizontal dividers</div>

<!-- Vertical divide (between rows) -->
<div class="divide-y">1px dividers between vertical children</div>
<div class="divide-y-0">No vertical dividers</div>
<div class="divide-y-2">2px vertical dividers</div>
<div class="divide-y-4">4px vertical dividers</div>
<div class="divide-y-8">8px vertical dividers</div>
<div class="divide-y-reverse">Reverse vertical dividers</div>
```

### Divide Style

```html
<div class="divide-y divide-solid">Solid dividers</div>
<div class="divide-y divide-dashed">Dashed dividers</div>
<div class="divide-y divide-dotted">Dotted dividers</div>
<div class="divide-y divide-double">Double dividers</div>
<div class="divide-y divide-none">No dividers</div>
```

### Divide Color

```html
<div class="divide-y divide-gray-200">Gray dividers</div>
<div class="divide-y divide-blue-500">Blue dividers</div>
<div class="divide-y divide-transparent">Transparent dividers</div>
```

### Outline Width

```html
<div class="outline-0">outline-width: 0</div>
<div class="outline-1">outline-width: 1px</div>
<div class="outline-2">outline-width: 2px</div>
<div class="outline-4">outline-width: 4px</div>
<div class="outline-8">outline-width: 8px</div>
```

### Outline Style

```html
<div class="outline-none">outline: 2px solid transparent</div>
<div class="outline">outline-style: solid</div>
<div class="outline-dashed">outline-style: dashed</div>
<div class="outline-dotted">outline-style: dotted</div>
<div class="outline-double">outline-style: double</div>
```

### Outline Offset

```html
<div class="outline-offset-0">outline-offset: 0</div>
<div class="outline-offset-1">outline-offset: 1px</div>
<div class="outline-offset-2">outline-offset: 2px</div>
<div class="outline-offset-4">outline-offset: 4px</div>
<div class="outline-offset-8">outline-offset: 8px</div>
```

---

## Effects

Control shadows, opacity, blur, and other visual effects.

### Box Shadow

```html
<div class="shadow-sm">Small shadow</div>
<div class="shadow">Default shadow</div>
<div class="shadow-md">Medium shadow</div>
<div class="shadow-lg">Large shadow</div>
<div class="shadow-xl">Extra large shadow</div>
<div class="shadow-2xl">2x extra large shadow</div>
<div class="shadow-inner">Inner shadow</div>
<div class="shadow-none">No shadow</div>

<!-- Shadow color -->
<div class="shadow-lg shadow-blue-500/50">Blue shadow with opacity</div>
<div class="shadow-lg shadow-red-500">Red shadow</div>
<div class="shadow-lg shadow-gray-900/25">Dark gray shadow</div>
```

### Opacity

```html
<div class="opacity-0">opacity: 0</div>
<div class="opacity-5">opacity: 0.05</div>
<div class="opacity-10">opacity: 0.1</div>
<div class="opacity-15">opacity: 0.15</div>
<div class="opacity-20">opacity: 0.2</div>
<div class="opacity-25">opacity: 0.25</div>
<div class="opacity-30">opacity: 0.3</div>
<div class="opacity-35">opacity: 0.35</div>
<div class="opacity-40">opacity: 0.4</div>
<div class="opacity-45">opacity: 0.45</div>
<div class="opacity-50">opacity: 0.5</div>
<div class="opacity-55">opacity: 0.55</div>
<div class="opacity-60">opacity: 0.6</div>
<div class="opacity-65">opacity: 0.65</div>
<div class="opacity-70">opacity: 0.7</div>
<div class="opacity-75">opacity: 0.75</div>
<div class="opacity-80">opacity: 0.8</div>
<div class="opacity-85">opacity: 0.85</div>
<div class="opacity-90">opacity: 0.9</div>
<div class="opacity-95">opacity: 0.95</div>
<div class="opacity-100">opacity: 1</div>
```

### Mix Blend Mode

```html
<div class="mix-blend-normal">mix-blend-mode: normal</div>
<div class="mix-blend-multiply">mix-blend-mode: multiply</div>
<div class="mix-blend-screen">mix-blend-mode: screen</div>
<div class="mix-blend-overlay">mix-blend-mode: overlay</div>
<div class="mix-blend-darken">mix-blend-mode: darken</div>
<div class="mix-blend-lighten">mix-blend-mode: lighten</div>
<div class="mix-blend-color-dodge">mix-blend-mode: color-dodge</div>
<div class="mix-blend-color-burn">mix-blend-mode: color-burn</div>
<div class="mix-blend-hard-light">mix-blend-mode: hard-light</div>
<div class="mix-blend-soft-light">mix-blend-mode: soft-light</div>
<div class="mix-blend-difference">mix-blend-mode: difference</div>
<div class="mix-blend-exclusion">mix-blend-mode: exclusion</div>
<div class="mix-blend-hue">mix-blend-mode: hue</div>
<div class="mix-blend-saturation">mix-blend-mode: saturation</div>
<div class="mix-blend-color">mix-blend-mode: color</div>
<div class="mix-blend-luminosity">mix-blend-mode: luminosity</div>
<div class="mix-blend-plus-darker">mix-blend-mode: plus-darker</div>
<div class="mix-blend-plus-lighter">mix-blend-mode: plus-lighter</div>
```

### Background Blend Mode

```html
<div class="bg-blend-normal">background-blend-mode: normal</div>
<div class="bg-blend-multiply">background-blend-mode: multiply</div>
<div class="bg-blend-screen">background-blend-mode: screen</div>
<div class="bg-blend-overlay">background-blend-mode: overlay</div>
<div class="bg-blend-darken">background-blend-mode: darken</div>
<div class="bg-blend-lighten">background-blend-mode: lighten</div>
<div class="bg-blend-color-dodge">background-blend-mode: color-dodge</div>
<div class="bg-blend-color-burn">background-blend-mode: color-burn</div>
<div class="bg-blend-hard-light">background-blend-mode: hard-light</div>
<div class="bg-blend-soft-light">background-blend-mode: soft-light</div>
<div class="bg-blend-difference">background-blend-mode: difference</div>
<div class="bg-blend-exclusion">background-blend-mode: exclusion</div>
<div class="bg-blend-hue">background-blend-mode: hue</div>
<div class="bg-blend-saturation">background-blend-mode: saturation</div>
<div class="bg-blend-color">background-blend-mode: color</div>
<div class="bg-blend-luminosity">background-blend-mode: luminosity</div>
```

---

## Transitions and Animations

Control CSS transitions and animations.

### Transition Property

```html
<div class="transition-none">No transition</div>
<div class="transition-all">Transition all properties</div>
<div class="transition">Transition common properties (color, background, border, shadow, transform)</div>
<div class="transition-colors">Transition colors only</div>
<div class="transition-opacity">Transition opacity only</div>
<div class="transition-shadow">Transition shadow only</div>
<div class="transition-transform">Transition transform only</div>
```

### Transition Duration

```html
<div class="duration-0">transition-duration: 0ms</div>
<div class="duration-75">transition-duration: 75ms</div>
<div class="duration-100">transition-duration: 100ms</div>
<div class="duration-150">transition-duration: 150ms</div>
<div class="duration-200">transition-duration: 200ms</div>
<div class="duration-300">transition-duration: 300ms</div>
<div class="duration-500">transition-duration: 500ms</div>
<div class="duration-700">transition-duration: 700ms</div>
<div class="duration-1000">transition-duration: 1000ms</div>
```

### Transition Timing Function

```html
<div class="ease-linear">transition-timing-function: linear</div>
<div class="ease-in">transition-timing-function: cubic-bezier(0.4, 0, 1, 1)</div>
<div class="ease-out">transition-timing-function: cubic-bezier(0, 0, 0.2, 1)</div>
<div class="ease-in-out">transition-timing-function: cubic-bezier(0.4, 0, 0.2, 1)</div>
```

### Transition Delay

```html
<div class="delay-0">transition-delay: 0ms</div>
<div class="delay-75">transition-delay: 75ms</div>
<div class="delay-100">transition-delay: 100ms</div>
<div class="delay-150">transition-delay: 150ms</div>
<div class="delay-200">transition-delay: 200ms</div>
<div class="delay-300">transition-delay: 300ms</div>
<div class="delay-500">transition-delay: 500ms</div>
<div class="delay-700">transition-delay: 700ms</div>
<div class="delay-1000">transition-delay: 1000ms</div>
```

### Animation

```html
<div class="animate-none">No animation</div>
<div class="animate-spin">Spinning animation (360deg rotation)</div>
<div class="animate-ping">Ping/pulse animation (for notifications)</div>
<div class="animate-pulse">Gentle pulse animation (for skeletons)</div>
<div class="animate-bounce">Bounce animation</div>

<!-- Practical examples -->
<button class="animate-spin">
  <svg>Loading spinner</svg>
</button>

<span class="animate-ping absolute inline-flex h-3 w-3 rounded-full bg-red-400">
  Notification indicator
</span>

<div class="animate-pulse bg-gray-200 h-4 rounded">
  Loading skeleton
</div>
```

---

## Transforms

Control transform properties for rotation, scaling, translation, and skewing.

### Transform Origin

```html
<div class="origin-center">transform-origin: center</div>
<div class="origin-top">transform-origin: top</div>
<div class="origin-top-right">transform-origin: top right</div>
<div class="origin-right">transform-origin: right</div>
<div class="origin-bottom-right">transform-origin: bottom right</div>
<div class="origin-bottom">transform-origin: bottom</div>
<div class="origin-bottom-left">transform-origin: bottom left</div>
<div class="origin-left">transform-origin: left</div>
<div class="origin-top-left">transform-origin: top left</div>
```

### Scale

```html
<!-- Uniform scale -->
<div class="scale-0">transform: scale(0)</div>
<div class="scale-50">transform: scale(0.5)</div>
<div class="scale-75">transform: scale(0.75)</div>
<div class="scale-90">transform: scale(0.9)</div>
<div class="scale-95">transform: scale(0.95)</div>
<div class="scale-100">transform: scale(1)</div>
<div class="scale-105">transform: scale(1.05)</div>
<div class="scale-110">transform: scale(1.1)</div>
<div class="scale-125">transform: scale(1.25)</div>
<div class="scale-150">transform: scale(1.5)</div>

<!-- Scale X only -->
<div class="scale-x-0">transform: scaleX(0)</div>
<div class="scale-x-50">transform: scaleX(0.5)</div>
<div class="scale-x-100">transform: scaleX(1)</div>
<div class="scale-x-150">transform: scaleX(1.5)</div>

<!-- Scale Y only -->
<div class="scale-y-0">transform: scaleY(0)</div>
<div class="scale-y-50">transform: scaleY(0.5)</div>
<div class="scale-y-100">transform: scaleY(1)</div>
<div class="scale-y-150">transform: scaleY(1.5)</div>
```

### Rotate

```html
<div class="rotate-0">transform: rotate(0deg)</div>
<div class="rotate-1">transform: rotate(1deg)</div>
<div class="rotate-2">transform: rotate(2deg)</div>
<div class="rotate-3">transform: rotate(3deg)</div>
<div class="rotate-6">transform: rotate(6deg)</div>
<div class="rotate-12">transform: rotate(12deg)</div>
<div class="rotate-45">transform: rotate(45deg)</div>
<div class="rotate-90">transform: rotate(90deg)</div>
<div class="rotate-180">transform: rotate(180deg)</div>

<!-- Negative rotation -->
<div class="-rotate-1">transform: rotate(-1deg)</div>
<div class="-rotate-2">transform: rotate(-2deg)</div>
<div class="-rotate-3">transform: rotate(-3deg)</div>
<div class="-rotate-6">transform: rotate(-6deg)</div>
<div class="-rotate-12">transform: rotate(-12deg)</div>
<div class="-rotate-45">transform: rotate(-45deg)</div>
<div class="-rotate-90">transform: rotate(-90deg)</div>
<div class="-rotate-180">transform: rotate(-180deg)</div>
```

### Translate

```html
<!-- Translate X -->
<div class="translate-x-0">transform: translateX(0)</div>
<div class="translate-x-1">transform: translateX(0.25rem)</div>
<div class="translate-x-2">transform: translateX(0.5rem)</div>
<div class="translate-x-4">transform: translateX(1rem)</div>
<div class="translate-x-8">transform: translateX(2rem)</div>
<div class="translate-x-1/2">transform: translateX(50%)</div>
<div class="translate-x-full">transform: translateX(100%)</div>

<!-- Negative translate X -->
<div class="-translate-x-1">transform: translateX(-0.25rem)</div>
<div class="-translate-x-4">transform: translateX(-1rem)</div>
<div class="-translate-x-1/2">transform: translateX(-50%)</div>
<div class="-translate-x-full">transform: translateX(-100%)</div>

<!-- Translate Y -->
<div class="translate-y-0">transform: translateY(0)</div>
<div class="translate-y-1">transform: translateY(0.25rem)</div>
<div class="translate-y-2">transform: translateY(0.5rem)</div>
<div class="translate-y-4">transform: translateY(1rem)</div>
<div class="translate-y-8">transform: translateY(2rem)</div>
<div class="translate-y-1/2">transform: translateY(50%)</div>
<div class="translate-y-full">transform: translateY(100%)</div>

<!-- Negative translate Y -->
<div class="-translate-y-1">transform: translateY(-0.25rem)</div>
<div class="-translate-y-4">transform: translateY(-1rem)</div>
<div class="-translate-y-1/2">transform: translateY(-50%)</div>
<div class="-translate-y-full">transform: translateY(-100%)</div>
```

### Skew

```html
<!-- Skew X -->
<div class="skew-x-0">transform: skewX(0deg)</div>
<div class="skew-x-1">transform: skewX(1deg)</div>
<div class="skew-x-2">transform: skewX(2deg)</div>
<div class="skew-x-3">transform: skewX(3deg)</div>
<div class="skew-x-6">transform: skewX(6deg)</div>
<div class="skew-x-12">transform: skewX(12deg)</div>

<!-- Negative skew X -->
<div class="-skew-x-1">transform: skewX(-1deg)</div>
<div class="-skew-x-6">transform: skewX(-6deg)</div>
<div class="-skew-x-12">transform: skewX(-12deg)</div>

<!-- Skew Y -->
<div class="skew-y-0">transform: skewY(0deg)</div>
<div class="skew-y-1">transform: skewY(1deg)</div>
<div class="skew-y-2">transform: skewY(2deg)</div>
<div class="skew-y-3">transform: skewY(3deg)</div>
<div class="skew-y-6">transform: skewY(6deg)</div>
<div class="skew-y-12">transform: skewY(12deg)</div>

<!-- Negative skew Y -->
<div class="-skew-y-1">transform: skewY(-1deg)</div>
<div class="-skew-y-6">transform: skewY(-6deg)</div>
<div class="-skew-y-12">transform: skewY(-12deg)</div>
```

---

## Filters

Control CSS filter effects.

### Blur

```html
<div class="blur-none">filter: blur(0)</div>
<div class="blur-sm">filter: blur(4px)</div>
<div class="blur">filter: blur(8px)</div>
<div class="blur-md">filter: blur(12px)</div>
<div class="blur-lg">filter: blur(16px)</div>
<div class="blur-xl">filter: blur(24px)</div>
<div class="blur-2xl">filter: blur(40px)</div>
<div class="blur-3xl">filter: blur(64px)</div>
```

### Brightness

```html
<div class="brightness-0">filter: brightness(0)</div>
<div class="brightness-50">filter: brightness(0.5)</div>
<div class="brightness-75">filter: brightness(0.75)</div>
<div class="brightness-90">filter: brightness(0.9)</div>
<div class="brightness-95">filter: brightness(0.95)</div>
<div class="brightness-100">filter: brightness(1)</div>
<div class="brightness-105">filter: brightness(1.05)</div>
<div class="brightness-110">filter: brightness(1.1)</div>
<div class="brightness-125">filter: brightness(1.25)</div>
<div class="brightness-150">filter: brightness(1.5)</div>
<div class="brightness-200">filter: brightness(2)</div>
```

### Contrast

```html
<div class="contrast-0">filter: contrast(0)</div>
<div class="contrast-50">filter: contrast(0.5)</div>
<div class="contrast-75">filter: contrast(0.75)</div>
<div class="contrast-100">filter: contrast(1)</div>
<div class="contrast-125">filter: contrast(1.25)</div>
<div class="contrast-150">filter: contrast(1.5)</div>
<div class="contrast-200">filter: contrast(2)</div>
```

### Grayscale

```html
<div class="grayscale-0">filter: grayscale(0)</div>
<div class="grayscale">filter: grayscale(100%)</div>
```

### Hue Rotate

```html
<div class="hue-rotate-0">filter: hue-rotate(0deg)</div>
<div class="hue-rotate-15">filter: hue-rotate(15deg)</div>
<div class="hue-rotate-30">filter: hue-rotate(30deg)</div>
<div class="hue-rotate-60">filter: hue-rotate(60deg)</div>
<div class="hue-rotate-90">filter: hue-rotate(90deg)</div>
<div class="hue-rotate-180">filter: hue-rotate(180deg)</div>

<!-- Negative values -->
<div class="-hue-rotate-15">filter: hue-rotate(-15deg)</div>
<div class="-hue-rotate-30">filter: hue-rotate(-30deg)</div>
<div class="-hue-rotate-60">filter: hue-rotate(-60deg)</div>
<div class="-hue-rotate-90">filter: hue-rotate(-90deg)</div>
<div class="-hue-rotate-180">filter: hue-rotate(-180deg)</div>
```

### Invert

```html
<div class="invert-0">filter: invert(0)</div>
<div class="invert">filter: invert(100%)</div>
```

### Saturate

```html
<div class="saturate-0">filter: saturate(0)</div>
<div class="saturate-50">filter: saturate(0.5)</div>
<div class="saturate-100">filter: saturate(1)</div>
<div class="saturate-150">filter: saturate(1.5)</div>
<div class="saturate-200">filter: saturate(2)</div>
```

### Sepia

```html
<div class="sepia-0">filter: sepia(0)</div>
<div class="sepia">filter: sepia(100%)</div>
```

### Drop Shadow

```html
<div class="drop-shadow-sm">Small drop shadow</div>
<div class="drop-shadow">Default drop shadow</div>
<div class="drop-shadow-md">Medium drop shadow</div>
<div class="drop-shadow-lg">Large drop shadow</div>
<div class="drop-shadow-xl">Extra large drop shadow</div>
<div class="drop-shadow-2xl">2XL drop shadow</div>
<div class="drop-shadow-none">No drop shadow</div>
```

### Backdrop Filters

```html
<!-- Backdrop blur -->
<div class="backdrop-blur-none">backdrop-filter: blur(0)</div>
<div class="backdrop-blur-sm">backdrop-filter: blur(4px)</div>
<div class="backdrop-blur">backdrop-filter: blur(8px)</div>
<div class="backdrop-blur-md">backdrop-filter: blur(12px)</div>
<div class="backdrop-blur-lg">backdrop-filter: blur(16px)</div>
<div class="backdrop-blur-xl">backdrop-filter: blur(24px)</div>
<div class="backdrop-blur-2xl">backdrop-filter: blur(40px)</div>
<div class="backdrop-blur-3xl">backdrop-filter: blur(64px)</div>

<!-- Backdrop brightness -->
<div class="backdrop-brightness-0">backdrop-filter: brightness(0)</div>
<div class="backdrop-brightness-50">backdrop-filter: brightness(0.5)</div>
<div class="backdrop-brightness-100">backdrop-filter: brightness(1)</div>
<div class="backdrop-brightness-150">backdrop-filter: brightness(1.5)</div>

<!-- Backdrop contrast -->
<div class="backdrop-contrast-0">backdrop-filter: contrast(0)</div>
<div class="backdrop-contrast-50">backdrop-filter: contrast(0.5)</div>
<div class="backdrop-contrast-100">backdrop-filter: contrast(1)</div>
<div class="backdrop-contrast-150">backdrop-filter: contrast(1.5)</div>

<!-- Backdrop grayscale -->
<div class="backdrop-grayscale-0">backdrop-filter: grayscale(0)</div>
<div class="backdrop-grayscale">backdrop-filter: grayscale(100%)</div>

<!-- Backdrop hue rotate -->
<div class="backdrop-hue-rotate-0">backdrop-filter: hue-rotate(0deg)</div>
<div class="backdrop-hue-rotate-90">backdrop-filter: hue-rotate(90deg)</div>
<div class="backdrop-hue-rotate-180">backdrop-filter: hue-rotate(180deg)</div>

<!-- Backdrop invert -->
<div class="backdrop-invert-0">backdrop-filter: invert(0)</div>
<div class="backdrop-invert">backdrop-filter: invert(100%)</div>

<!-- Backdrop opacity -->
<div class="backdrop-opacity-0">backdrop-filter: opacity(0)</div>
<div class="backdrop-opacity-50">backdrop-filter: opacity(0.5)</div>
<div class="backdrop-opacity-100">backdrop-filter: opacity(1)</div>

<!-- Backdrop saturate -->
<div class="backdrop-saturate-0">backdrop-filter: saturate(0)</div>
<div class="backdrop-saturate-100">backdrop-filter: saturate(1)</div>
<div class="backdrop-saturate-200">backdrop-filter: saturate(2)</div>

<!-- Backdrop sepia -->
<div class="backdrop-sepia-0">backdrop-filter: sepia(0)</div>
<div class="backdrop-sepia">backdrop-filter: sepia(100%)</div>
```

---

## Interactivity

Control cursor, pointer events, user select, and scroll behavior.

### Cursor

```html
<div class="cursor-auto">cursor: auto</div>
<div class="cursor-default">cursor: default</div>
<div class="cursor-pointer">cursor: pointer</div>
<div class="cursor-wait">cursor: wait</div>
<div class="cursor-text">cursor: text</div>
<div class="cursor-move">cursor: move</div>
<div class="cursor-help">cursor: help</div>
<div class="cursor-not-allowed">cursor: not-allowed</div>
<div class="cursor-none">cursor: none</div>
<div class="cursor-context-menu">cursor: context-menu</div>
<div class="cursor-progress">cursor: progress</div>
<div class="cursor-cell">cursor: cell</div>
<div class="cursor-crosshair">cursor: crosshair</div>
<div class="cursor-vertical-text">cursor: vertical-text</div>
<div class="cursor-alias">cursor: alias</div>
<div class="cursor-copy">cursor: copy</div>
<div class="cursor-no-drop">cursor: no-drop</div>
<div class="cursor-grab">cursor: grab</div>
<div class="cursor-grabbing">cursor: grabbing</div>
<div class="cursor-all-scroll">cursor: all-scroll</div>
<div class="cursor-col-resize">cursor: col-resize</div>
<div class="cursor-row-resize">cursor: row-resize</div>
<div class="cursor-n-resize">cursor: n-resize</div>
<div class="cursor-e-resize">cursor: e-resize</div>
<div class="cursor-s-resize">cursor: s-resize</div>
<div class="cursor-w-resize">cursor: w-resize</div>
<div class="cursor-ne-resize">cursor: ne-resize</div>
<div class="cursor-nw-resize">cursor: nw-resize</div>
<div class="cursor-se-resize">cursor: se-resize</div>
<div class="cursor-sw-resize">cursor: sw-resize</div>
<div class="cursor-ew-resize">cursor: ew-resize</div>
<div class="cursor-ns-resize">cursor: ns-resize</div>
<div class="cursor-nesw-resize">cursor: nesw-resize</div>
<div class="cursor-nwse-resize">cursor: nwse-resize</div>
<div class="cursor-zoom-in">cursor: zoom-in</div>
<div class="cursor-zoom-out">cursor: zoom-out</div>
```

### Pointer Events

```html
<div class="pointer-events-none">pointer-events: none</div>
<div class="pointer-events-auto">pointer-events: auto</div>
```

### User Select

```html
<div class="select-none">user-select: none</div>
<div class="select-text">user-select: text</div>
<div class="select-all">user-select: all</div>
<div class="select-auto">user-select: auto</div>
```

### Resize

```html
<textarea class="resize-none">resize: none</textarea>
<textarea class="resize-y">resize: vertical</textarea>
<textarea class="resize-x">resize: horizontal</textarea>
<textarea class="resize">resize: both</textarea>
```

### Scroll Behavior

```html
<div class="scroll-auto">scroll-behavior: auto</div>
<div class="scroll-smooth">scroll-behavior: smooth</div>
```

### Scroll Margin

```html
<div class="scroll-m-0">scroll-margin: 0</div>
<div class="scroll-m-4">scroll-margin: 1rem</div>
<div class="scroll-m-8">scroll-margin: 2rem</div>

<!-- Per side -->
<div class="scroll-mt-4">scroll-margin-top: 1rem</div>
<div class="scroll-mr-4">scroll-margin-right: 1rem</div>
<div class="scroll-mb-4">scroll-margin-bottom: 1rem</div>
<div class="scroll-ml-4">scroll-margin-left: 1rem</div>

<!-- Axis -->
<div class="scroll-mx-4">scroll-margin-left: 1rem; scroll-margin-right: 1rem</div>
<div class="scroll-my-4">scroll-margin-top: 1rem; scroll-margin-bottom: 1rem</div>
```

### Scroll Padding

```html
<div class="scroll-p-0">scroll-padding: 0</div>
<div class="scroll-p-4">scroll-padding: 1rem</div>
<div class="scroll-p-8">scroll-padding: 2rem</div>

<!-- Per side -->
<div class="scroll-pt-4">scroll-padding-top: 1rem</div>
<div class="scroll-pr-4">scroll-padding-right: 1rem</div>
<div class="scroll-pb-4">scroll-padding-bottom: 1rem</div>
<div class="scroll-pl-4">scroll-padding-left: 1rem</div>

<!-- Axis -->
<div class="scroll-px-4">scroll-padding-left: 1rem; scroll-padding-right: 1rem</div>
<div class="scroll-py-4">scroll-padding-top: 1rem; scroll-padding-bottom: 1rem</div>
```

### Scroll Snap Align

```html
<div class="snap-start">scroll-snap-align: start</div>
<div class="snap-end">scroll-snap-align: end</div>
<div class="snap-center">scroll-snap-align: center</div>
<div class="snap-align-none">scroll-snap-align: none</div>
```

### Scroll Snap Stop

```html
<div class="snap-normal">scroll-snap-stop: normal</div>
<div class="snap-always">scroll-snap-stop: always</div>
```

### Scroll Snap Type

```html
<div class="snap-none">scroll-snap-type: none</div>
<div class="snap-x">scroll-snap-type: x var(--tw-scroll-snap-strictness)</div>
<div class="snap-y">scroll-snap-type: y var(--tw-scroll-snap-strictness)</div>
<div class="snap-both">scroll-snap-type: both var(--tw-scroll-snap-strictness)</div>
<div class="snap-mandatory">--tw-scroll-snap-strictness: mandatory</div>
<div class="snap-proximity">--tw-scroll-snap-strictness: proximity</div>
```

### Touch Action

```html
<div class="touch-auto">touch-action: auto</div>
<div class="touch-none">touch-action: none</div>
<div class="touch-pan-x">touch-action: pan-x</div>
<div class="touch-pan-left">touch-action: pan-left</div>
<div class="touch-pan-right">touch-action: pan-right</div>
<div class="touch-pan-y">touch-action: pan-y</div>
<div class="touch-pan-up">touch-action: pan-up</div>
<div class="touch-pan-down">touch-action: pan-down</div>
<div class="touch-pinch-zoom">touch-action: pinch-zoom</div>
<div class="touch-manipulation">touch-action: manipulation</div>
```

### Will Change

```html
<div class="will-change-auto">will-change: auto</div>
<div class="will-change-scroll">will-change: scroll-position</div>
<div class="will-change-contents">will-change: contents</div>
<div class="will-change-transform">will-change: transform</div>
```

### Caret Color

```html
<input class="caret-inherit">caret-color: inherit</input>
<input class="caret-current">caret-color: currentColor</input>
<input class="caret-transparent">caret-color: transparent</input>
<input class="caret-black">caret-color: black</input>
<input class="caret-white">caret-color: white</input>
<input class="caret-blue-500">caret-color: blue</input>
```

### Accent Color

```html
<input type="checkbox" class="accent-inherit">accent-color: inherit</input>
<input type="checkbox" class="accent-current">accent-color: currentColor</input>
<input type="checkbox" class="accent-transparent">accent-color: transparent</input>
<input type="checkbox" class="accent-blue-500">accent-color: blue</input>
<input type="checkbox" class="accent-auto">accent-color: auto</input>
```

### Appearance

```html
<select class="appearance-none">appearance: none</select>
<select class="appearance-auto">appearance: auto</select>
```

---

## Accessibility

Screen reader utilities and focus management.

### Screen Reader Only

```html
<!-- Visually hide but keep accessible to screen readers -->
<span class="sr-only">
  This text is hidden visually but read by screen readers
</span>

<!-- Undo sr-only (useful with focus states) -->
<a href="#" class="sr-only focus:not-sr-only">
  Skip to main content
</a>
```

### Focus Visible

```html
<!-- Only show focus styles when using keyboard navigation -->
<button class="focus:outline-none focus-visible:ring-2 focus-visible:ring-blue-500">
  Accessible button
</button>
```

### Focus Within

```html
<!-- Style parent when any child has focus -->
<div class="focus-within:ring-2 focus-within:ring-blue-500">
  <input type="text">
</div>
```

### Forced Colors

```html
<!-- Styles for Windows High Contrast Mode -->
<div class="forced-colors:border forced-colors:border-[ButtonText]">
  High contrast mode support
</div>
```

### Reduced Motion

```html
<!-- Respect user's motion preferences -->
<div class="motion-reduce:animate-none motion-reduce:transition-none">
  Respects reduced motion
</div>

<div class="motion-safe:animate-bounce">
  Only animate when motion is allowed
</div>
```

### Print Styles

```html
<div class="print:hidden">Hidden when printing</div>
<div class="print:block hidden">Only shown when printing</div>
```

---

## Best Practices and Common Patterns

### Centering Content

```html
<!-- Horizontal centering with margin -->
<div class="mx-auto max-w-md">Centered container</div>

<!-- Flexbox centering -->
<div class="flex items-center justify-center h-screen">
  Centered both ways
</div>

<!-- Grid centering -->
<div class="grid place-items-center h-screen">
  Grid centered
</div>

<!-- Absolute centering -->
<div class="absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2">
  Absolutely centered
</div>
```

### Card Component

```html
<div class="bg-white rounded-lg shadow-md p-6 hover:shadow-lg transition-shadow">
  <h3 class="text-lg font-semibold text-gray-900">Card Title</h3>
  <p class="mt-2 text-gray-600">Card content goes here.</p>
</div>
```

### Button Styles

```html
<!-- Primary button -->
<button class="bg-blue-500 hover:bg-blue-600 text-white font-medium py-2 px-4 rounded-lg transition-colors">
  Primary
</button>

<!-- Secondary button -->
<button class="bg-gray-200 hover:bg-gray-300 text-gray-800 font-medium py-2 px-4 rounded-lg transition-colors">
  Secondary
</button>

<!-- Outline button -->
<button class="border-2 border-blue-500 text-blue-500 hover:bg-blue-500 hover:text-white font-medium py-2 px-4 rounded-lg transition-colors">
  Outline
</button>

<!-- Ghost button -->
<button class="text-blue-500 hover:bg-blue-50 font-medium py-2 px-4 rounded-lg transition-colors">
  Ghost
</button>

<!-- Disabled button -->
<button class="bg-gray-300 text-gray-500 font-medium py-2 px-4 rounded-lg cursor-not-allowed" disabled>
  Disabled
</button>
```

### Form Inputs

```html
<!-- Text input -->
<input
  type="text"
  class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent outline-none transition"
  placeholder="Enter text..."
>

<!-- Select -->
<select class="w-full px-4 py-2 border border-gray-300 rounded-lg bg-white focus:ring-2 focus:ring-blue-500 focus:border-transparent outline-none">
  <option>Option 1</option>
  <option>Option 2</option>
</select>

<!-- Checkbox -->
<label class="flex items-center space-x-2 cursor-pointer">
  <input type="checkbox" class="w-4 h-4 text-blue-500 rounded focus:ring-blue-500">
  <span class="text-gray-700">Checkbox label</span>
</label>

<!-- Error state -->
<input
  type="text"
  class="w-full px-4 py-2 border-2 border-red-500 rounded-lg focus:ring-2 focus:ring-red-500 focus:border-transparent outline-none"
>
<p class="mt-1 text-sm text-red-500">Error message</p>
```

### Navigation

```html
<!-- Horizontal nav -->
<nav class="flex space-x-4">
  <a href="#" class="text-gray-600 hover:text-gray-900 font-medium">Home</a>
  <a href="#" class="text-blue-500 font-medium">Active</a>
  <a href="#" class="text-gray-600 hover:text-gray-900 font-medium">About</a>
</nav>

<!-- Mobile hamburger menu -->
<button class="md:hidden p-2">
  <span class="block w-6 h-0.5 bg-gray-600 mb-1"></span>
  <span class="block w-6 h-0.5 bg-gray-600 mb-1"></span>
  <span class="block w-6 h-0.5 bg-gray-600"></span>
</button>
```

### Responsive Patterns

```html
<!-- Mobile-first responsive design -->
<div class="
  grid
  grid-cols-1
  sm:grid-cols-2
  md:grid-cols-3
  lg:grid-cols-4
  gap-4
">
  <!-- Grid items -->
</div>

<!-- Hide/show based on breakpoint -->
<div class="hidden md:block">Desktop only</div>
<div class="block md:hidden">Mobile only</div>

<!-- Responsive text -->
<h1 class="text-2xl sm:text-3xl md:text-4xl lg:text-5xl font-bold">
  Responsive heading
</h1>

<!-- Responsive padding -->
<div class="p-4 sm:p-6 md:p-8 lg:p-12">
  Responsive padding
</div>
```

### Modal/Overlay

```html
<!-- Backdrop -->
<div class="fixed inset-0 bg-black/50 backdrop-blur-sm z-40">
  <!-- Modal -->
  <div class="fixed top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 bg-white rounded-xl shadow-2xl p-6 z-50 max-w-md w-full">
    <h2 class="text-xl font-semibold">Modal Title</h2>
    <p class="mt-2 text-gray-600">Modal content</p>
  </div>
</div>
```

### Loading States

```html
<!-- Spinner -->
<div class="animate-spin rounded-full h-8 w-8 border-4 border-gray-200 border-t-blue-500"></div>

<!-- Skeleton loading -->
<div class="animate-pulse space-y-4">
  <div class="h-4 bg-gray-200 rounded w-3/4"></div>
  <div class="h-4 bg-gray-200 rounded"></div>
  <div class="h-4 bg-gray-200 rounded w-5/6"></div>
</div>

<!-- Button loading state -->
<button class="bg-blue-500 text-white px-4 py-2 rounded-lg flex items-center space-x-2" disabled>
  <svg class="animate-spin h-4 w-4" viewBox="0 0 24 24">...</svg>
  <span>Loading...</span>
</button>
```

### Tooltips

```html
<div class="relative group">
  <button class="px-4 py-2 bg-gray-200 rounded">Hover me</button>
  <div class="absolute bottom-full left-1/2 -translate-x-1/2 mb-2 px-3 py-1 bg-gray-900 text-white text-sm rounded opacity-0 group-hover:opacity-100 transition-opacity pointer-events-none">
    Tooltip text
    <div class="absolute top-full left-1/2 -translate-x-1/2 border-4 border-transparent border-t-gray-900"></div>
  </div>
</div>
```

### Badge/Tag

```html
<!-- Simple badge -->
<span class="inline-block px-2 py-1 text-xs font-medium bg-blue-100 text-blue-800 rounded-full">
  Badge
</span>

<!-- Status badges -->
<span class="inline-block px-2 py-1 text-xs font-medium bg-green-100 text-green-800 rounded-full">Active</span>
<span class="inline-block px-2 py-1 text-xs font-medium bg-yellow-100 text-yellow-800 rounded-full">Pending</span>
<span class="inline-block px-2 py-1 text-xs font-medium bg-red-100 text-red-800 rounded-full">Inactive</span>
```

### Avatar

```html
<!-- Circle avatar -->
<img src="avatar.jpg" class="w-10 h-10 rounded-full object-cover" alt="Avatar">

<!-- Avatar with status -->
<div class="relative">
  <img src="avatar.jpg" class="w-10 h-10 rounded-full object-cover" alt="Avatar">
  <span class="absolute bottom-0 right-0 w-3 h-3 bg-green-500 border-2 border-white rounded-full"></span>
</div>

<!-- Avatar group -->
<div class="flex -space-x-2">
  <img src="avatar1.jpg" class="w-8 h-8 rounded-full border-2 border-white" alt="">
  <img src="avatar2.jpg" class="w-8 h-8 rounded-full border-2 border-white" alt="">
  <img src="avatar3.jpg" class="w-8 h-8 rounded-full border-2 border-white" alt="">
</div>
```

### Dark Mode

```html
<!-- Dark mode support -->
<div class="bg-white dark:bg-gray-900 text-gray-900 dark:text-white">
  Content that adapts to dark mode
</div>

<!-- Dark mode button -->
<button class="bg-gray-200 dark:bg-gray-700 text-gray-800 dark:text-white px-4 py-2 rounded">
  Dark mode button
</button>
```

### Breakpoint Reference

| Prefix | Minimum width | CSS |
|--------|--------------|-----|
| sm     | 640px        | @media (min-width: 640px) |
| md     | 768px        | @media (min-width: 768px) |
| lg     | 1024px       | @media (min-width: 1024px) |
| xl     | 1280px       | @media (min-width: 1280px) |
| 2xl    | 1536px       | @media (min-width: 1536px) |

### State Variants Reference

| Variant | Description |
|---------|-------------|
| hover   | On mouse hover |
| focus   | On element focus |
| focus-visible | Focus via keyboard |
| focus-within | When child has focus |
| active  | While being pressed |
| visited | Visited links |
| disabled | Disabled state |
| checked | Checked inputs |
| first   | First child |
| last    | Last child |
| odd     | Odd children |
| even    | Even children |
| group-hover | When parent group is hovered |
| peer-checked | When peer sibling is checked |

### Arbitrary Values

```html
<!-- Use any CSS value with square brackets -->
<div class="w-[137px]">Exact width</div>
<div class="h-[calc(100vh-80px)]">Calculated height</div>
<div class="bg-[#1da1f2]">Custom hex color</div>
<div class="text-[22px]">Custom font size</div>
<div class="grid-cols-[1fr_2fr_1fr]">Custom grid</div>
<div class="top-[117px]">Custom position</div>
```

---

## Additional Resources

- Official Tailwind CSS Documentation: https://tailwindcss.com/docs
- Tailwind CSS Cheat Sheet: https://nerdcave.com/tailwind-cheat-sheet
- Tailwind Play (Online Playground): https://play.tailwindcss.com
- Headless UI (Tailwind Components): https://headlessui.com
- Tailwind UI (Premium Components): https://tailwindui.com
