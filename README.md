# Avito Hybrid Scraper

Гибридное приложение для сбора данных с Avito с аутентификацией через браузер (undetected-chromedriver) и высокоскоростным сбором данных через мобильное API (requests). Включает веб‑приложение (FastAPI + Uvicorn) для визуализации статистики и управления поисковыми запросами, а также Telegram‑бота для доставки данных и алертов.

Внимание: соблюдайте условия использования Avito и применяйте инструмент только для законных целей. Мобильное API не является публично задокументированным и может меняться без уведомления.

## Архитектура

Компоненты:
- AuthManager: выполняет вход через браузер, получает и валидирует cookies, хранит их на диске.
- Scraper: использует cookies для запросов к m.avito.ru (мобильное API), собирает списки и детали объявлений, опционально телефон.
- Storage: запись данных в CSV/JSON/JSONL и (для веб/UI и бота) в SQLite; ротация файлов.
- Web (FastAPI + Uvicorn):
  - Дашборд KPI и графики.
  - Таблица последних объявлений, фильтры, экспорт.
  - Управление поисковыми запросами (создание/редактирование/старт/пауза/стоп).
  - REST API: /api/stats/summary, /api/stats/timeseries, /api/items, /api/scrape/status, /api/searches.
- Telegram‑бот: рассылка новых объявлений, сводок, алертов; команды /summary, /latest, /queries, /status и т.д.
- RateLimiter/RetryPolicy: ограничение частоты запросов, экспоненциальные ретраи с jitter.

Данные:
- Файловый вывод (CSV/JSON/JSONL) — итоговые выгрузки.
- SQLite — рабочее хранилище для веб/бота: searches, items, search_items, stats_timeseries, scrape_status.

## Логика работы (поток данных)

1) Аутентификация
- При старте проверяется наличие сохранённых cookies. Если валидны — используется существующая сессия.
- При отсутствии/нелегитимности cookies запускается headless/GUI Chrome через undetected‑chromedriver, выполняется вход. При необходимости ожидается ручной ввод CAPTCHA. Cookies сохраняются на диск.

2) Получение поисковых запросов
- Источник запросов выбирается конфигурацией:
  - urls_file: чтение списка URL из файла.
  - db: чтение активных запросов (searches) из SQLite, управляемых через веб‑интерфейс.

3) Сбор данных
- Для каждого запроса/URL выполняются вызовы мобильного API Avito с учётом пагинации и лимитов.
- Извлекаются базовые поля объявления и, при необходимости, детали и номер телефона.
- Данные нормализуются и валидируются; сохраняются в файловые форматы и/или SQLite.
- Для режима db устанавливается привязка item_id ↔ search_id; прогресс и чекпоинты ведутся по каждому запросу отдельно.

4) Агрегирование и статистика
- Фоново/периодически формируются агрегаты и временные ряды; KPI доступны в веб‑приложении.

5) Доставка и оповещения
- В Telegram отправляются новые объявления (с батчингом) и периодические сводки; при инцидентах (401/403/429/ошибки) приходят алерты.

## Конфигурация

Поддерживается .ini/.json и переменные окружения (ENV приоритетнее). Основные параметры:
- auth: login, password, headless, user_data_dir, cookies_path, cookies_ttl_hours
- input: urls_file (в файловом режиме)
- searches: storage=sqlite|file, sqlite_path, default_filters, max_concurrency_per_search, schedule_cron/interval
- output: format=csv|json|jsonl, path, rotate_filesize_mb, compress, encoding
- http: user_agent, timeout_sec, max_retries, retry_backoff_base, retry_backoff_max
- rate_limits: rps_global, rps_per_host, random_delay_min/max, sleep_on_429_sec
- proxy: enabled, http_proxy, https_proxy, rotate_on_codes
- scraper: max_pages, max_items_per_url, concurrency, feature_flags (fetch_phone), source=urls_file|db
- logging: level, logfile, json, rotate
- web: enable, host, port (по умолчанию 7474), base_path, auth
- telegram: enable, bot_token, chat_id, parse_mode, batching, batch_size

Чувствительные данные передавайте только через переменные окружения или безопасные секрет‑хранилища.

## SQLite схема (минимум)
- searches(id, query, filters_json, status, created_at, updated_at)
- items(id PRIMARY KEY, url, title, price_amount, price_currency, city, address, published_at, main_image_url, seller_name, phone_e164, raw_json)
- search_items(search_id, item_id, first_seen_at, last_seen_at, UNIQUE(search_id, item_id))
- stats_timeseries(ts, metric, value, search_id)
- scrape_status(search_id, last_page, last_run_at, last_error)

## Запуск в Docker

Сервисы docker‑compose:
- app: FastAPI + Uvicorn (порт 7474), веб‑интерфейс и скрапер.
- bot: Telegram‑бот — отдельный сервис.
- chrome: selenium/standalone-chrome — браузер для undetected‑chromedriver.

Volumes:
- ./data:/app/data — SQLite (app.db), экспорт, логи.

ENV (пример):
- WEB_ENABLE=true, WEB_HOST=0.0.0.0, WEB_PORT=7474
- SEARCHES_SQLITE_PATH=/app/data/app.db
- AVITO_LOGIN, AVITO_PASSWORD
- TELEGRAM_BOT_TOKEN, TELEGRAM_CHAT_ID

Команды:
- app: uvicorn main:app --host 0.0.0.0 --port 7474 (health: GET /healthz)
- bot: python -m app.bot

Запуск:
- docker compose up -d --build
- Логи: docker compose logs -f app | bot

## Запуск без Docker (локально)

1) Установите Python 3.11+, создайте venv, установите зависимости (requirements.txt).
2) Заполните конфиг и/или ENV (см. раздел Конфигурация). Убедитесь, что установлен Chrome + chromedriver совместимой версии (для undetected‑chromedriver не всегда требуется системный chromedriver).
3) Запуск веб‑приложения: uvicorn main:app --host 0.0.0.0 --port 7474
4) Запуск бота: python -m app.bot

## Типовые сценарии

- Добавление запроса через веб‑интерфейс:
  1) Откройте http://localhost:7474, перейдите в раздел «Запро��ы».
  2) Создайте новый запрос (текст + фильтры), запустите скрапинг.
  3) Наблюдайте KPI и динамику; экспортируйте выборку при необходимости.

- Подписка на рассылку в Telegram:
  1) Запустите сервис bot с валидным TELEGRAM_BOT_TOKEN и CHAT_ID.
  2) Используйте /queries, /summary <search_id>, /latest <search_id> для просмотра.

## Логирование и ошибки

- Структурированные логи в файл и консоль; уровни INFO/DEBUG.
- Ретраи с backoff + jitter; чтение Retry‑After для 429.
- При 401/403 — автоматическая попытка обновления cookies; при неудаче — останов и запись чекпоинтов.
- Ошибки по одному объявлению не останавливают процесс.

## Ограничения и ответственность

- Соблюдайте условия использования Avito и применяй��е инструмент только для законных целей.
- Учитывайте, что мобильное API может меняться; потребуется актуализация адаптеров.

## Структура репозитория (рекомендуется)

- src/
  - auth_manager.py
  - scraper.py
  - storage.py
  - web/ (FastAPI, роуты, шаблоны)
  - bot/ (Telegram‑бот)
  - models.py, rate_limiter.py, retry.py, parser_adapters/, utils.py
- configs/
  - config.example.ini
- data/
  - app.db (SQLite), экспорт, логи
- docs/
  - index.md (ТЗ)

## Лицензия

Проект предназначен для внутреннего/личного использования. Убедитесь, что ваш сценарий использования соответствует законодательству и правилам Avito.