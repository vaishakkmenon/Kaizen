# Kaizen Rebrand Plan — Full Audit & Dependency Map

## How to Read This Plan

Each **batch** is a group of changes that can be applied together and tested as a unit. Within a batch, every cross-file dependency has been traced — if file A changes something that file B depends on, both are in the same batch. Batches are ordered so that earlier batches don't depend on later ones.

**Notation:**
- `[SAFE]` = Change is internal to one file, no cross-file impact
- `[COORDINATED]` = Change spans multiple files that must be updated together
- `[DEFERRED]` = Change is noted but intentionally postponed to a later batch

---

## Batch 0: Metadata & Manifest (No Code Impact)

These files are read by Anki's addon manager but have zero runtime cross-references in code.

### `manifest.json`
| Line | Current | New |
|------|---------|-----|
| 1 | `"name": "Onigiri (by Peace)"` | `"name": "Kaizen"` |
| 3 | `"author": "PeaceMonk"` | `"author": "Vaishak Menon"` |

### `meta.json`
| Field | Current | New |
|-------|---------|-----|
| `"name"` | `"Onigiri (by Peace)"` | `"Kaizen"` |

### `onigiri_layout.json` → rename to `kaizen_layout.json`
- **Dependency check:** `grep -rn "onigiri_layout" --include="*.py"` → **zero hits referencing this filename.** The file is never loaded by code. It appears to be a static reference/default. **Safe to rename.**

### `onigiri_logo.png` → rename to `kaizen_logo.png`
- **Dependency check — COORDINATED with:**
  - `web/welcome.html:301` — `src="/_addons/%%ADDON_PACKAGE%%/onigiri_logo.png"`
  - `web/credits.html:236` — `src="/_addons/%%ADDON_PACKAGE%%/onigiri_logo.png"`
- Both HTML files must update the `src` path in this same batch.

### `web/welcome.html`
| Line | Current | New |
|------|---------|-----|
| 301 | `onigiri_logo.png` | `kaizen_logo.png` |
| 302 | `Welcome to Onigiri!` | `Welcome to Kaizen!` |
| 304 | `github.com/thepeacemonk/Onigiri/releases/tag/v1.1.0-dev` | Update to your repo URL |
| 308 | `github.com/thepeacemonk/Onigiri` | Update to your repo URL |

### `web/credits.html`
| Line | Current | New |
|------|---------|-----|
| 236 | `onigiri_logo.png` | `kaizen_logo.png` |

### `LICENSE.txt` / `README.md`
- Update branding text. No code dependencies.

**Test:** Addon loads, welcome dialog shows new name/logo, credits dialog shows new logo.

---

## Batch 1: Menu Bar & Settings Dialog Title (`[COORDINATED]`)

The menu bar label "Onigiri" and the "Onigiri Settings" menu item text are user-facing strings.

### `menu_buttons.py`
| Line | Current | New | Notes |
|------|---------|-----|-------|
| 1 | `# --- Onigiri ---` | `# --- Kaizen ---` | Comment only |
| 19 | `def get_onigiri_version():` | `def get_kaizen_version():` | Function name |
| 40-41 | `Onigiri Error: addon_path not set` | `Kaizen Error: addon_path not set` | Print msg |
| 48 | `def setup_onigiri_menu(addon_path):` | `def setup_kaizen_menu(addon_path):` | Function name |
| 60 | `QMenu("Onigiri", mw)` | `QMenu("Kaizen", mw)` | **User-facing** |
| 82 | `"Onigiri Settings"` | `"Kaizen Settings"` | **User-facing** |
| 98 | `get_onigiri_version()` | `get_kaizen_version()` | Follows rename |

- **Cross-file dependency:** `__init__.py:303` calls `menu_buttons.setup_onigiri_menu(addon_path)`

### `__init__.py`
| Line | Current | New |
|------|---------|-----|
| 303 | `menu_buttons.setup_onigiri_menu(addon_path)` | `menu_buttons.setup_kaizen_menu(addon_path)` |

**No other file calls `setup_onigiri_menu` or `get_onigiri_version`.**

### `welcome_dialog.py`
| Line | Current | New |
|------|---------|-----|
| 17 | `self.setWindowTitle("Welcome to Onigiri")` | `self.setWindowTitle("Welcome to Kaizen")` |

**Test:** Menu bar shows "Kaizen", settings menu item says "Kaizen Settings", welcome dialog title is correct.

---

## Batch 2: Module Rename `onigiri_renderer.py` → `kaizen_renderer.py` (`[COORDINATED]`)

This is a file rename that requires updating every import site.

### File: `onigiri_renderer.py` → rename to `kaizen_renderer.py`

**All import sites (must all update in this batch):**

| File | Line | Current | New |
|------|------|---------|-----|
| `__init__.py` | 5 | `from . import onigiri_renderer` | `from . import kaizen_renderer` |
| `__init__.py` | 368 | `onigiri_renderer.render_onigiri_deck_browser` | `kaizen_renderer.render_kaizen_deck_browser` |
| `__init__.py` | 373 | (no change needed — calls `patcher._onigiri_render_deck_node`) | [DEFERRED to Batch 5] |
| `deck_tree_updater.py` | 6 | `from . import onigiri_renderer` | `from . import kaizen_renderer` |
| `deck_tree_updater.py` | 18 | `onigiri_renderer.RenderData(...)` | `kaizen_renderer.RenderData(...)` |
| `deck_tree_updater.py` | 37 | `onigiri_renderer.RenderData(...)` | `kaizen_renderer.RenderData(...)` |
| `deck_tree_updater.py` | 143 | `onigiri_renderer.RenderData(...)` | `kaizen_renderer.RenderData(...)` |
| `patcher.py` | 39 | `from . import onigiri_renderer` | `from . import kaizen_renderer` |

### Inside the renamed `kaizen_renderer.py`:
| Line | Current | New |
|------|---------|-----|
| 480 | `def render_onigiri_deck_browser(...)` | `def render_kaizen_deck_browser(...)` |

**No other file references `render_onigiri_deck_browser` except `__init__.py:368` (covered above).**

`RenderData` class (line 16) — name is generic, no rename needed.

**Test:** Addon loads, deck browser renders correctly.

---

## Batch 3: Pycmd Bridge Commands (`[COORDINATED]`)

These are string-matched commands sent from JS → Python via Anki's `pycmd()` bridge. Both the sender (JS/HTML) and handler (Python) must match exactly.

### Command: `openOnigiriSettings` → `openKaizenSettings`

| File | Line | Role | Current | New |
|------|------|------|---------|-----|
| `kaizen_renderer.py` (renamed) | 49 | Sender | `pycmd('openOnigiriSettings')` | `pycmd('openKaizenSettings')` |
| `web/injector.js` | 92 | Sender | `cmd: 'openOnigiriSettings'` | `cmd: 'openKaizenSettings'` |
| `patcher.py` | 1069 | Handler | `if cmd == "openOnigiriSettings":` | `if cmd == "openKaizenSettings":` |

### Command: `onigiri_create_deck` → `kaizen_create_deck`

| File | Line | Role | Current | New |
|------|------|------|---------|-----|
| `kaizen_renderer.py` | 65 | Sender | `pycmd('onigiri_create_deck')` | `pycmd('kaizen_create_deck')` |
| `web/injector.js` | 145 | Sender | `cmd: 'onigiri_create_deck'` | `cmd: 'kaizen_create_deck'` |
| `webview_handlers.py` | 12 | Handler | `if cmd == "onigiri_create_deck":` | `if cmd == "kaizen_create_deck":` |

### Command: `onigiri_collapse:{deck_id}` → `kaizen_collapse:{deck_id}`

| File | Line | Role | Current | New |
|------|------|------|---------|-----|
| `kaizen_renderer.py` | 990 | Sender | `pycmd(\`onigiri_collapse:...\`)` | `pycmd(\`kaizen_collapse:...\`)` |
| `patcher.py` | 3699 | Sender | `pycmd(\"onigiri_collapse:...\")` | `pycmd(\"kaizen_collapse:...\")` |
| `webview_handlers.py` | 29 | Handler | `cmd.startswith("onigiri_collapse:")` | `cmd.startswith("kaizen_collapse:")` |

### Command: `onigiri_toggle_favorite:{deck_id}` → `kaizen_toggle_favorite:{deck_id}`

| File | Line | Role | Current | New |
|------|------|------|---------|-----|
| `patcher.py` | 3547 | Sender | `pycmd('onigiri_toggle_favorite:...')` | `pycmd('kaizen_toggle_favorite:...')` |
| `webview_handlers.py` | 39 | Handler | `cmd.startswith("onigiri_toggle_favorite:")` | `cmd.startswith("kaizen_toggle_favorite:")` |

### Command: `onigiri_show_transfer_window:{...}` → `kaizen_show_transfer_window:{...}`

| File | Line | Role | Current | New |
|------|------|------|---------|-----|
| `templates.py` | 409 | Sender | `pycmd(\`onigiri_show_transfer_window:...\`)` | `pycmd(\`kaizen_show_transfer_window:...\`)` |
| `web/injector.js` | 361 | Sender | `pycmd(\`onigiri_show_transfer_window:...\`)` | `pycmd(\`kaizen_show_transfer_window:...\`)` |
| `webview_handlers.py` | 74 | Handler | `cmd.startswith("onigiri_show_transfer_window:")` | `cmd.startswith("kaizen_show_transfer_window:")` |
| `gamification/mod_transfer_window.py` | 361 | Handler | `message.startswith("onigiri_show_transfer_window:")` | `message.startswith("kaizen_show_transfer_window:")` |
| `gamification/mod_transfer_window.py` | 363 | Parser | `message.replace("onigiri_show_transfer_window:", "")` | `message.replace("kaizen_show_transfer_window:", "")` |

### Command: `saveSidebarState` — **NO RENAME NEEDED**
- Does not contain "onigiri". Leave as-is.

**Test:** Click settings icon → settings opens. Create deck → dialog opens. Collapse/expand decks → works. Toggle favorites → works. Transfer window → works.

---

## Batch 4: JS Global Objects (`[COORDINATED]`)

These are `window.*` globals defined in JS and called from Python via `web.eval()`.

### `OnigiriHeatmap` → `KaizenHeatmap`

| File | Line | Role | Current | New |
|------|------|------|---------|-----|
| `web/heatmap.js` | 7-8 | Definition | `window.OnigiriHeatmap = window.OnigiriHeatmap \|\| {}` | `window.KaizenHeatmap = ...` |
| `web/heatmap.js` | 363 | Closure | `})(window.OnigiriHeatmap)` | `})(window.KaizenHeatmap)` |
| `__init__.py` | 381 | Caller | `OnigiriHeatmap.render(...)` | `KaizenHeatmap.render(...)` |
| `patcher.py` | 987 | Caller | `OnigiriHeatmap.render(...)` | `KaizenHeatmap.render(...)` |

### `OnigiriEngine` → `KaizenEngine`

| File | Line | Role | Current | New |
|------|------|------|---------|-----|
| `web/engine.js` | 3 | Definition | `window.OnigiriEngine = {` | `window.KaizenEngine = {` |
| `web/engine.js` | 263 | Init call | `OnigiriEngine.init()` | `KaizenEngine.init()` |
| `web/engine.js` | 265 | Init call | `OnigiriEngine.init()` | `KaizenEngine.init()` |
| `kaizen_renderer.py` | 930 | Comment | `# Add OnigiriEngine JavaScript` | `# Add KaizenEngine JavaScript` |
| `kaizen_renderer.py` | 934 | Inline def | `window.OnigiriEngine = {` | `window.KaizenEngine = {` |
| `kaizen_renderer.py` | 942 | Console log | `'OnigiriEngine initialized'` | `'KaizenEngine initialized'` |
| `kaizen_renderer.py` | 1056-1058 | Init calls | `OnigiriEngine.init()` | `KaizenEngine.init()` |
| `deck_tree_updater.py` | 62 | Caller | `OnigiriEngine.updateDeckTree(...)` | `KaizenEngine.updateDeckTree(...)` |
| `deck_tree_updater.py` | 168 | Caller | `OnigiriEngine.updateDeckTree(...)` | `KaizenEngine.updateDeckTree(...)` |

### `OnigiriNotifications` → `KaizenNotifications`

| File | Line | Role | Current | New |
|------|------|------|---------|-----|
| `web/notifications.js` | 2 | Guard | `if (window.OnigiriNotifications)` | `if (window.KaizenNotifications)` |
| `web/notifications.js` | 30 | Error msg | `"OnigiriNotifications: pending..."` | `"KaizenNotifications: pending..."` |
| `web/notifications.js` | 167 | Assignment | `window.OnigiriNotifications = api` | `window.KaizenNotifications = api` |
| `gamification/mochi_messages.py` | 70-71 | Caller | `window.OnigiriNotifications.show(...)` | `window.KaizenNotifications.show(...)` |
| `gamification/restaurant_level.py` | 826-828 | Caller | `window.OnigiriNotifications.show(...)` | `window.KaizenNotifications.show(...)` |

### `SyncStatusManager` — **NO RENAME NEEDED**
- Does not contain "onigiri". Leave as-is.

**Test:** Heatmap renders. Deck tree updates on changes. Notifications fire. Sync indicator works.

---

## Batch 5: Python Function & Variable Names (`[SAFE]` — internal only)

These are function/method names and local variables that never cross file boundaries (except where already handled in earlier batches).

### `patcher.py`
| Line | Current | New |
|------|---------|-----|
| 3519 | `def _onigiri_render_deck_node(self, node, ctx):` | `def _kaizen_render_deck_node(self, node, ctx):` |

- **Cross-file dependency:** `__init__.py:373` → `patcher._onigiri_render_deck_node`
- **Must also update `__init__.py:373`:**
  ```
  DeckBrowser._render_deck_node = patcher._kaizen_render_deck_node
  ```

### `kaizen_renderer.py` (internal functions — all `[SAFE]`)
| Line | Current | New |
|------|---------|-----|
| 187 | `def _get_onigiri_stat_card_html(...)` | `def _get_kaizen_stat_card_html(...)` |
| 195 | `def _get_onigiri_retention_html()` | `def _get_kaizen_retention_html()` |
| 240 | `def _get_onigiri_heatmap_html()` | `def _get_kaizen_heatmap_html()` |
| 249 | `def _get_onigiri_favorites_html()` | `def _get_kaizen_favorites_html()` |
| 347 | `def _get_onigiri_restaurant_level_html()` | `def _get_kaizen_restaurant_level_html()` |
| 492 | `onigiri_layout = conf.get(...)` | `kaizen_layout = conf.get(...)` |
| 495 | `onigiri_grid_html = ""` | `kaizen_grid_html = ""` |
| 533,541,577 | All references to `onigiri_layout`, `onigiri_grid_html` | Update to `kaizen_layout`, `kaizen_grid_html` |

These are all internal to `kaizen_renderer.py` — called only within the same file. **No cross-file impact** except the function references used in `widget_generators` dict (line ~528), which is also internal.

### `settings.py` (internal variable names — all `[SAFE]`)
| Location | Current | New |
|----------|---------|-----|
| 5106 | `_ONIGIRI_DEFAULTS = {` | `_KAIZEN_DEFAULTS = {` |
| 5195-5204 | `self.onigiri_scroll`, `self.onigiri_archive_zone` | `self.kaizen_scroll`, `self.kaizen_archive_zone` |
| 5293-5329 | `saved_onigiri_layout`, `onigiri_grid_config`, `onigiri_archive_config` | `saved_kaizen_layout`, `kaizen_grid_config`, `kaizen_archive_config` |
| 5579 | `self._ONIGIRI_DEFAULTS` references | `self._KAIZEN_DEFAULTS` |

All of these are local to the `settings.py` class — no cross-file references.

### `__init__.py` (comment-only changes)
| Line | Current | New |
|------|---------|-----|
| 78 | `# Inject global Onigiri CSS` | `# Inject global Kaizen CSS` |
| 390 | `"""Updates the sync status indicator in the Onigiri menu."""` | `"""...in the Kaizen menu."""` |
| 405 | `Ensures that Onigiri takes control of external hooks` | `Ensures that Kaizen takes control...` |

### `deck_tree_updater.py`
| Line | Current | New |
|------|---------|-----|
| 94 | `hasattr(mw, "onigiri_transfer_window")` | `hasattr(mw, "kaizen_transfer_window")` |
| 96 | `mw.onigiri_transfer_window.close()` | `mw.kaizen_transfer_window.close()` |
| 99 | `mw.onigiri_transfer_window = None` | `mw.kaizen_transfer_window = None` |

- **Cross-file dependency:** `gamification/mod_transfer_window.py:355` → `mw.onigiri_transfer_window = web`
- **Must also update `gamification/mod_transfer_window.py:355`:**
  ```
  mw.onigiri_transfer_window = web  →  mw.kaizen_transfer_window = web
  ```

**Test:** Deck browser renders. Deck node rendering works. Settings dialog layout tab works. Transfer window cleanup works.

---

## Batch 6: CSS Classes, IDs & HTML Data Attributes (`[COORDINATED]`)

Each group below lists every file that references the selector. All files in a group must update together.

### Group 6A: Notification CSS classes

**Pattern:** `onigiri-notification-*` → `kaizen-notification-*`

| Selector | Files |
|----------|-------|
| `.onigiri-notification-stack` | `__init__.py:42`, `web/notifications.js:6,51`, `web/notifications.css:9,18` |
| `.onigiri-notification-card` | `web/notifications.js:72`, `web/notifications.css:24,37,43-48,51,81,85-88` |
| `.onigiri-notification-icon` | `web/notifications.js:87`, `web/notifications.css:56,68-71,75,81,85-88` |
| `.onigiri-notification-content` | `web/notifications.js:108`, `web/notifications.css:92` |
| `.onigiri-notification-title` | `web/notifications.js:111`, `web/notifications.css:98` |
| `.onigiri-notification-description` | `web/notifications.js:115`, `web/notifications.css:106` |

**CSS custom properties (in `web/notifications.css`):**
| Current | New |
|---------|-----|
| `--onigiri-notification-max-width` | `--kaizen-notification-max-width` |
| `--onigiri-notification-bg-light` | `--kaizen-notification-bg-light` |
| `--onigiri-notification-bg-dark` | `--kaizen-notification-bg-dark` |
| `--onigiri-notification-text-light` | `--kaizen-notification-text-light` |
| `--onigiri-notification-text-dark` | `--kaizen-notification-text-dark` |

**Files to edit:** `web/notifications.css`, `web/notifications.js`, `__init__.py`

### Group 6B: Heatmap IDs and classes

| Selector | Files |
|----------|-------|
| `#onigiri-heatmap-container` | `web/heatmap.css:5,387`, `__init__.py:381` (as string arg), `web/heatmap.js:310` |
| `#onigiri-profile-heatmap-container` | `web/heatmap.css:6,388`, `patcher.py:987` (as string arg) |
| `.onigiri-heatmap-header` | `web/heatmap.css:22,36-37`, `web/heatmap.js:310` |

**Files to edit:** `web/heatmap.css`, `web/heatmap.js`, `__init__.py`, `patcher.py`

### Group 6C: Favorites widget class

| Selector | Files |
|----------|-------|
| `.onigiri-favorites-widget` | `kaizen_renderer.py:258,322,333,344`, `web/menu.css:1255,1269` |

**Files to edit:** `kaizen_renderer.py`, `web/menu.css`

### Group 6D: Profile page class

| Selector | Files |
|----------|-------|
| `.onigiri-profile-page` | `web/profile.css:17`, `web/profile_page.js:35` |

**Files to edit:** `web/profile.css`, `web/profile_page.js`

### Group 6E: Reviewer header/button classes (inline CSS in `patcher.py`)

| Selector | Files |
|----------|-------|
| `#onigiri-reviewer-header` | `__init__.py:128,131,142` |
| `#onigiri-background-div` | `__init__.py:117,119` |
| `.onigiri-reviewer-header-buttons` | `patcher.py:1323` (+ more lines) |
| `.onigiri-reviewer-button` | `patcher.py:1324-1328,1499-1503,2247-2251` |
| `--onigiri-reviewer-header-offset` | `__init__.py:48,150` |

**Files to edit:** `__init__.py`, `patcher.py`

Note: These classes are styled via **inline CSS in patcher.py** (not in `menu.css`), so the CSS definitions and HTML usage are in the same file.

### Group 6F: Data attributes

| Attribute | Files |
|-----------|-------|
| `data-onigiri-icon` | `sidebar_api.py:179,191`, `web/menu.css:1397` |
| `data-onigiri-ease` | `patcher.py:4007,4016,4027,4036,4055,4079` |
| `data-onigiri-setup` | `web/injector.js:409,410,542` |
| `data-onigiri-classified` | `web/engine.js:248,249` |

**Files to edit:** `sidebar_api.py`, `web/menu.css`, `patcher.py`, `web/injector.js`, `web/engine.js`

### Group 6G: Menu CSS keyframe

| Name | Files |
|------|-------|
| `@keyframes onigiri-sync-pulse` | `web/menu.css:1221,1239` |

**Files to edit:** `web/menu.css` only (self-contained)

### Group 6H: Widget/restaurant classes (inline CSS in `kaizen_renderer.py`)

| Selector | Files |
|----------|-------|
| `.onigiri-widget-container` | `kaizen_renderer.py:541,606` |
| `.onigiri-widget-title` | `kaizen_renderer.py:574` |
| `.onigiri-restaurant-level-widget` | `kaizen_renderer.py:359,456,457,624,637,642,688,711,715,730,825,859,915` |

**Files to edit:** `kaizen_renderer.py` only (inline HTML + inline CSS, self-contained)

### Group 6I: Overview/external addon class

| Selector | Files |
|----------|-------|
| `.onigiri-external-overview-addon` | `web/overview.css:21` |

**Files to edit:** `web/overview.css`  
**Dependency check:** Search `onigiri-external-overview-addon` in .py/.js → found in `patcher.py` (line ~2180, generates HTML with this class). **Must update `patcher.py` too.**

**Test:** Full visual regression — every screen (deck browser, reviewer, overview, profile, notifications) renders correctly with no broken selectors.

---

## Batch 7: Config Keys — Addon Config (`config.py` DEFAULTS) (`[COORDINATED]`)

These are keys stored in the per-profile JSON settings file. Renaming them requires updating every `.get("onigiri_...")` and `["onigiri_..."]` reference across all files.

### 7A: `onigiriWidgetLayout` → `kaizenWidgetLayout`

**All references (8 sites):**
| File | Line(s) |
|------|---------|
| `config.py` | 102, 448 |
| `__init__.py` | 377 |
| `kaizen_renderer.py` | 492, 493 |
| `settings.py` | 4868, 5085, 5162, 5293, 12239 |

### 7B: `onigiri_reviewer_*` keys (40+ keys)

These are the `onigiri_reviewer_bg_mode`, `onigiri_reviewer_btn_*`, etc. keys. They appear in:
- `config.py` DEFAULTS (~lines 134-210)
- `patcher.py` (reads them via `conf.get(...)`)
- `settings.py` (reads/writes them)

**Strategy:** Global find-and-replace `"onigiri_reviewer_` → `"kaizen_reviewer_` across `config.py`, `patcher.py`, `settings.py`.

### 7C: `onigiri_overview_*` keys (~10 keys)

Same pattern. Files: `config.py`, `patcher.py`, `settings.py`.

### 7D: `onigiri_font_*` keys (6 keys)

Files: `config.py`, `patcher.py`, `settings.py`.

### 7E: `sidebarActionsMode`, `sidebarButtonLayout`, `sidebarCollapsed`

These do NOT contain "onigiri" — **no rename needed.**

**Test:** Settings save/load correctly. All visual preferences persist across restart.

---

## Batch 8: Config Keys — Collection Conf (`mw.col.conf`) (`[COORDINATED]`)

These are stored in the Anki collection database, not in the addon config file. Renaming them means existing users lose their settings (acceptable for personal fork).

### Full list of `mw.col.conf` keys to rename:

| Current Key | Files That Read/Write It |
|-------------|--------------------------|
| `onigiri_sidebar_collapsed` | `patcher.py:1105`, `kaizen_renderer.py:1064` |
| `onigiri_custom_deck_icons` | `icon_chooser.py:24,87,112,114` |
| `onigiri_favorite_decks` | `kaizen_renderer.py:255,314` |
| `onigiri_favorites` | `settings.py:10761,10792` |
| `onigiri_deck_focus_mode` | `kaizen_renderer.py:1065` |
| `onigiri_overview_style` | `settings.py:7584,11972,11974` |
| `onigiri_sidebar_main_bg_effect_mode` | `settings.py:7681,12103,12105` |
| `onigiri_sidebar_main_bg_effect_intensity` | `settings.py:7724,12107` |
| `onigiri_sidebar_opaque_tint_intensity` | `settings.py:7702,12108` |
| `onigiri_sidebar_opaque_tint_color_light` | `settings.py:7710,12109` |
| `onigiri_sidebar_opaque_tint_color_dark` | `settings.py:7711,12110` |
| `onigiri_profile_page_bg_mode` | `settings.py:8207` (+ `patcher.py`) |
| `onigiri_profile_page_bg_*` (6 keys) | `settings.py:8214-8221` (+ `patcher.py`) |
| `onigiri_profile_show_theme_light` | `settings.py:8226` (+ `patcher.py`) |
| `onigiri_profile_show_theme_dark` | `settings.py:8227` (+ `patcher.py`) |
| `onigiri_profile_show_backgrounds` | `settings.py:8228` (+ `patcher.py`) |
| `onigiri_profile_show_stats` | `settings.py:8229`, `config.py:422-423` (+ `patcher.py`) |
| `onigiri_canvas_inset_effect_mode` | `settings.py:8298` (+ `patcher.py`) |
| `onigiri_canvas_inset_effect_intensity` | `settings.py:8306` (+ `patcher.py`) |
| `onigiri_toolbar_bg_mode` | `patcher.py` (+ `settings.py`) |
| `onigiri_toolbar_bg_*` (4 keys) | `patcher.py`, `settings.py` |
| `onigiri_font_*` (6 keys) | `patcher.py`, `settings.py` |
| `onigiri_reviewer_bg_color_theme_mode` | `patcher.py`, `settings.py` |
| `onigiri_reviewer_bg_image_theme_mode` | `patcher.py`, `settings.py` |
| `onigiri_reviewer_bottom_bar_bg_*` (4 keys) | `patcher.py`, `settings.py` |
| `onigiri_overview_bg_*_theme_mode` (2 keys) | `patcher.py`, `settings.py` |
| `onigiri_check_sync_status` | `patcher.py` |
| `onigiri_conf` | `patcher.py` |

**Strategy:** Global find-and-replace `"onigiri_` → `"kaizen_` in `patcher.py`, `settings.py`, `config.py`, `kaizen_renderer.py`, `icon_chooser.py`. Then verify no orphaned references.

**Test:** All settings persist after save. Sidebar collapse state works. Custom deck icons appear. Favorites work. All background settings apply correctly.

---

## Batch 9: Image File References (`[COORDINATED]`)

### `system_files/profile_default/onigiri-san.png` → rename to `kaizen-default.png`
### `system_files/profile_default/onigiri-bg.png` → rename to `kaizen-bg.png`

**All references:**

| File | Line | Current | New |
|------|------|---------|-----|
| `kaizen_renderer.py` | 183 | `default_pic = "onigiri-san.png"` | `"kaizen-default.png"` |
| `kaizen_renderer.py` | 1093 | `onigiri-bg.png` | `kaizen-bg.png` |
| `patcher.py` | 196 | `default_pic = "onigiri-san.png"` | `"kaizen-default.png"` |
| `patcher.py` | 818 | `onigiri-bg.png` | `kaizen-bg.png` |
| `patcher.py` | 1524 | `onigiri-bg.png` | `kaizen-bg.png` |
| `settings.py` | 398 | `"onigiri-san.png"` | `"kaizen-default.png"` |
| `settings.py` | 437 | `"onigiri-bg.png"` | `"kaizen-bg.png"` |

**Test:** Profile picture fallback shows. Background image loads in all contexts (deck browser, reviewer, profile page).

---

## Batch 10: Comments, Print Statements & Debug Strings (`[SAFE]`)

These have zero functional impact. Purely cosmetic cleanup.

### Pattern: `[Onigiri ...]` → `[Kaizen ...]` in print/console statements
### Pattern: `# ... Onigiri ...` → `# ... Kaizen ...` in comments
### Pattern: `"""...Onigiri..."""` → `"""...Kaizen..."""` in docstrings

**Files (by count of comment/print references):**
- `settings.py` — extensive (dialog titles, section labels, tooltips)
- `patcher.py` — extensive
- `icon_chooser.py` — ~15 print statements
- `sidebar_api.py` — ~10 print statements + module docstring
- `webview_handlers.py` — ~5 print statements
- `config.py` — comments
- `constants.py` — review file for any branding
- `fonts.py` — module docstring
- `favorites_cleanup.py` — module docstring
- `gamification/*.py` — various (only matters if gamification stays)

**Test:** No functional test needed. Grep to confirm: `grep -rni "onigiri" --include="*.py"` returns zero hits.

---

## Batch 11: Gamification System — Archive/Disable (`[COORDINATED]`)

This is the biggest structural change. The gamification system (restaurant leveling, achievements, XP, taiyaki store, mochi messages, focus dango) should be disabled but code preserved.

### What to disable:

**`__init__.py` — remove/guard imports:**
| Line | Import | Action |
|------|--------|--------|
| 14 | `from .gamification import mochi_messages` | Comment out or guard |
| 15 | `from .gamification import mod_transfer_window` | Comment out or guard |
| 19 | `from .gamification import focus_dango` | Comment out or guard |
| 25 | `from .gamification.taiyaki_store import open_taiyaki_store` | Comment out or guard |
| 215+ | `verify_coin_integrity()` function + call | Guard behind `gamificationMode` |
| 322 | `verify_coin_integrity()` call in `on_profile_did_open` | Guard |
| 324 | `setup_shop_menu()` call | Guard |

**`menu_buttons.py` — remove gamification menu items:**
| Line(s) | Item | Action |
|---------|------|--------|
| 13 | `from .gamification.taiyaki_store import open_taiyaki_store` | Comment out |
| 67-78 | Gamification submenu (Restaurant Level, Mr. Taiyaki Store) | Remove or guard |

**`patcher.py` — guard gamification imports and calls:**
| Line | Import | Action |
|------|--------|--------|
| 41 | `from .gamification import restaurant_level` | Guard |
| 43 | `from .gamification.gamification import get_gamification_manager` | Guard |
| 46 | `from .gamification import focus_dango` | Guard |
| 48 | `from .gamification.restaurant_level_ui import RestaurantLevelWidget` | Guard |

**`settings.py` — guard gamification tab/settings:**
| Line | Import | Action |
|------|--------|--------|
| 33 | `from .gamification import restaurant_level` | Guard |
| 12304 | `from .gamification import focus_dango` | Guard |

**`kaizen_renderer.py` — guard restaurant level widget:**
| Line | Import | Action |
|------|--------|--------|
| 11 | `from .gamification import restaurant_level` | Guard |
| 347-477 | `_get_onigiri_restaurant_level_html()` | Guard or return empty |

**Config keys to force-disable:**
- `gamificationMode`: force `False`
- `achievements.enabled`: force `False`
- `restaurant_level.enabled`: force `False`
- `mochi_messages.enabled`: force `False`
- `daily_special.enabled`: force `False`

**Test:** Addon loads without gamification. No restaurant level widget. No mochi messages. No taiyaki store in menu. No XP tracking. Settings dialog still works without gamification tabs.

---

## Batch 12: TempContent Bug Fix (`[SAFE]`)

This was a fix you previously applied. Need to re-verify and re-apply.

**The bug:** `TempContent` (or equivalent render data) missing a `tree` attribute that external addons expect.

**Location:** `kaizen_renderer.py`, `RenderData` class (line 16).

**Fix:** Ensure `RenderData` has a `tree` attribute. Verify `deck_tree_updater.py` passes it correctly.

**Test:** Install with another addon that hooks into deck browser rendering. Confirm no `AttributeError`.

---

## Summary: Execution Order

| Batch | Scope | Files Modified | Risk |
|-------|-------|----------------|------|
| 0 | Metadata + static assets | 5 files + 1 rename | Minimal |
| 1 | Menu bar + dialog titles | 3 files | Minimal |
| 2 | Module rename | 5 files + 1 rename | Medium — import chain |
| 3 | Pycmd bridge commands | 6 files | Medium — JS↔Python sync |
| 4 | JS global objects | 8 files | Medium — JS↔Python sync |
| 5 | Python function/variable names | 7 files | Low — mostly internal |
| 6 | CSS classes/IDs/data attrs | 12 files | High — visual breakage |
| 7 | Addon config keys | 4 files | Medium — settings loss |
| 8 | Collection conf keys | 5 files | Medium — settings loss |
| 9 | Image file references | 4 files + 2 renames | Low |
| 10 | Comments & debug strings | 15+ files | Zero risk |
| 11 | Gamification disable | 6+ files | Medium — import chain |
| 12 | TempContent bug fix | 1 file | Low |

---

## Verification Checklist (After All Batches)

```bash
# Should return ZERO results:
grep -rni "onigiri" --include="*.py" --include="*.js" --include="*.css" --include="*.html" --include="*.json" .

# Except possibly:
# - gamification/ files (if archiving, not deleting)
# - LICENSE.txt (attribution to original author)
# - README.md (attribution section)
```