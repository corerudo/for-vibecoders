# exteraGram Plugin SDK Type Stubs (`.pyi`)

Официальные файлы типизации (Type Stubs) Python SDK для плагинов **exteraGram** и **AyuGram**.

## Источник
- Собрано и сгенерировано из официального репозитория сборок SDK: [`exteraSquad/plugins-pysdk-builds`](https://github.com/exteraSquad/plugins-pysdk-builds)
- Версия SDK: `1.4.6.5-beta` / `1.4.6.4-stable`
- Официальная документация: [plugins.exteragram.app](https://plugins.exteragram.app)

## Состав модулей
- `base_plugin.pyi` — базовый класс `BasePlugin`, метаданные, хуки (`add_hook`, `on_send_message_hook`, partial hooking: `hook_call_site`, `hook_field_read`, `hook_field_write`, `hook_string_constant`, `hook_return_value`), меню `MenuItemData`, `HookResult`, `HookStrategy`, `HookFilter`.
- `client_utils.pyi` — отправка сообщений (`send_text`, `send_photo`, `send_document`, `send_video`, `send_audio`, `edit_message`), фоновые очереди (`run_on_queue`), доступ к контроллерам (`get_messages_controller`, `get_user_config` и др.), многоаккаунтность (`AccountClient`, `get_client`, `get_selected_account`, `account_scope`).
- `intents.pyi` — `IntentsManager` для перехвата ссылок, диплинков и Telegram-интентов (exteraGram 12.6.4+).
- `log_utils.pyi` — логгер `Logger`, `LogLevel`, `dump`, `timed`, контекстный вывод в logcat под тегом `plugin.<id>`.
- `dev_server.pyi` — встроенный TCP/IPC dev-сервер для быстрой перезагрузки и отладки плагинов с ПК.
- `file_utils.pyi` — `FilesController`, работа со стандартными директориями Telegram и файлами.
- `hook_utils.pyi` — рефлексия (`find_class`, `java_class`, `get_method`, `get_constructor`, `get_field`, `get_private_field`, `set_private_field`).
- `markdown_utils.pyi` — утилиты форматирования текста.
- `plugin_settings.pyi` — настройки плагина.
- `_sdk_version.pyi` — версия SDK и системные хуки.
- `elyxcore/` — ядро модульного формата плагинов Elyx:
  - `assets.pyi` — работа с графикой, SVG, Lottie и ресурсами (`Assets`, `Asset`).
  - `localization.pyi` — локализация и строковые ресурсы (`Strings`).
  - `settings.pyi` — контроллер настроек Elyx (`SettingsController`).
  - `utils.pyi` — MVEL-исполнение, ленивые словари и хелперы.
- `extera_utils/` — расширенные утилиты:
  - `classes.pyi` — Class Proxy: генерация Java-классов из Python на лету (`java_subclass`, `joverride`, `jmethod`, `jfield`, `JA`, `JGS`, `JS`).
  - `lifecycle.pyi` — управление жизненным циклом.
  - `metadata_parser.pyi` — парсер метаданных AST.
  - `text_formatting.pyi` — парсер форматирования Markdown/HTML в сущности Telegram.
- `ui/` — компоненты интерфейса:
  - `settings.pyi` — декларативные элементы экрана настроек (`Header`, `Switch`, `Selector`, `Input`, `Text`, `EditText`, `Divider`, `Custom`, `SimpleSettingFactory`).
  - `alert.pyi` — `AlertDialogBuilder`.
  - `bulletin.pyi` — `BulletinHelper`.

## Использование в IDE
Для поддержки автодополнения и подсветки типов в VS Code / Cursor / PyCharm добавьте путь к этой папке в `python.analysis.extraPaths`:
```json
{
  "python.analysis.extraPaths": ["./stubs"]
}
```
