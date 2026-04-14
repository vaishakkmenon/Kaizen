# Kaizen Restructure Plan

## Current State

22 Python files dumped in root. `settings.py` is 11,714 lines. `patcher.py` is 3,633 lines. A 152KB debug log is committed. No `.gitignore` for Python artifacts.

## Target Structure

```
Kaizen/
├── __init__.py              # entry point — slim, just hooks and wiring
├── config.py                # config get/set/defaults (stays at root, everything imports it)
├── constants.py             # static data dicts (stays at root)
├── themes.py                # theme palette definitions (stays at root)
├── fonts.py                 # font loading (stays at root)
├── templates.py             # HTML template strings (stays at root)
├── kaizen_layout.json
├── manifest.json
├── meta.json
├── config.json
├── CLAUDE.md                # NEW — project context for Claude Code
├── .gitignore               # UPDATED
│
├── core/                    # Non-UI logic
│   ├── __init__.py
│   ├── renderer.py          # ← kaizen_renderer.py
│   ├── deck_tree.py         # ← deck_tree_updater.py
│   ├── heatmap.py           # ← heatmap.py
│   ├── sidebar_api.py       # ← sidebar_api.py
│   └── favorites.py         # ← favorites_cleanup.py
│
├── patches/                 # Split from patcher.py (3633 lines → ~6 files)
│   ├── __init__.py          # re-exports everything so `patcher.py` facade works
│   ├── css.py               # generate_dynamic_css, generate_conditional_css,
│   │                        # generate_font_css, generate_icon_css,
│   │                        # generate_icon_size_css, generate_profile_bar_fix_css,
│   │                        # generate_reviewer_buttons_css,
│   │                        # _hex_to_rgba, _mix_colors
│   ├── backgrounds.py       # _render_background_css, generate_deck_browser_backgrounds,
│   │                        # generate_reviewer_background_css,
│   │                        # generate_overview_background_css,
│   │                        # generate_toolbar_background_css,
│   │                        # generate_reviewer_bottom_bar_background_css,
│   │                        # generate_reviewer_top_bar_html_and_css,
│   │                        # _generate_outer_background_css,
│   │                        # generate_profile_page_background_css
│   ├── deck_node.py         # _kaizen_render_deck_node, _kaizen_render_deck_tree
│   ├── hooks.py             # apply_patches, apply_menu_styling, patch_qmenu,
│   │                        # patch_overview, patch_congrats_page,
│   │                        # take_control_of_deck_browser_hook,
│   │                        # _new_MainWebView_eventFilter,
│   │                        # _update_toolbar_visibility, _on_sync_did_finish
│   ├── profile.py           # ProfileDialog, open_profile,
│   │                        # _get_profile_pic_html, _get_profile_pill_html,
│   │                        # _get_theme_colors_html, _get_backgrounds_html,
│   │                        # _get_stats_html, _get_heatmap_data_and_config_for_profile,
│   │                        # _get_profile_header_html, _generate_profile_html_body,
│   │                        # get_sync_status
│   └── webview.py           # on_webview_js_message, _get_hook_name, _get_external_hooks
│
├── ui/                      # Split from settings.py (11714 lines → ~7 files) + dialogs
│   ├── __init__.py
│   ├── widgets.py           # FlowLayout, ThumbnailWorker, SelectionOverlay,
│   │                        # CircularColorButton, AnimatedToggleButton,
│   │                        # ProfileBarWidget, SidebarToggleButton, SectionGroup
│   │                        # (lines 49–560, ~510 lines)
│   ├── theme_widgets.py     # ColorSwatch, ThemeCardWidget, BirthdayWidget,
│   │                        # FontCardWidget
│   │                        # (lines 561–1094, ~530 lines)
│   ├── color_picker.py      # ColorMapWidget, GradientSlider, HueSlider, AlphaSlider,
│   │                        # FavoriteColorButton, ModernColorPickerDialog
│   │                        # (lines 1095–1592, ~500 lines)
│   ├── icon_picker.py       # IconPickerDialog
│   │                        # (lines 1593–1927, ~335 lines)
│   ├── search.py            # SearchResultWidget, SettingsSearchPage
│   │                        # (lines 1928–2191, ~265 lines)
│   ├── donation.py          # DonationDialog
│   │                        # (lines 2192–2260, ~70 lines)
│   ├── settings_dialog.py   # SettingsDialog + open_settings()
│   │                        # (lines 2261–11714, ~9450 lines — still big but isolated)
│   ├── helpers.py           # ← settings_helpers.py
│   ├── menu_buttons.py      # ← menu_buttons.py
│   ├── birthday.py          # ← birthday_dialog.py
│   ├── welcome.py           # ← welcome_dialog.py
│   ├── credits.py           # ← credits_dialog.py
│   ├── create_deck.py       # ← create_deck_dialog.py
│   ├── coloris.py           # ← coloris_picker.py
│   └── icon_chooser.py      # ← icon_chooser.py
│
├── web/                     # unchanged
├── system_files/            # unchanged
│   └── (move kaizen_logo.png here)
├── _archived/               # unchanged
└── docs/
    └── gameplan.md          # ← moved from root
```

## Execution Phases

Execute each phase as a separate commit. Test by importing the module after each phase.

---

### Phase 0: Quick Wins

1. Update `.gitignore`:
```
__pycache__/
*.pyc
.DS_Store
sidebar_debug.log
user_files/
meta.json
```

2. Remove `sidebar_debug.log` from tracking: `git rm --cached sidebar_debug.log`

3. Move `kaizen-gameplan.md` to `docs/gameplan.md` (mkdir docs first)

4. Move `kaizen_logo.png` to `system_files/kaizen_logo.png` and update all references

5. Commit: `chore: clean up repo — gitignore, move docs and logo`

---

### Phase 1: Create `core/` — Move Pure Logic Files

These files have minimal imports and no circular dependencies.

```bash
mkdir -p core
```

Move these files (rename as shown):
- `kaizen_renderer.py` → `core/renderer.py`
- `deck_tree_updater.py` → `core/deck_tree.py`
- `heatmap.py` → `core/heatmap.py`
- `sidebar_api.py` → `core/sidebar_api.py`
- `favorites_cleanup.py` → `core/favorites.py`

Create `core/__init__.py`:
```python
from .renderer import render_kaizen_deck_browser
from .deck_tree import _render_deck_tree_html_only
from .heatmap import get_heatmap_and_config
from .sidebar_api import register_sidebar_action, ensure_capture_hook_is_last
from .favorites import cleanup_favorites
```

**Create backward-compat facade files at root** so existing imports don't break:
```python
# kaizen_renderer.py (root — facade)
from .core.renderer import *

# deck_tree_updater.py (root — facade)
from .core.deck_tree import *

# heatmap.py (root — facade)
from .core.heatmap import *

# sidebar_api.py (root — facade)
from .core.sidebar_api import *

# favorites_cleanup.py (root — facade)
from .core.favorites import *
```

**Fix internal imports inside moved files.** Each file that does `from . import config` now needs `from .. import config` since it's one directory deeper. Specifically:

- `core/renderer.py`: change `from . import config, heatmap, deck_tree_updater, sidebar_api` → `from .. import config` and `from . import heatmap, deck_tree, sidebar_api` and `from . import patcher` → `from .. import patcher`. Also `from .templates import custom_body_template` → `from ..templates import custom_body_template`
- `core/deck_tree.py`: change `from . import kaizen_renderer` → `from . import renderer`
- `core/heatmap.py`: change `from . import config` → `from .. import config` and `from .config import DEFAULTS` → `from ..config import DEFAULTS`
- `core/sidebar_api.py`: change `from . import config` → `from .. import config`
- `core/favorites.py`: no imports to change (has none)

**Verify:** `python -c "from Kaizen.core import renderer; print('OK')"` from the Anki addons directory.

Commit: `refactor: move core logic files to core/`

---

### Phase 2: Create `ui/` — Move Dialog Files

```bash
mkdir -p ui
```

Move these files (rename as shown):
- `birthday_dialog.py` → `ui/birthday.py`
- `welcome_dialog.py` → `ui/welcome.py`
- `credits_dialog.py` → `ui/credits.py`
- `create_deck_dialog.py` → `ui/create_deck.py`
- `coloris_picker.py` → `ui/coloris.py`
- `icon_chooser.py` → `ui/icon_chooser.py`
- `menu_buttons.py` → `ui/menu_buttons.py`
- `settings_helpers.py` → `ui/helpers.py`

Create `ui/__init__.py` (empty or with key exports).

Create backward-compat facade files at root for each moved file:
```python
# birthday_dialog.py (root — facade)
from .ui.birthday import *
# ... same pattern for each
```

Fix internal imports in moved files — `from . import config` → `from .. import config`, etc.

For `ui/menu_buttons.py`, fix:
- `from . import credits_dialog` → `from . import credits as credits_dialog` (or update the actual reference)
- `from . import patcher` → `from .. import patcher`
- `from . import settings` → `from .. import settings`

For `ui/welcome.py`, fix:
- `from . import config` → `from .. import config`
- `from . import settings` → `from .. import settings`

Commit: `refactor: move dialog files to ui/`

---

### Phase 3: Split `patcher.py` into `patches/`

This is the hardest phase. `patcher.py` is 3,633 lines with 14 `generate_*` functions, profile dialog, webview handlers, deck node rendering, and patching hooks.

```bash
mkdir -p patches
```

**Step 1: Identify shared state.** These module-level variables are referenced across functions:
- `addon_path = os.path.dirname(__file__)` — each submodule should define its own as `os.path.dirname(os.path.dirname(__file__))` (one level up)
- `_profile_dialog = None` — goes in `patches/profile.py`

**Step 2: Extract files.** Cut-paste functions into the appropriate file based on the file map above. For each file:

- `patches/css.py` — all `generate_*_css` functions plus `_hex_to_rgba` and `_mix_colors` utility functions. These mostly depend on `config`, `constants`, and `fonts`. Imports: `from .. import config`, `from ..constants import COLOR_LABELS`, `from ..fonts import get_all_fonts`

- `patches/backgrounds.py` — all `_render_background_css`, `generate_*_background*`, `_generate_outer_background_css`, `generate_profile_page_background_css`, `generate_reviewer_top_bar_html_and_css`. Imports: `from .. import config`, `from .css import _hex_to_rgba, generate_dynamic_css`

- `patches/deck_node.py` — `_kaizen_render_deck_node`, `_kaizen_render_deck_tree`. Imports: `from .. import config`

- `patches/hooks.py` — `apply_patches`, `apply_menu_styling`, `patch_qmenu`, `patch_overview`, `patch_congrats_page`, `take_control_of_deck_browser_hook`, `_new_MainWebView_eventFilter`, `_update_toolbar_visibility`, `_on_sync_did_finish`. These call functions from other patches submodules, so import: `from . import css, backgrounds, deck_node`

- `patches/profile.py` — `ProfileDialog`, `open_profile`, `get_sync_status`, and all `_get_profile_*` helper functions. Imports: `from .. import config`, `from .css import generate_dynamic_css`

- `patches/webview.py` — `on_webview_js_message`, `_get_hook_name`, `_get_external_hooks`, plus merge in `webview_handlers.py`. Imports: `from .. import config`, `from ..core import deck_tree`

**Step 3: Create `patches/__init__.py`** that re-exports everything:
```python
# Re-export all public functions for backward compatibility with `from . import patcher`
from .css import *
from .backgrounds import *
from .deck_node import *
from .hooks import *
from .profile import *
from .webview import *
```

**Step 4: Replace root `patcher.py`** with a thin facade:
```python
# patcher.py — backward-compat facade
# All functionality has been split into patches/ subpackage
from .patches import *
```

**Step 5: Delete root `webview_handlers.py`** (merged into `patches/webview.py`). Update `__init__.py` import accordingly.

Commit: `refactor: split patcher.py into patches/ subpackage`

---

### Phase 4: Split `settings.py` into `ui/` submodules

`settings.py` is 11,714 lines. Split by class boundaries.

**Step 1: Extract widget classes** (lines 49–560) into `ui/widgets.py`:
- FlowLayout, ThumbnailWorker, SelectionOverlay, CircularColorButton, AnimatedToggleButton, ProfileBarWidget, SidebarToggleButton, SectionGroup
- These are generic Qt widgets with no settings-specific logic

**Step 2: Extract theme widgets** (lines 561–1094) into `ui/theme_widgets.py`:
- ColorSwatch, ThemeCardWidget, BirthdayWidget, FontCardWidget

**Step 3: Extract color picker** (lines 1095–1592) into `ui/color_picker.py`:
- ColorMapWidget, GradientSlider, HueSlider, AlphaSlider, FavoriteColorButton, ModernColorPickerDialog

**Step 4: Extract icon picker** (lines 1593–1927) into `ui/icon_picker.py`:
- IconPickerDialog

**Step 5: Extract search** (lines 1928–2191) into `ui/search.py`:
- SearchResultWidget, SettingsSearchPage

**Step 6: Extract donation** (lines 2192–2260) into `ui/donation.py`:
- DonationDialog

**Step 7: Move SettingsDialog** (lines 2261–11714) into `ui/settings_dialog.py`:
- SettingsDialog class
- `open_settings()` function

**Step 8: Update imports.** `ui/settings_dialog.py` will need to import the extracted widgets:
```python
from .widgets import FlowLayout, AnimatedToggleButton, SectionGroup, ...
from .theme_widgets import ThemeCardWidget, FontCardWidget, ...
from .color_picker import ModernColorPickerDialog
from .icon_picker import IconPickerDialog
from .search import SettingsSearchPage
from .donation import DonationDialog
from .. import config
from ..constants import COLOR_LABELS, ICON_DEFAULTS, ...
from ..themes import THEMES
from ..fonts import FONTS, get_all_fonts
```

**Step 9: Replace root `settings.py`** with a facade:
```python
# settings.py — backward-compat facade
from .ui.settings_dialog import SettingsDialog, open_settings
from .ui.widgets import *
from .ui.theme_widgets import *
from .ui.color_picker import *
from .ui.icon_picker import *
from .ui.search import *
from .ui.donation import *
from .ui.helpers import *
```

Commit: `refactor: split settings.py into ui/ submodules`

---

### Phase 5: Create CLAUDE.md

Create `CLAUDE.md` in the repo root with project context:

```markdown
# Kaizen — Anki Add-on

Fork of [Onigiri](https://github.com/thepeacemonk/Onigiri). Replaces Anki's default deck browser with a modern customizable dashboard.

## Architecture

- `__init__.py` — Entry point. Registers Anki hooks, injects CSS/JS.
- `config.py` — Config get/set/defaults. Central to everything.
- `constants.py` / `themes.py` / `fonts.py` — Static data.
- `templates.py` — HTML template strings for sidebar and layout.
- `core/` — Non-UI logic: renderer, deck tree, heatmap, sidebar API.
- `patches/` — Anki monkeypatching: CSS generation, background rendering, deck node rendering, webview message handling. Split from the original `patcher.py`.
- `ui/` — Qt dialogs and widgets: settings, color picker, icon picker, etc. Split from the original `settings.py` plus standalone dialog files.
- `web/` — CSS, JS, HTML served to Anki's webview (menu.css, engine.js, injector.js, heatmap.js).
- `system_files/` — Static assets: icons, fonts, images.
- `_archived/` — Disabled gamification system (preserved for possible future use).

## Key conventions

- Facade files at root (patcher.py, settings.py, etc.) re-export from subpackages for backward compatibility. Do not add new code to facades.
- All config keys use "kaizen_" prefix.
- CSS class names use "kaizen-" prefix.
- JS globals use "Kaizen" prefix (KaizenEngine, KaizenHeatmap).
- GPL-3.0 license. Original Onigiri attribution required.

## Testing

No test suite. Test by installing in Anki add-ons folder and verifying:
1. Deck browser renders (sidebar + main content)
2. Settings dialog opens
3. Heatmap renders
4. External add-ons don't crash (TempContent.tree fix)

## Working style

- Minimal targeted changes over rewrites
- Bottom-to-top when editing large files (line numbers don't shift)
- Grep before editing to find current line numbers
- One commit per logical change
```

Commit: `docs: add CLAUDE.md project context`

---

## Validation Checklist

After all phases, verify:

- [ ] `python -c "import ast; [ast.parse(open(f).read()) for f in __import__('glob').glob('**/*.py', recursive=True)]"` — no syntax errors
- [ ] All facade files at root still work: `from Kaizen import patcher; print(dir(patcher))`
- [ ] No remaining imports referencing old file names without facades
- [ ] `grep -rn "from \. import\|from \.\." --include="*.py" | grep -v __pycache__` — all imports valid
- [ ] Install in Anki, deck browser loads
- [ ] Settings dialog opens
- [ ] Sidebar collapse/expand works

## Risk Notes

- **Circular imports** are the #1 risk. The biggest danger: `core/renderer.py` imports `patcher` and `patcher` imports `kaizen_renderer`. After the move, this becomes `core/renderer.py` importing from `patches/` and `patches/` importing from `core/`. Resolve by using lazy imports (import inside functions) where circular refs exist.
- **`addon_path`** is used everywhere as `os.path.dirname(__file__)`. After moving files into subdirectories, `__file__` points deeper. Each submodule that needs the add-on root should use: `ADDON_PATH = os.path.dirname(os.path.dirname(__file__))`
- **Anki's module system** uses the folder name as the package. Our folder is `Kaizen` (capital K). All internal imports use relative imports (`from . import X`) so the folder name doesn't matter for those, but `mw.addonManager.addonFromModule()` returns the folder name.