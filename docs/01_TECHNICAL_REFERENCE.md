# Technical Reference — מימוש נוכחי

כל נתיבי המקור להלן יחסיים לתיקיית כלי העדכון. אם הוא חלק מהריפו הראשי, שורשו הוא `tools/windows-updater/`.

## מטרה וגבולות

הכלי מזהה טאבלטים מחוברים, מצמיד להם שמות מרשימת צי מקומית, בודק APK, מדמה פריסה או מבצע פיילוט עדכון ושומר יומן. אין לו backend ואין פקודת רשת יזומה בקוד היישום. רכיב ADB עשוי להפעיל שרת מקומי — אופליין אינו אומר שאין socket מקומי.

הוא אינו מייצר APK, אינו חותם מחדש על APK, אינו מחזיק keystore, אינו ממיר Debug ל־Release ואינו מבצע backup/restore של תחקירים. הוא אינו מנהל הרשאות Knox, Fully Kiosk, Device Owner, קוד נעילה או פתיחת USB debugging. אלה תנאי הכנה של המפעיל.

## רכיבים

| רכיב | אחריות |
|---|---|
| `App.xaml`, `App.xaml.cs` | משאבי UI כהים, יצירת החלון ותיעוד חריגות אתחול/UI |
| `MainWindow.xaml`, `.xaml.cs` | רשימת מכשירים, בחירת APK/CSV, בחירה וסימולציה, תזמור הפעולה והאישורים |
| `Services/AdbService.cs` | גילוי, קריאת מידע, משיכת APK מותקן, עדכון ובדיקת גרסה |
| `Services/ApkInspector.cs` | SHA-256 לקובץ, metadata מ־aapt2, בדיקת certificate |
| `Services/CertificateService.cs` | apksigner verify וקריאת SHA-256 של תעודת החותם |
| `Services/AndroidToolLocator.cs` | איתור ADB/Build Tools מתוך חבילה או SDK |
| `Services/JavaToolLocator.cs` | איתור Java להפעלה ישירה של JAR ב־Windows |
| `Services/ProcessRunner.cs` | הרצת תהליכים, stdout/stderr ו־exit code |
| `Services/FleetCsvService.cs` | קריאת CSV, קידוד ומיפוי צי לפי serial |
| `Services/AuditLogService.cs` | כתיבת שורת JSON לכל תוצאת מכשיר, בקובץ חודשי |
| `Models/` | TabletDevice, FleetRecord, ApkInspection, DeploymentLogEntry |
| `build-windows.ps1` | publish והעתקת Android tools ל־ZIP |
| `Start-KOMBAIN-Updater.cmd` | הפעלה אבחונית דרך dotnet ויומן launcher |

הפרויקט אינו MVVM מופרד: רוב תזמור הפריסה נמצא ב־code-behind של MainWindow. `TabletDevice` מיישם INotifyPropertyChanged רק לחלק מהשדות. זו נקודת refactor חשובה לגרסה משותפת למק ול־Windows.

## גילוי וקריאת מכשירים

1. הרצת `adb devices -l`.
2. פירוק פלט לפי שורות ורווחים. כל מכשיר במצב `device` מסומן אוטומטית לבחירה; unauthorized/offline מוצגים ללא בדיקת תוכן.
3. עבור מכשירים מורשים, קריאות מידע במקביל:

```text
adb -s <serial> shell getprop ro.build.version.release
adb -s <serial> shell dumpsys package mil.egoz.kombain
adb -s <serial> shell dumpsys battery
adb -s <serial> shell df -k /data
```

מספר ה־versionCode נקרא בעת אימות גרסה; בשורת המכשיר מוצג versionName. סוללה לא ידועה היא `-1`; אחסון לא ידוע הוא `-1`. מצב „מוכן לעדכון” לאחר סריקה אינו תוצאה של השוואת חתימה או בדיקת קיבולת מספקת. כישלון shell הופך לפלט ריק ולא תמיד מציג סיבת כשל מדויקת.

## בדיקת ה־APK הנבחר

`ApkInspector` מחשב SHA-256 על stream לקובץ, קורא `aapt2 dump badging <apk>` ומחלץ name/versionName/versionCode. ערך IsValid מחייב package מדויק `mil.egoz.kombain` והצלחת CertificateService. אין allowlist של certificate רשמי, אין תנאי שה־APK הוא Release, ואין תנאי קשיח שה־versionCode גדול מאפס או גבוה מהמותקן.

ב־Windows, אם אותר apksigner.bat, הכלי עוקף אותו ומריץ:

```text
<java.exe> -jar <build-tools>/lib/apksigner.jar verify --verbose --print-certs <apk>
```

בשאר המצבים הוא מפעיל את apksigner עצמו. הפלט stdout ו־stderr מאוחדים; regex מחפש `certificate SHA-256 digest:`. Digest עובר הסרת נקודתיים והמרה לאותיות גדולות. בכשל מוחזרת הודעה מהשורה הראשונה של הפלט. זהות חותם יחידה היא ההנחה הנוכחית: אין לוגיקה מלאה לריבוי חותמים או signing lineage/rotation.

בחירת APK תקין אינה אומרת שהוא ניתן להתקנה על כל הטאבלטים. ההתאמה למותקן נבדקת רק בפעולה אמיתית.

## סימולציה מול התקנה אמיתית

| שלב | סימולציה | פיילוט אמיתי |
|---|---|---|
| סריקה וקריאת מידע | ADB אמיתי | ADB אמיתי |
| hash/metadata/verify של APK נבחר | אמיתי | אמיתי |
| סוללה ידועה מתחת 20% | נכשל | נכשל |
| משיכת APK מותקן והשוואת certificate | לא מבוצע | מבוצע |
| העברה והתקנה | השהיות מדומות | adb install -r --no-streaming |
| בדיקת versionCode לאחר התקנה | השהיה מדומה | dumpsys והשוואה לערך היעד |
| יומן | Simulation=true | Simulation=false |
| מספר מכשירים | מספר מכשירים, עד 6 פעולות במקביל | בדיוק מכשיר אחד |

סימולציה ON בפתיחה. כיבוי המתג עם מספר מכשירים נבחרים משאיר את הראשון בלבד. בזמן התחלת התקנה אמיתית קיימת גם בדיקה נוספת למספר מכשירים; נדרשים אישור גרסה/מכשיר ואישור גיבוי נוסף.

## זרימת פיילוט אמיתי

```text
APK תקין + מכשיר מורשה יחיד + שני אישורים
  → סוללה (אם ידועה)
  → pm path → adb pull של APK מותקן לקובץ זמני
  → verify של המותקן → השוואת digest ל־APK הנבחר
  → adb install -r --no-streaming
  → exit code 0 ו־Success בפלט
  → קריאת versionCode → שוויון לגרסת היעד
  → סטטוס הצלחה + שורת JSONL
```

קריאת החתימה המותקנת משתמשת בשורה הראשונה של `pm path`. במכשיר עם split APK אין טיפול מפורש בכל ה־splits. הקובץ המקומי הוא `%TEMP%/kombain-installed-<guid>.apk` ונמחק ב־finally; קריסת תהליך יכולה להשאיר אותו.

הכלי משתמש רק בעדכון `-r` ואינו מסיר את האפליקציה. זה נועד לשמור נתונים, אך הצלחה אינה בדיקה שהדוחות/טיוטות נותרו או שהאפליקציה נפתחת. יש לבדוק זאת ידנית. אימות versionCode בודק שוויון ליעד — אינו מוכיח שהגרסה השתנתה אם אותה גרסה הותקנה מחדש.

## CSV

```csv
adb_serial,asset_number,display_name
EXAMPLE_SERIAL_001,TAB-001,טאבלט בדיקה
```

כותרות מספר סידורי ומספר פנימי נדרשות; שם תצוגה אופציונלי. כותרות נתמכות:

| שדה | כותרות |
|---|---|
| Serial | adb_serial, serial, מספר סידורי, סיראל |
| Asset | asset_number, asset, מספר פנימי, מספר נכס |
| Name | display_name, name, שם, שם תצוגה |

מפריד הוא פסיק בלבד. מרכאות ו־escaped quotes באותה שורה נתמכים; אין תמיכה בשדה מרובה שורות או זיהוי נקודה־פסיק. UTF-8 עם BOM, UTF-8 ללא BOM ו־UTF-16 עם BOM נתמכים. fallback ל־Windows-1255 ממפה אותיות עבריות וכמה סימנים בלבד; אינו מפענח מלא לכל התווים. שמירה מומלצת: CSV UTF-8 מתוך Excel המקורי שבו העברית תקינה. סימני `????` שכבר נשמרו אינם ניתנים לשחזור.

שורות חסרות serial/עמודות נדרשות מדולגות. serial כפול: השורה האחרונה גוברת. Serial לא מוכר אינו חסום מהתקנה — CSV הוא תצוגה ומיפוי, לא allowlist. רשימת הצי נשארת בזיכרון עד סגירת התהליך.

## יומנים

```text
%LOCALAPPDATA%/KOMBAIN/Updater/Logs/
  deployments-YYYY-MM.jsonl
  startup-error.log
  launcher.log
```

DeploymentLogEntry כולל Timestamp, DeploymentId, Serial, AssetNumber, ApkFile, VersionName, VersionCode, ApkSha256, CertificateSha256, Simulation, Result ו־Message. Result הוא success/failed. למרות תוצאה לכל מכשיר, הקובץ הוא חודשי משותף — לא קובץ נפרד לכל טאבלט. אין קריאת תוכן תחקירים ביומן, אך serial, מספר פנימי, שם APK ושגיאות עשויים להיות רגישים.

אין הצפנת יומן, rotation לפי נפח, נוהל מחיקה אוטומטי או נעילת כתיבה מקבילית. failure בכתיבת log יכול לזרוק חריגה אחרי הצלחת התקנה; אין להתקין שוב רק בגלל כשל ביומן בלי לבדוק גרסה מותקנת.

## מגבלות שיש לדעת לפני הרחבת הצי

- אין timeout לפקודות ADB/Java. CancellationToken אינו מחובר לכפתור ביטול; ביטול WaitForExit אינו הורג אוטומטית את התהליך החיצוני.
- SetBusy משבית רק כפתורי סריקה/פריסה. בחירת APK, CSV, מתג סימולציה וכפתורי בחירה אחרים אינם נעולים באופן מלא; אין immutable deployment plan.
- סוללה/אחסון נקראים בסריקה ועלולים להתיישן. אחסון מוצג אך אינו preflight חוסם; סוללה לא ידועה אינה חוסמת.
- אין בדיקת downgrade מוקדמת, בדיקת מינימום Android או אבחנה בין re-install באותה גרסה לעדכון חדש; שגיאות Android מפוענחות אחרי ניסיון התקנה.
- אין בדיקת hash חוזרת לפני install. קובץ APK שהוחלף אחרי בחירתו יכול לגרום לאי־התאמה בין הבדיקה/היומן לבין הבייטים המותקנים.
- שינוי checkbox של שורת מכשיר אינו מחובר באופן ישיר ל־UpdateDeployState; יש לאמת UI במצבי בחירה/ניקוי.
- אין recovery של פעולה אחרי קריסה, rollback אוטומטי, חידוש העברה או התחייבות שאפליקציה חוזרת לקיוסק לאחר התקנה.
- build-tools נבחר לפי מיון טקסטואלי של שם תיקייה, לא גרסה סמנטית. יש לתעד את גרסת הכלים ששימשה לבנייה.
- SHA-256 מחושב, אך לא מושווה לרשימת artifacts מאושרת. יש לתעד artifact/commit/hash כחלק מנוהל ההפצה.

אלה פערי תחזוקה לממשיך; לא בוצעו תיקוני מוצר במסגרת כתיבת התיעוד.
