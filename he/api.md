[תיעוד](index.md) / API

# API

ה-API הוא שירות FastAPI בפורט `8000`. Studio הוא הלקוח המובנה היחיד שלו, וכל מה ש-Studio עושה זמין לסקריפטים ולאינטגרציות. תיעוד אינטראקטיבי עם סכמות הבקשות והתגובות מוגש ב-`/docs`, ומסמך ה-OpenAPI ב-`/openapi.json`.

## אימות

כל נקודות הקצה מלבד `/health`, `/auth/*`, `/docs` ו-`/openapi.json` דורשות את העוגייה `looksee_session`. קבלו אותה בהתחברות והשתמשו בה שוב במשך שבעה ימים:

```bash
curl -c cookies.txt -H 'content-type: application/json' \
  -d '{"email":"owner@example.com","password":"..."}' \
  http://127.0.0.1:8000/auth/login

curl -b cookies.txt http://127.0.0.1:8000/workflows
```

## שגיאות

שגיאות מחזירות JSON עם הודעת `detail`. שגיאות בגרף של תהליך עבודה מוסיפות `code` יציב ואת `node_id` של הצומת הבעייתי, כדי שלקוח יוכל למקד אותו:

```json
{ "detail": "camera 'camera' reaches no detect node", "code": "detect_node_missing", "node_id": "camera" }
```

| סטטוס | משמעות |
| --- | --- |
| `401` | סשן חסר או לא תקין; דוא"ל או סיסמה שגויים |
| `402` | הגרף משתמש בצומת Enterprise בלי רישיון (`feature_not_licensed`) |
| `404` | תהליך עבודה, התראה, אישור גישה או נכס לא מוכרים |
| `409` | חשבון הבעלים כבר קיים |
| `422` | התיקוף נכשל: שדה מחוץ לטווח, או שהגרף אינו יכול לרוץ (`code` מוגדר) |
| `503` | ספריית הנכסים אינה מוגדרת |

[צמתים](nodes.md#תיקוף) מפרט כל קוד שגיאה של גרף.

## נקודות קצה

### תקינות והגדרה ראשונית

| שיטה ונתיב | תיאור |
| --- | --- |
| `GET /health` | מחזיר `{"status":"ok"}`; ללא אימות |
| `GET /auth/status` | `{"requires_setup": true}` עד שהבעלים קיים |
| `POST /auth/setup` | יצירת הבעלים: `email`, `name`, `password`. מגדיר את עוגיית הסשן. |
| `POST /auth/login` | `email`, `password`. מגדיר את עוגיית הסשן. |
| `POST /auth/logout` | מנקה את העוגייה |
| `GET /auth/me` | המשתמש המחובר |
| `GET /entitlements` | `{"edition": "community", "features": []}` או תכונות ה-Enterprise |

### מודלים

| שיטה ונתיב | תיאור |
| --- | --- |
| `GET /models` | החבילות שהתגלו: `id`, `name`, `classes` (`class_id`, `label`, `event_kind`), `recommended_confidence_threshold` |

### תהליכי עבודה

| שיטה ונתיב | תיאור |
| --- | --- |
| `GET /workflows` | כל תהליכי העבודה עם המצלמות והמצבים שלהם, החדשים ראשונים |
| `POST /workflows` | יצירה: `name`, `description` אופציונלי, `graph` אופציונלי. מחזיר `201`. |
| `GET /workflows/{id}` | תהליך עבודה אחד |
| `PATCH /workflows/{id}` | עדכון חלקי של `name`, `description`, `enabled`, `graph`. הגדרת `enabled` ל-`true` מתקפת את הגרף ומפעילה את המצלמות. |
| `DELETE /workflows/{id}` | עצירה ומחיקה; מחזיר `204` |

גרף הוא `{"nodes": [...], "edges": [...], "comments": [...]}`. לכל צומת יש `id`, `position` עם `x` ו-`y`, ו-`data` שה-`kind` שלו בוחר את סוג הצומת ושאר שדותיו מפורטים ב[צמתים](nodes.md). לכל קשת יש `id`, `source`, `target` ו-`branch` אופציונלי בערך `if` או `else` לקשתות שיוצאות ממסנן.

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

### התראות

| שיטה ונתיב | תיאור |
| --- | --- |
| `GET /alerts` | החדשות ראשונות. מסננים: `camera_id`, `workflow_id`, `severity` (`info`, `warning`, `critical`), `limit` (1 עד 500, ברירת מחדל 100) |
| `DELETE /alerts/{id}` | מחיקת התראה אחת |
| `DELETE /alerts` | מחיקת התראות, אופציונלית בהיקף של `camera_id` או `workflow_id` |

להתראה יש `id`, `workflow_id`, `camera_id`, `kind`, `severity`, `message`, `payload` (חותמת זמן, זיהויים, מטא-נתונים, `snapshot_url` כאשר Snapshot רץ לפני ה-Alert) ו-`created_at`.

### אישורי גישה

| שיטה ונתיב | תיאור |
| --- | --- |
| `GET /credentials` | `id`, `name`, `type`, `summary`, חותמות זמן. תוכני ה-payload לעולם אינם מוחזרים. |
| `POST /credentials` | `name`, `type`, `payload`. שדות ה-payload לכל סוג נמצאים ב[פעולות ואינטגרציות](actions-and-integrations.md#אישורי-גישה). |
| `PATCH /credentials/{id}` | עדכון חלקי של `name` ו-`payload`; השמטת `payload` שומרת על הסוד הקיים |
| `DELETE /credentials/{id}` | מחיקה |

### נכסים

זמין כשספריית הנכסים מוגדרת; אחרת כל קריאה מחזירה `503`.

| שיטה ונתיב | תיאור |
| --- | --- |
| `GET /assets` | האובייקטים ב-bucket |
| `POST /assets` | העלאת multipart עם שדה `file`; מחזיר `201` |
| `GET /assets/{key}/content` | הקובץ במטמון, עם בקשות טווח (range) לניגון |
| `DELETE /assets/{key}` | מחיקת האובייקט |

### תמונות מצב

`GET /snapshots/<file>.jpg` מגיש תמונות שנכתבו על ידי פעולת Snapshot. הבקשה זקוקה לעוגיית הסשן.

## WebSocket

`ws://<host>:8000/ws/cameras/{camera_id}` מזרים הודעות חיות עבור מצלמה אחת ללקוח שמציג את עוגיית הסשן. ה-socket הוא חד-כיווני; מהודעות נכנסות מתעלמים. כל הודעה היא JSON עם `type`:

| סוג | שדות | מתי |
| --- | --- | --- |
| `detections` | `ts`, `frame_width`, `frame_height`, `detections[]` | כל פריים מעובד |
| `event` | `kind`, `ts`, `frame_width`, `frame_height`, `detections[]` | כל אירוע שעובר את זמן הצינון של האירועים |
| `worker` | `status`, `ts`, `reason` | המצלמה משנה מצב |
| `alert` | `id`, `kind`, `severity`, `message`, `ts`, `snapshot_url` | פעולת Alert מופעלת |

לזיהוי יש `label`, `bounding_box` בצורה `[x_min, y_min, x_max, y_max]` בפיקסלי הפריים, `confidence`, `class_id` ו-`tracker_id` (או `null`).

## ה-payload של Webhook

פעולות Webhook ו-MQTT מוסרות JSON זה לכל אירוע:

```json
{
  "camera_id": "01a061c1-ff68-7611-af72-436d9d5ba907",
  "kind": "HELMET_DETECTED",
  "ts": "2026-09-02T10:58:20.983914+00:00",
  "metadata": { "count": 2, "model_id": "ppe-helmets" },
  "snapshot_url": "/snapshots/20260902-105820-01a061c1-3f9a2c1e.jpg"
}
```

`snapshot_url` קיים רק כאשר פעולת Snapshot רצה קודם לכן בגרף, והוא יחסי למקור ה-API.
