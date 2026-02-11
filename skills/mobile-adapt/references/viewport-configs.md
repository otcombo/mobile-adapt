# Viewport Test Configurations

## Test Viewports

| Device | Width | Height | Purpose |
|--------|-------|--------|---------|
| iPhone SE | 375 | 667 | Smallest usable mobile viewport |
| iPhone 14 Pro | 393 | 852 | Mainstream smartphone |
| iPad Mini | 768 | 1024 | Tablet breakpoint |

## Playwright Command Sequence

For each viewport, execute in order:

### 1. Resize Browser

```
browser_resize(width, height)
```

### 2. Navigate to File

Construct `file://` URL from the absolute path of the HTML file:

```
browser_navigate("file:///absolute/path/to/file.html")
```

**Important**: Always use the absolute path. If the user provides a relative path, resolve it against the current working directory.

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
- `index-before-393px.png`
- `index-after-393px.png`
- `index-before-768px.png`
- `index-after-768px.png`

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

### Navigation Click Test

1. Use `browser_snapshot()` to find navigation links/buttons
2. Click the first navigation element with `browser_click()`
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

## file:// URL Construction

Given a file path, construct the URL as follows:

- Absolute path `/home/user/site/index.html` → `file:///home/user/site/index.html`
- Relative path `./site/index.html` → resolve to absolute first, then prefix with `file://`
- Paths with spaces must be percent-encoded: `/my folder/page.html` → `file:///my%20folder/page.html`
