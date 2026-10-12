# Text Formatting

Parse HTML/Markdown formatted text and convert formatting entities to `TLRPC` objects for the Telegram API (module: `extera_utils.text_formatting`).

You generally don't need this module directly if using `client_utils.send_message` (or `send_text`, etc.) — those accept a `parse_mode` parameter. Use this module when you need to manually parse text for other purposes.

## `parse_text`

```python
from extera_utils.text_formatting import parse_text

def parse_text(text: str, parse_mode: Optional[str] = 'HTML', is_caption: bool = False) -> Dict[str, Any]:
    ...
```

Parameters:
- `text`: input string with formatting tags or markdown
- `parse_mode`: `'HTML'` (default) or `'Markdown'`
- `is_caption`: if `True`, the returned dict text key is `"caption"`; otherwise `"message"`

Returns a dict with:
- `"message"` (or `"caption"`): plain text with formatting markers removed
- `"entities"`: list of `TLRPC.MessageEntity` objects

## Usage example

```python
from extera_utils.text_formatting import parse_text
from client_utils import send_message

# Parsing HTML
html_input = "<b>Bold</b>, <i>Italic</i>, and <a href='https://example.com'>Link</a>"
parsed_html = parse_text(html_input, parse_mode='HTML')
# parsed_html is roughly:
# {
#   "message": "Bold, Italic, and Link",
#   "entities": [TL_messageEntityBold(...), TL_messageEntityItalic(...), TL_messageEntityTextUrl(...)]
# }

# You can pass this dict directly to send_message (via kwargs unpacking if keys match params)
# But client_utils functions handle this automatically via the parse_mode argument.

# Parsing Markdown
md_input = "**Bold**, __Italic__, and [Link](https://example.com)"
parsed_md = parse_text(md_input, parse_mode='Markdown')
```

## Supported Formatting

### HTML tags

- `<b>`, `<strong>` — Bold
- `<i>`, `<em>` — Italic
- `<u>` — Underline
- `<s>`, `<del>`, `<strike>` — Strikethrough
- `<a href="...">` — Text Link
- `<code>` — Inline Code
- `<pre language="...">` — Preformatted Code Block (language optional)
- `<spoiler>`, `<tg-spoiler>` — Spoiler
- `<blockquote>` — Blockquote (add `expandable` or `collapsed` attribute for expandable blockquotes, e.g. `<blockquote expandable>`)
- `<emoji id="...">` — Custom Emoji

### Markdown

- `*bold*` — Bold
- `_italic_` — Italic
- `__underline__` — Underline
- `~strikethrough~` — Strikethrough
- `||spoiler||` — Spoiler
- `` `code` `` — Inline Code
- ` ```code block``` ` — Preformatted Code Block
- ` ```python ... ``` ` — Code Block with language
- `[text](url)` — Text Link
- `![alt](tg://emoji?id=123)` — Custom Emoji
- `> Quote` — Blockquote
- `**> Quote` — Expandable Quote

## Helper Classes

### `TLEntityType`

Enum of supported entity types: `CODE`, `PRE`, `STRIKETHROUGH`, `TEXT_LINK`, `BOLD`, `ITALIC`, `UNDERLINE`, `SPOILER`, `CUSTOM_EMOJI`, `BLOCKQUOTE`.

### `RawEntity`

Intermediate representation of an entity before conversion to TLRPC. Contains `offset`, `length`, and extra attributes like `url`, `language`, `document_id` (for emojis), `collapsed` (for blockquotes).
