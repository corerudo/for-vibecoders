# Client Utilities

Helpers for queues, requests, multi-account scoping, notifications, controllers, fragments, and sending/editing messages (module: `client_utils`).

## Multi-Account Scoping & `AccountClient`

Every action in `client_utils` supports account scoping. Background accounts remain active in exteraGram and will trigger your hooks. If you do not pass `account=...`, methods default to the UI-selected account (and emit a warning if called during a background hook).

### 1. `self.client(account)` / `get_client(account)`

Returns an `AccountClient` bound to a specific account:

```python
# Inside BasePlugin:
client = self.client(account)

# Outside BasePlugin:
from client_utils import get_client
client = get_client(account)

# All methods are pre-scoped to that account:
client.send_text(peer_id, "Hello!")
me = client.get_user_config().getCurrentUser()
ctrl = client.get_messages_controller()
```

### 2. Explicit `account=` Parameter

All standalone functions accept `account: int | None = None`:

```python
from client_utils import send_text, get_messages_controller

send_text(peer_id, "Hello", account=account)
ctrl = get_messages_controller(account=account)
```

### 3. Context & Account Scope

- `get_selected_account() -> int`: Returns the account currently visible in the app UI.
- `get_hook_account() -> int | None`: Returns the account of the hook currently executing on the thread.
- `account_scope(account)`: Context manager scoping all calls within its block to `account`.

---

## Queues (Background Threads)

Use `run_on_queue(...)` for work that should not block the UI thread: network requests, file I/O, heavy parsing, long computations.

```python
import time
from log_utils import get_logger
from client_utils import run_on_queue

logger = get_logger("my_plugin")

def my_long_task(parameter: str):
    logger.info("Task started with: %s", parameter)
    time.sleep(5)
    logger.info("Task finished for: %s", parameter)

# Run on the default PLUGINS_QUEUE
run_on_queue(lambda: my_long_task("some_data"))
```

Target a specific queue and add a delay in milliseconds:

```python
from client_utils import GLOBAL_QUEUE, run_on_queue

run_on_queue(lambda: my_long_task("other_data"), GLOBAL_QUEUE, 2500)
```

Queue-name constants:
```python
STAGE_QUEUE = "stageQueue"
GLOBAL_QUEUE = "globalQueue"
CACHE_CLEAR_QUEUE = "cacheClearQueue"
SEARCH_QUEUE = "searchQueue"
PHONE_BOOK_QUEUE = "phoneBookQueue"
THEME_QUEUE = "themeQueue"
EXTERNAL_NETWORK_QUEUE = "externalNetworkQueue"
PLUGINS_QUEUE = "pluginsQueue"
```

For the raw Java `DispatchQueue`, use `get_queue_by_name(...)`:
```python
from client_utils import PLUGINS_QUEUE, get_queue_by_name

plugins_dispatch_queue = get_queue_by_name(PLUGINS_QUEUE)
```

---

## Sending Telegram API Requests

`send_request(request, fn, *, account=None)` sends a raw Telegram `TLObject` through `ConnectionsManager`. Pass a normal Python callback — the SDK wraps it into a `RequestDelegate`.

```python
from java.lang import Integer
from org.telegram.tgnet import TLRPC
from client_utils import send_request

def handle_read_contents_response(response, error):
    if error:
        self.logger.error("Error reading message contents: %s", error.text)
        return

    if response and isinstance(response, TLRPC.TL_messages_affectedMessages):
        self.logger.info("PTS: %s, Count: %s", response.pts, response.pts_count)

req = TLRPC.TL_messages_readMessageContents()
req.id.add(Integer(12345))

request_id = send_request(req, handle_read_contents_response, account=account)
```

---

## Sending Messages and Media

The `send_*` helpers build `SendMessageParams`, prepare media objects when needed, and send on the proper thread. All accept optional `parse_mode="HTML"` or `parse_mode="Markdown"`, and `account=account`.

### `send_text`
```python
from client_utils import send_text

peer_id = 123456789

send_text(peer_id, "Hello from my plugin!", account=account)
send_text(peer_id, "This is a reply.", replyToMsg=message_object, account=account)
send_text(peer_id, "**Hello** from my *plugin*!", parse_mode="Markdown", account=account)
```

### `send_photo`
```python
from client_utils import send_photo

send_photo(peer_id, photo_path, caption="Here is a photo!", account=account)
send_photo(peer_id, photo_path, caption="<b>High quality</b> photo.", high_quality=True, parse_mode="HTML", account=account)
```

### `send_document`
```python
from client_utils import send_document

send_document(peer_id, file_path, caption="Here is the zip file.", account=account)
```

### `send_video`
```python
from client_utils import send_video

# SDK fills common video attributes (duration, size)
send_video(peer_id, video_path, caption="Check out this video!", account=account)
```

### `send_audio`
```python
from client_utils import send_audio

send_audio(peer_id, audio_path, caption="Listen to this!", account=account)
```

### Low-level `send_message`
Accepts a dictionary mapping onto `SendMessagesHelper.SendMessageParams`:

```python
from client_utils import send_message

send_message({
    "peer": 123456789,
    "message": "Scheduled message from plugin",
    "scheduleDate": 1735689600,
    "notify": False,
}, account=account)
```

Supported keys: `peer`, `message`, `caption`, `location`, `photo`, `user`, `document`, `videoEditedInfo`, `game`, `poll`, `pollSendParams`, `pollIndex`, `todo`, `invoice`, `mediaWebPage`, `cover`, `path`, `replyToMsg`, `replyToTopMsg`, `webPage`, `searchLinks`, `retryMessageObject`, `entities`, `replyMarkup`, `params`, `notify`, `scheduleDate`, `scheduleRepeatPeriod`, `ttl`, `parentObject`, `sendAnimationData`, `updateStickersOrder`, `effect_id`, `hasMediaSpoilers`, `replyToStoryItem`, `sendingStory`, `replyQuote`, `invert_media`, `sendingHighQuality`, `stars`, `payStars`, `monoForumPeer`, `quick_reply_shortcut`, `quick_reply_shortcut_id`, `suggestionParams`, `isLivePhoto`, `livePhotoTimestamp`, `dice_stake`, `inputRichMessage`.

`replyToMsg` and `replyToTopMsg` must be `org.telegram.messenger.MessageObject` instances.

---

## Editing Messages

`edit_message(...)` edits an existing `MessageObject` — text, replacement media file, spoiler state:

```python
from client_utils import edit_message

edit_message(message_obj, text="This is the new, edited text.", account=account)
edit_message(message_obj, text="This is <b>bold</b> text.", parse_mode="HTML", account=account)

new_photo_path = "/path/to/another/image.jpg"
edit_message(message_obj, file_path=new_photo_path, text="Here is a new photo instead.", account=account)
edit_message(message_obj, file_path=new_photo_path, with_spoiler=True, account=account)
```

---

## Fragments, Controllers, and Account Helpers

All controller getters accept optional `account`:

```python
from client_utils import (
    get_last_fragment,
    get_account_instance,
    get_messages_controller,
    get_contacts_controller,
    get_media_data_controller,
    get_connections_manager,
    get_location_controller,
    get_notifications_controller,
    get_messages_storage,
    get_send_messages_helper,
    get_file_loader,
    get_secret_chat_helper,
    get_download_controller,
    get_notifications_settings,
    get_notification_center,
    get_media_controller,
    get_user_config,
)

fragment = get_last_fragment()
account_instance = get_account_instance(account)
messages_controller = get_messages_controller(account)
connections_manager = get_connections_manager(account)
send_helper = get_send_messages_helper(account)
user_cfg = get_user_config(account)

if user_cfg.getCurrentUser():
    current_user = user_cfg.getCurrentUser()
```

---

## `NotificationCenterDelegate`

Pre-wrapped Python base class implementing `NotificationCenter.NotificationCenterDelegate`:

```python
from java.lang import Object
from java import jarray
from client_utils import NotificationCenterDelegate, get_notification_center
from org.telegram.messenger import NotificationCenter

class MyDelegate(NotificationCenterDelegate):
    def didReceivedNotification(self, id: int, account: int, args: jarray(Object)):
        if id == NotificationCenter.didUpdateConnectionState:
            print(f"Connection state changed for account {account}:", args)

delegate = MyDelegate()
notification_center = get_notification_center(account)
notification_center.addObserver(delegate, NotificationCenter.didUpdateConnectionState)
```

---

## Displaying Bulletins

```python
from ui.bulletin import BulletinHelper

BulletinHelper.show_info("This is an informational message.")
```

Full bulletin documentation: see `ui_dialogs_bulletins.md`.
