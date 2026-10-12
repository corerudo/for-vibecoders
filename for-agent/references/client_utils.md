# Client Utilities

Helpers for queues, requests, notifications, controllers, fragments, and sending/editing messages (module: `client_utils`).

## Queues (background threads)

Use `run_on_queue(...)` for work that should not block the UI thread: network requests, file I/O, heavy parsing, long computations.

```python
import time

from android_utils import log
from client_utils import run_on_queue


def my_long_task(parameter: str):
    log(f"Task started with: {parameter}")
    time.sleep(5)
    log(f"Task finished for: {parameter}")


# Run on the default PLUGINS_QUEUE.
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
if plugins_dispatch_queue:
    pass  # call DispatchQueue methods directly
```

## Sending Telegram API requests

`send_request(request, fn)` sends a raw Telegram `TLObject` through the current account's `ConnectionsManager`. Pass a normal Python callback — the SDK wraps it into a `RequestDelegate`.

```python
from java.lang import Integer
from org.telegram.tgnet import TLRPC

from android_utils import log
from client_utils import send_request


def handle_read_contents_response(response, error):
    if error:
        log(f"Error reading message contents: {error.text}")
        return

    if response and isinstance(response, TLRPC.TL_messages_affectedMessages):
        log(f"Successfully read contents. PTS: {response.pts}, Count: {response.pts_count}")
    else:
        log(f"Unexpected response type: {type(response)}")


req = TLRPC.TL_messages_readMessageContents()
req.id.add(Integer(12345))

request_id = send_request(req, handle_read_contents_response)
log(f"Sent TL_messages_readMessageContents, request ID: {request_id}")
```

You usually do not need to instantiate `RequestCallback` manually unless you specifically want the proxy class itself.

## Sending Messages and Media

The `send_*` helpers build the proper `SendMessageParams`, prepare media objects when needed, and send on the correct thread. All accept optional `parse_mode="HTML"` or `parse_mode="Markdown"`.

### `send_text`

```python
from client_utils import send_text

peer_id = 123456789

send_text(peer_id, "Hello from my plugin!")
send_text(peer_id, "This is a reply.", replyToMsg=message_object)
send_text(peer_id, "**Hello** from my *plugin*!", parse_mode="Markdown")
```

### `send_photo`

```python
from client_utils import send_photo

send_photo(peer_id, photo_path, caption="Here is a photo!")
send_photo(peer_id, photo_path, caption="<b>High quality</b> photo.", high_quality=True, parse_mode="HTML")
```

### `send_document`

```python
from client_utils import send_document

send_document(peer_id, file_path, caption="Here is the zip file.")
```

### `send_video`

```python
from client_utils import send_video

# The SDK will fill common video attributes such as size and duration.
send_video(peer_id, video_path, caption="Check out this video!")
```

### `send_audio`

```python
from client_utils import send_audio

# The SDK will fill common audio metadata when possible.
send_audio(peer_id, audio_path, caption="Listen to this!")
```

### Low-level `send_message`

For more control, use `send_message(params, parse_mode=None)` — accepts a dict mapping onto `SendMessagesHelper.SendMessageParams`.

```python
from client_utils import send_message

send_message({
    "peer": 123456789,
    "message": "Scheduled message from plugin",
    "scheduleDate": 1735689600,
    "notify": False,
})
```

Supported keys include: `peer`, `message`, `caption`, `location`, `photo`, `user`, `document`, `videoEditedInfo`, `game`, `poll`, `pollSendParams`, `pollIndex`, `todo`, `invoice`, `mediaWebPage`, `cover`, `path`, `replyToMsg`, `replyToTopMsg`, `webPage`, `searchLinks`, `retryMessageObject`, `entities`, `replyMarkup`, `params`, `notify`, `scheduleDate`, `scheduleRepeatPeriod`, `ttl`, `parentObject`, `sendAnimationData`, `updateStickersOrder`, `effect_id`, `hasMediaSpoilers`, `replyToStoryItem`, `sendingStory`, `replyQuote`, `invert_media`, `sendingHighQuality`, `stars`, `payStars`, `monoForumPeer`, `quick_reply_shortcut`, `quick_reply_shortcut_id`, `suggestionParams`, `isLivePhoto`, `livePhotoTimestamp`, `dice_stake`, `inputRichMessage`.

`replyToMsg` and `replyToTopMsg` should be Telegram `MessageObject` instances, not raw message ids.

`parse_mode` is case-insensitive, currently supports only `html`/`markdown`; when provided the SDK parses `message`/`caption` and fills entities automatically.

## Editing Messages

`edit_message(...)` edits an existing `MessageObject` — text, replacement media file, spoiler state.

```python
from client_utils import edit_message

edit_message(message_obj, text="This is the new, edited text.")
edit_message(message_obj, text="This is <b>bold</b> text.", parse_mode="HTML")

new_photo_path = "/path/to/another/image.jpg"
edit_message(message_obj, file_path=new_photo_path, text="Here is a new photo instead.")
edit_message(message_obj, file_path=new_photo_path, with_spoiler=True)
```

Notes: `message_obj` must be a real `org.telegram.messenger.MessageObject`; pass `text=` to replace text/caption; pass `file_path=` to replace attached media; unsupported `parse_mode` raises `ValueError`.

## Fragments, Controllers, and Account Helpers

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
account_instance = get_account_instance()
messages_controller = get_messages_controller()
connections_manager = get_connections_manager()
send_helper = get_send_messages_helper()
user_cfg = get_user_config()

if user_cfg.getCurrentUser():
    current_user = user_cfg.getCurrentUser()
```

## `NotificationCenterDelegate`

A ready-made Python base class implementing `NotificationCenter.NotificationCenterDelegate`, for listening to client-side notifications.

```python
from java.lang import Object
from java import jarray

from client_utils import NotificationCenterDelegate, get_notification_center
from org.telegram.messenger import NotificationCenter


class MyDelegate(NotificationCenterDelegate):
    def didReceivedNotification(self, id: int, account: int, args: jarray(Object)):
        if id == NotificationCenter.didUpdateConnectionState:
            print("Connection state changed:", args)


delegate = MyDelegate()
notification_center = get_notification_center()
notification_center.addObserver(delegate, NotificationCenter.didUpdateConnectionState)

# Later, when no longer needed:
# notification_center.removeObserver(delegate, NotificationCenter.didUpdateConnectionState)
```

You do not need to wrap `NotificationCenterDelegate` with `dynamic_proxy(...)` yourself — subclassing is enough. If you need a generated Java subclass instead of a simple interface proxy, see `class_proxy.md`.

## Displaying Bulletins

```python
from ui.bulletin import BulletinHelper

BulletinHelper.show_info("This is an informational message.")
```

Full list of bulletin helpers: see `ui_dialogs_bulletins.md`.
