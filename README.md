KOMBAIN USB Fleet Updater

עמדת Windows אופליין לעדכון מקבילי ומבוקר של טאבלטי KOMBAIN דרך USB‑ADB.

התיעוד הטכני העדכני: docs/00_START_HERE.md, כולל תוכנית התאמה ל־macOS. סימולציה תומכת במספר מכשירים; התקנה אמיתית כיום מוגבלת לפיילוט יחיד.

מצב נוכחי

אב־טיפוס ראשון כולל:





גילוי מספר מכשירי ADB לפי serial נפרד.



קריאת דגם, Android, גרסת KOMBAIN, סוללה ואחסון פנוי.



ייבוא רשימת צי מקומית מ־CSV.



חישוב SHA‑256 ובדיקת package/certificate באמצעות Android Build Tools.



ממשק עברי RTL לפי ערכת הצבעים של KOMBAIN.



סימולציית פריסה בטוחה כברירת מחדל.



מצב פיילוט להתקנה אמיתית על טאבלט יחיד, עם אישור כפול.



תשתית פריסה עם מגבלה של שש פעולות במקביל; בממשק התקנה אמיתית מוגבלת כיום למכשיר אחד.



תרגום שגיאות התקנה נפוצות לעברית.



השוואת חתימת SHA-256 מול ה־APK שמותקן בפועל בכל טאבלט.



אימות versionCode לאחר ההתקנה.



שורת JSONL לכל תוצאת מכשיר בקובץ מקומי חודשי משותף.



דרישות עמדת Windows





Windows 11 x64



.NET 8 SDK לבנייה או .NET 8 Desktop Runtime להפעלה



Android Platform Tools (adb)



Android Build Tools (aapt2, apksigner)



Java/JDK להפעלת apksigner.jar (אינו נכלל בחבילה)



כבלי USB איכותיים ורכזות USB בעלות אספקת חשמל עצמאית



בנייה

ברירת המחדל מיועדת לבנייה אופליין. היא אינה משתמשת ב־ILLink ואינה מורידה runtime pack. נדרש במחשב הבנייה .NET 8 SDK מלא; להפעלת התוצאה נדרש .NET 8 Desktop Runtime.

powershell -ExecutionPolicy Bypass -File .\build-windows.ps1

אם בונים במחשב המחובר לרשת ורוצים חבילה עצמאית שאינה דורשת Runtime בעמדת ההפעלה:

powershell -ExecutionPolicy Bypass -File .\build-windows.ps1 -SelfContained

האפשרות העצמאית עשויה להוריד מ־NuGet את חבילות ה־runtime ואת Microsoft.NET.ILLink.Tasks; לכן היא אינה ברירת המחדל לעמדה אופליין.

אם מתקבלת שגיאת NU1100, יש לוודא שהותקן .NET 8 SDK ולא רק Runtime:

dotnet --list-sdks

הפלט צריך לכלול שורה שמתחילה ב־8.. אין צורך לשנות מקורות NuGet או לחבר את העמדה לרשת עבור הבנייה הרגילה.

בטיחות

מצב סימולציה מסומן מחדש בכל הפעלה. ניתן לכבות אותו רק לאחר בחירת APK תקין וחיבור מכשיר. בשלב הפיילוט התקנה אמיתית מוגבלת לטאבלט אחד ודורשת אישור כפול. הכלי משווה את תעודת החתימה מול האפליקציה המותקנת, משתמש ב־adb install -r, אינו מבטל את Fully Kiosk, אינו משנה Device Owner ואינו מסיר את KOMBAIN לפני עדכון.

שם החבילה הנוכחי מוגדר ב־AdbService.KombainPackage. יש לאמת אותו מול APK ה־release לפני מסירה.

הפעלה ואבחון

יש לחלץ ולהשאיר יחד את כל תוכן KOMBAIN-Updater-win-x64.zip; ה־EXE אינו מיועד להעתקה בנפרד משאר קובצי התיקייה.

להפעלה רגילה פתח את KOMBAIN USB Fleet Updater.exe. אם הממשק אינו עולה, הפעל את Start-KOMBAIN-Updater.cmd. במקרה של כשל יוצג הפלט ויישמר יומן תחת:

%LOCALAPPDATA%\KOMBAIN\Updater\Logs

