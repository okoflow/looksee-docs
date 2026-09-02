[Документация](index.md) / Развёртывание

# Развёртывание

[Начало работы](getting-started.md) запускает LookSee на одной машине. Эта
страница описывает сервер, которым пользуются другие люди: адресация, TLS
через обратный прокси, инференс на GPU, резервные копии и обновления.

## Сервер в вашей сети

Задайте `WEBRTC_HOST_IP` в `.env` равным адресу, по которому пользователи
обращаются к серверу, например `192.168.1.20`. Браузеры подключаются к этому
адресу за живым видео, а URL `RUNTIME_*` по умолчанию направляют Studio на
него. Смените пароли из примера, затем запустите стек:

```bash
docker compose up -d --build
```

Studio доступна по адресу `http://192.168.1.20:3000`. Веб-камеры браузера в
такой схеме не работают: браузеры разрешают доступ к камере только на
`localhost` или по HTTPS, поэтому рабочему процессу с веб-камерой нужна
настройка TLS, описанная ниже.

## TLS через обратный прокси

Терминируйте TLS в обратном прокси перед Studio, API и портом WebRTC
MediaMTX. В примере используется [Caddy](https://caddyserver.com/), который
получает сертификаты и проксирует WebSocket без дополнительной настройки;
подойдёт любой прокси, пробрасывающий апгрейд WebSocket.

```caddyfile
studio.example.com {
    reverse_proxy 127.0.0.1:3000
}

api.example.com {
    reverse_proxy 127.0.0.1:8000
}

media.example.com {
    reverse_proxy 127.0.0.1:8889
}
```

Укажите Studio и API публичные имена в `.env`:

```bash
WEBRTC_HOST_IP=203.0.113.10          # the server's public address, for WebRTC media
RUNTIME_API_URL=https://api.example.com
RUNTIME_WS_URL=wss://api.example.com
RUNTIME_MEDIAMTX_WEBRTC_URL=https://media.example.com
CORS_ORIGIN_REGEX=^https://studio\.example\.com$
AUTH_COOKIE_SECURE=true
```

Три имени делят регистрируемый домен `example.com`, поэтому cookie сессии,
выставленный API, отправляется с запросами Studio. Сигнализация WebRTC идёт
через прокси; сам медиапоток идёт по UDP-порту `8189` напрямую на
`WEBRTC_HOST_IP`, поэтому откройте этот порт на брандмауэре и не проксируйте
его.

После правки `.env` перезапустите затронутые сервисы:

```bash
docker compose up -d
```

## Инференс на GPU

Опубликованный образ инференса запускает модели на CPU. ONNX Runtime
предпочитает CUDA, когда она доступна, а у пакета `looksee-inference` есть
дополнение `gpu`, которое устанавливает `onnxruntime-gpu` вместо
`onnxruntime`. Для этого Dockerfile предоставляет два аргумента сборки:

| Аргумент | По умолчанию | Назначение |
| --- | --- | --- |
| `TARGET` | `cpu` | Какое дополнение устанавливать: `cpu` или `gpu` |
| `BASE_IMAGE` | `python:3.12-slim-trixie` | Базовый образ; сборке для GPU нужен образ со средой выполнения CUDA и cuDNN, соответствующими выпуску ONNX Runtime, плюс Python 3.12 |

Соберите и запустите с файлом переопределения compose, который пробрасывает
GPU:

```yaml
# compose.gpu.yaml
services:
  inference:
    build:
      args:
        TARGET: gpu
        BASE_IMAGE: <cuda base image with python 3.12>
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

```bash
docker compose -f compose.yaml -f compose.gpu.yaml up -d --build inference
```

Хосту нужны драйвер NVIDIA и NVIDIA Container Toolkit. При запуске журнал
инференса перечисляет провайдеры выполнения, которые нашёл ONNX Runtime;
`CUDAExecutionProvider` подтверждает, что GPU используется.

## Оценка ресурсов

Стоимость детекции — это число прогонов модели в секунду по всем камерам:
сумма **Checks per second** каждого узла Detect. Стоимость декодирования
зависит от разрешения и частоты кадров потока и не зависит от детекции. Два
рычага делают развёртывание на CPU комфортным:

- Используйте дополнительные потоки камер с разрешением 720p или ниже.
- Держите **Checks per second** на уровне от 1 до 2, если сценарий не требует
  большего.

Увеличивайте `INFERENCE_CPUS` и `INFERENCE_MEMORY` по мере добавления камер.
API, PostgreSQL и Valkey по сравнению с этим легки. Запускайте одну реплику
API: паузы между событиями и состояние трекинга живут в памяти API.

## Резервные копии

Состояние живёт в двух томах, которые стоит резервировать, `postgres_data` и
`api_keys`, плюс `api_snapshots`, если вы храните изображения-доказательства.
Compose добавляет к именам томов префикс с именем проекта, по умолчанию
`looksee`.

```bash
# Database
docker compose exec -T postgres pg_dump -U looksee looksee > looksee.sql

# Signing and encryption secret, and snapshots
docker run --rm -v looksee_api_keys:/data -v "$PWD":/backup alpine \
  tar czf /backup/api_keys.tgz -C /data .
docker run --rm -v looksee_api_snapshots:/data -v "$PWD":/backup alpine \
  tar czf /backup/api_snapshots.tgz -C /data .
```

Восстановление в свежий стек: запустите только `postgres`, загрузите дамп
через `psql`, распакуйте архивы в тома тем же способом, затем запустите
остальное. Если задать `SECRET_KEY` в `.env` вместо того, чтобы полагаться на
сгенерированный файл, ключ становится частью резервной копии вашей
конфигурации.

## Обновления

```bash
git pull
docker compose up -d --build
```

Сервис `api-migrate` применяет миграции базы данных до запуска API, а API и
сервис инференса должны работать одной версии, потому что их контракты
сообщений меняются вместе. После обновления проверьте `docker compose ps`;
[история изменений](changelog.md) перечисляет изменения, требующие действий
оператора.

## Нативные сервисы

Для разработки инфраструктура может работать в контейнерах, а сервисы — из
дерева исходников. Эту схему описывает
[CONTRIBUTING.md](https://github.com/okoflow/looksee/blob/main/CONTRIBUTING.md#development-setup)
в репозитории.
