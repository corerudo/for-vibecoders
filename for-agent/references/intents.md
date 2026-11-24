# Intents & Deep Links

Register global `before` and `after` handlers for Telegram deep links, t.me URLs, and Android intents.

> [!NOTE]
> `IntentsManager` requires **exteraGram 12.6.4+ (build 66999+)** and SDK **1.4.4.3+**.

---

## Overview

The `intents` module enables plugins to intercept and react to links handled by exteraGram before or after the app executes its default intent handler.

Handlers come in two stages:
- **`before` handlers**: Executed before the original handler. Can inspect parameters or stop/divert navigation.
- **`after` handlers**: Executed after the original handler completes.

---

## Import

```python
from intents import IntentsManager as IM
```

Or register through `BasePlugin`:
```python
from base_plugin import BasePlugin, IntentHookType
from intents import IntentsManager
```

---

## Registering Global Handlers

### 1. `before` Handler

```python
from intents import IntentsManager as IM

def on_chat_link(id):
    print("Chat id opened:", id)

# Register handler for tg://chat?id=...
handle = IM.new_global_before_handler(
    IM.HandlerInfo(
        callback=on_chat_link,
        scheme="tg",
        host="chat",
        required_path_args_names=["id"],
    )
)
```

Alternatively, `IntentsManager.new_global_before_handler` accepts keyword arguments for filters directly.

### 2. `after` Handler

```python
from intents import IntentsManager as IM

def on_after_link(intent=None, scheme=None, host=None):
    print(f"Handled link: {scheme}://{host}")

handle = IM.new_global_after_handler(
    IM.HandlerInfo(
        callback=on_after_link,
        scheme="https",
        host="t.me",
    )
)
```

---

## Registering via `BasePlugin`

When working inside a `BasePlugin` subclass, use `self.add_intent_hook`:

```python
from base_plugin import BasePlugin, IntentHookType
from intents import IntentsManager as IM

class MyLinkPlugin(BasePlugin):
    def on_plugin_load(self):
        info = IM.HandlerInfo(
            callback=self.on_link_clicked,
            scheme="https",
            host="t.me",
            priority=10
        )
        self.handler_id = self.add_intent_hook(info, IntentHookType.BEFORE)

    def on_link_clicked(self, url=None, **kwargs):
        self.logger.info("Intercepted t.me link: %s", url)

    def on_plugin_unload(self):
        if hasattr(self, "handler_id") and self.handler_id:
            self.remove_intent_hook(self.handler_id)
```

---

## Filters

Both `new_global_before_handler` and `new_global_after_handler` support granular matching criteria:

| Filter | Type | Description | Example |
|---|---|---|---|
| `scheme` | `str` | Matches `intent.data.scheme` | `"tg"`, `"https"` |
| `host` | `str` | Matches `intent.data.host` | `"t.me"`, `"resolve"`, `"join"` |
| `path` | `str` | Matches path or regex | `"/c/.*"` |
| `required_path_args_names` | `List[str]` | Requires specific URL query parameter names | `["id"]`, `["token"]` |
| `action` | `str` | Intent action string | `"android.intent.action.VIEW"` |
| `whitelist_flags` | `List[int]` | Required intent flags | `[Intent.FLAG_ACTIVITY_NEW_TASK]` |
| `blacklist_flags` | `List[int]` | Forbidden intent flags | |
| `type` | `str` | MIME type string | `"text/plain"` |
| `categories` | `List[str]` | Intent category names | `["android.intent.category.BROWSABLE"]` |
| `priority` | `int` | Execution order (higher numbers run earlier) | `10` |

---

## URL Parsing Helper

Parse Telegram link parameters with `IM.parse`:

```python
from intents import IntentsManager as IM

params = IM.parse("tg://resolve?domain=exteragram&post=123")
# params -> {'domain': 'exteragram', 'post': '123'}
```

---

## Unregistering Handlers

```python
# Via handle object:
handle.unhandle()

# Or via handler_id:
IM.unhandle(handle.handler_id)
```
