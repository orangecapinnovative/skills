---
name: tailwindcss-styling
description: "Enforces Tailwind CSS utility classes via className instead of inline styles. Always uses canonical class syntax (suggestCanonicalClasses) — CSS variable shorthand and direct utilities over arbitrary brackets."
category: frontend
risk: safe
source: local
tags: "[tailwind, css, styling, className, frontend]"
date_added: "2026-05-22"
---

# tailwindcss-styling

## Purpose

To enforce Tailwind CSS utility-first styling conventions: always use `className` with Tailwind utility classes instead of inline `style` attributes. This keeps styles co-located with markup, fully responsive, theme-aware, and tree-shakeable.

## When to Use This Skill

This skill should be used when:
- Writing or modifying any React/Next.js component UI
- Reviewing code that uses inline `style={{...}}` attributes
- Refactoring existing inline styles to Tailwind classes
- Adding new visual styling (colors, spacing, layout, typography)
- Implementing responsive design or dark-mode variants
- Reviewing code that uses verbose arbitrary-value syntax where a canonical form exists

---

## Core Rule

**Prefer `className` with Tailwind utility classes. Avoid `style={{...}}` inline attributes.**

```tsx
// ✅ Correct — Tailwind className
<div className="flex items-center gap-4 px-6 py-3 bg-blue-600 text-white rounded-lg shadow-md">
  Content
</div>

// ❌ Avoid — inline style
<div style={{ display: 'flex', alignItems: 'center', gap: '16px', padding: '12px 24px', backgroundColor: '#2563eb', color: '#fff', borderRadius: '8px', boxShadow: '0 4px 6px ...' }}>
  Content
</div>
```

---

## Allowed Exceptions for Inline Styles

Inline `style` is acceptable **only** when:

1. **Dynamic computed values** that have no Tailwind equivalent — e.g., percentage widths from data, canvas dimensions, or animation keyframe offsets:
   ```tsx
   // Acceptable: value is truly dynamic and has no Tailwind utility
   <div style={{ width: `${progress}%` }} className="h-2 bg-green-500 rounded" />
   ```

2. **CSS custom property (design token) injection** — passing a token into a variable slot:
   ```tsx
   // Acceptable: setting a CSS variable, not raw style values
   <div style={{ '--hue': hue } as React.CSSProperties} className="bg-[hsl(var(--hue),70%,50%)]" />
   ```

3. **Third-party component APIs** that only accept a `style` prop and provide no `className` prop.

In all other cases, use Tailwind classes.

---

## Standard Tailwind Syntax Reference

### Layout

```tsx
// Flexbox
<div className="flex items-center justify-between gap-4" />
<div className="flex flex-col gap-2" />

// Grid
<div className="grid grid-cols-3 gap-6" />
<div className="grid grid-cols-[1fr_2fr] gap-4" />

// Sizing
<div className="w-full max-w-lg h-screen min-h-0" />

// Positioning
<div className="relative">
  <span className="absolute inset-0" />
</div>
```

### Spacing

```tsx
// Padding
<div className="p-4 px-6 py-3 pt-2 pr-4 pb-2 pl-4" />

// Margin
<div className="m-auto mt-4 mx-6 my-2" />

// Space between children
<ul className="space-y-2" />
```

### Typography

```tsx
<p className="text-sm font-medium text-gray-700 leading-relaxed tracking-wide" />
<h1 className="text-2xl font-bold text-gray-900" />
<span className="text-xs text-muted-foreground uppercase" />
```

### Colors & Backgrounds

```tsx
<div className="bg-white dark:bg-gray-900 text-gray-900 dark:text-gray-100" />
<button className="bg-blue-600 hover:bg-blue-700 text-white" />
<div className="border border-gray-200 dark:border-gray-700" />
```

### Borders & Shadows

```tsx
<div className="rounded-lg border border-gray-200 shadow-sm" />
<div className="rounded-full ring-2 ring-blue-500 ring-offset-2" />
```

### Responsive Design

Use breakpoint prefixes — mobile-first:

```tsx
<div className="flex flex-col md:flex-row gap-4 md:gap-6" />
<p className="text-sm lg:text-base" />
<div className="hidden md:block" />
```

Breakpoints: `sm` (640px), `md` (768px), `lg` (1024px), `xl` (1280px), `2xl` (1536px).

### State Variants

```tsx
<button className="bg-blue-600 hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500 active:scale-95 disabled:opacity-50 disabled:cursor-not-allowed" />
<a className="text-blue-600 hover:text-blue-800 visited:text-purple-600" />
```

### Dark Mode

```tsx
<div className="bg-white dark:bg-gray-900 text-gray-900 dark:text-white border-gray-200 dark:border-gray-700" />
```

### Arbitrary Values

When no standard utility exists, use square-bracket notation rather than inline style:

```tsx
// ✅ Arbitrary value via className
<div className="w-[340px] top-[72px] bg-[#ff6b6b] grid-cols-[repeat(auto-fill,minmax(200px,1fr))]" />

// ❌ Avoid
<div style={{ width: '340px', top: '72px', backgroundColor: '#ff6b6b' }} />
```

### CSS Variable Tokens (Design System Integration)

When the project uses CSS custom properties as design tokens, use the **canonical CSS variable shorthand** — parentheses, not `[var(...)]` brackets (see Canonical Class Syntax below):

```tsx
// ✅ Canonical shorthand (Tailwind v4)
<div className="bg-(--color-primary) text-(--color-text) shadow-(--shadow-md)" />

// ❌ Verbose — avoid
<div className="bg-[var(--color-primary)] text-[var(--color-text)] shadow-[var(--shadow-md)]" />
```

---

## Canonical Class Syntax (suggestCanonicalClasses)

Follow the `suggestCanonicalClasses` rule from Tailwind CSS IntelliSense: always prefer the shortest canonical form over verbose arbitrary-value brackets when an equivalent exists.

### Rule 1 — CSS Variable Shorthand

Replace `[var(--x)]` with `(--x)` for any CSS custom property:

| Verbose (avoid) | Canonical (prefer) |
|---|---|
| `bg-[var(--color-primary)]` | `bg-(--color-primary)` |
| `text-[var(--color-text)]` | `text-(--color-text)` |
| `border-[var(--color-primary)]` | `border-(--color-primary)` |
| `ring-[var(--color-ring)]` | `ring-(--color-ring)` |
| `fill-[var(--icon-color)]` | `fill-(--icon-color)` |
| `p-[var(--spacing-md)]` | `p-(--spacing-md)` |
| `w-[var(--sidebar-width)]` | `w-(--sidebar-width)` |
| `shadow-[var(--shadow-card)]` | `shadow-(--shadow-card)` |

This applies to **any utility prefix** that accepts a color, size, or length value.

### Rule 2 — Direct Numeric Utilities

When the arbitrary value is a bare number or a value that directly maps to a Tailwind utility name, drop the brackets:

| Verbose (avoid) | Canonical (prefer) |
|---|---|
| `duration-[120ms]` | `duration-120` |
| `duration-[200ms]` | `duration-200` |
| `duration-[300ms]` | `duration-300` |
| `delay-[100ms]` | `delay-100` |
| `delay-[150ms]` | `delay-150` |
| `opacity-[50%]` | `opacity-50` |
| `opacity-[75%]` | `opacity-75` |
| `z-[10]` | `z-10` |
| `z-[50]` | `z-50` |
| `order-[2]` | `order-2` |
| `col-span-[3]` | `col-span-3` |
| `row-span-[2]` | `row-span-2` |
| `grow-[1]` | `grow` |
| `shrink-[0]` | `shrink-0` |

### Rule 3 — Scale-Mapped Spacing

When a pixel value maps cleanly to a Tailwind spacing scale unit (1 unit = 4 px), use the scale name:

| Verbose (avoid) | Canonical (prefer) |
|---|---|
| `w-[16px]` | `w-4` |
| `h-[32px]` | `h-8` |
| `p-[8px]` | `p-2` |
| `gap-[12px]` | `gap-3` |
| `mt-[24px]` | `mt-6` |
| `rounded-[4px]` | `rounded` |
| `rounded-[8px]` | `rounded-lg` |

**When to keep brackets**: the value does not divide evenly on the 4 px scale, or the exact pixel value is meaningful and must not be rounded (e.g., `w-[340px]`, `top-[72px]`).

### Quick-Reference: Canonical vs Arbitrary

```tsx
// ✅ All canonical
<button className="
  bg-(--color-primary)
  text-(--color-text-inverted)
  border-(--color-border)
  duration-150
  opacity-90
  z-10
  gap-3
  rounded-lg
" />

// ❌ All verbose — avoid
<button className="
  bg-[var(--color-primary)]
  text-[var(--color-text-inverted)]
  border-[var(--color-border)]
  duration-[150ms]
  opacity-[90%]
  z-[10]
  gap-[12px]
  rounded-[8px]
" />
```

---

## Refactoring Checklist

When converting inline styles to Tailwind:

| Inline style property | Tailwind equivalent |
|---|---|
| `display: flex` | `flex` |
| `flexDirection: column` | `flex-col` |
| `alignItems: center` | `items-center` |
| `justifyContent: space-between` | `justify-between` |
| `gap: 16px` | `gap-4` |
| `padding: 12px 24px` | `py-3 px-6` |
| `margin: 0 auto` | `mx-auto` |
| `width: 100%` | `w-full` |
| `borderRadius: 8px` | `rounded-lg` |
| `fontWeight: 600` | `font-semibold` |
| `fontSize: 14px` | `text-sm` |
| `color: #6b7280` | `text-gray-500` |
| `backgroundColor: #fff` | `bg-white` |
| `boxShadow: ...` | `shadow-sm` / `shadow-md` / `shadow-lg` |
| `overflow: hidden` | `overflow-hidden` |
| `position: absolute` | `absolute` |
| `zIndex: 10` | `z-10` |
| `cursor: pointer` | `cursor-pointer` |
| `opacity: 0.5` | `opacity-50` |
| `transition: all 200ms` | `transition-all duration-200` |

---

## Limitations

- Dynamic values that must be computed at runtime (e.g., from a DB or user input) are the one valid exception — use inline style for those values only, and still apply all other styling via className.
- Do not use `@apply` to extract classes unless creating a shared component that is re-used many times across the codebase.
- The CSS variable shorthand `(--x)` requires Tailwind v4. On v3 projects, keep `[var(--x)]`.
- This skill does not configure Tailwind itself — see the `tailwind-patterns` skill for v4 configuration.
