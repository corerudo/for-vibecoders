---
name: exteragram-plugin-sdk
description: Use this skill whenever writing, editing, debugging, or explaining exteraGram Telegram plugins (Python plugins run inside the exteraGram Android app via its plugin engine / Chaquopy). Trigger this for any .py file meant to be loaded as an exteraGram plugin, any mention of "exteraGram plugin", "BasePlugin", "userbot module" built on this SDK, hooking Telegram requests/updates/messages, building plugin settings screens, sending/editing Telegram messages from a plugin, Xposed-style Java method hooking from Python, or generating Java proxy classes from Python (class proxy / java_subclass). Always consult this skill before writing exteraGram plugin code, even for small or "quick" plugins — the metadata format, hook registration, and settings API have exact, easy-to-get-wrong requirements.
---

# exteraGram Plugin SDK

Reference for building Python plugins for the exteraGram Telegram client. Plugins run inside the app through a built-in plugin engine powered by Chaquopy (Python-Java interop) and support Xposed-style method hooking.

**Baseline for this SDK tree:** SDK version `1.4.3.10`, Python `3.11`, minimum app version `exteraGram 12.8.1+`.

## Before writing any plugin code

Read `references/plugin_class.md` and `references/first_plugin.md` first — they cover metadata rules and hook registration, which are the two most common sources of bugs. Then read whichever other reference file matches the task (settings UI, sending messages, low-level hooking, Java proxying).

## Core rules that are easy to get wrong

1. **Metadata must be plain top-level constants.** The loader parses metadata with AST, so `__id__`, `__name__`, etc. must be literal assignments at module level — never computed, never inside a function or class, never built dynamically.
   - `__id__` and `__name__` are the only required fields.
   - `__id__` must be 2–32 chars, start with a letter, and contain only latin letters, digits, `_`, or `-`.
   - Full field list: `__id__`, `__name__`, `__description__`, `__author__`, `__version__`, `__icon__`, `__app_version__`, `__sdk_version__`, `__requirements__`.
   - `__requirements__` is a list of pure-Python PyPI package names/specs (PEP 508). Packages with compiled/binary extensions (numpy, pandas, cryptography, etc.) are NOT supported — only `-none-any.whl` wheels install. See `references/pip_dependencies.md`.
   - The plugin filename on-device should match `__id__ + ".py"`.

2. **Every plugin file has exactly one class inheriting from `BasePlugin`** (from `base_plugin`).

3. **Hooks must be explicitly registered in `on_plugin_load`.** Just defining `on_send_message_hook(...)` etc. as a method does nothing by itself — you must call the matching `self.add_*` registration method (e.g. `self.add_on_send_message_hook()`, `self.add_hook("TL_...")`) inside `on_plugin_load`.

4. **`HookResult` decides the outcome of intercepted operations.** Always return a `HookResult`:
   - `HookResult()` — leave unchanged (default/no-op)
   - `HookResult(strategy=HookStrategy.MODIFY, params=params)` (or `request=`/`response=`/`update=`/`updates=` depending on the hook type) — use the modified object
   - `HookResult(strategy=HookStrategy.CANCEL)` — stop the operation entirely
   - `HookResult(strategy=HookStrategy.MODIFY_FINAL, ...)` — modify and stop further plugin processing for that event

5. **Always validate before assuming shape.** E.g. in `on_send_message_hook`, check `isinstance(getattr(params, "message", None), str)` before treating it as text — many sends are media-only.

6. **Never do slow/blocking work (network, file I/O, heavy compute) directly inside a hook on the UI thread.** Use `run_on_queue(...)` from `client_utils` (see `references/client_utils.md`).

7. **Settings screens** come from `create_settings(self) -> List[Any]`, returning a list of dataclasses from `ui.settings` (`Header`, `Switch`, `Selector`, `Input`, `Text`, `EditText`, `Divider`, `Custom`). Persist values with `self.get_setting(key, default)` / `self.set_setting(key, value, reload_settings=False)`. See `references/plugin_settings.md`.

## Reference files (read as needed)

- `references/first_plugin.md` — minimal walkthrough of a complete first plugin (`.hello` command example); good template to copy from.
- `references/plugin_class.md` — full metadata spec, plugin lifecycle (`on_plugin_load`/`on_plugin_unload`/`on_app_event`), settings-storage helpers, menu items (`MenuItemData`/`MenuItemType`), and all event hooks (request/response/update/outgoing-message) with `HookResult`/`HookStrategy` details.
- `references/plugin_settings.md` — full settings UI system: built-in controls, parameter table, custom rows via `Custom`/`SimpleSettingFactory`, when to escalate to Class Proxy.
- `references/client_utils.md` — queues (`run_on_queue`), sending raw Telegram requests (`send_request`), sending/editing messages and media (`send_text`/`send_photo`/`send_document`/`send_video`/`send_audio`/`send_message`/`edit_message`), controller/fragment accessors, `NotificationCenterDelegate`, bulletins.
- `references/android_utils.md` — UI-thread helpers (`run_on_ui_thread`), Java listener wrappers (`R`, `OnClickListener`, `OnLongClickListener`), `log`, `copy_to_clipboard`.
- `references/xposed_hooking.md` — Xposed-style Java method/constructor hooking: `MethodHook`, `MethodReplacement`, functional hooks, `HookFilter`, `param` object, full hook/unhook workflow with examples.
- `references/class_proxy.md` — generating real Java subclasses from Python (`extera_utils.classes`): `java_subclass`, `joverride`/`joverload`, `jmethod`, `jfield`, MVEL-backed methods, constructors, `JA`/`JGS`/`JIR`/`JS` Java object access modifiers, troubleshooting.
- `references/ui_dialogs_bulletins.md` — `AlertDialogBuilder` (dialogs) and `BulletinHelper` (bottom notifications).
- `references/file_utils.md` — standard Telegram directories and read/write/delete file helpers.
- `references/text_formatting.md` — `extera_utils.text_formatting.parse_text` for HTML/Markdown → plain text + entities.
- `references/hook_utils.md` — reflection helpers: `find_class`, `get_private_field`/`set_private_field`, static field variants.
- `references/pip_dependencies.md` — `__requirements__`, pure-Python-only limitation, dependency conflicts, pre-installed libraries (`requests`, `pillow`, `lxml`, `beautifulsoup4`, `PyYAML`, `packaging`, `debugpy`).
- `references/setup_and_common_classes.md` — local project setup, on-device plugin path, DevServer/debugpy remote-attach config, and a lookup table of commonly referenced Telegram Java source classes (ActionBar, ChatActivity, MessageObject, MessagesController, etc.) with GitHub links.

## Quick decision guide

- Need to react to the user typing a command / sending a message → outgoing-message hook (`on_send_message_hook` + `self.add_on_send_message_hook()`).
- Need to intercept a Telegram API call (request/response) or incoming update → `self.add_hook("TL_...")` + `pre_request_hook`/`post_request_hook`/`on_update_hook`/`on_updates_hook`.
- Need a configurable option → add to `create_settings()`, read with `self.get_setting(...)`.
- Need to call/override a Java method that isn't exposed as a plugin hook → Xposed hooking (`references/xposed_hooking.md`), and only use Class Proxy (`references/class_proxy.md`) if you need a real Java subclass (e.g. a custom View, or a Java API that requires subclassing rather than an interface).
- Need to send or edit a message/media → `client_utils` send/edit helpers.
- Need background/network/file work → `run_on_queue`, never inline in a hook.
