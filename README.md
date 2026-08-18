# LavandaBot

Оркестрация сети ботов Lavanda (доски объявлений в Telegram): единый `docker-compose.yml`
для запуска всех сервисов, которые раньше поднимались вручную через `screen`.

Этот репозиторий содержит **только** compose-файл, `.env.example` и README. Код самих
сервисов лежит в отдельных репозиториях и клонируется рядом (см. «Быстрый старт»).

## Архитектура

| Сервис | Репозиторий | Стек | БД | Роль |
|---|---|---|---|---|
| `postgres` | — | Postgres 16 | — | БД для HelperChatBotManager |
| `helper-bot` | [HelperChatBotManager](https://github.com/SaveNTech/HelperChatBotManager) | aiogram | Postgres | Модерация групп-классифайдов: проверяет оплаченный доступ (`ads_limit`) перед тем как разрешить пользователю написать объявление, приём оплат через ЮKassa, стоп-слова, автоудаление отказных сообщений |
| `parser-bot` | [TelegramParser](https://github.com/SaveNTech/TelegramParser) — `main_bot.py` | aiogram | SQLite (`tg_parser.db`) | Админ-панель: группы-источники, целевые группы по городам, категории/ключевые слова |
| `parser-userbot` | [TelegramParser](https://github.com/SaveNTech/TelegramParser) — `main_userbot_linux.py` | pyrogram (`pyrotgfork`) | SQLite (`tg_parser.db`, общий с `parser-bot`) | Слушает группы-источники под юзер-аккаунтом, фильтрует по стоп-словам и репостит в целевую группу нужного города |
| `ads-bot` | [LavandaAds](https://github.com/SaveNTech/LavandaAds) | aiogram | — | Статичный бот с прайсом на рекламу в сети чатов, без БД |

`parser-bot` и `parser-userbot` собираются из одного образа (`TelegramParser/Dockerfile`),
различается только команда запуска.

## Быстрый старт

```bash
git clone <url-этого-репозитория> LavandaBot
cd LavandaBot
git clone https://github.com/SaveNTech/HelperChatBotManager
git clone https://github.com/SaveNTech/TelegramParser
git clone https://github.com/SaveNTech/LavandaAds
```

### 1. Локальные конфиги, которые не хранятся в git

Оба сервиса гитигнорят файл со списком Telegram ID админов — его нужно создать руками:

`HelperChatBotManager/handlers/config.py`:
```python
tg_id_list = [123456789]  # твой Telegram ID и ID других админов
```

`TelegramParser/bot/config.py`:
```python
tg_id_list = [123456789]
```

Без этих файлов сборка образов упадёт с понятной ошибкой на этапе `docker compose build`
(намеренно — иначе контейнер падал бы в рантайме молча).

### 2. Переменные окружения

```bash
cp .env.example .env
```

и заполнить:

| Переменная | Для чего |
|---|---|
| `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` | БД HelperChatBotManager |
| `HCM_BOT_TOKEN` | токен бота-модератора чатов |
| `HCM_ERROR_CHAT_ID` | чат, куда падают ошибки HelperChatBotManager |
| `HCM_YOOTOKEN` | provider token ЮKassa для приёма оплат |
| `TP_BOT_TOKEN` | токен админ-бота парсера |
| `TP_API_ID` / `TP_API_HASH` | Telegram API-креды юзер-аккаунта (my.telegram.org) |
| `TP_SESSION_NAME` | имя файла сессии pyrogram (по умолчанию `parser_session`) |
| `TP_ERROR_CHAT` | чат, куда падают ошибки юзербота |
| `ADS_BOT_TOKEN` | токен бота с прайсом на рекламу |

### 3. Сборка и первый запуск

```bash
docker compose build
```

Юзербот парсера — это обычный Telegram-аккаунт, при первом запуске pyrogram спросит номер
телефона, код подтверждения и, если включена, пароль 2FA. Это нужно сделать один раз
интерактивно, до того как сервис уйдёт в фон:

```bash
docker compose run --rm parser-userbot
```

После успешного логина файл сессии сохранится в volume `parser_data` и переживёт
пересоздание контейнера. Дальше — обычный запуск в фоне:

```bash
docker compose up -d
```

### Данные

- `pg_data` — данные Postgres.
- `parser_data` — общий volume `parser-bot`/`parser-userbot`, смонтирован как `/data`:
  `tg_parser.db` (SQLite) и `sessions/` (сессия pyrogram).

## Что было исправлено (2026-08-17)

При переносе на Docker вскрылось несколько багов в текущем коде сервисов:

- **TelegramParser падал на Linux при старте.** `userbot/client.py` содержал
  неиспользуемый импорт `from msilib.text import dirname` — `msilib` есть только в Windows
  и убран из stdlib начиная с Python 3.13. Импорт был мёртвым кодом (нигде не
  использовался), просто удалён.
- **HelperChatBotManager иногда пропускал сообщения неоплативших пользователей.**
  В `handlers/groups.py` часть кода (`get_groups()`, чтение `message.from_user.id`)
  выполнялась до `try/except`. Любая ошибка БД или сообщение без `from_user`
  (например, отправленное от имени привязанного канала) роняли хэндлер необработанным
  исключением до вызова `bot.delete_message()` — сообщение оставалось висеть в чате.
  Всю логику перенесли внутрь `try/except`, заодно убрали неиспользуемый вызов
  `get_groups()` (его результат нигде не читался — фильтр по группам был закомментирован).
- **HelperChatBotManager не создавал таблицы в Postgres.** `create_db()` был импортирован,
  но вызов закомментирован в `main.py` — на чистой БД (как раз наш случай с Docker)
  первый же запрос падал бы `UndefinedTableError`. Раскомментировано.
- `req.txt` в обоих репозиториях лежал в UTF-16 (артефакт `pip freeze` на Windows) —
  пересохранён в UTF-8 для надёжной сборки образов.
- **`MAIN_CHAT` убран.** Переменная читалась на уровне модуля (`int(os.getenv("MAIN_CHAT"))`)
  и без неё юзербот падал при импорте, хотя значение нигде не использовалось — реальная
  логика назначения целевой группы полностью на БД (город источника → `target_group.group_id`).
  Мёртвый код (`chat_id`, `save_album_after_delay`) удалён из `group_handler.py`.
- **Добавлен `LavandaAds`.** В репозитории не было ни `requirements.txt`, ни `Dockerfile` —
  добавлены. Заодно убраны случайно закоммиченные `__pycache__/*.pyc` и `.idea/`.
