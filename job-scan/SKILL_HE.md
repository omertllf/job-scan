---
name: job-scan
description: >
  סורק את Gmail לאיתור מיילי התראות משרות מ-LinkedIn, Glassdoor, Indeed,
  AllJobs, דרושים ולוחות אחרים. מחלץ ומדדה רשימת משרות, בודק מיילי אישור
  הגשה ודחייה, מדרג כל משרה לפי התאמה לקורות החיים, ממליץ על גרסת קורות
  החיים המתאימה לכל תפקיד, וכותב דוח HTML מעוצב לתיקייה מקומית.
  הפעילו כשמשתמש אומר "סרוק משרות", "הרץ סריקת משרות", "בדוק משרות חדשות",
  "עדכן את רשימת המשרות", "אילו משרות נכנסו", "עדכון חיפוש עבודה",
  "דרג את המשרות שלי", "איזה קורות חיים לשלוח", או כל בקשה לרענן את עוקב המשרות.
compatibility: "דורש Gmail MCP וכלי גישה לקבצים (Desktop Commander MCP או מקבילו) לקריאת קורות חיים מקומיים וכתיבת דוח HTML."
---

# סקיל סריקת משרות

סורק את Gmail לאיתור מיילי התראות משרות, מחלץ ומדדה רשימות, בודק סטטוס
הגשה ודחייה, מדרג כל משרה מול קורות החיים של המשתמש, ממליץ על גרסת
קורות החיים המתאימה, וכותב דוח HTML מעוצב לתיקייה מקומית שהמשתמש בחר.

---

## שלב 0 — הגדרת פרמטרי משתמש

לפני הכל, קבעו את ההגדרות הבאות. אם ערך חסר, שאלו את המשתמש פעם אחת לפני
שמתחילים.

| הגדרה | תיאור |
|---|---|
| `cv_folder` | נתיב מקומי לתיקייה עם קורות החיים |
| `output_folder` | נתיב מקומי לשמירת דוח ה-HTML |
| `report_email` | כתובת מייל לשליחת טיוטת הדוח |
| `target_role` | תפקיד היעד של המשתמש (למשל "מנהל מוצר בכיר", "VP Engineering") |
| `target_domains` | תעשיות/תחומי יעד (למשל "SaaS, FinTech, AI") |
| `target_geography` | העדפת מיקום (למשל "ישראל", "ריחוק", "תל אביב") |
| `scan_days` | כמה ימים אחורה לסרוק (ברירת מחדל: 30) |

אם המשתמש סיפק ערכים אלה בשיחה, השתמשו בהם ישירות. אם זה ריצה חוזרת
עם הגדרות שכבר נקבעו, השתמשו בהן מחדש.

### טעינת קורות חיים

קראו את כל קבצי קורות החיים מ-`cv_folder`. פורמטים נתמכים: `.docx`, `.pdf`, `.txt`.
עבור כל קובץ שנמצא, חלצו:
- **תווית** — שם גרסה קצר מתוך שם הקובץ (למשל "קו"ח_PM", "Resume_v3")
- **תאריך עדכון אחרון** — ממטא-דאטה של מערכת הקבצים
- **כיוון תפקיד** — באיזה תפקיד/בכירות קורות החיים מתמקדים?
- **מילות מפתח מובילות** — תחומים, טכנולוגיות, מתודולוגיות בולטות
- **ייחוד** — מה מבדיל גרסה זו מאחרות?

אם לא נמצאו קורות חיים, דלגו על שלבים 0 ו-4, ציינו זאת בדוח, והמשיכו.

---

## שלב 1 — חיפוש מיילי התראות ב-Gmail

הריצו את חיפושי Gmail הבאים דרך `search_threads`. התאימו לפי לוחות המשרות
שאליהם המשתמש רשום. אספו את כל ה-thread IDs.

### דפוסי שולח נפוצים (התאימו למקורות האמיתיים של המשתמש)

```
from:jobalerts-noreply@linkedin.com newer_than:{scan_days}d
from:jobs-listings@linkedin.com newer_than:{scan_days}d
from:linkedin@e.linkedin.com subject:jobs newer_than:{scan_days}d
from:noreply@glassdoor.com newer_than:{scan_days}d
from:alerts@glassdoor.com newer_than:{scan_days}d
from:alert@indeed.com newer_than:{scan_days}d
from:jobalert@indeed.com newer_than:{scan_days}d
from:Alljobs@alljob.co.il newer_than:{scan_days}d
from:noreply@drushim.co.il newer_than:{scan_days}d
```

אם מקור לא מחזיר תוצאות, סמנו אותו — ייתכן שכתובת השולח השתנתה. המשיכו
עם המקורות שכן החזירו תוצאות.

חפשו גם בפח הזבל של LinkedIn להודעות שנמחקו בטעות:
```
in:trash from:jobalerts-noreply@linkedin.com newer_than:{scan_days}d
in:trash from:jobs-listings@linkedin.com newer_than:{scan_days}d
```

משכו את גוף המייל המלא דרך `get_thread`. עבור כל משרה שנמצאה, חלצו:

| שדה | הערות |
|---|---|
| `title` | כותרת המשרה כפי שנכתבה |
| `company` | שם החברה |
| `location` | עיר / ריחוק / היברידי |
| `source` | LinkedIn / Glassdoor / Indeed / AllJobs / דרושים / אחר |
| `url` | קישור ישיר למשרה אם קיים |
| `date_found` | תאריך מייל ההתראה |
| `description_snippet` | 300 התווים הראשונים של תיאור המשרה אם זמינים |
| `from_trash` | true אם שוחזר מפח הזבל |

---

## שלב 2 — כפילויות

כפילות היא כל שתי שורות שבהן **גם** `title` **וגם** `company` זהים
(ללא תלות ברישיות, התעלמו מפיסוק). בעת מיזוג:
- שמרו את השורה עם `date_found` המוקדם ביותר
- העדיפו את ה-URL המפורט יותר
- מזגו את שדה `source` כדי להציג את כל המקורות (למשל "LinkedIn, Glassdoor")
- שמרו את `description_snippet` הארוך יותר
- אם אחד הגיע מפח הזבל ואחד מהתיבה, שמרו את גרסת התיבה

---

## שלב 3 — בדיקת סטטוס הגשה

### 3א — מיילי אישור הגשה

```
subject:("application received" OR "application submitted" OR "we received your application" OR "thank you for applying" OR "תודה על פנייתך" OR "הגשת מועמדות") newer_than:{scan_days}d
from:jobs-noreply@linkedin.com subject:"application" newer_than:{scan_days}d
```

חלצו שם חברה ותפקיד מכל מייל אישור.

### 3ב — מיילי דחייה

חפשו **גם בתיבת הדואר וגם בפח הזבל**:
```
subject:("we regret" OR "not moving forward" OR "other candidates" OR "position has been filled" OR "unfortunately" OR "לא התקדמנו" OR "לא נבחרת" OR "we have decided") newer_than:{scan_days}d
in:trash subject:("regret" OR "not moving forward" OR "unfortunately" OR "other candidates") newer_than:{scan_days}d
```

סמנו דחיות לאחר ראיון (כשהמייל או הנושא מציין שהייתה ראיון לפני הדחייה).

### 3ג — התאמת סטטוס לרשימת המשרות

לכל משרה ברשימה, קבעו `status`:
- `הוגש` — מייל אישור מתאים לחברה + תפקיד זה
- `נדחה` — מייל דחייה מתאים
- `קשר מגייס` — הגיע דרך LinkedIn Recruiter InMail
- `חדש` — אין התאמה

התאמה מטושטשת על שם חברה (התעלמו מסיומות בע"מ/Ltd/Inc). כשיש ספק — ברירת
המחדל היא `חדש`.

---

## שלב 4 — דירוג התאמה והמלצת גרסת קורות חיים

**הריצו שלב זה רק אם נטענו קורות חיים בשלב 0.**

הערכת ההתאמה מבוססת על `target_role`, `target_domains` ו-`target_geography`
שהוגדרו בשלב 0.

### מימדי ניקוד התאמה

| מימד | מה להעריך |
|---|---|
| התאמת בכירות | האם רמת התפקיד מתאימה ליעד המשתמש? |
| התאמת תחום | האם תעשיית התפקיד מתאימה ל-`target_domains`? |
| התאמת היקף | גודל צוות, P&L, גיאוגרפיה — מתאים לרקע? |
| גיאוגרפיה | האם התפקיד ב-`target_geography` או remote? |
| התאמת תואר | התאמה ריאלית, מאמץ, או מתחת לרמה? |

### רמות התאמה

- **גבוהה** — התאמה חזקה בבכירות + לפחות 2 אותות תחום. עדיפות להגשה.
- **בינונית** — התאמה חלקית. בכירות מתאימה אך תחום סמוך, או להיפך.
- **נמוכה** — התאמה חלשה. פער בכירות משמעותי, תחום לא קשור, או בעיית מיקום.

### המלצת גרסת קורות חיים

השוו את השפה והדגשים של המשרה מול הייחוד של כל גרסת קורות חיים (מהשלב 0).
המליצו על הגרסה שהמיצוב שלה הכי קרוב לשפת המשרה. אם יש רק גרסה אחת —
המליצו עליה לכל התאמות גבוה/בינוני.

### נימוק התאמה

משפט אחד (עד 15 מילים):
- "התאמה חזקה ל-AI delivery, היקף גלובלי, בכירות מיושרת"
- "התאמת תחום טובה אבל תואר נמוך ממטרה ברמה"
- "תחום לא קשור — רק אם החברה היא יעד אסטרטגי"

---

## שלב 5 — כתיבת דוח HTML

כתבו קובץ HTML עצמאי דרך כלי גישה לקבצים:

**שם קובץ:** `Job_Scan_{YYYY-MM-DD}.html`
**נתיב:** `{output_folder}\Job_Scan_{YYYY-MM-DD}.html`

### CSS מוטמע (הכניסו verbatim ל-`<head>`)

```css
body { font-family: Segoe UI, Arial, sans-serif; background: #f0f4f8; margin: 0; padding: 20px; color: #1a1a2e; }
h1 { background: #1F4E79; color: white; padding: 16px 24px; border-radius: 8px; margin-bottom: 6px; font-size: 1.4em; }
.subtitle { color: #555; margin-bottom: 20px; font-size: 0.9em; padding-left: 4px; }
.stats { display: flex; gap: 12px; margin-bottom: 20px; flex-wrap: wrap; }
.stat { background: white; border-radius: 8px; padding: 12px 20px; box-shadow: 0 1px 4px rgba(0,0,0,0.1); text-align: center; min-width: 100px; }
.stat .num { font-size: 1.8em; font-weight: bold; }
.stat .lbl { font-size: 0.75em; color: #666; margin-top: 2px; }
.green .num { color: #2e7d32; } .yellow .num { color: #f57f17; } .red .num { color: #c62828; }
.blue .num { color: #1565c0; } .purple .num { color: #6a1b9a; }
.alert-box { background: #fff3cd; border: 1px solid #ffc107; border-radius: 8px; padding: 12px 16px; margin-bottom: 20px; font-size: 0.9em; }
.alert-box strong { color: #856404; }
section { margin-bottom: 28px; }
h2 { font-size: 1em; font-weight: bold; padding: 8px 14px; border-radius: 6px; margin: 0 0 10px 0; }
.h-high { background: #c6efce; color: #1a5c1a; }
.h-med  { background: #ffeb9c; color: #7d5a00; }
.h-low  { background: #ffc7ce; color: #7d0000; }
.h-track { background: #dde3ea; color: #333; }
.h-cv { background: #375623; color: white; }
table { width: 100%; border-collapse: collapse; background: white; border-radius: 8px; overflow: hidden; box-shadow: 0 1px 4px rgba(0,0,0,0.1); font-size: 0.82em; }
th { background: #2E75B6; color: white; padding: 8px 10px; text-align: left; }
td { padding: 9px 10px; border-bottom: 1px solid #e8e8e8; vertical-align: top; line-height: 1.4; }
tr:last-child td { border-bottom: none; }
tr:hover td { background: #f5f9ff; }
.fit-h { background: #c6efce; color: #1a5c1a; font-weight: bold; padding: 2px 8px; border-radius: 12px; white-space: nowrap; }
.fit-m { background: #ffeb9c; color: #7d5a00; font-weight: bold; padding: 2px 8px; border-radius: 12px; white-space: nowrap; }
.fit-l { background: #ffc7ce; color: #7d0000; font-weight: bold; padding: 2px 8px; border-radius: 12px; white-space: nowrap; }
.status-new  { background: #e3f2fd; color: #0d47a1; padding: 2px 6px; border-radius: 10px; font-size: 0.85em; }
.status-appl { background: #BDD7EE; color: #00008b; padding: 2px 6px; border-radius: 10px; font-size: 0.85em; }
.status-rej  { background: #e0e0e0; color: #333; padding: 2px 6px; border-radius: 10px; font-size: 0.85em; }
.status-rec  { background: #e2efda; color: #2e5e1a; padding: 2px 6px; border-radius: 10px; font-size: 0.85em; }
.cv-tag { background: #1F4E79; color: white; padding: 1px 6px; border-radius: 4px; font-size: 0.8em; white-space: nowrap; }
.src-li { background: #0077b5; color: white; padding: 1px 5px; border-radius: 3px; font-size: 0.75em; }
.src-gd { background: #0caa41; color: white; padding: 1px 5px; border-radius: 3px; font-size: 0.75em; }
.src-in { background: #2557a7; color: white; padding: 1px 5px; border-radius: 3px; font-size: 0.75em; }
.src-aj { background: #ef821b; color: white; padding: 1px 5px; border-radius: 3px; font-size: 0.75em; }
.src-dr { background: #6d28d9; color: white; padding: 1px 5px; border-radius: 3px; font-size: 0.75em; }
.src-ot { background: #607d8b; color: white; padding: 1px 5px; border-radius: 3px; font-size: 0.75em; }
.star { color: #ffa000; }
.warn { color: #e65100; font-weight: bold; }
.notes { color: #555; font-size: 0.85em; }
.rej-pi { background: #ff9999; font-weight: bold; }
```

### מבנה הדוח (סעיפים לפי סדר)

1. `<h1>📋 סריקת משרות יומית — {DD בחודש YYYY}</h1>`
2. `.subtitle`: מקורות שנסרקו + נתיב תיקיית קורות החיים
3. שורת `.stats`: התאמה גבוהה, בינונית, נמוכה, הוגש, נדחה, מגייס
4. `.alert-box` (רק אם יש פריטים דחופים — ראו חוקים למטה)
5. **גבוהה** (`h2.h-high "🟢 התאמה גבוהה — הגישו עכשיו"`) — טבלת 10 עמודות
6. **בינונית** (`h2.h-med "🟡 התאמה בינונית — לבדיקה"`) — טבלת 10 עמודות
7. **נמוכה** (`h2.h-low "🔴 התאמה נמוכה — לדלג"`) — 5 עמודות: תאריך, כותרת, חברה, מקור, סיבה לדלג
8. **עוקב הגשות** (`h2.h-track "📊 עוקב הגשות"`) — 5 עמודות: תאריך, כותרת, חברה, תוצאה, הערות
9. **מפתח קורות חיים** (`h2.h-cv "📄 מפתח גרסאות קורות חיים"`) — 4 עמודות: תווית, קובץ, מתאים ל, דוגמאות היום
10. כותרת תחתית: `נוצר על ידי job-scan — {תאריך}`

### סדר עמודות לטבלה המלאה (גבוה ובינוני)

תאריך | כותרת משרה | חברה | מיקום | מקור | התאמה | סטטוס | קו"ח מומלץ | נימוק | הערות

### כיתות תג מקור

- LinkedIn → `<span class="src-li">LinkedIn</span>`
- Glassdoor → `<span class="src-gd">Glassdoor</span>`
- Indeed → `<span class="src-in">Indeed</span>`
- AllJobs → `<span class="src-aj">AllJobs</span>`
- דרושים → `<span class="src-dr">דרושים</span>`
- אחר → `<span class="src-ot">{שם מקור}</span>`

הוסיפו ציונים כטקסט רגיל: "HOT", "Easy Apply", "(פח — הוחמץ)", "מגייס"

### חוקי תיבת התראה

כללו רק לפריטים דחופים באמת:
- פניות מגייס שלא נענו מעל 3 ימים
- משרות HOT שפורסמו היום ומצדיקות הגשה באותו יום
- משרות שהוחזרו מפח הזבל ועשויות להיות רגישות לזמן

השמיטו לחלוטין אם אין פריטים דחופים.

---

## שלב 6 — דיווח למשתמש

לאחר כתיבת קובץ ה-HTML, ענו עם:

```
סריקה הושלמה — {תאריך}

📋 {N} משרות ייחודיות נמצאו
🟢 {N} התאמה גבוהה  |  🟡 {N} בינונית  |  🔴 {N} נמוכה
✅ {N} הוגשו  |  ❌ {N} נדחו  |  🆕 {N} חדשות

מקורות: {רשימת מקורות עם ספירות}
גרסאות קורות חיים לדירוג: {תוויות}

📄 דוח נשמר: {output_folder}\Job_Scan_{date}.html

🎯 המשרות הטובות ביותר (התאמה גבוהה, חדשות):
  • {כותרת} @ {חברה} — {נימוק} → השתמשו ב-{קו"ח}
  • ...

⚠️ דגלים (אם יש):
  - לא נמצאו מיילים מ-[מקור] — כתובת השולח עשויה השתנות
  - {N} דחיות לא הותאמו למשרה ספציפית
  - תיקיית קורות חיים ריקה — דירוג דולג
```

רשמו עד 5 משרות מובילות (התאמה גבוהה + סטטוס חדש).

---

## שלב 7 — טיוטת מייל

צרו טיוטת Gmail דרך Gmail MCP. **אל תשלחו אוטומטית.**

- **אל:** `{report_email}`
- **נושא:** `דוח סריקת משרות — {תאריך היום}`
- **גוף:**

```
דוח סריקת משרות — {תאריך}

סיכום
-----
סה"כ משרות ייחודיות: {N}
🟢 התאמה גבוהה: {N}  |  🟡 בינונית: {N}  |  🔴 נמוכה: {N}
✅ הוגשו: {N}  |  ❌ נדחו: {N}  |  🆕 חדשות: {N}

מקורות: {פירוט}
גרסאות קורות חיים: {תוויות}

המשרות המובילות (התאמה גבוהה, חדשות)
--------------------------------------
1. {כותרת} @ {חברה} — {מיקום}
   התאמה: {נימוק}
   קורות חיים לשימוש: {תווית}
   קישור: {url}

(עד 5 משרות)

דגלים
------
{כל אזהרה}

דוח מלא: {output_folder}\Job_Scan_{date}.html
```

לאחר יצירת הטיוטה, ציינו: "טיוטה נוצרה — בדקו ושלחו כשמוכנים."

---

## מקרי קצה

- **לא נמצאו מיילים כלל**: עצרו ועדכנו את המשתמש. אל תכתבו דוח ריק.
- **כישלון חלקי במקור**: המשיכו עם המקורות הזמינים, סמנו את החסרים.
- **אין קורות חיים**: דלגו על שלבים 0 ו-4. השאירו התאמה, קו"ח מומלץ, נימוק כ-"—".
- **גרסת קורות חיים אחת בלבד**: הריצו בכל זאת דירוג — ציון ההתאמה שימושי גם ללא השוואה.
- **גוף מייל בעברית**: פרסו כמות שהוא. אל תתרגמו כותרות או שמות חברות.
- **LinkedIn Recruiter InMail**: מגיע כ-`messages-noreply@linkedin.com`. מקור = "LinkedIn Recruiter", סטטוס = "קשר מגייס". דרגו אם יש תיאור.
- **אותה משרה ממספר מקורות**: מזגו לשורה אחת, שלבו מקורות, שמרו דירוג.
- **תיאור קצר מדי לדירוג**: סמנו התאמה כ-"?" ורשמו "תיאור לא מספיק לדירוג".
- **משרות שנמחקו אוטומטית לפח**: חפשו גם בפח, הציגו בעוקב ההגשות עם דגל "הוחמץ" ו-⚠️.
- **ריצה ראשונה / ללא הגדרה קודמת**: שאלו על `cv_folder`, `output_folder`, `report_email`, `target_role`, `target_domains` ו-`target_geography` לפני הסריקה.
