---
name: exteragram-plugin-sdk
description: Use this skill whenever writing, editing, debugging, or explaining exteraGram Telegram plugins (Python plugins run inside the exteraGram Android app via its plugin engine / Chaquopy). Trigger this for any .py or .elyx file meant to be loaded as an exteraGram plugin, any mention of "exteraGram plugin", "BasePlugin", "Elyx", "userbot module" built on this SDK, hooking Telegram requests/updates/messages, building plugin settings screens, sending/editing Telegram messages from a plugin, multi-account handling, Xposed-style Java method hooking from Python, partial instruction hooking, or generating Java proxy classes from Python (class proxy / java_subclass). Always consult this skill before writing exteraGram plugin code, even for small or "quick" plugins — the metadata format, hook registration, multi-account scoping, and settings API have exact, easy-to-get-wrong requirements.
---

# exteraGram Plugin SDK

Reference for building Python plugins for the **exteraGram** and **AyuGram** Telegram clients. Plugins run inside the app through a built-in plugin engine powered by Chaquopy (Python-Java interop) and support Xposed-style method hooking and instruction-level partial hooking.

**Baseline for this SDK tree:** SDK version `1.4.4.3+` / `1.4.6.4-stable` / `1.4.6.5-beta`, Python `3.11`, minimum app version `exteraGram 12.5.1+` (with intents requiring `12.6.4+`).

Official Type Stubs (`.pyi`) are available in the repository under `./stubs/` for complete IDE typing and autocompletion.

## Before writing any plugin code

Read `references/plugin_class.md` and `references/first_plugin.md` first — they cover metadata rules, hook registration, and logging. If building a multi-file plugin with assets or localization, read `references/elyx.md`. For sending messages or reacting to events across accounts, read `references/multi_account.md`.

## Core rules that are easy to get wrong

1. **Metadata must be plain top-level constants.** The loader parses metadata with AST, so `__id__`, `__name__`, etc. must be literal assignments at module level — never computed, never inside a function or class, never built dynamically.
   - `__id__` and `__name__` are the only required fields.
   - `__id__` must be 2–32 chars, start with a letter, and contain only latin letters, digits, `_`, or `-`.
   - Full field list: `__id__`, `__name__`, `__description__`, `__author__`, `__version__`, `__icon__`, `__category__`, `__app_version__`, `__sdk_version__`, `__requirements__`.
   - `__category__` values: `"appearance"`, `"chats"`, `"media"`, `"privacy"`, `"tools"`, `"integrations"`, `"fun"`, `"libraries"`.
   - `__requirements__` is a list of pure-Python PyPI package names/specs (PEP 508). Packages with compiled/binary extensions (numpy, pandas, cryptography, etc.) are NOT supported — only `-none-any.whl` wheels install. See `references/pip_dependencies.md`.
   - The plugin filename on-device should match `__id__ + ".py"`. (For multi-file plugins, use the Elyx `.elyx` / `.eaf` format, see `references/elyx.md`).

2. **Every plugin file has exactly one class inheriting from `BasePlugin`** (from `base_plugin`).

3. **Hooks must be explicitly registered in `on_plugin_load`.** Just defining `on_send_message_hook(...)` or `on_update_hook(...)` as a method does nothing by itself — you must call the matching registration method (e.g. `self.add_on_send_message_hook()`, `self.add_hook("TL_...")`, `self.add_intent_hook(...)`) inside `on_plugin_load`.

4. **Multi-Account Scoping is mandatory.** exteraGram keeps every logged-in account connected simultaneously, and background accounts fire hooks.
   - Every hook receives `account: int`.
   - When reacting to a hook, always bind with `client = self.client(account)` or pass `account=account` to `client_utils` functions (`send_text`, `get_messages_controller`, etc.).
   - Calling helpers without `account` uses the UI-selected account, which sends messages or mutates state from the WRONG account. See `references/multi_account.md`.

5. **`HookResult` decides the outcome of intercepted operations.** Always return a `HookResult`:
   - `HookResult()` — leave unchanged (default/no-op)
   - `HookResult(strategy=HookStrategy.MODIFY, params=params)` (or `request=`/`response=`/`update=`/`updates=` depending on the hook type) — use the modified object
   - `HookResult(strategy=HookStrategy.CANCEL)` — stop the operation entirely
   - `HookResult(strategy=HookStrategy.MODIFY_FINAL, ...)` — modify and stop further plugin processing for that event

6. **Use `self.logger` instead of `print()` or `self.log()`.** `self.logger` writes to Android logcat with level filtering, automatic tag `plugin.<id>`, origin line numbers, and formatted exception dumps. Filter with `adb logcat -s plugin.<id>`. See `references/logging.md`.

7. **Never do slow/blocking work (network, file I/O, heavy compute) directly inside a hook on the UI thread.** Use `run_on_queue(...)` from `client_utils` (see `references/client_utils.md`).

8. **Settings screens** come from `create_settings(self) -> List[Any]`, returning a list of dataclasses from `ui.settings` (`Header`, `Switch`, `Selector`, `Input`, `Text`, `EditText`, `Divider`, `Custom`). Persist values with `self.get_setting(key, default)` / `self.set_setting(key, value, reload_settings=False)`. See `references/plugin_settings.md`.

## Reference files (read as needed)

- `references/first_plugin.md` — minimal walkthrough of a complete first plugin (`.hello` command example); good template to copy from.
- `references/plugin_class.md` — full metadata spec (`__category__`), plugin lifecycle, settings helpers, menu items, event hooks, and hook strategies.
- `references/elyx.md` — structured multi-file plugins (`.elyx` / `.eaf`), `refmap.yml`, `metainfo.yml`, ElyxBuilder CLI, resources/assets, and localization (`strings`).
- `references/multi_account.md` — multi-account rules, background account events, `self.client(account)`, `get_client(account)`, and `account_scope`.
- `references/logging.md` — `self.logger` (`log_utils.Logger`), levels, `dump`, `timed`, exception handling, and logcat filtering.
- `references/intents.md` — `IntentsManager` (exteraGram 12.6.4+), link interception (`tg://`, `t.me`), URL parsing, global before/after handlers.
- `references/dev_server.md` — internal TCP development server (`127.0.0.1:42690`), ADB port forward, hot reload, and remote debugging with `debugpy`.
- `references/plugin_settings.md` — full settings UI system: built-in controls, parameter table, custom rows via `Custom`/`SimpleSettingFactory`, when to escalate to Class Proxy.
- `references/client_utils.md` — `AccountClient`, background queues (`run_on_queue`), sending raw Telegram requests (`send_request`), sending/editing messages and media, controller/fragment accessors, `NotificationCenterDelegate`, bulletins.
- `references/android_utils.md` — UI-thread helpers (`run_on_ui_thread`), Java listener wrappers (`R`, `OnClickListener`, `OnLongClickListener`), clipboard.
- `references/xposed_hooking.md` — Xposed-style Java method/constructor hooking (`MethodHook`, `MethodReplacement`, functional hooks, filters) and partial instruction-level hooks (`hook_call_site`, `hook_field_read`, `hook_field_write`, `hook_string_constant`, `hook_return_value`, `PartialChain`).
- `references/class_proxy.md` — generating real Java subclasses from Python (`extera_utils.classes`): `java_subclass`, `joverride`/`joverload`, `jmethod`, `jfield`, MVEL-backed methods, constructors, `JA`/`JGS`/`JIR`/`JS` Java object access modifiers, troubleshooting.
- `references/ui_dialogs_bulletins.md` — `AlertDialogBuilder` (dialogs) and `BulletinHelper` (bottom notifications).
- `references/file_utils.md` — standard Telegram directories and read/write/delete file helpers.
- `references/text_formatting.md` — `extera_utils.text_formatting.parse_text` for HTML/Markdown → plain text + entities.
- `references/hook_utils.md` — reflection helpers: `find_class`, `java_class`, `get_method`, `get_constructor`, `get_field`, `get_private_field`/`set_private_field`, static field variants.
- `references/pip_dependencies.md` — `__requirements__`, pure-Python-only limitation, dependency conflicts, pre-installed libraries (`requests`, `pillow`, `lxml`, `beautifulsoup4`, `PyYAML`, `packaging`, `debugpy`).
- `references/setup_and_common_classes.md` — local project setup, on-device plugin path, DevServer/debugpy remote-attach config, and lookup table of commonly referenced Telegram Java source classes.

## Quick decision guide

- Need to react to the user typing a command / sending a message → outgoing-message hook (`on_send_message_hook` + `self.add_on_send_message_hook()`).
- Need to intercept a Telegram API call or incoming update → `self.add_hook("TL_...")` + `pre_request_hook`/`post_request_hook`/`on_update_hook`/`on_updates_hook` (always bind to `account` using `self.client(account)`).
- Need to intercept Telegram links or deep links (`t.me`, `tg://`) → `references/intents.md` (`IntentsManager` or `self.add_intent_hook`).
- Need a configurable option → add to `create_settings()`, read with `self.get_setting(...)`.
- Project is growing into multiple files with images, translations, or config → use Elyx (`references/elyx.md`).
- Need to call/override a Java method that isn't exposed as a plugin hook → Xposed hooking (`references/xposed_hooking.md`).
- Need to intercept a single instruction (call-site, field access, string constant, return) inside a method → partial hooks (`hook_call_site`, `hook_string_constant`, etc. in `references/xposed_hooking.md`).
- Need to create a custom View or real Java subclass → Class Proxy (`references/class_proxy.md`).
- Need to send or edit a message/media → `client_utils` send/edit helpers with `account=account`.
- Need background/network/file work → `run_on_queue`, never inline in a hook.
