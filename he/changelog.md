[תיעוד](index.md) / יומן שינויים

# יומן שינויים

היסטוריית הגרסאות של LookSee. הגרסאות פועלות לפי ניהול גרסאות סמנטי (semantic versioning), והתאריכים הם תאריכי השחרור. שינויים ב-`main` שטרם שוחררו מפורטים תחת *Unreleased*.

## [Unreleased]

## [0.2.0] - 2026-09-05

שדרוג מ-0.1.0:

- הגדירו `STORAGE_PASSWORD` ב-`.env`. ל-`POSTGRES_PASSWORD` ול-`MTX_MEDIA_PASSWORD` אין עוד ערכי דוגמה וגם אותם יש להגדיר; compose מסרב לעלות כל עוד אחד משלושת הערכים ריק.
- החליפו את `MTX_MEDIA_PASSWORD`: גרסה 0.1.0 שלחה אותה לכל דפדפן שטען את Studio.
- הריצו `docker compose up -d --build` ותנו ל-`api-migrate` להחיל את מיגרציה `0002` ול-`storage-init` ליצור את ה-bucket של הווידאו. עדכנו את ה-API, שירות ההסקה, Studio ו-MediaMTX יחד: MediaMTX מאשר כעת כל חיבור דרך ה-API.
- אירועים אינם ממתינים עוד ל-`EVENT_COOLDOWN_SECONDS` לפני הכניסה לגרף. היכן שזמן הצינון דיכא פעולות חוזרות, הוסיפו מסנן Debounce או זמן צינון של Alert.
- פרופיל ה-compose בשם `docs` ו-`DOCS_PORT` הוסרו; התיעוד חי במאגר זה.

### נוסף

- אחסון וידאו מובנה תואם S3: השירות `storage` (RustFS) עם ה-volume בשם `storage_data`, שנוצר ומוכן על ידי compose, כך שמצלמות File עובדות בלי bucket חיצוני. משתני `S3_*` עדיין מכוונים את ה-API ל-bucket חיצוני.
- תור משלוחים (outbox): הודעות Webhook, Telegram, Discord, Slack, דוא"ל ו-MQTT נשמרות ב-PostgreSQL ומנוסות שוב אחרי כשלים זמניים עם השהיה שמוכפלת עד חמש דקות, ולכל היותר שמונה ניסיונות. `GET /deliveries` מפרט את התור ו-`POST /deliveries/{id}/retry` מכניס משלוח שנכשל לתור מחדש. בקשות Webhook נושאות כותרת `Idempotency-Key`.
- הרשאות מדיה למצלמה אחת: `POST /cameras/{id}/media-access` מנפיק אסימון לחמש דקות לקריאה או לפרסום של מצלמה אחת, ו-MediaMTX מאשר כל חיבור דרך ה-API.
- הגבלת קצב להתחברות ולהגדרה הראשונית: 20 ניסיונות ללקוח ו-100 בסך הכול בדקה, ואחריהם `429` עם `Retry-After`.
- בדיקת Origin: בקשות שמשנות מצב וחיבורי WebSocket שה-`Origin` שלהם אינו תואם לא ל-`CORS_ORIGIN_REGEX` ולא למקור ה-API נדחים עם `403`.
- אינטגרציה רציפה (CI) מריצה בדיקות lint, טסטים, בדיקת המיגרציות, בניית Studio ובדיקת תצורת compose בכל pull request, עם ערכות בדיקה לחוזים המשותפים, ל-API, לשירות ההסקה ולחבילת ה-Enterprise.
- תיעוד המאגר: README, מדריך תרומה, מדיניות אבטחה, קוד התנהגות וטפסי issue. תיעוד זה באנגלית, ברוסית, בעברית ובקוריאנית.

### השתנה

- סיסמת המדיה של MediaMTX היא סוד צד-שרת המשותף ל-API, לשירות ההסקה ול-MediaMTX; Studio אינו מקבל אותה עוד.
- `EVENT_COOLDOWN_SECONDS` מגביל רק את הודעות ה-`event` של הפיד החי; כל אירוע נכנס לגרף.
- כשלי אחסון מחזירים `503` עם הודעה קריאה במקום שגיאה פנימית.
- פריימי זיהוי שתהליך API שנעצר לא אישר מעובדים שוב אחרי דקה, והפריימים מעובדים אחד-אחד.
- `.env.example` מקובץ לפי שירות ומפרט רק את ההגדרות שהתקנה מגדירה; סיסמאות נדרשות ריקות.
- Studio מצרף את הגופן שלו במקום לטעון אותו מ-Google Fonts, שומר עריכות שנעשו בזמן שמירה, ומקשר את **Documentation** למאגר זה.
- MediaMTX 1.20.1.

### תוקן

- החלפת מצלמה מ-RTSP ל-Browser webcam השאירה את MediaMTX מושך את הזרם הישן; תצורת הנתיב מוחלפת כעת במקום להיות מתוקנת חלקית.
- סשני WebRTC נסגרים בעזיבת המוניטור, ושמירת תהליך עבודה אינה מתנגשת עוד בעריכות שנעשו במהלך השמירה.
- כלי ה-lint של Studio למבנה המונו-ריפו והתראת אבטחה על תלות ב-Studio.

## [0.1.0] - 2026-08-06

הגרסה הראשונה.

- עורך תהליכי עבודה עם הצמתים Camera, Detect, If / Else, Class, Size, Zone, Schedule ו-Debounce והפעולות Alert, Snapshot, Webhook, Telegram, Email, MQTT ו-Discord; Count, Line crossing, Dwell ו-Slack במהדורת Enterprise.
- מקורות מצלמה דרך RTSP, RTMP, SRT, WHEP, מצלמות רשת של הדפדפן וקובצי וידאו מספריית נכסים תואמת S3, הנקלטים דרך MediaMTX.
- הסקת ONNX לייצואי D-FINE עם מעקב ByteTrack על CPU, CUDA או CoreML.
- מוניטור חי עם ניגון WebRTC, שכבות-על של זיהויים ואזורים, פיד אירועים והיסטוריית התראות עם תמונות מצב.
- מאגר אישורי גישה מוצפן, חשבון בעלים עם עוגיות סשן ופריסת Docker Compose עם PostgreSQL ו-Valkey.

[Unreleased]: https://github.com/okoflow/looksee/compare/v0.2.0...HEAD
[0.2.0]: https://github.com/okoflow/looksee/compare/v0.1.0...v0.2.0
[0.1.0]: https://github.com/okoflow/looksee/releases/tag/v0.1.0
