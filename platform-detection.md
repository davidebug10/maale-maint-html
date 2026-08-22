# זיהוי פלטפורמה (Platform Detection) — מעלה המשלוחים

> **מסמך ידע פנימי** | עודכן לאחרונה: 2026-07-26
> מרכז את כל הידע שנצבר בפרויקט על זיהוי הסביבה שבה רץ הקוד:
> אפליקציית אנדרואיד / אפליקציית אייפון / דפדפן נייד / דפדפן מחשב —
> ואיך מכוונים CSS, JS ו-HTML לפלטפורמה אחת בלבד בלי לפגוע באחרות.

---

## תוכן עניינים

1. [שתי מערכות זיהוי נפרדות — ההבחנה הקריטית](#1-שתי-מערכות-זיהוי-נפרדות)
2. [הסיגנלים האמינים — טבלת האמת](#2-הסיגנלים-האמינים)
3. [סיגנלים שקרניים — מה שנכווינו ממנו](#3-סיגנלים-שקרניים)
4. [מערכת 1: תיוג פלטפורמה באתר (mh-android-app / mh-ios-app)](#4-מערכת-1-תיוג-באתר)
5. [מערכת 2: זיהוי עצמאי בדף ה-Maintenance (iframe)](#5-מערכת-2-דף-ה-maintenance)
6. [מתכוני זיהוי מוכנים לשימוש](#6-מתכוני-זיהוי)
7. [מצב דיבאג (?debug=1)](#7-מצב-דיבאג)
8. [הבדלי רינדור בין פלטפורמות (Quirks)](#8-הבדלי-רינדור)
9. [הקשר: Deep Links ולמה JS לא יכול לפתוח אפליקציה מגוגל](#9-deep-links)
10. [כללי עבודה — סיכום מהיר](#10-כללי-עבודה)

---

## 1. שתי מערכות זיהוי נפרדות

בפרויקט קיימות **שתי מערכות זיהוי שונות לחלוטין**, שרצות בהקשרים שונים ואינן מדברות זו עם זו. בלבול ביניהן הוא מקור הטעות מספר 1.

### מערכת א' — האתר עצמו (Hyperzod DOM)

- חיה בקובץ `custom-footer.js` (נטען דרך GitHub Pages CDN לתוך Hyperzod).
- מזריקה מחלקות על תגית `<html>` של האתר: `mh-android-app` / `mh-ios-app`.
- מאפשרת ל-CSS ב-`global-cdn.css` להתייחס לפלטפורמה אחת:
  `html.mh-android-app .selector { ... }`
- **תקפה רק בדפי האתר/האפליקציה של Hyperzod.**

### מערכת ב' — דף ה-Maintenance / השקה (iframe עצמאי)

- דף HTML עצמאי שמתארח ב-GitHub Pages ומוטמע ב-Hyperzod דרך **iframe**.
- `custom-footer.js` ו-`global-cdn.css` **לא נטענים בתוכו** — ה-iframe הוא document נפרד לגמרי.
- לכן המחלקות `mh-android-app` / `mh-ios-app` **לא קיימות שם**, והדף חייב לזהות את הסביבה **בעצמו**, עם קוד זיהוי משלו בתוך ה-`index.html`.
- יתרון ייחודי ל-iframe: `document.referrer` חושף מי "ההורה" שמארח אותו — וזה הסיגנל הכי חזק שיש לנו (ראו סעיף 2).

> **כלל אצבע:** כל דף שיושב ב-iframe (maintenance, דפי נחיתה עתידיים) — זיהוי עצמאי בתוך הדף. כל דבר שרץ בתוך ה-DOM של Hyperzod — משתמשים במחלקות ה-`mh-*` הקיימות.

---

## 2. הסיגנלים האמינים

טבלת האמת, כפי שאומתה בבדיקות אמיתיות על מכשירים אמיתיים:

| סביבה | User Agent | סיגנלים אמינים |
|---|---|---|
| **אפליקציית אנדרואיד** (Chromium WebView) | מכיל `Android` **וגם** הטוקן `wv` | `/\bwv\b/.test(ua)` ✅<br>ב-iframe: `document.referrer` מכיל `hyperzod` ✅ |
| **דפדפן כרום באנדרואיד** | מכיל `Android`, `Chrome/`, **וגם** `Safari/` | `Android` ללא `wv` ✅ |
| **אפליקציית אייפון** (WKWebView) | מכיל `iPhone` אבל **חסר** הטוקן `Safari/` | `/iPhone|iPad|iPod/i.test(ua) && !/Safari\//.test(ua)` ✅<br>ב-iframe: `document.referrer` מכיל `hyperzod` ✅ |
| **ספארי באייפון** | מכיל `iPhone` **וגם** `Safari/604.1` | iPhone עם `Safari/` ✅ |
| **דפדפן מחשב** | ללא `iPhone`/`Android` | היעדר שני הטוקנים ✅ |

### פירוט הסיגנלים

**א. הטוקן `wv` (אנדרואיד WebView)**
ה-User Agent של WebView באנדרואיד מכיל את הטוקן `wv` (קיצור של webview). דוגמה אמיתית לתבנית:
`... Android 14; ... wv) AppleWebKit/537.36 ... Chrome/... Mobile Safari/537.36`

חשוב: הבדיקה המדויקת היא עם גבולות מילה — `/\bwv\b/` — ולא `/wv/i` "רחב", שעלול לתפוס את האותיות wv בתוך מילים אחרות ב-UA. שתי הגרסאות הופיעו בקוד שלנו בתקופות שונות; **הגרסה עם `\b` היא הנכונה והעדכנית** (יושמה בתיקון דף ה-Maintenance ביולי 2026).

**ב. היעדר `Safari/` (אפליקציית אייפון)**
WKWebView באפליקציה לא כולל את הטוקן `Safari/` ב-UA — רק `AppleWebKit`. ספארי אמיתי תמיד כולל `Safari/604.1` (או דומה).
UA אמיתי שנמדד בספארי-אייפון בפרויקט:
`Mozilla/5.0 (iPhone; CPU iPhone OS 18_5 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/18.5 Mobile/15E148 Safari/604.1`

⚠️ **הבדיקה הזאת תקפה רק לאייפון.** באנדרואיד גם כרום רגיל כולל את הטוקן `Safari/` — אז "חסר Safari" איננו סימן לאפליקציה באנדרואיד.

**ג. `document.referrer` — הסיגנל החזק ביותר (רק בתוך iframe)**
האפליקציה של Hyperzod טוענת את האתר מהדומיין `xcfqtqgd.hyperzod.app` (ולא מ-`www.maalehamishlohim.co.il` שדרכו מגיעים גולשי דפדפן). דף שיושב ב-iframe רואה את דומיין ההורה ב-`document.referrer`:

```javascript
var fromApp = /hyperzod/i.test(document.referrer || '');
```

- זהו זיהוי חד-משמעי של "אני בתוך האפליקציה" — בלי לנחש לפי דפדפנים וגרסאות UA.
- עובד גם באנדרואיד וגם באייפון.
- **מוגבל ל-iframe בלבד** — בדפי האתר הרגילים ה-referrer מתנהג אחרת ולא משמש לזיהוי.

---

## 3. סיגנלים שקרניים

רשימת הבדיקות שנוסו, נכשלו, ו**אסור לחזור אליהן**:

### ❌ `window.nativeVibrateShort`
נראה כמו פונקציית גשר-נייטיב שקיימת רק באפליקציה — אבל **Hyperzod מזריקים אותה גם בדפדפן רגיל** (נמדד: `function` גם בספארי-אייפון על הדומיין שלנו). חסרת ערך לזיהוי.

### ❌ `window.webkit.messageHandlers`
עבד יפה בתוך דפי האתר (undefined בספארי, object ב-WKWebView) — אבל **בדף ה-Maintenance נתן false positive**: בחלק מגרסאות ספארי-אייפון האובייקט קיים גם בדפדפן הרגיל. התוצאה: כפתורי הורדת האפליקציה הוסתרו בטעות מגולשי דפדפן בנייד. הבדיקה **נמחקה לחלוטין** מדף ה-Maintenance בתיקון של 2026-07-11 והוחלפה בזיהוי רב-סיגנלים. לא להחזיר אותה.

### ❌ `window.navigator.standalone` / `display-mode: standalone`
נמדדו בפועל: מחזירים `undefined` / `false` גם במצבים שרצינו לזהות. רלוונטיים ל-PWA, לא לאפליקציות Hyperzod.

### ❌ הסתמכות על קיום אלמנט DOM (למשל `#MainFooter`)
בעבר זיהינו "דפדפן" לפי קיום הפוטר של האתר. Hyperzod שינו את ה-DOM וה-ID נעלם — הזיהוי נשבר בשקט. אלמנטים של הפלטפורמה אינם חוזה יציב.

---

## 4. מערכת 1: תיוג באתר

הסקריפט שחי ב-`custom-footer.js` (הותקן 2026-06-27):

```javascript
/* Platform Detection - תיוג פלטפורמה על תגית <html> */
(function() {
  'use strict';
  var html = document.documentElement;
  var ua = navigator.userAgent || '';

  // אפליקציית אייפון - WKWebView חושף את window.webkit.messageHandlers
  var isIOSApp = !!(window.webkit && window.webkit.messageHandlers);

  // אפליקציית אנדרואיד - ה-User Agent של WebView מכיל "wv"
  var isAndroidApp = /wv/i.test(ua);

  if (isAndroidApp) { html.classList.add('mh-android-app'); }
  if (isIOSApp) { html.classList.add('mh-ios-app'); }
})();
```

### הערות חשובות על הסקריפט הזה

1. **בהקשר של דפי האתר** בדיקת `webkit.messageHandlers` עבדה באמינות בבדיקות שלנו (ה-false positive התגלה בדף ה-Maintenance). אם אי-פעם יתגלה זיהוי שגוי גם באתר — לשדרג לשיטת "היעדר `Safari/`" כמו בדף ה-Maintenance.
2. הבדיקה כאן היא `/wv/i` הישנה. אם עורכים את הסקריפט בעתיד — לשדרג ל-`/\bwv\b/`.
3. `mh-android-app` תופס את **האפליקציה בלבד**, לא את כרום באנדרואיד. אין כרגע מחלקה ל"דפדפן אנדרואיד" באתר — אם יידרש, מוסיפים `mh-android-browser` לפי המתכון בסעיף 6.

### שימוש ב-CSS

```css
/* דוגמה אמיתית מהפרויקט: הרמת הסרגל התחתון מעל כפתורי המערכת של אנדרואיד */
html.mh-android-app .floating-nav-pill {
    bottom: 28px !important;   /* ערך התחלתי — טעון כוונון לפי מכשיר */
}
```

כך התיקון נוגע **רק** באפליקציית אנדרואיד — אייפון ודפדפן לא מושפעים.

---

## 5. מערכת 2: דף ה-Maintenance

הזיהוי העצמאי שחי בתוך `index.html` של דף ההשקה (הגרסה המתוקנת, 2026-07-11):

```javascript
var ua  = navigator.userAgent || '';
var ref = document.referrer || '';

var fromApp   = /hyperzod/i.test(ref);                                // ההורה הוא דומיין האפליקציה
var androidWv = /\bwv\b/.test(ua);                                    // WebView אנדרואיד
var iosWv     = /iPhone|iPad|iPod/i.test(ua) && !/Safari\//.test(ua); // אפליקציית אייפון

var inApp = fromApp || androidWv || iosWv;   // מספיק סיגנל אחד
```

### עקרונות שמאחורי המבנה

- **רב-סיגנלים עם OR:** מספיק סיגנל אחד כדי להיחשב "בתוך האפליקציה". עדיף זיהוי-יתר של אפליקציה (הסתרת כפתורי הורדה ממישהו שכבר באפליקציה = נזק אפסי) על פני זיהוי-חסר.
- **ברירת המחדל בטוחה:** האלמנטים התלויים בזיהוי (כפתורי ההורדה) מתחילים `display:none` ב-CSS, וה-JS **חושף** אותם רק כשבטוח שזה דפדפן (`!inApp`). אם ה-JS נכשל — לא מוצג תוכן שגוי, רק חסר.
- **שימוש בדף:** כפתורי ההורדה (App Store `id6757471366`, Play Store `com.customer.maalehamishlohim`) מוצגים רק בדפדפן — אין טעם להציע "הורד את האפליקציה" למי שכבר בתוכה.

---

## 6. מתכוני זיהוי

מתכונים מוכנים לכל תרחיש. כולם בדוקים או נגזרים ישירות מטבלת האמת.

### "כל אנדרואיד" — אפליקציה + דפדפן (לא אייפון, לא מחשב)

```javascript
var isAndroid = /Android/i.test(navigator.userAgent);
```

הבדיקה הפשוטה ביותר, והיא מספיקה: ה-UA של WebView באנדרואיד מכיל `Android`, וגם כרום בנייד. אייפון ומחשב לא מכילים את הטוקן. **זה המתכון לגרסת HTML נפרדת "לאנדרואיד בלבד" בדף ה-Maintenance.**

### "אפליקציית אנדרואיד בלבד" (לא כרום באנדרואיד)

```javascript
// בתוך iframe (maintenance) — הכי אמין:
var isAndroidApp = /\bwv\b/.test(navigator.userAgent) ||
                   (/Android/i.test(navigator.userAgent) && /hyperzod/i.test(document.referrer || ''));

// בתוך דפי האתר — פשוט להשתמש במחלקה הקיימת:
document.documentElement.classList.contains('mh-android-app')
```

### "דפדפן באנדרואיד בלבד" (כרום, לא האפליקציה)

```javascript
var isAndroidBrowser = /Android/i.test(navigator.userAgent) &&
                       !/\bwv\b/.test(navigator.userAgent) &&
                       !/hyperzod/i.test(document.referrer || '');
```

### "אפליקציית אייפון בלבד"

```javascript
var isIOSApp = /iPhone|iPad|iPod/i.test(navigator.userAgent) &&
               !/Safari\//.test(navigator.userAgent);
```

(ב-iframe אפשר להוסיף `|| /hyperzod/i.test(document.referrer)` בשילוב עם בדיקת iPhone.)

### "כל דפדפן, לא אפליקציה" (הזיהוי של דף ה-Maintenance)

ראו סעיף 5 — `!inApp`.

### יישום ב-HTML: קובץ אחד עם מצבים, לא redirect

כשצריך גרסה שונה של דף לפלטפורמה מסוימת, **הדרך המומלצת** היא קובץ אחד שמכיל את שתי הגרסאות, כששתיהן מוסתרות כברירת מחדל וה-JS חושף את המתאימה:

```html
<div id="version-default" style="display:none"> ... </div>
<div id="version-android" style="display:none"> ... </div>
<script>
  var isAndroid = /Android/i.test(navigator.userAgent);
  document.getElementById(isAndroid ? 'version-android' : 'version-default')
          .style.display = 'block';
</script>
```

למה לא redirect לקובץ נפרד (`android.html`)? כי יש כתובת iframe אחת ב-Hyperzod, redirect יוצר הבהוב טעינה, ותחזוקה של שני קבצים מכפילה סיכון לסטייה ביניהם.

---

## 7. מצב דיבאג

דף ה-Maintenance כולל מצב דיבאג מובנה: הוספת `?debug=1` לכתובת מציגה שורת טקסט קטנה בתחתית המסך (רקע שחור, טקסט לבן 11px) עם ערכי הזיהוי בזמן אמת:

```
ref: ... | androidWv: true/false | iosWv: true/false | inApp: true/false
```

**למה זה קיים:** אי אפשר לפתוח DevTools על אפליקציה שהותקנה מהחנות (`chrome://inspect` לא זמין). שורת הדיבאג מאפשרת לצלם מסך מכל מכשיר ולדעת בדיוק איזה סיגנל "משקר" — בלי לנחש.

**כלל:** כל זיהוי חדש שמתווסף לדף (למשל `isAndroid`) — להוסיף גם לשורת הדיבאג.

---

## 8. הבדלי רינדור

אותו קוד רץ בשלוש סביבות שמתנהגות שונה. הלקחים שנצברו:

### אנדרואיד (Chromium WebView)

- **`env(safe-area-inset-bottom)` מחזיר `0px`** — אלא אם `viewport-fit=cover` מוגדר ברמת האפליקציה (בשליטת Hyperzod, לא שלנו). לכן ריווח מעל סרגל המערכת נעשה עם **פיקסלים קבועים** (הערך הנוכחי לסרגל התחתון: 28px, טעון כוונון).
- דליפות CSS מסלקטורים גנריים מדי (`:has()` רחב) מתגלות לפעמים **רק באנדרואיד** — ה-DOM של Vuetify עשוי להתרנדר עם קינון שונה מהדפדפן.
- כרום באנדרואיד **לא** מפעיל App Links תוך כדי גלישה (ראו סעיף 9).

### אייפון (WKWebView)

- **`backdrop-filter` + טרנספורמציות 3D כבדות מקריסים את ה-WebView** באייפונים ישנים. בדף ה-Maintenance זה חייב גרסה מוגבלת-ביצועים: אפס backdrop-filter, מקסימום צלליות ואנימציות מוגדר.
- בתוך עץ `preserve-3d`: המאפיינים `filter`, `overflow` (שאינו visible) ו-`clip` משטחים את הקשר ה-3D וגורמים לכשלים.
- Universal Links עובדים מכל מקום, כולל כרום-על-אייפון (לרוב).

### דפדפן מחשב

- הסביבה ה"סלחנית" — קוד שעובד רק במחשב הוא לא הוכחה לכלום. **בדיקה = שלוש הסביבות**, תמיד.

---

## 9. Deep Links

הקשר חשוב שקשור לזיהוי פלטפורמה, מתועד כאן כי הוא צץ שוב ושוב:

- **JS שרץ בדף לא יכול לגרום לאפליקציה להיפתח מתוצאת גוגל.** ההחלטה "דפדפן או אפליקציה" מתקבלת על-ידי מערכת ההפעלה **לפני** שהדף נטען. המנגנון האמיתי: Android App Links (`assetlinks.json`) ו-iOS Universal Links (`apple-app-site-association`) — מוגדרים ברמת הדומיין והאפליקציה (Hyperzod), הוגדרו ואומתו.
- **אסימטריה מובנית:** iOS פותח את האפליקציה מכל מקום (כולל כרום). אנדרואיד-כרום **בכוונה** נשאר בדפדפן במהלך גלישה — App Links מופעלים רק מלינקים שמגיעים מחוץ לדפדפן (WhatsApp וכו'). זו התנהגות OS שאי אפשר לעקוף.
- **הפתרון החלקי בשליטתנו:** כפתור עם `intent://` URL באנדרואיד-כרום — פותח את האפליקציה אם מותקנת, נופל ל-Play Store אם לא. (טרם יושם — רעיון עתידי.)
- **אימות אמת ל-App Links:** ה-API של גוגל —
  `https://digitalassetlinks.googleapis.com/v1/statements:list?source.web.site=https://www.maalehamishlohim.co.il&relation=delegate_permission/common.handle_all_urls`
  — הוא מקור האמת, לא המסך הירוק ב-Play Console.

---

## 9.5 הגשר הנייטיב של אפליקציית אנדרואיד (התגלה 2026-08-22)

האפליקציה בנויה **React Native** ומזריקה לכל דף (כולל iframes) את
`window.ReactNativeWebView`. הפרוטוקול חולץ מהקוד של הייפרזוד עצמם
(`cdn-store.hyperzod.app/assets/index-*.js`):

```javascript
// פתיחת קישור בדפדפן חיצוני — כך הייפרזוד פותחים בעצמם את צ'אט התמיכה:
window.ReactNativeWebView.postMessage(JSON.stringify({
  postMessageType: "openNativeExternalWebview",
  url: "https://example.com?dontAskForLocation=true&isExternalBrowser=true"
}));
// סוגי הודעות נוספים שנצפו בקוד שלהם:
// "OpenUrl" (url), "requestAppToOpenShare" (dataForShare),
// "notifyNativeStatusBarColor", "deviceTokenRequested", "openLocationSettings"
```

### ⚠️ אזהרה קריטית — קריסת אפליקציה אמיתית

ה-`onMessage` של האפליקציה מריץ `JSON.parse` **בלי try/catch** על כל הודעה.
שליחת מחרוזת שאינה JSON תקין (למשל `"whatsapp://..."` גולמי) **מקריסה את
האפליקציה כולה** (קרה בפועל, דוח קריסה: `JSON Parse error: Unexpected
character: w`). לשלוח אך ורק JSON בפורמט `{postMessageType: ...}` מוכר.

### עוד לקחים מאותה חקירה

- סכימות (`whatsapp://`, `intent://`) בתוך **iframe** (כמו דף ה-Maintenance)
  נכשלות עם `ERR_UNKNOWN_URL_SCHEME`, וניווט ה-iframe עצמו לסכימה מחליף את
  הדף בדף שגיאה והורג את כל ה-JS. לעומת זאת, ניווט ה**פריים הראשי** לסכימה
  (`window.top.location.href = "whatsapp://..."`) **עובד** — האפליקציה מיירטת
  ניווט ראשי ופותחת את האפליקציה החיצונית (אומת במכשיר 2026-08-22).
- `target="_blank"` בתוך האפליקציה לא עושה כלום (אין תמיכה בריבוי חלונות).
- ניווט `window.top.location` מה-iframe **מותר** (framed=true, אין sandbox).
- דף wa.me מנסה בעצמו `whatsapp://` — לכן גם ניווט אליו בתוך ה-WebView נכשל.

## 10. כללי עבודה

סיכום מהיר לשליפה:

1. **iframe (maintenance) = זיהוי עצמאי בתוך הדף. אתר Hyperzod = מחלקות `mh-android-app` / `mh-ios-app`.** לעולם לא לערבב.
2. **"כל אנדרואיד" = `/Android/i.test(ua)`.** פשוט ומספיק.
3. **אפליקציית אנדרואיד = `/\bwv\b/`** (עם גבולות מילה). ב-iframe להוסיף בדיקת referrer.
4. **אפליקציית אייפון = iPhone בלי `Safari/`.** לא `webkit.messageHandlers` (משקר בדף ה-Maintenance), לא `nativeVibrateShort` (משקר תמיד).
5. **ברירת מחדל בטוחה:** תוכן תלוי-פלטפורמה מתחיל מוסתר, ה-JS חושף. כשל JS = חוסר, לא שגיאה גלויה.
6. **כל זיהוי חדש נכנס לשורת ה-`?debug=1`.**
7. **בדיקה בשלוש סביבות** לפני הכרזה על הצלחה — ורק המשתמש מאשר שזה עובד.
8. **גרסאות פלטפורמה = קובץ אחד עם מצבים**, לא redirect לקבצים נפרדים.
