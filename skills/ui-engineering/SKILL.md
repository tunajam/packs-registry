# UI Engineering Skill

Excellence in interface design. Patterns, anti-patterns, and learnings from real builds.

## Core Principle

**Stability over cleverness.** Users should feel confident that the interface won't surprise them. Interactive elements stay put. Content doesn't jump. Actions have predictable outcomes.

---

## Layout Stability

### Rule: Interactive elements must not move when interacted with

When a user clicks a toggle, button, or control, that element (and nearby elements) should remain in the same position.

**Anti-pattern: Layout shift on toggle**
```tsx
// ❌ BAD - Toggle causes sibling elements to shift
<div className="flex gap-4">
  <Switch /> <span>Available only</span>
  {showPriceRange && <PriceSlider />}  // Appears/disappears, shifts everything
  <Button>Clear filters</Button>  // Moves left/right
</div>
```

**Pattern: Reserve space or use fixed positioning**
```tsx
// ✅ GOOD - Fixed-width containers, content doesn't shift
<div className="flex gap-4">
  <div className="w-40">  {/* Fixed width */}
    <Switch /> <span>Available only</span>
  </div>
  <div className="w-64">  {/* Fixed width, always takes space */}
    <span>Price range</span>
    <Switch />
    {showPriceRange && <PriceSlider />}  // Expands within its container
  </div>
  <Button className="ml-auto">Clear filters</Button>  {/* Anchored to right */}
</div>
```

**Techniques:**
- Use `min-w-*` or fixed widths on toggle containers
- Anchor buttons to edges with `ml-auto` or absolute positioning
- Use `opacity-0` instead of conditional rendering when hiding content that may reappear
- Reserve vertical space with `min-h-*` when content expands downward
- Consider `visibility: hidden` over `display: none` to preserve layout

### Rule: Expandable content should not push siblings

When a toggle reveals additional UI (like a slider), it should:
1. Expand downward (not sideways)
2. Or use overlay/popover positioning
3. Or pre-allocate the space

```tsx
// ✅ GOOD - Expansion happens vertically, in its own container
<div className="flex items-start gap-4">
  <div className="flex-shrink-0">
    <Switch /> Available only
  </div>
  <div className="flex-shrink-0 w-48">
    <div className="flex items-center justify-between">
      <span>Price range</span>
      <Switch />
    </div>
    {showPriceRange && (
      <div className="mt-2">
        <PriceSlider />  {/* Expands BELOW, not sideways */}
      </div>
    )}
  </div>
  <Button className="ml-auto flex-shrink-0">Clear filters</Button>
</div>
```

---

## Loading States

### Rule: Skeleton loaders should match final layout

Don't show a spinner that gets replaced by a completely different layout. Show a skeleton that mirrors the actual content dimensions.

```tsx
// ❌ BAD - Spinner, then content pops in
{loading ? <Spinner /> : <ProductCard />}

// ✅ GOOD - Skeleton matches final layout
{loading ? <ProductCardSkeleton /> : <ProductCard />}
```

### Rule: Buttons should not resize when loading

```tsx
// ❌ BAD - Button shrinks when text changes
<Button>{loading ? "..." : "Add to Cart"}</Button>

// ✅ GOOD - Fixed width or min-width
<Button className="min-w-[120px]">
  {loading ? <Spinner className="w-4 h-4" /> : "Add to Cart"}
</Button>
```

---

## Forms & Inputs

### Rule: Error messages should not shift form layout

Reserve space for error messages or use absolute positioning.

```tsx
// ❌ BAD - Error pushes content down
<Input />
{error && <p className="text-red-500">{error}</p>}
<Button>Submit</Button>

// ✅ GOOD - Space always reserved
<div className="mb-6">  {/* Fixed margin includes error space */}
  <Input />
  <p className={`text-red-500 text-sm h-5 ${error ? '' : 'invisible'}`}>
    {error || ' '}
  </p>
</div>
<Button>Submit</Button>
```

### Rule: Labels should not move when input state changes

Focus rings, validation states, and hover effects should not change element dimensions.

---

## Transitions & Animation

### Rule: Animate properties that don't cause reflow

Prefer `transform` and `opacity` over `width`, `height`, `margin`, `padding`.

```tsx
// ❌ BAD - Animates layout-triggering property
<div className="transition-all" style={{ height: expanded ? '200px' : '0' }}>

// ✅ GOOD - Uses transform for smooth animation
<div 
  className="transition-transform origin-top"
  style={{ transform: expanded ? 'scaleY(1)' : 'scaleY(0)' }}
>
```

### Rule: Content appearing should feel intentional

Use subtle fades or slides for new content. Avoid jarring pops.

```tsx
// ✅ Good patterns for appearing content
className="animate-in fade-in duration-200"
className="animate-in slide-in-from-top-2 duration-150"
```

---

## Responsive Design

### Rule: Touch targets should be at least 44x44px

Mobile users need adequate tap targets.

```tsx
// ❌ BAD - Tiny touch target
<button className="p-1 text-xs">×</button>

// ✅ GOOD - Adequate touch target
<button className="p-2 min-w-[44px] min-h-[44px] flex items-center justify-center">
  <X className="w-4 h-4" />
</button>
```

### Rule: Don't hide critical actions on mobile

If it's important on desktop, it's important on mobile. Adapt the layout, don't remove functionality.

---

## Visual Hierarchy

### Rule: One primary action per view

Don't have multiple competing "call to action" buttons. One should be clearly primary.

```tsx
// ❌ BAD - Competing buttons
<Button variant="default">Save</Button>
<Button variant="default">Publish</Button>
<Button variant="default">Preview</Button>

// ✅ GOOD - Clear hierarchy
<Button variant="outline">Preview</Button>
<Button variant="outline">Save Draft</Button>
<Button variant="default">Publish</Button>  {/* Primary action */}
```

### Rule: Use whitespace to group related elements

Don't rely solely on borders or backgrounds. Proximity is a powerful grouping tool.

---

## Accessibility

### Rule: All interactive elements need visible focus states

```tsx
// ✅ Always include focus-visible
className="focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring"
```

### Rule: Color should not be the only indicator

Pair color with icons, text, or patterns for colorblind users.

```tsx
// ❌ BAD - Only color indicates error
<Input className={error ? 'border-red-500' : ''} />

// ✅ GOOD - Color + icon + text
<div>
  <Input className={error ? 'border-red-500' : ''} />
  {error && (
    <p className="text-red-500 flex items-center gap-1">
      <AlertCircle className="w-4 h-4" /> {error}
    </p>
  )}
</div>
```

---

## Learnings Log

Document specific issues and fixes from our projects here.

### 2026-01-29: Wishing - Filter toggles causing layout shift

**Issue:** Toggle switches for "Available only" and "Price range" caused the entire filter bar to reflow. "Clear filters" button moved left/right.

**Root cause:** Flex container with `gap` and conditionally rendered content.

**Fix:** 
- Fixed widths on toggle containers
- Anchor "Clear filters" to the right with `ml-auto`
- Price range slider expands vertically, not horizontally

---

## Quick Checklist

Before shipping any UI:

- [ ] Click every interactive element — does anything shift?
- [ ] Toggle every toggle — stable?
- [ ] Expand/collapse every expandable — smooth?
- [ ] Tab through the page — focus states visible?
- [ ] Check on mobile — touch targets adequate?
- [ ] Test with slow network — loading states graceful?
- [ ] Check light/dark mode if applicable
