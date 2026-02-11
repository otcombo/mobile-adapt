---
name: mobile-adapt
description: "Adapt static HTML pages for mobile. Use when asked to '适配移动端', 'mobile adapt', '移动端优化', 'responsive fix', 'make mobile friendly', or 'mobile-adapt'."
metadata:
  author: otcombo
  version: "1.0.0"
  argument-hint: "<html-file-or-folder-path>"
---

# Mobile Adapt Skill

Audit, fix, and verify mobile responsiveness for static HTML pages. The workflow proceeds through 6 layers: Input Parsing → Visual Baseline → Code Audit → Fix Application → Verification → Report Output.

**Breakpoint system**: Uses [Google Material Design Window Size Classes](https://developer.android.com/develop/ui/compose/layout/window-size-classes) — Compact (< 600px), Medium (600–839px), Expanded (≥ 840px).

**Complementary to `web-design-guidelines`**: This skill focuses specifically on mobile adaptation, while `web-design-guidelines` covers broader UI/UX concerns (color, typography, layout patterns, accessibility). After running this skill, suggest the user run `/web-design-guidelines` for a complete review.

---

## Execution Modes

After parsing the input (file/folder) and reading the HTML, **ask the user to choose a mode** using `AskUserQuestion`:

```
question: "选择适配模式？"
header: "Mode"
options:
  - label: "快速修复 (Quick)"
    description: "仅修复 CRITICAL + HIGH 问题（viewport、溢出、触控目标、响应式布局、字体）。不截图、不做交互测试。适合快速迭代或已知问题少的页面。约 10 轮对话完成。"
  - label: "全面审计 (Thorough)"
    description: "完整 22 条规则审计 + 修复。含 3 组视口 before/after 截图对比、汉堡菜单注入、交互测试（溢出/导航/表单/触控目标）、详细中文报告。适合首次适配或复杂页面。约 30-40 轮对话完成。"
```

### Mode Comparison

| | Quick | Thorough |
|---|---|---|
| **审计范围** | CRITICAL (C1-C4) + HIGH (H1-H5) | 全部 22 条规则 (C/H/M/L) |
| **修复范围** | CRITICAL + HIGH | CRITICAL + HIGH + MEDIUM + L4 |
| **截图** | 无 | 3 视口 × before/after = 6 张 |
| **交互测试** | 仅水平溢出检测 | 溢出 + 汉堡菜单 + 导航 + 表单 + 触控目标 |
| **导航适配** | CSS 纵向堆叠 | CSS-only 汉堡菜单 (checkbox hack) |
| **报告** | 简洁摘要（变更列表） | 完整报告（截图对比 + 交互结果 + 手动建议） |
| **HTTP 服务** | 不需要 | 需要临时启动 |
| **适用场景** | 快速迭代、简单页面、已部分适配 | 首次适配、复杂页面、需要验证效果 |

---

## Execution Flow

### Layer 0: Input Parsing

**Argument handling:**

The skill argument can be:

1. **A single HTML file path** (e.g., `index.html`, `/path/to/page.html`) — process that file.
2. **A folder path** (e.g., `./site/`, `/path/to/project/`) — find all `.html` files in the folder (non-recursive), process each one sequentially. Share a single HTTP server across all files.
3. **No argument / empty** — ask the user which file or folder to process using `AskUserQuestion`.

For folder mode, output a combined report with per-file sections.

**For each HTML file:**

1. **Read the HTML file**.
2. **Ask the user to choose a mode** (Quick or Thorough) — see "Execution Modes" above. Only ask once; apply the same mode to all files in folder mode.
3. **Discover associated CSS files**: Parse `<link rel="stylesheet" href="...">` and `<style>` blocks.
   - For relative `href`, resolve against the HTML file's directory.
   - For absolute URLs (CDN), note them but do NOT modify (report only).
3. **Detect CSS frameworks**:
   - **Bootstrap**: Look for `bootstrap` in `<link>` href or class names like `container`, `row`, `col-`, `btn`, `navbar`.
   - **Tailwind CSS**: Look for `tailwindcss` in `<link>`/`<script>` or extensive use of utility classes (`flex`, `grid`, `p-`, `m-`, `text-`, `w-`, `bg-`).
   - If a framework is detected:
     - **Skip**: Layout fixes (H3) — framework handles responsive layout.
     - **Skip**: Navigation fixes (H5) — framework has its own nav components.
     - **Keep**: All CRITICAL fixes, touch target (H1), image (H2), font (H4), and all MEDIUM/LOW checks.
     - **Report**: Note the detected framework and adjusted strategy.

### Layer 1: Visual Baseline (Playwright Screenshots)

> **Quick mode**: Skip this entire layer. Proceed directly to Layer 2.

> **Thorough mode**: Execute the full sequence below.

Read the viewport configurations from `references/viewport-configs.md`.

**Start an HTTP server** in the HTML file's directory (needed because Playwright MCP blocks `file://`):

```bash
cd <html-directory> && python3 -m http.server <random-port>
```

For each of the 3 test viewports (375px Compact, 768px Medium, 1024px Expanded):

1. `browser_resize(width, height)`
2. `browser_navigate("http://localhost:<port>/<filename>")`
3. `browser_wait_for(time: 2)` — allow full render
4. `browser_take_screenshot(type: "png", filename: "<base>-before-<width>px.png", fullPage: true)`
5. `browser_snapshot()` — capture accessibility tree for later comparison

Save the before screenshots. These will be compared to after screenshots in Layer 4.

### Layer 2: Code Audit

Read the full checklist from `references/mobile-checklist.md`.

> **Quick mode**: Only audit CRITICAL (C1-C4) and HIGH (H1-H5) rules. Skip H6 and all MEDIUM/LOW.

> **Thorough mode**: Audit all rules (C1-C4, H1-H6, M1-M9, L1-L4).

**For each applicable rule**:

1. Apply the **Detection** method described in the checklist.
2. Search the HTML file and all associated CSS files for the patterns.
3. Classify as:
   - **FAIL**: Rule is violated, needs fixing.
   - **PASS**: Rule is already satisfied.
   - **SKIP**: Rule is not applicable (e.g., no forms → skip M3; framework detected → skip H3/H5).
4. For FAIL items, note the specific file and line where the issue was found (or where the fix should go).

**Generate an internal findings list** (used in Layer 3 and Layer 5). Do not output to the user yet.

### Layer 3: Fix Application

> **Quick mode**: Fix CRITICAL + HIGH only. For H5 navigation, use simple CSS vertical stacking (not hamburger menu).

> **Thorough mode**: Fix CRITICAL + HIGH + MEDIUM + L4. For H5 navigation, inject full CSS-only hamburger menu.

Process FAIL items in priority order: **CRITICAL → HIGH → MEDIUM** (skip LOW except L4).

**Fix Principles:**

1. **Additive only** — Never delete existing CSS rules. Only add new rules or modify attributes.
2. **Marked changes** — All injected CSS must be wrapped in a `/* mobile-adapt: auto-generated */` comment block.
3. **Precise edits** — Use the `Edit` tool for all modifications. No full-file rewrites.
4. **Respect external resources** — Never modify CDN-hosted CSS. Only report issues.
5. **CSS injection location**:
   - If the page has a `<style>` block: append to its end (before `</style>`).
   - If the page uses an external local CSS file: append to the end of that file.
   - If neither: create a new `<style>` block before `</head>`.
6. **HTML injection location**:
   - Viewport meta → after `<meta charset="...">` in `<head>`.
   - If no `<head>` exists: create `<head>...</head>` before `<body>`.
   - `loading="lazy"` → directly on `<img>` tags.

**Framework-aware fixes:**

When Bootstrap or Tailwind is detected:
- Use framework-native classes when possible (e.g., Tailwind `min-h-[44px]` instead of raw CSS).
- If adding raw CSS is cleaner (for CRITICAL items), add it in a separate `<style>` block marked as `/* mobile-adapt: auto-generated — framework override */`.

**Grouping injected CSS:**

Combine all mobile-adapt CSS rules into a single block per file:

```css
/* ========================================
   mobile-adapt: auto-generated
   Generated rules for mobile responsiveness.
   DO NOT edit manually — re-run mobile-adapt to update.
   ======================================== */

/* C2: Touch action */
a, button, input, select, textarea, [role="button"] {
  touch-action: manipulation;
}

/* C3: Text size adjust */
html {
  -webkit-text-size-adjust: 100%;
  text-size-adjust: 100%;
}

/* ... more rules grouped by ID ... */

/* ======================================== */
```

### Layer 4: Verification (Playwright Screenshots + Interaction Tests)

**After all fixes are applied**, run verification.

> **Quick mode**: Start an HTTP server if not already running. Only run the horizontal overflow test at 375px using `browser_evaluate()`. No screenshots.

> **Thorough mode**: Full verification below.

#### 4a. After Screenshots (Thorough only)

For each of the 3 viewports (375px, 768px, 1024px):
1. `browser_resize(width, height)`
2. `browser_navigate("http://localhost:<port>/<filename>")` — reload to pick up changes
3. `browser_wait_for(time: 2)`
4. `browser_take_screenshot(type: "png", filename: "<base>-after-<width>px.png", fullPage: true)`
5. `browser_snapshot()` — capture updated accessibility tree

#### 4b. Interaction Tests (at 375px viewport)

Run the interaction tests defined in `references/viewport-configs.md`:

1. **Horizontal overflow test** (Quick + Thorough): Use `browser_evaluate()` to check `document.body.scrollWidth > window.innerWidth`. If true after fixes, flag as **regression**.
2. **Hamburger menu test** (Thorough only, at 375px): Verify hamburger icon is visible, click to open, verify nav items appear, click to close.
3. **Navigation click test** (Thorough only): Open hamburger (if compact), click a nav link, verify no layout break.
4. **Form focus test** (Thorough only): If forms exist, click an input, verify focus state.
5. **Touch target audit** (Thorough only): Use `browser_evaluate()` to measure all interactive elements. Report any still below 44x44px.

### Layer 5: Output Report (Chinese)

Generate the final report in **Chinese (中文)**.

> **Quick mode**: Use the simplified report format (see below).

> **Thorough mode**: Use the full report template (see below).

#### Quick Mode Report

```markdown
# 移动端快速适配报告

**文件**: `{html_file_path}`
**模式**: 快速修复
**检测到的框架**: {framework_or_none}

## 修复内容

### CRITICAL
- [FIXED/PASS] C1: Viewport Meta — {说明}
- [FIXED/PASS] C2: Touch Action — {说明}
- [FIXED/PASS] C3: Text Size Adjust — {说明}
- [FIXED/PASS] C4: 水平溢出防护 — {说明}

### HIGH
- [FIXED/PASS/SKIP] H1: 触控目标 ≥ 44px — {说明}
- [FIXED/PASS] H2: 响应式图片 — {说明}
- [FIXED/PASS/SKIP] H3: 响应式布局 — {说明}
- [FIXED/PASS] H4: 字体 ≥ 14px — {说明}
- [FIXED/PASS/SKIP] H5: 导航适配 — {说明}

## 溢出检测
{PASS/FAIL}: scrollWidth={n} vs viewportWidth={n}

> 如需截图对比、交互测试和完整 MEDIUM/LOW 审计，请使用「全面审计」模式重新运行。
```

#### Thorough Mode Report

Use this full template:

---

```markdown
# 移动端适配报告

**文件**: `{html_file_path}`
**检测到的框架**: {framework_or_none}
**检查时间**: {timestamp}

---

## 变更摘要

### CRITICAL（已修复 {n}/{total}）

| 规则 | 状态 | 说明 | 位置 |
|------|------|------|------|
| C1: Viewport Meta | PASS/FIXED/SKIP | ... | file:line |

### HIGH（已修复 {n}/{total}）

| 规则 | 状态 | 说明 | 位置 |
|------|------|------|------|
| H1: 触控目标 | PASS/FIXED/SKIP | ... | file:line |

### MEDIUM（已修复 {n}/{total}）

| 规则 | 状态 | 说明 | 位置 |
|------|------|------|------|
| M1: 流式排版 | PASS/FIXED/SKIP/FLAG | ... | file:line |

### LOW（仅标记）

| 规则 | 状态 | 说明 |
|------|------|------|
| L1: 减弱动画 | FLAG/PASS | ... |

---

## 截图对比

### Compact — Phone (375px)
- Before: `{base}-before-375px.png`
- After: `{base}-after-375px.png`

### Medium — Tablet Portrait (768px)
- Before: `{base}-before-768px.png`
- After: `{base}-after-768px.png`

### Expanded — Tablet Landscape / Desktop (1024px)
- Before: `{base}-before-1024px.png`
- After: `{base}-after-1024px.png`

---

## 交互测试结果

| 测试项 | 结果 | 备注 |
|--------|------|------|
| 水平溢出检测 | PASS/FAIL | scrollWidth vs viewportWidth |
| 汉堡菜单 | PASS/FAIL/SKIP | 点击展开/收起是否正常 |
| 导航点击 | PASS/FAIL/SKIP | ... |
| 表单聚焦 | PASS/FAIL/SKIP | ... |
| 触控目标尺寸 | {n} 个元素不达标 | 列出前 5 个 |

---

## 手动建议

以下项目需要人工审查，未自动修复：

- M8: 颜色对比度 — 请使用 WebAIM Contrast Checker 检查
- L1: 减弱动画 — 建议为动画添加 `prefers-reduced-motion` 媒体查询
- L2: 打印样式 — 如需打印功能，请添加 `@media print` 规则
- L3: 深色模式 — 如需支持深色模式，请添加 `prefers-color-scheme: dark`
- ... (only list items that are FLAG status)

---

> 如需完整 Web 界面审查（色彩、排版、可访问性等），请运行 `/web-design-guidelines`
```

---

## Framework Detection Details

### Bootstrap Detection

Indicators (any 2+ = Bootstrap):
- `<link>` href contains `bootstrap`
- `<script>` src contains `bootstrap`
- HTML contains classes: `container`, `row`, `col-xs-`, `col-sm-`, `col-md-`, `col-lg-`, `col-xl-`
- HTML contains classes: `btn`, `btn-primary`, `navbar`, `nav-link`

**Strategy adjustment**: Skip H3 (layout), H5 (nav). Bootstrap's grid and navbar are responsive by default.

### Tailwind CSS Detection

Indicators (any 2+ = Tailwind):
- `<link>` or `<script>` src contains `tailwindcss` or `tailwind`
- HTML has 10+ elements with Tailwind utility classes: `flex`, `grid`, `p-[0-9]`, `m-[0-9]`, `text-[a-z]`, `w-`, `h-`, `bg-`, `rounded`, `shadow`
- `tailwind.config` referenced in the page

**Strategy adjustment**: Skip H3 (layout). For H1 (touch targets), suggest Tailwind classes like `min-h-[44px] min-w-[44px]` in the report but still add CSS fallback.

---

## Edge Cases

### No `<head>` Tag

If the HTML file has no `<head>`:
1. Create `<head>` with viewport meta.
2. Inject `<style>` block inside new `<head>`.
3. Place `<head>...</head>` before `<body>` (or at the start of the file if no `<body>` either).

### Already Responsive

If the audit finds 0 CRITICAL and 0 HIGH failures:
- Still take before/after screenshots (they should be identical).
- Report all PASS results.
- Focus the report on MEDIUM and LOW suggestions.

### Large Files (> 5000 lines)

For HTML/CSS files exceeding 5000 lines:
- Do NOT read the entire file upfront.
- Use `Grep` to search for specific patterns (viewport meta, fixed widths, font-size, etc.).
- Use `Read` with `offset` and `limit` to read specific sections.
- Apply fixes using `Edit` with precise `old_string` matching.

### Inline Styles

For elements with inline `style` attributes containing fixed widths:
- Report them in the audit but do NOT modify inline styles (too fragile).
- Add CSS overrides using attribute selectors if critical:

```css
/* mobile-adapt: auto-generated — inline style override */
@media (max-width: 839px) {
  [style*="width:"] {
    max-width: 100% !important;
  }
}
```

### Multiple CSS Files

When the page links multiple local CSS files:
- Audit ALL of them.
- Inject fixes into the LAST local CSS file (to ensure highest specificity via cascade order).
- If no local CSS file exists, inject into a `<style>` block in HTML.

---

## Integration Notes

### With `web-design-guidelines`

- **This skill** handles: viewport, responsive layout, touch targets, mobile CSS, mobile-specific interactions.
- **web-design-guidelines** handles: color theory, typography hierarchy, spacing systems, accessibility (ARIA, screen readers), animation principles, dark mode design.
- **Overlap**: Both check font sizes and some accessibility concerns. If both are run, the second will see fixes from the first.
- **Recommended order**: Run `mobile-adapt` first (structural fixes), then `web-design-guidelines` (design refinement).

### Cleanup

After all layers complete, kill the temporary HTTP server:

```bash
pkill -f "python3 -m http.server <port>"
```
