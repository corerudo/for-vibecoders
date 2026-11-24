# Logging in exteraGram Plugins

Modern logging for exteraGram plugins using `log_utils`. Every plugin has access to `self.logger`, writing directly to Android logcat with automatic tagging, log levels, origin locations, tracebacks, and structured object dumps.

---

## The Rule

> [!TIP]
> Always use `self.logger` (or `get_logger()`) instead of `self.log()` or `print()`.  
> `self.log()` is legacy: it always logs at debug priority under a generic tag that cannot be easily filtered.

```python
from base_plugin import BasePlugin

class MyPlugin(BasePlugin):
    def on_plugin_load(self):
        self.logger.info("Plugin %s v%s loaded", self.name, self.version)
        self.logger.debug("Settings: %s", self.export_settings())
```

---

## Logcat Tag and Filtering

Every plugin automatically logs under the tag:
```text
plugin.<__id__>
```

Filter your plugin's logs via ADB:
```bash
adb logcat -s plugin.my_plugin
```

Every entry automatically includes the source file, method, and line:
```text
I plugin.my_plugin: [my_plugin.py:on_plugin_load:6] Plugin My Plugin v1.0 loaded
```

---

## Log Levels

| Level | Method | Purpose |
|---|---|---|
| `LogLevel.VERBOSE` | `logger.verbose(...)` | Highly detailed tracing (e.g. inside high-frequency hooks) |
| `LogLevel.DEBUG` | `logger.debug(...)` | Debugging info needed during development |
| `LogLevel.INFO` | `logger.info(...)` | Normal lifecycle milestones (load, unload, user action) |
| `LogLevel.WARNING` | `logger.warning(...)` | Recoverable or unexpected issues |
| `LogLevel.ERROR` | `logger.error(...)`, `logger.exception(...)` | Failures and exceptions |

By default:
- **Developer Mode ON**: Shows `DEBUG` and above.
- **Developer Mode OFF**: Shows `INFO` and above (zero overhead for regular users).

Change the level programmatically:
```python
from log_utils import LogLevel

self.logger.level = LogLevel.VERBOSE   # Show everything
self.logger.level = LogLevel.OFF       # Silence logger
self.logger.level = None               # Revert to default
```

---

## Formatting: Use `%s` Arguments, Not F-Strings

In high-frequency hooks, avoid formatting f-strings before checking log level:

```python
# GOOD: String formatting is only performed IF debug level is active
self.logger.debug("User %s sent update %s", user_id, update_name)

# BAD: F-string is constructed on every single call regardless of log level
self.logger.debug(f"User {user_id} sent update {update_name}")
```

---

## Exceptions and Tracebacks

Use `logger.exception()` inside `except` blocks to automatically capture and format tracebacks:

```python
try:
    self.process_incoming(update)
except Exception:
    self.logger.exception("Failed to process update %s", update_name)
```

Outside of `except`, pass `exc_info=err`:
```python
self.logger.error("Operation failed", exc_info=err)
```

---

## Object Dumps (`dump`)

Inspect complex Java/Python objects, TL messages, or dictionaries:

```python
# Dump to logcat
self.logger.dump(message.messageOwner, label="MessageOwner")

# Or render to string
from log_utils import dump

text = dump(my_dict, depth=2, max_items=20)
```

---

## Execution Timers (`timed`)

Measure the wall-clock time of operations:

```python
with self.logger.timed("parse_feed"):
    self.parse_large_feed()
# Logs: [timing] parse_feed finished in 14.2ms
```

---

## Child Loggers

For modular plugins (e.g. Elyx), create sub-loggers:
```python
network_logger = self.logger.child("network")
# Logs under tag: plugin.my_plugin.network
```
