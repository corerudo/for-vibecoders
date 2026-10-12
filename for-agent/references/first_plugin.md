# First Plugin

A realistic first plugin: when the user sends `.hello Alice`, the plugin rewrites the outgoing message, with a configurable greeting template in plugin settings.

## Minimal plugin shape

Every plugin file should contain metadata as plain top-level constants and one class inheriting from `BasePlugin`.

```python
from base_plugin import BasePlugin

__id__ = "hello_world"
__name__ = "Hello World"
__description__ = "My first exteraGram plugin"
__author__ = "Your Name"
__version__ = "1.0.0"
__icon__ = "exteraPlugins/1"
__app_version__ = ">=12.8.1"
__sdk_version__ = ">=1.4.3.10"


class HelloWorldPlugin(BasePlugin):
    pass
```

## Complete first plugin

```python
from typing import Any, List

from base_plugin import BasePlugin, HookResult, HookStrategy
from ui.settings import Header, Input, Text

__id__ = "hello_world"
__name__ = "Hello World"
__description__ = "Rewrites .hello commands into a friendly message"
__author__ = "Your Name"
__version__ = "1.0.0"
__icon__ = "exteraPlugins/1"
__app_version__ = ">=12.8.1"
__sdk_version__ = ">=1.4.3.10"


DEFAULT_TEMPLATE = "Hello, {name}!"


class HelloWorldPlugin(BasePlugin):
    def on_plugin_load(self):
        # Register the outgoing-message hook once when the plugin is loaded.
        self.add_on_send_message_hook()
        self.log("Hello World plugin loaded")

    def create_settings(self) -> List[Any]:
        return [
            Header(text="Hello World"),
            Input(
                key="template",
                text="Greeting template",
                default=DEFAULT_TEMPLATE,
                subtext="Use {name} where the entered name should appear.",
                icon="msg_edit",
            ),
            Text(
                text="Example",
                subtext=".hello Alice -> Hello, Alice!",
                icon="msg_info",
            ),
        ]

    def on_send_message_hook(self, account: int, params: Any) -> HookResult:
        # Some sends are media-only, so always validate that we really have text.
        if not isinstance(getattr(params, "message", None), str):
            return HookResult()

        raw_text = params.message.strip()
        if not raw_text.startswith(".hello"):
            return HookResult()

        # Split only once so ".hello Alice Smith" keeps the full name.
        parts = raw_text.split(" ", 1)
        name = parts[1].strip() if len(parts) > 1 else ""

        if not name:
            params.message = "Usage: .hello <name>"
            return HookResult(strategy=HookStrategy.MODIFY, params=params)

        # Read the saved template or fall back to the default one.
        template = self.get_setting("template", DEFAULT_TEMPLATE)

        try:
            params.message = template.format(name=name)
        except Exception:
            # If the user saved a broken template, fail softly and explain why.
            params.message = "Template must contain valid Python format placeholders, for example {name}."

        # Tell the plugin engine that we changed outgoing message params.
        return HookResult(strategy=HookStrategy.MODIFY, params=params)
```

## What this example teaches

1. **Metadata is parsed statically.** The loader reads metadata from the Python file with AST, so values such as `__id__` and `__version__` should stay as plain top-level constants.
2. **Hooks must be registered.** Implementing `on_send_message_hook(...)` is not enough by itself — you also need `self.add_on_send_message_hook()` inside `on_plugin_load`.
3. **Settings are regular Python dataclasses**, built from objects in `ui.settings`. `Header` creates a section title, `Input` persists a string under a key, `Text` is a read-only helper row.
4. **`HookResult` decides what happens next.** For outgoing messages: `HookResult()` = leave unchanged; `HookResult(strategy=HookStrategy.MODIFY, params=params)` = use updated params; `HookResult(strategy=HookStrategy.CANCEL)` = stop the send entirely.

## Testing it

1. Enable the plugin.
2. Open the plugin settings and change the template if desired.
3. Send `.hello Alice` in any chat → expect `Hello, Alice!`.
4. If the template is changed to `Welcome back, {name}.`, then `.hello Alice` becomes `Welcome back, Alice.`.

## Common beginner mistakes

- Forgetting `self.add_on_send_message_hook()` in `on_plugin_load`.
- Building metadata dynamically instead of using plain constants.
- Assuming `params.message` always exists (it doesn't for media-only sends).
- Doing slow network or file work directly in a hook on the UI thread — use `run_on_queue(...)` instead (see `client_utils.md`).
