[Документация](index.md) / API

# API

API — это сервис FastAPI на порту `8000`. Studio — его единственный
встроенный клиент, и всё, что делает Studio, доступно скриптам и
интеграциям. Интерактивная документация со схемами запросов и ответов
доступна по `/docs`, документ OpenAPI — по `/openapi.json`.

## Аутентификация

Все конечные точки, кроме `/health`, `/auth/*`, `/docs` и `/openapi.json`,
требуют cookie `looksee_session`. Получите его, войдя в систему, и
используйте семь дней:

```bash
curl -c cookies.txt -H 'content-type: application/json' \
  -d '{"email":"owner@example.com","password":"..."}' \
  http://127.0.0.1:8000/auth/login

curl -b cookies.txt http://127.0.0.1:8000/workflows
```

## Ошибки

Ошибки возвращают JSON с сообщением `detail`. Ошибки графа рабочего процесса
добавляют стабильный `code` и `node_id` проблемного узла, чтобы клиент мог
его выделить:

```json
{ "detail": "camera 'camera' reaches no detect node", "code": "detect_node_missing", "node_id": "camera" }
```

| Статус | Значение |
| --- | --- |
| `401` | Сессия отсутствует или недействительна; неверные адрес электронной почты или пароль |
| `402` | Граф использует узел Enterprise без лицензии (`feature_not_licensed`) |
| `404` | Неизвестный рабочий процесс, оповещение, учётные данные или файл |
| `409` | Учётная запись владельца уже существует |
| `422` | Проверка не пройдена: поле вне допустимого диапазона, или граф не может работать (задан `code`) |
| `503` | Библиотека файлов не настроена |

[Узлы](nodes.md#валидация) перечисляют каждый код ошибки графа.

## Конечные точки

### Состояние и первоначальная настройка

| Метод и путь | Описание |
| --- | --- |
| `GET /health` | Возвращает `{"status":"ok"}`; без аутентификации |
| `GET /auth/status` | `{"requires_setup": true}`, пока владелец не создан |
| `POST /auth/setup` | Создать владельца: `email`, `name`, `password`. Выставляет cookie сессии. |
| `POST /auth/login` | `email`, `password`. Выставляет cookie сессии. |
| `POST /auth/logout` | Удаляет cookie |
| `GET /auth/me` | Вошедший пользователь |
| `GET /entitlements` | `{"edition": "community", "features": []}` или возможности Enterprise |

### Модели

| Метод и путь | Описание |
| --- | --- |
| `GET /models` | Обнаруженные пакеты моделей: `id`, `name`, `classes` (`class_id`, `label`, `event_kind`), `recommended_confidence_threshold` |

### Рабочие процессы

| Метод и путь | Описание |
| --- | --- |
| `GET /workflows` | Все рабочие процессы с их камерами и статусами, новые первыми |
| `POST /workflows` | Создать: `name`, необязательные `description` и `graph`. Возвращает `201`. |
| `GET /workflows/{id}` | Один рабочий процесс |
| `PATCH /workflows/{id}` | Частичное обновление `name`, `description`, `enabled`, `graph`. Установка `enabled` в `true` проверяет граф и запускает камеры. |
| `DELETE /workflows/{id}` | Остановить и удалить; возвращает `204` |

Граф — это `{"nodes": [...], "edges": [...], "comments": [...]}`. У каждого
узла есть `id`, `position` с `x` и `y` и `data`, где `kind` выбирает тип
узла, а остальные поля перечислены в разделе [Узлы](nodes.md). У каждой
связи есть `id`, `source`, `target` и необязательный `branch` со значением
`if` или `else` для связей, выходящих из фильтра.

```json
{
  "name": "Fire watch",
  "graph": {
    "nodes": [
      {"id": "camera", "position": {"x": 0, "y": 80}, "data": {"kind": "camera_source", "name": "Warehouse", "source_type": "rtsp", "url": "rtsp://192.168.1.30/stream1"}},
      {"id": "detect", "position": {"x": 240, "y": 80}, "data": {"kind": "detect", "model_id": "fire-smoke", "event_kinds": ["FIRE_DETECTED", "SMOKE_DETECTED"], "confidence_threshold": 0.3, "inference_fps": 1}},
      {"id": "alert", "position": {"x": 480, "y": 80}, "data": {"kind": "log_alert_action", "severity": "critical", "cooldown_seconds": 0}}
    ],
    "edges": [
      {"id": "e1", "source": "camera", "target": "detect"},
      {"id": "e2", "source": "detect", "target": "alert"}
    ]
  }
}
```

### Оповещения

| Метод и путь | Описание |
| --- | --- |
| `GET /alerts` | Новые первыми. Фильтры: `camera_id`, `workflow_id`, `severity` (`info`, `warning`, `critical`), `limit` (от 1 до 500, по умолчанию 100) |
| `DELETE /alerts/{id}` | Удалить одно оповещение |
| `DELETE /alerts` | Удалить оповещения, при необходимости только по `camera_id` или `workflow_id` |

Оповещение содержит `id`, `workflow_id`, `camera_id`, `kind`, `severity`,
`message`, `payload` (метка времени, детекции, метаданные, `snapshot_url`,
если перед Alert сработал Snapshot) и `created_at`.

### Учётные данные

| Метод и путь | Описание |
| --- | --- |
| `GET /credentials` | `id`, `name`, `type`, `summary`, метки времени. Содержимое `payload` никогда не возвращается. |
| `POST /credentials` | `name`, `type`, `payload`. Поля `payload` для каждого типа перечислены в разделе [Действия и интеграции](actions-and-integrations.md#учётные-данные). |
| `PATCH /credentials/{id}` | Частичное обновление `name` и `payload`; без `payload` сохранённый секрет остаётся прежним |
| `DELETE /credentials/{id}` | Удалить |

### Файлы

Доступны, когда настроена библиотека файлов; иначе каждый вызов возвращает
`503`.

| Метод и путь | Описание |
| --- | --- |
| `GET /assets` | Объекты в бакете |
| `POST /assets` | Загрузка multipart с полем `file`; возвращает `201` |
| `GET /assets/{key}/content` | Кэшированный файл, с поддержкой range-запросов для воспроизведения |
| `DELETE /assets/{key}` | Удалить объект |

### Снимки

`GET /snapshots/<file>.jpg` отдаёт изображения, записанные действием
Snapshot. Запросу нужен cookie сессии.

## WebSocket

`ws://<host>:8000/ws/cameras/{camera_id}` передаёт живые сообщения одной
камеры клиенту, предъявившему cookie сессии. Сокет однонаправленный;
входящие кадры игнорируются. Каждое сообщение — JSON с полем `type`:

| Тип | Поля | Когда |
| --- | --- | --- |
| `detections` | `ts`, `frame_width`, `frame_height`, `detections[]` | Каждый обработанный кадр |
| `event` | `kind`, `ts`, `frame_width`, `frame_height`, `detections[]` | Каждое событие, прошедшее паузу между событиями |
| `worker` | `status`, `ts`, `reason` | Камера меняет статус |
| `alert` | `id`, `kind`, `severity`, `message`, `ts`, `snapshot_url` | Срабатывает действие Alert |

Детекция содержит `label`, `bounding_box` в виде `[x_min, y_min, x_max, y_max]`
в пикселях кадра, `confidence`, `class_id` и `tracker_id` (или `null`).

## Тело запроса Webhook

Действия Webhook и MQTT доставляют такой JSON для каждого события:

```json
{
  "camera_id": "01a061c1-ff68-7611-af72-436d9d5ba907",
  "kind": "HELMET_DETECTED",
  "ts": "2026-09-02T10:58:20.983914+00:00",
  "metadata": { "count": 2, "model_id": "ppe-helmets" },
  "snapshot_url": "/snapshots/20260902-105820-01a061c1-3f9a2c1e.jpg"
}
```

`snapshot_url` присутствует, только если ранее в графе сработало действие
Snapshot, и указан относительно origin API.
