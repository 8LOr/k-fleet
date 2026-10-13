# Windows — בנייה, תפעול ואבחון

## הכנת העמדה

מערכת היעד היא Windows 11 x64. לבנייה נדרש .NET 8 SDK מלא, Android SDK Platform Tools ו־Build Tools. להפעלת publish הרגיל נדרש .NET 8 Desktop Runtime. Java דרושה ל־apksigner.jar ואינה נארזת בסקריפט הנוכחי. .NET ו־Java הם תלויות נפרדות.

בחר גרסאות כלים ותעד אותן. ניתן להכין תלויות על מחשב מורשה המחובר לרשת ואז להעביר לעמדה; תפעול הטאבלטים נעשה אופליין. offline publish אינו מבטיח בנייה במחשב חסר targeting packs או תלויות.

בדיקות בסיסיות ב־PowerShell:

```powershell
dotnet --list-sdks
dotnet --list-runtimes
& "$env:LOCALAPPDATA\Android\Sdk\platform-tools\adb.exe" version
```

לפני חיבור הטאבלטים יש להשיג חיבור ADB מורשה, כבל נתונים, אישור RSA של העמדה ונהלי פתיחת/סגירת הרשאות המותרים בצי. הכלי אינו פותח אותם אוטומטית. במידת הצורך מתקינים דרייבר USB מאושר של היצרן על Windows. רשימת serials וקודי ניהול נשמרים מחוץ לקוד.

## בנייה

הרץ משורש הכלי:

```powershell
powershell -ExecutionPolicy Bypass -File .\build-windows.ps1
```

SDK במיקום אחר:

```powershell
powershell -ExecutionPolicy Bypass -File .\build-windows.ps1 -AndroidSdk "D:\Android\Sdk"
```

פרמטרים: Configuration (Release), Runtime (win-x64), AndroidSdk, SelfContained. Runtime משמש ב־SelfContained; publish הרגיל אינו מקבע RID.

ברירת המחדל היא framework-dependent, ללא SingleFile/ReadyToRun/Trim. לפני הבנייה נמחקת תיקיית הפלט הייעודית `artifacts/KOMBAIN-Updater` — אין לשמור שם APK, CSV או קובץ ידני. הקוד המקורי אינו נמחק.

הסקריפט מבצע restore עם ignore-failed-sources ולאחריו publish עם no-restore. הוא מעתיק ADB ושני DLLs של Windows, aapt2.exe, apksigner.bat ותיקיית lib; תעודות חתימה ו־Java אינן נכללות.

```text
artifacts/
  KOMBAIN-Updater/
    KOMBAIN USB Fleet Updater.exe
    KOMBAIN USB Fleet Updater.dll
    ... runtimeconfig / deps / קובצי runtime נדרשים
    Start-KOMBAIN-Updater.cmd
    tools/platform-tools/
    tools/build-tools/lib/apksigner.jar
  KOMBAIN-Updater-win-x64.zip
```

להפצה חילץ את כל התוכן יחד. ה־EXE אינו עצמאי. אפשר לבקש SelfContained, אך נדרש מטמון runtime packs או רשת בזמן הבנייה. גם SelfContained אינו כולל Java. launcher.cmd עדיין מפעיל dotnet, ולכן ב־SelfContained ללא runtime מערכתי יש להפעיל את ה־EXE לצורך הפעלה רגילה; אין להניח שקובץ CMD יעבוד באותה סביבה.

## סדר תפעול

1. הפעל EXE מתוך התיקייה המלאה. בדוק שסימולציה מסומנת.
2. חבר טאבלט ניסוי ולחץ סריקת טאבלטים. `unauthorized` מחייב אישור RSA בטאבלט; `offline` מחייב בדיקת חיבור.
3. ייבא CSV מקומי לפי הצורך; הוא מסייע לזהות מכשיר אך אינו תנאי הרשאה.
4. בחר APK. דרוש package תואם ואימות certificate תקין. טביעת קובץ ותעודה הם שני ערכים שונים.
5. הרץ סימולציה ובדוק מזהי מכשיר, גרסה מוצגת, הודעת תוצאה ויומן עם Simulation=true.
6. לפיילוט: ודא שחזור/גיבוי תקין של נתוני טאבלט הניסוי, בחר מכשיר אחד, בטל סימולציה ואשר פעמיים.
7. הצלחה אמיתית דורשת חתימת מותקן תואמת, הצלחת install ו־versionCode השווה ליעד.
8. פתח KOMBAIN על הטאבלט ובדוק דוחות, טיוטות, הגדרות, מתגי עטלף/גיל ופעולות מייצגות. חזור להגנות הקיוסק לפי נוהל הצי.

הכלי אינו מאפשר כיום פיילוט אמיתי של 2/4/8 מכשירים. הרחבה זו היא פיתוח עתידי אחרי בדיקת מכשיר אחד ותיקון המגבלות במסמך הטכני.

## אבחון

| הודעה/תופעה | משמעות/פעולה |
|---|---|
| NU1100 / ILLink | בדוק SDK מלא; הרץ publish הרגיל. חבילה ישנה או SelfContained עשויים להזדקק להורדות |
| Path/File/Directory לא מזוהים | גרסת מקור ישנה עם using System.IO חסר; העבר את המקור התואם לחבילה |
| Copy-Item lib exists | סקריפט ישן או פלט שנותר נעול; סגור כלי ורוץ עם הסקריפט הנוכחי |
| EXE לא עולה | שמור DLL/JSON לידו; הפעל CMD וקרא launcher.log ו־startup-error.log |
| XamlParseException Color/Background | תבנית ישנה השתמשה ב־Color במקום Brush; בדוק שמשאבי XAML תואמים למקור |
| apksigner.bat not recognized | המימוש הנוכחי עוקף BAT כאשר נמצא הנתיב; בדוק שלא פועל EXE ישן וש־tools/build-tools קיים |
| Java לא נמצאה | הגדר JAVA_HOME לתיקיית JDK/JBR המכילה bin/java.exe; JavaToolLocator לא מחפש PATH בלבד |
| apksigner.jar לא נמצא | חסרה lib בחבילה או אותר SDK אחר; תקן חבילת Build Tools |
| package mismatch | בחר APK של mil.egoz.kombain |
| cannot verify installed certificate | בדוק pm path, הרשאת pull וקיום אפליקציה; אל תבצע הסרה כפתרון |
| UPDATE_INCOMPATIBLE | המפתח אינו תואם למותקן; השתמש ב־APK שנחתם במפתח המקורי |
| VERSION_DOWNGRADE | versionCode של היעד נמוך מדי; בנה גרסה מתאימה |
| INSUFFICIENT_STORAGE | פנה מקום לפי נוהל בלי למחוק דוחות/נתוני אפליקציה |
| Hebrew ???? ב־CSV | שמור מה־Excel המקורי כ־CSV UTF-8; תווים שכבר אבדו אינם ניתנים לשחזור |
| יומן נכשל אחרי install | בדוק גרסה מותקנת לפני ניסיון חוזר; חסרה הפרדת תוצאת התקנה מתוצאת audit |

## מינימום בדיקות מסירה

תעד Windows, .NET, Java, Build Tools, ADB, commit/hash של הכלי ושל APK, תוצאת סימולציה, תוצאת התקנה/נתונים ובדיקה עם כבל מנותק או permission denied. בדוק נתיבים עם רווחים ועברית, CSV עם BOM, חותם שונה, אותה גרסה, downgrade וסוללה נמוכה. אין צורך לחשוף serial אמיתי במסמך ההעברה.

הבדיקות הללו הן נוהל נדרש; כתיבת המסמך אינה עדות שהן עברו.
