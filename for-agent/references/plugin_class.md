# Plugin Class

Plugin metadata, lifecycle hooks, menu items, and request/update interception.

## Metadata

Plugin metadata must be defined as plain top-level values. Do not build them dynamically — the loader parses them from the source file with AST.

```python
__id__ = "better_previews"
__name__ = "Better Previews"
__description__ = "Modifies specific URLs for better previews"
__author__ = "@immat0x1"
__version__ = "1.0.0"
__icon__ = "exteraPlugins/1"
__app_version__ = ">=12.8.1"
__sdk_version__ = ">=1.4.3.10"
__requirements__ = ["mpmath", "tinydb"]
```

Supported metadata keys: `__id__`, `__name__`, `__description__`, `__author__`, `__version__`, `__icon__`, `__app_version__`, `__sdk_version__`, `__requirements__`.

Legacy compatibility: `__min_version__` is still accepted, treated as `__app_version__ = ">=..."`.

Important details:
- `__id__` and `__name__` are the only truly required fields.
- `__id__` must be 2–32 characters, start with a letter, and contain only latin letters, digits, `_`, or `-`.
- `__requirements__` is a list of packages to install through PIP (pure-Python only — see `pip_dependencies.md`).
- `__description__` supports basic markdown rendering in the UI.
- `__author__` may be plain text or a Telegram-style handle such as `@yourUsername`.
- `__version__` defaults to `1.0` if omitted.
- `__icon__` uses the format `StickerPackShortName/index`, e.g. `exteraPlugins/1`.
- `__app_version__` supports operators `>=`, `<=`, `==`, `>`, `<`.
- `__sdk_version__` lets you require a minimum plugin SDK version.

This SDK targets the `1.4.3.10` API surface and expects `exteraGram 12.8.1+`.

## Basic Structure

Every plugin file should define one class inheriting from `BasePlugin`.

```python
from base_plugin import BasePlugin

__id__ = "weather"
__name__ = "Weather"


class WeatherPlugin(BasePlugin):
    pass
```

## Settings

To expose a settings screen, implement `create_settings(self) -> List[Any]`.

```python
from typing import Any, List

from base_plugin import BasePlugin
from ui.settings import Header, Switch


class MyPlugin(BasePlugin):
    def create_settings(self) -> List[Any]:
        return [
            Header(text="General"),
            Switch(key="enabled", text="Enable feature", default=True),
        ]
```

Full settings UI system: see `plugin_settings.md`.

## Plugin Events

### Load and unload

```python
from base_plugin import BasePlugin


class DebugPlugin(BasePlugin):
    def on_plugin_load(self):
        # Good place to register hooks, listeners, or background tasks.
        self.log("Plugin loaded!")

    def on_plugin_unload(self):
        # Good place for cleanup that is not handled automatically.
        self.log("Plugin unloaded!")
```

`on_plugin_load` runs when the plugin is enabled or restored during app startup. `on_plugin_unload` runs when the plugin is disabled or the app is shutting down.

### Application events

```python
from base_plugin import AppEvent, BasePlugin


class DebugPlugin(BasePlugin):
    def on_app_event(self, event_type: AppEvent):
        if event_type == AppEvent.START:
            self.log("App is starting")
        elif event_type == AppEvent.STOP:
            self.log("App is stopping")
        elif event_type == AppEvent.PAUSE:
            self.log("App moved to background")
        elif event_type == AppEvent.RESUME:
            self.log("App returned to foreground")
```

`AppEvent` values: `START`, `STOP`, `PAUSE`, `RESUME`.

## Settings Storage Helpers

`BasePlugin` includes helpers for persistent plugin settings:
- `self.get_setting(key, default=None)`
- `self.set_setting(key, value, reload_settings=False)`
- `self.export_settings()`
- `self.import_settings(settings, reload_settings=True)`

```python
current_value = self.get_setting("enabled", True)
self.set_setting("enabled", not current_value, reload_settings=True)

backup = self.export_settings()
self.import_settings(backup, reload_settings=False)
```

## Menu Items

Add menu entries to several app menus through `MenuItemData`.

```python
from typing import Any, Dict

from base_plugin import BasePlugin, MenuItemData, MenuItemType


class MyMenuPlugin(BasePlugin):
    def on_plugin_load(self):
        self.message_item_id = self.add_menu_item(
            MenuItemData(
                menu_type=MenuItemType.MESSAGE_CONTEXT_MENU,
                text="Log Message Info",
                on_click=self.handle_message_click,
                icon="msg_info",
                subtext="Debug helper",
                priority=10,
            )
        )

        self.profile_item_id = self.add_menu_item(
            MenuItemData(
                menu_type=MenuItemType.PROFILE_ACTION_MENU,
                text="Log User Info",
                on_click=self.handle_profile_click,
                icon="user_search",
            )
        )

    def handle_message_click(self, context: Dict[str, Any]):
        self.log(f"Message menu item clicked. Context keys: {list(context.keys())}")

        message = context.get("message")
        if message:
            self.log(f"Clicked on message ID: {message.getId()}")
            self.log(f"Message text: {message.messageText}")

    def handle_profile_click(self, context: Dict[str, Any]):
        self.log(f"Profile menu item clicked. Context keys: {list(context.keys())}")

        user = context.get("user")
        if user:
            self.log(f"Profile opened for user: {user.first_name} (ID: {user.id})")
```

### `MenuItemData` fields

- `menu_type: MenuItemType` — required target menu
- `text: str` — required visible title
- `on_click: Callable[[Dict[str, Any]], None]` — required click handler
- `item_id: Optional[str]` — optional stable id if you want to remove the item later
- `icon: Optional[str]` — drawable name, e.g. `"msg_info"`
- `subtext: Optional[str]` — secondary line below the title
- `condition: Optional[str]` — MVEL expression controlling visibility
- `priority: int` — larger values usually appear earlier

### `MenuItemType`

`MESSAGE_CONTEXT_MENU`, `DRAWER_MENU`, `CHAT_ACTION_MENU`, `PROFILE_ACTION_MENU`.

### The `on_click` context

Common keys (exact set depends on menu type/screen): `account`, `context`, `fragment`, `dialog_id`, `user`, `userId`, `userFull`, `chat`, `chatId`, `chatFull`, `encryptedChat`, `message`, `groupedMessages`, `botInfo`.

Inspect during development:

```python
def handle_click(self, context):
    self.log(f"Available keys: {list(context.keys())}")
```

### Removing menu items

Menu items are removed automatically when the plugin unloads, but you can remove one manually:

```python
if self.message_item_id:
    self.remove_menu_item(self.message_item_id)
```

## Event Hooks

Event hooks let a plugin intercept Telegram requests, responses, updates, or outgoing message parameters.

Register with:
- `self.add_hook(name, match_substring=False, priority=0)`
- `self.add_on_send_message_hook(priority=0)`

```python
class GhostModePlugin(BasePlugin):
    def on_plugin_load(self):
        self.add_hook("TL_messages_setTyping")
        self.add_hook("TL_messages_setEncryptedTyping")
        self.add_hook("TL_account_updateStatus")
        self.add_on_send_message_hook()
```

Hook results are returned as `HookResult` with a `HookStrategy`.

```python
from typing import Any, List

from base_plugin import BasePlugin, HookResult, HookStrategy
from ui.settings import Switch


TYPING_REQUESTS = ["TL_messages_setTyping", "TL_messages_setEncryptedTyping"]


class GhostModePlugin(BasePlugin):
    def on_plugin_load(self):
        for req_name in TYPING_REQUESTS:
            self.add_hook(req_name)

        self.add_hook("TL_account_updateStatus")

    def pre_request_hook(self, request_name: str, account: int, request: Any) -> HookResult:
        if request_name in TYPING_REQUESTS and self.get_setting("dont_send_typing", True):
            self.log(f"Blocking request: {request_name}")
            return HookResult(strategy=HookStrategy.CANCEL)

        if request_name == "TL_account_updateStatus" and self.get_setting("force_offline", True):
            request.offline = True
            return HookResult(strategy=HookStrategy.MODIFY, request=request)

        return HookResult()

    def post_request_hook(self, request_name: str, account: int, response: Any, error: Any) -> HookResult:
        if request_name == "TL_messages_sendMessage" and not error:
            self.log("Successfully sent a message!")
        return HookResult()

    def create_settings(self) -> List[Any]:
        return [
            Switch(key="dont_send_typing", text="Don't send typing status", default=True),
            Switch(key="force_offline", text="Always appear offline", default=True),
        ]
```

### Hook strategies

- `HookStrategy.DEFAULT` — do nothing special
- `HookStrategy.CANCEL` — stop the operation
- `HookStrategy.MODIFY` — return a modified object
- `HookStrategy.MODIFY_FINAL` — return a modified object and stop further plugin processing for that event

Depending on the hook type, the modified object should be assigned to `HookResult.request`, `.response`, `.update`, `.updates`, or `.params`.

### Update hooks

```python
def on_update_hook(self, update_name: str, account: int, update: Any) -> HookResult:
    result = HookResult()

    if update_name == "TL_updateNewMessage":
        self.log(f"Intercepted update: {update_name}")

    return result


def on_updates_hook(self, container_name: str, account: int, updates: Any) -> HookResult:
    result = HookResult()

    if container_name == "TL_updates":
        self.log(f"Received updates container: {container_name}")

    return result
```

### Outgoing message hook

```python
def on_send_message_hook(self, account: int, params: Any) -> HookResult:
    result = HookResult()

    if hasattr(params, "message") and isinstance(params.message, str):
        if params.message.startswith(".hello"):
            params.message = params.message.replace(".hello", "Hello from plugin")
            result.strategy = HookStrategy.MODIFY
            result.params = params

    return result
```

## Xposed Method Hooking

For low-level Java method hooking through LSPosed/Xposed-style APIs, see `xposed_hooking.md`.
