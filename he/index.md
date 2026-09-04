# תיעוד LookSee

[English](../en/index.md) · [Русский](../ru/index.md) · **עברית** · [한국어](../ko/index.md)

LookSee היא מערכת ניתוח וידאו באירוח עצמי. היא צופה בשידורים חיים, במצלמות רשת של הדפדפן ובקובצי וידאו, מריצה עליהם זיהוי אובייקטים מבוסס ONNX והופכת זיהויים לאירועים. גרף של תהליך עבודה מחליט מה קורה הלאה: מסננים מצמצמים את האירועים למה שחשוב, ופעולות מעבירות התראות, תמונות מצב והודעות לאנשים ולמערכות שזקוקים להם.

הכול רץ על החומרה שלכם מקובץ Docker Compose אחד. קוד המקור נמצא במאגר [okoflow/looksee](https://github.com/okoflow/looksee) תחת רישיון Apache-2.0; מהדורת Enterprise מוסיפה חציית קו, זמן שהייה, ספירה ו-Slack.

![עורך תהליכי העבודה עם תהליך עבודה לבקרת חבישת קסדות](../images/editor.png)

## מתחילים כאן

- [תחילת העבודה](getting-started.md) — התקנה עם Docker Compose, יצירת חשבון הבעלים והוספת מודל זיהוי.
- [תהליך העבודה הראשון שלכם](first-workflow.md) — מקנבס ריק ועד התראה פעילה.

## מדריך

- [מושגים](concepts.md) — תהליכי עבודה, צמתים, אירועים, מצלמות והרצות.
- [מצלמות](cameras.md) — RTSP, RTMP, SRT, מצלמות רשת של הדפדפן, WHEP וקובצי וידאו.
- [מודלים](models.md) — פורמט החבילה, כיצד תוויות הופכות לאירועים וכיצד לייצא מודל D-FINE ל-ONNX.
- [צמתים](nodes.md) — כל צומת עם השדות, המגבלות וכללי התיקוף שלו.
- [פעולות ואינטגרציות](actions-and-integrations.md) — אישורי גישה, תבניות הודעה, Telegram, Discord, דוא"ל, MQTT, webhooks ו-Slack.
- [ניטור והתראות](monitoring-and-alerts.md) — התצוגה החיה, פיד האירועים, היסטוריית ההתראות ותמונות המצב.

## תפעול

- [תצורה](configuration.md) — כל משתנה סביבה, פורט ו-volume.
- [פריסה](deployment.md) — שרת ברשת שלכם, TLS, מעבדי GPU, גיבויים ושדרוגים.
- [אבטחה](security.md) — חשבונות, סשנים, סודות וגישה למדיה.
- [פתרון בעיות](troubleshooting.md) — מצלמות שנשארות **Pending**, אין וידאו, מודלים חסרים, התראות שלא נמסרו.

## עיון

- [API](api.md) — נקודות קצה HTTP, הודעות WebSocket ופורמט השגיאות.
- [מהדורת Enterprise](enterprise.md) — מהדורות, תכונות ומפתח הרישיון.
- [יומן שינויים](changelog.md) — היסטוריית הגרסאות.

## תרומה לפרויקט

התיעוד כתוב ב-Markdown במאגר [looksee-docs](https://github.com/okoflow/looksee-docs); קובץ ה-README שלו מפרט את כללי הכתיבה. תרומות קוד פועלות לפי [CONTRIBUTING.md](https://github.com/okoflow/looksee/blob/main/CONTRIBUTING.md) במאגר הראשי, ופגיעויות אבטחה מדווחות דרך [דיווח פרטי](https://github.com/okoflow/looksee/security/advisories/new).
