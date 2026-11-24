# Multi-Account Handling

How account scoping works in **exteraGram**, why hooks fire for accounts not currently visible on screen, and how to avoid sending messages or querying data from the wrong account.

---

## Why This Matters

exteraGram keeps all logged-in accounts connected simultaneously. Background accounts continue receiving updates, syncing chats, sending requests, and triggering your plugin's hooks.

By default, legacy functions in `client_utils` target **the account currently selected in the UI**. If a hook fires for Account 1 while the user is actively viewing Account 0, calling `send_text(...)` without specifying the account will send the message from **Account 0 (the wrong account!)**.

---

## The Core Rule

Every hook method provides the specific `account: int` that triggered the event:

```python
def on_update_hook(self, update_name: str, account: int, update: Any) -> HookResult:
    ...

def on_send_message_hook(self, account: int, params: Any) -> HookResult:
    ...

def pre_request_hook(self, request_name: str, account: int, request: Any) -> HookResult:
    ...
```

**Always tie your reaction to that `account`.**

---

## Pattern 1: `self.client(account)` (Recommended)

`self.client(account)` returns an `AccountClient` bound to that account. All client methods are pre-scoped:

```python
from base_plugin import BasePlugin, HookResult

class AutoReplyPlugin(BasePlugin):
    def on_plugin_load(self):
        self.add_hook("TL_updateNewMessage")

    def on_update_hook(self, update_name: str, account: int, update) -> HookResult:
        if update_name != "TL_updateNewMessage":
            return HookResult()

        # Bind to the account for this event
        client = self.client(account)

        me = client.get_user_config().getCurrentUser()
        dialog_id = update.message.dialog_id

        client.send_text(dialog_id, f"Auto-reply as {me.first_name}")
        return HookResult()
```

Outside of a `BasePlugin` class, use `get_client(account)` from `client_utils`:
```python
from client_utils import get_client

client = get_client(account)
client.send_text(peer, "Hello")
```

---

## Pattern 2: Explicit `account=` Parameter

Every helper in `client_utils` accepts an optional `account`:

```python
from client_utils import get_messages_controller, get_user_config, send_text

ctrl = get_messages_controller(account)
user_cfg = get_user_config(account)
send_text(peer, "Hello", account=account)
```

---

## Pattern 3: Scoped Context (`account_scope`)

Wrap multiple calls with `account_scope`:

```python
from client_utils import account_scope, send_text, get_messages_controller

with account_scope(account):
    ctrl = get_messages_controller() # uses account
    send_text(peer, "Message")       # uses account
```

---

## What Happens If You Forget?

The SDK monitors account context. If a helper is called without `account=` while handling an event for a different account, a warning is logged to logcat:

```text
W plugin.my_plugin: get_messages_controller() is using account 0 because it is the one selected in the UI,
but it was called while handling account 1. Pass account=1, or use get_client(1), if you meant that account.
```

---

## Explicitly Targeting the Selected Account

If you intentionally want to interact with the account currently displayed on the user's screen (e.g., when adding an item to the user's Saved Messages from a UI action):

```python
from client_utils import get_selected_account, send_text

send_text(peer, "Saved note", account=get_selected_account())
```
