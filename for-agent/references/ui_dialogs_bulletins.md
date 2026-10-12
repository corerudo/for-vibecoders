# UI: Alert Dialogs & Bulletins

## Alert Dialog Builder

A Pythonic wrapper (`ui.alert.AlertDialogBuilder`) for creating/managing Telegram-style `AlertDialog`s. Wraps `org.telegram.ui.ActionBar.AlertDialog.Builder`.

### Basic usage

```python
from ui.alert import AlertDialogBuilder
from client_utils import get_last_fragment
from android_utils import log

current_fragment = get_last_fragment()
if not current_fragment:
    log("Cannot show dialog, no current fragment.")

activity = current_fragment.getParentActivity()
if not activity:
    log("Cannot show dialog, no parent activity.")

builder = AlertDialogBuilder(activity)  # Default is ALERT_TYPE_MESSAGE
builder.set_title("My Plugin Alert")
builder.set_message("This is an important message from the plugin.")


def on_positive_click(bld: AlertDialogBuilder, which: int):
    log("Positive button clicked!")
    bld.dismiss()


def on_negative_click(bld: AlertDialogBuilder, which: int):
    log("Negative button clicked!")
    bld.dismiss()


builder.set_positive_button("OK", on_positive_click)
builder.set_negative_button("Cancel", on_negative_click)

builder.create()  # optional, useful to tweak flags before show()
builder.set_cancelable(True)
builder.show()
```

### Dialog types

Controlled by `progress_style` in the constructor:
- `AlertDialogBuilder.ALERT_TYPE_MESSAGE` (default) — standard message dialog
- `AlertDialogBuilder.ALERT_TYPE_LOADING` — determinate horizontal progress bar (`builder.set_progress(value)`)
- `AlertDialogBuilder.ALERT_TYPE_SPINNER` — indeterminate spinner, for loading states

```python
loading_builder = AlertDialogBuilder(activity, AlertDialogBuilder.ALERT_TYPE_SPINNER)
loading_builder.set_title("Loading Data...")
loading_builder.set_message("Please wait while data is being fetched.")
loading_builder.show()
loading_builder.set_cancelable(False)  # best called after create() or show()

# loading_builder.dismiss()
```

### Key methods

**Initialization**: `AlertDialogBuilder(context, progress_style=ALERT_TYPE_MESSAGE, resources_provider=None)`

**Content**: `set_title(title)`, `set_message(message)`, `set_message_text_view_clickable(clickable)`, `set_view(view, height=-2)`, `set_items(items, listener=None, icons=None)` (listener gets `(builder, index)`)

**Buttons**: `set_positive_button(text, listener=None)`, `set_negative_button(text, listener=None)`, `set_neutral_button(text, listener=None)` — listeners receive `(AlertDialogBuilder, button_id)` where button ids are `AlertDialogBuilder.BUTTON_POSITIVE` etc. `make_button_red(button_type)` styles a button red (e.g. `AlertDialogBuilder.BUTTON_NEGATIVE`).

**Listeners**: `set_on_back_button_listener(listener=None)`, `set_on_dismiss_listener(listener=None)`, `set_on_cancel_listener(listener=None)`

**Appearance/Behavior**: `set_top_image(res_id, background_color)`, `set_top_drawable(drawable, background_color)`, `set_top_animation(res_id, size, auto_repeat, background_color, layer_colors=None)`, `set_dim_enabled(enabled)`, `set_dialog_button_color_key(theme_key)`, `set_blurred_background(blur, blur_behind_if_possible=True)`, `set_cancelable(cancelable)`, `set_canceled_on_touch_outside(cancel)` (best called after create()/show())

**Lifecycle**: `create() -> AlertDialogBuilder`, `show() -> AlertDialogBuilder`, `dismiss()`, `get_dialog() -> Optional[AlertDialog]`, `get_button(button_type) -> Optional[View]` (call after create()/show())

**Progress**: `set_progress(progress)` for `ALERT_TYPE_LOADING` dialogs (0-100)

### Example: dialog with items

```python
from ui.alert import AlertDialogBuilder
from client_utils import get_last_fragment
from android_utils import log


def on_item_click(bld: AlertDialogBuilder, which: int):
    items_list = ["Option A", "Option B", "Option C"]
    log(f"Item '{items_list[which]}' (index {which}) selected.")
    bld.dismiss()


item_builder = AlertDialogBuilder(activity)
item_builder.set_title("Choose an Option")
item_builder.set_items(["Option A", "Option B", "Option C"], on_item_click)
item_builder.set_negative_button("Cancel", lambda b, w: b.dismiss())
item_builder.show()
```

### Important notes

- Always provide a valid Android `Context` (usually an Activity): `get_last_fragment().getParentActivity()` is the common way to get one.
- Listener callables receive the Python `AlertDialogBuilder` instance as the first argument, so you can call `bld.dismiss()` from within the callback.
- Dialog manipulation should generally happen on the Android UI thread — use `android_utils.run_on_ui_thread` if acting from a background thread.
- Proxy listeners include basic try/except blocks that log errors in Python callbacks, preventing crashes.

---

## Bulletin Helper

`ui.bulletin.BulletinHelper` provides static methods to show Telegram's "Bulletin" bottom-of-screen notifications, which auto-dismiss. Wraps `org.telegram.ui.Components.BulletinFactory`.

If no fragment is passed explicitly, fallback order: the fragment you passed → current safe fragment from `LaunchActivity` → visible bulletin bottom sheet if active → global bulletin factory as last resort.

All `BulletinHelper.show_...` methods automatically run on the Android UI thread — no need to wrap in `run_on_ui_thread`.

### Basic usage

```python
from ui.bulletin import BulletinHelper
from client_utils import get_last_fragment
from org.telegram.messenger import R as R_tg

current_fragment = get_last_fragment()

BulletinHelper.show_info("This is some information.", current_fragment)
BulletinHelper.show_error("An error occurred processing your request.", current_fragment)
BulletinHelper.show_success("Action completed successfully!", current_fragment)
```

### Standard bulletins

- `show_info(message, fragment=None)` — default info icon
- `show_error(message, fragment=None)` — default error/alert icon
- `show_success(message, fragment=None)` — default success/check icon

### Custom simple bulletins

```python
BulletinHelper.show_simple("Processing...", R_tg.raw.timer, current_fragment)
```

`show_simple(text, icon_res_id, fragment=None)` — single-line bulletin with custom Lottie icon (`icon_res_id` is a Lottie animation resource, e.g. `R_tg.raw.some_animation`).

```python
BulletinHelper.show_two_line("Download Complete", "File saved to gallery.", R_tg.raw.ic_download_done, current_fragment)
```

`show_two_line(title, subtitle, icon_res_id, fragment=None)` — two-line bulletin.

### Bulletins with actions

```python
def open_settings_action():
    print("Settings button clicked!")


BulletinHelper.show_with_button(
    "Plugin settings updated.", R_tg.raw.info, "Configure", open_settings_action, current_fragment,
)
```

`show_with_button(text, icon_res_id, button_text, on_click, fragment=None, duration=DURATION_PROLONG)`.

```python
def perform_delete():
    print("Item permanently deleted.")


def undo_delete():
    print("Delete operation undone.")


BulletinHelper.show_undo(
    "Item moved to trash.", on_undo=undo_delete, on_action=perform_delete, fragment=current_fragment,
)
```

`show_undo(text, on_undo, on_action=None, subtitle=None, fragment=None)` — "Undo"-style bulletin; `on_action` fires after a delay if undo not pressed.

### Contextual bulletins (predefined)

- `show_copied_to_clipboard(message=None, fragment=None)`
- `show_link_copied(is_private_link_info=False, fragment=None)`
- `show_file_saved_to_gallery(is_video=False, amount=1, fragment=None)`
- `show_file_saved_to_downloads(file_type_enum_name="UNKNOWN", amount=1, fragment=None)` — `file_type_enum_name` is a string name from `BulletinFactory.FileType` (e.g. `"PHOTO_TO_DOWNLOADS"`, `"GIF"`, `"MUSIC"`)

```python
BulletinHelper.show_file_saved_to_downloads("MUSIC", amount=3, fragment=current_fragment)
```

### Durations

`BulletinHelper.DURATION_SHORT` (1500ms), `DURATION_LONG` (2750ms), `DURATION_PROLONG` (5000ms) — usable with methods like `show_with_button`.

### Finding Lottie animations (`R.raw...`)

Lottie animations for bulletin icons are raw resources in Telegram's codebase (`TMessagesProj/src/main/res/raw/`, e.g. `info.json`, `success.json`, `delete.json`), accessed via `org.telegram.messenger.R.raw.animation_name` (e.g. `R_tg.raw.info`).
