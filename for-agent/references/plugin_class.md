# Plugin Class

Plugin metadata, lifecycle hooks, menu items, logger, multi-account handling, and request/update interception.

## Metadata

Plugin metadata must be defined as plain top-level values. Do not build them dynamically — the loader parses them from the source file with AST.

```python
__id__ = "better_previews"
__name__ = "Better Previews"
__description__ = "Modifies specific URLs for better previews"
__author__ = "@immat0x1"
__version__ = "1.0.0"
__icon__ = "exteraPlugins/1"
__category__ = ["chats", "tools"]
__app_version__ = ">=12.5.1"
__sdk_version__ = ">=1.4.4.3"
__requirements__ = ["mpmath", "tinydb"]
```

Supported metadata keys:
- `__id__` (required): 2–32 characters, starts with a letter, latin letters, digits, `_`, or `-`. Matches filename `<__id__>.py`.
- `__name__` (required): Human-readable plugin name.
- `__description__`: Summary of what the plugin does (supports basic markdown in UI).
- `__author__`: Author name or Telegram handle (`@yourUsername`).
- `__version__`: Version string (defaults to `"1.0"` if omitted).
- `__icon__`: Format `StickerPackShortName/index`, e.g. `"exteraPlugins/1"`.
- `__category__`: Plugin catalog category. Use a single string or a list of up to three strings:
  - `"appearance"`: Themes, Monet, interface tweaks, icons, and layout changes.
  - `"chats"`: Messages, chat lists, folders, and conversation features.
  - `"media"`: Photos, videos, music, voice messages, and stickers.
  - `"privacy"`: Privacy, ghost mode, security, and hiding information.
  - `"tools"`: Productivity helpers, automation, and utilities.
  - `"integrations"`: AI, translators, and external services.
  - `"fun"`: Games, jokes, and entertainment.
  - `"libraries"`: Shared code that other plugins depend on.
- `__app_version__`: Target app version requirement (e.g. `">=12.5.1"`). Supports `>=`, `<=`, `==`, `>`, `<`.
- `__sdk_version__`: Required minimum plugin SDK version (e.g. `">=1.4.4.3"`).
- `__requirements__`: List of pure-Python PyPI packages installed automatically (see `pip_dependencies.md`).

Legacy compatibility: `__min_version__` is still accepted, treated as `__app_version__ = ">=..."`.

**SDK Baseline:** Targets the current `1.4.4.3+` / `1.4.6.x` API surface and expects `exteraGram 12.5.1+` (with intents on `12.6.4+`).

## Basic Structure

Every plugin file should define one class inheriting from `BasePlugin`.

```python
from base_plugin import BasePlugin

__id__ = "weather"
__name__ = "Weather"
__category__ = "tools"
__app_version__ = ">=12.5.1"
__sdk_version__ = ">=1.4.4.3"


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

### Load and Unload

```python
from base_plugin import BasePlugin


class DebugPlugin(BasePlugin):
    def on_plugin_load(self):
        # Register hooks, listeners, intents, or background tasks
        self.logger.info("Plugin loaded!")

    def on_plugin_unload(self):
        # Cleanup that is not handled automatically
        self.logger.info("Plugin unloaded!")
```

- `on_plugin_load` runs when the plugin is enabled or restored during app startup.
- `on_plugin_unload` runs when the plugin is disabled or the app is shutting down.

### Application Events

```python
from base_plugin import AppEvent, BasePlugin


class DebugPlugin(BasePlugin):
    def on_app_event(self, event_type: AppEvent):
        if event_type == AppEvent.START:
            self.logger.info("App is starting")
        elif event_type == AppEvent.STOP:
            self.logger.info("App is stopping")
        elif event_type == AppEvent.PAUSE:
            self.logger.info("App moved to background")
        elif event_type == AppEvent.RESUME:
            self.logger.info("App returned to foreground")
```

`AppEvent` values: `START`, `STOP`, `PAUSE`, `RESUME`.

## Logging

`self.logger` writes to Android logcat under the tag `plugin.<id>` with levels, origin trace, and object dumps:

```python
self.logger.debug("Hook installed for %s", class_name)

try:
    self.process(update)
except Exception:
    self.logger.exception("Failed to process update")

self.logger.dump(message.messageOwner)
```

Filter logs:
```bash
adb logcat -s plugin.<id>
```

See `logging.md` for complete details.

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

Add menu entries to app menus through `MenuItemData`.

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
        self.logger.info("Message menu item clicked. Context keys: %s", list(context.keys()))

        message = context.get("message")
        if message:
            self.logger.info("Clicked message ID: %s, text: %s", message.getId(), message.messageText)

    def handle_profile_click(self, context: Dict[str, Any]):
        self.logger.info("Profile menu item clicked. Context keys: %s", list(context.keys()))

        user = context.get("user")
        if user:
            self.logger.info("Profile opened for: %s (ID: %s)", user.first_name, user.id)
```

### `MenuItemData` Fields
- `menu_type: MenuItemType` — required target menu.
- `text: str` — required visible title.
- `on_click: Callable[[Dict[str, Any]], None]` — required click handler.
- `on_long_click: Optional[Callable[[Dict[str, Any]], None]]` — optional long click handler.
- `item_id: Optional[str]` — optional stable id if you want to remove the item later.
- `icon: Optional[str]` — drawable name, e.g. `"msg_info"`.
- `subtext: Optional[str]` — secondary line below the title.
- `condition: Optional[str]` — MVEL expression controlling visibility.
- `priority: int` — larger values appear earlier.

### `MenuItemType`
- `MenuItemType.MESSAGE_CONTEXT_MENU` — context menu on long-pressing a message.
- `MenuItemType.DRAWER_MENU` — side navigation drawer.
- `MenuItemType.MAIN_MENU` — main three-dot menu.
- `MenuItemType.CHAT_ACTION_MENU` — action bar menu inside a chat.
- `MenuItemType.PROFILE_ACTION_MENU` — action menu on user/channel profile.

### Removing Menu Items
Removed automatically on plugin unload, or manually:
```python
if self.message_item_id:
    self.remove_menu_item(self.message_item_id)
```

## Event Hooks & Multi-Account Scoping

Event hooks let a plugin intercept Telegram requests, responses, updates, or outgoing message parameters.

Register with:
- `self.add_hook(name, match_substring=False, priority=0)`
- `self.add_on_send_message_hook(priority=0)`
- `self.add_intent_hook(info, type)` (see `intents.md`)
- `self.add_file_hook(file_info)`

> [!IMPORTANT]
> **Multi-Account Rule:** Every hook receives `account: int`. Because background accounts remain active, always use `self.client(account)` (or pass `account=account`) when performing actions or sending messages in response to a hook. See `multi_account.md`.

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
        self.add_on_send_message_hook()

    def pre_request_hook(self, request_name: str, account: int, request: Any) -> HookResult:
        if request_name in TYPING_REQUESTS and self.get_setting("dont_send_typing", True):
            self.logger.debug("Blocking request: %s for account %s", request_name, account)
            return HookResult(strategy=HookStrategy.CANCEL)

        if request_name == "TL_account_updateStatus" and self.get_setting("force_offline", True):
            request.offline = True
            return HookResult(strategy=HookStrategy.MODIFY, request=request)

        return HookResult()

    def post_request_hook(self, request_name: str, account: int, response: Any, error: Any) -> HookResult:
        if request_name == "TL_messages_sendMessage" and not error:
            self.logger.debug("Message successfully sent on account %s", account)
        return HookResult()

    def create_settings(self) -> List[Any]:
        return [
            Switch(key="dont_send_typing", text="Don't send typing status", default=True),
            Switch(key="force_offline", text="Always appear offline", default=True),
        ]
```

### Hook Strategies
- `HookStrategy.DEFAULT` — leave unchanged (no-op).
- `HookStrategy.CANCEL` — cancel the operation entirely.
- `HookStrategy.MODIFY` — use the modified object.
- `HookStrategy.MODIFY_FINAL` — modify and stop further plugins from processing this event.

### Update Hooks
```python
def on_update_hook(self, update_name: str, account: int, update: Any) -> HookResult:
    result = HookResult()

    if update_name == "TL_updateNewMessage":
        client = self.client(account)
        self.logger.debug("Received new message update on account %s", account)

    return result


def on_updates_hook(self, container_name: str, account: int, updates: Any) -> HookResult:
    return HookResult()
```

### Outgoing Message Hook
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

## Low-Level Java Method Hooking
For hooking Java methods and constructors directly via bytecode/Xposed hooking, see `xposed_hooking.md`.
