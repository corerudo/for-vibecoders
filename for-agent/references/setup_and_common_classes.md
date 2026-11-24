# Setup & Common Telegram Source Classes

## Setup

### Install a compatible app build

Use a build of exteraGram or AyuGram with plugin support enabled; treat `exteraGram 12.5.1+` as the practical baseline for this SDK (with `intents` requiring `12.6.4+`). Test builds are published in the official beta channel (`t.me/exteraGramCI`).

### Enable the plugin engine

`exteraGram Preferences` > `Plugins` → enable the plugin engine. Then open the plugin info screen and toggle **Developer Mode** on.

### Create a local project

For single-file plugins:
```
my-plugin/
`-- hello_world.py
```

For multi-file structured Elyx plugins (see `elyx.md`):
```
my-plugin/
|-- refmap.yml
|-- metainfo.yml
|-- main.py
`-- src/
```

Desktop-development tips:
- Keep the filename matching `__id__ + ".py"`.
- Point your IDE's Python analysis paths to this repository's `stubs/` directory for full typing, autocompletion, and signatures.
- Keep metadata as plain top-level constants in `.py` (or in `metainfo.yml` for Elyx).

Minimal starter file:

```python
from base_plugin import BasePlugin

__id__ = "hello_world"
__name__ = "Hello World"
__description__ = "My first exteraGram plugin"
__author__ = "Your Name"
__version__ = "1.0.0"
__icon__ = "exteraPlugins/1"
__category__ = "tools"
__app_version__ = ">=12.5.1"
__sdk_version__ = ">=1.4.4.3"


class HelloWorldPlugin(BasePlugin):
    pass
```

### Connect to the device & DevServer

Connect the phone to the PC via USB and ensure ADB is working:
```bash
adb devices
adb forward tcp:42690 tcp:42690
```

With `elyb dev` (see `elyx.md`) or custom tools talking to `DevServer` (see `dev_server.md`), you can push code and hot-reload plugins instantly without restarting the app.

On-device plugin files live under:
```
/data/user/0/com.exteragram.messenger/files/plugins/<plugin_id>.py
```

### Remote debugging with `debugpy`

Forward the debugger port:
```bash
adb forward tcp:5678 tcp:5678
```

VS Code `launch.json`:
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
                    "localRoot": "${workspaceFolder}",
                    "remoteRoot": "/data/user/0/com.exteragram.messenger/files/plugins"
                }
            ]
        }
    ]
}
```

### Main modules at a glance

- `base_plugin` — metadata, lifecycle, event hooks, menu items, Xposed helpers, partial hooks (`hook_call_site`, `hook_field_read`, `hook_field_write`, etc.).
- `client_utils` — requests, multi-account helpers (`AccountClient`, `self.client(account)`), queues, sending/editing messages.
- `intents` — `IntentsManager` for handling custom schemes, `t.me` links, and Android intents.
- `log_utils` — `self.logger` (`Logger`), `LogLevel`, dumps, timers, logcat tags `plugin.<id>`.
- `dev_server` — internal development server on `127.0.0.1:42690` for reload and live synchronization.
- `elyx` (`elyxcore`) — multi-file plugin framework (`strings`, `assets`, `settings`, `refmap`).
- `android_utils` — UI thread helpers, listener wrappers, clipboard.
- `file_utils` — directories and file I/O helpers.
- `hook_utils` — reflection helpers for classes, methods, constructors, and fields.
- `ui.*` — dialogs, bulletins, settings UI.
- `extera_utils.*` — class proxy DSL, text formatting helpers, metadata utilities.

---

## Common Telegram Source Classes

| Class | Path | Notes |
|---|---|---|
| `LaunchActivity` | `TMessagesProj/src/main/java/org/telegram/ui/LaunchActivity.java` | App initialization; custom links and intent dispatching. |
| `ProfileActivity` | `TMessagesProj/src/main/java/org/telegram/ui/ProfileActivity.java` | User and channel profile fragment. |
| `ChatActivity` | `TMessagesProj/src/main/java/org/telegram/ui/ChatActivity.java` | Chat rendering and functionality. |
| `MessageObject` | `TMessagesProj/src/main/java/org/telegram/messenger/MessageObject.java` | High-level wrapper for `TLRPC.Message`. |
| `ChatMessageCell` | `TMessagesProj/src/main/java/org/telegram/ui/Cells/ChatMessageCell.java` | Handles message bubble rendering. |
| `AndroidUtilities` | `TMessagesProj/src/main/java/org/telegram/messenger/AndroidUtilities.java` | Core utilities (dp conversion, screen dimensions, formatters). |
| `MessagesController` | `TMessagesProj/src/main/java/org/telegram/messenger/MessagesController.java` | Managing chats, users, dialogs, and caching. |
| `MessagesStorage` | `TMessagesProj/src/main/java/org/telegram/messenger/MessagesStorage.java` | Manages local SQLite database state (`getDatabase()`). |
| `SendMessagesHelper` | `TMessagesProj/src/main/java/org/telegram/messenger/SendMessagesHelper.java` | Methods for sending messages and media. |
| `BulletinFactory` | `TMessagesProj/src/main/java/org/telegram/ui/Components/BulletinFactory.java` | Bottom notification popups. |
| `AlertDialog` | `TMessagesProj/src/main/java/org/telegram/ui/ActionBar/AlertDialog.java` | Alert dialogs shown over current fragment. |
| `TLRPC` | `TMessagesProj/src/main/java/org/telegram/tgnet/TLRPC.java` | Schema types, requests, and updates: https://corefork.telegram.org/schema |

GitHub source root: https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/
