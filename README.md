# for-vibecoders

RU:
этот репозиторий создан специально для людей, что хотят вайбкодингом делать плагины для **ExteraGram 12.5.1+ и AyuGram на его основе**
репо содержит в себе анализы реальных плагинов, сами файлы плагинов, описания где что работает, актуальные файлы типизации SDK (type stubs 1.4.6.x), гайды по Elyx, хукам, мультиаккаунтности и ссылки на полезные источники
также вы можете написать мне в тг ([@jaxbastard](https://t.me/jaxbastard)), чтобы я добавил ваш плагин с его анализом

EN:
this repo is built specifically for people who want to do vibe coding to make plugins for **ExteraGram 12.5.1+ and AyuGram based on it**
repo contains analyses of real plugins, the plugin files themselves, descriptions of what works where, updated SDK type stubs (`.pyi` 1.4.6.x), guides on Elyx, hooks, multi-account handling, links to useful sources, etc
all documentation is available in Russian and English (see [`for-agent/`](https://github.com/corerudo/for-vibecoders/tree/main/for-agent) and [`stubs/`](https://github.com/corerudo/for-vibecoders/tree/main/stubs))
you can also hit me up on tg ([@jaxbastard](https://t.me/jaxbastard)) and i'll add your plugin with its analysis

---

## 📁 RU: структура / EN: structure

| папка, файл | RU: что внутри | EN: what's inside |
|-------------|----------------|-------------------|
| [`allpg/`](https://github.com/corerudo/for-vibecoders/tree/main/allpg) | папки плагинов, каждая содержит .plugin и <n>-analysis.md | plugin folders, each contains .plugin and <n>-analysis.md |
| [`stubs/`](https://github.com/corerudo/for-vibecoders/tree/main/stubs) | официальные файлы типизации SDK (.pyi) для автодополнения и подсветки типов в IDE | official SDK type stubs (.pyi) for IDE autocompletion and type checking |
| [`for-agent/`](https://github.com/corerudo/for-vibecoders/tree/main/for-agent) | Skill для ИИ-агентов с правилами разработки плагинов, требованиями актуального SDK (1.4.4.3+ / 1.4.6.x), справочниками по API, хукам, Elyx, многоаккаунтности, Intents, UI и Java-интеграции | Skill for AI agents with plugin development rules, latest SDK requirements (1.4.4.3+ / 1.4.6.x), and API references for hooks, Elyx, multi-account, Intents, UI, and Java integration |
| [`plugins-index.md`](https://github.com/corerudo/for-vibecoders/blob/main/plugins-index.md) | перепись всех имеющихся в репозитории плагинов и анализов | full inventory of all plugins and analyses in the repository |
| [`repo-plus-context7.md`](https://github.com/corerudo/for-vibecoders/blob/main/repo-plus-context7.md) | почему репо + context7 лучше использовать вместе | why repo + context7 work better together |
| [`context7.json`](https://github.com/corerudo/for-vibecoders/blob/main/context7.json) | настройки индексации репо в context7 | config for indexing this repo in context7 |

---

## 🔗 RU: полезные ссылки / EN: useful links

RU:
- [документация ExteraGram plugins](https://plugins.exteragram.app/docs) — официальная дока по plugin API
- [exteraSquad/plugins-pysdk-builds](https://github.com/exteraSquad/plugins-pysdk-builds) — официальный репозиторий сборок Python SDK и файлов типизации (1.4.6.x)
- [исходники Telegram для Android](https://github.com/DrKLO/Telegram/tree/009e97356f966bb81eceba113d210230bf383122) — база для понимания внутренностей
- [канал с плагинами](https://t.me/jxbplug) — канал с плагинами. владелец: [@jaxbastard](https://t.me/jaxbastard)
- [ExteraGram sources](https://github.com/exteraSquad/exteraGram) — содержит сурсы ExteraGram
 
EN:
- [ExteraGram plugins docs](https://plugins.exteragram.app/docs) — official plugin API documentation
- [exteraSquad/plugins-pysdk-builds](https://github.com/exteraSquad/plugins-pysdk-builds) — official repository for Python SDK builds and type stubs (1.4.6.x)
- [Telegram for Android source](https://github.com/DrKLO/Telegram/tree/009e97356f966bb81eceba113d210230bf383122) — base for understanding internals
- [plugins channel](https://t.me/jxbplug) — channel with plugins. owner: [@jaxbastard](https://t.me/jaxbastard)
- [ExteraGram sources](https://github.com/exteraSquad/exteraGram) — contains ExteraGram sources

---

<sub>apache-2.0 · ExteraGram 12.5.1+ · SDK 1.4.4.3+ / 1.4.6.x · AyuGram based on it</sub>
