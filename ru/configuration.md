[Документация](index.md) / Конфигурация

# Конфигурация

LookSee настраивается через переменные окружения. В Docker Compose они
берутся из `.env` рядом с `compose.yaml`; `.env.example` перечисляет каждую
переменную с её значением по умолчанию. Сервисы читают переменные при
запуске, поэтому изменение вступает в силу после `docker compose up -d`.

## Стек Compose

Эти переменные задают сам стек: образы, порты, пароли и ограничения ресурсов.

| Переменная | По умолчанию | Значение |
| --- | --- | --- |
| `WEBRTC_HOST_IP` | обязательна | Адрес, по которому браузеры обращаются к MediaMTX за живым видео: `127.0.0.1` на одной машине, адрес в локальной сети на сервере. Также хост по умолчанию в URL `RUNTIME_*`. |
| `POSTGRES_PASSWORD` | обязательна | Пароль базы данных. |
| `MTX_MEDIA_PASSWORD` | обязательна | Пароль медиапользователя MediaMTX. Виден браузерам, которые загружают Studio. |
| `MTX_MEDIA_USER` | `media` | Пользователь MediaMTX с правами чтения и публикации. |
| `POSTGRES_USER`, `POSTGRES_DB` | `looksee` | Пользователь и имя базы данных. |
| `POSTGRES_PORT` | `5432` | Порт хоста для PostgreSQL, привязан к `127.0.0.1`. |
| `REDIS_PORT` | `6379` | Порт хоста для Valkey, привязан к `127.0.0.1`. |
| `REDIS_MAXMEMORY` | `512mb` | Лимит памяти Valkey; политика вытеснения — `noeviction`. |
| `MTX_RTSP_PORT` | `8554` | Порт RTSP. |
| `MTX_WEBRTC_PORT` | `8889` | Порт сигнализации и воспроизведения WebRTC. |
| `MTX_WEBRTC_ICE_PORT` | `8189` | Порт медиапотока WebRTC (UDP). |
| `MTX_LOGLEVEL` | `info` | Уровень журналирования MediaMTX. |
| `API_PORT` | `8000` | Порт хоста для API. |
| `WEB_PORT` | `3000` | Порт хоста для Studio. |
| `INFERENCE_CPUS` | `4.0` | Лимит CPU контейнера инференса; также ограничивает число потоков ONNX Runtime. |
| `INFERENCE_MEMORY` | `4g` | Лимит памяти контейнера инференса. |
| `REGISTRY`, `TAG` | `looksee`, `latest` | Префикс имени и тег трёх образов приложения. |

Остальные сервисы тоже ограничены: `api` — 2 CPU и 1 ГБ, `postgres` — 2 CPU
и 2 ГБ, `mediamtx` — 2 CPU и 1 ГБ, `redis` — 1 CPU и 768 МБ, `studio` —
1 CPU и 512 МБ. Чтобы изменить их, отредактируйте `compose.yaml` или добавьте
файл переопределения.

## API

| Переменная | По умолчанию | Значение |
| --- | --- | --- |
| `LICENSE_KEY` | не задана | Лицензионный ключ Enterprise. Не задана или пуста — работает редакция Community. |
| `DATABASE_URL` | задаётся compose | URL SQLAlchemy, `postgresql+asyncpg://user:password@host:5432/looksee`. |
| `REDIS_URL` | задаётся compose | URL Valkey, `redis://host:6379/0`. |
| `MEDIAMTX_API_URL` | задаётся compose | Управляющий API MediaMTX, `http://mediamtx:9997`. |
| `MODELS_DIR` | `/app/models` | Каталог пакетов моделей. |
| `SNAPSHOTS_DIR` | `/data/snapshots` | Куда действие Snapshot пишет файлы JPEG; отдаются по `/snapshots`. |
| `SECRET_KEY` | не задана | Корневой секрет для подписи сессий и шифрования учётных данных. Если не задана, секрет генерируется при первом запуске и сохраняется в `SECRET_KEY_FILE`. |
| `SECRET_KEY_FILE` | `/data/keys/secret.key` | Расположение сгенерированного секрета на томе `api_keys`. |
| `AUTH_COOKIE_SECURE` | `false` | Помечать cookie сессии как `Secure`. Установите `true` за HTTPS. |
| `CORS_ORIGIN_REGEX` | `^https?://(localhost\|127\.0\.0\.1)(:\d+)?$` | Источники (origin), которым разрешено обращаться к API из браузера. Задайте origin Studio, когда это не localhost. |
| `EVENT_COOLDOWN_SECONDS` | `2` | Минимальный промежуток между двумя событиями одного типа на одной камере. `0` отключает паузу. |
| `EVENT_TIMEZONE` | `UTC` | Часовой пояс для фильтра Schedule, имя IANA, например `Europe/Berlin`. |
| `RECONCILE_INTERVAL_SECONDS` | `30` | Как часто API повторно публикует желаемое состояние камер и повторяет попытки для отказавших камер. |
| `CONSUMER_GROUP` | `api-workers` | Группа потребителей потока Valkey для кадров детекции. |
| `S3_ENDPOINT_URL`, `S3_BUCKET` | пусто | Библиотека файлов для камер File. Для включения нужны обе. |
| `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY` | пусто | Учётные данные библиотеки файлов. |
| `S3_REGION` | `auto` | Регион; `auto` для Cloudflare R2. |
| `S3_PREFIX` | пусто | Префикс ключей внутри бакета. |
| `MEDIA_CACHE_DIR` | задаётся compose | Где кэшируются скачанные файлы для воспроизведения. |
| `MEDIA_MOUNT_DIR` | `/media` | Где тот же кэш видит MediaMTX. |

## Инференс

| Переменная | По умолчанию | Значение |
| --- | --- | --- |
| `REDIS_URL` | задаётся compose | URL Valkey. |
| `MEDIAMTX_RTSP_URL` | `rtsp://mediamtx:8554` | Откуда читаются пути камер. |
| `MTX_MEDIA_USER`, `MTX_MEDIA_PASSWORD` | из `.env` | Учётные данные для чтения путей камер. |
| `RTSP_TRANSPORT` | `tcp` | `tcp` или `udp` для чтения из MediaMTX. |
| `MODELS_DIR` | `/app/models` | Каталог пакетов моделей. |
| `FIRST_FRAME_TIMEOUT_SECONDS` | `30` | Сколько ждать первый кадр, а затем каждый следующий во время работы, прежде чем сообщить об ошибке. |
| `LAST_FRAME_TTL_SECONDS` | `10` | Сколько последний JPEG каждой камеры остаётся доступным для действия Snapshot. |

## Studio

Studio читает свою конфигурацию на сервере при каждом запросе и передаёт
публичную часть браузеру, поэтому образ можно перенацелить без пересборки.

| Переменная | По умолчанию | Значение |
| --- | --- | --- |
| `RUNTIME_API_URL` | `http://<WEBRTC_HOST_IP>:<API_PORT>` | Базовый URL API, как его видит браузер. |
| `RUNTIME_WS_URL` | `ws://<WEBRTC_HOST_IP>:<API_PORT>` | Базовый URL WebSocket, как его видит браузер. |
| `RUNTIME_MEDIAMTX_WEBRTC_URL` | `http://<WEBRTC_HOST_IP>:<MTX_WEBRTC_PORT>` | URL WebRTC MediaMTX, как его видит браузер. |
| `RUNTIME_MEDIAMTX_MEDIA_USER`, `RUNTIME_MEDIAMTX_MEDIA_PASSWORD` | медиапользователь MediaMTX | Передаются браузеру для воспроизведения и публикации веб-камеры. |
| `RUNTIME_DOCS_URL` | `http://<WEBRTC_HOST_IP>:3002/docs` | Адрес ссылки **Documentation** в боковой панели. |
| `RUNTIME_GITHUB_URL` | `https://github.com/okoflow/looksee` | Адрес ссылки **GitHub**. |
| `SERVER_API_URL` | `http://api:8000` | Адрес API, который использует сам сервер Studio для проверки входа. Браузеру не передаётся. |

За обратным прокси задайте трём URL `RUNTIME_*` публичные адреса; пример
есть в разделе [Развёртывание](deployment.md).

## Порты

| Порт | Сервис | Привязка | Назначение |
| --- | --- | --- | --- |
| `3000` | studio | все интерфейсы | Веб-интерфейс |
| `8000` | api | все интерфейсы | HTTP API и WebSocket |
| `8554` | mediamtx | все интерфейсы | RTSP |
| `8889` | mediamtx | все интерфейсы | Сигнализация и воспроизведение WebRTC |
| `8189/udp` | mediamtx | все интерфейсы | Медиапоток WebRTC |
| `9997` | mediamtx | `127.0.0.1` | Управляющий API MediaMTX |
| `5432` | postgres | `127.0.0.1` | PostgreSQL |
| `6379` | redis | `127.0.0.1` | Valkey |

## Тома

| Том | Сервис | Содержимое |
| --- | --- | --- |
| `postgres_data` | postgres | Рабочие процессы, камеры, учётные данные, оповещения, пользователи |
| `redis_data` | redis | Поток детекций и каналы команд; потеря безопасна |
| `api_snapshots` | api | Файлы JPEG снимков |
| `api_keys` | api | Сгенерированный `secret.key` |
| `media-cache` | api, mediamtx | Кэшированные видеофайлы для камер File |
| `./models` (bind) | api, inference | Пакеты моделей, только чтение |

`postgres_data` и `api_keys` хранят состояние, которое стоит резервировать;
без секрета сохранённые учётные данные невозможно расшифровать, а все сессии
завершаются. Раздел [Развёртывание](deployment.md#резервные-копии) описывает
порядок резервного копирования.
