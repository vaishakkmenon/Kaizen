# Kaizen Layout Architecture

Audit of every CSS source, the DOM structure, sidebar state machine, resize logic, and
known conflicts. No fixes applied — this is description only.

---

## 1. CSS Sources

### Injection order for the Deck Browser (defined in `__init__.py:inject_menu_files`)

All injected into `web_content.head` in this order:

| # | Source | What it controls |
|---|--------|-----------------|
| 1 | `patcher.generate_dynamic_css(conf)` | CSS variables (`:root` / `.night-mode`), font faces, glassmorphism on stat cards |
| 2 | `web/menu.css` (inline `<style>`) | Full deck-browser layout: body, sidebar, resize handle, main-content, deck table, collapsed/focus states |
| 3 | `web/heatmap.css` (inline `<style>`) | Heatmap grid (year/month/week views), skeleton loader, tooltips |
| 4 | `patcher.generate_profile_bar_fix_css()` | Repositions `.profile-pic` to `position: absolute` inside profile bar |
| 5 | `patcher.generate_deck_browser_backgrounds(addon_path)` | Background on `.container.modern-main-menu` and `.sidebar-left`; always appends `.main-content { background: transparent !important; }` |
| 6 | `patcher.generate_icon_css(addon_package, conf)` | `mask-image` for all icons (data URIs), custom deck icons per `data-did` |
| 7 | `patcher.generate_conditional_css(conf)` | Conditional visibility: hide stats-grid, deck-counts, etc. |
| 8 | `patcher.generate_icon_size_css()` | `width`/`height` on icon selectors from config |
| 9 | `notifications.css` (`<link>`) | Notification stack positioning |
| 10 | `injector.js`, `engine.js`, `heatmap.js`, `notifications.js` (`<script>`) | UI behaviour (see sections 3 & 4) |

Additionally, `templates.py:custom_body_template` contains a `<style>` block **in the HTML body**
(not the head). It is injected as `body=` parameter to `stdHtml()` and therefore parses
**after** all head styles.

`core/renderer.py` injects a second `<style>` block **also in the body** via the `{stats}`
placeholder. It sets critical layout overrides with `!important`.

---

### `web/menu.css` — what each section controls

| Selector | Key rules | Notes |
|----------|-----------|-------|
| `html` | `height: 100%` | Establishes height reference for `body` |
| `body` | `display: flex; overflow: auto; height: 100%` | Flex container for the whole page |
| `.container.modern-main-menu` | `display: flex; height: 100%; position: relative` | Three-column flex row: sidebar + handle + main |
| `.sidebar-left` | `flex-shrink: 0; flex-grow: 0; min-width: 325px; position: relative; overflow: hidden` | Fixed-width left column; two `transition` declarations (conflict — see §5) |
| `.resize-handle` | `width: 15px; flex-shrink: 0; position: relative` | Drag target between sidebar and main |
| `.main-content` | `flex-grow: 1; min-width: 0; overflow-y: auto; overflow-x: hidden` | Right column, scrollable |
| `.sidebar-toolbar` | `position: absolute; top: 15px; left: 20px` | Icon row of toggle/focus/edit buttons |
| `.sidebar-toggle-btn`, `.deck-focus-btn`, `.deck-edit-btn` | `position: absolute` (overridden to `relative` when inside `.sidebar-toolbar`) | |
| `.sidebar-left.sidebar-collapsed` | `width: 50px !important; min-width: 50px; max-width: 50px` | Fully overrides all width |
| `.sidebar-left.deck-focus-mode .sidebar-expanded-content > *` | `display: none !important` on profile-bar, menu-items, h2 | Focus mode hides nav, keeps deck list |
| `.sidebar-left.sidebar-only-mode` | `width: auto !important; max-width: 800px !important` | When col_count == 0 |
| `#deck-list-container` (line 302) | `overflow: hidden; flex-grow: 1; min-height: 0` | First rule block |
| `#deck-list-container` (line 1121) | `contain: layout style paint; overflow: hidden` | Second, later rule block — wins within menu.css |

---

### `web/overview.css` — controls Overview screen

- `body`: `display: flex; align-items: center; justify-content: center; height: 100%`  
- `.overview-header`: `position: absolute; z-index: 1000` — floats above content  
- `.overview-button`: button styling with `!important` on background and opacity

---

### `web/congrats.css` — controls Congrats (all done) screen

- `body`: `display: flex; align-items: center; justify-content: center; height: 100%; overflow: hidden`  
- Duplicates `.overview-header` and `.overview-button` from overview.css (copy-paste, not shared)

---

### `web/heatmap.css` — controls heatmap widget only

- No layout-affecting rules outside the heatmap container  
- `#kaizen-heatmap-container`: `display: flex; flex-direction: column; overflow: hidden`  
- Year/month/week views use CSS grid internally

---

### `patches/css.py` — dynamic CSS generators

**`generate_dynamic_css(conf)`**
- Outputs `<style id="modern-menu-dynamic-styles">`
- Sets all CSS variables: `:root { --bg, --border, --canvas-inset, ... }` and `.night-mode { ... }`
- Text/accent variables scoped to `.kaizen-ui, [class*="kaizen-"], .modern-menu, .modern-menu *:not(.card)`
- Optional glassmorphism: `backdrop-filter: blur(Xpx)` on `.stat-card, #kaizen-heatmap-container`
- Calls `generate_font_css()` and prepends its output

**`generate_font_css(addon_package)`**
- Outputs `<style id="kaizen-font-styles">`
- `@font-face` rules for selected fonts
- CSS variables: `--font-main`, `--font-subtle`, `--font-small-title`, `--font-size-*`
- Applies `font-family: var(--font-main)` to `body:not(.card)` with `!important`

**`generate_icon_css(addon_package, conf)`**
- Outputs `<style id="modern-menu-icon-styles">`
- `mask-image` (data URI) for: options gear, folder, deck, subdeck, filtered_deck, add, browse, stats, sync, settings, more, get_shared, create_deck, import_file, retention_star, focus, edit
- Per-`data-did` rules for custom deck icons (emoji, PNG, or SVG)
- Hide rules (`display: none !important`) for icon types toggled off

**`generate_icon_size_css()`**
- Outputs `<style id="modern-menu-icon-size-styles">`
- `width/height` on: `a.deck::before`, `.menu-item .icon`, `a.collapse`, `td.opts a`

**`generate_conditional_css(conf)`**
- Outputs `<style id="modern-menu-conditional-styles">`
- Optionally hides: `.stats-grid`, `.deck-counts .zero`, `.deck-counts`

**`generate_reviewer_buttons_css(conf)`**
- Controls reviewer bottom bar (`#outer`) height, button radius, padding, colour  
- Used in: reviewer context and `ReviewerBottomBar` context

**`generate_profile_bar_fix_css()`**
- Outputs `<style id="kaizen-profile-bar-fix">`
- Overrides `.profile-bar` to use `padding: 6px 8px 6px 50px; min-height: 50px`
- Overrides `.profile-pic, .profile-pic-placeholder` to `position: absolute; top: 5px; left: 5px`
- **Conflicts** with menu.css `.profile-bar` which has `margin-bottom: 20px; border-radius: 50px` — this fix wins (injected after menu.css)

---

### `patches/backgrounds.py` — background CSS generators

All functions call `_render_background_css()` internally, which generates a `<style>` block.

**`generate_deck_browser_backgrounds(addon_path)`**
- Target: `.container.modern-main-menu` (main background)
- Modes: `color`, `image`, `image_color`, `accent`, `slideshow`
- Image/image_color modes: `::before` (or `::after` for slideshow second layer) on `.container.modern-main-menu`
  - `position: absolute; top: 50%; left: 50%; width: 100%; height: 100%; transform: translate(-50%, -50%) scale(N)`
  - `filter: blur(Xpx); opacity: N; z-index: -1`
  - Sets `.container.modern-main-menu { position: relative; overflow: hidden; }`
  - Sets `html { overflow: hidden !important; }` when body is the target (not applicable here — container is target)
- Sidebar target: `.sidebar-left`
  - Modes: `main` (passthrough/glassmorphism/opaque tint), `custom` (color/accent/image_color)
  - Sidebar image mode: `::before` with `position: absolute`, same pattern as above
- Always appends `<style>.main-content { background: transparent !important; }</style>`

**`generate_reviewer_background_css(addon_path)`**
- Target: `body::before` — `position: fixed; top: 50%; left: 50%; width: 100%; height: 100%`
- Sets `html { overflow: hidden !important; }` in image mode

**`generate_overview_background_css(addon_path)`**
- Same as reviewer: `body::before` with `position: fixed`

**`generate_reviewer_bottom_bar_background_css(addon_path)`** and **`generate_toolbar_background_css(addon_path)`**
- Target their respective Qt-injected webview elements

---

### `core/renderer.py` — inline `<style>` in body HTML

Inside `stats_block_html` (injected as `{stats}` in `custom_body_template`):

```css
/* Always present */
.unified-grid {
    display: grid;
    grid-template-columns: repeat(N, 1fr);   /* N = col_count from config */
    grid-auto-rows: minmax(110px, auto);
    gap: 15px;
}

.sidebar-left { max-width: none !important; }   /* Unconditionally removes max-width */

.main-content {
    padding: Xpx !important;   /* 40, 20, or 60 depending on col_count */
    display: flex !important;  /* or 'none' when sidebar-only mode */
    flex-direction: column;
    align-items: center;
}

.main-content > * {
    max-width: 1600px !important;   /* or 900px when col_count <= 4 */
}

.modern-main-menu.container {
    justify-content: center !important;   /* or flex-start, depending on sidebar-only mode */
}
```

These use `!important` and are in the body, so they override all head styles.

---

### `templates.py` — inline `<style>` in body HTML (always present)

```css
/* Always present, in body, so overrides head styles */
.sidebar-expanded-content {
    display: flex; flex-direction: column; height: 100%; overflow: hidden;
}
#deck-list-container {
    flex: 1; overflow-y: auto; min-height: 0;
}
```

Also defines: `.deck-edit-mode` rules, `.context-menu-item`, `.favorite-star-icon`,
`.sidebar-toggle-btn.active`, `.sidebar-left .menu-item` padding/sizing overrides.

---

### `__init__.py` — context-specific injection

| Context | CSS injected |
|---------|-------------|
| `DeckBrowser` | All of the above (items 1–10) |
| `Overview` | `generate_dynamic_css`, `generate_overview_background_css`, `overview.css`, top-bar CSS, `notifications.css` |
| `Reviewer` | `notifications.css`, `generate_notification_position_css`, `generate_reviewer_background_css`, `generate_reviewer_buttons_css`, top-bar CSS/HTML |
| `ReviewerBottomBar` | `generate_reviewer_bottom_bar_background_css`, `generate_reviewer_buttons_css` |
| `Toolbar` / `BottomBar` | `generate_toolbar_background_css` (if `hideNativeHeaderAndBottomBar` is False) |

---

## 2. Layout Structure

### DOM tree for the Deck Browser page

```
html  (height: 100%)                                             [menu.css]
└── body  (display:flex; overflow:auto; height:100%)             [menu.css]
    └── [Anki stdHtml wrapper]
        │
        ├── <style>  [templates.py — in body, high cascade priority]
        │
        ├── .container.modern-main-menu   (display:flex; height:100%; position:relative)
        │   │
        │   ├── .sidebar-left   (flex-shrink:0; flex-grow:0; min-width:325px;
        │   │   │                position:relative; overflow:hidden)
        │   │   │
        │   │   ├── .sidebar-toolbar   [position:absolute; top:15px; left:20px]
        │   │   │   ├── .sidebar-toggle-btn   [position:relative inside toolbar]
        │   │   │   ├── .deck-focus-btn
        │   │   │   ├── .deck-edit-btn
        │   │   │   ├── .deck-transfer-btn   (hidden until edit-mode selection)
        │   │   │   └── .action-btn×N   (add/browse/stats/sync/settings/more)
        │   │   │       [only rendered if sidebarActionsMode='collapsed']
        │   │   │
        │   │   ├── .sidebar-expanded-content   (flex-col; height:100%; overflow:hidden)
        │   │   │   ├── h2  (welcome message)
        │   │   │   ├── .profile-bar   (with inline background-image or bg-color style)
        │   │   │   ├── [menu-items: add, browse, stats, sync, settings, more]
        │   │   │   │   [only rendered if sidebarActionsMode='list']
        │   │   │   ├── #deck-list-header   (flex row: "DECKS" h2 + edit buttons)
        │   │   │   └── #deck-list-container   (flex:1; overflow-y:auto; min-height:0)
        │   │   │       └── table.deck-table#decktree
        │   │   │           └── tbody
        │   │   │               └── tr.deck.is-[deck|folder|subdeck|filtered] × N
        │   │   │                   ├── td.collapse-cell  (a.collapse or span.collapse)
        │   │   │                   ├── td.decktd  (display:flex)
        │   │   │                   │   ├── .deck-prefix  (indent spacer)
        │   │   │                   │   ├── .deck-info  (flex; a.deck::before icon + a.deck name)
        │   │   │                   │   └── .deck-counts  (new/learn/review bubbles)
        │   │   │                   └── td.opts  (display:none; shown in edit mode)
        │   │   │
        │   │   └── .collapsed-content-wrapper
        │   │       └── .sidebar-collapsed-content  (display:none → flex when collapsed)
        │   │           └── .collapsed-icon-btn × 5  (add, browse, stats, sync, settings)
        │   │
        │   ├── .resize-handle   (width:15px; flex-shrink:0; position:relative; cursor:col-resize)
        │   │   └── .resize-handle-indicator   [position:absolute; animated on hover]
        │   │
        │   └── .main-content   (flex-grow:1; min-width:0; overflow-y:auto; overflow-x:hidden)
        │       └── .injected-stats-block
        │           ├── h1.kaizen-widget-title
        │           └── .unified-grid  (CSS grid, N columns, set by renderer.py inline style)
        │               └── .kaizen-widget-container × N  [grid-area from inline style attr]
        │                   └── one of:
        │                       ├── .stat-card  (studied / time / pace / retention)
        │                       ├── #kaizen-heatmap-container
        │                       ├── .kaizen-favorites-widget
        │                       └── .external-widget-container  (from external add-on hooks)
        │
        └── #deck-context-menu  [position:absolute; z-index:10000; display:none by default]
```

### Positioning context summary

| Element | Position | Containing block |
|---------|----------|-----------------|
| `.sidebar-toolbar` | `absolute` | `.sidebar-left` (position:relative) |
| `.sidebar-toggle-btn` (base) | `absolute` | `.sidebar-left` |
| `.sidebar-toggle-btn` (in toolbar) | `relative` (overridden) | `.sidebar-toolbar` flex |
| `.resize-handle-indicator` | `absolute` | `.resize-handle` (position:relative) |
| `.container.modern-main-menu::before` | `absolute` | `.container` (position:relative) |
| `.sidebar-left::before` (bg image) | `absolute` | `.sidebar-left` (position:relative) |
| `body::before` (reviewer/overview bg) | `fixed` | viewport |
| `#deck-context-menu` | `absolute` | nearest positioned ancestor |
| `heatmap-day-cell::after` (tooltip) | `absolute` | `.heatmap-day-cell` |

---

## 3. Sidebar State Machine

### State table

| State | CSS classes on `.sidebar-left` | `mw.col.conf` keys |
|-------|-------------------------------|-------------------|
| Expanded (default) | *(none)* | `kaizen_sidebar_collapsed=False` |
| Collapsed | `sidebar-collapsed` | `kaizen_sidebar_collapsed=True` |
| Deck Focus Mode | `deck-focus-mode` | `kaizen_deck_focus_mode=True` |
| Collapsed + Focus | `sidebar-collapsed deck-focus-mode` | both above |
| Sidebar-Only | `sidebar-only-mode` | `col_count==0` or `unifiedGridRows==0` |
| Skeleton (transient) | `skeleton-loading` | — |

### State: Expanded

- Width: inline style `width: Npx` (from `mw.col.conf["modern_menu_sidebar_width"]`, default 300)
- `.sidebar-expanded-content`: visible (`display: flex`)
- `.sidebar-collapsed-content`: hidden (`display: none`)
- JS toggle: click `.sidebar-toggle-btn` → adds `.sidebar-collapsed`, saves width to `dataset.preCollapseWidth`, removes inline width

### State: Collapsed

CSS applied by `.sidebar-collapsed`:
```css
width: 50px !important;
min-width: 50px;
max-width: 50px;
padding: 10px 5px;
overflow: hidden;
```
- `.sidebar-expanded-content { display: none }` — full nav hidden
- `.sidebar-collapsed-content { display: flex }` — icon rail visible
- `.sidebar-toolbar { left: 50% !important; transform: translateX(-50%) !important }` — toolbar centred
- `.deck-focus-btn, .deck-edit-btn { opacity: 0; pointer-events: none }` — hidden
- JS toggle: click `.sidebar-toggle-btn` → removes `.sidebar-collapsed`, restores `dataset.preCollapseWidth` as inline width
- pycmd: `saveSidebarState:true/false`

### State: Deck Focus Mode

CSS applied by `.deck-focus-mode`:
```css
.sidebar-left.deck-focus-mode .sidebar-expanded-content .profile-bar,
.sidebar-left.deck-focus-mode .sidebar-expanded-content .add-button-dashed,
.sidebar-left.deck-focus-mode .sidebar-expanded-content .menu-item,
.sidebar-left.deck-focus-mode .sidebar-expanded-content .menu-group,
.sidebar-left.deck-focus-mode .sidebar-expanded-content h2:first-of-type {
    display: none !important;
}
```
- DOM mutation: `#deck-list-header` is physically **moved** by JS (`updateDeckFocusLayout()`) to be a direct child of `.sidebar-left` (not inside `.sidebar-expanded-content`), making it visible above the hidden content
- JS toggle: click `.deck-focus-btn` → `sidebar.classList.toggle('deck-focus-mode')`
- pycmd: `saveDeckFocusState:true/false`

### State: Sidebar-Only Mode

- Set by Python at render time when `col_count == 0` or `unifiedGridRows == 0`
- Class added to `.sidebar-left` in `renderer.py`
- CSS in menu.css: `width: auto !important; max-width: 800px !important`
- CSS in renderer.py inline: `.main-content { display: none !important }`, `.modern-main-menu.container { justify-content: center !important }`
- `.sidebar-only-mode .sidebar-toggle-btn { display: none }` — hide toggle in this mode

### State: Skeleton Loading (transient)

- Python renders `sidebar_initial_class` containing `skeleton-loading`
- CSS: profile-bar, menu-items made transparent; pulse animation plays
- injector.js `init()`: after 150ms → `sidebar.classList.remove('skeleton-loading')`

### Python config keys for persistence

| Config key | Controls |
|------------|---------|
| `kaizen_sidebar_collapsed` | Whether sidebar starts collapsed |
| `kaizen_deck_focus_mode` | Whether deck focus mode is active |
| `modern_menu_sidebar_width` | Pixel width of expanded sidebar |
| `sidebarActionsMode` | `"list"` (buttons in sidebar) vs `"collapsed"` (buttons in toolbar) |

---

## 4. Resize Behavior

### `setupResizeHandle()` in `web/injector.js`

**Setup** (runs once on `init()`):
- Finds `.resize-handle` and `.sidebar-left`
- Guards against double-setup via `handle.dataset.kaizenSetup`
- Creates `.resize-handle-indicator` child element
- Attaches 5 event listeners; stores them on `handle._resizeHandlers` for cleanup

**Mouse events:**
| Event | Handler |
|-------|---------|
| `mousemove` on handle | Moves indicator position vertically to follow cursor |
| `mouseleave` on handle | Resets indicator to 50% (center) |
| `mousedown` on handle | Starts resize session |
| `mousemove` on document | Updates sidebar width (rAF-throttled) |
| `mouseup` on document | Ends resize session, saves width |

**On `mousedown`:**
1. Records `startX = e.clientX`, `startWidth = sidebar.getBoundingClientRect().width`
2. Detects if layout is centered (`sidebar-only-mode` class or `justifyContent: center` on parent) → sets `isCentered = true`
3. Locks current width: `sidebarEl.style.setProperty('width', Xpx, 'important')`
4. Clears max-width: `sidebarEl.style.setProperty('max-width', 'none', 'important')`
5. Adds `.is-resizing` to sidebar and handle (disables CSS transitions)
6. Sets `body { user-select: none; cursor: col-resize }`

**Width calculation (rAF callback):**
```javascript
deltaX = lastClientX - startX
effectiveDelta = isCentered ? deltaX * 2 : deltaX   // double for centered layout
newWidth = Math.max(325, startWidth + effectiveDelta)  // min 325px enforced
sidebarEl.style.setProperty('width', newWidth + 'px', 'important')
```
- No maximum — `800px` limit is commented out

**On `mouseup`:**
- Removes `.is-resizing`, restores `body` styles
- Calls `pycmd('saveSidebarWidth:N')` → Python saves to `mw.col.conf["modern_menu_sidebar_width"]`

**Collapse/expand width lifecycle** (in `init()` toggle handler):
- **Collapsing**: `dataset.preCollapseWidth = getBoundingClientRect().width`, then `style.removeProperty('width')` and `style.removeProperty('max-width')` — lets `.sidebar-collapsed` CSS take over
- **Uncollapsing**: `style.removeProperty('width')`, then `style.setProperty('width', savedWidth + 'px')` (no `!important`) — CSS `.sidebar-collapsed` is already removed so base rules apply

**Width at each state:**
| State | Width source | Value |
|-------|-------------|-------|
| Expanded, no resize | inline style (from config) | `modern_menu_sidebar_width`px (default 300) |
| Expanded, after resize | inline style `!important` | last dragged value |
| Collapsed | CSS `.sidebar-collapsed !important` | 50px (overrides inline) |
| Sidebar-only | CSS `.sidebar-only-mode !important` | `auto`, max 800px |
| Uncollapsed after resize | inline style (no `!important`) | `dataset.preCollapseWidth`px |

**`refreshResizeHandle()`**
- Called after edit-mode toggle (deck-edit-btn click, ESC key via MutationObserver)
- Removes all 5 stored handlers, deletes `kaizenSetup` flag, re-runs `setupResizeHandle()`

---

## 5. Known Problems

### P1 — Duplicate `transition` on `.sidebar-left` (menu.css lines 34 and 41)

```css
.sidebar-left {
    transition: none !important;   /* line 34 — intended to disable all transitions */
    /* ... other properties ... */
    transition: width 0.3s ease, padding 0.3s ease !important;  /* line 41 — overrides line 34 */
}
```
Line 41 always wins. `transition: none` is never applied. During normal state the smooth collapse transition works, but `.is-resizing` rule (`transition: none !important`) also fights this.

---

### P2 — `#deck-list-container` has three conflicting `overflow` rules

| Source | Rule | Position in cascade |
|--------|------|---------------------|
| menu.css line 302 | `overflow: hidden` | Head, earlier |
| templates.py `<style>` | `overflow-y: auto` | **Body** — wins over head |
| menu.css line 1121 | `overflow: hidden` (via `contain: layout style paint`) | Head, later than line 302 |

The template body `<style>` (which sets `overflow-y: auto`) comes **after** head styles in document order and therefore wins. However, this is fragile: the menu.css line 1121 rule sets `overflow: hidden` as a later rule in the same stylesheet, which should be overridden by the body style. Whether Qt WebEngine respects this cascade ordering needs verification.

---

### P3 — `body { overflow: hidden }` set by `backgrounds.py` in image mode

When the deck browser background mode is `image` or `image_color` for non-body selectors:
```python
# backgrounds.py line 74
container_css += "html { background: transparent !important; overflow: hidden !important; }"
```
This injects `html { overflow: hidden !important; }` and (for body-targeting):
```python
body { overflow: hidden !important; }
```
This directly fights menu.css `body { overflow: auto }`. Since backgrounds.py is injected **after** menu.css in the head (item 5 vs item 2), `overflow: hidden !important` wins — reverting the fix and re-enabling the black screen on resize.

**This is likely the dominant cause of the current black screen issue.**

---

### P4 — `renderer.py` inline style overrides `.main-content` unconditionally

In `stats_block_html` (body `<style>`):
```css
.main-content {
    padding: Xpx !important;   /* overrides menu.css padding: 40px */
    display: flex !important;  /* or 'none' — unconditionally set */
}
```
Since this is in the body, it always wins. Padding is set to 40/20/60px depending on `col_count`. Any future fix to `.main-content` padding in menu.css will have no effect.

---

### P5 — `generate_profile_bar_fix_css()` conflicts with menu.css `.profile-bar`

menu.css sets `.profile-bar { margin-bottom: 20px; border-radius: 50px; padding: 6px }`.  
The fix CSS (item 4, injected after menu.css) overrides: `padding: 6px 8px 6px 50px` and converts `.profile-pic` from a flex child to `position: absolute`. This breaks the margin-bottom on `.profile-bar` in some states and changes the height calculation for the bar.

---

### P6 — Two `!important` width systems fight over `.sidebar-left`

Three separate systems write `width` to `.sidebar-left`:
1. Python inline style at render time: `style="width: 300px;"` (no `!important`)
2. JS resize handler: `style.setProperty('width', Xpx, 'important')`
3. CSS `.sidebar-collapsed { width: 50px !important }`
4. CSS `.sidebar-only-mode { width: auto !important; max-width: 800px !important }`
5. renderer.py: `<style>.sidebar-left { max-width: none !important; }</style>` (always injected)

Rule 5 (`max-width: none !important` from renderer.py body style) overrides rule 4 (`max-width: 800px !important` from menu.css head style) because body styles win in document order. This means **sidebar-only mode's max-width cap is always defeated**.

---

### P7 — `.collapsed-content-wrapper` buttons duplicate `.sidebar-toolbar` action buttons

templates.py HTML includes `.collapsed-content-wrapper > .sidebar-collapsed-content` with 5 `.collapsed-icon-btn` divs (add, browse, stats, sync, settings) hard-coded.

injector.js `setupActionButtons()` creates `.action-btn` divs in `.sidebar-toolbar` for the same actions (when `sidebarActionsMode = 'collapsed'`).

In collapsed mode, both exist in the DOM. Only the `.sidebar-collapsed-content` ones are visible (the toolbar ones are hidden by the `.sidebar-collapsed .deck-focus-btn / .deck-edit-btn { opacity: 0 }` rule, but `.action-btn` doesn't get that rule). This means in collapsed mode with `sidebarActionsMode = 'collapsed'`, both button sets may be partially visible.

---

### P8 — Slideshow mode uses `::before` AND `::after` on `.container.modern-main-menu`

backgrounds.py slideshow CSS:
```css
.container.modern-main-menu::before { /* base layer, always visible */ }
.container.modern-main-menu::after  { /* transition layer, fades in/out */ }
```
This **consumes both pseudo-elements** on the container. Any other feature that tries to use `::before` or `::after` on `.container.modern-main-menu` will conflict. Currently nothing else does, but it's a latent trap.

---

### P9 — `#deck-list-header` DOM mutation creates fragile state

In deck-focus mode, `updateDeckFocusLayout()` physically moves `#deck-list-header` out of `.sidebar-expanded-content` and makes it a direct child of `.sidebar-left`. On focus-off, it moves it back. If a page refresh occurs mid-state (common during deck operations), the header may end up in the wrong parent, breaking layout or triggering Anki's MutationObserver to enter an inconsistent edit-mode state.

---

### P10 — Resize reflow hack in injector.js may cause FOUC

```javascript
window.addEventListener('resize', () => {
    document.body.style.display = 'none';
    document.body.offsetHeight;   // force reflow
    document.body.style.display = '';
});
```
This fires on every window resize event (which fires continuously while dragging the window edge). Hiding and showing the body on every frame causes a flash-of-unstyled-content and may interact badly with the Qt WebEngine rendering pipeline that this hack is meant to fix.
