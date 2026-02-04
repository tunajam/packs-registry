# Skill: Frontend Design

Build frontend interfaces that look intentionally designed, not AI-generated. This is your guide to making things that feel like a human designer sweated the details.

## When to Use

Activate this skill when building any user-facing interface: landing pages, dashboards, components, full apps, marketing sites. If pixels are involved, this applies.

## The Problem With AI-Generated UI

Most AI output looks the same. You've seen it: Inter font, purple-to-blue gradient, white cards with rounded corners, evenly-spaced grids. It's technically fine and emotionally dead. The goal isn't "correct" — it's "memorable."

## Before You Write Code

Spend 30 seconds on these questions. Skip this and you'll build something generic.

1. **Who is this for?** A developer tool feels different than a kids' app. A fintech dashboard isn't a music player.
2. **What's the one thing?** Every good interface has one detail someone remembers. A signature interaction. An unexpected color. A layout that breaks convention. Pick yours.
3. **What's the mood?** Not "professional" — that's a cop-out. Try: tense, playful, serene, raw, luxurious, chaotic, clinical, warm. Mood drives every decision downstream.

## Typography

Typography is 80% of design. Get this right and you can phone in the rest.

**The rules:**
- **Pick a display font with character.** Google Fonts has hundreds of good options. Literally anything except Inter, Roboto, Arial, Open Sans, or system-ui. Those are the "I didn't choose a font" fonts.
- **Pair it.** Display + body. Contrast in style (serif display + sans body, geometric display + humanist body). Match in spirit.
- **Size with conviction.** Hero text should be massive (4xl-7xl). Don't be timid. Big type on a page is a design decision. Medium type is a missed opportunity.
- **Mind the details.** Letter-spacing on uppercase text. Line-height on body copy (1.5-1.7). Font-weight variation creates hierarchy without color.

**Good pairings to start from (but don't reuse across projects):**
- Clash Display + Satoshi
- Fraunces + Work Sans
- Cabinet Grotesk + General Sans
- Playfair Display + Source Sans 3
- Space Mono + DM Sans
- Instrument Serif + Instrument Sans
- Syne + Inter Tight (Inter Tight is fine as body — Inter as display is the sin)

**Load fonts properly:**
```tsx
// Next.js — use next/font for zero layout shift
import { Syne, DM_Sans } from 'next/font/google'

const display = Syne({ subsets: ['latin'], variable: '--font-display' })
const body = DM_Sans({ subsets: ['latin'], variable: '--font-body' })
```

## Color

Color is emotion. A palette isn't a list of hex codes — it's a feeling.

**The rules:**
- **One dominant color.** Not five equal ones. One color owns the page. Everything else supports it.
- **Avoid the AI palette.** Purple-to-blue gradients, teal accents on white, the "SaaS starter kit" look. You know it when you see it.
- **Dark modes should be designed, not inverted.** Dark backgrounds aren't just `bg-gray-900`. Try deep blues (`slate-950`), warm darks (`stone-950`), or tinted blacks. Pure `#000` is harsh.
- **Accent colors should surprise.** Amber on deep navy. Lime on charcoal. Coral on cream. The unexpected pairing is the one people remember.

**With Tailwind / CSS variables:**
```css
/* Don't just use Tailwind defaults — extend the palette */
:root {
  --color-surface: 250 250 247;    /* warm off-white, not pure white */
  --color-ink: 28 25 23;           /* warm near-black, not pure black */
  --color-accent: 234 88 12;       /* a real orange, not a safe blue */
}
```

**Gradients that work:**
- Subtle mesh gradients for backgrounds (CSS `radial-gradient` layered)
- Single-hue gradients (light amber to deep amber) over multi-hue
- Gradient text only on hero elements, never on body copy

## Layout

The grid is a tool, not a religion. The most interesting interfaces break it intentionally.

**The rules:**
- **Asymmetry is your friend.** A 7/5 column split is more interesting than 6/6. Offset elements. Let things breathe unevenly.
- **Negative space is a feature.** Cramming content into every pixel is a sign of fear. Let sections breathe. A hero with 40vh of padding and 3 words hits harder than a wall of content.
- **Break the container.** Let images bleed to the edge. Overlap elements. Pull quotes that extend past the content width. These signal intentional design.
- **Density has its place.** Dashboards, data tables, admin panels — these should be dense. The mistake is making everything dense OR everything spacious. Match density to context.

**With Tailwind:**
```tsx
{/* Asymmetric hero — not centered, not balanced */}
<section className="grid grid-cols-12 min-h-[80vh] items-end pb-24">
  <div className="col-span-12 md:col-span-7 md:col-start-2">
    <h1 className="text-6xl md:text-8xl font-display tracking-tight">
      Something worth<br />remembering
    </h1>
  </div>
</section>

{/* Bento-style grid — varied sizes create rhythm */}
<div className="grid grid-cols-2 md:grid-cols-4 gap-3">
  <div className="col-span-2 row-span-2 bg-accent rounded-2xl p-8">Feature</div>
  <div className="bg-surface rounded-2xl p-6">Stat</div>
  <div className="bg-surface rounded-2xl p-6">Stat</div>
  <div className="col-span-2 bg-surface rounded-2xl p-6">Wide card</div>
</div>
```

## Motion & Interaction

Animation should feel like the interface is alive, not like it's performing.

**The rules:**
- **Entrance animations:** Stagger children on page load. Fade + slight translate (8-12px, not 20+). Keep durations short (200-400ms). Ease-out, not linear.
- **Micro-interactions over spectacle.** A button that subtly scales on hover. A card that lifts its shadow. An input that glows on focus. These compound into "this feels polished."
- **One signature moment per page.** A number that counts up. A section that reveals on scroll. A cursor effect. Don't animate everything — animate one thing memorably.
- **Respect `prefers-reduced-motion`.** Always. Wrap animations in the media query or use Tailwind's `motion-safe:` prefix.

**CSS-first when possible:**
```css
/* Staggered entrance — no JS needed */
@keyframes fade-up {
  from { opacity: 0; transform: translateY(8px); }
  to { opacity: 1; transform: translateY(0); }
}

.stagger-children > * {
  animation: fade-up 0.4s ease-out both;
}
.stagger-children > *:nth-child(1) { animation-delay: 0ms; }
.stagger-children > *:nth-child(2) { animation-delay: 60ms; }
.stagger-children > *:nth-child(3) { animation-delay: 120ms; }

/* Respect user preference */
@media (prefers-reduced-motion: reduce) {
  .stagger-children > * { animation: none; }
}
```

**For React — use Framer Motion sparingly:**
```tsx
// Page-level entrance, not per-element
<motion.div
  initial={{ opacity: 0, y: 8 }}
  animate={{ opacity: 1, y: 0 }}
  transition={{ duration: 0.3, ease: 'easeOut' }}
>
  {children}
</motion.div>
```

## Component Libraries (shadcn/ui, Radix, etc.)

Component libraries are foundations, not finished products. Using shadcn out of the box is like using a WordPress theme — functional but forgettable.

**The rules:**
- **Override the defaults.** Change the border-radius. Swap the font. Adjust the color tokens. The default shadcn theme is intentionally neutral — it's meant to be customized.
- **Extend, don't wrap.** Add variants to existing components instead of wrapping them in custom divs with overrides.
- **Icons matter.** Lucide (shadcn default) is fine, but Phosphor, Heroicons, or Tabler give you a different character. Pick one set and commit.

```tsx
// shadcn Button with personality — not the default
<Button
  className="rounded-full bg-amber-500 text-stone-950 hover:bg-amber-400
             font-semibold tracking-wide shadow-lg shadow-amber-500/25
             transition-all hover:shadow-xl hover:shadow-amber-500/30
             hover:-translate-y-0.5 active:translate-y-0"
>
  Get started
</Button>
```

## Texture & Depth

Flat design is fine. Flat AND generic is death. Add texture.

**Techniques:**
- **Noise/grain overlay.** A subtle SVG noise filter at 3-5% opacity adds warmth. Works on any background.
- **Layered shadows.** Not one `shadow-lg` — stack multiple shadows at different offsets and blurs for realistic depth.
- **Border subtlety.** `border-black/5` on light mode, `border-white/10` on dark. Barely there, but it separates elements.
- **Background patterns.** Dot grids, subtle lines, topographic patterns. CSS-only or lightweight SVG.

```css
/* Noise texture overlay — apply to a ::before pseudo-element */
.noise::before {
  content: '';
  position: absolute;
  inset: 0;
  background: url("data:image/svg+xml,%3Csvg viewBox='0 0 256 256' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.9' numOctaves='4' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23n)'/%3E%3C/svg%3E");
  opacity: 0.03;
  pointer-events: none;
  mix-blend-mode: overlay;
}

/* Layered shadow for depth */
.card-elevated {
  box-shadow:
    0 1px 2px rgba(0,0,0,0.04),
    0 4px 8px rgba(0,0,0,0.04),
    0 12px 24px rgba(0,0,0,0.06);
}
```

## Responsive Design

Mobile isn't a shrunk desktop. It's a different context.

**The rules:**
- **Design mobile-first, then expand.** Tailwind's responsive prefixes (`md:`, `lg:`) work this way by default. Use them.
- **Touch targets: 44px minimum.** Buttons, links, interactive elements. No exceptions.
- **Stack, don't shrink.** Side-by-side layouts on desktop should stack on mobile, not compress. Horizontal scrolling is (almost) never the answer.
- **Typography scales.** Hero text at `text-4xl` mobile, `text-7xl` desktop. Use `clamp()` or Tailwind responsive classes.

## Dark Mode

Dark mode isn't an afterthought. Design it or don't ship it.

**The rules:**
- **Tinted darks over pure gray.** `slate-950` (blue-tinted), `stone-950` (warm), `zinc-950` (cool). Pure `gray-950` is lifeless.
- **Reduce contrast slightly.** Text at `text-gray-100` or `text-gray-200`, not pure white. Less strain, more sophistication.
- **Shadows become glows.** Drop shadows don't work on dark backgrounds. Use subtle border highlights or soft glows instead.
- **Images need adjustment.** Slightly reduce brightness on images in dark mode. `brightness-90` in Tailwind.

```tsx
{/* Dark mode with tinted background and reduced contrast */}
<div className="bg-white dark:bg-stone-950 text-stone-900 dark:text-stone-100">
  <div className="border border-stone-200 dark:border-stone-800
                  shadow-lg dark:shadow-none dark:ring-1 dark:ring-stone-800">
    {/* Card content */}
  </div>
</div>
```

## The Checklist

Before shipping, verify:

- [ ] **Font:** Not Inter, Roboto, Arial, or system default
- [ ] **Color:** Not purple-to-blue gradient on white
- [ ] **Layout:** At least one asymmetric or grid-breaking element
- [ ] **Motion:** Entry animation or meaningful micro-interaction exists
- [ ] **Dark mode:** Designed (tinted darks, adjusted contrast), not auto-inverted
- [ ] **Mobile:** Tested at 375px width, touch targets are 44px+
- [ ] **Texture:** At least one of: noise, layered shadows, subtle borders, background pattern
- [ ] **The one thing:** Can you point to a single detail someone would remember?

## What NOT to Do

These are the tells that an AI built this:

- ❌ Inter/Roboto as the only font
- ❌ Purple-blue gradients everywhere
- ❌ Perfect 50/50 symmetry on every section
- ❌ White cards, gray background, blue buttons — the "SaaS starter pack"
- ❌ `shadow-lg` on everything with no other depth technique
- ❌ Centered text for everything, including paragraphs
- ❌ Stock gradient orbs floating in the background
- ❌ Every section having exactly 3 feature cards
- ❌ Generic placeholder copy ("Transform your workflow")
- ❌ Animations on every single element
- ❌ Using the same aesthetic across different projects

## Final Note

Good design isn't about complexity. A black-and-white page with perfect typography and generous whitespace beats a rainbow gradient with parallax scrolling. The point is intention. Every choice — font, color, spacing, animation — should be a deliberate decision, not a default.

Make it look like someone cared. Because they did.
