# macOS — תוכנית התאמה וקומפילציה

## מצב נוכחי

ה־csproj משתמש ב־UseWPF=true וב־net8.0-windows, והמסכים תלויים ב־System.Windows, Microsoft.Win32.OpenFileDialog ו־WPF XAML. `dotnet publish -r osx-arm64` על הפרויקט הזה אינו יוצר כלי GUI מקומי למק. Cross-build של תוצר Windows על Mac אינו גרסת macOS.

דרישת המוצר היא להשאיר כלי תפעול אופליין דרך USB, ממשק עברי RTL, בדיקות APK, סימולציה, פיילוט, עדכון ללא uninstall ויומן. אין צורך לשנות את אפליקציית Android כדי ליצור את ממשק המק.

## מסלול מוצע

לשמר לוגיקת C# ולבודד מנוע שאינו תלוי ב־WPF. אפשר לבנות ממשק משותף ב־Avalonia, שתומכת ב־Windows וב־macOS (Apple Silicon ו־Intel), ולא להעתיק ישירות XAML של WPF. זו הצעה הנדסית, לא migration שכבר בוצעה. יש לבחור גרסה יציבה ולנעול SDK וחבילות לפי התאימות שלה. [תיעוד Avalonia רשמי](https://docs.avaloniaui.net/docs/supported-platforms).

```text
src/
  Kombain.Updater.Core/       models, inspection, fleet, deployment engine
  Kombain.Updater.Platform/   tool paths, process runner, filesystem/logs
  Kombain.Updater.Desktop/    cross-platform UI + ViewModels
tests/                       process fixtures and domain/engine tests
scripts/
  build-windows.ps1
  build-macos.sh
```

השמות והמבנה מוצעים; אינם קבצים קיימים בחבילה. קודם להפריד orchestration מ־MainWindow ולהבטיח ש־Windows ממשיך לפעול, ורק אז לממש UI נוסף/משותף.

## מה משתף ומה משתנה

| אזור | מצב/פעולת port |
|---|---|
| מודלי APK/CSV ו־SHA-256 | ניתנים לשיתוף לאחר הוצאת תלות ב־UI |
| AdbService | רוב הפקודות משותפות; בדוק פלט, UTF-8, temp והנחות splits |
| DeployOne והאישורים | חילוץ למנוע עם תוצאות מובנות; UI מיישם אישורים |
| ProcessRunner | Java/ADB ישירים עם ArgumentList; בלי cmd.exe/BAT; timeout ו־process kill |
| JavaToolLocator | כיום non-Windows מחזיר java בלבד; נדרש JAVA_HOME ובדיקה באמצעות java -version, איתור JDK macOS |
| AndroidToolLocator | הוסף SDK ברירת מחדל של Mac והתאמה ל־.app bundle; לא להשתמש ב־.exe/DLLs |
| CSV | שמור UTF-8/BOM; fallback מלא/מפורש במקום מיפוי חלקי לפי הצורך |
| Logs | ספק נתיב writable ומוצג במק, למשל Library/Application Support; בדוק actual path |
| GUI | חלונות, file picker, טבלה, progress, theme ו־RTL חדשים/מותאמים |
| Packaging | .app, Info.plist, arm64/x64, executable permissions ו־native dependencies |

## כלים במק

מפתח צריך .NET SDK בגרסה שנבחרה, Android SDK Platform Tools/Build Tools ו־Java תואם. SDK נמצא לעיתים ב־`~/Library/Android/sdk`; AndroidToolLocator הנוכחי אינו מנסה נתיב זה כברירת מחדל ולכן נדרש נתיב מפורש או משתנה ANDROID_HOME.

אפשר להתחיל בפיילוט מק עם תלויות חיצוניות מותקנות ו־Tool Paths מפורשים. בהמשך לארוז Android tools למק במקום הכלים של Windows. Java אינה מתווספת אוטומטית לחבילה. אם מבקשים לכלול JRE, יש להגדיר מקור, ארכיטקטורה, רישוי ורכיבי חבילה בפירוש.

סדר איתור מוצע: הגדרה מפורשת → כלי חבילה → SDK/Java home → PATH שנבדק. תהליך שהופעל דרך Finder לא בהכרח מקבל PATH של shell; אין להסתמך על Homebrew או ~/.zshrc ללא בדיקה.

אין דרישה לשנות Device Owner או לפרמט טאבלט. גם במק נדרש אישור RSA וחיבור USB debugging שמותר לפי מדיניות הצי.

## תוכנית ביצוע ושערי קבלה

1. **ליבת C# עצמאית:** בדיקות תוצאות/כישלונות של inspection, CSV, signature ו־version; process runner ניתן להחלפה בבדיקות. Windows pilot נשמר.
2. **CLI אבחוני במק:** סריקה, קריאת מידע ו־verify APK בלבד; אין install אוטומטי בבדיקת smoke.
3. **ממשק מק:** טבלה RTL ושמות בעברית, בחירה, file picker וסימולציה; plan קפוא ונעילת controls בזמן פעולה.
4. **הקשחת פעולות:** timeouts, cancellation שהורגת child processes, hash מחדש לפני install, version preflight, serialize logs ומצב audit-failed נפרד.
5. **פיילוט יחיד:** שני אישורים, digest תואם, install -r, versionCode, בדיקת דוחות ידנית. Windows ו־Mac משתמשים באותו מנוע ללא עקיפות אבטחה.
6. **אריזה:** app bundle עבור arm64; x64 אם מחשבי Intel כלולים בצי. פלט הנפתח מ־Finder ללא terminal, עם כל התלויות המוצהרות ויומן נגיש.
7. **צי:** הרחבת כמות רק לאחר בדיקות 1/2/4/8 עם מכשירים אמיתיים ו־USB hub מוזן. אין להפעיל מקביליות רק מפני שהסמפור הוא 6.

## קומפילציה עתידית

הפקודות הבאות הן דוגמאות לאחר יצירת Desktop project רב־פלטפורמי; אין להריצן כאילו הפרויקט הזה כבר קיים:

```sh
dotnet restore src/Kombain.Updater.Desktop/Kombain.Updater.Desktop.csproj
dotnet publish src/Kombain.Updater.Desktop/Kombain.Updater.Desktop.csproj -c Release -r osx-arm64 --self-contained true
dotnet publish src/Kombain.Updater.Desktop/Kombain.Updater.Desktop.csproj -c Release -r osx-x64 --self-contained true
```

publish אינו יוצר אוטומטית חבילת .app תקינה. הוסף מבנה bundle, Info.plist, אייקון וקובצי כלים לפי מדריך הפריסה; בדוק את דרישות signing/notarization לשיטת ההפצה שנבחרה. [מדריך macOS של Avalonia](https://docs.avaloniaui.net/docs/deployment/macos).

חתימת אפליקציית המק (codesign) וחתימת APK Android הן מערכות נפרדות. כלי מק מחזיק APK חתום ובודק אותו; אין להעביר אליו את keystore לצורך הפצת עדכונים. keystore נדרש למחשב/תהליך הבנייה של KOMBAIN, לא לעמדת ה־updater.

## הגדרת הצלחה

גרסת המק מתקבלת רק אחרי פתיחה מ־Finder במחשב היעד, גילוי טאבלט, UTF-8/עברית, אימות APK, סימולציה עם יומן ופיילוט שמירת נתונים. בדיקה על macOS אינה מחליפה regression ב־Windows. שימוש ברשת להכנת תלויות/חתימת חבילה מתועד בנפרד; עמדת תפעול הטאבלטים פועלת אופליין.
