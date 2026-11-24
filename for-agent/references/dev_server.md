# Development Server & Remote Debugging

Connect desktop tools to exteraGram via ADB to push plugin updates, trigger reloads, and debug Python code live.

---

## Overview

The exteraGram Plugin SDK includes an internal TCP/IPC server (`dev_server`) designed for local developer tooling. It allows:
- Querying installed plugins and their state.
- Live reloading plugins without restarting exteraGram.
- Installing, removing, and pushing `.py` and `.elyx` plugins.
- Remote debugging via Microsoft's `debugpy` or PyCharm.

> [!CAUTION]
> The DevServer should only be active during development. It exposes endpoints that can install, replace, execute, and delete plugin code.

---

## Connecting via ADB

The server binds to the Android device loopback interface on port `42690`.

Forward the port to your PC:
```bash
adb devices
adb forward tcp:42690 tcp:42690
```

Desktop tools then connect to `127.0.0.1:42690`.

---

## Protocol & Message Format

Communication takes place over a raw TCP socket exchanging line-delimited UTF-8 JSON objects.

Every request must include:
- `@`: command name string
- `#`: client-generated request ID string
- Additional command-specific payload fields

### 1. Ping
Request:
```json
{"@": "ping", "#": "req-1"}
```
Response:
```json
{"#": "req-1", "pong": true}
```

### 2. Reload Plugin
Request:
```json
{"@": "reload_plugin", "#": "req-2", "id": "my_plugin"}
```
Response:
```json
{"#": "req-2", "success": true}
```

### 3. List Plugins
Request:
```json
{"@": "list_plugins", "#": "req-3"}
```
Response:
```json
{"#": "req-3", "plugins": [{"id": "my_plugin", "name": "My Plugin", "enabled": true, "version": "1.0.0"}]}
```

---

## Hot Reload Workflow with ElyxBuilder

If using Elyx:
```bash
adb forward tcp:42690 tcp:42690
elyb dev
```
`elyb dev` automatically connects to `127.0.0.1:42690`, watches your plugin directory for changes, builds deltas, and sends reload requests.

---

## Remote Debugging with `debugpy`

`debugpy` is pre-installed in the plugin environment. You can attach VS Code or Cursor to a running plugin:

```python
import debugpy

# Inside on_plugin_load:
try:
    debugpy.listen(("0.0.0.0", 5678))
    self.logger.info("Debugger listening on port 5678")
except Exception as e:
    self.logger.warning("debugpy listen failed: %s", e)
```

Forward the debugger port:
```bash
adb forward tcp:5678 tcp:5678
```

VS Code `launch.json`:
```json
{
  "name": "Attach to exteraGram",
  "type": "python",
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
```
