[תיעוד](index.md) / תצורה

# תצורה

LookSee מוגדרת באמצעות משתני סביבה. עם Docker Compose הם מגיעים מ-`.env` שליד `compose.yaml`; `.env.example` מפרט את המשתנים שהתקנה מגדירה, מקובצים לפי שירות, והטבלאות שלהלן מכסות את השאר עם ברירות המחדל שלהם. השירותים קוראים את המשתנים שלהם בהפעלה, ולכן שינוי נכנס לתוקף אחרי `docker compose up -d`.

## סטאק ה-compose

משתנים אלה מעצבים את הסטאק עצמו: אימג'ים, פורטים, סיסמאות ומגבלות משאבים.

| משתנה | ברירת מחדל | משמעות |
| --- | --- | --- |
| `WEBRTC_HOST_IP` | נדרש | הכתובת שדפדפנים משתמשים בה כדי להגיע ל-MediaMTX לווידאו חי: `127.0.0.1` על מכונה אחת, כתובת ה-LAN על שרת. גם המארח שבברירת המחדל בכתובות `RUNTIME_*`. |
| `POSTGRES_PASSWORD` | נדרש | סיסמת מסד הנתונים. |
| `MTX_MEDIA_PASSWORD` | נדרש | סיסמת משתמש השירות של MediaMTX, משותפת ל-API, לשירות ההסקה ול-MediaMTX. לעולם אינה נשלחת לדפדפנים. |
| `MTX_MEDIA_USER` | `media` | משתמש השירות של MediaMTX עם הרשאות קריאה ופרסום. |
| `STORAGE_PASSWORD` | נדרש | המפתח הסודי של אחסון הווידאו המובנה; מפתח הגישה הוא `looksee`. |
| `STORAGE_PORT` | `9000` | פורט המארח של ה-S3 API של האחסון, מאוגד ל-`127.0.0.1`. |
| `POSTGRES_USER`, `POSTGRES_DB` | `looksee` | משתמש ושם מסד הנתונים. |
| `POSTGRES_PORT` | `5432` | פורט המארח של PostgreSQL, מאוגד ל-`127.0.0.1`. |
| `REDIS_PORT` | `6379` | פורט המארח של Valkey, מאוגד ל-`127.0.0.1`. |
| `REDIS_MAXMEMORY` | `512mb` | מגבלת הזיכרון של Valkey; מדיניות הפינוי היא `noeviction`. |
| `MTX_RTSP_PORT` | `8554` | פורט RTSP. |
| `MTX_WEBRTC_PORT` | `8889` | פורט האיתות והניגון של WebRTC. |
| `MTX_WEBRTC_ICE_PORT` | `8189` | פורט המדיה של WebRTC (UDP). |
| `MTX_LOGLEVEL` | `info` | רמת הלוג של MediaMTX. |
| `MTX_AUTHHTTPADDRESS` | `http://api:8000/internal/media/auth` | היכן MediaMTX מבקש אישור. שנו רק כשה-API רץ מחוץ ל-compose. |
| `API_PORT` | `8000` | פורט המארח של ה-API. |
| `WEB_PORT` | `3000` | פורט המארח של Studio. |
| `INFERENCE_CPUS` | `4.0` | מגבלת ה-CPU של קונטיינר ההסקה; מגבילה גם את מספר החוטים של ONNX Runtime. |
| `INFERENCE_MEMORY` | `4g` | מגבלת הזיכרון של קונטיינר ההסקה. |
| `REGISTRY`, `TAG` | `looksee`, `latest` | קידומת שם האימג' והתג של שלושת אימג'י היישום. |

גם שאר השירותים מוגבלים: `api` 2 מעבדים ו-1 GB, `postgres` 2 מעבדים ו-2 GB, `mediamtx` 2 מעבדים ו-1 GB, `storage` 2 מעבדים ו-1 GB, `redis` מעבד אחד ו-768 MB, `studio` מעבד אחד ו-512 MB. ערכו את `compose.yaml` או הוסיפו קובץ override כדי לשנותם.

## API

| משתנה | ברירת מחדל | משמעות |
| --- | --- | --- |
| `LICENSE_KEY` | לא מוגדר | מפתח רישיון Enterprise. לא מוגדר או ריק מריץ את מהדורת Community. |
| `DATABASE_URL` | נקבע על ידי compose | כתובת SQLAlchemy, `postgresql+asyncpg://user:password@host:5432/looksee`. |
| `REDIS_URL` | נקבע על ידי compose | כתובת Valkey, `redis://host:6379/0`. |
| `MEDIAMTX_API_URL` | נקבע על ידי compose | API הבקרה של MediaMTX, `http://mediamtx:9997`. |
| `MODELS_DIR` | `/app/models` | ספריית חבילות המודלים. |
| `SNAPSHOTS_DIR` | `/data/snapshots` | היכן פעולת Snapshot כותבת קובצי JPEG; מוגש ב-`/snapshots`. |
| `SECRET_KEY` | לא מוגדר | סוד השורש לחתימת סשנים ולהצפנת אישורי גישה. כשאינו מוגדר, סוד נוצר בהפעלה הראשונה ונשמר ב-`SECRET_KEY_FILE`. |
| `SECRET_KEY_FILE` | `/data/keys/secret.key` | מיקום הסוד שנוצר, על ה-volume בשם `api_keys`. |
| `AUTH_COOKIE_SECURE` | `false` | מסמן את עוגיית הסשן כ-`Secure`. הגדירו ל-`true` מאחורי HTTPS. |
| `CORS_ORIGIN_REGEX` | `^https?://(localhost\|127\.0\.0\.1)(:\d+)?$` | מקורות (origins) שמורשים לקרוא ל-API מדפדפן. בקשות שמשנות מצב וחיבורי WebSocket ממקורות אחרים נדחים. הגדירו למקור של Studio כשאינו localhost. |
| `EVENT_COOLDOWN_SECONDS` | `2` | המרווח המזערי בין שתי הודעות `event` מאותו סוג באותה מצלמה בפיד החי. `0` מבטל את זמן הצינון. הגרף מקבל כל אירוע. |
| `EVENT_TIMEZONE` | `UTC` | אזור הזמן למסנן Schedule, שם IANA כמו `Europe/Berlin`. |
| `RECONCILE_INTERVAL_SECONDS` | `30` | באיזו תכיפות ה-API מפרסם מחדש את מצב המצלמות הרצוי ומנסה שוב מצלמות שנכשלו. |
| `CONSUMER_GROUP` | `api-workers` | קבוצת הצרכנים של זרם Valkey לפריימי זיהוי. |
| `S3_ENDPOINT_URL`, `S3_BUCKET` | `http://storage:9000`, `looksee` | ספריית הנכסים למצלמות File. compose מכוון אותם לאחסון המובנה; הגדירו את שניהם כדי להשתמש ב-bucket חיצוני תואם S3. בהרצה מקומית שניהם חייבים להיות מוגדרים כדי להפעיל את הספרייה. |
| `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY` | `looksee`, `STORAGE_PASSWORD` | פרטי הגישה לספריית הנכסים. |
| `S3_REGION` | `auto` | האזור (region) של bucket חיצוני; `auto` עבור Cloudflare R2. |
| `S3_PREFIX` | ריק | קידומת מפתח בתוך ה-bucket. |
| `MEDIA_CACHE_DIR` | נקבע על ידי compose | היכן נכסים שהורדו נשמרים במטמון לניגון. |
| `MEDIA_MOUNT_DIR` | `/media` | היכן MediaMTX רואה את אותו מטמון. |

## הסקה

| משתנה | ברירת מחדל | משמעות |
| --- | --- | --- |
| `REDIS_URL` | נקבע על ידי compose | כתובת Valkey. |
| `MEDIAMTX_RTSP_URL` | `rtsp://mediamtx:8554` | מהיכן נקראים נתיבי המצלמות. |
| `MTX_MEDIA_USER`, `MTX_MEDIA_PASSWORD` | מ-`.env` | פרטי הגישה לקריאת נתיבי המצלמות. |
| `RTSP_TRANSPORT` | `tcp` | `tcp` או `udp` לקריאה מ-MediaMTX. |
| `MODELS_DIR` | `/app/models` | ספריית חבילות המודלים. |
| `FIRST_FRAME_TIMEOUT_SECONDS` | `30` | כמה זמן לחכות לפריים הראשון, ולפריים הבא בזמן ריצה, לפני דיווח על שגיאה. |
| `LAST_FRAME_TTL_SECONDS` | `10` | כמה זמן ה-JPEG האחרון של כל מצלמה נשאר זמין לפעולת Snapshot. |

## Studio

Studio קורא את התצורה שלו בשרת בכל בקשה ומעביר את החלק הציבורי לדפדפן, ולכן אפשר להפנות אימג' מחדש בלי בנייה מחדש.

| משתנה | ברירת מחדל | משמעות |
| --- | --- | --- |
| `RUNTIME_API_URL` | `http://<WEBRTC_HOST_IP>:<API_PORT>` | כתובת הבסיס של ה-API כפי שנראית מהדפדפן. |
| `RUNTIME_WS_URL` | `ws://<WEBRTC_HOST_IP>:<API_PORT>` | כתובת הבסיס של WebSocket כפי שנראית מהדפדפן. |
| `RUNTIME_MEDIAMTX_WEBRTC_URL` | `http://<WEBRTC_HOST_IP>:<MTX_WEBRTC_PORT>` | כתובת WebRTC של MediaMTX כפי שנראית מהדפדפן. |
| `RUNTIME_DOCS_URL` | `https://github.com/okoflow/looksee-docs` | היעד של הקישור **Documentation** בסרגל הצד; compose מקבע אותו למאגר התיעוד. |
| `RUNTIME_GITHUB_URL` | `https://github.com/okoflow/looksee` | היעד של הקישור **GitHub**. |
| `SERVER_API_URL` | `http://api:8000` | כתובת ה-API שבה משתמש שרת Studio עצמו לבדיקת ההתחברות. לעולם אינה נשלחת לדפדפן. |

מאחורי פרוקסי הפוך (reverse proxy), הגדירו את שלוש כתובות `RUNTIME_*` לכתובות הציבוריות; ב[פריסה](deployment.md) יש דוגמה.

## פורטים

| פורט | שירות | מאוגד ל- | מטרה |
| --- | --- | --- | --- |
| `3000` | studio | כל הממשקים | ממשק web |
| `8000` | api | כל הממשקים | HTTP API ו-WebSocket |
| `8554` | mediamtx | כל הממשקים | RTSP |
| `8889` | mediamtx | כל הממשקים | איתות וניגון WebRTC |
| `8189/udp` | mediamtx | כל הממשקים | מדיה של WebRTC |
| `9997` | mediamtx | `127.0.0.1` | API הבקרה של MediaMTX |
| `9000` | storage | `127.0.0.1` | ה-S3 API של אחסון הווידאו |
| `5432` | postgres | `127.0.0.1` | PostgreSQL |
| `6379` | redis | `127.0.0.1` | Valkey |

## Volumes

| Volume | שירות | תוכן |
| --- | --- | --- |
| `postgres_data` | postgres | תהליכי עבודה, מצלמות, אישורי גישה, התראות, משתמשים, הודעות בתור |
| `storage_data` | storage | קובצי וידאו שהועלו למצלמות File |
| `redis_data` | redis | זרם הזיהויים וערוצי הפקודות; אפשר לאבד בבטחה |
| `api_snapshots` | api | קובצי JPEG של תמונות מצב |
| `api_keys` | api | הקובץ `secret.key` שנוצר |
| `media-cache` | api, mediamtx | קובצי וידאו במטמון למצלמות File |
| `./models` (bind) | api, inference | חבילות מודלים, לקריאה בלבד |

`postgres_data`, `api_keys` ו-`storage_data` מחזיקים את המצב שכדאי לגבות; בלי הסוד, אי אפשר לפענח אישורי גישה שמורים וכל סשן מנותק. [פריסה](deployment.md#גיבויים) מתאר שגרת גיבוי.
