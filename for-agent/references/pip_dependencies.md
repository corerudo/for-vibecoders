# PIP & Dependencies

Using external Python libraries in plugins via the built-in PIP-like package manager.

## Declaring Dependencies

List libraries in `__requirements__`, a list of strings in standard PEP 508 requirement format.

```python
__id__ = "inline_calculator"
__name__ = "Inline Calculator"
__requirements__ = ["mpmath", "tinydb"]

from base_plugin import BasePlugin, HookResult, HookStrategy
from mpmath import mp
from tinydb import TinyDB

class CalculatorPlugin(BasePlugin):
    ...
```

When a user installs the plugin, exteraGram automatically downloads these libraries and their dependencies.

## How it works

1. **Resolution**: fetches package metadata from PyPI to find a version satisfying constraints.
2. **Pure-Python only**: only downloads "pure-Python" wheels (files ending in `-none-any.whl`).
3. **Isolation**: libraries install into a shared `libs` directory; if multiple plugins need the same version, it's downloaded once.
4. **Automatic cleanup**: uninstalling a plugin removes dependencies no longer needed by any other plugin.

## Limitations

### 1. No binary extensions

The most significant limitation. Libraries with C/C++ extensions (numpy, pandas, scipy, cryptography, opencv, etc.) **cannot be installed** — they must be compiled for the specific Android architecture and linked against the app's Python runtime.

- Allowed: packages that say "Universal" or "none-any" in their wheel filename.
- Forbidden: packages requiring a specific platform (manylinux, macosx, win_amd64) or with compiled components.

### 2. Dependency conflicts

All plugins share the same set of installed libraries (unlike a virtualenv). If Plugin A requires `requests==2.25.1` and Plugin B requires `requests==2.31.0`, a conflict occurs. The system tries a version satisfying both; if none is active/available, installing the second plugin fails.

### 3. Network access

Installation requires internet access to reach `pypi.org` and download files.

### 4. Limited marker support

Basic environment markers (`python_version`, `sys_platform`) are supported; complex markers or those referring to `extra` might not work as expected.

## Pre-installed Libraries

Some libraries are pre-installed for performance/stability — no need to list in `__requirements__` (doing so is harmless):

- `beautifulsoup4` — HTML/XML parsing (web scraping)
- `debugpy` — official Python remote debugger (used by the Dev Server)
- `lxml` — XML/HTML processing
- `packaging` — core package utilities
- `pillow` — PIL fork for image manipulation
- `requests` — HTTP library
- `PyYAML` — YAML parser/emitter

Python runtime version: `3.11`.

## Troubleshooting

- **"No pure-Python wheel found"** — the library contains compiled code; cannot be used unless manually added to the core app.
- **"Dependency conflict"** — another installed plugin uses an incompatible version of the same library.
- **"Package not found"** — check spelling / PyPI availability.
