# Setup & Common Telegram Source Classes

## Setup

### Install a recent app build

Use a recent build of exteraGram with plugin support enabled; treat `exteraGram 12.8.1+` as the practical minimum for this SDK tree. Test builds are usually published in the beta channel (`t.me/exteraGramCI`).

### Enable the plugin engine

`exteraGram Preferences` > `Plugins` → enable the plugin engine. Then open the plugin info screen and enable developer mode.

### Create a local project

A simple layout is enough:

```
my-plugin/
`-- hello_world.py
```

Desktop-development tips:
- Keep the filename stable; the least surprising setup is `filename == __id__ + ".py"`.
- Point your IDE at this repository's `src/` directory for imports and autocomplete.
- Keep metadata as plain top-level constants so the loader can parse them.

Minimal starter file:

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

### Connect to the device

Connect the phone to the PC and make sure ADB is on `PATH`. Two common workflows:
- Copy the `.py` file into the app's plugins directory and reload it manually.
- Use custom tooling that talks to the built-in `DevServer` for write/reload/debug flows.

On-device plugin files live under a path similar to:

```
/data/user/0/com.exteragram.messenger/files/plugins/<plugin_id>.py
```

**Debugging ports**: the built-in development server listens on `42690` by default; the debugger port is separate and chosen by you/your tooling (example below uses `5678`).

VS Code remote attach example:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python Debugger: Attach exteraGram plugin",
            "type": "debugpy",
            "request": "attach",
            "connect": {
                "host": "localhost",
                "port": 5678
            },
            "pathMappings": [
                {
                    "localRoot": "/path/to/your/workspace",
                    "remoteRoot": "/data/user/0/com.exteragram.messenger/files/plugins"
                }
            ]
        }
    ]
}
```

Inside that remote directory, the plugin file should still match the plugin id, e.g. `hello_world.py`.

### First pages to keep open

Setup → First Plugin (`first_plugin.md`) → Plugin Class (`plugin_class.md`) → Plugin Settings (`plugin_settings.md`) → Client Utilities (`client_utils.md`).

### Main modules at a glance

- `base_plugin` — metadata, lifecycle, event hooks, menu items, Xposed helpers
- `android_utils` — UI thread helpers, listener wrappers, logging, clipboard
- `client_utils` — requests, controllers, queues, sending/editing messages
- `file_utils` — directories plus simple file I/O helpers
- `hook_utils` — reflection helpers for classes and private fields
- `ui.*` — dialogs, bulletins, settings
- `extera_utils.*` — class proxy DSL, text formatting helpers, metadata utilities

---

## Common Telegram Source Classes

It's recommended to have a local copy of Telegram sources open in Android Studio when doing deep hooking work.

| Class | Path | Notes |
|---|---|---|
| `LaunchActivity` | `TMessagesProj/src/main/java/org/telegram/ui/LaunchActivity.java` | App initialization happens here; custom links are handled here too. |
| `ProfileActivity` | `TMessagesProj/src/main/java/org/telegram/ui/ProfileActivity.java` | User and channel profile fragment. |
| `ChatActivity` | `TMessagesProj/src/main/java/org/telegram/ui/ChatActivity.java` | Chat rendering and functionality. |
| `MessageObject` | `TMessagesProj/src/main/java/org/telegram/messenger/MessageObject.java` | Wrapper for `TLRPC.Message`. |
| `ChatMessageCell` | `TMessagesProj/src/main/java/org/telegram/ui/Cells/ChatMessageCell.java` | Handles message rendering. |
| `AndroidUtilities` | `TMessagesProj/src/main/java/org/telegram/messenger/AndroidUtilities.java` | Contains many useful utilities. |
| `MessagesController` | `TMessagesProj/src/main/java/org/telegram/messenger/MessagesController.java` | Methods related to managing app state and Telegram requests. |
| `MessagesStorage` | `TMessagesProj/src/main/java/org/telegram/messenger/MessagesStorage.java` | Manages local database state; `database` field for custom SQLite queries. |
| `SendMessagesHelper` | `TMessagesProj/src/main/java/org/telegram/messenger/SendMessagesHelper.java` | Methods for sending all kinds of messages, including files. |
| `BulletinFactory` | `TMessagesProj/src/main/java/org/telegram/ui/Components/BulletinFactory.java` | Small notification messages shown at the bottom. |
| `AlertDialog` | `TMessagesProj/src/main/java/org/telegram/ui/ActionBar/AlertDialog.java` | Alert dialogs shown on top of the current fragment; support custom action-button handlers. |
| `TLRPC` (all Telegram request models) | `TMessagesProj/src/main/java/org/telegram/tgnet/TLRPC.java` and `TMessagesProj/src/main/java/org/telegram/tgnet/tl/` | Human-readable schema reference (not always up to date): https://corefork.telegram.org/schema |

GitHub source root: https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/
