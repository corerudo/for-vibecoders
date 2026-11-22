```
# Анализ разработки плагина genderlib

> Гендерно-ориентационная библиотека по команде: отправка большого форматированного справочника одним сообщением со сворачиваемой цитатой

---

## Архитектура: как это работает в целом

Плагин вешает хук на отправку сообщений через `add_on_send_message_hook`. Когда пользователь отправляет сообщение, состоящее ровно из префикса и команды (по умолчанию `.sgl`), исходный текст заменяется на встроенный справочник `SGL_TEXT`. В отличие от схемы «отменить и отправить своё», здесь используется `HookStrategy.MODIFY` — сообщение отправляется, но с подменённым содержимым и заранее распарсенными entities.

Markdown справочника разбирается штатным `parse_markdown` из `markdown_utils`, entities конвертируются в `TLRPC`-объекты, и поверх них вручную достраивается `TL_messageEntityBlockquote` — весь справочник после первой строки уходит в сворачиваемую цитату.

Настройки: пользовательская команда (`Input` с валидацией), выбор префикса из трёх вариантов через кастомную View с чипсами (`Custom`), и переключатель сворачивания цитаты (`Switch`).

---

## Ключевые решения и нюансы

### 1. `HookStrategy.MODIFY` вместо `CANCEL`

Главное архитектурное отличие от «тяжёлых» плагинов. Команда `.sgl` не глушится, а мутирует на лету:

```python
params.message = text
params.entities = entities
return HookResult(strategy=HookStrategy.MODIFY, params=params)
```

Не нужен `send_text`, нет проблем с `msg_id`, нет асинхронности — фреймворк сам отправляет уже изменённый `params`. Это самый дешёвый способ «ответить командой» на сообщение.

---

### 2. Точное сравнение команды: `strip` + `casefold`

```python
if not command or message.strip().casefold() != (prefix + command).casefold():
    return HookResult()
```

Сообщение должно состоять **ровно** из `prefix + command` — никаких аргументов не поддерживается. `strip()` прощает случайные пробелы по краям, `casefold()` делает матч регистронезависимым (`.SGL`, `.sGl` — всё сработает). Если совпадения нет — возвращается пустой `HookResult()`, т.е. no-op, стандартная отправка продолжается.

---

### 3. Нормализация команды: защита от пользовательского ввода

```python
while command and not command[0].isalnum() and command[0] != "_":
    command = command[1:]
```

Пользователь может вписать в поле команды `.sgl` или `/sgl` — плагин срезает все ведущие не-буквенные символы (кроме `_`). `isalnum()` в Python Unicode-совместимый, поэтому кириллические команды тоже проходят. Эта нормализация продублирована и в `on_command_change` (при вводе в настройках), и в хуке (при матче) — команда валидна вне зависимости от того, где её почистили.

Если после очистки строка пустая — в `on_command_change` значение сбрасывается на дефолт:

```python
if not command:
    self.set_setting("command_pg", DEFAULT_COMMAND, reload_settings=True)
```

`reload_settings=True` перечитывает UI настроек, и пользователь сразу видит установленный дефолт.

---

### 4. `parse_markdown` + `to_tlrpc_object()` — штатный парсинг богатого текста

```python
parsed = parse_markdown(SGL_TEXT)
text = parsed.text
entities = ArrayList()

for entity in parsed.entities:
    entities.add(entity.to_tlrpc_object())
```

`SGL_TEXT` хранится в исходнике в читаемом Markdown: `*жирный*`, `### заголовки`, списки через `-`. `parse_markdown` превращает разметку в плоский текст + список обёрток entities, у каждой есть конвертер `to_tlrpc_object()` в нативный объект Telegram. Это избавляет от ручной раскладки offset/length для каждого стиля.

---

### 5. Ручная сборка `TL_messageEntityBlockquote` со `collapsed`

```python
quote_start = text.find("\n") + 1
if quote_start > 0:
    blockquote = TLRPC.TL_messageEntityBlockquote()
    blockquote.offset = utf16_length(text[:quote_start])
    blockquote.length = utf16_length(text[quote_start:])
    blockquote.collapsed = bool(self.get_setting("collapsed", True))
    entities.add(blockquote)
```

Цитата покрывает всё после первой строки — жирный заголовок «Добро пожаловать...» остаётся снаружи, а весь справочник уходит внутрь. Граница считается по первому `\n`; если его нет, `find` вернёт `-1`, `+1` даст `0`, и гард `if quote_start > 0` отключит цитату.

`collapsed` — флаг сворачиваемой цитаты (фича новых клиентов, отсюда и `__min_version__ = "11.12.1"`): свёрнутый блок показывается с многоточием и разворачивается по тапу. Состояние берётся из настройки `Switch`.

Entities из Markdown и blockquote накладываются друг на друга (жирные заголовки внутри цитаты) — Telegram корректно обрабатывает вложенное форматирование.

---

### 6. `utf16_length` — offsets в UTF-16 единицах

```python
def utf16_length(value):
    return len(value.encode("utf-16-le")) // 2
```

Telegram считает `offset` и `length` entities в UTF-16 code units, а не в Python-символах. Любой символ вне BMP (эмодзи, некоторые редкие иероглифы) занимает 2 единицы. Правильный подсчёт через кодировку `utf-16-le` — без него любые entities в тексте с эмодзи «поехали» бы.

---

### 7. Кастомная View префиксов через `Custom`

Выбор префикса — не стандартный элемент настроек, а программно собранный Android-layout:

```python
prefixes = ("!", ".", ",")
...
return Custom(view=root)
```

Структура: горизонтальный `LinearLayout` → иконка → подписи («Префикс» / «Символ перед командой») → трек с тремя чипсами. Клик по чипсу сразу пишет выбор в настройки:

```python
self.set_setting("prefix_index", index)
```

Активный чипс подсвечивается акцентным цветом, неактивные — прозрачным фоном.

---

### 8. Защита типов настроек: `bool` — это `int` в Python

```python
if not isinstance(selected, int) or isinstance(selected, bool) or selected not in range(len(prefixes)):
    selected = 1
```

Нюанс: в Python `True` — подкласс `int`, и `True in range(3)` вернёт `True`. Отдельная проверка `isinstance(selected, bool)` отсекает битые булевы значения из хранилища. Аналогичная защита стоит и для `command` (`isinstance(command, str)`) — настройки могут вернуть что угодно, и плагин не должен падать на этом.

---

### 9. Темизация через `Theme.getColor`

```python
text_color = Theme.getColor(Theme.key_windowBackgroundWhiteBlackText)
hint_color = Theme.getColor(Theme.key_windowBackgroundWhiteGrayText2)
accent_color = Theme.getColor(Theme.key_windowBackgroundWhiteValueText)
```

Все цвета UI берутся из ключей темы клиента — настройки автоматически поддерживают светлую/тёмную тему и любые акценты без единой строки на тему.

---

### 10. `argb` — знаковые цвета для Java

```python
def argb(alpha, rgb):
    value = ((alpha & 0xFF) << 24) | (rgb & 0xFFFFFF)
    return value - (1 << 32) if value >= (1 << 31) else value
```

Java-цвета — знаковые 32-битные int. Значения с установленным старшим битом (> `0x7FFFFFFF`) Java ожидает отрицательными. Helper приводит беззнаковую сборку к знаковой — иначе `setColor()` получил бы лишний большой long и упал.

---

### 11. `dynamic_proxy` для кликов

```python
class Click1(dynamic_proxy(View.OnClickListener)):
    def __init__(self, fn):
        super().__init__()
        self._fn = fn

    def onClick(self, view):
        self._fn(view)
```

Android-требование: `setOnClickListener` принимает Java-интерфейс. Обёртка через `dynamic_proxy` позволяет навесить Python-колбэк. Внутри `make_click(index)` замыкание захватывает индекс чипса — классический фабричный паттерн обработчиков в цикле.

---

### 12. Fallback-цепочки без падений

- `set_bold` пробует `AndroidUtilities.bold()`, при неудаче — `getTypeface("fonts/rmedium.ttf")`, при неудаче — обычный шрифт.
- `load_icon` при любом сбое вернёт `None`, и layout просто не добавит иконку (`if icon_drawable is not None`), а отступ `labels_params.leftMargin` ставится в 0.

Плагин никогда не падает из-за отсутствующих ресурсов клиента.

---

### 13. Молчаливая деградация при исключении

```python
except Exception:
    return HookResult()
```

Любая ошибка парсинга/сборки entities сводится к no-op — сообщение уходит как есть. Минус: если исключение случилось **после** совпадения команды, в чат уйдёт сырой текст `.sgl`. Плюс: плагин гарантированно не ломает отправку сообщений.

---

## Паттерны, которые стоит переиспользовать

### Подмена содержимого отправляемого сообщения (вместо отмены)

```python
def on_send_message_hook(self, account, params):
    result = HookResult()
    if message.strip().casefold() != (prefix + command).casefold():
        return result
    # ...подготовить text и entities...
    params.message = text
    params.entities = entities
    return HookResult(strategy=HookStrategy.MODIFY, params=params)
```

---

### Отправка Markdown с entities и сворачиваемой цитатой

```python
from markdown_utils import parse_markdown
from org.telegram.tgnet import TLRPC
from java.util import ArrayList

parsed = parse_markdown(MARKDOWN_TEXT)
entities = ArrayList()
for entity in parsed.entities:
    entities.add(entity.to_tlrpc_object())

quote_start = parsed.text.find("\n") + 1
if quote_start > 0:
    bq = TLRPC.TL_messageEntityBlockquote()
    bq.offset = utf16_length(parsed.text[:quote_start])
    bq.length = utf16_length(parsed.text[quote_start:])
    bq.collapsed = True
    entities.add(bq)
```

---

### Корректные offsets для Telegram entities

```python
def utf16_length(value):
    return len(value.encode("utf-16-le")) // 2
```

---

### Чипсы-переключатели в настройках

```python
state = {"index": selected}
chips = []

def refresh():
    for i, chip in enumerate(chips):
        active = i == state["index"]
        bg = GradientDrawable()
        bg.setColor(accent_color if active else 0)
        bg.setCornerRadius(dp(9))
        chip.setBackground(bg)
        chip.setTextColor(-1 if active else text_color)

def make_click(index):
    def on_click(view):
        if state["index"] == index:
            return
        state["index"] = index
        self.set_setting("prefix_index", index)
        refresh()
    return on_click
```

---

### Валидация числовых настроек (с учётом `bool`)

```python
val = self.get_setting("some_index", 0)
if not isinstance(val, int) or isinstance(val, bool) or val not in range(count):
    val = 0
```

---

### Валидация текстовой настройки в `on_change`

```python
def on_command_change(self, new_value):
    command = new_value.strip() if isinstance(new_value, str) else ""
    while command and not command[0].isalnum() and command[0] != "_":
        command = command[1:]
    if not command:
        self.set_setting("command_pg", DEFAULT_COMMAND, reload_settings=True)
```

---

## Полная схема работы (.sgl)

```
on_plugin_load()
    └── add_on_send_message_hook()

on_send_message_hook(account, params)
    ├── message не str → HookResult() (no-op)
    ├── command = get_setting("command_pg", "sgl")
    ├── prefix_index = get_setting("prefix_index", 1) + валидация типа
    ├── prefix = prefixes[prefix_index]          (! . ,)
    ├── нормализация command (срез не-буквенных префиксов)
    ├── message.strip().casefold() != prefix+command → HookResult()
    ├── parse_markdown(SGL_TEXT) → text + entities
    ├── entities → to_tlrpc_object() → ArrayList
    ├── TL_messageEntityBlockquote:
    │       offset   = utf16_length(text[:первая строка])
    │       length   = utf16_length(остаток)
    │       collapsed = get_setting("collapsed", True)
    ├── params.message  = text
    ├── params.entities = entities
    └── HookResult(MODIFY, params)
            └── в чат уходит справочник со сворачиваемой цитатой

исключение → HookResult()   (команда уйдёт как обычный текст)

create_settings()
    ├── Input(command_pg, on_change=on_command_change)
    ├── Custom(view=prefix chips)   ← клик → set_setting("prefix_index", i)
    └── Switch(collapsed, default=True)
```
```

