# Анализ разработки плагина muztep

> Поиск и отправка треков с muztep.net прямо из чата AyuGram / ExteraGram

---

## Архитектура: как это работает в целом

Плагин перехватывает отправку сообщений через `add_on_send_message_hook`. Если сообщение начинается с `.shm автор - название` — оригинальная отправка отменяется, запускается фоновая задача на `PLUGINS_QUEUE`. Она ищет трек на muztep.net, скачивает его, тегирует метаданными и отправляет как аудио с обложкой и caption.

Центральная проблема всего плагина — правильно передать Telegram все данные трека (название, исполнитель, обложка, длительность), не потеряв их по пути.

---

## Ключевые решения и нюансы

### 1. `add_on_send_message_hook` — точка входа

`on_send_message_hook` срабатывает до реальной отправки сообщения. `HookStrategy.CANCEL` полностью её блокирует — пользователь отправляет `.shm ...`, но это сообщение не уходит в чат.

```python
result.strategy = HookStrategy.CANCEL
return result
```

Важно возвращать `result` с `CANCEL` **до** запуска фоновой задачи — иначе сообщение успеет уйти.

---

### 2. `run_on_queue` — обязательный уход в фон

Все сетевые операции (поиск, скачивание, парсинг) выполняются в `PLUGINS_QUEUE`, а не в UI-потоке. Без этого приложение замерзает на время запроса.

```python
run_on_queue(
    lambda: self._search_and_send(artist_q, title_q, peer_id),
    PLUGINS_QUEUE,
)
```

`lambda` нужна, потому что `run_on_queue` принимает callable без аргументов — аргументы захватываются из замыкания.

---

### 3. `BypassInternalUriCheck` — почему без него не работает `send_audio`

Telegram проверяет URI файла перед отправкой через `AndroidUtilities.isInternalUri`. Файлы из кэш-директории приложения проходят эту проверку как «внутренние» и блокируются — Telegram не позволяет отправлять их напрямую.

Хук подменяет этот метод, всегда возвращая `False` («не внутренний»):

```python
class BypassInternalUriCheck(MethodReplacement):
    def replace_hooked_method(self, param):
        return False
```

Хук навешивается непосредственно перед отправкой и снимается сразу после — в `finally`, чтобы не влиять на остальную работу приложения.

---

### 4. `_hook_uri` / `_unhook_uri` — минимальное окно хука

```python
self._hook_uri()
try:
    send_audio(...)
finally:
    self._unhook_uri()
```

Хук активен только на время вызова `send_audio`. `finally` гарантирует снятие даже при исключении — иначе `isInternalUri` остался бы сломан до перезапуска приложения.

Отдельный `self._unhook_uri()` в блоке `except` выше нужен для случаев, когда исключение произошло до блока `try/finally` с `send_audio`.

---

### 5. `_search_track` — парсинг первой ссылки из поиска

```python
link = soup.find("a", href=re.compile(r"^/track/"))
```

Берётся первая ссылка на трек из результатов поиска. `re.compile(r"^/track/")` — фильтр по началу href, чтобы не захватить ссылки на артистов и альбомы.

Из URL трека регуляркой извлекается числовой `track_id` — он нужен для API скачивания:

```python
re.search(r"/track/[^/]+/(\d+)", track_url)
```

---

### 6. `_get_track_meta` — данные из `.track-actions`

Страница трека содержит `div.track-actions` с атрибутами `data-title`, `data-artist`, `data-thumb`. Это надёжнее, чем парсить `<h1>` или `og:title`, потому что данные уже очищены и структурированы самим сайтом.

```python
actions = soup.find(class_="track-actions")
title = actions.get("data-title")
artist = actions.get("data-artist")
thumb_url = actions.get("data-thumb")
```

Если парсинг не дал результата — используются query-строки пользователя как fallback:

```python
real_title = meta["title"] if meta["title"] else title_q
real_artist = meta["artist"] if meta["artist"] else artist_q
```

---

### 7. `_get_cdn_url` — получение прямой ссылки на файл

muztep.net предоставляет API для скачивания по `track_id`. Важно передавать `Referer` страницы трека — без него сервер возвращает ошибку.

```python
headers["Referer"] = track["track_url"]
```

---

### 8. `_audio_ext` — определение реального формата файла

Файлы на muztep называются `.mp3`, но по факту являются AAC/M4A. Реальный формат определяется по `Content-Type` ответа сервера:

```python
if ct in ("audio/mp4", "audio/x-m4a", "audio/aac"):
    return ".m4a"
```

Если `Content-Type` не помог — проверяется расширение в URL. Фоллбек — `.mp3`. Это важно для `mutagen`, который определяет формат по расширению.

---

### 9. `_send_track` — ручная сборка `TL_documentAttributeAudio`

Главное техническое решение плагина. `send_audio` из `client_utils` читает метаданные из файловых тегов, но это не всегда надёжно. `_send_track` обходит это, вручную собирая Telegram-объект с нужными данными:

```python
attr = TLRPC.TL_documentAttributeAudio()
attr.duration = self._duration(path)
attr.title = title or ""
attr.performer = artist or ""
attr.flags = attr.flags | 1 | 2
document.attributes.add(attr)
```

`flags = flags | 1 | 2` — битовые флаги, сигнализирующие Telegram что поля `title` (бит 0) и `performer` (бит 1) заполнены. Без этого клиент их игнорирует.

Если `_prepare_document` или импорт `TLRPC` падает — fallback на обычный `send_audio`:

```python
except Exception:
    send_audio(peer_id, path, caption=caption)
    return
```

---

### 10. Обложка через `ImageLoader.scaleAndSaveImage`

```python
bmp = BitmapFactory.decodeByteArray(cover_bytes, 0, len(cover_bytes))
thumb = ImageLoader.scaleAndSaveImage(bmp, 320, 320, 80, False)
document.thumbs.add(thumb)
bmp.recycle()
```

`BitmapFactory.decodeByteArray` — стандартный Android API для декодирования изображения из байтов. `ImageLoader.scaleAndSaveImage` — внутренний метод Telegram, который масштабирует и сохраняет превью в нужном формате, возвращая `TLRPC.PhotoSize`. Именно такой объект ожидает `document.thumbs`.

`bmp.recycle()` обязателен — Bitmap в Android выделяет нативную память, которую GC не освобождает автоматически.

---

### 11. `_duration` через `mutagen`

```python
m = mutagen.File(path)
if m is not None and m.info is not None:
    return float(m.info.length)
```

`mutagen.File` автоматически определяет формат файла и парсит его заголовок. `m.info.length` — длина в секундах. Эта информация нужна для `TL_documentAttributeAudio.duration` — без неё Telegram не показывает прогресс-бар у трека.

---

### 12. Удаление файла: `finally` после отправки, `except` при ошибке

Исходная проблема: `finally` удалял файл сразу, не дожидаясь завершения загрузки, потому что `send_audio` асинхронен — он ставит файл в очередь, а не ждёт окончания отправки.

Решение: файл хранится в `get_cache_dir()` (не во временной памяти, а в кэш-директории приложения). Удаляется в двух случаях:
- В `finally` после `send_audio` — при успешном завершении
- В `except` — при любой ошибке до отправки

```python
try:
    send_audio(...)
finally:
    self._unhook_uri()
    try:
        os.remove(tmp_path)
    except Exception:
        pass
```

---

### 13. Буллетины как индикатор прогресса

Поскольку операция фоновая, пользователь не видит прогресса. Три буллетина решают это:

- `"ищу: ..."` — запрос к поиску отправлен
- `"выгружаю: ..."` — трек найден, скачиваем
- `"найс: ..."` — файл передан в очередь отправки

При ошибке — `BulletinHelper.show_with_button` с кнопкой «Копировать», которая копирует текст исключения в буфер обмена. Это критично для отладки, поскольку плагин работает на реальном устройстве без доступа к логам.

---

### 14. `_safe_name` — безопасное имя файла

```python
name = re.sub(r'[\\/:*?"<>|\x00-\x1f]', "_", name)[:120].strip()
return name or "track"
```

Символы из запрещённого набора заменяются на `_`. Лимит 120 символов — защита от слишком длинных путей на Android. Fallback `"track"` — если после очистки строка пустая.

---

## Паттерны, которые стоит переиспользовать

### Отмена отправки сообщения и уход в фон

```python
def on_send_message_hook(self, account, params):
    result = HookResult()
    # ...проверки...
    run_on_queue(lambda: self._do_work(peer_id), PLUGINS_QUEUE)
    result.strategy = HookStrategy.CANCEL
    return result
```

### Временный хук с гарантированным снятием

```python
self._hook_uri()
try:
    # операция, требующая хука
finally:
    self._unhook_uri()
```

### Ручная сборка аудио-документа с метаданными

```python
from org.telegram.tgnet import TLRPC
from client_utils import _prepare_document, send_message

document = _prepare_document(path, "audio/mpeg")
attr = TLRPC.TL_documentAttributeAudio()
attr.title = title
attr.performer = artist
attr.duration = duration
attr.flags = attr.flags | 1 | 2
document.attributes.add(attr)
send_message({"peer": peer_id, "document": document, "path": path, "caption": caption})
```

### Fallback-цепочка для ненадёжных операций

```python
try:
    # основной путь (сложный, но полный)
    ...
except Exception:
    # запасной путь (простой, но работает)
    send_audio(peer_id, path, caption=caption)
    return
```

---

## Полная схема работы

```
on_plugin_load()
    └── add_on_send_message_hook()

on_send_message_hook(account, params)
    ├── проверка: starts_with(".shm ")
    ├── парсинг: artist_q, title_q из текста
    ├── run_on_queue → _search_and_send()
    └── HookStrategy.CANCEL

_search_and_send(artist_q, title_q, peer_id)
    ├── BulletinHelper: "ищу: ..."
    ├── _search_track(query)
    │       ├── GET /search/{query}
    │       ├── BeautifulSoup: первая ссылка /track/
    │       └── regex: extract track_id
    ├── _get_track_meta(track_url)
    │       ├── GET track_url
    │       └── BeautifulSoup: .track-actions → data-title, data-artist, data-thumb
    ├── BulletinHelper: "выгружаю: ..."
    ├── _get_cdn_url(track, artist, title)
    │       └── GET /api/download/{track_id} → json.url
    ├── _download(cdn_url, artist, title)
    │       ├── _audio_ext(): Content-Type → .m4a / .mp3
    │       └── streaming download → get_cache_dir()
    ├── GET thumb_url → cover_bytes
    ├── _hook_uri()
    ├── _send_track(peer_id, path, caption, artist, title, cover_bytes)
    │       ├── _prepare_document(path, "audio/mpeg")
    │       ├── TL_documentAttributeAudio: title, performer, duration, flags
    │       ├── BitmapFactory + ImageLoader → document.thumbs
    │       └── send_message({document, path, caption})
    │           [fallback: send_audio(peer_id, path)]
    ├── BulletinHelper: "найс: ..."
    └── finally: _unhook_uri() + os.remove(tmp_path)

on_plugin_unload()
    └── _unhook_uri()
```
