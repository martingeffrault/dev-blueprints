# Tailwind CSS v4 (2025)

> **Last updated**: January 2026
> **Versions covered**: 4.0+
> **Engine**: Oxide (Rust-based, 5x faster)

---

## Philosophy (2025-2026)

Tailwind CSS v4 is a **complete rewrite** with a CSS-first configuration approach and massive performance improvements.

**Key philosophical shifts:**
- **CSS-first configuration** — `@theme` directive replaces `tailwind.config.js`
- **Native CSS features** — Real `@layer`, `@property`, `color-mix()`
- **Zero config** — Works out of the box, automatic content detection
- **5x faster full builds** — 100x faster incremental builds (microseconds)
- **Modern browsers only** — Requires Safari 16.4+, Chrome 111+, Firefox 128+
- **Container queries built-in** — No plugin needed

---

## TL;DR

- Use `@import "tailwindcss"` instead of `@tailwind` directives
- Configure in CSS with `@theme`, not in JavaScript
- Container queries are built-in: `@container`, `@min-*`, `@max-*`
- Use `@utility` for custom utilities, `@custom-variant` for custom variants
- Modern syntax: `flex!` not `!flex`, `bg-linear-to-r` not `bg-gradient-to-r`
- Remove `@tailwindcss/container-queries` plugin (now built-in)
- Run upgrade tool: `npx @tailwindcss/upgrade`

---

## Best Practices

### Basic Setup (v4)

```css
/* app.css */
@import "tailwindcss";

/* That's it! No @tailwind directives needed */
```

### Custom Theme with @theme

```css
@import "tailwindcss";

@theme {
  /* Colors */
  --color-brand-50: oklch(97% 0.02 250);
  --color-brand-500: oklch(55% 0.25 250);
  --color-brand-900: oklch(25% 0.15 250);

  /* Typography */
  --font-display: "Cal Sans", sans-serif;
  --font-body: "Inter", system-ui, sans-serif;

  /* Spacing */
  --spacing-18: 4.5rem;
  --spacing-128: 32rem;

  /* Breakpoints */
  --breakpoint-3xl: 1920px;

  /* Animations */
  --animate-fade-in: fade-in 0.3s ease-out;
}

@keyframes fade-in {
  from { opacity: 0; }
  to { opacity: 1; }
}
```

### Custom Utilities

```css
@utility content-auto {
  content-visibility: auto;
}

@utility scrollbar-hidden {
  scrollbar-width: none;
  &::-webkit-scrollbar {
    display: none;
  }
}

@utility text-balance {
  text-wrap: balance;
}
```

### Custom Variants

```css
@custom-variant dark (&:where(.dark, .dark *));
@custom-variant hocus (&:hover, &:focus);
@custom-variant group-hocus (:merge(.group):hover &, :merge(.group):focus &);

/* Theme variant */
@custom-variant theme-midnight {
  &:where([data-theme="midnight"] *) {
    @slot;
  }
}
```

### Container Queries (Built-in)

```html
<!-- Parent container -->
<div class="@container">
  <!-- Responsive to container, not viewport -->
  <div class="@sm:flex @lg:grid @lg:grid-cols-3">
    <!-- @sm = container >= 24rem -->
    <!-- @lg = container >= 32rem -->
  </div>
</div>

<!-- Named containers -->
<div class="@container/sidebar">
  <div class="@lg/sidebar:hidden">
    Hidden when sidebar is large
  </div>
</div>
```

### Modern Class Syntax

```html
<!-- Important modifier: suffix not prefix -->
<div class="flex!">...</div>  <!-- v4 -->
<div class="!flex">...</div>  <!-- v3 (deprecated) -->

<!-- Gradients renamed -->
<div class="bg-linear-to-r from-blue-500 to-purple-500">...</div>  <!-- v4 -->
<div class="bg-gradient-to-r from-blue-500 to-purple-500">...</div> <!-- v3 -->

<!-- Arbitrary values unchanged -->
<div class="grid-cols-[1fr_2fr_1fr]">...</div>
<div class="text-[clamp(1rem,5vw,2rem)]">...</div>
```

### Dark Mode

```css
/* CSS-based dark mode */
@custom-variant dark (&:where(.dark, .dark *));

/* Or media query based */
@custom-variant dark (@media (prefers-color-scheme: dark));
```

```html
<html class="dark">
  <body class="bg-white dark:bg-gray-900">
    <p class="text-gray-900 dark:text-white">
      Adapts to dark mode
    </p>
  </body>
</html>
```

### Responsive Design

```html
<div class="
  flex flex-col
  sm:flex-row
  md:grid md:grid-cols-2
  lg:grid-cols-3
  xl:grid-cols-4
  2xl:grid-cols-6
">
  <!-- Mobile first responsive design -->
</div>
```

### OKLCH Colors

```css
@theme {
  /* OKLCH provides better perceptual uniformity */
  --color-primary-50: oklch(97% 0.01 250);
  --color-primary-100: oklch(94% 0.02 250);
  --color-primary-200: oklch(88% 0.05 250);
  --color-primary-300: oklch(80% 0.10 250);
  --color-primary-400: oklch(70% 0.15 250);
  --color-primary-500: oklch(60% 0.20 250);
  --color-primary-600: oklch(50% 0.20 250);
  --color-primary-700: oklch(40% 0.18 250);
  --color-primary-800: oklch(30% 0.15 250);
  --color-primary-900: oklch(20% 0.10 250);
}
```

---

## Anti-Patterns

### ❌ Using JavaScript Config for Simple Projects

**Why it's bad**: CSS config is simpler and faster to parse.

```javascript
// ❌ DON'T — tailwind.config.js (v3 style)
module.exports = {
  theme: {
    extend: {
      colors: {
        brand: '#3b82f6'
      }
    }
  }
}
```

```css
/* ✅ DO — CSS config */
@theme {
  --color-brand: #3b82f6;
}
```

### ❌ Using Removed/Renamed Utilities

**Why it's bad**: They don't exist in v4.

```html
<!-- ❌ DON'T — v3 syntax -->
<div class="!flex">...</div>
<div class="bg-gradient-to-r">...</div>
<div class="decoration-slice">...</div>

<!-- ✅ DO — v4 syntax -->
<div class="flex!">...</div>
<div class="bg-linear-to-r">...</div>
<div class="box-decoration-slice">...</div>
```

### ❌ Installing Container Queries Plugin

**Why it's bad**: Built into v4 core.

```bash
# ❌ DON'T
npm install @tailwindcss/container-queries

# ✅ DO — Just use it directly
# <div class="@container">...</div>
```

### ❌ Using @tailwind Directives

**Why it's bad**: Replaced by CSS @import.

```css
/* ❌ DON'T — v3 style */
@tailwind base;
@tailwind components;
@tailwind utilities;

/* ✅ DO — v4 style */
@import "tailwindcss";
```

### ❌ Inline Styles for Custom Values

**Why it's bad**: Arbitrary values work for almost everything.

```html
<!-- ❌ DON'T -->
<div style="width: 137px; margin-top: 23px;">...</div>

<!-- ✅ DO -->
<div class="w-[137px] mt-[23px]">...</div>
```

### ❌ Overly Long Class Strings

**Why it's bad**: Hard to read and maintain.

```html
<!-- ❌ DON'T -->
<button class="inline-flex items-center justify-center rounded-md text-sm font-medium ring-offset-background transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50 bg-primary text-primary-foreground hover:bg-primary/90 h-10 px-4 py-2">
  Click me
</button>

<!-- ✅ DO — Extract to component or @apply -->
<button class="btn btn-primary">Click me</button>
```

```css
/* Using @utility for common patterns */
@utility btn {
  @apply inline-flex items-center justify-center rounded-md text-sm font-medium;
  @apply transition-colors focus-visible:outline-none focus-visible:ring-2;
  @apply disabled:pointer-events-none disabled:opacity-50;
}

@utility btn-primary {
  @apply bg-primary text-primary-foreground hover:bg-primary/90;
}
```

### ❌ Not Using Class Sorting

**Why it's bad**: Inconsistent class order across codebase.

```bash
# ✅ DO — Install Prettier plugin
npm install -D prettier-plugin-tailwindcss
```

```json
// .prettierrc
{
  "plugins": ["prettier-plugin-tailwindcss"]
}
```

---

## v3 to v4 Migration

### Upgrade Tool

```bash
npx @tailwindcss/upgrade
```

### Key Changes

| v3 | v4 |
|----|-----|
| `@tailwind base/components/utilities` | `@import "tailwindcss"` |
| `tailwind.config.js` | `@theme` in CSS |
| `!flex` (important prefix) | `flex!` (important suffix) |
| `bg-gradient-to-*` | `bg-linear-to-*` |
| `decoration-slice` | `box-decoration-slice` |
| `container` config options | `@utility container` |
| `@tailwindcss/container-queries` | Built-in |

### Browser Requirements

| Browser | Minimum Version |
|---------|-----------------|
| Safari | 16.4 |
| Chrome | 111 |
| Firefox | 128 |
| Edge | 111 |

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 4.0 | Jan 2025 | Complete rewrite, CSS-first config, Oxide engine, 5x faster |
| 4.1 | Mar 2025 | Native `@starting-style` support, improved animations, bug fixes |
| 4.x | 2025 | Ongoing improvements, Tailwind Plus ecosystem updates |

### Tailwind Plus (2025)

Tailwind UI has been rebranded to **Tailwind Plus** with expanded features:
- All examples updated to support Tailwind CSS v4.0 with version picker (v4.0/v3.4)
- v4.0 snippets leverage new features for simplified code
- New component patterns taking advantage of CSS-first configuration

### v4.0 Key Features

- **Oxide Engine**: Rust-based, 5x faster full builds, 100x faster incremental (44ms → 5ms)
- **CSS-First Config**: `@theme` directive replaces JavaScript config
- **Modern CSS**: Native `@layer`, `@property`, `color-mix()`
- **Container Queries**: Built into core
- **Automatic Content Detection**: No need to configure template paths
- **35% Smaller**: Reduced dependency footprint
- **OKLCH Colors**: P3 wide-gamut colors (more vibrant blues/greens)

### Breaking Changes (v3 → v4)

| v3 | v4 |
|----|-----|
| `@tailwind base/components/utilities` | `@import "tailwindcss"` |
| `tailwind.config.js` | `@theme { }` in CSS |
| `border-gray-200` default | `border-currentColor` default |
| `ring-3` + `ring-blue-500` default | `ring-1` + `ring-currentColor` |
| `!flex` (prefix) | `flex!` (suffix) |
| `bg-gradient-to-r` | `bg-linear-to-r` |
| Content paths required | Auto-detected |
| Works with Sass/Less | Not compatible (Tailwind IS the preprocessor) |

### Migration

```bash
# Automatic upgrade tool
npx @tailwindcss/upgrade

# Manual steps:
# 1. Remove tailwind.config.js (move theme to @theme)
# 2. Replace @tailwind directives with @import
# 3. Update deprecated utilities (gradient → linear)
# 4. Test border/ring colors (now currentColor)
```

---

## Quick Reference

| Task | Solution |
|------|----------|
| Import Tailwind | `@import "tailwindcss"` |
| Custom colors | `@theme { --color-name: value; }` |
| Custom utility | `@utility name { ... }` |
| Custom variant | `@custom-variant name { ... }` |
| Container query | `@container` + `@sm:`, `@lg:` |
| Dark mode | `dark:` prefix + `@custom-variant dark` |
| Important | `flex!` (suffix) |
| Arbitrary value | `w-[137px]`, `text-[#1a2b3c]` |
| Class sorting | `prettier-plugin-tailwindcss` |

---

## Performance Checklist

1. **Purge unused CSS** — Automatic in v4
2. **Use CSS variables** — Native via `@theme`
3. **Minimize arbitrary values** — Define in theme when repeated
4. **Use class sorting** — Prettier plugin for consistency
5. **Layer your custom CSS** — Use `@layer` for specificity control

---

## Resources

- [Official Tailwind CSS v4 Documentation](https://tailwindcss.com/docs)
- [Tailwind CSS v4 Announcement](https://tailwindcss.com/blog/tailwindcss-v4)
- [Upgrade Guide](https://tailwindcss.com/docs/upgrade-guide)
- [Tailwind CSS Tips (v4)](https://www.nikolailehbr.ink/blog/tailwindcss-v4-tips/)
- [Prettier Plugin](https://github.com/tailwindlabs/prettier-plugin-tailwindcss)
