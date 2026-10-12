# Android Utilities

Helpers for UI-thread work, Android listeners, logging, clipboard access, and Java `Runnable` wrappers (module: `android_utils`).

Most-used for: running code on the UI thread, attaching simple click listeners, logging/debugging.

## Wrappers for Java Interfaces

### `R` (Runnable Proxy)

`R(fn)` creates a real Java `Runnable` backed by a Python callable. Useful when some Android/Telegram API expects a `java.lang.Runnable` instance (e.g. `view.post(...)`, delayed callbacks).

```python
from android_utils import R, log


def my_task():
    log("Runnable task executed.")


runnable_instance = R(my_task)

some_view = ...
some_view.post(runnable_instance)
```

`run_on_ui_thread(...)` already wraps your callable internally — use `R(...)` only when some other API explicitly requires a `Runnable` object.

### `OnClickListener`

```python
from android_utils import OnClickListener, log
from android.view import View


def handle_button_click(view: View):
    log(f"Button {view.getId()} was clicked.")


button = ...
button.setOnClickListener(OnClickListener(handle_button_click))
```

The callable receives the clicked `View` as its only argument.

### `OnLongClickListener`

```python
from android_utils import OnLongClickListener, log
from android.view import View


def handle_button_long_click(view: View):
    log(f"Button {view.getId()} was long-clicked.")
    return True


button = ...
button.setOnLongClickListener(OnLongClickListener(handle_button_long_click))

# Or with a lambda:
button.setOnLongClickListener(OnLongClickListener(lambda v: (log("Long click!"), True)[1]))
```

The wrapped callable should return a bool: `True` consumes the long-click, `False` lets Android continue processing it.

## Utility Functions

### `run_on_ui_thread`

`run_on_ui_thread(func, delay=0)` schedules a Python callable on the Android UI thread. Any UI mutation (text changes, view updates, dialogs, adapters) should happen there.

```python
from android_utils import run_on_ui_thread


def update_ui_content():
    text_view = ...
    text_view.setText("Updated from Python on UI thread")


run_on_ui_thread(update_ui_content)          # as soon as possible
run_on_ui_thread(update_ui_content, 500)     # with a 500ms delay
```

Parameters: `func` — Python callable; `delay` — optional delay in milliseconds.

### `log`

`log(data)` sends data into the app's logging pipeline. Simple values (`str`, `int`, `float`, `bool`, `None`) are logged as text; complex objects go through the Java-side object inspector, useful for debugging Telegram/Android objects.

```python
from android_utils import log

log("This is a simple log message.")
log(f"Current account: {2}")
log(True)

log(user_object)
log(some_list)

try:
    x = 1 / 0
except Exception as e:
    log(f"An error occurred: {e}")
    import traceback
    log(traceback.format_exc())
```

### `copy_to_clipboard`

`copy_to_clipboard(text)` copies a string to the Android clipboard, showing the standard "copied" bulletin on success.

```python
from android_utils import copy_to_clipboard

copy_to_clipboard("https://t.me/exteraPlugins")
```

Handy for deeplinks, debug info, exported IDs, or user-facing share actions.
