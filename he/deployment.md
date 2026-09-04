[תיעוד](index.md) / פריסה

# פריסה

[תחילת העבודה](getting-started.md) מריץ את LookSee על מכונה אחת. עמוד זה מכסה שרת שאנשים אחרים משתמשים בו: כתובות, TLS דרך פרוקסי הפוך (reverse proxy), הסקה על GPU, גיבויים ושדרוגים.

## שרת ברשת שלכם

הגדירו את `WEBRTC_HOST_IP` ב-`.env` לכתובת שבה המשתמשים מגיעים לשרת, למשל `192.168.1.20`. דפדפנים מתחברים לכתובת זו לווידאו חי, וכתובות `RUNTIME_*` שבברירת המחדל מכוונות את Studio אליה. שנו את סיסמאות הדוגמה, ואז הפעילו את הסטאק:

```bash
docker compose up -d --build
```

Studio נמצא בכתובת `http://192.168.1.20:3000`. מצלמות רשת של הדפדפן אינן עובדות בהגדרה זו: דפדפנים מאפשרים גישה למצלמה רק ב-`localhost` או דרך HTTPS, ולכן תהליך עבודה עם מצלמת רשת זקוק להגדרת ה-TLS שלהלן.

## TLS עם פרוקסי הפוך

סיימו את ה-TLS בפרוקסי הפוך שלפני Studio, ה-API ופורט ה-WebRTC של MediaMTX. הדוגמה משתמשת ב-[Caddy](https://caddyserver.com/), שמשיג תעודות ומעביר WebSockets בלי תצורה נוספת; כל פרוקסי שמעביר שדרוגי WebSocket עובד.

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

כוונו את Studio ואת ה-API לשמות הציבוריים ב-`.env`:

```bash
WEBRTC_HOST_IP=203.0.113.10          # the server's public address, for WebRTC media
RUNTIME_API_URL=https://api.example.com
RUNTIME_WS_URL=wss://api.example.com
RUNTIME_MEDIAMTX_WEBRTC_URL=https://media.example.com
CORS_ORIGIN_REGEX=^https://studio\.example\.com$
AUTH_COOKIE_SECURE=true
```

שלושת השמות משתפים את הדומיין הרשום `example.com`, ולכן עוגיית הסשן שה-API מגדיר נשלחת עם הבקשות של Studio. איתות ה-WebRTC עובר דרך הפרוקסי; המדיה עצמה זורמת דרך פורט UDP `8189` ישירות אל `WEBRTC_HOST_IP`, ולכן פתחו פורט זה בחומת האש ואל תעבירו אותו דרך הפרוקסי.

אתחלו את השירותים המושפעים אחרי עריכת `.env`:

```bash
docker compose up -d
```

## הסקה על GPU

אימג' ההסקה המפורסם מריץ מודלים על ה-CPU. ONNX Runtime מעדיף CUDA כשהוא זמין, ולחבילה `looksee-inference` יש תוסף `gpu` שמתקין `onnxruntime-gpu` במקום `onnxruntime`. ה-Dockerfile חושף לכך שני ארגומנטי בנייה:

| ארגומנט | ברירת מחדל | מטרה |
| --- | --- | --- |
| `TARGET` | `cpu` | איזה תוסף להתקין: `cpu` או `gpu` |
| `BASE_IMAGE` | `python:3.12-slim-trixie` | אימג' הבסיס; בניית GPU זקוקה לאימג' עם CUDA runtime ו-cuDNN שמתאימים לגרסת ONNX Runtime, ועם Python 3.12 |

בנו והריצו עם קובץ override של compose שמעביר את ה-GPU:

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

המארח זקוק לדרייבר של NVIDIA ול-NVIDIA Container Toolkit. בהפעלה, לוג ההסקה מפרט את ספקי הביצוע ש-ONNX Runtime מצא; `CUDAExecutionProvider` מאשר שה-GPU בשימוש.

## תכנון קיבולת

עלות הזיהוי היא מספר הרצות המודל בשנייה על פני כל המצלמות: סכום **Checks per second** של כל צמתי Detect. עלות הפענוח תלויה ברזולוציית הזרם ובקצב הפריימים, ללא קשר לזיהוי. שני מנופים שומרים על פריסת CPU נוחה:

- השתמשו בזרמי המשנה של המצלמות ב-720p או פחות.
- השאירו את **Checks per second** על 1 עד 2, אלא אם התרחיש דורש יותר.

העלו את `INFERENCE_CPUS` ואת `INFERENCE_MEMORY` ככל שמתווספות מצלמות. ה-API, PostgreSQL ו-Valkey קלים בהשוואה. הריצו עותק API אחד: מצב המסננים וזמני הצינון חיים בזיכרון של ה-API.

## גיבויים

המצב נשמר בשלושה volumes שכדאי לגבות: `postgres_data`, `api_keys` ו-`storage_data` עם הסרטונים שהועלו, ובנוסף `api_snapshots` אם אתם שומרים תמונות ראיה. Compose מוסיף לשמות ה-volumes קידומת של שם הפרויקט, `looksee` כברירת מחדל.

```bash
# Database
docker compose exec -T postgres pg_dump -U looksee looksee > looksee.sql

# Signing and encryption secret, and snapshots
docker run --rm -v looksee_api_keys:/data -v "$PWD":/backup alpine \
  tar czf /backup/api_keys.tgz -C /data .
docker run --rm -v looksee_api_snapshots:/data -v "$PWD":/backup alpine \
  tar czf /backup/api_snapshots.tgz -C /data .

# Uploaded videos
docker run --rm -v looksee_storage_data:/data -v "$PWD":/backup alpine \
  tar czf /backup/storage_data.tgz -C /data .
```

שחזרו לסטאק נקי בהפעלת `postgres` לבדו, טעינת ה-dump באמצעות `psql`, חילוץ הארכיונים אל ה-volumes באותה דרך, ואז הפעלת השאר. הגדרת `SECRET_KEY` ב-`.env` במקום להסתמך על הקובץ שנוצר הופכת את המפתח לחלק מגיבוי התצורה שלכם.

## שדרוגים

```bash
git pull
docker compose up -d --build
```

השירות `api-migrate` מחיל מיגרציות של מסד הנתונים לפני שה-API מופעל, וה-API, שירות ההסקה, Studio ו-MediaMTX חייבים לרוץ באותה גרסה מכיוון שהחוזים שלהם משתנים יחד. קראו קודם את [יומן השינויים](changelog.md): גרסה עשויה להוסיף משתנה נדרש ל-`.env`, שבלעדיו compose מסרב לעלות. בדקו את `docker compose ps` לאחר מכן.

## שירותים מקומיים

לפיתוח, התשתית יכולה לרוץ בקונטיינרים בזמן שהשירותים רצים מעץ המקור. [CONTRIBUTING.md](https://github.com/okoflow/looksee/blob/main/CONTRIBUTING.md#development-setup) במאגר מתאר הגדרה זו.
