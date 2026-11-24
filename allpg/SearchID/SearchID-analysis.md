# Анализ разработки плагина SearchID

> **Назначение:** добавляет поиск плагинов ExteraGram по ID через штатный поиск менеджера плагинов.
>
> **Версия плагина:** 1.0.3

---

## 1. Краткое описание

### Что делает плагин

SearchID позволяет находить установленные плагины по их идентификатору (`__id__`), а не только по отображаемому названию.

Плагин не создаёт собственный экран, список результатов или поисковую систему. Он подключается к построению штатного списка плагинов и на время этой операции изменяет результат `Plugin.getName()`: если введённый запрос встречается в ID, метод возвращает строку вида `Название ID`.

За счёт этого ID становится дополнительным текстом, по которому может сработать существующая фильтрация менеджера плагинов.

### Какую проблему решает

Названия плагинов не всегда совпадают с их ID. Например, пользователь может знать технический идентификатор `animtou`, но в списке видеть название `Анимированные касания`. Штатный поиск по отображаемому названию не обязан находить такой плагин по ID.

SearchID добавляет возможность искать по полному ID и его части, не реализуя отдельный алгоритм отбора элементов.

### Основной принцип реализации

Используются два Xposed-хука:

1. `PluginsActivity.fillItems` — перед выполнением метода плагин читает приватное поле `query` и сохраняет нормализованный запрос в собственном состоянии.
2. `Plugin.getName` — после оригинального вызова плагин получает ID объекта. Если запрос является подстрокой ID, результат `getName()` заменяется на `Название ID`.

После завершения `fillItems` поисковое состояние очищается. Вне этого участка вызовы `getName()` не изменяются.

### Границы ответственности

Плагин:

- добавляет ID как дополнительный источник совпадения при поиске плагинов;
- использует существующие классы и Xposed-механизм ExteraGram;
- временно изменяет возвращаемое значение `getName()` во время построения списка;
- показывает копируемое сообщение с traceback при ошибках.

Плагин **не**:

- создаёт собственный поиск или список результатов;
- меняет сохранённое имя, ID или метаданные плагина;
- записывает ID в настройки или постоянное хранилище;
- обращается к сети;
- изменяет содержимое сообщений, чатов или аккаунтов;
- добавляет отдельный экран настроек.

---

## 2. Область анализа и совместимость

| Параметр | Значение | Примечание |
|---|---|---|
| Версия плагина | `1.0.2` | Указана в `__version__` |
| ID плагина | `SearchID` | Указан в `__id__` |
| ExteraGram | `>=13.2.2` | Ограничение из `__app_version__` |
| AyuGram | Не заявлена | Используются классы с пакетом `com.exteragram.messenger` |
| Plugin SDK | В текущем `main` исходников ETG указан `1.4.6.5` | Плагин не задаёт собственный `__sdk_version__` |
| Python runtime | `3.11` | Указан в Gradle-конфигурации Python SDK ExteraGram |
| Android API / ABI | Отдельных требований плагина нет | Нет собственного native-кода и прямых ABI-зависимостей |

`__app_version__ = ">=13.2.2"` означает минимальное требование, а не ограничение ровно на 13.2.2. Оно не доказывает совместимость со всеми последующими версиями: при изменении внутренних классов, полей или методов хуки могут перестать находить нужные точки.

### Что входит в анализ

- полный исходник `SearchID-1.0.2.plugin`;
- используемые классы `BasePlugin` и `MethodHook`;
- реализация `hook_method()` и `unhook_method()` в SDK;
- реализация `find_class()` и `get_private_field()`;
- реализация `android_utils.log()` и `copy_to_clipboard()`;
- реализация `BulletinHelper.show_with_button()`;
- метаданные SDK и настройка Python runtime в исходниках ExteraGram;
- ресурс `R.raw.error`, на который ссылается плагин;
- официальная документация по метаданным, хукам, Reflection, Bulletin и логированию.

### Что не входит в анализ

В доступных файлах публичного репозитория ExteraGram удалось открыть исходники SDK и связанные ресурсы, но не удалось получить декларации самих `PluginsActivity` и `Plugin` из приложения. Поэтому здесь не приводятся выдуманные полные Java/Kotlin-сигнатуры этих методов и не утверждается, что их объявления были повторно сверены по файлу класса в ветке `main`.

Плагин использует эти классы и методы напрямую по именам, а его рабочее поведение подтверждено владельцем плагина. Для самостоятельного повторного подтверждения внутренней реализации приложения нужен соответствующий исходник или APK целевой сборки 13.2.2, доступный для просмотра через JADX.

---

## 3. Использованные источники

### Исходник плагина

| Файл | Что проверялось |
|---|---|
| `SearchID-1.0.2.plugin` | Метаданные, импорты, хуки, состояние, обработка ошибок, регистрация и снятие хуков |

### Исходники ExteraGram

| Файл | Что проверялось |
|---|---|
| [`plugins-pysdk/src/base_plugin.py`](https://github.com/exteraSquad/exteraGram/blob/main/plugins-pysdk/src/base_plugin.py) | `BasePlugin`, `MethodHook`, `hook_method()`, `hook_all_methods()`, `unhook_method()`, учёт хуков в `PluginsController` |
| [`plugins-pysdk/src/hook_utils.py`](https://github.com/exteraSquad/exteraGram/blob/main/plugins-pysdk/src/hook_utils.py) | Реальное поведение `find_class()` и `get_private_field()` |
| [`plugins-pysdk/src/android_utils.py`](https://github.com/exteraSquad/exteraGram/blob/main/plugins-pysdk/src/android_utils.py) | Реализация `log()`, `copy_to_clipboard()` и UI-thread helper |
| [`plugins-pysdk/src/ui/bulletin.py`](https://github.com/exteraSquad/exteraGram/blob/main/plugins-pysdk/src/ui/bulletin.py) | Реализация `show_with_button()` и передача callback в Android Bulletin |
| [`plugins-pysdk/src/_sdk_version.py`](https://github.com/exteraSquad/exteraGram/blob/main/plugins-pysdk/src/_sdk_version.py) | Версия SDK, указанная непосредственно в исходном файле |
| [`TMessagesProj/gradle/extera-python.gradle`](https://github.com/exteraSquad/exteraGram/blob/main/TMessagesProj/gradle/extera-python.gradle) | Версия Python и конфигурация Python runtime |
| [`TMessagesProj/src/main/res/raw/error.json`](https://github.com/exteraSquad/exteraGram/blob/main/TMessagesProj/src/main/res/raw/error.json) | Наличие Lottie-ресурса ошибки, используемого через `R.raw.error` |

### Документация ExteraGram

| Страница | Что сверялось |
|---|---|
| [Plugin Class](https://plugins.exteragram.app/docs/plugin-class) | Метаданные плагина и смысл `__app_version__` |
| [Xposed Method Hooking](https://plugins.exteragram.app/docs/xposed-hooking) | `MethodHook`, `param.getResult()`, `param.setResult()`, `hook_method()` и снятие хуков |
| [Hook Utilities](https://plugins.exteragram.app/docs/hook-utils) | Назначение `find_class()` и `get_private_field()` |
| [Bulletin Helper](https://plugins.exteragram.app/docs/bulletin-helper) | Аргументы `show_with_button()` и автоматический переход в UI-thread |
| [Logging](https://plugins.exteragram.app/docs/logging) | Разница между новым `self.logger` и legacy `android_utils.log()` |

### Важное расхождение документации и исходников SDK

В текущем просмотренном файле [`_sdk_version.py`](https://github.com/exteraSquad/exteraGram/blob/main/plugins-pysdk/src/_sdk_version.py) указана версия SDK `1.4.6.5`. Страница Introduction документации показывает baseline `1.4.4.3`. Для описания фактической реализации SDK в этом анализе использована версия из исходного файла репозитория; документация использована для объяснения публичного API и сопоставления его поведения.

Плагин не задаёт `__sdk_version__`, поэтому отдельного требования к версии SDK в его метаданных нет.

### Ограничение проверки классов приложения

Точные объявления `com.exteragram.messenger.plugins.ui.PluginsActivity` и `com.exteragram.messenger.plugins.Plugin` в доступной части публичного дерева исходников открыть не удалось. Из-за этого в анализе фиксируются используемые имена методов, полей и фактическая техника обращения к ним из плагина, но не придумываются сигнатуры, модификаторы доступа или детали внутренней фильтрации, которые не подтверждены доступным файлом приложения.

---

## 4. Пользовательское поведение

| Действие пользователя | Реакция плагина | Видимый результат |
|---|---|---|
| Вводит полный ID плагина | Запрос сравнивается с ID каждого плагина во время построения списка | Плагин может попасть в результаты по ID |
| Вводит часть ID | Выполняется проверка вхождения подстроки | Совпадения по части ID также находятся |
| Вводит ID в другом регистре | Запрос и ID приводятся к нижнему регистру | Регистр не влияет на совпадение |
| Ищет по обычному названию | Результат `getName()` остаётся исходным, если ID не совпал | Обычный поиск по названию продолжает работать штатно |
| Вводит запрос с пробелами по краям | `strip()` убирает пробелы по краям | Сравнение выполняется по очищенному запросу |
| Плагин не имеет ID или имени (`None`) | Обработчик прекращает изменение результата | Исходное значение не подменяется |
| Поисковый запрос пустой | `GetNameHook` сразу завершает работу | ID не добавляется к имени |
| Происходит ошибка загрузки или работы хука | Формируется traceback, запись отправляется в log, показывается Bulletin | Пользователь видит сообщение с кнопкой `Copy` |

### Основной сценарий — поиск по ID

1. Пользователь открывает менеджер плагинов и вводит ID или его часть.
2. ExteraGram вызывает `PluginsActivity.fillItems()`.
3. Хук до оригинального метода читает `query` из экземпляра Activity.
4. Во время выполнения `fillItems()` вызывается `Plugin.getName()`.
5. После оригинального `getName()` хук получает ID текущего объекта.
6. Если запрос содержится в ID, результат меняется на `Название ID`.
7. Штатная обработка списка получает строку, содержащую ID, и может использовать её как дополнительное совпадение.
8. После завершения `fillItems()` поисковое состояние очищается.

### Сценарий — поиск по названию

1. Пользователь вводит название.
2. `fillItems()` сохраняет запрос.
3. `GetNameHook` сравнивает запрос с ID.
4. Если ID не содержит запрос, хук не вызывает `param.setResult()`.
5. Оригинальное имя остаётся результатом `getName()`, и поиск по названию продолжает обрабатываться клиентом.

---

## 5. Архитектура: как это работает в целом

SearchID использует штатный поиск и добавляет в его входные данные ID плагина.

### Общая схема

```text
Пользователь вводит запрос
        │
        ▼
PluginsActivity.fillItems()
        │
        ├── before_hooked_method
        │       ├── searching = True
        │       └── query = get_private_field(thisObject, "query")
        │
        ▼
Оригинальное построение списка плагинов
        │
        ▼
Plugin.getName()
        │
        ├── оригинальный метод возвращает имя
        │
        └── after_hooked_method
                ├── проверить searching и query
                ├── получить Plugin.getId()
                └── если query in id:
                        param.setResult("Название ID")
        │
        ▼
Штатная обработка списка
        │
        ▼
PluginsActivity.fillItems() завершён
        │
        ▼
after_hooked_method
        ├── searching = False
        └── query = ""
```

### Компоненты и ответственность

| Компонент | Ответственность | Вход | Выход / эффект |
|---|---|---|---|
| `SearchIDPlugin` | Инициализация, регистрация хуков, хранение состояния и cleanup | Lifecycle ExteraGram | Установленные хуки и состояние поиска |
| `FillItemsHook` | Получение запроса до построения списка и очистка после него | Экземпляр `PluginsActivity` | `searching` и `query` в экземпляре плагина |
| `GetNameHook` | Добавление ID к результату имени при совпадении | Экземпляр `Plugin`, исходный результат `getName()` | Изменённый результат `getName()` только во время поиска |
| `_show_error()` | Логирование traceback и показ копируемого Bulletin | Текст ошибки и исключение | Запись в log и сообщение с кнопкой `Copy` |
| SDK `BasePlugin` | Регистрация и снятие Xposed-хуков | Java `Member`, обработчик `MethodHook` | Hook handle `XC_MethodHook.Unhook` |
| SDK `hook_utils` | Поиск Java-класса и чтение приватного поля | Имя класса / Java object и имя поля | Java Class или значение поля |

Главная архитектурная идея — расширить значение, которое уже получает штатная логика, вместо того чтобы заменять само построение списка.

---

## 6. Точки интеграции с ExteraGram / Android

### Реестр точек интеграции

| Класс | Метод / поле | Использование в SearchID | Доступ |
|---|---|---|---|
| `com.exteragram.messenger.plugins.ui.PluginsActivity` | все методы `fillItems` | Точка входа для получения текущего запроса и граница, внутри которой действует флаг поиска | Устанавливаются через `hook_all_methods()` по имени метода |
| `com.exteragram.messenger.plugins.ui.PluginsActivity` | `query` | Приватное поле с поисковым запросом | Читается через `get_private_field()` |
| `com.exteragram.messenger.plugins.Plugin` | `getName` | Метод, результат которого временно изменяется | Метод находится через Reflection |
| `com.exteragram.messenger.plugins.Plugin` | `getId` | Получение ID экземпляра плагина | Вызывается через `param.thisObject` |
| `org.telegram.messenger.R` | `raw.error` | Иконка ошибки для Bulletin | Android resource ID |

### `PluginsActivity.fillItems`

**Назначение в плагине:** получить активный поисковый запрос до обработки списка и сбросить состояние после её окончания.

**Регистрация:**

```python
fill_hooks = self.hook_all_methods(
    activity_class,
    "fillItems",
    FillItemsHook(self),
    priority=100,
)
if not fill_hooks:
    raise RuntimeError("Не удалось установить хуки PluginsActivity.fillItems")
self.hooks.extend(fill_hooks)
```

Плагин больше не получает отдельный `Method` через Chaquopy reflector и не выбирает `fill_methods[0]`. SDK/Xposed устанавливает хук на все методы класса `PluginsActivity` с именем `fillItems`. Это охватывает перегрузки с таким именем, не завися от порядка, в котором Reflection вернул методы.

`hook_all_methods()` возвращает список отдельных `XC_MethodHook.Unhook`. Каждый handle добавляется в `self.hooks` отдельно через `extend()`, чтобы `_unhook_all()` мог снять их по одному.

Точные Java/Kotlin-сигнатуры перегрузок здесь не перечисляются: анализируемый код выбирает их по имени и передаёт выбор SDK/Xposed.

### Поле `PluginsActivity.query`

**Назначение:** источник текущего запроса поиска.

```python
query = get_private_field(param.thisObject, "query")
self.plugin.query = "" if query is None else str(query).strip().lower()
```

`param.thisObject` — экземпляр, на котором был вызван Java-метод. Поэтому плагин читает поле у текущего `PluginsActivity`, а не у класса целиком.

Поле читается, но не изменяется. Плагин не заменяет содержимое поискового поля и не вмешивается во ввод пользователя.

### `Plugin.getName`

**Назначение:** точка, через которую плагин расширяет текст для сопоставления с запросом.

```python
plugin_reflector = plugin_class._chaquopy_reflector
get_name_methods = plugin_reflector.getMethods("getName")
```

Хук работает после оригинального метода, поэтому получает уже готовый результат через `param.getResult()`.

```python
name = param.getResult()
```

При совпадении ID новое значение передаётся через `param.setResult()`:

```python
param.setResult("{} {}".format(name, plugin_id))
```

Плагин не меняет поле с именем объекта. Он изменяет только результат конкретного вызова метода.

### `Plugin.getId`

ID получается у того же Java-объекта, на котором вызван `getName()`:

```python
plugin_id = param.thisObject.getId()
```

Затем значение явно преобразуется в Python-строку:

```python
plugin_id = str(plugin_id)
```

Это позволяет выполнить строковое сравнение независимо от исходного Python-представления Java-значения.

### `R.raw.error`

В коде используется:

```python
jclass("org.telegram.messenger.R").raw.error
```

В исходниках ExteraGram присутствует ресурс `TMessagesProj/src/main/res/raw/error.json`. SDK Bulletin принимает `R.raw` resource ID как Lottie-иконку.

### Ограничения точек интеграции

- `PluginsActivity`, `Plugin`, `fillItems`, `query`, `getName` и `getId` — внутренние элементы приложения, а не независимый публичный API самого плагина.
- Переименование класса, поля или метода может сломать Reflection.
- `fillItems` перехватывается по имени через `hook_all_methods()`, поэтому хук устанавливается на все перегрузки этого метода в классе.
- Для `getName` по-прежнему выбирается первый элемент `get_name_methods`; если у метода появятся перегрузки, нужно проверить порядок и выбранный `Member`.
- Поведение нескольких перегрузок `fillItems`, если они вызывают друг друга, зависит от фактического потока вызовов клиента: каждый перехваченный вызов запускает свой before/after callback.
- Минимальная версия `>=13.2.2` не гарантирует, что внутренние имена сохранятся во всех будущих сборках.

---

## 7. Хуки и Reflection

### Реестр хуков

| № | Класс и метод | Тип хука | Callback | Priority | Назначение | Где снимается |
|---:|---|---|---|---:|---|---|
| 1 | `PluginsActivity.fillItems` | before + after | `FillItemsHook` | `100` | Считать запрос до выполнения и очистить состояние после | `on_plugin_unload()` через `_unhook_all()` |
| 2 | `Plugin.getName` | after | `GetNameHook` | `5` | Изменить результат, если запрос содержится в ID | `on_plugin_unload()` через `_unhook_all()` |

### Получение Java-классов

```python
activity_class = find_class(
    "com.exteragram.messenger.plugins.ui.PluginsActivity"
)
plugin_class = find_class(
    "com.exteragram.messenger.plugins.Plugin"
)
```

`find_class()` из SDK вызывает `jclass(class_name)`. Если класс не удаётся найти, helper записывает ошибку в log и возвращает `None`. Плагин проверяет оба результата и прекращает инициализацию с `RuntimeError`, если хотя бы один класс отсутствует.

### Получение методов через Chaquopy reflector

```python
plugin_reflector = plugin_class._chaquopy_reflector
get_name_methods = plugin_reflector.getMethods("getName")
```

Chaquopy reflector теперь используется только для получения `Plugin.getName`, поскольку `hook_method()` принимает конкретный Java `Member`.

Для `PluginsActivity.fillItems` reflector больше не используется: класс передаётся непосредственно в `hook_all_methods(activity_class, "fillItems", ...)`, а поиск методов по имени выполняет Xposed API внутри SDK.

`_chaquopy_reflector` — внутренний механизм Chaquopy-обёртки, а не отдельный публичный helper из `hook_utils.py`. В SearchID он остаётся только на участке `getName`, где нужен конкретный `Member` для `hook_method()`.

### Почему используется `MethodHook`

Оба обработчика наследуются от `MethodHook`:

```python
class FillItemsHook(MethodHook):
    ...
```

```python
class GetNameHook(MethodHook):
    ...
```

Это позволяет использовать разные фазы одного вызова:

- `before_hooked_method()` получает поле `query` перед оригинальным `fillItems()`;
- `after_hooked_method()` очищает временное состояние после `fillItems()`;
- `after_hooked_method()` у `GetNameHook` читает оригинальный результат `getName()` и при необходимости заменяет его.

Для `getName()` нужен именно `after`, потому что `param.getResult()` содержит результат оригинального метода только после его выполнения. Затем `param.setResult()` заменяет значение, которое будет возвращено вызывающему коду.

### Приоритеты

- `fillItems`: `priority=100`.
- `getName`: `priority=5`.

Эти значения заданы в самом плагине. Первый хук должен подготовить состояние до того, как начнётся обработка списка. Второй изменяет результат после вызова оригинального метода. При взаимодействии с чужими хуками итоговый порядок также зависит от их приоритетов.

### Регистрация

Для `PluginsActivity.fillItems` используется `BasePlugin.hook_all_methods()`:

- принимает Java-класс, имя метода и обработчик;
- создаёт Java callback для `MethodHook`;
- вызывает `XposedBridge.hookAllMethods()`;
- регистрирует полученные handles в `PluginsController`;
- возвращает список отдельных `XC_MethodHook.Unhook` либо `None`, если установить хуки не удалось.

Для `Plugin.getName` используется `BasePlugin.hook_method()`:

- принимает конкретный `java.lang.reflect.Member`;
- вызывает `XposedBridge.hookMethod()`;
- возвращает один `XC_MethodHook.Unhook` либо `None`.

Регистрация `fillItems` в текущей версии:

```python
fill_hooks = self.hook_all_methods(
    activity_class,
    "fillItems",
    FillItemsHook(self),
    priority=100,
)
if not fill_hooks:
    raise RuntimeError("Не удалось установить хуки PluginsActivity.fillItems")
self.hooks.extend(fill_hooks)
```

Здесь нет fallback: `hook_all_methods()` — основной и единственный способ регистрации `fillItems`. Handles из возвращённого списка сохраняются по одному. Хук `getName` затем регистрируется отдельно через `hook_method()` и добавляется через `append()`.

### Снятие хуков

```python
def _unhook_all(self):
    remaining = []
    for hook in reversed(self.hooks):
        try:
            self.unhook_method(hook)
        except Exception as exc:
            remaining.append(hook)
            log(
                "SearchID: не удалось снять хук: "
                f"{exc}\n{traceback.format_exc()}"
            )
    self.hooks = list(reversed(remaining))
```

Хуки снимаются в обратном порядке относительно регистрации. Ошибка снятия одного хука не прерывает попытку снять остальные. Не снятые handles сохраняются в `self.hooks`, а ошибка записывается в log.

В SDK `unhook_method()` вызывает `PluginsController.removeXposedHook()`. SDK также учитывает установленные хуки в контроллере плагинов; SearchID дополнительно хранит собственные handles, чтобы явно управлять снятием и обработать частичную инициализацию.

---

## 8. Подробный разбор ключевой логики

### 8.1. Сохранение поискового состояния

**Задача:** сделать текущий запрос доступным для хука `Plugin.getName()` только во время построения списка.

```python
def before_hooked_method(self, param):
    try:
        self.plugin.searching = True
        query = get_private_field(param.thisObject, "query")
        self.plugin.query = "" if query is None else str(query).strip().lower()
    except Exception as exc:
        self.plugin.searching = False
        self.plugin.query = ""
        _show_error("SearchID: ошибка чтения поиска", exc)
```

**Как работает:**

1. Включает флаг `searching`.
2. Передаёт текущий объект Activity в `get_private_field()`.
3. Читает поле `query`.
4. Преобразует запрос в строку, удаляет пробелы по краям и приводит к нижнему регистру.
5. Если значение равно `None`, записывает пустую строку.
6. При исключении, которое дошло до `except`, сбрасывает оба поля и показывает ошибку.

**Почему используется флаг `searching`:**

Хук `Plugin.getName()` установлен на метод класса `Plugin` и может срабатывать в других местах приложения. Флаг ограничивает изменение результата только теми вызовами, которые происходят в период `fillItems()`.

**Важная особенность SDK helper:**

`get_private_field()` сам перехватывает ошибки Reflection и возвращает `None`. Поэтому если поле `query` не найдено или недоступно, исключение обычно не попадёт во внешний `except` SearchID. В этом случае запрос станет пустым, а хук `getName` не будет менять имена. Для этого helper отсутствие значения и ошибка чтения не различаются по возвращаемому результату.

### 8.2. Изменение результата `getName()`

**Задача:** добавить ID к строке только при совпадении поискового запроса с идентификатором.

```python
def after_hooked_method(self, param):
    try:
        if not self.plugin.searching or not self.plugin.query:
            return

        plugin_id = param.thisObject.getId()
        if plugin_id is None:
            return

        name = param.getResult()
        if name is None:
            return

        plugin_id = str(plugin_id)
        if self.plugin.query in plugin_id.lower():
            param.setResult("{} {}".format(name, plugin_id))
    except Exception as exc:
        _show_error("SearchID: ошибка поиска", exc)
```

**Как работает:**

1. Проверяет, что сейчас выполняется построение списка и запрос не пустой.
2. Получает ID у того же объекта `Plugin`, на котором был вызван `getName()`.
3. Получает оригинальный результат `getName()`.
4. Пропускает изменение, если ID или имя отсутствуют.
5. Преобразует ID в строку.
6. Проверяет вхождение нормализованного запроса в ID в нижнем регистре.
7. При совпадении заменяет результат на `Название ID`.

Если совпадения нет, `param.setResult()` не вызывается и остаётся оригинальный результат `getName()`.

### 8.3. Семантика поиска

Сравнение выполняется как проверка подстроки:

```python
self.plugin.query in plugin_id.lower()
```

Это не точное равенство. Поэтому запрос `cactus` найдёт ID `cactuslib_mini`, а запрос `cactuslib_mini` найдёт его полностью.

Обе стороны сравнения приводятся к нижнему регистру. В начале запроса дополнительно удаляются пробелы по краям.

Запрос не разбивается на отдельные слова. Например, строка `cactus mini` будет искаться как одна подстрока ID и не эквивалентна одновременному поиску двух частей.

### 8.4. Почему не реализуется собственная фильтрация

SearchID не получает коллекцию установленных плагинов, не строит `UItem` и не повторяет код менеджера.

Вместо этого он меняет данные, которые использует штатный код. Такой подход уменьшает объём вмешательства: плагину не нужно поддерживать собственную копию сортировки, создания строк и обновления списка.

При этом подмена выполняется на время вызова `getName()` внутри активного поиска. Если другой код вызывает `getName()` вне этого контекста, имя остаётся исходным.

Важное следствие: во время активного `fillItems()` строку `Название ID` получает любой внутренний потребитель результата `getName()` в этом вызове, не только фильтр. Точное использование результата внутри конкретной реализации `PluginsActivity` зависит от кода клиента.

### 8.5. Регистрация `fillItems` на все перегрузки

**Задача:** не выбирать одну перегрузку вручную, а устанавливать хук на все методы `fillItems` класса `PluginsActivity`.

```python
fill_hooks = self.hook_all_methods(
    activity_class,
    "fillItems",
    FillItemsHook(self),
    priority=100,
)
if not fill_hooks:
    raise RuntimeError("Не удалось установить хуки PluginsActivity.fillItems")
self.hooks.extend(fill_hooks)
```

SDK передаёт класс и имя метода в `XposedBridge.hookAllMethods()`. Результат — список handles, по одному на установленный хук. Плагин сохраняет каждый handle отдельно.

`Plugin.getName` по-прежнему находится через `plugin_class._chaquopy_reflector.getMethods("getName")`, после чего первый найденный `Member` передаётся в `hook_method()`.

**Практический эффект:** регистрация `fillItems` больше не зависит от `fill_methods[0]` и от успешной установки хука на один выбранный `Member`. Все перегрузки с этим именем получают один и тот же `FillItemsHook` с приоритетом `100`.

### 8.6. Частичная регистрация и откат

**Задача:** не оставлять хуки `fillItems` установленными, если регистрация `getName` завершается ошибкой.

После успешной регистрации `fillItems` все возвращённые handles сразу добавляются в `self.hooks` по одному через `extend()`. Если регистрация `getName` завершается ошибкой или возвращает `None`, выполнение попадает в общий `except`:

```python
except Exception as exc:
    self._unhook_all()
    _show_error("SearchID: ошибка загрузки", exc)
```

`_unhook_all()` пытается снять все handles, которые уже были сохранены. Таким образом, сбой регистрации `getName` не оставляет ранее установленные хуки `fillItems` без попытки очистки.

Если снятие какого-либо хука тоже завершается ошибкой, его handle остаётся в списке для последующей попытки снятия при вызове cleanup.

---

## 9. Состояние и структура данных

| Переменная | Тип | Начальное значение | Где изменяется | Где читается | Срок жизни |
|---|---|---|---|---|---|
| `self.hooks` | `list` из hook handles | `[]` | `on_plugin_load()`, `_unhook_all()` | `_unhook_all()` | Время жизни экземпляра плагина |
| `self.searching` | `bool` | `False` | `FillItemsHook.before_hooked_method()`, `after_hooked_method()`, `on_plugin_unload()` | `GetNameHook.after_hooked_method()` | Один цикл построения списка |
| `self.query` | `str` | `""` | `FillItemsHook.before_hooked_method()`, `after_hooked_method()`, `on_plugin_unload()` | `GetNameHook.after_hooked_method()` | Один цикл построения списка |

### Где хранится источник истины

Сам поисковый запрос хранится в поле `PluginsActivity.query`. SearchID его не меняет, а только читает.

`self.query` — временная копия нормализованного значения, нужная для работы второго хука. Она не является постоянным хранилищем и не переживает завершение `fillItems()`.

### Хранение hook handles

В `self.hooks` хранятся отдельные объекты `XC_MethodHook.Unhook`: несколько handles, возвращённых `hook_all_methods()` для перегрузок `fillItems`, и один handle, возвращённый `hook_method()` для `getName`.

Это не названия методов и не экземпляры `MethodHook`, а handles, необходимые для снятия каждого зарегистрированного хука.

### Синхронизация

В SearchID нет `Lock`, `threading.local()` или другого механизма синхронизации. `searching` и `query` — общие поля экземпляра плагина.

Текущая логика рассчитана на последовательный сценарий: один из перехваченных методов `fillItems()` устанавливает состояние, затем внутри его выполнения вызываются `getName()`, после чего after-callback соответствующего `fillItems()` очищает состояние.

Поскольку хук установлен на все перегрузки, важно учитывать возможные вложенные вызовы между перегрузками. Если один перехваченный `fillItems()` вызывает другой, after-callback внутреннего вызова может очистить общее состояние до завершения внешнего. В плагине нет отдельного счётчика глубины или стека контекста вызовов; это ограничение общей модели `searching`/`query`, а не ошибка регистрации handles.

### Постоянные настройки и кэш

Плагин не использует:

- `get_setting()` / `set_setting()`;
- файлы настроек;
- базы данных;
- кэш;
- сетевую синхронизацию.

---

## 10. UI и взаимодействие с пользователем

Отдельного UI у SearchID нет. Пользователь продолжает использовать штатный экран менеджера плагинов и его поисковое поле.

Изменяется только результат вызова `Plugin.getName()` во время обработки списка. Плагин не создаёт собственные строки интерфейса и не вызывает обновление адаптера самостоятельно.

### Bulletin при ошибках

Для ошибок используется:

```python
BulletinHelper.show_with_button(
    message,
    jclass("org.telegram.messenger.R").raw.error,
    "Copy",
    on_click=lambda: copy_to_clipboard(trace),
)
```

Параметры:

- `message` — короткое сообщение об ошибке;
- `R.raw.error` — Lottie-иконка ошибки;
- `"Copy"` — подпись кнопки;
- `on_click` — копирование полного traceback.

Реализация `show_with_button()` в SDK формирует Bulletin и передаёт задачу в `run_on_ui_thread()`. Поэтому самостоятельно вызывать `run_on_ui_thread()` вокруг этого метода не требуется.

---

## 11. Потоки и асинхронность

В коде SearchID нет собственных потоков, executor, очередей, таймеров или сетевых задач.

Хуки выполняют небольшую синхронную работу в потоке, в котором вызывается перехваченный Java-метод. Плагин не переключает `fillItems()` или `getName()` на другой поток.

Показ Bulletin отличается: `BulletinHelper.show_with_button()` передаёт создание и показ уведомления в `run_on_ui_thread()`. Это необходимо, поскольку работа с Android UI должна выполняться в UI-thread.

Кнопка `Copy` вызывает `copy_to_clipboard(trace)`. SDK обращается к Android Clipboard API; при успешном копировании дополнительно показывает Bulletin о копировании.

---

## 12. Сеть, файлы и внешние зависимости

### Сетевые запросы

Не используются.

SearchID не отправляет поисковые запросы, ID плагинов, сообщения или другие данные на внешние серверы.

### Файлы и хранилище

Плагин не читает и не записывает пользовательские файлы. Постоянное хранилище не используется.

### Импорты и зависимости

| Импорт | Назначение |
|---|---|
| `traceback` | Формирование полного traceback |
| `java.jclass` | Получение Java-класса `R` |
| `BasePlugin` | Базовый класс плагина и lifecycle |
| `MethodHook` | Реализация before/after callbacks |
| `android_utils.copy_to_clipboard` | Копирование traceback |
| `android_utils.log` | Запись диагностического сообщения |
| `hook_utils.find_class` | Поиск классов приложения |
| `hook_utils.get_private_field` | Чтение приватного поля `query` |
| `ui.bulletin.BulletinHelper` | Показ сообщения об ошибке с кнопкой `Copy` |

Все импорты относятся к стандартному Python runtime или Plugin SDK ExteraGram. В `__requirements__` нет сторонних pip-зависимостей.

---

## 13. Нативный код и бинарные ресурсы

Собственных `.so`, DEX, JAR, Base64-бинарников или shader-файлов в плагине нет.

Используемый `R.raw.error` — ресурс самого приложения ExteraGram, а не встроенный бинарный файл SearchID. В публичном дереве ETG присутствует `TMessagesProj/src/main/res/raw/error.json`.

---

## 14. Ошибки, диагностика и логи

### Стратегия обработки ошибок

| Участок | Что может пойти не так | Реакция плагина |
|---|---|---|
| `find_class()` | Класс не найден или не загружается | SDK пишет сообщение в log и возвращает `None`; SearchID прекращает регистрацию |
| Получение reflector / методов | У класса нет ожидаемого reflector или имя метода не найдено | Исключение попадает в `on_plugin_load()`; вызывается cleanup и `_show_error()` |
| `hook_method()` | Xposed не смог установить хук | SDK пишет ошибку и возвращает `None`; SearchID выбрасывает `RuntimeError` |
| Чтение `query` | Поле отсутствует или Reflection завершилась ошибкой | SDK helper возвращает `None`; SearchID использует пустой запрос |
| Обработка `getName()` | Ошибка получения ID или результата | `_show_error()` записывает traceback и показывает Bulletin |
| Снятие хуков | `unhook_method()` завершился ошибкой | Ошибка записывается в log; handle остаётся в `self.hooks` |
| Показ Bulletin | Ошибка возникла при вызове helper | Синхронное исключение логируется отдельным сообщением |

### Формирование traceback

```python
trace = "".join(
    traceback.format_exception(
        type(exc),
        exc,
        exc.__traceback__,
    )
)
```

Traceback записывается через `android_utils.log()` и передаётся callback кнопки `Copy`.

### Логирование

В текущей версии используется:

```python
from android_utils import copy_to_clipboard, log
```

Функция `android_utils.log()` существует в SDK, но в документации помечена как legacy: она пишет сообщения на уровне debug и не даёт отдельного фильтруемого тега `plugin.SearchID`.

Новый SDK logger через `self.logger` предпочтительнее для новых реализаций. Здесь используется модульная функция, потому что `_show_error()` находится на уровне модуля и не получает экземпляр `SearchIDPlugin`.

### Bulletin и копирование

`show_with_button()` принимает текст, resource ID, подпись кнопки и callback. Реализация SDK создаёт Bulletin на UI-thread.

При нажатии `Copy` вызывается `copy_to_clipboard(trace)`. В SDK эта функция вызывает `AndroidUtilities.addToClipboard(text)`, а при успешном копировании показывает отдельное уведомление `show_copied_to_clipboard()`.

Обёртка `try/except` в `_show_error()` ловит ошибку, если сам вызов `show_with_button()` завершился исключением до возврата. Поскольку создание Bulletin планируется через UI-thread helper, исключение, возникшее уже внутри отложенной UI-задачи, не обязательно попадёт в этот внешний `except`.

---

## 15. Lifecycle и очистка ресурсов

### `__init__`

```python
def __init__(self):
    super().__init__()
    self.hooks = []
    self.searching = False
    self.query = ""
```

Создаётся базовый экземпляр `BasePlugin`, пустой список handles и начальное состояние поиска.

### `on_plugin_load`

Порядок действий:

1. Находит `PluginsActivity`.
2. Находит `Plugin`.
3. Получает через reflector список методов `getName`.
4. Проверяет, что класс `Plugin` и список `getName` доступны.
5. Вызывает `hook_all_methods()` для всех методов `PluginsActivity` с именем `fillItems` и приоритетом `100`.
6. Проверяет, что SDK вернул непустой список handles, и добавляет каждый handle в `self.hooks`.
7. Устанавливает хук `Plugin.getName` с приоритетом `5`.
8. Сохраняет handle `getName` в `self.hooks`.

Если любой этап завершается исключением, выполняется `_unhook_all()`, затем показывается сообщение об ошибке.

### Во время работы

Постоянно активны хук `getName` и хуки всех найденных перегрузок `fillItems`. `searching` и `query` заполняются только вокруг выполнения `fillItems()` и очищаются в after callback.

### `on_plugin_unload`

```python
def on_plugin_unload(self):
    self._unhook_all()
    self.searching = False
    self.query = ""
```

При выгрузке:

- выполняется попытка снять все сохранённые хуки;
- ошибки снятия отдельных хуков не прекращают очистку остальных;
- handles, которые не удалось снять, остаются в `self.hooks`;
- состояние поиска сбрасывается.

SDK дополнительно хранит hook handles в `PluginsController`. SearchID вызывает ручной `unhook_method()` для явной очистки и сохраняет не снятые handles для повторной попытки при следующем cleanup.

### Частичная инициализация

Если `hook_all_methods()` установил хуки для нескольких перегрузок, все возвращённые handles сразу сохраняются в `self.hooks`. Если последующая регистрация `getName` завершается ошибкой или возвращает `None`, общий обработчик ошибки вызывает `_unhook_all()` и пытается снять каждый ранее установленный handle.

Это предотвращает оставление части хуков `fillItems` активной после неудачной регистрации `getName`.

---

## 16. Совместимость и различия версий

### Метаданные 1.0.2

```python
__version__ = "1.0.2"
__app_version__ = ">=13.2.2"
```

`__app_version__` — актуальное поле метаданных SDK для требования к версии приложения. Документация также указывает, что старое `__min_version__` принимается как legacy alias для `__app_version__ = ">=..."`.

В плагине не указано `__sdk_version__`, поэтому загрузчик не получает отдельное ограничение минимальной версии SDK из метаданных SearchID.

### Отличия от 1.0.1

| Участок | 1.0.1 | 1.0.2 |
|---|---|---|
| Версия | `1.0.1` | `1.0.2` |
| Минимальная версия приложения | `__min_version__ = "12.5.1"` | `__app_version__ = ">=13.2.2"` |
| Логирование ошибок | Только Bulletin | Добавлен `android_utils.log()` с traceback |
| Ошибка показа Bulletin | Полностью игнорировалась | Записывается отдельная ошибка в log |
| Ошибка чтения запроса | Сбрасывался `searching` | Сбрасываются `searching` и `query` |
| Приведение ID к строке | Выполнялось внутри `str()` в условии | Выполняется явно перед сравнением |
| Проверка класса | Проверка через truthiness | Проверка `is None` |
| Проверка метода | Проверка списка и длины | Проверка пустого списка |
| Регистрация `fillItems` | Сначала `hook_method()` для первого найденного метода; при неудаче — `hook_all_methods()` | `hook_all_methods()` используется сразу как основной способ; все handles сохраняются отдельно |
| Частичная регистрация | Не было общего отката | При ошибке вызывается `_unhook_all()` |
| Снятие хуков | Цикл с пустым `except` и очистка списка | Централизованный `_unhook_all()`, обратный порядок, логирование ошибки, сохранение не снятых handles |
| Поиск по ID | Подстрока без учёта регистра | Семантика поиска сохранена |

Основное поведение поиска не менялось: `fillItems` сохраняет запрос, `getName` проверяет ID, при совпадении возвращает `Название ID`.

### Особенности совместимости

- Пакет `com.exteragram.messenger...` привязывает реализацию к ExteraGram.
- Приватное поле `query` и внутренняя реализация методов могут меняться между версиями приложения.
- `__app_version__` задаёт нижнюю границу, но не проверяет наличие нужных Java-методов.
- `_chaquopy_reflector` — внутренняя деталь Python/Java interop, а не публичный helper из `hook_utils`.
- `hook_all_methods()` снижает зависимость от порядка Reflection и покрывает перегрузки `fillItems`, но при изменении имени метода или класса регистрация всё равно завершится неудачно.
- Для `getName` по-прежнему важно проверять, что выбранный через `getMethods()[0]` метод соответствует ожидаемому методу без аргументов.

---

## 17. Производительность и использование ресурсов

Плагин добавляет два метода-перехватчика:

- один вызов до и один после `fillItems()`;
- дополнительную проверку на каждом вызове `Plugin.getName()`, пока установлен хук.

В обработчике `getName()` выполняется несколько простых операций: проверка двух полей состояния, получение ID, получение оригинального результата, преобразование ID в строку и сравнение подстроки.

Reflection выполняется при загрузке плагина, а не для каждого элемента списка. Сетевой активности, файловых операций, декодирования изображений, больших буферов и собственного кэша нет.

Фактические замеры времени или памяти в исходнике отсутствуют. Код не содержит отдельного механизма измерения производительности.

---

## 18. Безопасность и приватность

Плагин взаимодействует только с локальным менеджером плагинов и его объектами.

Он читает:

- поисковый запрос из `PluginsActivity`;
- имя и ID объектов `Plugin`.

Он не читает сообщения, контакты, данные аккаунтов, содержимое чатов или токены. Никакие данные не отправляются на сервер.

### Поверхности риска

1. **Reflection к приватному полю.** `query` может быть переименовано или удалено в новой версии клиента. В этом случае SDK helper вернёт `None`, а поиск по ID не сработает.
2. **Reflection к внутренним методам.** `fillItems` и `getName` — внутренняя реализация клиента, их имена и набор перегрузок могут измениться.
3. **Изменение результата метода.** Во время `fillItems()` другие потребители `getName()` также могут получить строку с добавленным ID. Изменение не записывается в объект плагина постоянно.
4. **Логи.** Traceback может содержать имена классов, методов и пути выполнения. Плагин не добавляет в лог содержимое сообщений или данные аккаунта.
5. **Callback копирования.** По нажатию кнопки пользователь сам инициирует копирование traceback в буфер обмена.

Для SearchID не требуется серверная авторизация, хранение секретов, файловые разрешения или отдельная защита сетевых запросов.

---

## 19. Паттерны, которые можно переиспользовать

### 19.1. Расширение штатного поиска через getter

**Когда применять:** когда штатный фильтр уже использует строковое поле или getter, а нужно добавить ещё один поисковый атрибут без создания собственного списка.

**Реализация в SearchID:**

1. Найти метод, внутри которого строится список и вызывается getter.
2. Получить условие поиска до запуска этого метода.
3. Перехватить getter после оригинального вызова.
4. При совпадении вернуть составное значение, включающее дополнительный атрибут.
5. Сбросить контекст поиска после завершения основного метода.

**Почему полезно:** плагин повторно использует штатную фильтрацию и не копирует внутреннюю структуру UI.

**Ограничения:** нужно убедиться, что getter действительно используется нужным фильтром. Возвращаемое значение может одновременно использоваться для отображения, сортировки или других операций.

### 19.2. Ограничение действия общего хука контекстным флагом

**Когда применять:** если метод перехватывается глобально, но модификация нужна только во время одного конкретного сценария.

**Реализация в SearchID:**

```python
if not self.plugin.searching or not self.plugin.query:
    return
```

Флаг и запрос устанавливаются в before-хуке метода-обёртки и сбрасываются в after-хуке.

**Ограничения:** общий флаг подходит для последовательного синхронного сценария. Для вложенных или конкурентных вызовов потребуется отдельное состояние на вызов или другой контекстный механизм.

### 19.3. Чтение приватного поля через SDK helper

**Когда применять:** если необходимое состояние находится в private-поле Java-объекта и публичного getter нет.

**Реализация в SearchID:**

```python
query = get_private_field(param.thisObject, "query")
```

**Почему полезно:** helper проходит по иерархии классов, получает declared field, вызывает `setAccessible(True)` и читает значение.

**Ограничения:** это хрупкая интеграция. Helper возвращает `None`, если поле отсутствует или чтение завершилось ошибкой; вызывающий код должен учитывать это. Имя поля может измениться после обновления клиента или обфускации.

### 19.4. Сохранение handles и откат частичной регистрации

**Когда применять:** когда плагин устанавливает несколько независимых хуков, и загрузка считается успешной только после регистрации всех необходимых точек.

**Реализация в SearchID:**

- `hook_all_methods()` возвращает список handles для `fillItems`;
- каждый handle сразу добавляется в `self.hooks` через `extend()`;
- handle `getName`, возвращённый `hook_method()`, добавляется через `append()`;
- при последующей ошибке вызывается общий cleanup;
- handles снимаются в обратном порядке;
- не снятые handles не теряются, а остаются в списке;
- ошибки cleanup записываются в log.

**Почему полезно:** это предотвращает оставление части функциональности активной после неудачной инициализации.

**Ограничения:** cleanup должен быть повторно вызываемым, а ошибки снятия не должны незаметно удалять handle из списка.

### 19.5. Копируемая диагностика через Bulletin

**Когда применять:** когда пользователю нужно сообщить об ошибке и одновременно дать возможность отправить разработчику полный traceback.

**Реализация в SearchID:**

- собрать traceback;
- записать его в log;
- показать короткое сообщение через `BulletinHelper.show_with_button()`;
- назначить callback кнопке `Copy`;
- скопировать traceback через `copy_to_clipboard()`.

**Ограничения:** не помещать токены, пароли или приватные данные в traceback и текст Bulletin. `show_with_button()` уже переключает создание уведомления на UI-thread.

### 19.6. Сохранение handles, возвращаемых `hook_all_methods()`

**Когда применять:** когда нужно перехватить все перегрузки метода с одним именем и SDK возвращает несколько handles.

**Реализация в SearchID:**

```python
fill_hooks = self.hook_all_methods(
    activity_class,
    "fillItems",
    FillItemsHook(self),
    priority=100,
)
if not fill_hooks:
    raise RuntimeError("Не удалось установить хуки PluginsActivity.fillItems")

self.hooks.extend(fill_hooks)
```

Список проверяется до сохранения, а затем каждый handle добавляется в общий список отдельно. Это важно для корректной работы `_unhook_all()`, который вызывает `unhook_method()` для каждого handle, а не для вложенного списка.

---

## 20. Полная схема работы плагина

```text
on_plugin_load()
    │
    ├── find_class(PluginsActivity)
    ├── find_class(Plugin)
    │
    ├── получить reflector Plugin
    ├── получить getName
    │
    ├── hook_all_methods(PluginsActivity, "fillItems", FillItemsHook, priority=100)
    │       └── сохранить каждый handle отдельно
    │
    ├── hook_method(getName, GetNameHook, priority=5)
    │       └── сохранить handle
    │
    └── при ошибке:
            ├── _unhook_all()
            └── _show_error()
                    │
                    ▼
Пользователь вводит поисковый запрос
    │
    ▼
PluginsActivity.fillItems()
    │
    ├── before:
    │       ├── searching = True
    │       ├── прочитать поле query
    │       └── сохранить strip().lower()
    │
    ▼
Оригинальная логика fillItems()
    │
    ▼
Plugin.getName()
    │
    ├── оригинальный getName() возвращает имя
    │
    └── after:
            ├── проверить searching
            ├── проверить непустой query
            ├── получить getId()
            ├── получить оригинальный результат
            └── если query содержится в ID:
                    └── param.setResult("Имя ID")
    │
    ▼
Штатная обработка списка плагинов
    │
    ▼
fillItems() завершён
    │
    └── after:
            ├── searching = False
            └── query = ""
    │
    ▼
on_plugin_unload()
    │
    ├── снять сохранённые handles в обратном порядке
    ├── сохранить handles, которые не удалось снять
    ├── searching = False
    └── query = ""
```
