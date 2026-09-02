[תיעוד](index.md) / מהדורת Enterprise

# מהדורת Enterprise

LookSee מסופקת בסט אחד של אימג'ים עם שתי מהדורות. מהדורת Community היא הקוד שמחוץ לספרייה `ee/` של המאגר, מורשה תחת Apache-2.0. מהדורת Enterprise היא הקוד שבתוך `ee/`, מורשה תחת [רישיון LookSee Enterprise](https://github.com/okoflow/looksee/blob/main/ee/LICENSE), ונפתחת באמצעות מפתח רישיון.

## תכונות

| תכונה | צמתים | מה היא מוסיפה |
| --- | --- | --- |
| מסנני מדידה | Count, Line crossing, Dwell | ספירת אובייקטים בתוך אזור מול סף, ספירת חציות של קו עם כיוון, וזיהוי אובייקטים שנשארים באזור מעבר למגבלת זמן. שלושתם נשענים על מעקב אחר אובייקטים בין פריימים. |
| אינטגרציות Enterprise | Slack | הודעות ל-webhook נכנס של Slack, עם אותן תבניות כמו שאר פעולות ההודעות. |

במהדורת Community צמתים אלה מופיעים בפלטה עם מנעול ואי אפשר לגרור אותם אל הקנבס. תהליך עבודה שמכיל אחד מהם, למשל אחרי ייבוא גרף דרך ה-API, נדחה ב-**Run** עם קוד השגיאה `feature_not_licensed` וסטטוס HTTP 402.

## הפעלה

הגדירו את המפתח ב-`.env` ואתחלו את ה-API:

```bash
LICENSE_KEY=<your key>
```

```bash
docker compose up -d api
```

`GET /entitlements` מדווח על המהדורה והתכונות הפעילות:

```json
{ "edition": "enterprise", "features": ["measurement_filters", "enterprise_integrations"] }
```

Studio קורא את אותה נקודת קצה ופותח את הפלטה. תהליכי עבודה שנשמרו עם צמתי Enterprise מתחילים לעבוד ב-**Run** הבא.

## תנאי הרישוי

רישיון Enterprise מתיר העתקה ושינוי של הקוד לפיתוח ולבדיקות בלי מנוי. שימוש בייצור דורש רישיון תקף למספר המושבים (seats). תרומות ל-`ee/` מתקבלות תחת אותו רישיון, כמתואר ב-[CONTRIBUTING.md](https://github.com/okoflow/looksee/blob/main/CONTRIBUTING.md) של המאגר.
