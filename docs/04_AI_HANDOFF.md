# פרומפט המשך ורשימת מסירה

## קבצים להעברה

העבר את כל שורש כלי העדכון, כולל:

```text
docs/00_START_HERE.md
docs/01_TECHNICAL_REFERENCE.md
docs/02_WINDOWS_OPERATIONS.md
docs/03_MACOS_PORT_PLAN.md
docs/04_AI_HANDOFF.md
SOURCE_SHA256.txt
README.md
WINDOWS_TEST_CHECKLIST.md
Kombain.Updater.sln
build-windows.ps1
Start-KOMBAIN-Updater.cmd
sample-data/fleet-example.csv
src/Kombain.Updater/Kombain.Updater.csproj
src/Kombain.Updater/App.xaml
src/Kombain.Updater/App.xaml.cs
src/Kombain.Updater/MainWindow.xaml
src/Kombain.Updater/MainWindow.xaml.cs
src/Kombain.Updater/app.manifest
src/Kombain.Updater/Models/*.cs
src/Kombain.Updater/Services/*.cs
```

לא נדרשים node_modules, מקור אפליקציית Android או keystore כדי להבין/לפתח את כלי העדכון. לניסוי נדרש APK חתום ותואם וטאבלט מורשה. רשימת צי אמיתית, קודי ניהול, יומנים ונתוני תחקיר נשמרים מחוץ לחבילת ה־AI. לא מצרפים artifacts/bin/obj או תלויות מערכת מתוך המחשב המקומי.

SOURCE_SHA256.txt מזהה את snapshot הקוד בחבילה. הוא אינו חתימה קריפטוגרפית של המפיץ או הוכחה שהקוד נבדק ב־Windows.

## פרומפט מוכן להמשך

```text
Continue development of KOMBAIN Fleet USB Updater using this source package.

Read all docs/ files and the current source before editing. This is a separate offline USB-ADB updater, not the Android KOMBAIN app. Current UI is C#/.NET 8 WPF targeting net8.0-windows. Source snapshot hashes are in SOURCE_SHA256.txt. ZIP delivery names are not authoritative assembly versions.

The owner requires a macOS build as well as continued Windows support. A WPF project cannot become a native Mac GUI just by changing RID. Use docs/03_MACOS_PORT_PLAN.md as a proposed port plan: extract a platform-independent deployment engine and models, keep process/tool/logging adapters explicit, then add a cross-platform desktop UI (Avalonia is a candidate, not implemented yet). Verify framework versions and supported OS/architecture from official docs before pinning dependencies.

Preserve these constraints:
- Offline tablet operation over USB ADB; no cloud MDM or factory reset.
- Existing Fully Kiosk Single App Device Owner and Knox controls are handled by the authorized operator. Do not automate bypassing them.
- Simulation ON at startup. Real install remains one-tablet pilot with two confirmations until physical tests authorize wider rollout.
- Exact Android package mil.egoz.kombain, candidate APK signature verification, installed certificate comparison before installation, adb install -r --no-streaming, and target versionCode verification.
- Never uninstall, clear data, root, auto-downgrade, or install an unverifiable/different signer APK to force progress.
- The tool does not sign APKs or need a private keystore. APK production/signing happens separately.
- UTF-8 Hebrew/RTL, locally imported redacted/sample CSV, local result logging with Simulation flag.

Current limitations are real work items, not completed capabilities:
- Simulation skips installed certificate inspection and actual installation/version-change checks.
- Free-storage capacity is displayed but not enforced; battery may be unknown/stale.
- MainWindow code-behind owns deployment; not a standalone MVVM/core architecture.
- Commands lack timeouts and effective child-process cancellation.
- UI busy state does not lock all controls; candidate file can change after inspection.
- The monthly JSONL writer has no serialized queue; an audit failure can follow a successful install.
- No automatic app-data backup/restore, rollback, or verified Kiosk relaunch.
- Signature comparison assumes one signer; installed split APK handling is incomplete.
- JavaToolLocator is Windows-specific; macOS SDK lookup and Finder PATH behavior need a real adapter.

Begin with a source inventory and a reproducible Windows baseline. Propose the smallest tested port milestone. First extract a shared engine and read-only Mac diagnostics; then simulation UI, then one-device pilot. Build both Windows and Mac artifacts with explicit dependency/version manifests, .app packaging, and appropriate architecture. Validate launch from Finder without terminal environment assumptions.

Use deterministic tests for process errors, metadata, CSV, plan immutability, signature mismatch, version validation, and audit failures. Hardware tests must cover a real pilot preserving existing KOMBAIN reports, USB disconnect, unauthorized devices, Hebrew paths and logs. Do not claim a Mac build, signed artifact, or successful fleet rollout until verified with saved evidence.

Do not upload real serials, fleet CSVs, report data, credentials or keys. Preserve existing source changes and history. Document source-level results separately from tests reported by the operator.
```

## תחזוקה עתידית

כאשר הממשיך משנה התנהגות, עליו לעדכן את המסמך הטכני, נוהל הפיילוט, מספר גרסת הכלי ותוצאות הבדיקות יחד. יש לשמור את snapshot המקורי להשוואה; אין להחליף תיעוד דרישה בטענה שהיא כבר מומשה.
