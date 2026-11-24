# Elyx: Structured & Multi-File Plugins

Elyx is the modern, structured plugin format for **exteraGram** and **AyuGram**. It maintains the familiar `BasePlugin` API but removes the limitation of packing entire projects into a single `.py` file.

An Elyx plugin archive has an `.elyx` or `.eaf` extension (ZIP-compatible) and supports multiple Python modules, separate asset/image files, standalone localization files (`.yml`), bundled wheel dependencies, and live reload during desktop development.

---

## When to Use Elyx vs Single-file Plugin

- **Single-file (`.py` / `.plugin`)**: Best for simple, self-contained plugins (quick commands, simple hooks, 1–2 settings) with no external assets.
- **Elyx (`.elyx` / `.eaf`)**: Recommended when:
  - The project grows into multiple modules or packages.
  - UI needs images, SVGs, Lottie animations, or external config files.
  - Text should be translated into multiple languages without hardcoding strings in Python.
  - The plugin bundles local pure-Python wheels (`libs/`).
  - You develop using desktop tooling with instant hot-reload on device.

---

## Mental Model & Archive Layout

```text
my_plugin.elyx
|-- refmap.yml            # Entrypoint & path mapping
|-- metainfo.yml          # Plugin metadata (id, name, version, requirements, etc.)
|-- main.py               # Plugin class entrypoint
|-- src/                  # Additional Python modules
|   |-- __init__.py
|   `-- helpers.py
|-- assets/               # Resource files (images, icons, json)
|   `-- logo.svg
`-- strings/              # Localization files
    |-- strings_en.yml
    `-- strings_ru.yml
```

### Entrypoint Mapping (`refmap.yml`)

The runtime reads `refmap.yml` at the archive root to locate components:

```yaml
main: "main.py"
metainfo: "metainfo.yml"
assets: "assets/"
strings: "strings/"
src: "src/"
libs: "libs/"
```

### Metadata (`metainfo.yml`)

Instead of top-level Python AST constants, Elyx plugins declare metadata cleanly in YAML:

```yaml
id: "better_previews"
name: "Better Previews"
description: "Modifies URLs and embeds richer previews"
author: "@your_name"
version: "1.0.0"
icon: "exteraPlugins/1"
category: "tools"
app_version: ">=12.5.1"
sdk_version: ">=1.4.4.3"
requirements:
  - "mpmath"
  - "tinydb"
```

---

## The Public `elyx` API

Plugin code has one stable public import surface:

```python
from elyx import assets, metainfo, refmap, settings, strings
```

> [!IMPORTANT]
> Always import from `elyx`. Never import from internal runtime modules (`elyxcore._*`), as they are internal implementation details subject to change.

### 1. Localization (`strings`)

Declare translations in YAML files under `strings/`:

`strings/strings_en.yml`:
```yaml
greeting: "Hello, {name}!"
settings_title: "Plugin Settings"
notifications_enabled: "Notifications enabled"
```

`strings/strings_ru.yml`:
```yaml
greeting: "Привет, {name}!"
settings_title: "Настройки плагина"
notifications_enabled: "Уведомления включены"
```

Using in Python:
```python
from elyx import strings

# Automatically picks translation matching app language:
text = strings.get("greeting", name="Alice")
# Or attribute style:
title = strings.settings_title
```

### 2. Assets (`assets`)

Load bundled images, icons, and files:

```python
from elyx import assets

# Get asset wrapper
logo_asset = assets.get("logo.svg")

# Available conversions:
drawable = logo_asset.as_drawable()        # Android Drawable
bitmap = logo_asset.as_bitmap()            # Android Bitmap
bytes_data = logo_asset.as_bytes()         # Raw bytes
stream = logo_asset.as_stream()            # Java InputStream
file_obj = logo_asset.as_file()            # Python file-like object
```

### 3. Settings Controller (`settings`)

Access plugin-scoped settings without needing a `self` reference:

```python
from elyx import settings

is_enabled = settings.get("enabled", default=True)
settings.set("enabled", False, reload_settings=True)
```

### 4. Android Interop Callbacks

Elyx exports pre-wrapped Java functional interface adapters:
```python
from elyx import OnClickListener, Runnable, Callback, Callback2, Callback3
```

---

## Main Plugin File Structure

The entrypoint file (e.g. `main.py`) inherits from `BasePlugin` exactly like single-file plugins:

```python
from base_plugin import BasePlugin, HookResult
from elyx import strings, settings, assets
from src.helpers import process_chat_message

class MyElyxPlugin(BasePlugin):
    def on_plugin_load(self):
        self.logger.info("Loaded %s", strings.settings_title)
        self.add_on_send_message_hook()

    def on_send_message_hook(self, account: int, params) -> HookResult:
        return process_chat_message(self, account, params)
```

---

## ElyxBuilder CLI Tool

The official CLI tool `ElyxBuilder` manages Elyx projects:

```bash
# Install / upgrade
pip install --upgrade ElyxBuilder

# Create new project interactively
elyb new

# Create new project with flags
elyb new -g -n "My Plugin" -a "@developer"

# Build .elyx archive
elyb build

# Build compiled archive (optimized byte-code)
elyb build -c

# Connect to phone and watch for changes (hot reload)
elyb dev
```

### Hot Reload with `elyb dev`
When connected via ADB:
```bash
adb forward tcp:42690 tcp:42690
elyb dev
```
Any file change in your project directory is packaged and pushed over the DevServer connection to exteraGram instantly, triggering a reload without app restart.
