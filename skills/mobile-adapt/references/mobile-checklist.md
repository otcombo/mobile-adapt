# Mobile Adaptation Checklist

Rules are organized by severity. Each rule includes: check condition, fix code, and detection method.

---

## CRITICAL (Must Fix)

These issues make the page unusable on mobile. Always fix.

### C1: Viewport Meta Tag

**Check**: `<head>` contains `<meta name="viewport" content="width=device-width, initial-scale=1">`.

**Detection**: Search HTML for `<meta name="viewport"`. If missing or has `width=` set to a fixed pixel value.

**Fix**: Inject after `<meta charset="...">` in `<head>`. If no `<head>`, create one.

```html
<meta name="viewport" content="width=device-width, initial-scale=1">
```

If an existing viewport meta has incorrect content (e.g., `width=1024`), replace its `content` attribute.

---

### C2: Touch Action Manipulation (300ms Delay)

**Check**: CSS includes `touch-action: manipulation` on interactive elements.

**Detection**: Search CSS for `touch-action`. If absent on `a`, `button`, `input`, `select`, `textarea`.

**Fix**: Add to the mobile-adapt CSS block:

```css
/* mobile-adapt: auto-generated */
a, button, input, select, textarea, [role="button"] {
  touch-action: manipulation;
}
```

---

### C3: Text Size Adjust

**Check**: CSS includes `-webkit-text-size-adjust: 100%` on `html` or `body`.

**Detection**: Search CSS for `text-size-adjust`. If absent.

**Fix**: Add to the mobile-adapt CSS block:

```css
/* mobile-adapt: auto-generated */
html {
  -webkit-text-size-adjust: 100%;
  text-size-adjust: 100%;
}
```

---

### C4: Horizontal Overflow Prevention

**Check**: No element causes horizontal scrollbar on mobile viewports.

**Detection**:
1. Search CSS for fixed widths > 100vw (e.g., `width: 1200px` without responsive override).
2. Search for elements with `overflow-x` not set when content might overflow.
3. Playwright scroll test confirms.

**Fix**: Add to the mobile-adapt CSS block:

```css
/* mobile-adapt: auto-generated */
html, body {
  overflow-x: hidden;
  max-width: 100vw;
}

*, *::before, *::after {
  box-sizing: border-box;
}
```

**Note**: Also search for fixed-width containers (`width: XXXpx` where XXX > 375) and add responsive overrides:

```css
/* mobile-adapt: auto-generated - responsive width override */
@media (max-width: 839px) {
  .fixed-width-element {
    width: 100%;
    max-width: 100vw;
  }
}
```

---

## HIGH (Should Fix)

These issues significantly degrade mobile usability.

### H1: Touch Target Size (>= 44x44px)

**Check**: All interactive elements (buttons, links, inputs) have minimum 44x44px touch area.

**Detection**: Playwright `browser_evaluate` to measure bounding rects of all clickable elements. Flag any < 44px in either dimension.

**Fix**: Add to the mobile-adapt CSS block:

```css
/* mobile-adapt: auto-generated */
@media (max-width: 839px) {
  a, button, [role="button"] {
    min-height: 44px;
    min-width: 44px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
  }

  input, select, textarea {
    min-height: 44px;
    font-size: 16px; /* Prevents iOS zoom on focus */
  }
}
```

**Note**: Only apply min-height/min-width fix if the element is actually undersized. For inline links within text, use padding instead:

```css
/* mobile-adapt: auto-generated */
@media (max-width: 839px) {
  p a, li a, span a, td a {
    padding: 8px 4px;
    margin: -8px -4px;
  }
}
```

---

### H2: Responsive Images

**Check**: Images have `max-width: 100%` and `height: auto`.

**Detection**:
1. Search CSS for `img` rules. Check for `max-width`.
2. Search HTML for `<img>` tags without `loading="lazy"` on below-fold images.

**Fix**: Add to the mobile-adapt CSS block:

```css
/* mobile-adapt: auto-generated */
img, video, svg, picture {
  max-width: 100%;
  height: auto;
}
```

Add `loading="lazy"` to `<img>` tags that are not in the first viewport (skip the first 1-2 images).

---

### H3: Responsive Layout (No Fixed Width Containers)

**Check**: No container has a fixed width that exceeds mobile viewport.

**Detection**: Search CSS for patterns like `width: \d{4,}px` (4+ digit pixel widths) or `min-width: \d{3,}px` without corresponding media query overrides.

**Fix**: Add responsive overrides:

```css
/* mobile-adapt: auto-generated */
@media (max-width: 839px) {
  [style*="width"], .container, .wrapper, main, section, article {
    max-width: 100%;
    padding-left: 16px;
    padding-right: 16px;
    box-sizing: border-box;
  }

  table {
    display: block;
    overflow-x: auto;
    -webkit-overflow-scrolling: touch;
    max-width: 100%;
  }
}
```

---

### H4: Minimum Font Size (>= 14px)

**Check**: No text renders below 14px on mobile.

**Detection**: Search CSS for `font-size` values < 14px (e.g., `font-size: 10px`, `font-size: 0.7rem` assuming 16px base).

**Fix**: Add responsive override for small text:

```css
/* mobile-adapt: auto-generated */
@media (max-width: 839px) {
  body {
    font-size: 16px;
    line-height: 1.5;
  }

  small, .small, .caption, .footnote {
    font-size: 14px;
  }
}
```

---

### H5: Navigation Adaptation — Hamburger Menu

**Check**: Navigation menus adapt for Compact screens (< 600px) using a hamburger toggle pattern.

**Detection**:
1. Search for `<nav>` elements or common nav patterns (`.nav`, `.navbar`, `.menu`, `#navigation`).
2. Check if nav uses `display: flex` or `display: inline-block` horizontally without a `@media` override.
3. Check if a hamburger toggle already exists (look for `.hamburger`, `.menu-toggle`, `#menu-btn`, checkbox-based toggle).
4. Check Playwright snapshot at 375px for overflowing nav items.

**Fix**: Inject a CSS-only hamburger menu using the checkbox hack.

**Step 1 — Add hamburger toggle HTML** inside the `<nav>` (or its parent `<header>`), BEFORE the nav links:

```html
<!-- mobile-adapt: auto-generated hamburger toggle -->
<input type="checkbox" id="mobile-adapt-menu-toggle" class="mobile-adapt-menu-toggle" aria-hidden="true">
<label for="mobile-adapt-menu-toggle" class="mobile-adapt-hamburger" aria-label="Toggle navigation menu">
  <span class="mobile-adapt-hamburger-line"></span>
</label>
```

**Step 2 — Add CSS** for the hamburger behavior:

```css
/* mobile-adapt: auto-generated — hamburger menu */

/* Hide hamburger toggle checkbox */
.mobile-adapt-menu-toggle {
  display: none;
}

/* Hide hamburger icon on larger screens */
.mobile-adapt-hamburger {
  display: none;
}

@media (max-width: 599px) {
  /* Show hamburger icon */
  .mobile-adapt-hamburger {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 44px;
    height: 44px;
    cursor: pointer;
    position: relative;
    z-index: 100;
    -webkit-tap-highlight-color: transparent;
  }

  /* Hamburger icon lines */
  .mobile-adapt-hamburger-line {
    display: block;
    width: 24px;
    height: 2px;
    background: currentColor;
    position: relative;
    transition: background 0.3s;
  }

  .mobile-adapt-hamburger-line::before,
  .mobile-adapt-hamburger-line::after {
    content: '';
    display: block;
    width: 24px;
    height: 2px;
    background: currentColor;
    position: absolute;
    transition: transform 0.3s;
  }

  .mobile-adapt-hamburger-line::before {
    top: -7px;
  }

  .mobile-adapt-hamburger-line::after {
    top: 7px;
  }

  /* Animate to X when open */
  .mobile-adapt-menu-toggle:checked + .mobile-adapt-hamburger .mobile-adapt-hamburger-line {
    background: transparent;
  }

  .mobile-adapt-menu-toggle:checked + .mobile-adapt-hamburger .mobile-adapt-hamburger-line::before {
    transform: rotate(45deg);
    top: 0;
  }

  .mobile-adapt-menu-toggle:checked + .mobile-adapt-hamburger .mobile-adapt-hamburger-line::after {
    transform: rotate(-45deg);
    top: 0;
  }

  /* Hide nav links by default on compact screens */
  nav, .nav, .navbar, .menu, [role="navigation"] {
    max-height: 0;
    overflow: hidden;
    transition: max-height 0.3s ease-out;
    width: 100%;
  }

  /* Show nav links when hamburger is checked */
  .mobile-adapt-menu-toggle:checked ~ nav,
  .mobile-adapt-menu-toggle:checked ~ .nav,
  .mobile-adapt-menu-toggle:checked ~ .navbar,
  .mobile-adapt-menu-toggle:checked ~ .menu,
  .mobile-adapt-menu-toggle:checked ~ [role="navigation"],
  .mobile-adapt-menu-toggle:checked ~ * nav,
  .mobile-adapt-menu-toggle:checked ~ * .nav {
    max-height: 500px;
  }

  /* Stack nav items vertically */
  nav ul, nav ol, .nav, .menu ul {
    flex-direction: column;
    padding: 0;
    margin: 0;
    list-style: none;
  }

  nav a, .nav a, .menu a {
    display: block;
    padding: 12px 16px;
    min-height: 44px;
    border-bottom: 1px solid rgba(128, 128, 128, 0.2);
  }
}
```

**HTML injection rules**:
- Place the checkbox + label as the FIRST children inside the nav's parent container (e.g., `<header>`).
- The checkbox MUST be a sibling of (or ancestor-adjacent to) the `<nav>` element for the `~` selector to work.
- If the nav structure makes sibling targeting impossible, wrap the toggle + nav in a new `<div>` container.

**Note**: On Medium screens (600-839px), nav links remain visible in their original horizontal layout. The hamburger only appears on Compact (< 600px).

---

### H6: Mobile-First Media Query Recommendation

**Check**: Whether existing media queries use `min-width` (mobile-first) or `max-width` (desktop-first) approach.

**Detection**: Count `@media` rules. If primarily using `min-width`, note as already mobile-first. If using `max-width`, it's desktop-first — still functional but note the pattern.

**Fix**: No auto-fix. Report the current approach and recommend mobile-first for future development.

---

## MEDIUM (Recommended)

These improve the experience but are not critical.

### M1: Fluid Typography with clamp()

**Check**: Headings and body text use `clamp()` or `vw` units for fluid scaling.

**Detection**: Search CSS for heading rules (`h1`-`h6`). Check if font-size is fixed pixel/rem without responsive scaling.

**Fix**: Suggest (do not auto-apply unless headings overflow):

```css
/* mobile-adapt: auto-generated */
@media (max-width: 839px) {
  h1 { font-size: clamp(1.5rem, 5vw, 2.5rem); }
  h2 { font-size: clamp(1.25rem, 4vw, 2rem); }
  h3 { font-size: clamp(1.1rem, 3.5vw, 1.5rem); }
}
```

---

### M2: Safe Area (Notch) Handling

**Check**: Page accounts for notched devices using `env(safe-area-inset-*)`.

**Detection**: Search CSS for `env(safe-area-inset`. If absent and page has fixed/sticky positioned elements.

**Fix**: Add to the mobile-adapt CSS block:

```css
/* mobile-adapt: auto-generated */
@supports(padding: env(safe-area-inset-top)) {
  body {
    padding-left: env(safe-area-inset-left);
    padding-right: env(safe-area-inset-right);
  }

  /* Fixed bottom elements */
  [style*="position: fixed"][style*="bottom"],
  .fixed-bottom, .sticky-bottom {
    padding-bottom: env(safe-area-inset-bottom);
  }
}
```

Also add to viewport meta: `viewport-fit=cover`:

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
```

---

### M3: Form Input Optimization

**Check**: Form inputs use appropriate `type`, `inputmode`, and `autocomplete` attributes.

**Detection**: Search HTML for `<input>` tags. Check:
- Email fields: should have `type="email"` and `inputmode="email"`
- Phone fields: should have `type="tel"` and `inputmode="tel"`
- Number fields: should have `inputmode="numeric"` or `inputmode="decimal"`
- Search fields: should have `type="search"`
- Password fields: should have `autocomplete="current-password"` or `autocomplete="new-password"`

**Fix**: Update `<input>` attributes directly in HTML. Common patterns:

```html
<!-- Email -->
<input type="email" inputmode="email" autocomplete="email">

<!-- Phone -->
<input type="tel" inputmode="tel" autocomplete="tel">

<!-- Number -->
<input type="text" inputmode="numeric" pattern="[0-9]*">
```

---

### M4: Scrollable Container Touch Optimization

**Check**: Horizontally scrollable containers have `-webkit-overflow-scrolling: touch` and scroll snap.

**Detection**: Search CSS for `overflow-x: auto` or `overflow-x: scroll` or `overflow: auto`.

**Fix**: Add touch scrolling enhancement:

```css
/* mobile-adapt: auto-generated */
@media (max-width: 839px) {
  [style*="overflow"], .scroll-container, .carousel, .slider {
    -webkit-overflow-scrolling: touch;
    scroll-snap-type: x mandatory;
    scrollbar-width: none;
  }

  [style*="overflow"]::-webkit-scrollbar {
    display: none;
  }
}
```

---

### M5: Image Lazy Loading

**Check**: Below-fold images have `loading="lazy"`.

**Detection**: Search HTML for `<img>` tags. The first 1-2 images (likely hero/logo) should NOT be lazy. All others should have `loading="lazy"`.

**Fix**: Add `loading="lazy"` attribute to `<img>` tags (skip first 2):

```html
<img src="..." loading="lazy" alt="...">
```

---

### M6: Font Loading Optimization

**Check**: `@font-face` declarations include `font-display: swap`.

**Detection**: Search CSS for `@font-face`. Check if `font-display` is specified.

**Fix**: If `@font-face` exists without `font-display`, add it. If fonts are loaded via `<link>` (Google Fonts etc.), append `&display=swap` to the URL.

---

### M7: Focus Visible State

**Check**: Interactive elements have visible focus indicators.

**Detection**: Search CSS for `:focus` or `:focus-visible` rules. If absent, flag.

**Fix**: Add to the mobile-adapt CSS block:

```css
/* mobile-adapt: auto-generated */
:focus-visible {
  outline: 2px solid #005fcc;
  outline-offset: 2px;
}
```

---

### M8: Color Contrast (Flag Only)

**Check**: Text-to-background contrast meets WCAG AA (4.5:1 for normal text, 3:1 for large text).

**Detection**: Note any obvious low-contrast patterns (light gray on white, etc.) during code review.

**Fix**: **Do NOT auto-fix.** Report in the audit with specific selectors and suggest manual review. Changing colors may break the design intent.

---

### M9: Tap Highlight Customization

**Check**: Default tap highlight is customized for a cleaner mobile feel.

**Detection**: Search CSS for `-webkit-tap-highlight-color`. If absent.

**Fix**: Add to the mobile-adapt CSS block:

```css
/* mobile-adapt: auto-generated */
a, button, [role="button"] {
  -webkit-tap-highlight-color: rgba(0, 0, 0, 0.1);
}
```

---

## LOW (Nice to Have)

Report these but only auto-fix L4.

### L1: prefers-reduced-motion Support

**Check**: Animations respect `prefers-reduced-motion`.

**Detection**: Search CSS for `animation` or `transition` properties. If found, check for `@media (prefers-reduced-motion: reduce)`.

**Fix**: **Flag only.** Suggest:

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}
```

---

### L2: Print Styles

**Check**: Page has print-specific styles.

**Detection**: Search CSS for `@media print`.

**Fix**: **Flag only.** Note absence in report.

---

### L3: Dark Mode Support

**Check**: Page supports `prefers-color-scheme: dark`.

**Detection**: Search CSS for `prefers-color-scheme`.

**Fix**: **Flag only.** Note absence in report.

---

### L4: Text Line Width (<= 65ch)

**Check**: Main content text containers have a max-width that limits line length.

**Detection**: Check if main content containers (`<main>`, `<article>`, `<p>`, `.content`) have `max-width` set in `ch` or equivalent.

**Fix**: Add to the mobile-adapt CSS block (auto-fix because it improves readability significantly):

```css
/* mobile-adapt: auto-generated */
@media (min-width: 840px) {
  article, .content, main > p, main > div {
    max-width: 65ch;
  }
}
```

**Note**: This rule targets desktop/tablet (min-width: 769px) — on mobile, width is naturally constrained.
