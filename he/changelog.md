[תיעוד](index.md) / יומן שינויים

# יומן שינויים

היסטוריית הגרסאות של LookSee. הגרסאות פועלות לפי ניהול גרסאות סמנטי (semantic versioning); רשומות תחת *טרם שוחרר* נמצאות ב-`main` ומסופקות עם הגרסה הבאה.

## טרם שוחרר

- אינטגרציה רציפה (CI) מריצה בדיקות lint, טסטים, בדיקת המיגרציות ובניית Studio בכל pull request.
- ערכות בדיקה לחוזים המשותפים, ל-API, לשירות ההסקה ולחבילת ה-Enterprise.
- תיעוד המאגר: README, מדריך תרומה, מדיניות אבטחה, קוד התנהגות וטפסי issue.
- תיעוד באנגלית, ברוסית, בעברית ובקוריאנית.
- כלי ה-lint של Studio תוקנו למבנה המונו-ריפו והתראת אבטחה על תלות נפתרה.

## 0.1.0

הגרסה הראשונה.

- עורך תהליכי עבודה עם הצמתים Camera, Detect, If / Else, Class, Size, Zone, Schedule ו-Debounce והפעולות Alert, Snapshot, Webhook, Telegram, Email, MQTT ו-Discord; Count, Line crossing, Dwell ו-Slack במהדורת Enterprise.
- מקורות מצלמה דרך RTSP, RTMP, SRT, WHEP, מצלמות רשת של הדפדפן וקובצי וידאו מספריית נכסים תואמת S3, הנקלטים דרך MediaMTX.
- הסקת ONNX לייצואי D-FINE עם מעקב ByteTrack על CPU, CUDA או CoreML.
- מוניטור חי עם ניגון WebRTC, שכבות-על של זיהויים ואזורים, פיד אירועים והיסטוריית התראות עם תמונות מצב.
- מאגר אישורי גישה מוצפן, חשבון בעלים עם עוגיות סשן ופריסת Docker Compose עם PostgreSQL ו-Valkey.
