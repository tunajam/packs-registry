# Tailwind CSS v4 Context

Comprehensive guide to Tailwind CSS v4 - utility-first CSS framework with CSS-native configuration.

## Core Concepts (v4)

Tailwind v4 uses CSS-native configuration with `@theme` directive instead of JavaScript config files.

## Installation

```bash
# Install Tailwind CSS v4
npm install tailwindcss @tailwindcss/postcss

# postcss.config.mjs
export default {
  plugins: {
    "@tailwindcss/postcss": {},
  },
}
```

```css
/* app.css */
@import "tailwindcss";
```

## Theme Variables with @theme

```css
@import "tailwindcss";

@theme {
  /* Colors */
  --color-brand: oklch(0.72 0.11 221.19);
  --color-brand-light: oklch(0.85 0.08 221.19);
  --color-brand-dark: oklch(0.55 0.14 221.19);
  
  /* Fonts */
  --font-display: "Cal Sans", sans-serif;
  --font-body: "Inter", sans-serif;
  
  /* Spacing (extends default) */
  --spacing-18: 4.5rem;
  --spacing-22: 5.5rem;
  
  /* Breakpoints */
  --breakpoint-3xl: 120rem;
  
  /* Shadows */
  --shadow-soft: 0 4px 6px -1px rgb(0 0 0 / 0.05);
  
  /* Border Radius */
  --radius-pill: 9999px;
  
  /* Animations */
  --animate-fade-in: fade-in 0.3s ease-out;
  
  @keyframes fade-in {
    from { opacity: 0; }
    to { opacity: 1; }
  }
}
```

## Theme Variable Namespaces

| Namespace | Utilities |
|-----------|-----------|
| `--color-*` | `bg-*`, `text-*`, `border-*`, `fill-*`, etc. |
| `--font-*` | `font-*` (font-family) |
| `--text-*` | `text-*` (font-size) |
| `--font-weight-*` | `font-*` (weights) |
| `--spacing-*` | `p-*`, `m-*`, `gap-*`, `w-*`, `h-*`, etc. |
| `--radius-*` | `rounded-*` |
| `--shadow-*` | `shadow-*` |
| `--breakpoint-*` | `sm:`, `md:`, `lg:`, etc. |
| `--container-*` | `@sm:`, `@md:`, container queries |
| `--ease-*` | `ease-*` |
| `--animate-*` | `animate-*` |

## Overriding Default Theme

```css
@import "tailwindcss";

@theme {
  /* Override specific value */
  --breakpoint-sm: 30rem;
  
  /* Reset entire namespace */
  --color-*: initial;
  
  /* Define custom palette */
  --color-primary: oklch(0.6 0.2 250);
  --color-secondary: oklch(0.7 0.15 180);
}
```

## Utility Patterns

### Layout
```html
<!-- Flexbox -->
<div class="flex items-center justify-between gap-4">
<div class="flex flex-col md:flex-row">

<!-- Grid -->
<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
<div class="grid grid-cols-[200px_1fr_auto] gap-4">

<!-- Container -->
<div class="container mx-auto px-4">
<div class="max-w-prose mx-auto">
```

### Responsive Design
```html
<!-- Mobile-first breakpoints -->
<div class="w-full md:w-1/2 lg:w-1/3">
<div class="text-sm md:text-base lg:text-lg">
<div class="hidden md:block">

<!-- Container queries (v4) -->
<div class="@container">
  <div class="@md:grid-cols-2 @lg:grid-cols-3">
</div>
```

### Typography
```html
<h1 class="text-4xl font-bold tracking-tight text-gray-900">
<p class="text-base text-gray-600 leading-relaxed">
<span class="text-sm font-medium text-gray-500 uppercase tracking-wide">

<!-- Prose (typography plugin) -->
<article class="prose prose-lg prose-gray">
```

### Spacing & Sizing
```html
<!-- Padding/Margin -->
<div class="p-4 md:p-6 lg:p-8">
<div class="px-4 py-2">
<div class="mt-4 mb-8 mx-auto">

<!-- Width/Height -->
<div class="w-full max-w-md">
<div class="h-screen min-h-[400px]">
<div class="size-10"> <!-- w-10 h-10 -->
```

### Colors & Backgrounds
```html
<!-- Opacity modifiers -->
<div class="bg-black/50">
<div class="text-white/80">

<!-- Gradients -->
<div class="bg-gradient-to-r from-blue-500 to-purple-600">
<div class="bg-gradient-to-br from-gray-900 via-gray-800 to-gray-900">

<!-- Dark mode -->
<div class="bg-white dark:bg-gray-900 text-gray-900 dark:text-white">
```

### Borders & Effects
```html
<!-- Borders -->
<div class="border border-gray-200 rounded-lg">
<div class="border-b-2 border-blue-500">
<div class="ring-2 ring-blue-500 ring-offset-2">

<!-- Shadows -->
<div class="shadow-sm hover:shadow-md transition-shadow">
<div class="shadow-xl shadow-blue-500/20">
```

### Interactivity
```html
<!-- Hover/Focus -->
<button class="hover:bg-blue-600 focus:ring-2 focus:ring-blue-500">
<input class="focus:border-blue-500 focus:ring-1 focus:ring-blue-500">

<!-- Transitions -->
<div class="transition-all duration-300 ease-out">
<div class="transition-colors duration-200">

<!-- Transforms -->
<div class="hover:scale-105 hover:-translate-y-1 transition-transform">
```

### State Variants
```html
<!-- Group hover -->
<div class="group">
  <span class="group-hover:text-blue-500">
</div>

<!-- Peer states -->
<input class="peer" type="checkbox">
<label class="peer-checked:text-blue-500">

<!-- Data attributes -->
<div class="data-[state=open]:rotate-180">
<div class="data-[active=true]:bg-blue-500">

<!-- Aria states -->
<button class="aria-disabled:opacity-50 aria-expanded:bg-gray-100">
```

## Dark Mode

```css
/* Enable class-based dark mode */
@import "tailwindcss";

@variant dark (&:where(.dark, .dark *));
```

```html
<html class="dark">
  <div class="bg-white dark:bg-gray-900">
</html>

<!-- Or system preference -->
<html class="dark:bg-gray-900">
```

## Custom Utilities with @utility

```css
@import "tailwindcss";

@utility text-balance {
  text-wrap: balance;
}

@utility scrollbar-hide {
  -ms-overflow-style: none;
  scrollbar-width: none;
  &::-webkit-scrollbar {
    display: none;
  }
}
```

## Component Patterns

### Card
```html
<div class="rounded-xl border bg-card p-6 shadow-sm">
  <h3 class="text-lg font-semibold">Title</h3>
  <p class="mt-2 text-muted-foreground">Description</p>
</div>
```

### Button
```html
<button class="inline-flex items-center justify-center rounded-md bg-primary px-4 py-2 text-sm font-medium text-primary-foreground transition-colors hover:bg-primary/90 focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring disabled:pointer-events-none disabled:opacity-50">
  Click me
</button>
```

### Input
```html
<input class="flex h-10 w-full rounded-md border border-input bg-background px-3 py-2 text-sm ring-offset-background file:border-0 file:bg-transparent file:text-sm file:font-medium placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring disabled:cursor-not-allowed disabled:opacity-50">
```

## Performance Tips

1. **Use content paths correctly** - Only scan files that use Tailwind classes
2. **Avoid dynamic class names** - Use complete class names for tree-shaking
3. **Use @apply sparingly** - Prefer utility classes directly
4. **Leverage CSS variables** - For runtime theme changes

```css
/* ❌ Won't work - dynamic classes */
<div class={`text-${color}-500`}>

/* ✅ Works - complete classes */
<div class={color === 'red' ? 'text-red-500' : 'text-blue-500'}>
```

## Best Practices

1. **Mobile-first** - Start with mobile styles, add breakpoints for larger screens
2. **Semantic class order** - Layout → Box model → Typography → Visual → Misc
3. **Extract components** - Not utilities; let Tailwind handle the CSS
4. **Use design tokens** - Define colors, spacing in @theme for consistency
5. **Prefer CSS variables** - For values that might change at runtime
