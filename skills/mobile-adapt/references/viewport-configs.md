# Viewport Test Configurations

## Breakpoint System — Google Window Size Classes

Based on [Material Design Window Size Classes](https://developer.android.com/develop/ui/compose/layout/window-size-classes), converted from dp to CSS px (1dp = 1px on web):

| Size Class | Width Range | CSS Breakpoint | Common Devices |
|------------|-------------|----------------|----------------|
| Compact | < 600px | `max-width: 599px` | Phone in portrait |
| Medium | 600–839px | `min-width: 600px` and `max-width: 839px` | Tablet portrait, foldable portrait |
| Expanded | >= 840px | `min-width: 840px` | Tablet landscape, desktop |

**Key breakpoints for media queries:**

- `@media (max-width: 599px)` — Compact only (phone-specific, e.g., hamburger nav)
- `@media (max-width: 839px)` — Compact + Medium (main mobile/tablet breakpoint for most fixes)
- `@media (min-width: 840px)` — Expanded (desktop overrides)

## Test Viewports

| Device | Width | Height | Size Class | Purpose |
|--------|-------|--------|------------|---------|
| iPhone SE / Generic Phone | 375 | 667 | Compact | Worst-case phone experience |
| iPad Mini / Generic Tablet | 768 | 1024 | Medium | Tablet portrait experience |
| iPad Landscape / Small Desktop | 1024 | 768 | Expanded | Tablet landscape / desktop |

## Playwright Command Sequence

For each viewport, execute in order:

### 1. Resize Browser

```
browser_resize(width, height)
```

### 2. Navigate to Page

Use HTTP server or `file://` URL:

```
browser_navigate("http://localhost:<port>/<filename>")
```

**Note**: If `file://` protocol is blocked by Playwright MCP, start a local HTTP server:

```bash
cd <html-file-directory> && python3 -m http.server <port>
```

Then navigate to `http://localhost:<port>/<filename>`.

### 3. Wait for Render

```
browser_wait_for(time: 2)
```

Allow 2 seconds for CSS/JS to fully render.

### 4. Take Screenshot

```
browser_take_screenshot(type: "png", filename: "<name>", fullPage: true)
```

### 5. Capture Accessibility Snapshot

```
browser_snapshot()
```

Use the snapshot to verify element visibility, touch target presence, and navigation structure.

## Screenshot Naming Convention

Format: `{base}-{phase}-{width}px.png`

- `{base}`: HTML filename without extension (e.g., `index`, `about`)
- `{phase}`: `before` or `after`
- `{width}`: Viewport width in pixels

Examples:
- `index-before-375px.png`
- `index-after-375px.png`
- `index-before-768px.png`
- `index-after-768px.png`
- `index-before-1024px.png`
- `index-after-1024px.png`

## Interaction Tests (After Fix)

Perform these tests at the **375px** viewport (worst case):

### Scroll Test (Horizontal Overflow Check)

```javascript
// In browser_evaluate:
() => {
  return {
    bodyScrollWidth: document.body.scrollWidth,
    viewportWidth: window.innerWidth,
    hasHorizontalOverflow: document.body.scrollWidth > window.innerWidth
  }
}
```

If `hasHorizontalOverflow` is true, flag as **CRITICAL** regression.

### Hamburger Menu Test (Compact only)

1. Use `browser_snapshot()` to verify hamburger toggle is visible
2. Click the hamburger toggle with `browser_click()`
3. Verify nav menu items appear in the snapshot
4. Click again to close, verify items are hidden

### Navigation Click Test

1. Open the hamburger menu (if compact viewport)
2. Click the first navigation link with `browser_click()`
3. Verify no layout breakage by checking `browser_snapshot()` again

### Form Focus Test

1. Use `browser_snapshot()` to find `<input>`, `<textarea>`, `<select>` elements
2. Click/focus the first input with `browser_click()`
3. Verify the element receives focus (check snapshot for focused state)

### Touch Target Size Check

```javascript
// In browser_evaluate:
() => {
  const clickables = document.querySelectorAll('a, button, input, select, textarea, [role="button"], [onclick]');
  const tooSmall = [];
  clickables.forEach(el => {
    const rect = el.getBoundingClientRect();
    if (rect.width > 0 && rect.height > 0 && (rect.width < 44 || rect.height < 44)) {
      tooSmall.push({
        tag: el.tagName,
        text: el.textContent?.slice(0, 30),
        width: Math.round(rect.width),
        height: Math.round(rect.height)
      });
    }
  });
  return { total: clickables.length, tooSmall };
}
```

## Serving Files

**Preferred**: Start a temporary HTTP server for the test session:

```bash
cd <directory-containing-html> && python3 -m http.server <random-port>
```

Remember to kill the server after all tests complete:

```bash
pkill -f "python3 -m http.server <port>"
```
