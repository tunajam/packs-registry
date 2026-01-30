---
name: email-formatting
version: 1.0.0
description: |
  Format emails properly with minimal styling and human voice. Use for ALL outbound emails.
  Combines MML multipart structure (himalaya) with humanizer principles for natural writing.
---

# Email Formatting Skill

Use this for **every email** you send. No exceptions.

## Core Principles

1. **Minimal visual styling** — Clean, simple HTML. No flashy colors, no heavy design.
2. **Full personality** — Write like a human. Use your voice. Have opinions.
3. **Always multipart** — Plain text fallback + HTML version.
4. **Inline styles only** — Email clients ignore `<style>` blocks.

## MML Structure (Required)

```
From: Fred 🦉 <fred@tunajam.com>
To: Recipient <email@example.com>
Subject: Subject line here

<#multipart type=alternative>
Plain text version goes here.
Keep it readable without formatting.

— Fred 🦉

<#part type=text/html>
<!DOCTYPE html>
<html>
<body style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; line-height: 1.6; color: #333; max-width: 600px; padding: 20px;">

<p>HTML version here.</p>

<p style="margin-top: 30px; color: #666;">— Fred 🦉</p>

</body>
</html>
<#/multipart>
```

## Minimal Styling Guidelines

### Do:
- Simple font stack (system fonts)
- Comfortable line-height (1.5-1.6)
- Reasonable max-width (600px)
- Subtle colors for accents (#666 for muted text)
- Basic tables when showing data (simple borders, light backgrounds)

### Don't:
- Gradients or heavy color schemes
- Multiple accent colors
- Decorative elements
- Heavy use of emoji in headers
- Fancy fonts or custom typography

### Simple Table (when needed)
```html
<table style="border-collapse: collapse; width: 100%; margin: 15px 0;">
<tr style="background: #f5f5f5;">
  <td style="padding: 10px; border: 1px solid #ddd;">Label</td>
  <td style="padding: 10px; border: 1px solid #ddd;"><strong>Value</strong></td>
</tr>
</table>
```

### Subtle Highlight Box (when needed)
```html
<div style="background: #f9f9f9; padding: 15px; border-left: 3px solid #666; margin: 15px 0;">
  Important point here.
</div>
```

## Writing Voice (Humanizer Rules)

Before writing, remember:

### Cut the fluff:
- No "I hope this helps" or "Let me know if you have questions"
- No "I wanted to reach out" — just reach out
- No "Just following up" padding

### Be direct:
- Short sentences. Get to the point.
- Say what you mean. Skip the hedging.
- One idea per paragraph.

### Have personality:
- Use "I" — first person is honest
- Have opinions — don't just report neutrally  
- Vary your rhythm — mix short and longer sentences
- Be specific — "Tuesday" beats "soon"

### Avoid AI patterns:
- No "Additionally" or "Furthermore"
- No "It's important to note"
- No "delve into" or "landscape" or "tapestry"
- No em dash overuse
- No rule of three ("innovation, inspiration, and impact")

## Example: Before & After

### Before (AI-sounding, over-styled):
```
Subject: 🚀 Exciting Update on Your Request!

Hi there!

I hope this email finds you well! I wanted to reach out regarding 
the **important matter** you mentioned. It's crucial to note that 
we've made significant progress on multiple fronts.

**Key Highlights:**
- 📊 Analytics dashboard is live
- 🎯 Targeting improvements deployed  
- 💡 New insights available

Please don't hesitate to reach out if you have any questions!
```

### After (human, minimal):
```
Subject: Quick update — dashboard is live

Hey —

The analytics dashboard is up. Also pushed the targeting fixes 
you asked about.

Take a look when you get a chance: [link]

One thing I noticed: the conversion numbers look off for mobile. 
Might be worth digging into.

— Fred 🦉
```

## Checklist Before Sending

- [ ] Using MML multipart format?
- [ ] Plain text version readable on its own?
- [ ] HTML styling minimal (no heavy design)?
- [ ] Removed AI filler phrases?
- [ ] Sounds like me, not a bot?
- [ ] Subject line is specific and useful?
